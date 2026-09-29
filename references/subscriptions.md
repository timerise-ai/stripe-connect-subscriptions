# Platform subscription billing

Charging your own sellers a recurring platform fee, off-session, against a saved
card. Money flowing *in*, on the same Stripe account the marketplace pays *out*
from.

## Why not Stripe Billing subscriptions

Stripe Subscriptions (products, prices, `subscriptions.create`) is the obvious
choice and is right for many platforms. This module deliberately issues its own
invoices and charges them with plain PaymentIntents. The trade-off:

| | Stripe Subscriptions | Own invoices + PaymentIntents (this) |
|---|---|---|
| Proration, trials, coupons | Built in | You build it |
| Plan catalog | Products/prices at Stripe | In your code |
| Dunning policy | Stripe Smart Retries | Yours, explicit and testable |
| Invoice numbering / PDFs | Built in | You build it |
| Plan tied to app entitlements | Two sources of truth to keep in sync | One |
| Changing a plan | API call + a webhook to sync | A code deploy |

**Choose this shape when the plan drives in-app entitlements**: feature caps,
commission rates, category limits. Keeping the catalog in code means a plan change
ships with the deploy instead of requiring someone to run SQL or click in the
dashboard against each environment. If your plans are purely "how much do they
pay", use Stripe Subscriptions and skip this file.

> If you go the code-owned route, make the code the source of truth and treat any
> plan table as a projection of it. Two writable copies drift, and the drift is
> invisible until a customer is billed the wrong amount.

## The billing instrument

Card entry happens only in Stripe-hosted elements. Persist the PaymentMethod id
plus brand and last4, **never the PAN**. That is the whole PCI SAQ-A posture.

```ts
// lib/payments/subscriptions.ts
/** Get-or-create the seller's platform Stripe Customer. Distinct from their
 *  connected account: this Customer is charged BY the platform. */
export async function ensureBillingCustomer(tenantId: string): Promise<string> {
  const tenant = await store.getTenantBilling(tenantId);
  if (!tenant) throw new HttpError(404, "tenant_not_found");
  if (tenant.billingCustomerId) return tenant.billingCustomerId;

  const customer = await stripeClient().customers.create({
    name: tenant.displayName ?? undefined,
    metadata: { tenantId },
  });
  await store.setBillingCustomer(tenantId, customer.id);
  return customer.id;
}

/** Start collecting a card: a SetupIntent scoped to that Customer. */
export async function createBillingSetupIntent(
  tenantId: string,
): Promise<{ clientSecret: string }> {
  const customerId = await ensureBillingCustomer(tenantId);
  const intent = await stripeClient().setupIntents.create({
    customer: customerId,
    // Required for later off-session charges; without it the card may demand
    // authentication at charge time and every renewal fails.
    usage: "off_session",
    payment_method_types: ["card"],
  });
  if (!intent.client_secret) throw new HttpError(500, "setup_intent_failed");
  return { clientSecret: intent.client_secret };
}

/**
 * Persist a confirmed SetupIntent's payment method. Verifies the intent
 * succeeded AND belongs to this tenant's own Customer. Without that second
 * check a client could bind a PaymentMethod set up for someone else.
 */
export async function confirmBillingPaymentMethod(
  tenantId: string,
  setupIntentId: string,
  ctx: ActorCtx = {},
): Promise<{ brand: string | null; last4: string | null }> {
  const tenant = await store.getTenantBilling(tenantId);
  if (!tenant?.billingCustomerId) {
    throw new HttpError(409, "no_billing_customer", { detail: "create a setup intent first" });
  }

  const intent = await stripeClient().setupIntents.retrieve(setupIntentId, {
    expand: ["payment_method"],
  });
  if (intent.status !== "succeeded") {
    throw new HttpError(409, "setup_intent_not_succeeded", {
      detail: `setup intent status is ${intent.status}`,
    });
  }
  const intentCustomer =
    typeof intent.customer === "string" ? intent.customer : (intent.customer?.id ?? null);
  if (intentCustomer !== tenant.billingCustomerId) {
    throw new HttpError(409, "setup_intent_customer_mismatch");
  }
  const pm = intent.payment_method;
  if (!pm || typeof pm === "string") throw new HttpError(409, "setup_intent_missing_payment_method");

  const brand = pm.card?.brand ?? null;
  const last4 = pm.card?.last4 ?? null;
  await store.setBillingPaymentMethod(tenantId, { id: pm.id, brand, last4 });
  await recordAudit({
    action: "subscriptions.billing_method_updated",
    tenantId,
    payload: { paymentMethodId: pm.id, brand, last4 },
    ...ctx,
  });
  return { brand, last4 };
}
```

Removal detaches at Stripe **best-effort** and always clears the columns. A
detach that fails must not leave a card the seller cannot remove:

```ts
await stripeClient().paymentMethods.detach(pmId).catch(() => undefined);
await store.clearBillingPaymentMethod(tenantId);
```

## Issuing invoices

Idempotent per `(tenant, period)`, and enforce that with a **unique index**, not
just the code check. A concurrent or re-triggered cron pass then hits a
constraint violation instead of double-issuing.

```ts
export async function issueDueInvoices(): Promise<{ tenants: number; issued: number }> {
  const today = new Date().toISOString().slice(0, 10);
  const plans = await store.listPlans();
  const tenants = await store.activeTenants();

  let issued = 0;
  for (const t of tenants) {
    const plan = plans.get(t.subscriptionPlanCode);
    if (!plan || plan.tier === "free") continue; // free tiers carry no fee

    const latest = await store.latestInvoicePeriodEnd(t.id);
    if (latest && latest > today) continue; // current period has not ended

    const periodStart = latest ?? today;
    const periodEnd = addPeriod(periodStart, t.billingCycle);
    const amount = t.billingCycle === "annual" ? plan.annualFee : plan.monthlyFee;

    const row = await store.insertInvoice({
      tenantId: t.id,
      planCode: plan.code,
      periodStart,
      periodEnd,
      amount,
      currencyCode: plan.currencyCode,
      status: "open",
      dueAt: new Date(Date.now() + 7 * 86_400_000).toISOString(),
    });
    if (!row) continue; // unique violation: another pass issued it
    issued++;
    await notify("subscription.invoice_issued", t.id, { amount, periodStart, periodEnd });
  }
  return { tenants: tenants.length, issued };
}

function addPeriod(startIso: string, cycle: "monthly" | "annual"): string {
  const d = new Date(startIso);
  if (cycle === "annual") d.setUTCFullYear(d.getUTCFullYear() + 1);
  else d.setUTCMonth(d.getUTCMonth() + 1);
  return d.toISOString().slice(0, 10);
}
```

**Apply scheduled cancellations before issuing**, or a seller who cancelled last
month gets billed again:

```ts
export async function applyDueSubscriptionCancellations(): Promise<{ cancelled: number }> {
  // active tenants whose subscription_cancel_at has arrived become cancelled
  return store.cancelDueSubscriptions(new Date().toISOString());
}
```

Recording a cancellation *marker* without a job that enforces it is the classic
half-built cancel: the seller sees "cancels on the 30th" and is billed on the
31st.

## Charging off-session

```ts
const MAX_CHARGE_ATTEMPTS = 3;
const RETRY_DELAY_MS = 3 * 24 * 3600_000;

export async function chargeInvoice(
  invoiceId: string,
  ctx: ActorCtx = {},
): Promise<ChargeOutcome> {
  const invoice = await store.getInvoice(invoiceId);
  if (!invoice) throw new HttpError(404, "invoice_not_found");
  if (invoice.status !== "open" && invoice.status !== "past_due") {
    throw new HttpError(409, "invoice_not_chargeable", {
      detail: `invoice status is ${invoice.status}`,
    });
  }
  const tenant = await store.getTenantBilling(invoice.tenantId);
  if (!tenant?.billingCustomerId || !tenant.billingPaymentMethodId) {
    throw new HttpError(409, "no_billing_payment_method");
  }

  try {
    const intent = await stripeClient().paymentIntents.create(
      {
        amount: toMinorUnits(invoice.amount),
        currency: invoice.currencyCode.trim().toLowerCase(),
        customer: tenant.billingCustomerId,
        payment_method: tenant.billingPaymentMethodId,
        off_session: true,
        confirm: true,
        description: `Subscription ${invoice.planCode} (${invoice.periodStart} to ${invoice.periodEnd})`,
        // `purpose` is what the webhook dispatcher routes on; without it, a
        // platform charge would be mistaken for an order payment and settled.
        metadata: {
          purpose: "subscription_invoice",
          invoiceId: invoice.id,
          tenantId: tenant.id,
        },
      },
      // Attempt number in the key, so a deliberate retry is a NEW charge while a
      // double-submitted "Pay now" is not.
      { idempotencyKey: `subinv:${invoice.id}:${invoice.chargeAttempts}` },
    );

    if (intent.status === "succeeded") {
      return await applyChargeSuccess(invoice, tenant, intent.id, ctx);
    }
    return await applyChargeFailure(
      invoice,
      tenant,
      `payment_intent_status:${intent.status}`,
      intent.id,
      ctx,
    );
  } catch (err) {
    if (err instanceof HttpError) throw err;
    // A declined off-session charge surfaces as a Stripe error carrying the
    // created (failed) PaymentIntent. A NETWORK error does not; see below.
    const stripeErr = err as { message?: string; payment_intent?: { id?: string } };
    return await applyChargeFailure(
      invoice,
      tenant,
      stripeErr.message ?? String(err),
      stripeErr.payment_intent?.id ?? null,
      ctx,
    );
  }
}
```

## Dunning

Explicit and testable: retry every 3 days, suspend on the third failure, and let
any success, including a manual retry past the cap, settle and reactivate.

```
open --charge fails--> open (attempts+1, next_charge_at = +3d)
                          |  ... at attempts == 3
                          v
                      past_due  +  tenant.subscription_status = 'past_due'
                          |
                    manual "Pay now" succeeds
                          v
                        paid    +  tenant reactivated to 'active'
```

```ts
async function applyChargeFailure(
  invoice: SubscriptionInvoiceRow,
  tenant: TenantBilling,
  errorMessage: string,
  paymentIntentId: string | null,
  ctx: ActorCtx,
): Promise<ChargeOutcome> {
  const attempts = invoice.chargeAttempts + 1;
  const suspend = attempts >= MAX_CHARGE_ATTEMPTS;

  await store.updateInvoice(invoice.id, {
    chargeAttempts: attempts,
    lastChargeError: errorMessage,
    nextChargeAt: new Date(Date.now() + RETRY_DELAY_MS).toISOString(),
    paymentIntentId: paymentIntentId ?? invoice.paymentIntentId,
    ...(suspend ? { status: "past_due" as const } : {}),
  });

  if (suspend) {
    // Never resurrect a CANCELLED tenant into dunning.
    if (tenant.subscriptionStatus !== "cancelled") {
      await store.setSubscriptionStatus(tenant.id, "past_due");
    }
    await notify("subscription.payment_failed", tenant.id, {
      amount: invoice.amount,
      reason: errorMessage,
      // Attempt number in the dedupe key, or the third failure is swallowed as
      // a duplicate of the first and nobody is told.
      dedupeKey: `subinv-failed:${invoice.id}:${attempts}`,
    });
  }
  return { invoiceId: invoice.id, outcome: "failed", paymentIntentId, attempts, error: errorMessage };
}
```

The sweep charges only invoices still inside the dunning window:

```ts
export async function chargeDueInvoices(): Promise<{
  due: number; charged: number; failed: number; skipped: number;
}> {
  // status = 'open' AND charge_attempts < 3 AND (next_charge_at is null or <= now)
  // Bound the page: each row costs a Stripe round-trip, and an unbounded sweep
  // will eventually exceed the function timeout mid-run.
  const LIMIT = 200;
  const invoices = await store.dueOpenInvoices({ limit: LIMIT });
  if (invoices.length === 0) return { due: 0, charged: 0, failed: 0, skipped: 0 };

  // Only bill tenants that can actually be billed. Checked in bulk rather than
  // per invoice, so one query covers the whole page.
  const billable = new Set<string>();
  for (const t of await store.billingEligibleTenants([
    ...new Set(invoices.map((i) => i.tenantId)),
  ])) {
    billable.add(t.id);
  }

  let charged = 0;
  let failed = 0;
  let skipped = 0;
  for (const inv of invoices) {
    if (!billable.has(inv.tenantId)) {
      skipped++;
      continue;
    }
    try {
      const outcome = await chargeInvoice(inv.id);
      if (outcome.outcome === "paid") charged++;
      else failed++;
    } catch (err) {
      // One bad invoice must not abort the sweep for everyone behind it.
      logger.warn({ err: String(err), invoiceId: inv.id }, "subscription.charge_failed");
      skipped++;
    }
  }

  // Never let the cap pass silently: a truncated sweep that reports success
  // reads as "everything is billed" when it is not.
  if (invoices.length === LIMIT) {
    logger.warn({ limit: LIMIT }, "subscription.charge_sweep_truncated");
  }
  return { due: invoices.length, charged, failed, skipped };
}
```

`past_due` invoices are deliberately **not** re-swept: the attempt cap is the
policy, and the manual "Pay now" is the way out. Say so in the seller's UI, or
they will wait for a retry that never comes.

## The webhook backstop

The synchronous path can be interrupted between Stripe charging the card and your
row being written. `payment_intent.succeeded` / `.payment_failed` carry
`metadata.purpose === "subscription_invoice"`, and this applies the same outcome
idempotently.

```ts
export async function applySubscriptionChargeOutcome(
  pi: Stripe.PaymentIntent,
  requestId?: string | null,
): Promise<{ applied: boolean }> {
  if (pi.metadata?.purpose !== "subscription_invoice") return { applied: false };
  const invoiceId = pi.metadata.invoiceId;
  if (!invoiceId) {
    logger.warn({ paymentIntentId: pi.id }, "subscription.charge_webhook_missing_invoice");
    return { applied: false };
  }

  const invoice = await store.getInvoice(invoiceId);
  if (!invoice) return { applied: false };
  if (invoice.status === "paid") return { applied: false };

  const tenant = await store.getTenantBilling(invoice.tenantId);
  if (!tenant) return { applied: false };

  if (pi.status === "succeeded") {
    await applyChargeSuccess(invoice, tenant, pi.id, { requestId });
    return { applied: true };
  }

  // Failure dedupe. Match on the intent id when we have it; ALSO treat a
  // recorded failure with no intent id as already-applied when the error text
  // matches. The synchronous path only learns the intent id when the SDK error
  // carries one; a network error or timeout does not, so keying on the id
  // alone lets the webhook record a SECOND failure for the same charge, burning
  // two of the three dunning attempts and suspending the seller a cycle early.
  const alreadyRecorded =
    invoice.lastChargeError !== null &&
    (invoice.paymentIntentId === pi.id || invoice.paymentIntentId === null);
  if (alreadyRecorded) {
    // Adopt the id we now know, so a later delivery dedupes on the fast path.
    if (invoice.paymentIntentId === null) {
      await store.updateInvoice(invoice.id, { paymentIntentId: pi.id });
    }
    return { applied: false };
  }

  await applyChargeFailure(
    invoice,
    tenant,
    pi.last_payment_error?.message ?? "payment_failed",
    pi.id,
    { requestId },
  );
  return { applied: true };
}
```

## Tests worth keeping

```ts
// lib/payments/subscriptions.test.ts
it("does not double-count a failure the sync path recorded without an intent id", async () => {
  // The network-error case: attempts already 1, no intent id on the row.
  state.invoice = { chargeAttempts: 1, paymentIntentId: null, lastChargeError: "timeout" };
  await applySubscriptionChargeOutcome(failedIntent({ id: "pi_x" }));
  expect(state.invoice.chargeAttempts).toBe(1);   // not 2
  expect(state.invoice.paymentIntentId).toBe("pi_x"); // adopted for next time
});

it("reactivates a past_due tenant when a manual retry succeeds", async () => {
  state.tenant.subscriptionStatus = "past_due";
  state.invoice = { status: "past_due", chargeAttempts: 3 };
  await chargeInvoice(state.invoice.id);
  expect(state.invoice.status).toBe("paid");
  expect(state.tenant.subscriptionStatus).toBe("active");
});

it("never resurrects a cancelled tenant into dunning", async () => {
  state.tenant.subscriptionStatus = "cancelled";
  state.invoice = { chargeAttempts: 2 };
  await chargeInvoice(state.invoice.id); // fails: third attempt
  expect(state.tenant.subscriptionStatus).toBe("cancelled");
});
```

## Routes

| Route | Purpose |
|---|---|
| `GET /v1/tenants/me/subscription` | Current plan, status, next renewal |
| `POST /v1/tenants/me/subscription:change-plan` | Upgrade / downgrade |
| `POST /v1/tenants/me/subscription:cancel` | Schedule cancellation at period end |
| `GET /v1/tenants/me/subscription/invoices` | Paginated history, keyset by `created_at` |
| `POST /v1/tenants/me/subscription/invoices/{id}/pay` | Manual retry, bypasses the attempt cap |
| `POST /v1/tenants/me/billing/setup-intent` | Start card collection |
| `POST /v1/tenants/me/billing/payment-method` | Confirm the SetupIntent |

## Related

[webhooks.md](webhooks.md) routes the backstop events,
[stripe-adapter.md](stripe-adapter.md) for the client,
[money.md](money.md) for `toMinorUnits`.
