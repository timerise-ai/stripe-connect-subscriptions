# Webhooks

One endpoint, two signing secrets, three classes of event. The whole design
exists to satisfy one property: **an event must be handled exactly once, even
though Stripe will deliver it more than once and your handler may crash halfway
through.**

## Idempotency: the insert is a claim, not a receipt

The obvious implementation is wrong in a way that loses events permanently:

```ts
// WRONG: drops every event whose first delivery crashed mid-handling.
if (await alreadySeen(event.id)) return ok();
await markSeen(event.id);
await handle(event);
```

If the handler throws after the row was written, Stripe retries, the row exists,
and the retry is swallowed as a duplicate. The event is never handled, silently and
forever.

The fix: the row records *claimed* and *processed* separately, and a duplicate
insert re-reads the row to find out which.

```ts
// lib/payments/webhook-events.ts
export async function recordWebhookEvent(input: {
  provider: string;
  externalEventId: string;
  type: string;
  payload: unknown;
}): Promise<{ duplicate: boolean; id: string | null }> {
  const inserted = await store.insertWebhookEvent(input); // unique (provider, event_id)
  if (inserted.ok) return { duplicate: false, id: inserted.id };
  if (inserted.conflict) {
    const existing = await store.getWebhookEvent(input.provider, input.externalEventId);
    // Delivered before, but the earlier delivery never finished. The provider's
    // retry must re-run the (idempotent) handlers, or the event is lost.
    if (existing && existing.processedAt === null) {
      return { duplicate: false, id: existing.id };
    }
    return { duplicate: true, id: existing?.id ?? null };
  }
  throw new Error(`webhook_record_failed: ${inserted.error}`);
}

/**
 * Flip the processed marker once the handlers finished. Keyed on the NATURAL
 * key: the route holds Stripe's event id, not the row uuid. Passing a provider
 * id to an `id` filter matches nothing and leaves `processed_at` null forever,
 * which quietly re-runs every replay.
 */
export async function markWebhookProcessed(
  provider: string,
  externalEventId: string,
): Promise<void> {
  await store.markWebhookProcessed(provider, externalEventId);
}
```

**This only works because the handlers are themselves idempotent.** Re-running
them is the recovery mechanism, so every handler downstream must tolerate being
called twice. That is what the settlement claim, the atomic fee-status
transitions and the conditional updates elsewhere are for.

## The route

```ts
// app/api/webhooks/stripe/route.ts
export const dynamic = "force-dynamic";

/**
 * Payment and dispute events are signed with the platform-account endpoint
 * secret; `account.updated` is signed with the connected-account one.
 * `constructWebhookEvent` verifies against both.
 */
export const POST = route(async (req, { requestId }) => {
  const signature = req.headers.get("stripe-signature");
  if (!signature) {
    logger.warn({ requestId }, "stripe.webhook.missing_signature");
    throw new HttpError(401, "invalid_signature", { detail: "missing stripe-signature" });
  }

  // The RAW body, before any parsing. `req.json()` re-serializes and the
  // signature will never verify again.
  const rawBody = await req.text();

  let event: Stripe.Event;
  try {
    event = constructWebhookEvent(rawBody, signature);
  } catch (err) {
    // Log explicitly. A typed-error-to-JSON mapper usually does NOT log, so
    // without this a misconfigured signing secret leaves no trace on your side
    // at all; only Stripe's dashboard shows the failures, and you debug blind.
    logger.warn({ err: String(err), requestId }, "stripe.webhook.rejected");
    throw err;
  }

  // One line per delivery, before any handling, so "did the webhook reach this
  // environment?" is answerable from logs alone. `livemode` catches the classic
  // test-key/live-endpoint mix-up.
  logger.info(
    { eventId: event.id, type: event.type, livemode: event.livemode, requestId },
    "stripe.webhook.received",
  );

  const PAYMENT_EVENTS = new Set<string>([
    "payment_intent.succeeded",
    "payment_intent.payment_failed",
    "charge.succeeded",
    "charge.dispute.created",
    "charge.dispute.closed",
  ]);

  if (PAYMENT_EVENTS.has(event.type)) {
    const recorded = await recordWebhookEvent({
      provider: "stripe",
      externalEventId: event.id,
      type: event.type,
      payload: event,
    });
    if (recorded.duplicate) {
      logger.info({ eventId: event.id, requestId }, "stripe.webhook.duplicate");
      return jsonOk({ ok: true, duplicate: true }, { headers: { "Idempotent-Replay": "true" } });
    }

    await dispatchPaymentEvent(event, requestId);

    // Only after the handlers returned. A throw above leaves processed_at null
    // and Stripe's retry re-runs them.
    await markWebhookProcessed("stripe", event.id);
    logger.info({ eventId: event.id, type: event.type, requestId }, "stripe.webhook.processed");
    return jsonOk({ ok: true });
  }

  if (event.type === "account.updated") {
    await applyAccountUpdate(event.data.object as Stripe.Account, requestId);
    return jsonOk({ ok: true });
  }

  if (event.type === "payment_method.detached") {
    await removeSavedPaymentMethod((event.data.object as Stripe.PaymentMethod).id);
    return jsonOk({ ok: true });
  }
  if (event.type === "payment_method.automatically_updated") {
    // A reissued or compromised card was auto-updated by the network. Flag it so
    // the customer is prompted, rather than discovering it at the next charge.
    await flagPaymentMethod((event.data.object as Stripe.PaymentMethod).id);
    return jsonOk({ ok: true });
  }

  // ACK anything else, or Stripe retries it for days.
  return jsonOk({ ok: true, ignored: event.type });
});
```

`account.updated` deliberately skips the `webhook_events` table: its handler is
naturally idempotent (a conditional flag flip) and Connect emits it often enough
that the table would fill with noise. Every event whose handler is *not*
naturally idempotent must go through the claim.

## Dispatching

The one place the marketplace and subscription flows meet. They are told apart by
`metadata.purpose`, stamped when the intent was created.

```ts
async function dispatchPaymentEvent(event: Stripe.Event, requestId: string): Promise<void> {
  switch (event.type) {
    case "payment_intent.succeeded": {
      const pi = event.data.object as Stripe.PaymentIntent;
      // A platform charge, not an order payment: settle the invoice and stop.
      if (pi.metadata?.purpose === "subscription_invoice") {
        await applySubscriptionChargeOutcome(pi, requestId);
        return;
      }
      const resolved = await store.intentByExternalId(pi.id);
      if (!resolved) {
        // Not ours (another integration on the same account, or a stale test
        // event). Warn and ACK; never throw, or Stripe retries for days.
        logger.warn({ externalIntentId: pi.id }, "webhook.intent_not_found");
        return;
      }
      await settleOrder(resolved.orderId, requestId);
      return;
    }

    case "payment_intent.payment_failed": {
      const pi = event.data.object as Stripe.PaymentIntent;
      if (pi.metadata?.purpose === "subscription_invoice") {
        await applySubscriptionChargeOutcome(pi, requestId);
        return;
      }
      await handlePaymentFailed(pi.id);
      return;
    }

    case "charge.succeeded": {
      // Backstop only for the gateway fee; settlement fetches it itself. See
      // reconciliation.md for why an early arrival must defer.
      const charge = event.data.object as Stripe.Charge;
      const fee = await resolveChargeFee(charge);
      if (fee && charge.payment_intent) {
        await reconcileGatewayFee(charge.payment_intent as string, fee.amount, fee.currency);
      }
      return;
    }

    case "charge.dispute.created": {
      const dispute = event.data.object as Stripe.Dispute;
      await handleChargebackOpened({
        providerDisputeId: dispute.id,
        externalIntentId: dispute.payment_intent as string,
        amount: fromMinorUnits(dispute.amount),
        currency: dispute.currency.toUpperCase(),
        reasonCode: dispute.reason,
        dueBy: dispute.evidence_details?.due_by
          ? new Date(dispute.evidence_details.due_by * 1000).toISOString()
          : null,
        requestId,
      });
      return;
    }

    case "charge.dispute.closed": {
      const dispute = event.data.object as Stripe.Dispute;
      await handleChargebackResolved({
        providerDisputeId: dispute.id,
        outcome:
          dispute.status === "won" ? "won" : dispute.status === "lost" ? "lost" : "accepted",
        requestId,
      });
      return;
    }
  }
}

/** The balance transaction may arrive as an id; retrieve it if so. */
async function resolveChargeFee(
  charge: Stripe.Charge,
): Promise<{ amount: string; currency: string } | null> {
  let bt = charge.balance_transaction;
  if (typeof bt === "string") bt = await stripeClient().balanceTransactions.retrieve(bt);
  if (!bt || typeof bt === "string") return null;
  return { amount: fromMinorUnits(bt.fee), currency: bt.currency.toUpperCase() };
}
```

## The webhook is not the only trigger

A webhook can be delayed, dropped, or simply never registered for the
environment. When that happens a charged order sits unpaid: the seller cannot
fulfil it, and the unpaid-order sweep would cancel an order the buyer already
paid for.

So **read paths ask Stripe directly**. The buyer's order page and the seller's
order detail call this; it is safe because `settleOrder` short-circuits once the
intent is locally `succeeded`.

```ts
// lib/payments/sync.ts
export type PaymentSyncResult =
  | { settled: true }
  | {
      settled: false;
      reason:
        | "no_intent"          // offline order, or nothing at the provider
        | "settled_locally"    // the webhook won the race
        | "not_succeeded"      // confirmed not charged
        | "awaiting_action"    // the buyer is mid-3DS
        | "processing"         // charge in flight, verdict pending
        | "check_failed";      // we could not reach Stripe
    };

export async function syncOrderPaymentFromProvider(
  orderId: string,
  requestId?: string,
): Promise<PaymentSyncResult> {
  const intent = await store.intentByOrder(orderId);
  if (!intent?.externalId) return { settled: false, reason: "no_intent" };
  // If the order still reads `pending` here, settlement crashed between the
  // intent flip and the order update: charged, so never safe to cancel.
  if (intent.status === "succeeded") return { settled: false, reason: "settled_locally" };

  try {
    const remote = await provider.retrieveIntent(intent.externalId);
    if (remote.status !== "succeeded") {
      // Stripe documents `processing` as cancellable only "in rare cases", so a
      // cancel here would release stock against money very likely to land.
      if (remote.status === "processing") return { settled: false, reason: "processing" };
      // Only Stripe's own `requires_action` sets this, so an intent that was
      // never confirmed does NOT land here; see the adapter's mapIntent.
      if (remote.requiresActionState) return { settled: false, reason: "awaiting_action" };
      return { settled: false, reason: "not_succeeded" };
    }
    const { settled } = await settleOrder(orderId, requestId);
    return settled ? { settled: true } : { settled: false, reason: "settled_locally" };
  } catch (err) {
    logger.warn({ err: String(err), orderId }, "payments.sync_failed");
    return { settled: false, reason: "check_failed" };
  }
}
```

**Only `no_intent` and `not_succeeded` authorise cancelling the order.** Every
other reason means the buyer may have been, or is about to be, charged. Getting
this table wrong cancels paid orders: this is the single most consequential
enum in the module.

## Events to register

| Endpoint | Events | Secret |
|---|---|---|
| **A: platform account** | `payment_intent.succeeded`, `payment_intent.payment_failed`, `charge.succeeded`, `charge.dispute.created`, `charge.dispute.closed`, `payment_method.detached`, `payment_method.automatically_updated` | `STRIPE_WEBHOOK_SECRET` |
| **B: connected accounts** | `account.updated` | `STRIPE_CONNECT_WEBHOOK_SECRET` |

Same URL, two endpoint registrations, two secrets. Anything else is ACKed and
ignored, so extra subscriptions only waste deliveries.

## Related

[stripe-adapter.md](stripe-adapter.md) for signature verification,
[settlement.md](settlement.md) for what `payment_intent.succeeded` triggers,
[operations.md](operations.md) for registering and testing the endpoints.
