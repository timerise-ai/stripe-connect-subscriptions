# Connect accounts

Creating connected accounts, tracking their capabilities, and the region rule
that decides whether any of it will ever work.

## Read this first: the region rule

**A Stripe transfer can only reach a connected account in a country your
platform's country is allowed to transfer to.** Cross-border transfers on the
payments balance work only *between* the United States, Canada, the United
Kingdom, the EEA and Switzerland. A platform anywhere else — Hong Kong,
Singapore, Malaysia, Australia, Japan, Brazil — can transfer **only to connected
accounts in its own country**.

It is a **transfer-time** rule, not an onboarding-time rule. That is the whole
problem:

| Step | Wrong-country seller | Note |
|---|---|---|
| Create the connected account | ✅ succeeds | An HK platform may create a US account |
| Seller completes onboarding | ✅ succeeds | KYC passes normally |
| `account.updated` → charges + payouts enabled | ✅ both true | Your app marks the seller ready |
| Buyer checkout → charge | ✅ succeeds | **The buyer's money is taken** |
| `transfers.create` at settlement | ❌ rejected | `Funds can't be sent to accounts located in US because it's restricted outside of your platform's region` |
| Order status | ✅ reaches `paid` | Leg recorded unfunded; **the seller is never paid** |

Nothing warns you until real money has moved, and the seller looks perfectly
healthy right up to the moment it cannot be paid.

### Check it before you write any code

```bash
# Your platform account's country
curl -s https://api.stripe.com/v1/account -u "$STRIPE_SECRET_KEY:" \
  | jq '.country, .default_currency'

# The countries that country may transfer to
curl -s https://api.stripe.com/v1/country_specs/HK -u "$STRIPE_SECRET_KEY:" \
  | jq '.supported_transfer_countries'
```

`US` and `GB` each return the ~37-country corridor. `HK` returns `["HK"]`, `SG`
returns `["SG"]`. **A one-entry list means there is no way to pay a seller abroad
on the standard rails.**

### Your options, all expensive to reverse

| # | Option | Cost |
|---|---|---|
| A | Keep the platform where it is; onboard sellers **in the same country** | A code change to the country list, plus re-onboarding |
| B | Move the platform into the corridor (US/CA/GB/EEA/CH) | **A new platform account** — country is immutable. New keys, re-onboard every seller, re-register webhooks, and a legal entity in that country. 0.25% cross-border fee (waived within the EEA and UK↔EEA) |
| C | Ask Stripe for Cross-border / Global payouts | Not self-serve outside the corridor; commercial negotiation, unknown lead time. Unavailable on a recipient service agreement |

### Constrain the country picker to the answer

The failure mode in the earlier implementation was mundane: the onboarding
country picker defaulted to `US` and did not even offer the platform's own
country, so every tester accepted the default and every account was unpayable.

```ts
// lib/payments/stripe-countries.ts
/**
 * Countries a seller may open a connected account in. Keep this equal to
 * `country_specs/<platform country>.supported_transfer_countries` and the two
 * can never drift into an unpayable seller.
 *
 * PAYOUT ACCOUNTS ONLY. This list answers "where can this seller open a Stripe
 * account", never "where is this address". Anything postal — a service address,
 * a shipping address, an identity document — must use the full ISO-3166 list, or
 * you strand sellers in the ~180 countries Stripe does not cover here.
 *
 * Pure data, no imports — safe to import from a client component.
 */
export const STRIPE_COUNTRIES: Array<{ value: string; label: string }> = [
  { value: "US", label: "United States" },
  { value: "GB", label: "United Kingdom" },
  { value: "CA", label: "Canada" },
  // … the corridor, or your single country
];

export const DEFAULT_STRIPE_COUNTRY = "US";
```

### A wrong-country account cannot be fixed

**A connected account's country is immutable after creation** — Standard, Express
and Custom alike. No patch, no support ticket, no migration. To fix a seller:

1. Clear their stored `stripe_account_id` (and any partner account row), so your
   code creates a fresh account instead of reusing the old id.
2. Have them run onboarding again, picking the correct country.
3. Leave the old `acct_…` dormant (test mode) or reject it (live).

## Creating the account

Accounts v2 (`POST /v2/core/accounts`) with an Express dashboard. v2 accounts stay
**v1-interoperable**: the `acct_…` id works with v1 PaymentIntents and transfers,
and — critically — still emits the classic `account.updated` Connect event, so
the webhook receiver below is unchanged by the v2 migration.

```ts
// lib/payments/onboarding.ts
import { stripeClient } from "./stripe";

export type StripeOnboardingLink = { accountId: string; url: string };

/**
 * Create (or reuse) an Express connected account and return a fresh hosted
 * onboarding link. The platform is `fees_collector` and `losses_collector`,
 * which is what keeps it the platform of record under separate charges and
 * transfers.
 *
 * Account links are single-use and short-lived — always mint a new one rather
 * than storing the URL.
 */
export async function createExpressOnboardingLink(opts: {
  existingAccountId?: string | null;
  /** ISO-3166 alpha-2. Must be in STRIPE_COUNTRIES — see the region rule above. */
  country: string;
  email?: string | null;
  refreshUrl: string;
  returnUrl: string;
}): Promise<StripeOnboardingLink> {
  const stripe = stripeClient();

  let accountId = opts.existingAccountId ?? undefined;
  if (!accountId) {
    const account = await stripe.v2.core.accounts.create({
      contact_email: opts.email ?? undefined,
      dashboard: "express",
      identity: { country: opts.country },
      defaults: {
        responsibilities: {
          fees_collector: "application",
          losses_collector: "application",
        },
      },
      configuration: {
        merchant: { capabilities: { card_payments: { requested: true } } },
        recipient: {
          capabilities: { stripe_balance: { stripe_transfers: { requested: true } } },
        },
      },
    });
    accountId = account.id;
  }

  const link = await stripe.v2.core.accountLinks.create({
    account: accountId,
    use_case: {
      type: "account_onboarding",
      account_onboarding: {
        configurations: ["merchant", "recipient"],
        refresh_url: opts.refreshUrl,
        return_url: opts.returnUrl,
      },
    },
  });

  return { accountId, url: link.url };
}
```

**Persist `accountId` immediately on first creation**, before returning the link.
The webhook resolves the seller from `account.updated` by that id alone, and a
seller who abandons onboarding must resume into the *same* account rather than
accumulating orphans.

## The onboarding route

Structure travels; the auth guard and validation are seams.

```ts
// app/api/v1/onboarding/payments/stripe/route.ts
export const dynamic = "force-dynamic";

export const POST = route(async (req, { requestId }) => {
  const { user, claims } = await requireSession();            // ← host's auth seam
  const input = await parseBody(req, Body);                   // ← host's validation seam

  // Authorize BEFORE touching the service-role client. Only the seller's own
  // owner/admin may onboard its payment account.
  if (
    claims.tenant_id !== input.tenantId ||
    (claims.tenant_role !== "owner" && claims.tenant_role !== "admin")
  ) {
    throw new HttpError(403, "forbidden", { detail: "must be tenant owner/admin" });
  }

  const tenant = await store.getTenantAccounts(input.tenantId);
  if (!tenant) throw new HttpError(404, "tenant_not_found");

  const base = siteBaseUrl();
  const { accountId, url } = await createExpressOnboardingLink({
    existingAccountId: tenant.stripeAccountId,
    country: input.country,
    email: user.email,
    refreshUrl: `${base}/dashboard/onboarding?stripe=refresh`,
    returnUrl: `${base}/dashboard/onboarding?stripe=return`,
  });

  if (!tenant.stripeAccountId) {
    await store.setTenantStripeAccount(input.tenantId, accountId);
  }

  await recordAudit({
    action: "payments.stripe.onboarding_started",
    tenantId: input.tenantId,
    payload: { accountId, reused: Boolean(tenant.stripeAccountId) },
    requestId,
    actorUserId: user.id,
  });

  return jsonOk({ accountId, onboardingUrl: url }, { status: 201 });
});
```

`refreshUrl` is where Stripe sends a seller whose link expired — it must mint a
new link, so point it back at the same flow, not at a static page.

## `account.updated`

The only signal that a seller can actually trade. Both flags must be true.

```ts
export async function applyAccountUpdate(
  account: Stripe.Account,
  requestId?: string,
): Promise<{ handled: "partner" | "tenant" | "unknown" }> {
  const chargesEnabled = account.charges_enabled === true;
  const payoutsEnabled = account.payouts_enabled === true;

  // Partner (per-membership) accounts share this event. Reconcile them FIRST,
  // and before any `ready` short-circuit, so a capability being *pulled* is
  // reflected too — that is the case the short-circuit would swallow.
  const partner = await store.applyPartnerAccountUpdate(
    account.id,
    chargesEnabled,
    payoutsEnabled,
  );
  if (partner) return { handled: "partner" };

  if (!(chargesEnabled && payoutsEnabled)) return { handled: "unknown" };

  const tenant = await store.getTenantByStripeAccount(account.id);
  if (!tenant) return { handled: "unknown" };       // not ours — ACK and move on
  if (tenant.paymentsReady) return { handled: "tenant" };  // duplicate delivery

  await store.setPaymentsReady(tenant.id, true);
  await recordAudit({
    action: "payments.stripe.account_ready",
    tenantId: tenant.id,
    payload: { accountId: account.id },
    requestId,
  });
  return { handled: "tenant" };
}
```

### Capability state, for per-membership partner accounts

Only assert a KYC status when capabilities cross a **meaningful boundary** —
otherwise a seller still working through onboarding flips to a scary
`restricted` the moment Stripe emits an interim update.

```ts
let nextKyc = prior.kycStatus;
if (chargesEnabled && payoutsEnabled) {
  nextKyc = "verified";
} else if (prior.kycStatus === "verified") {
  // A previously-good account losing a capability is a real regression.
  nextKyc = "restricted";
}
// A still-`pending` account keeps `pending`. Do not "improve" this.
```

## What gates what

Separate these three; conflating them is how sellers get stuck.

| Gate | Controls | Signal |
|---|---|---|
| `charges_enabled` | Can buyers pay for this seller's items | `account.updated` |
| `payouts_enabled` | Can money leave to their bank | `account.updated` |
| Your own identity/KYC check | Whether escrowed funds may be **released** | Your KYC provider |

A defensible posture, used in the earlier implementation and worth copying, is
**deferred KYC**: completing Stripe onboarding alone lets a seller list and sell
immediately, while your own identity verification gates *payouts*. Sellers get
moving on day one; funds stay escrowed until you actually know who they are.

## Related

[webhooks.md](webhooks.md) delivers `account.updated`,
[settlement.md](settlement.md) is where a wrong-country account fails,
[operations.md](operations.md) has the setup checklist.
