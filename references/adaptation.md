# Adaptation

Fitting this module into a host app so a reviewer cannot tell it came from
somewhere else.

## Probe the host first

```bash
cat package.json | grep -A60 '"dependencies"'   # Next major, ORM, validation, tests
ls app lib components 2>/dev/null
cat tsconfig.json | grep -A5 '"paths"'          # @/ or ~/ or relative
ls middleware.ts proxy.ts 2>/dev/null           # Next 16 uses proxy.ts
cat CLAUDE.md AGENTS.md 2>/dev/null | head -60  # the house rules, already decided
```

Then read **two existing route handlers end to end** — ideally ones that mutate
data — and write down the house pattern: the exact auth call, the error shape,
how the DB client is obtained, whether bodies are validated and with what.

`CLAUDE.md` / `AGENTS.md` outrank anything in this skill.

## The rename

The skill's vocabulary and the most likely host equivalents:

| Skill | Means | Common host names |
|---|---|---|
| `Tenant` | The selling organization | Seller, Vendor, Merchant, Store, Org, Workspace |
| `Order` | The buyer's whole purchase | Order, Purchase, Cart |
| `VendorOrder` | One seller's slice of an order | SubOrder, Fulfilment, LineGroup, SellerOrder |
| `Partner` | A commissioned third party under a tenant | Affiliate, Agent, Referrer, Rep |
| `vendorNet` | What the seller earns on that slice | sellerNet, payoutAmount, netAmount |
| `PaymentIntent` row | Your record of the charge | Payment, Charge, Transaction |

Confirm the rename **with the user before generating**, then apply it everywhere
at once — types, columns, route paths, variables, comments, strings. A half-done
rename teaches the next reader that both names are live.

**Do not rename** genuine Stripe/technical terms: `transfer_group`,
`source_transaction`, `account.updated`, `client_secret`, `acct_`, `whsec_`,
`off_session`, `idempotencyKey`. They belong to Stripe, not to your domain, and
renaming them breaks every search against Stripe's docs.

## Seam by seam

### Auth guard

The skill writes `requireSession()` returning `{ user, claims }`. Replace with the
host's call, keeping the shape: **authorize before touching a service-role
client.** Every route in this module writes with elevated privilege, so the guard
is the only thing standing between a seller and another seller's payouts.

```ts
// Clerk                        // NextAuth                  // Supabase
const { userId, orgId } = auth();   const s = await auth();      const { data: { user } } =
                                                                   await supabase.auth.getUser();
```

Check **every** route, including `[id]` ones. Collection routes reliably get the
guard; detail routes are where it goes missing.

### Data access

The skill talks to a `store` object. Implement it in the host's existing style —
if the host uses Drizzle with a repository layer, this module gets Drizzle with a
repository layer, even where the skill shows raw SQL. **Never mix data-access
styles inside one codebase.**

Three operations must keep their exact semantics, whatever the ORM:

| Operation | Non-negotiable property |
|---|---|
| `claimSettlement` | A single atomic conditional update returning whether *this* caller won |
| `claimFeeAllocation` | Same — `pending → allocated` must be a race-free claim |
| `claimTransferRetry` | Same, and it must also write the next backoff time |

If your ORM cannot express "update … where status = 'pending' returning id" as
one statement, drop to raw SQL for these three. A read-then-write is not
equivalent and will double-pay under load.

### Audit and notifications

`recordAudit({ action, tenantId, payload, requestId, actorUserId })` and
`notify(type, tenantId, payload)` are call sites, not implementations. Map to the
host's audit table and outbox, or stub them — but keep the call sites. They are
what makes "who changed this, and when" answerable, and re-adding them later means
touching every path again.

Event names worth preserving as a set:

```
payments.stripe.onboarding_started    payments.stripe.account_ready
payments.transfer_failed              payments.transfer_retried
partner.payout_blocked                kyc.fee_collected
subscription.invoice_issued           subscription.invoice_paid
subscription.payment_failed           subscription.plan_changed
```

### Validation

The skill shows `parseBody(req, Body)` with a zod-shaped schema. Swap the library;
keep the boundary. `as SomeType` on request input is a cast, not a check, and this
module's inputs decide where money goes.

### Money column

If the host already stores money, **do not introduce a second representation.**
Adapt the money module's scale to theirs (integer minor units is the common
alternative — keep the same functions, change `SCALE` to 0 and drop the
`toMinorUnits` divisor). Two money types in one codebase is the worst outcome.

### Background work

Four crons, cadences in [operations.md](operations.md). Map to the host's
scheduler. If it has none, the sweeps can run opportunistically from a read path
(rate-limited) — say so explicitly rather than shipping jobs that never fire.

## Order of work

Inward-out; type-check after each layer. Fixing a rename at step 1 is one edit; at
step 6 it is thirty.

1. **Money** — pure, no dependencies, tests pass immediately.
2. **Types + schema/migration** — with the rename applied.
3. **Store** — the host's ORM, its file layout, its naming.
4. **Adapter** — client, provider interface, Stripe implementation.
5. **Webhook route** — the host's error shape and logging.
6. **Settlement, then reconciliation, then subscriptions.**
7. **Crons and the admin surfaces.**

## What to leave behind

Do not port these unless the host actually needs them:

- The partner/commission split, if there are only sellers and a platform.
- Rolling reserves, if you pay out immediately.
- The KYC-gated payout hold, if your compliance model differs — but keep *some*
  gate between escrow release and payout, or a fraudulent seller cashes out.
- The escrow window itself, if sellers are paid on capture. Note that removing it
  removes your only lever for refunds after payout.

Each removal is a decision worth recording in a comment, because the next reader
will wonder whether it was an omission.

## Checklist

- [ ] `package.json`, `CLAUDE.md`/`AGENTS.md` read; no new dependency added unasked
- [ ] Two existing route handlers read; house pattern written down
- [ ] Rename confirmed with the user, applied everywhere at once
- [ ] Stripe/technical terms left alone
- [ ] Auth guard on every route, including `[id]` routes
- [ ] The three atomic claims are single statements
- [ ] One money representation in the codebase, not two
- [ ] Audit and notification call sites kept (or explicitly stubbed)
- [ ] Region rule checked for the host's actual platform account
- [ ] Host's lint, type-check, tests and build all pass
