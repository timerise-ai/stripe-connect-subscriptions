# Settlement

The fan-out that turns one paid charge into N seller transfers, escrow holds and
reserves. Runs once per order. It is the most failure-prone code in the module,
because every step touches an external system after the buyer's money has already
moved.

## Contract

| Property | How |
|---|---|
| **Single-flight** | An atomic leased claim on the payment intent |
| **Resumable** | Per-seller-leg idempotency, so a crash resumes where it stopped |
| **Non-fatal on seller failure** | A rejected transfer is recorded unfunded and the fan-out continues |
| **Ordered** | Ledger intent written before money moves, everywhere a crash is possible |

## Single-flight: why a status check is not enough

Two callers reach settlement for the same order **routinely**: the webhook and
the read-path sync fire within milliseconds of each other. The obvious guard
("is the intent already `succeeded`?") cannot stop them: that flag is written at
the *end* of the fan-out, seconds of Stripe latency later, so both callers pass
it and both fan out. In the earlier implementation this decremented inventory
twice: a 10-unit purchase took 20 units of stock.

Keep the status check as a cheap fast path, but make the real exclusion an
atomic leased claim.

```ts
// lib/payments/settlement.ts
export async function settleOrder(
  orderId: string,
  requestId?: string,
): Promise<{ settled: boolean; transfers: number }> {
  const intent = await store.intentByOrder(orderId);
  if (!intent) throw new HttpError(404, "payment_intent_not_found");

  // Cheap short-circuit for an obvious duplicate; the claim rejects this too.
  if (intent.status === "succeeded") return { settled: false, transfers: 0 };

  // Losing the claim is NOT an error; it means someone else is settling. Return
  // the same shape as "already settled", which every caller already reads as
  // "don't cancel, don't retry now".
  if (!(await store.claimSettlement(intent.id))) {
    logger.info({ orderId }, "settle.claim_lost");
    return { settled: false, transfers: 0 };
  }

  try {
    return await runSettlement({ orderId, intent, requestId });
  } catch (err) {
    // Hand the lease back so the provider's retry resumes at once rather than
    // waiting it out. Best-effort: lease expiry is the backstop.
    await store.releaseSettlementClaim(intent.id).catch(() => undefined);
    throw err;
  }
}
```

A **failed claim RPC counts as "not claimed"**. Skipping a settlement is safe:
the webhook and the read-path sync both retry. Proceeding without the claim is
not.

## Per-leg resume, done right

Settlement is not transactional, so a crash mid-fan-out must resume. The naive
resume marker ("does this order have any transfer yet?") strands every seller
after the one that committed before the crash.

The subtler trap, and the one worth stating loudly:

> **Build the resume set from *seller* legs only, never from all transfers.**

A vendor order can produce two rows: a partner-split leg, written first, then the
seller's own leg. If the set is keyed on `vendorOrderId` regardless of
`destinationKind`, a crash in the window between those two writes makes the
resume skip the whole vendor order: the seller is never paid, no escrow hold is
created, and their sub-order stays `pending` forever. The retry sweep cannot
rescue it either, because no unfunded seller leg exists to re-drive.

```ts
async function runSettlement(ctx: {
  orderId: string;
  intent: PaymentIntentRow;
  requestId?: string;
}): Promise<{ settled: boolean; transfers: number }> {
  const { orderId, intent, requestId } = ctx;

  // Resume markers, kept separate per leg kind. Two sets, not one.
  const prior = await store.transfersForIntent(intent.id);
  const settledSellerIds = new Set(
    prior.filter((t) => t.destinationKind === "tenant").map((t) => t.vendorOrderId),
  );
  const settledPartnerIds = new Set(
    prior.filter((t) => t.destinationKind === "partner").map((t) => t.vendorOrderId),
  );

  const vendorOrders = await store.vendorOrdersForOrder(orderId);
  let transfersCreated = 0;

  // Fund every transfer from the buyer's charge, not the platform's available
  // balance. Best-effort: a failed lookup degrades to available-balance
  // behaviour rather than blocking settlement, and a leg rejected that way is
  // picked up by the retry sweep.
  const sourceTransactionId = await resolveTransferSource(intent.externalId, orderId);

  for (const vo of vendorOrders) {
    let sellerAmount = vo.vendorNet;

    const destination = await store.sellerAccountId(vo.tenantId);
    if (!destination) {
      // Seller has not finished onboarding. Record the whole pool as an unfunded
      // claim against them, including any partner cut, since nothing can route
      // until they onboard, and let the retry sweep fund it later.
      if (!settledSellerIds.has(vo.id)) {
        const held = vo.membershipId ? add(sellerAmount, vo.partnerCommissionAmount) : sellerAmount;
        await store.insertTransfer({
          paymentIntentId: intent.id,
          vendorOrderId: vo.id,
          tenantId: vo.tenantId,
          destinationKind: "tenant",
          amount: held,
          currency: vo.currencyCode,
          externalId: null,
          status: "pending",
        });
        transfersCreated++;
      }
      await recordVendorLedger(vo);
      continue;
    }

    // --- Partner split -----------------------------------------------------
    if (vo.membershipId && gt(vo.partnerCommissionAmount, ZERO) && !settledPartnerIds.has(vo.id)) {
      const partnerAccount = await store.partnerAccountFor(vo.membershipId, "stripe");
      if (partnerAccount?.payoutsEnabled && partnerAccount.externalAccountId) {
        const partnerTransfer = await attemptTransfer(
          {
            amount: vo.partnerCommissionAmount,
            currency: vo.currencyCode,
            destinationAccountId: partnerAccount.externalAccountId,
            transferGroup: orderId,
            sourceTransactionId,
            idempotencyKey: `transfer:partner:${vo.id}`,
            metadata: { orderId, vendorOrderId: vo.id, kind: "partner" },
          },
          { tenantId: vo.tenantId, vendorOrderId: vo.id, kind: "partner", requestId },
        );
        await store.insertTransfer({
          paymentIntentId: intent.id,
          vendorOrderId: vo.id,
          tenantId: vo.tenantId,
          destinationKind: "partner",
          membershipId: vo.membershipId,
          amount: vo.partnerCommissionAmount,
          currency: vo.currencyCode,
          externalId: partnerTransfer.externalId,
          status: partnerTransfer.status,
        });
        transfersCreated++;
      } else {
        // No payable partner account, so the cut folds back into the seller. This
        // is a ROUTING decision, not a failure: a partner leg the provider
        // *rejected* stays owed to the partner and must never fold.
        sellerAmount = add(sellerAmount, vo.partnerCommissionAmount);
        await recordAudit({
          action: "partner.payout_blocked",
          tenantId: vo.tenantId,
          payload: { vendorOrderId: vo.id, amount: vo.partnerCommissionAmount },
          requestId,
        });
      }
    }

    // --- Seller leg --------------------------------------------------------
    if (!settledSellerIds.has(vo.id)) {
      const sellerTransfer = await attemptTransfer(
        {
          amount: sellerAmount,
          currency: vo.currencyCode,
          destinationAccountId: destination,
          transferGroup: orderId,
          sourceTransactionId,
          idempotencyKey: `transfer:tenant:${vo.id}`,
          metadata: { orderId, vendorOrderId: vo.id, kind: "tenant" },
        },
        { tenantId: vo.tenantId, vendorOrderId: vo.id, kind: "tenant", requestId },
      );
      await store.insertTransfer({
        paymentIntentId: intent.id,
        vendorOrderId: vo.id,
        tenantId: vo.tenantId,
        destinationKind: "tenant",
        amount: sellerAmount,
        currency: vo.currencyCode,
        externalId: sellerTransfer.externalId,
        status: sellerTransfer.status,
      });
      transfersCreated++;
    }

    await recordVendorLedger(vo);
  }

  await store.setIntentStatus(intent.id, "succeeded");
  await store.setOrderStatus(orderId, "paid");
  await reconcileFeeAfterSettlement(intent.externalId, orderId);
  await finalizeOrderItems(orderId);

  return { settled: true, transfers: transfersCreated };
}
```

Two calls at the end of `runSettlement` are **host hooks**, not part of this
module: `reconcileFeeAfterSettlement` (see
[reconciliation.md](reconciliation.md): it fetches the charge fee and calls
`reconcileGatewayFee`, best-effort, because settlement must never fail on a fee
fetch) and `finalizeOrderItems` (consume inventory, confirm reservations: your
domain, not payments). Both must be best-effort and idempotent: settlement can be
re-entered.

## A rejected transfer must never throw

The single most important guard in this module.

```ts
/**
 * Run a transfer, degrading to an unfunded ledger row instead of throwing.
 * Returns a null `externalId` when the money did not move, so callers can skip
 * anything that needs a funded transfer to act on (a reversal, for instance).
 */
async function attemptTransfer(
  input: CreateTransferInput,
  ctx: { tenantId: string; vendorOrderId: string; kind: "tenant" | "partner"; requestId?: string },
): Promise<{ externalId: string | null; status: "succeeded" | "pending" }> {
  try {
    const transfer = await provider.createTransfer(input);
    return { externalId: transfer.externalId, status: "succeeded" };
  } catch (err) {
    const detail = err instanceof Error ? err.message : String(err);
    logger.error({ err: detail, ...ctx, amount: input.amount }, "settle.transfer_failed");
    await recordAudit({
      action: "payments.transfer_failed",
      tenantId: ctx.tenantId,
      payload: { ...ctx, amount: input.amount, destination: input.destinationAccountId, error: detail },
      requestId: ctx.requestId,
    });
    return { externalId: null, status: "pending" };
  }
}
```

**Why it cannot propagate.** The buyer's money is already captured, so paying the
seller cannot be a precondition for recognising the order as paid. Letting the
rejection propagate stranded orders on `pending` and made the webhook 500 on every
retry, forever, because the common cause (a connected account outside the
platform's region) is not fixed by retrying. The order still reaches `paid`; the
unfunded leg is visible to admins with the provider's own reason in `lastError`,
audited, and re-driven by the retry sweep.

## Funding the transfer from the charge

```ts
/**
 * Stripe's `source_transaction`. Without it the provider draws on the platform's
 * AVAILABLE balance and rejects the transfer while the charge is still settling,
 * which on a young platform account is every transfer: the whole balance sits
 * in `pending` for the settlement delay, so the fan-out never funds anything and
 * the split is invisible on the connected account.
 *
 * Never throws. Money already moved on the buyer's side, so a charge lookup
 * failing must not abort the fan-out; it degrades to the available-balance
 * behaviour, and a leg rejected that way is picked up by the retry sweep.
 */
export async function resolveTransferSource(
  intentExternalId: string | null,
  orderId: string,
): Promise<string | null> {
  if (!intentExternalId) return null;
  try {
    return await provider.resolveTransferSource(intentExternalId);
  } catch (err) {
    logger.warn({ err: String(err), orderId }, "settle.transfer_source_unresolved");
    return null;
  }
}
```

## Escrow and reserves

Escrow holds the seller's net until a delivery or completion window elapses.
Physical goods get a longer window than services; define both constants in one
place so the persisted `release_at` and the release-eligibility check cannot
drift.

The holds, the reserve and the vendor order's status are written for **every**
vendor order on **every** run, not only when a seller leg was just created. Keyed
on the transfer set, a crash after the seller leg was written skipped them on
resume, and a seller who had not onboarded never got them at all, so the retry
sweep funded a transfer that no payout would ever release. Both inserts are
idempotent on a unique key ([store.md](store.md)), so a re-run adds nothing twice.

```ts
async function recordVendorLedger(vo: VendorOrderRow): Promise<void> {
  await createEscrowHolds(vo.id, vo.tenantId, vo.currencyCode);
  await accrueReserve(vo.id, vo.tenantId, vo.currencyCode, vo.vendorNet);
  await store.setVendorOrderStatus(vo.id, "paid");
}
```

The `release-escrow` sweep releases a due hold only once its vendor order has a
funded seller leg (`store.fundedSellerTransfer`). A hold on a leg the retry sweep
has not funded yet waits: releasing it would put money into the payable balance
that is not on the connected account, and the payout for the whole balance
would be rejected.

A task that asks for a payout hold ("paid out after N days") sets these
constants and nothing else: `PHYSICAL_ESCROW_DAYS` to N and
`SERVICE_ESCROW_HOURS` to N times 24, both names kept, neither merged into a new
one. The transfer still runs at settlement, with
`source_transaction`; the hold lives on the payable balance, and the connected
account's manual payout schedule ([connect-accounts.md](connect-accounts.md)) is
what keeps Stripe from paying the seller before it releases. Delaying the
transfer instead is a different money model: the retry sweep, the escrow release
and the payable balance all assume the seller's leg moved at settlement.

```ts
export const PHYSICAL_ESCROW_DAYS = 14;
export const SERVICE_ESCROW_HOURS = 48;

async function createEscrowHolds(
  vendorOrderId: string,
  tenantId: string,
  currency: string,
): Promise<void> {
  const items = await store.orderItemsForVendorOrder(vendorOrderId);
  const now = Date.now();
  for (const it of items) {
    const releaseAt = it.productId
      ? new Date(now + PHYSICAL_ESCROW_DAYS * 86_400_000)
      : new Date(now + SERVICE_ESCROW_HOURS * 3_600_000);
    await store.insertEscrowHold({
      orderItemId: it.id,
      vendorOrderId,
      tenantId,
      amount: it.vendorNet,
      currency,
      status: "held",
      releaseAt: releaseAt.toISOString(),
    });
  }
}

/** Rolling reserve: a % of turnover withheld, for sellers on that schedule. */
export const DEFAULT_RESERVE_PCT = "0.0500";
const RESERVE_RELEASE_DAYS = 90;

async function accrueReserve(
  vendorOrderId: string,
  tenantId: string,
  currency: string,
  vendorNet: string,
): Promise<void> {
  const schedule = await store.payoutSchedule(tenantId);
  if (schedule?.cadence !== "rolling_reserve") return;
  // Quantized at the source, so the payable balance (which subtracts open
  // reserves) stays minor-unit-representable for the payout.
  const amount = roundToMinorUnits(mulRate(vendorNet, DEFAULT_RESERVE_PCT));
  if (isZero(amount)) return;
  await store.holdReserve({
    vendorOrderId,
    tenantId,
    amount,
    currency,
    reservePct: DEFAULT_RESERVE_PCT,
    reason: "rolling_reserve",
    releaseAt: new Date(Date.now() + RESERVE_RELEASE_DAYS * 86_400_000).toISOString(),
  });
}
```

## Reversible headroom

Reversals **stack** on one transfer: the gateway-fee clawback at settlement, then
a refund of the same order later. Stripe rejects a reversal that would push
cumulative reversals past the original transfer amount, and that rejection
fails the whole refund.

```ts
/**
 * How much of a transfer can still be reversed. Clamped at zero, never
 * negative. Money the seller never received (the retained gateway fee) cannot be
 * clawed back a second time on refund; carry that shortfall as a signed
 * `adjustments` row instead.
 */
export async function reversibleHeadroom(
  transferId: string,
  transferAmount: string,
): Promise<string> {
  const reversals = await store.reversalsForTransfer(transferId);
  const reversed = sum(reversals.map((r) => r.amount));
  return max(sub(transferAmount, reversed), ZERO);
}
```

Every reversal call site must cap at this. It is the difference between a refund
that works and one that 400s at Stripe with the customer waiting.

## The payable balance and the payout gate

```ts
export async function computePayableBalance(
  tenantId: string,
  currency: string,
): Promise<string> {
  const [released, reserves, adjustments, scheduled, reversals] = await Promise.all([
    store.sumEscrow(tenantId, currency, "released"),
    store.sumOpenReserves(tenantId, currency),
    store.sumAdjustments(tenantId, currency),
    store.sumPayouts(tenantId, currency, ["scheduled", "in_transit", "paid"]),
    store.sumReversalsForTenant(tenantId, currency),
  ]);
  // released - reserves - reversals + adjustments - scheduled
  return sub(sub(add(sub(released, reserves), adjustments), reversals), scheduled);
}
```

Gate a payout on all of these, and return a distinct error code for each so the
seller is told *which* one is blocking them:

| Check | Error |
|---|---|
| `payments_ready` false, or on compliance hold | `kyc_incomplete` |
| Your own identity verification not approved | `kyc_incomplete` |
| No connected account id | `kyc_incomplete` |
| An open risk hold | `risk_hold_active` |
| Balance below the seller's minimum | `minimum_payout_not_met` |

Round the payout amount **down** to minor units (`roundToMinorUnits(x, 2,
"down")`). A payout that rounds up exceeds the balance that justified it.

## Related

[reconciliation.md](reconciliation.md) re-drives what failed here,
[data-model.md](data-model.md) for the claim function,
[money.md](money.md) for `mulRate`, `roundToMinorUnits`, `sum`.
