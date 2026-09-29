# Operations

Setting it up, proving it works, and diagnosing it when it does not. This is the
half that decides whether the integration survives contact with real sellers.

## Environment

```bash
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...   # pk_live_... in production
STRIPE_SECRET_KEY=sk_test_...                    # sk_live_... in production
STRIPE_WEBHOOK_SECRET=whsec_...                  # endpoint A: platform events
STRIPE_CONNECT_WEBHOOK_SECRET=whsec_...          # endpoint B: account.updated
CRON_SECRET=...                                  # guards the cron URL triggers
```

Ship these five names in a tracked `.env.example`, every value empty, and add
`!.env.example` to `.gitignore` when it ignores `.env*`. Add only what the host's
own seams need (a database URL, its auth secret); never a variable that stands in
for something the templates hard-code, such as the platform's country, which
lives in `STRIPE_COUNTRIES` ([connect-accounts.md](connect-accounts.md)).

Test and live keys are different. Crossing them fails in ways that read like code
bugs: a live webhook secret verified with test keys fails signature checks, and a
test key cannot see live accounts at all.

## Setup, in order

1. **Check the region rule first.** Before anything else, confirm your platform's
   country can transfer to the countries your sellers will be in. See
   [connect-accounts.md](connect-accounts.md). Everything else can be fixed later;
   this one cannot.
2. **Enable Connect** in the dashboard and complete the platform profile.
3. **Keys**: Developers, then API keys, per mode.
4. **Register two webhook endpoints** at the same URL, with the event sets in
   [webhooks.md](webhooks.md). Two registrations, two secrets.
5. **Set the env vars** per environment.
6. **Verify** (below), in test mode, then repeat in live.

Locally:

```bash
stripe listen --forward-to localhost:3000/api/webhooks/stripe
stripe listen --listen-to-connect --forward-to localhost:3000/api/webhooks/stripe
```

Each prints its own `whsec_...`. A single listener forwarding both streams may share
one secret locally, which is why local success does not prove the two-secret
setup is right in a deployed environment.

## Verification

1. **Signature path.** *Send test event* on the endpoint page, or
   `stripe trigger payment_intent.succeeded`. Expect `200`, and a row in
   `webhook_events` keyed `(stripe, event.id)`. A replay returns `200` with
   `Idempotent-Replay: true`. A tampered signature returns `401`.
2. **Region path, before any end-to-end test.** Confirm
   `country_specs/<platform country>.supported_transfer_countries` contains the
   country you are about to onboard the test seller in. If it does not, checkout
   will still charge the card and the order will still reach `paid`, **and the
   seller will not be funded**. Do not treat that run as green.
3. **End to end.** Onboard a seller in a supported country, run a card checkout
   with a Stripe test card. On `payment_intent.succeeded` expect the order `paid`,
   one transfer row per seller sub-order, and escrow holds fanned out.
4. **Log trace.** `stripe.webhook.received`, then `stripe.webhook.processed`. A
   `received` with no `processed` means the fan-out threw; read the error beside
   it.
5. **Repeat in live mode.**

## Crons

| Job | Cadence | Does |
|---|---|---|
| `retry-transfers` | every 20 min | Re-drives unfunded legs |
| `release-escrow` | hourly | Flips due holds to `released` |
| `sweep-gateway-fees` | hourly | Resumes stuck fee reconciliation |
| `reconcile-orphan-payments` | hourly | Finds charges with no settled order |
| `issue-subscription-invoices` | daily | Cancellations, then issue, then charge due |

On platforms where scheduled jobs only run in production (Vercel among them),
give every cron a URL trigger guarded by a shared secret so staging can be driven
by hand:

```
https://<deployment>/api/cron/retry-transfers?key=$CRON_SECRET
```

Return the summary in the response body (`{ funded, deferred, skipped }`): it is
the fastest way to confirm a fix without tailing logs.

## Observability

Log these; each answers a question you will actually be asked.

| Event | Fields | Answers |
|---|---|---|
| `stripe.webhook.received` | `eventId, type, livemode` | Did the webhook reach this environment? |
| `stripe.webhook.rejected` | `err` | Is the signing secret wrong? |
| `stripe.webhook.processed` | `eventId, type` | Did handling finish? |
| `settle.claim_lost` | `orderId` | Was this a duplicate settlement attempt? |
| `settle.transfer_failed` | `err, tenantId, amount, kind` | Which seller was not paid, and why? |
| `settle.transfer_source_unresolved` | `orderId` | Why did a transfer fall back to the available balance? |
| `transfer_retry.funded` / `.adopted` / `.failed` | `transferId, attempt` | Is the backlog clearing? |
| `gateway_fee.currency_mismatch` | `orderId, feeCurrency` | Which orders need manual review? |
| `payments.settled_via_sync` | `orderId` | Is the webhook actually working, or is the read path carrying it? |

That last one is the sleeper. If `payments.settled_via_sync` is common, your
webhook is not being delivered and you are running on the fallback, which works
until a buyer never opens the order page.

Operator surfaces worth building, in value order:

1. **Unfunded transfers list** with `lastError`, amount, seller, age. The single
   most useful screen; without it "the seller was never paid" is undiagnosable.
2. **Payable balance breakdown** per seller (released, reserves, reversals,
   adjustments, scheduled), not just the total.
3. **Gateway-fee status column** so `manual_review` rows are visible, not
   indistinguishable from unreconciled ones.
4. **Webhook event log** with `processed_at`, filterable by type.

## Troubleshooting

**`Funds can't be sent to accounts located in XX because it's restricted outside
of your platform's region`**
The region rule. The order still reaches `paid` and the leg is recorded unfunded;
the money sits on the platform balance. Not fixable on the seller by config: the
account country is immutable, so they must re-onboard in a supported country.
Once the accounts are right, the retry cron funds the historical backlog itself.

**`You have insufficient available funds in your Stripe account`**
The available balance is `0.00` because the charges behind it are still in the
*pending* balance (the settlement delay: a young account can hold everything
there for a week). Naming the charge as `source_transaction` removes the
dependency entirely. Seeing this means the leg predates that fix, or the adapter
could not resolve the charge. Check for `settle.transfer_source_unresolved`.

**Transfer rejected on currency, once the region is right**
The platform balance is held in specific currencies, but settlement transfers in
the *order's* currency. A transfer in a currency you hold no balance in fails
with `balance_insufficient`. Check `GET /v1/balance` against the order's currency
before looking anywhere else.

**Order stuck `pending` after a successful charge**
Three distinct causes; the logs tell them apart:
- No `stripe.webhook.received` at all: the delivery never landed. Wrong URL,
  wrong secret, or the endpoint is behind deployment protection. The read-path
  sync should be rescuing these; if it is not, check that it is wired into the
  order page.
- `received` with no `processed`: the fan-out threw. Read the error beside it.
- `received` and `processed`, order still pending: settlement crashed between
  the intent flip and the order update. The order **is** paid; never cancel it.

**`500 stripe_misconfigured`**
`STRIPE_SECRET_KEY` unset, or neither webhook secret set.

**`401 invalid_signature`**
The endpoint's secret does not match the env var, or test/live are crossed. Since
both secrets are tried, swapping them still works; two *wrong* ones do not. Also
returned when the event timestamp is outside Stripe's 5-minute tolerance, which
looks identical: check the clock before assuming the secret is wrong.

**Seller never goes ready / `payments_ready` stays false**
`account.updated` is not arriving. Endpoint B missing, or the event subscribed on
the platform stream instead of the connected-account one. Both `charges_enabled`
and `payouts_enabled` must be true.

**Subscription charges fail with `authentication_required`**
The SetupIntent was created without `usage: "off_session"`, so the card was never
authorised for unattended charges. Re-collect the card.

## Runbook checklist

- [ ] Platform country's `supported_transfer_countries` checked and recorded
- [ ] Country picker constrained to that list; default is a payable country
- [ ] Both webhook endpoints registered, with separate secrets in separate vars
- [ ] `payment_intent.succeeded` test event returns `200`; replay returns `Idempotent-Replay`
- [ ] End-to-end checkout produces transfers **and** escrow holds
- [ ] Every cron has a manual URL trigger and returns a summary
- [ ] Unfunded-transfers list exists and shows `lastError`
- [ ] Alert on: unfunded legs older than 24h, `gateway_fee` rows in `manual_review`,
      `stripe.webhook.rejected`, and a rising `payments.settled_via_sync` rate

## Related

[connect-accounts.md](connect-accounts.md) for the region rule in full,
[reconciliation.md](reconciliation.md) for what the sweeps do,
[webhooks.md](webhooks.md) for endpoint registration.
