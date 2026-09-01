# Reconciliation

Two convergent sweeps that fix what settlement could not finish: the **transfer
retry**, which funds legs the provider rejected, and the **gateway-fee
reconcile**, which records Stripe's processing fee and claws it back off the
seller. Both are designed so that running them repeatedly is always safe and
never doubles anything.

## Transfer retry

`attemptTransfer` degrades a rejected transfer to an unfunded row so the buyer's
paid order is never held hostage. Without a sweep, nothing ever re-attempts those
legs — which is a split payment whose seller half silently never lands: the charge
succeeds, the transfer group is stamped on it, and the connected account shows
nothing.

Most rejections are **fixable, not permanent**:

- the seller had not finished onboarding when their first order settled;
- a partner's payout account was activated afterwards;
- the platform's available balance did not cover it (largely pre-empted by
  `source_transaction`, still reachable);
- a provider blip, rate limit, or timeout;
- an account/region misconfiguration corrected at the account level.

```ts
/** Rows per run. Each costs 2–4 provider round-trips — size to your timeout. */
const SWEEP_ROW_LIMIT = 100;
/** 10 min doubling per attempt, capped at 24h. */
const RETRY_BASE_MS = 10 * 60_000;
const RETRY_MAX_MS = 24 * 3_600_000;

export type TransferRetrySummary = { funded: number; deferred: number; skipped: number };

export async function retryUnfundedTransfers(requestId?: string): Promise<TransferRetrySummary> {
  const due = await store.dueUnfundedTransfers(SWEEP_ROW_LIMIT).catch((err) => {
    // A failed read is never an absence. Throw rather than reporting "0 legs to
    // retry" and letting a broken queue look healthy.
    logger.error({ err: String(err) }, "transfer_retry.scan_failed");
    throw err;
  });

  const summary: TransferRetrySummary = { funded: 0, deferred: 0, skipped: 0 };
  for (const leg of due) {
    summary[await retryLeg(leg, requestId)]++;
  }
  if (summary.funded || summary.deferred) logger.info({ ...summary }, "transfer_retry.swept");
  return summary;
}
```

### The claim *is* the backoff

One conditional update does three jobs: it excludes concurrent runners, it
schedules the next attempt, and it survives a crash mid-provider-call — the leg
is simply left correctly scheduled rather than pinned as claimed forever.

```ts
async function retryLeg(
  leg: TransferRow,
  requestId?: string,
): Promise<keyof TransferRetrySummary> {
  const attempts = leg.retryAttempts;
  const claimed = await store.claimTransferRetry(leg.id, {
    attempts: attempts + 1,
    nextAttemptAt: new Date(
      Date.now() + Math.min(RETRY_BASE_MS * 2 ** attempts, RETRY_MAX_MS),
    ).toISOString(),
  });
  // Matched zero rows: another runner owns it, or it no longer qualifies.
  if (!claimed) return "skipped";

  const intent = await store.intentById(leg.paymentIntentId);
  if (intent?.status !== "succeeded") {
    // Nothing to fund a transfer against. Defer, do not mark resolved.
    await store.recordTransferFailure(leg.id, "payment intent is not settled");
    return "deferred";
  }

  // Re-read the destination EVERY pass — the whole point of the retry is that a
  // seller or partner may have onboarded since the order settled.
  const destination = await resolveDestination(leg);
  if (!destination) {
    await store.recordTransferFailure(leg.id, "no payout account for this destination");
    return "deferred";
  }

  // Money may ALREADY have moved: attemptTransfer records a leg unfunded on any
  // throw, including a timeout raised after the provider committed. The
  // idempotency key covers a re-send inside Stripe's 24h window; beyond it, this
  // lookup is the only guard against paying twice. Best-effort — a failed lookup
  // falls through to the key rather than blocking the repair.
  const existing = await findExistingTransfer(intent.orderId, leg);
  if (existing) {
    await store.markTransferFunded(leg.id, existing.externalId);
    logger.info({ transferId: leg.id, externalId: existing.externalId }, "transfer_retry.adopted");
    return "funded";
  }

  const sourceTransactionId = await resolveTransferSource(intent.externalId, intent.orderId);

  try {
    const transfer = await provider.createTransfer({
      amount: leg.amount,
      currency: leg.currencyCode,
      destinationAccountId: destination,
      transferGroup: intent.orderId,
      sourceTransactionId,
      // The SETTLEMENT key, deliberately — inside Stripe's window a re-send
      // resolves to the original transfer instead of creating a second one.
      idempotencyKey: leg.vendorOrderId
        ? `transfer:${leg.destinationKind}:${leg.vendorOrderId}`
        : `transfer:retry:${leg.id}`,
      metadata: {
        orderId: intent.orderId,
        ...(leg.vendorOrderId ? { vendorOrderId: leg.vendorOrderId } : {}),
        kind: leg.destinationKind,
      },
    });
    await store.markTransferFunded(leg.id, transfer.externalId);
    await recordAudit({
      action: "payments.transfer_retried",
      tenantId: leg.tenantId,
      payload: { amount: leg.amount, externalId: transfer.externalId, attempt: attempts + 1 },
      requestId,
    });
    return "funded";
  } catch (err) {
    const detail = err instanceof Error ? err.message : String(err);
    // The backoff was already applied by the claim; this only carries the reason
    // forward. Truncate — provider messages can be long.
    await store.recordTransferFailure(leg.id, detail.slice(0, 500));
    logger.warn({ transferId: leg.id, err: detail }, "transfer_retry.failed");
    return "deferred";
  }
}

async function findExistingTransfer(
  orderId: string,
  leg: TransferRow,
): Promise<{ externalId: string } | null> {
  try {
    return await provider.findTransferInGroup({
      transferGroup: orderId,
      vendorOrderId: leg.vendorOrderId,
      kind: leg.destinationKind,
    });
  } catch (err) {
    logger.warn({ err: String(err), transferId: leg.id }, "transfer_retry.lookup_failed");
    return null;
  }
}
```

`lastError` on the row is the operator's fastest diagnostic: it distinguishes a
region block from a balance shortfall from a missing payout account without
digging through audit events. Surface it in the admin list.

## Gateway-fee reconciliation

Stripe's processing fee comes out of the platform's balance, but economically the
seller bears it. So the fee must be **recorded** against each seller sub-order and
**clawed back** out of what the seller received.

Clawing back means a real `transfer_reversals` row against the funded seller
transfer. If no funded transfer exists, degrade to a signed `adjustments` debit —
but note the difference: an adjustment reduces the *recorded* payable balance
while the cash stays in the seller's Stripe balance, so their Stripe payout will
exceed your recorded net. Prefer the reversal, which is why timing matters below.

### The state machine

Durable, not one-shot. Each seller sub-order carries its own status.

```
pending ──(fee known, share claimed)──▶ allocated ──(clawback done)──▶ recorded
   │                                                                     ▲
   └──(fee/order currency mismatch)──▶ manual_review    allocated rows ───┘
                                                        whose clawback failed
```

| Transition | Mechanism | Why |
|---|---|---|
| `pending → allocated` | Atomic conditional update | Settlement, webhook and sweep race routinely; only one may claim a row |
| Share persisted **before** money moves | Write, then reverse | A crash leaves a resumable `allocated`, not a lost fee |
| `allocated → recorded` | After the clawback | Convergent: re-derives what is still owed from the ledger, so a resume never double-debits |
| `→ manual_review` | Currency mismatch | The sweep stops retrying a case that can never succeed on its own |

### Ordering: why an early `charge.succeeded` must defer

`charge.succeeded` usually outruns settlement. If the fee is reconciled then,
there is no funded seller transfer yet, so the clawback can only degrade to a
ledger-only adjustment — leaving the fee cash in the seller's Stripe balance and
making their Stripe payout exceed the recorded net.

So: **settlement is the primary writer** (it fetches the fee itself, right after
the transfers are funded), and an early webhook defers. The rows stay `pending`
and the sweep retries even if the settlement-time fetch failed.

```ts
export async function reconcileGatewayFee(
  externalIntentId: string,
  feeAmount: string,
  currency: string,
): Promise<void> {
  const resolved = await store.intentByExternalId(externalIntentId);
  if (!resolved) return;

  // Defer to settlement, which reconciles right after funding the transfers.
  if (resolved.status !== "succeeded") {
    logger.info({ externalIntentId }, "gateway_fee.deferred_to_settlement");
    return;
  }

  const rows = await store.feeRowsForOrder(resolved.orderId);
  if (rows.length === 0) return;
  const open = rows.filter(
    (v) => v.gatewayFeeStatus === "pending" || v.gatewayFeeStatus === "allocated",
  );
  if (open.length === 0) return;

  // The fee is denominated in the PLATFORM account's settlement currency. If the
  // order was presented in another, the two amounts do not share a unit and
  // there is no FX anywhere in this ledger. Recording the raw figure would
  // corrupt the row and reverse the wrong number off the transfer. Park it.
  if (!isZero(feeAmount) && rows.some((v) => v.currencyCode !== currency)) {
    logger.error(
      { orderId: resolved.orderId, feeCurrency: currency },
      "gateway_fee.currency_mismatch",
    );
    for (const v of open) await store.setFeeStatus(v.id, "manual_review");
    return;
  }

  // Deterministic: same fee, same weights → same split, so a partially
  // reconciled order re-derives identical shares for rows still pending.
  const portions = distribute(
    feeAmount,
    rows.map((v) => Number(v.vendorNet)),
  );

  for (let i = 0; i < rows.length; i++) {
    const v = rows[i] as (typeof rows)[number];
    if (v.gatewayFeeStatus === "recorded" || v.gatewayFeeStatus === "manual_review") continue;

    let portion: string;
    if (v.gatewayFeeStatus === "allocated") {
      // Crash-resume: the share was locked at claim time; only the clawback is
      // outstanding.
      portion = v.gatewayFeeAmount;
    } else {
      portion = portions[i] as string;
      if (isZero(portion)) {
        // A genuinely zero share is still a reconciliation RESULT — record it,
        // or the dashboard reads "pending settlement" forever and the sweep
        // revisits the row on every pass.
        await store.setFeeStatus(v.id, "recorded");
        continue;
      }
      // Atomic claim. Only the update that flips pending → allocated proceeds.
      const claimed = await store.claimFeeAllocation(v.id, portion);
      if (!claimed) continue;
    }

    await clawBackFee(v, portion);
    await store.setFeeStatus(v.id, "recorded");
  }
}
```

### The clawback

```ts
async function clawBackFee(row: FeeRow, portion: string): Promise<void> {
  const transfer = await store.fundedSellerTransfer(row.id);

  if (transfer?.externalId) {
    // Cap at the headroom — a later refund also reverses this transfer, and
    // Stripe rejects cumulative reversals past the original amount.
    const headroom = await reversibleHeadroom(transfer.id, transfer.amount);
    const amount = min(portion, headroom);
    if (isZero(amount)) return;

    const rev = await provider.reverseTransfer({
      transferExternalId: transfer.externalId,
      amount,
      currency: row.currencyCode,
      reason: "gateway_fee",
      idempotencyKey: `fee:${row.id}`,
    });
    await store.insertTransferReversal({
      transferId: transfer.id,
      amount,
      currency: row.currencyCode,
      reason: "gateway_fee",
      externalId: rev.externalId,
    });
    return;
  }

  // No funded transfer to reverse (seller not onboarded, or a held leg). Debit
  // the ledger instead. Keyed by a stable reason so a resume is convergent.
  await store.createAdjustment({
    tenantId: row.tenantId,
    amount: negate(portion),
    currency: row.currencyCode,
    reason: `gateway_fee:${row.id}`,
  });
}
```

## The sweeps, and why they exist

| Sweep | Cadence | Fixes |
|---|---|---|
| `retry-transfers` | every 20 min | Legs the provider rejected |
| `sweep-gateway-fees` | hourly | Rows stuck `pending`/`allocated` — a missing `charge.succeeded` registration, a transient error |
| `release-escrow` | hourly | Holds whose `release_at` has passed |
| `reconcile-orphan-payments` | hourly | Charges with no settled order |

**Never let a sweep cap silently.** If a run is bounded (`SWEEP_ROW_LIMIT`), log
what was left behind. A truncated sweep that reports success reads as "everything
is reconciled" when it is not.

## Related

[settlement.md](settlement.md) for `reversibleHeadroom` and `attemptTransfer`,
[money.md](money.md) for `distribute`,
[operations.md](operations.md) for triggering a sweep by hand.
