# The data-access contract

Every module in this skill talks to one object called `store`. This file is that
object's contract and the reference SQL behind its three non-obvious methods.

The interface exists so the settlement engine is testable against a fake — **not**
to abstract away your ORM. In a host app, implement it in that app's own idiom
(see [adaptation.md](adaptation.md)); do not introduce a repository layer the rest
of the codebase does not have.

Types referenced here (`PaymentIntentRow`, `TransferRow`, `SubscriptionInvoiceRow`,
`IntentStatus`, `TransferStatus`, `EscrowStatus`, `GatewayFeeStatus`,
`ProviderName`) are defined in [data-model.md](data-model.md).

## The interface

**Three methods are not ordinary CRUD.** `claimSettlement`, `claimFeeAllocation`
and `claimTransferRetry` must each be a *single atomic conditional update* that
reports whether this caller won. A read-then-write is not equivalent and will
double-pay under load. If your ORM cannot express them in one statement, drop to
raw SQL for those three.

```ts
export interface PaymentsStore {
  // --- intents -------------------------------------------------------------
  intentById(id: string): Promise<PaymentIntentRow | null>;
  intentByOrder(orderId: string): Promise<PaymentIntentRow | null>;
  intentByExternalId(externalId: string): Promise<PaymentIntentRow | null>;
  setIntentStatus(id: string, status: IntentStatus): Promise<void>;
  /** ATOMIC. True = this caller owns the fan-out. */
  claimSettlement(intentId: string, leaseSeconds?: number): Promise<boolean>;
  releaseSettlementClaim(intentId: string): Promise<void>;

  // --- orders --------------------------------------------------------------
  setOrderStatus(orderId: string, status: string): Promise<void>;
  vendorOrdersForOrder(orderId: string): Promise<VendorOrderRow[]>;
  setVendorOrderStatus(vendorOrderId: string, status: string): Promise<void>;
  orderItemsForVendorOrder(vendorOrderId: string): Promise<OrderItemRow[]>;

  // --- transfers -----------------------------------------------------------
  insertTransfer(t: NewTransfer): Promise<{ id: string }>;
  transfersForIntent(intentId: string): Promise<TransferRow[]>;
  fundedSellerTransfer(vendorOrderId: string): Promise<TransferRow | null>;
  dueUnfundedTransfers(limit: number): Promise<TransferRow[]>;
  /** ATOMIC. Bumps attempts + next-attempt time; false = another runner won. */
  claimTransferRetry(
    id: string,
    next: { attempts: number; nextAttemptAt: string },
  ): Promise<boolean>;
  markTransferFunded(id: string, externalId: string): Promise<void>;
  recordTransferFailure(id: string, detail: string): Promise<void>;
  insertTransferReversal(r: NewTransferReversal): Promise<{ id: string }>;
  reversalsForTransfer(transferId: string): Promise<{ amount: string }[]>;

  // --- ledger --------------------------------------------------------------
  insertEscrowHold(h: NewEscrowHold): Promise<{ id: string }>;
  holdReserve(r: NewReserve): Promise<{ id: string }>;
  createAdjustment(a: NewAdjustment): Promise<{ id: string }>;
  payoutSchedule(tenantId: string): Promise<{ cadence: string } | null>;
  sumEscrow(tenantId: string, currency: string, status: EscrowStatus): Promise<string>;
  sumOpenReserves(tenantId: string, currency: string): Promise<string>;
  sumAdjustments(tenantId: string, currency: string): Promise<string>;
  sumPayouts(tenantId: string, currency: string, statuses: string[]): Promise<string>;
  sumReversalsForTenant(tenantId: string, currency: string): Promise<string>;

  // --- gateway fee ---------------------------------------------------------
  feeRowsForOrder(orderId: string): Promise<FeeRow[]>;
  /** ATOMIC. `pending -> allocated`, persisting the share. */
  claimFeeAllocation(vendorOrderId: string, portion: string): Promise<boolean>;
  setFeeStatus(vendorOrderId: string, status: GatewayFeeStatus): Promise<void>;

  // --- accounts ------------------------------------------------------------
  getTenantAccounts(tenantId: string): Promise<TenantAccounts | null>;
  getTenantByStripeAccount(accountId: string): Promise<TenantAccounts | null>;
  setTenantStripeAccount(tenantId: string, accountId: string): Promise<void>;
  setPaymentsReady(tenantId: string, ready: boolean): Promise<void>;
  sellerAccountId(tenantId: string): Promise<string | null>;
  partnerAccountFor(
    membershipId: string,
    provider: ProviderName,
  ): Promise<PartnerAccountRow | null>;
  /** Null when no partner account matches — the caller then tries the tenant. */
  applyPartnerAccountUpdate(
    externalAccountId: string,
    chargesEnabled: boolean,
    payoutsEnabled: boolean,
  ): Promise<PartnerAccountRow | null>;

  // --- webhooks ------------------------------------------------------------
  insertWebhookEvent(e: NewWebhookEvent): Promise<InsertWebhookResult>;
  getWebhookEvent(
    provider: string,
    externalEventId: string,
  ): Promise<{ id: string; processedAt: string | null } | null>;
  markWebhookProcessed(provider: string, externalEventId: string): Promise<void>;

  // --- subscriptions -------------------------------------------------------
  getTenantBilling(tenantId: string): Promise<TenantBilling | null>;
  setBillingCustomer(tenantId: string, customerId: string): Promise<void>;
  setBillingPaymentMethod(
    tenantId: string,
    pm: { id: string; brand: string | null; last4: string | null },
  ): Promise<void>;
  clearBillingPaymentMethod(tenantId: string): Promise<void>;
  setSubscriptionStatus(tenantId: string, status: string): Promise<void>;
  activeTenants(): Promise<ActiveTenant[]>;
  listPlans(): Promise<Map<string, PlanRow>>;
  getInvoice(id: string): Promise<SubscriptionInvoiceRow | null>;
  /** Null on a unique violation — another pass issued this period. */
  insertInvoice(i: NewInvoice): Promise<{ id: string } | null>;
  updateInvoice(id: string, patch: Partial<SubscriptionInvoiceRow>): Promise<void>;
  latestInvoicePeriodEnd(tenantId: string): Promise<string | null>;
  dueOpenInvoices(opts: { limit: number }): Promise<{ id: string; tenantId: string }[]>;
  /** Tenants with a billing card whose status is `active` or `past_due`. */
  billingEligibleTenants(tenantIds: string[]): Promise<{ id: string }[]>;
  cancelDueSubscriptions(nowIso: string): Promise<{ cancelled: number }>;
}

/** Distinguishes a unique-constraint conflict from a genuine failure — the
 *  webhook claim depends on telling them apart. */
export type InsertWebhookResult =
  | { ok: true; conflict?: false; id: string; error?: undefined }
  | { ok: false; conflict: true; id?: undefined; error?: undefined }
  | { ok: false; conflict?: false; id?: undefined; error: string };

export type VendorOrderRow = {
  id: string;
  tenantId: string;
  membershipId: string | null;
  currencyCode: string;
  vendorNet: string;
  partnerCommissionAmount: string;
};

export type OrderItemRow = {
  id: string;
  productId: string | null;
  vendorNet: string;
};

export type FeeRow = {
  id: string;
  tenantId: string;
  currencyCode: string;
  vendorNet: string;
  gatewayFeeAmount: string;
  gatewayFeeStatus: GatewayFeeStatus;
};

export type TenantAccounts = {
  id: string;
  stripeAccountId: string | null;
  paymentsReady: boolean;
  complianceHold: boolean;
};

export type PartnerAccountRow = {
  id: string;
  membershipId: string;
  tenantId: string;
  externalAccountId: string | null;
  chargesEnabled: boolean;
  payoutsEnabled: boolean;
  kycStatus: "pending" | "verified" | "rejected" | "restricted";
};

export type TenantBilling = {
  id: string;
  displayName: string | null;
  billingCustomerId: string | null;
  billingPaymentMethodId: string | null;
  billingCardBrand: string | null;
  billingCardLast4: string | null;
  subscriptionStatus: string;
};

export type ActiveTenant = {
  id: string;
  subscriptionPlanCode: string;
  billingCycle: "monthly" | "annual";
};

export type PlanRow = {
  code: string;
  tier: string;
  monthlyFee: string;
  annualFee: string;
  currencyCode: string;
};

export type NewTransfer = {
  paymentIntentId: string;
  vendorOrderId: string | null;
  tenantId: string;
  destinationKind: "tenant" | "partner";
  membershipId?: string | null;
  amount: string;
  currency: string;
  externalId: string | null;
  status: TransferStatus;
};

export type NewTransferReversal = {
  transferId: string;
  amount: string;
  currency: string;
  reason: string;
  externalId: string | null;
};

export type NewEscrowHold = {
  orderItemId: string;
  vendorOrderId: string;
  tenantId: string;
  amount: string;
  currency: string;
  status: EscrowStatus;
  releaseAt: string;
};

export type NewReserve = {
  tenantId: string;
  amount: string;
  currency: string;
  reservePct: string;
  reason: string;
  releaseAt: string;
};

export type NewAdjustment = {
  tenantId: string;
  amount: string;
  currency: string;
  reason: string;
};

export type NewWebhookEvent = {
  provider: string;
  externalEventId: string;
  type: string;
  payload: unknown;
};

export type NewInvoice = {
  tenantId: string;
  planCode: string;
  periodStart: string;
  periodEnd: string;
  amount: string;
  currencyCode: string;
  status: "open";
  dueAt: string;
};
```

### The three atomic claims, in SQL

```sql
-- claimTransferRetry: the claim IS the backoff (see reconciliation.md).
update transfers
   set retry_attempts = $2, retry_next_attempt_at = $3
 where id = $1
   and status = 'pending'
   and external_id is null
   and retry_next_attempt_at <= now()
returning id;

-- claimFeeAllocation: persist the share and win the row in one statement.
update vendor_orders
   set gateway_fee_amount = $2, gateway_fee_status = 'allocated'
 where id = $1
   and gateway_fee_status = 'pending'
returning id;
```

`claimSettlement` is the `fn_claim_settlement` function below — it needs the
lease arithmetic, so it stays a function rather than an inline statement.

