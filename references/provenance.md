# Provenance

Written by the engineers who have shipped this module. The earlier
implementation it was audited against was the payments and billing module of a
multi-vendor marketplace running Stripe Connect (separate charges and
transfers) plus platform subscription billing on one Stripe account, with unit
tests, an operations runbook, and an incident history behind the comments.

The templates here are **hardened, not faithful**. Defects found during the audit
are fixed in the code you see, and every deviation is recorded below. Where a
claim could not be verified, it says so.

## Fixed in the templates

### 1. A crash between two transfer writes strands a seller permanently

The earlier implementation built its crash-resume skip set from **all**
transfer rows for the intent, keyed on `vendorOrderId` regardless of
`destinationKind`. A vendor order
can write two rows: the partner split first, then the seller's own leg. If
settlement crashed in the window between them, the resumed run saw a transfer for
that vendor order and skipped it entirely: no seller transfer, no escrow hold, no
reserve, and the sub-order left `pending` forever.

Nothing recovers it. The retry sweep only re-drives *unfunded seller legs*, and no
such row was ever written, so the money stays on the platform balance with
nothing pointing at it, and the seller's order queue shows a paid order stuck
pending.

Narrow window, permanent consequence, and invisible to every test that does not
inject a crash at exactly that point.

**Shipped:** two separate resume sets, `settledSellerIds` and
`settledPartnerIds`, filtered by `destinationKind`, in
[settlement.md](settlement.md).

### 2. One declined charge can burn two dunning attempts

`applySubscriptionChargeOutcome` deduped a failure against
`invoice.payment_intent_id === pi.id`. But the synchronous path only records an
intent id when the Stripe SDK error carries one: a card decline does, a **network
error or timeout does not**. When the charge failed that way, the row kept its old
(or null) intent id, the webhook's comparison did not match, and a second failure
was recorded for the same charge.

Consequence: a seller is suspended after two real failures instead of three, and
the dunning email fires early.

**Shipped:** the dedupe also treats a recorded failure with a null intent id as
already-applied, and adopts the id so later deliveries dedupe on the fast path, in
[subscriptions.md](subscriptions.md). A regression test is included.

### 3. The Stripe client pinned no API version

`new Stripe(key, { appInfo })` with no `apiVersion` falls back to the version the
installed SDK pins to. Deterministic per lockfile, so nothing is broken today,
but bumping the `stripe` package silently changes request and response shapes
across every call site in the module, including money-carrying ones.

**Shipped:** an explicit `apiVersion` and `maxNetworkRetries`, with a comment
explaining why the pin exists so nobody removes it, in
[stripe-adapter.md](stripe-adapter.md).

### 4. Unbounded sweeps

`chargeDueInvoices` selected every due invoice with no limit and charged them
sequentially, one Stripe round-trip each. Correct, but on a large tenant base it
exceeds the function timeout mid-run; the next run picks up, so it self-heals, and
therefore nobody notices the sweep never completes.

**Shipped:** an explicit page limit, and the rule that a bounded sweep must log
what it left behind, in [reconciliation.md](reconciliation.md),
[subscriptions.md](subscriptions.md).

## Kept deliberately

These look wrong and are not. Do not "fix" them.

- **A rejected transfer does not throw.** It records an unfunded leg and continues.
  Letting it propagate stranded paid orders on `pending` and made the webhook 500
  on every retry forever, and the common cause (a wrong-region connected account) is
  not fixed by retrying.
- **The webhook-events insert is a claim, not a receipt.** A duplicate insert
  re-reads the row and re-runs the handlers if `processed_at` is null. Treating
  every conflict as a duplicate drops any event whose first delivery crashed.
- **The retry sweep reuses the settlement idempotency key.** Deliberate: inside
  Stripe's 24h window it resolves to the original transfer rather than creating a
  second one. The `findTransferInGroup` lookup covers the window beyond that.
- **`requiresActionState` is set only on Stripe's own `requires_action`.** An
  unconfirmed intent maps to the same status but carries no action state, which is
  what keeps "the buyer is mid-3DS" distinguishable from "the buyer never
  started". Collapsing them cancels paid orders.
- **A currency mismatch parks the row for manual review** instead of converting.
  There is no FX in this ledger; a guessed rate mis-charged real sellers in the
  earlier implementation (a fee denominated in one currency reversed off a
  transfer in another).
- **`account.updated` skips the idempotency table.** Its handler is a conditional
  flag flip, naturally idempotent, and Connect emits it often enough that the
  table would fill with noise.
- **A partner leg that the provider rejects stays owed to the partner** and does
  not fold into the seller. Folding is a *routing* decision made when no payable
  partner account exists; a rejection is a failure path.
- **Country lists for payouts and for addresses are separate.** In the earlier
  implementation they were briefly shared, which told sellers their service
  address and passport country were wherever Stripe happened to support payouts.

## Added

Not in the earlier implementation; designed in the skill and marked as such.

- **The `PaymentsStore` seam.** The earlier implementation calls Supabase
  directly throughout. The interface was introduced when the skill was written
  and has never run as an abstraction, so the three atomic claims are the places
  most likely to be implemented wrongly behind it. [adaptation.md](adaptation.md)
  flags them.
- **The alerting list** in [operations.md](operations.md). The earlier
  implementation had the logs and the admin surfaces but no documented alerts.
- **The "leave behind" guidance** in [adaptation.md](adaptation.md).

## Not verified

Stated plainly rather than implied:

- **The SQL has not been executed.** Schema, indexes and the claim functions are
  transcribed and adjusted for portability (schema qualifiers flattened, RLS
  reduced to a shape); they compile in the reader's head, not in a database. Run
  them against a scratch Postgres before trusting the DDL.
- **`stripe.v2.core.accounts.create` shapes** are as the earlier implementation
  used them against a live account, but Accounts v2 is newer than the rest of the
  module: check the current API reference before shipping onboarding.
- **The `apiVersion` string** in the adapter is the one `stripe@22` pins to, and
  it type-checks against that SDK. A different SDK major will reject it at compile
  time, which is the intended behaviour, not a bug.
- The TypeScript templates were assembled into a scratch project against the real
  `stripe` types and compile clean under `strict` **and**
  `--noUncheckedIndexedAccess`. The `store` calls are checked against the
  interface in [store.md](store.md), not against any real implementation, and the
  host seams (auth, audit, logger, HTTP) are declared stubs.
- The money module's behavioural tests were **run and pass** (9 tests). Two of
  them initially encoded wrong expectations about `mulRate` (it rounds at the
  4-dp internal scale, not at the minor unit), which is exactly the confusion the
  corrected tests now pin down. Nothing else in the skill has executable tests.

## Provenance of the onboarding code

The Connect onboarding helper in [connect-accounts.md](connect-accounts.md) was
**withdrawn from the earlier implementation** shortly before the skill was
written, not because it was broken, but for the business reason described by the
region rule: a platform registered outside the corridor can only pay sellers in
its own country, so offering Stripe onboarding to sellers elsewhere created
accounts that could never be funded. The code shipped and worked; the template
is the version that ran, not a reconstruction.

That is the most useful thing in this skill, and it is worth stating plainly:
**a Connect integration can be entirely correct and still never pay anyone**, if
the platform's account region and the sellers' countries are incompatible. Check
that before you write a line.

## If you are fixing an existing implementation

Fix order, most damaging first:

1. The crash-resume skip set (section 1): silent, permanent, unrecoverable.
2. The dunning double-count (section 2): suspends paying customers early.
3. Pin the API version (section 3): before the next SDK bump, not after.
4. Bound the sweeps (section 4).
