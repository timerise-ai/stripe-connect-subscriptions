# Architecture

Why the money moves the way it does. Read this before writing code — most of the
hard rules elsewhere are consequences of the choices here.

## The charge model

Stripe offers three ways for a platform to take money on a seller's behalf.

| Model | Shape | Why not / why |
|---|---|---|
| **Direct charge** | Charge created *on* the connected account; platform takes `application_fee_amount` | Seller is merchant of record; one seller per charge; platform never holds the funds, so it cannot escrow, reserve or claw back |
| **Destination charge** | Charge on the platform with `transfer_data.destination` | **One destination per charge.** A two-seller cart cannot be expressed at all |
| **Separate charges and transfers** ✅ | One charge on the platform carrying a `transfer_group`, then N `transfers.create({ destination })` under that group | The only model that supports many sellers per charge, and the only one where the platform holds funds long enough to escrow, reserve and reverse |

**This skill is built entirely on separate charges and transfers.** The
consequence is that `application_fee_amount` is unusable: the platform's cut is
simply the money it does not transfer out. Commission is arithmetic in your
ledger, not a Stripe field.

The second consequence is the one people miss: **paying the seller is now a
separate operation that can fail on its own**, minutes or days after the buyer
was charged. Every design decision downstream follows from that.

## The two flows

```
MARKETPLACE (Connect)                    PLATFORM SUBSCRIPTION (Billing)
buyer's card                             seller's saved card
  → charge on platform account             → off_session PaymentIntent
  → transfers to connected accounts        → settles one subscription invoice
  → payouts to seller bank                 → dunning on failure
money flows OUT of the platform          money flows IN to the platform
```

They share the Stripe client, the webhook endpoint, the money type and the
idempotency discipline — and nothing else. Keep them in separate modules; the
only place they meet is the webhook dispatcher, which routes on
`metadata.purpose` (see [webhooks.md](webhooks.md)).

## Money movement, end to end

```
1. checkout      create PaymentIntent, transfer_group = orderId, status=processing
2. buyer pays    3DS if required; Stripe charges the card
3. webhook       payment_intent.succeeded  ─┐
   OR read-path  syncOrderPaymentFromProvider ├─▶ settleOrder(orderId)
                                             ─┘   (both routinely fire; see below)
4. settlement    claim the intent (leased, atomic)
                 for each seller sub-order:
                   transfer  → connected account  (source_transaction = the charge)
                   escrow    → hold seller's net until release_at
                   reserve   → withhold a % of turnover
                 mark intent succeeded, order paid
5. convergence   retry-transfers cron re-drives unfunded legs
                 gateway-fee reconcile claws the Stripe fee back off the seller
6. release       release-escrow cron flips holds to released
7. payout        seller (or cron) triggers a payout from the payable balance,
                 gated on KYC, risk holds and a minimum threshold
```

**Step 3 has two independent triggers on purpose.** A webhook that is delayed,
dropped, or never registered for the environment leaves a charged order sitting
unpaid — the seller cannot fulfil it and the unpaid-order sweep would cancel it.
So read paths (the buyer's order page, the seller's order detail) also ask Stripe
directly for the intent's status and settle on success. Both callers reach
`settleOrder` within milliseconds of each other, which is exactly why settlement
needs a real mutual-exclusion claim rather than a status check.

## The payable balance is derived, not stored

```
payable = Σ released escrow holds
        − Σ open reserves
        − Σ transfer reversals        (refunds, chargebacks, fee clawbacks)
        + Σ adjustments               (manual corrections, both signs)
        − Σ payouts already scheduled/in-transit/paid
```

Derived, because a stored balance needs a cron to maintain it and **a failed cron
makes every account look healthy**. Recomputing from the ledger is a handful of
indexed sums and it cannot silently drift. Store the *components*; compute the
total on read.

The same principle governs status everywhere in this module: derive from the
ledger where you can, and where you must store a status (the gateway-fee state
machine, the transfer status), make every transition an atomic conditional update
so two writers cannot both think they won.

## Idempotency, three different mechanisms

Do not use one mechanism for all three problems — they fail differently.

| Problem | Mechanism | Detail |
|---|---|---|
| Stripe delivers an event twice | A `webhook_events` row unique on `(provider, event_id)` | The insert is a **claim**, not proof of handling — see [webhooks.md](webhooks.md) |
| Two of *our* callers settle the same order | A leased claim on the payment intent (`fn_claim_settlement`) | A lease, so a crashed run is resumable — see [settlement.md](settlement.md) |
| We re-send the same request to Stripe | `idempotencyKey` on the SDK call, derived from stable ids | Stripe's window is **24 hours**; beyond it the key is meaningless — see [reconciliation.md](reconciliation.md) |

The 24-hour limit is the one that bites. A leg re-driven three days later needs a
*lookup* ("does Stripe already hold a transfer for this leg?"), not a key.

## Failure posture

State this explicitly, because it is counter-intuitive and future readers will
try to "fix" it:

- **Buyer-side failures are fatal.** If the charge fails, cancel the order and
  release inventory.
- **Seller-side failures are never fatal.** The buyer's money already moved. A
  rejected transfer is recorded as an unfunded leg and retried; it must not throw,
  must not 500 the webhook, and must not stop the order reaching `paid`.
- **Ledger writes are ordered before money movement** wherever a crash is
  possible. Persist the intent to move money (`allocated`), then move it, then
  mark it done. A crash between the first two leaves a resumable state; a crash
  in the other order loses money silently.

## Currency

One rule, stated once: **there is no FX anywhere in this module.** A transfer, a
reversal and a fee must all be denominated in the same currency as the thing they
act on. Where two currencies meet — a Stripe processing fee reported in the
platform's settlement currency against an order presented in another — the row is
parked for manual review rather than converted with a guessed rate. Guessing
mis-charged real sellers in the earlier implementation.

## Related

[data-model.md](data-model.md) for the tables this implies,
[settlement.md](settlement.md) for the fan-out,
[money.md](money.md) for the arithmetic.
