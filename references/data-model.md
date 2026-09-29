# Data model

**This module needs a relational store.** Not a preference: the ledger relies on
`sum()` over indexed rows, unique constraints for webhook idempotency, and atomic
conditional updates for every state transition. A document store cannot express
the settlement claim safely, and a marketplace that double-pays is worse than one
that ships later.

Reference implementations here: raw SQL/Postgres (portable to Drizzle, Prisma,
Kysely) and the Supabase client. Pick the one your host already uses; a host
with no database gets the raw-SQL store on Postgres through `pg`, with the
migration below. An in-memory store is a test fixture, never the default: the
ledger is the record of who is owed money, and it must survive a restart. The
data-access contract every module in this skill calls is in
[store.md](store.md).

## Neutral types

SDK-free, so both implementations derive from them. ISO strings at the boundary,
`null` never `undefined`, amounts as strings.

```ts
// lib/payments/types.ts
export type ProviderName = "stripe";
export type IntentStatus =
  | "requires_action" | "processing" | "succeeded" | "failed" | "refunded";

/** `held` = allocated to the seller but never sent at the provider. */
export type TransferStatus = "pending" | "succeeded" | "failed" | "reversed" | "held";
export type GatewayFeeStatus = "pending" | "allocated" | "recorded" | "manual_review";
export type EscrowStatus = "held" | "released" | "reversed" | "frozen";

export type PaymentIntentRow = {
  id: string;
  orderId: string;
  provider: ProviderName;
  externalId: string;
  amount: string;
  currencyCode: string;
  status: IntentStatus;
  settleClaimedAt: string | null;
  createdAt: string;
};

export type TransferRow = {
  id: string;
  paymentIntentId: string;
  vendorOrderId: string | null;
  tenantId: string;
  destinationKind: "tenant" | "partner";
  membershipId: string | null;
  amount: string;
  currencyCode: string;
  /** Null while the leg is unfunded: the money did not move. */
  externalId: string | null;
  status: TransferStatus;
  retryAttempts: number;
  retryNextAttemptAt: string | null;
  lastError: string | null;
  createdAt: string;
};

export type EscrowHoldRow = {
  id: string;
  orderItemId: string;
  vendorOrderId: string;
  tenantId: string;
  amount: string;
  currencyCode: string;
  status: EscrowStatus;
  releaseAt: string;
};

export type SubscriptionInvoiceRow = {
  id: string;
  tenantId: string;
  planCode: string;
  periodStart: string;
  periodEnd: string;
  amount: string;
  currencyCode: string;
  status: "open" | "paid" | "past_due" | "void" | "refunded";
  chargeAttempts: number;
  paymentIntentId: string | null;
  lastChargeError: string | null;
  nextChargeAt: string | null;
  dueAt: string;
  paidAt: string | null;
};
```

## Schema

Amounts are `numeric(19,4)`; times are `timestamptz` in UTC. Adjust schema names
and the `tenants`/`orders` foreign keys to your host's vocabulary.

```sql
-- migrations/0001_payments.sql
-- The buyer-side charge. `external_id` is unique so a webhook can resolve an
-- order from Stripe's id alone.
create table payment_intents (
  id uuid primary key default gen_random_uuid(),
  order_id uuid not null references orders(id) on delete cascade,
  provider text not null check (provider in ('stripe')),
  external_id text not null unique,
  amount numeric(19,4) not null,
  currency_code char(3) not null,
  status text not null default 'processing' check (status in (
    'requires_action','processing','succeeded','failed','refunded')),
  requires_action_state text check (requires_action_state in ('3ds','redirect','other')),
  -- Settlement lease. Leased rather than a boolean so a crashed run is
  -- resumable without an operator unsticking it.
  settle_claimed_at timestamptz,
  raw jsonb,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
create index payment_intents_order_idx on payment_intents (order_id);
create index payment_intents_status_idx on payment_intents (status, updated_at);

-- One settlement leg. `external_id is null` means the money did not move: the
-- leg is a claim on the platform balance that the retry sweep will re-drive.
create table transfers (
  id uuid primary key default gen_random_uuid(),
  payment_intent_id uuid not null references payment_intents(id) on delete cascade,
  vendor_order_id uuid references vendor_orders(id) on delete set null,
  tenant_id uuid not null references tenants(id),
  destination_kind text not null default 'tenant' check (destination_kind in ('tenant','partner')),
  membership_id uuid references partner_memberships(id),
  amount numeric(19,4) not null,
  currency_code char(3) not null,
  external_id text,
  status text not null default 'pending' check (status in (
    'pending','succeeded','failed','reversed','held')),
  retry_attempts integer not null default 0,
  retry_next_attempt_at timestamptz not null default now(),
  last_error text,
  created_at timestamptz not null default now(),
  check ((destination_kind = 'partner') = (membership_id is not null))
);
create index transfers_payment_intent_idx on transfers (payment_intent_id);
create index transfers_tenant_idx on transfers (tenant_id, status);
-- The retry sweep's scan. Partial, because only unfunded legs are ever swept.
create index transfers_unfunded_idx on transfers (retry_next_attempt_at)
  where status = 'pending' and external_id is null;

-- Reversals stack on one transfer (fee clawback, then a refund), so the
-- reversible headroom must be computed from this table, never assumed.
create table transfer_reversals (
  id uuid primary key default gen_random_uuid(),
  transfer_id uuid not null references transfers(id) on delete cascade,
  amount numeric(19,4) not null check (amount > 0),
  currency_code char(3) not null,
  reason text not null,
  external_id text,
  created_at timestamptz not null default now()
);
create index transfer_reversals_transfer_idx on transfer_reversals (transfer_id);

create table escrow_holds (
  id uuid primary key default gen_random_uuid(),
  order_item_id uuid not null,
  vendor_order_id uuid not null references vendor_orders(id) on delete cascade,
  tenant_id uuid not null references tenants(id),
  amount numeric(19,4) not null,
  currency_code char(3) not null,
  status text not null default 'held' check (status in ('held','released','reversed','frozen')),
  release_at timestamptz not null,
  created_at timestamptz not null default now()
);
-- One hold per item: settlement writes holds on every run, and this makes the
-- re-run a no-op (insert ... on conflict (order_item_id) do nothing).
create unique index escrow_holds_item_uniq on escrow_holds (order_item_id);
create index escrow_holds_release_idx on escrow_holds (release_at) where status = 'held';
create index escrow_holds_tenant_idx on escrow_holds (tenant_id, currency_code, status);

create table reserves (
  id uuid primary key default gen_random_uuid(),
  -- Set when accrued from a sale; unique so a settlement re-run accrues once.
  vendor_order_id uuid unique references vendor_orders(id),
  tenant_id uuid not null references tenants(id),
  amount numeric(19,4) not null,
  currency_code char(3) not null,
  reserve_pct numeric(5,4),
  reason text not null,
  release_at timestamptz not null,
  released_at timestamptz,
  created_at timestamptz not null default now()
);
create index reserves_open_idx on reserves (tenant_id, currency_code) where released_at is null;

-- Signed. A debit the platform takes back is negative; a manual correction can
-- be either. Keeps the balance formula a plain sum.
create table adjustments (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid not null references tenants(id),
  amount numeric(19,4) not null,
  currency_code char(3) not null,
  reason text not null,
  created_at timestamptz not null default now()
);
create index adjustments_tenant_idx on adjustments (tenant_id, currency_code);

create table payouts (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid not null references tenants(id),
  amount numeric(19,4) not null,
  currency_code char(3) not null,
  external_id text,
  status text not null default 'scheduled' check (status in (
    'scheduled','in_transit','paid','failed')),
  created_at timestamptz not null default now()
);
create index payouts_tenant_idx on payouts (tenant_id, currency_code, status);

-- Webhook idempotency. The unique constraint IS the mechanism; `processed_at`
-- distinguishes "claimed" from "handled" (see webhooks.md).
create table webhook_events (
  id uuid primary key default gen_random_uuid(),
  provider text not null,
  external_event_id text not null,
  type text not null,
  payload jsonb not null,
  received_at timestamptz not null default now(),
  processed_at timestamptz,
  unique (provider, external_event_id)
);
```

Columns added to your existing tables:

```sql
-- Sellers
alter table tenants
  add column stripe_account_id text unique,       -- acct_...; unique so a webhook resolves one row
  add column payments_ready boolean not null default false,
  add column compliance_hold boolean not null default false,
  -- Platform-subscription billing instrument. Brand/last4 only, never the PAN.
  add column billing_customer_id text,
  add column billing_payment_method_id text,
  add column billing_card_brand text,
  add column billing_card_last4 text;

-- Per-seller sub-order: the unit a transfer and a fee share are keyed to.
alter table vendor_orders
  add column vendor_net numeric(19,4) not null default 0,
  add column gateway_fee_amount numeric(19,4) not null default 0,
  add column gateway_fee_status text not null default 'pending'
    check (gateway_fee_status in ('pending','allocated','recorded','manual_review'));

-- Platform subscription invoices
alter table subscription_invoices
  add column payment_intent_id text,
  add column charge_attempts integer not null default 0,
  add column last_charge_error text,
  add column next_charge_at timestamptz;
create index subscription_invoices_next_charge_idx
  on subscription_invoices (next_charge_at) where status = 'open';
-- Makes the issuance cron idempotent at the database, not just in code.
create unique index subscription_invoices_tenant_period_uniq
  on subscription_invoices (tenant_id, period_start);
```

## The settlement claim

The mutual exclusion that makes settlement single-flight. A **lease**, not a
boolean: a run that crashes mid-fan-out must be resumable when the lease expires,
without anyone unsticking it by hand.

```sql
create or replace function fn_claim_settlement(
  p_intent_id uuid,
  p_lease_seconds int default 300
) returns boolean
language plpgsql
security definer          -- deliberate: callers run as the request's role
set search_path = ''      -- required with security definer
as $$
declare v_claimed boolean;
begin
  update public.payment_intents pi
  set settle_claimed_at = now()
  where pi.id = p_intent_id
    and pi.status <> 'succeeded'
    and (pi.settle_claimed_at is null
         or pi.settle_claimed_at < now() - make_interval(secs => p_lease_seconds))
  returning true into v_claimed;
  return coalesce(v_claimed, false);
end;
$$;

-- Hand the lease back when settlement failed before completing, so the
-- provider's retry resumes at once instead of waiting the lease out.
create or replace function fn_release_settlement_claim(p_intent_id uuid)
returns void language plpgsql security definer set search_path = '' as $$
begin
  update public.payment_intents pi set settle_claimed_at = null
  where pi.id = p_intent_id and pi.status <> 'succeeded';
end;
$$;
```

**300 seconds is not arbitrary.** It must exceed the worst-case fan-out (one
Stripe round-trip per seller leg plus escrow writes) and stay under the webhook
retry interval, or a slow settlement is re-entered while still running. Raise it
if carts routinely carry many sellers.

## Access-control posture

State this explicitly: the most common porting mistake is assuming the absence
of policies is safe.

- **All writes in this module are server-side, via a service role that bypasses
  RLS.** The request must be authorized in the route handler *before* the
  service-role client is touched.
- **Reads are policy-gated**: super-admin sees everything; a seller sees rows
  scoped to its own `tenant_id`; a buyer sees its own order's intent.
- `payment_intents.raw` and `webhook_events.payload` hold provider payloads.
  Restrict them to admins: they carry card metadata and customer detail.

Supabase RLS shape, with the performance form that matters (the wrapped
`(select auth.jwt())` is hoisted by the planner; a bare `auth.jwt()` re-evaluates
per row):

```sql
alter table transfers enable row level security;
alter table transfers force row level security;

create policy transfers_select on transfers for select using (
  ((select auth.jwt()) ->> 'role') = 'super_admin'
  or (tenant_id = ((select auth.jwt()) ->> 'tenant_id')::uuid
      and ((select auth.jwt()) ->> 'tenant_role') in ('owner','admin'))
);
-- Writes: service role only. `with check` as well as `using`, or a row can be
-- updated *out* of the caller's tenant.
create policy transfers_write on transfers for all
  using (((select auth.jwt()) ->> 'role') = 'super_admin')
  with check (((select auth.jwt()) ->> 'role') = 'super_admin');
```

## Related

[settlement.md](settlement.md) uses the claim,
[reconciliation.md](reconciliation.md) uses the partial index,
[adaptation.md](adaptation.md) maps these names onto your host's.
