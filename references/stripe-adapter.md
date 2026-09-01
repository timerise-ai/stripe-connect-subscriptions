# Stripe client and adapter

The whole Stripe surface behind one interface, so the rest of the module never
imports `stripe` directly. Two payoffs: the settlement engine is unit-testable
against a fake, and a second rail (PayPal, Adyen) slots in without touching
settlement.

## The client

```ts
// lib/payments/stripe.ts
import Stripe from "stripe";

let cached: Stripe | null = null;

export function stripeClient(): Stripe {
  if (cached) return cached;
  const key = process.env.STRIPE_SECRET_KEY;
  if (!key) throw new Error("stripe_misconfigured: STRIPE_SECRET_KEY is not set");
  cached = new Stripe(key, {
    // Pin the API version explicitly. Left unset, the SDK uses the version its
    // own release pins to — so bumping the `stripe` package silently changes
    // request and response shapes. Pin it, and an SDK upgrade is a deliberate
    // migration instead of a surprise in production.
    //
    // The type is a literal union of the versions YOUR installed SDK knows, so a
    // stale string here is a compile error rather than a runtime surprise. Read
    // the current one from `Stripe.API_VERSION` and change it deliberately.
    apiVersion: "2026-05-27.dahlia",
    appInfo: { name: "your-platform" },
    // Stripe retries idempotently on network failure; 2 is a sane ceiling for a
    // serverless function with a request deadline.
    maxNetworkRetries: 2,
  });
  return cached;
}

/** Test hook only — reset the memoized client between fixtures. */
export function _resetStripeClientCache(): void {
  cached = null;
}
```

> The value shown pins to `stripe@22`. Use whatever `Stripe.API_VERSION` reports
> for your installed SDK, and read the changelog before moving it. The string is
> incidental; the *pin* is the point.

## Dual webhook secrets

Stripe signs **connected-account events with a different endpoint secret** than
platform events, even when both endpoints point at the same URL. Try each
configured secret and accept the first that verifies.

```ts
export function constructWebhookEvent(rawBody: string, signature: string): Stripe.Event {
  const secrets = [
    process.env.STRIPE_WEBHOOK_SECRET,          // endpoint A — platform events
    process.env.STRIPE_CONNECT_WEBHOOK_SECRET,  // endpoint B — account.updated
  ].filter((s): s is string => Boolean(s));
  if (secrets.length === 0) {
    throw new Error("stripe_misconfigured: no webhook signing secret is set");
  }
  const stripe = stripeClient();
  let lastErr: unknown;
  for (const secret of secrets) {
    try {
      return stripe.webhooks.constructEvent(rawBody, signature, secret);
    } catch (err) {
      lastErr = err;
    }
  }
  // Both failed. Also the path a replayed-too-late event takes: constructEvent
  // enforces a 5-minute timestamp tolerance.
  throw new Error(
    `invalid_signature: ${lastErr instanceof Error ? lastErr.message : String(lastErr)}`,
  );
}
```

Because both are tried, swapping the two env vars still works — but two *wrong*
secrets fail every delivery with an identical `401`, which is indistinguishable
from a tampered payload. [operations.md](operations.md) has the triage.

## The provider contract

```ts
// lib/payments/provider.ts
export type CreateIntentInput = {
  orderId: string;
  amount: string;
  currency: string;
  /** Routing key shared by the charge and every transfer under it (== orderId). */
  transferGroup: string;
  customerEmail?: string | null;
  /** Stable, so a retried checkout can never double-charge. */
  idempotencyKey: string;
  metadata?: Record<string, string>;
};

export type ProviderIntent = {
  externalId: string;
  clientSecret?: string | null;
  status: IntentStatus;
  requiresActionState?: "3ds" | "redirect" | "other" | null;
  raw?: unknown;
};

export type RefundInput = {
  intentExternalId: string;
  amount: string;
  currency: string;
  reason?: string;
  idempotencyKey: string;
};
export type ProviderRefund = {
  externalId: string;
  status: "pending" | "succeeded" | "failed";
  raw?: unknown;
};

export type CreateTransferInput = {
  amount: string;
  currency: string;
  destinationAccountId: string;
  transferGroup: string;
  idempotencyKey: string;
  /**
   * The charge this transfer is carved out of. Without it the transfer is drawn
   * from the platform's AVAILABLE balance and is rejected while the charge is
   * still settling — which is every transfer on a young platform account, whose
   * whole balance sits in `pending` for the settlement delay. With it, Stripe
   * accepts the transfer regardless of available balance and the money lands
   * when the charge settles.
   */
  sourceTransactionId?: string | null;
  metadata?: Record<string, string>;
};
export type ProviderTransfer = {
  externalId: string;
  status: "pending" | "succeeded" | "failed";
  raw?: unknown;
};

export type ReverseTransferInput = {
  transferExternalId: string;
  amount: string;
  currency: string;
  reason: string;
  idempotencyKey: string;
};
export type ProviderReversal = { externalId: string; raw?: unknown };

export type CreatePayoutInput = {
  accountId: string;
  amount: string;
  currency: string;
  idempotencyKey: string;
  metadata?: Record<string, string>;
};
export type ProviderPayout = {
  externalId: string;
  status: "scheduled" | "in_transit" | "paid" | "failed";
  raw?: unknown;
};

/** Identifies one settlement leg inside a transfer group. */
export type TransferLegRef = {
  transferGroup: string;
  vendorOrderId: string | null;
  kind: "tenant" | "partner";
};

export interface PaymentProvider {
  readonly name: ProviderName;
  createIntent(input: CreateIntentInput): Promise<ProviderIntent>;
  retrieveIntent(externalId: string): Promise<ProviderIntent>;
  refund(input: RefundInput): Promise<ProviderRefund>;
  /** The provider's processing fee for a settled charge; null until priced. */
  retrieveChargeFee(id: string): Promise<{ amount: string; currency: string } | null>;
  /** Kill an intent so it can never be charged. Throws if already succeeded. */
  cancelIntent(externalId: string): Promise<ProviderIntent>;
}

export interface PayoutProvider {
  readonly name: ProviderName;
  createTransfer(input: CreateTransferInput): Promise<ProviderTransfer>;
  reverseTransfer(input: ReverseTransferInput): Promise<ProviderReversal>;
  createPayout(input: CreatePayoutInput): Promise<ProviderPayout>;
  /** The charge to fund transfers from; null before Stripe creates it. */
  resolveTransferSource(intentExternalId: string): Promise<string | null>;
  /** Does the provider already hold a transfer for this leg? See reconciliation.md. */
  findTransferInGroup(ref: TransferLegRef): Promise<{ externalId: string } | null>;
}

export interface CombinedProvider extends PaymentProvider, PayoutProvider {}
```

## The Stripe implementation

```ts
// lib/payments/providers/stripe.ts
import type Stripe from "stripe";
import { constructWebhookEvent, stripeClient } from "../stripe";
import { fromMinorUnits, toMinorUnits } from "../money";

export class StripeProvider implements CombinedProvider {
  readonly name: ProviderName = "stripe";

  private get stripe(): Stripe {
    return stripeClient();
  }

  async createIntent(input: CreateIntentInput): Promise<ProviderIntent> {
    const intent = await this.stripe.paymentIntents.create(
      {
        amount: toMinorUnits(input.amount),
        currency: input.currency.toLowerCase(),
        // Stamped now, matched by every transfer later. Also what makes
        // `transfers.list({ transfer_group })` a usable recovery tool.
        transfer_group: input.transferGroup,
        // Pin to `card` rather than `automatic_payment_methods` unless your
        // checkout genuinely handles the alternatives: with automatic methods
        // the Payment Element surfaces every method enabled on the account,
        // including bank debits whose delayed settlement breaks the escrow
        // timings this module assumes.
        payment_method_types: ["card"],
        receipt_email: input.customerEmail ?? undefined,
        metadata: { orderId: input.orderId, ...input.metadata },
      },
      { idempotencyKey: input.idempotencyKey },
    );
    return mapIntent(intent);
  }

  async retrieveIntent(externalId: string): Promise<ProviderIntent> {
    return mapIntent(await this.stripe.paymentIntents.retrieve(externalId));
  }

  /**
   * Stripe accepts a cancel in every pre-capture state including
   * `requires_action`, and **rejects it once the intent has succeeded** — so a
   * throw here is the unpaid-order sweep's signal that the buyer finished the
   * 3DS challenge first and the order must be settled, not cancelled.
   */
  async cancelIntent(externalId: string): Promise<ProviderIntent> {
    return mapIntent(await this.stripe.paymentIntents.cancel(externalId));
  }

  /**
   * The processing fee, read off the charge's balance transaction. Denominated
   * in the PLATFORM account's settlement currency, which is not necessarily the
   * order's — the caller must compare before recording (see reconciliation.md).
   * Null while the balance transaction does not exist yet.
   */
  async retrieveChargeFee(
    intentExternalId: string,
  ): Promise<{ amount: string; currency: string } | null> {
    const intent = await this.stripe.paymentIntents.retrieve(intentExternalId, {
      expand: ["latest_charge.balance_transaction"],
    });
    const charge = intent.latest_charge;
    if (!charge || typeof charge === "string") return null;
    const bt = charge.balance_transaction;
    if (!bt || typeof bt === "string") return null;
    return { amount: fromMinorUnits(bt.fee), currency: bt.currency.toUpperCase() };
  }

  async refund(input: RefundInput): Promise<ProviderRefund> {
    const refund = await this.stripe.refunds.create(
      {
        payment_intent: input.intentExternalId,
        amount: toMinorUnits(input.amount),
        reason: stripeRefundReason(input.reason),
        metadata: input.reason ? { reason: input.reason } : undefined,
      },
      { idempotencyKey: input.idempotencyKey },
    );
    return { externalId: refund.id, status: mapRefundStatus(refund.status), raw: refund };
  }

  /** The charge id to fund transfers from. Read off the intent, never stored. */
  async resolveTransferSource(intentExternalId: string): Promise<string | null> {
    const intent = await this.stripe.paymentIntents.retrieve(intentExternalId);
    const charge = intent.latest_charge;
    if (!charge) return null;
    return typeof charge === "string" ? charge : charge.id;
  }

  /**
   * Transfers are listable by `transfer_group`, and settlement stamps
   * `vendorOrderId` + `kind` into each leg's metadata, so the match is exact.
   * One page suffices: a group holds one transfer per seller sub-order plus
   * partner legs. If your carts can exceed 100 legs, paginate — silently
   * missing an existing transfer here means paying it twice.
   */
  async findTransferInGroup(ref: TransferLegRef): Promise<{ externalId: string } | null> {
    const list = await this.stripe.transfers.list({
      transfer_group: ref.transferGroup,
      limit: 100,
    });
    const hit = list.data.find(
      (t) =>
        t.metadata?.kind === ref.kind &&
        (ref.vendorOrderId == null || t.metadata?.vendorOrderId === ref.vendorOrderId),
    );
    return hit ? { externalId: hit.id } : null;
  }

  async createTransfer(input: CreateTransferInput): Promise<ProviderTransfer> {
    const transfer = await this.stripe.transfers.create(
      {
        amount: toMinorUnits(input.amount),
        currency: input.currency.toLowerCase(),
        destination: input.destinationAccountId,
        transfer_group: input.transferGroup,
        ...(input.sourceTransactionId
          ? { source_transaction: input.sourceTransactionId }
          : {}),
        metadata: input.metadata,
      },
      { idempotencyKey: input.idempotencyKey },
    );
    return { externalId: transfer.id, status: "succeeded", raw: transfer };
  }

  async reverseTransfer(input: ReverseTransferInput): Promise<ProviderReversal> {
    const reversal = await this.stripe.transfers.createReversal(
      input.transferExternalId,
      { amount: toMinorUnits(input.amount), metadata: { reason: input.reason } },
      { idempotencyKey: input.idempotencyKey },
    );
    return { externalId: reversal.id, raw: reversal };
  }

  /** Connected-account payout — note `stripeAccount`, not a body field. */
  async createPayout(input: CreatePayoutInput): Promise<ProviderPayout> {
    const payout = await this.stripe.payouts.create(
      {
        amount: toMinorUnits(input.amount),
        currency: input.currency.toLowerCase(),
        metadata: input.metadata,
      },
      { idempotencyKey: input.idempotencyKey, stripeAccount: input.accountId },
    );
    return { externalId: payout.id, status: mapPayoutStatus(payout.status), raw: payout };
  }

  async verifyWebhook(rawBody: string, headers: Headers): Promise<Stripe.Event> {
    const signature = headers.get("stripe-signature");
    if (!signature) throw new Error("invalid_signature: missing stripe-signature");
    return constructWebhookEvent(rawBody, signature);
  }
}
```

## Status mapping

Collapse Stripe's states into the module's five. The mapping is deliberately
lossy in one place and precise in another — both matter.

```ts
function mapIntent(intent: Stripe.PaymentIntent): ProviderIntent {
  return {
    externalId: intent.id,
    clientSecret: intent.client_secret,
    status: mapIntentStatus(intent.status),
    // Set ONLY on Stripe's own `requires_action`. An intent that was never
    // confirmed (`requires_payment_method`) maps to the same status but carries
    // no action state — so "the buyer is mid-3DS" stays distinguishable from
    // "the buyer never started", which decides whether the order may be
    // cancelled. Collapsing these strands or over-cancels orders.
    requiresActionState: intent.status === "requires_action" ? "3ds" : null,
    raw: intent,
  };
}

function mapIntentStatus(status: Stripe.PaymentIntent.Status): IntentStatus {
  switch (status) {
    case "succeeded":
      return "succeeded";
    case "processing":
      return "processing";
    case "canceled":
      return "failed";
    default:
      // requires_payment_method | requires_confirmation | requires_action |
      // requires_capture
      return "requires_action";
  }
}

function mapRefundStatus(status: Stripe.Refund["status"]): ProviderRefund["status"] {
  switch (status) {
    case "succeeded":
      return "succeeded";
    case "failed":
      return "failed";
    default:
      return "pending";
  }
}

function mapPayoutStatus(status: Stripe.Payout["status"]): ProviderPayout["status"] {
  switch (status) {
    case "paid":
      return "paid";
    case "failed":
    case "canceled":
      return "failed";
    default:
      return "in_transit";
  }
}

/** Stripe accepts only three refund reasons; anything else must be omitted. */
function stripeRefundReason(reason?: string): Stripe.RefundCreateParams.Reason | undefined {
  if (reason === "fraudulent" || reason === "duplicate" || reason === "requested_by_customer") {
    return reason;
  }
  return undefined;
}
```

## Idempotency keys

Derive them from stable domain ids — never from a timestamp or a random value,
which defeats the point on the retry that matters.

| Operation | Key | Note |
|---|---|---|
| Checkout intent | `intent:{orderId}` | Survives a double-submitted checkout |
| Seller transfer | `transfer:tenant:{vendorOrderId}` | Re-used by the retry sweep on purpose |
| Partner transfer | `transfer:partner:{vendorOrderId}` | Distinct kind, same sub-order |
| Fee clawback | `fee:{vendorOrderId}` | One clawback per seller sub-order |
| Subscription charge | `subinv:{invoiceId}:{attemptNumber}` | Attempt is in the key so a *deliberate* retry is a new charge |

**Stripe's idempotency window is 24 hours.** Anything re-driven later needs
`findTransferInGroup`, not a key.

## Related

[webhooks.md](webhooks.md) consumes `verifyWebhook`,
[settlement.md](settlement.md) consumes the payout half.
