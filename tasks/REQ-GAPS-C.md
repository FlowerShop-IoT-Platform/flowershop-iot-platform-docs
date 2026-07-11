# Requirements Gap Analysis — Option C (Addendum)
## Vendor Self-Registration, Platform Subscription, Vase Ordering & Asset Recovery

> Companion to REQ-GAPS.md (Mobile API / Option C) and REQ-GAPS-B.md (Portals / Option B).
> Scope: vendor self-onboarding, SaaS subscription billing, smart vase hardware ordering,
>         and offline vase escalation / asset-recovery workflows.
> Priority legend: P0 = critical blocker, P1 = high value, P2 = nice to have.
> Analysis date: 2026-06-27

---

## Terminology note

The existing `Subscription` aggregate (`/src/backend/FlowerShop.Domain/Aggregates/Subscription/Subscription.cs`)
handles **customer** bouquet-alert preferences (flower type, radius, shop). This document uses
"platform subscription" or "vendor billing subscription" for the entirely separate SaaS billing
concept. The two are not related.

---

## Area 1 — Vendor Self-Registration (Portal UI & API Gaps)

### REQ-VONB-01 `[MISSING]` Vendor Portal sign-up page — P0

`POST /api/v1/vendor-accounts/register-with-admin` exists and is anonymous, but there is **no
UI entry point** in the Vendor Portal. `VendorPortal/Pages/Account/` contains only `ForgotPassword`
and `ResendVerification` pages. Vendors who visit the portal have no way to start the registration
flow without calling the raw API.

Required: multi-step wizard in `VendorPortal/Pages/Account/Register.cshtml` covering:
- Step 1 — Vendor type + business name + physical address (for flower shops) **or** website URL
  (for freelance florists with no physical shop)
- Step 2 — Billing address (separate from shop/residential address)
- Step 3 — Payment details (card via Stripe Setup Intent) for post-trial billing
- Step 4 — Terms & Conditions acceptance + submit

The Sign-up link must appear on the Login page.

### REQ-VONB-02 `[MISSING]` Online address support for freelancers — P1

`VendorWithAdminRegisterRequest` (in `VendorAccountsController.cs`) accepts `FullAddress`
(physical location) but has no field for an online / social-media address or website URL.
A freelance florist who sells only online has no way to provide this.

Required: add `OnlineAddress` (URL, nullable) to the request DTO; store it on the
`FreelanceFlorist` aggregate and its EF configuration.

### REQ-VONB-03 `[MISSING]` Billing address fields in registration request — P1

Neither the registration request DTO nor the vendor domain aggregates have a billing address
separate from the shop/residential address. Platform invoices and Stripe billing need a dedic-
ated billing address.

Required: add `BillingAddress` (street, city, postalCode, country) to:
- `VendorWithAdminRegisterRequest`
- `FlowerShop` / `FreelanceFlorist` aggregates
- EF configurations and a migration

### REQ-VONB-04 `[MISSING]` Stripe customer creation on vendor registration — P0

When a new vendor registers, no Stripe Customer record is created and no `StripeCustomerId` is
stored on the vendor aggregate. Without a Stripe Customer the platform cannot:
- save the payment method provided at sign-up
- create a Stripe Subscription for recurring billing
- issue invoices / receipts

Required:
- Extend `IStripeService` with `CreateCustomerAsync(email, name, billingAddress)`.
- Call it inside `VendorOnboardingService.RegisterVendorWithAdminAsync()` after vendor creation.
- Persist `StripeCustomerId` on both `FlowerShop` and `FreelanceFlorist` aggregates plus a
  migration column.

### REQ-VONB-05 `[MISSING]` Stripe SetupIntent for payment method capture at sign-up — P1

After the Stripe Customer exists, the registration flow must capture the payment method (card or
bank account) so that billing can start automatically after the trial period without manual
re-entry.

Required:
- Extend `IStripeService` with `CreateSetupIntentAsync(stripeCustomerId)` → returns
  `clientSecret` for the Stripe Payment Element.
- Add `POST /api/v1/vendor-accounts/setup-intent` (anonymous, takes vendorId from the in-
  progress registration session) that returns the client secret.
- Vendor Portal wizard step 3 renders the Stripe Payment Element using this secret.

### REQ-VONB-06 `[MISSING]` Terms & Conditions acceptance tracking — P1

There is no record of when (or whether) a vendor accepted the platform terms. This is a legal
and GDPR requirement.

Required: add `TermsAcceptedAt` (DateTime?) and `TermsVersion` (string?) to both vendor
aggregates and persist them; populate from the registration request.

### REQ-VONB-07 `[MISSING]` Admin approval step (optional gating) — P2

Currently a self-registered vendor is immediately active. The platform may want to review new
vendors before they appear on the customer map.

Required: add `ApprovalStatus` (Pending / Approved / Rejected) to vendor aggregates;
set to `Pending` on self-registration, `Approved` when an admin clicks Approve in the Admin
Portal. Vendors in `Pending` status cannot publish bouquets (bouquet creation returns 403).

---

## Area 2 — Vendor Platform Subscription (SaaS Billing)

### REQ-VSUB-01 `[MISSING]` VendorPlatformSubscription domain aggregate — P0

No domain model exists for a vendor's recurring SaaS billing plan. Required aggregate:

```
VendorPlatformSubscription
  VendorId, VendorType
  PlanId (string — e.g. "starter", "pro")
  Status (enum): Trial | Active | PastDue | Cancelled | Suspended
  StripeSubscriptionId (string)
  StripeCustomerId (string)
  TrialStartedAt (DateTime)
  TrialEndsAt (DateTime)
  CurrentPeriodStart (DateTime)
  CurrentPeriodEnd (DateTime)
  CancelledAt (DateTime?)
  SuspendedAt (DateTime?)
```

Domain methods: `StartTrial(trialDays)`, `Activate(stripeSubscriptionId)`,
`MarkPastDue()`, `Suspend()`, `Cancel()`, `Reinstate()`.

### REQ-VSUB-02 `[MISSING]` Trial period initiation on registration — P0

When a vendor completes registration, the platform must automatically start a 30-day trial
by creating a `VendorPlatformSubscription` in `Trial` status and a Stripe Subscription with
a 30-day `trial_end` value. No charge occurs during the trial.

Required:
- Call `IStripeSubscriptionService.CreateSubscriptionWithTrialAsync(stripeCustomerId, planId,
  trialDays: 30)` inside `VendorOnboardingService`.
- Persist the resulting `VendorPlatformSubscription` record (new table + migration).

### REQ-VSUB-03 `[MISSING]` Stripe Subscription webhook handling — P0

After the trial ends, Stripe fires events that must update the local subscription record.
Required webhook handlers (extend `StripeWebhookController`):

| Stripe event | Local action |
|---|---|
| `customer.subscription.updated` | Update status, period dates |
| `invoice.paid` | Mark `Active`, reset `PastDue` |
| `invoice.payment_failed` | Mark `PastDue`; notify vendor |
| `customer.subscription.deleted` | Mark `Cancelled` |
| `customer.subscription.trial_will_end` | Notify vendor (3 days before) |

### REQ-VSUB-04 `[MISSING]` Access enforcement based on subscription status — P1

Vendors in `Cancelled` or `Suspended` status must not be able to create new bouquets or orders.
Required:
- Inject `IVendorSubscriptionRepository` (get by VendorId) into `CreateBouquetCommandHandler`.
- Return `Result.Failure("subscription-inactive")` if status is `Cancelled` or `Suspended`.
- Controller maps to 403 Forbidden.

### REQ-VSUB-05 `[MISSING]` Admin portal — subscription view per vendor — P1

Admin Portal `Vendors/Details.cshtml` shows only address / vase count. Platform admins need to
see each vendor's subscription status inline.

Required fields to display: plan name, status badge (Trial / Active / PastDue / Cancelled /
Suspended), trial start date, trial end date, last billing date, next billing date.

Required: new `ApiClient.GetVendorSubscriptionAsync(vendorId)` calling
`GET /api/v1/admin/vendors/{id}/subscription` (PlatformAdmin policy).

### REQ-VSUB-06 `[MISSING]` Admin portal — subscription list with filters — P1

Platform admins need a global view of all subscriptions for renewals, churn monitoring, and
revenue forecasting.

Required: new Admin Portal page `Subscriptions/List.cshtml` calling
`GET /api/v1/admin/subscriptions` with filters: `status`, `trialExpiringSoon` (7 days),
`planId`.

### REQ-VSUB-07 `[MISSING]` Vendor portal — billing / subscription page — P1

Vendors need to see their current plan, next payment date, and billing history; and they must
be able to update their payment method or cancel.

Required: new `VendorPortal/Pages/Billing/Index.cshtml` with:
- Current plan and status
- Trial countdown (if applicable)
- Next billing date and amount
- "Update payment method" → new Stripe SetupIntent flow
- "Cancel subscription" → confirmation → `DELETE /api/v1/vendor/subscription`

### REQ-VSUB-08 `[MISSING]` Stripe Subscription service extension — P0

`IStripeService` currently handles only one-time PaymentIntents and Refunds. Subscription
billing requires additional methods:

```csharp
CreateSubscriptionWithTrialAsync(string customerId, string priceId, int trialDays)
  → (subscriptionId, clientSecret, trialEnd)

UpdateSubscriptionPaymentMethodAsync(string subscriptionId, string paymentMethodId)

CancelSubscriptionAsync(string subscriptionId)

GetSubscriptionAsync(string subscriptionId)
```

Implement in `StripeService`, wrapped in the same Polly resilience pipeline.

---

## Area 3 — Smart Vase Hardware Ordering

### REQ-VORD-01 `[MISSING]` VaseOrder domain aggregate — P0

There is no way for a vendor to request new smart vases. Vases are registered by platform
admins only. A vendor-facing order flow requires a new aggregate:

```
VaseOrder
  VaseOrderId
  VendorId, VendorType
  Quantity (int)
  ShippingAddress (Address value object)
  Status (enum): Requested | Confirmed | Shipped | Delivered | Cancelled
  Notes (string?)
  OrderedAt (DateTime)
  ConfirmedAt (DateTime?)
  ShippedAt (DateTime?)
  DeliveredAt (DateTime?)
  TrackingNumber (string?)
  AdminNotes (string?)
```

Domain methods: `Confirm(adminNotes?)`, `MarkShipped(trackingNumber)`,
`MarkDelivered()`, `Cancel(reason)`.

Domain events: `VaseOrderPlaced`, `VaseOrderConfirmed`, `VaseOrderShipped`,
`VaseOrderDelivered`, `VaseOrderCancelled`.

### REQ-VORD-02 `[MISSING]` VaseOrder API endpoints — P0

Required backend endpoints:

| Verb | Path | Policy | Description |
|---|---|---|---|
| `POST` | `/api/v1/vendor/vase-orders` | VendorAccess | Vendor places order |
| `GET` | `/api/v1/vendor/vase-orders` | VendorAccess | Vendor lists own orders |
| `GET` | `/api/v1/admin/vase-orders` | PlatformAdmin | Admin lists all orders |
| `GET` | `/api/v1/admin/vase-orders/{id}` | PlatformAdmin | Admin views order detail |
| `PUT` | `/api/v1/admin/vase-orders/{id}/status` | PlatformAdmin | Admin advances status |

### REQ-VORD-03 `[MISSING]` Vendor portal — order new vases — P1

Vendors see their assigned vases in `Vases/Index.cshtml` but have no way to request
additional ones.

Required:
- New button "Order New Vases" on `Vases/Index.cshtml`.
- New page `Vases/Order.cshtml.cs` (quantity + shipping address form).
- New page `Vases/Orders.cshtml.cs` listing all vase orders for this vendor with status
  badges (Requested / Confirmed / Shipped / Delivered).

### REQ-VORD-04 `[MISSING]` Admin portal — vase order management — P1

Platform admins must be able to see all pending hardware requests and advance their status.

Required:
- New Admin Portal section `VaseOrders/List.cshtml` — all orders, filterable by status.
- New page `VaseOrders/Details.cshtml` — order detail with status-advance buttons (Confirm,
  Mark Shipped + tracking number input, Mark Delivered), admin notes field.

### REQ-VORD-05 `[MISSING]` Admin portal — vase inventory per vendor — P1

Admin Portal `Vendors/Details.cshtml` currently shows total vase count. Admins need a
breakdown showing the full asset picture for each vendor:

| Column | Source |
|---|---|
| Ordered (not yet delivered) | VaseOrders with status ≠ Delivered/Cancelled |
| Delivered (registered to vendor) | SmartVases with Assignment.VendorId |
| Online | SmartVases where ConnectionStatus = Online |
| Offline | SmartVases where ConnectionStatus = Offline |
| Offline > 7 days | SmartVases where GoOfflineAt < now − 7 days |
| Last seen | SmartVase.LastHeartbeat |

A vase offline > 7 days must be visually highlighted (red row) with a quick link to the
incident record (see Area 4).

---

## Area 4 — Smart Vase Asset Recovery & Offline Escalation

### REQ-VASR-01 `[MISSING]` VaseIncident domain aggregate — P0

When a vase has been offline for 7+ days with no vendor response, the platform must open a
case and track all follow-up activity. Required aggregate:

```
VaseIncident
  VaseIncidentId
  VaseId, VendorId, VendorType
  IncidentType (enum): ProlongedOffline | ReturnRequested | HardwareDefect
  Status (enum): Open | FollowUpSent | ReturnRequested | Resolved | Escalated
  OpenedAt (DateTime)
  ResolvedAt (DateTime?)
  ResolutionNotes (string?)
  Actions: List<VaseIncidentAction>
```

`VaseIncidentAction`:
```
ActionType (enum): EmailSent | CallMade | VisitPlanned | VisitCompleted |
                   ReturnRequested | ReturnShipped | ReturnReceived |
                   VendorResponded | AdminNote
Notes (string)
ActorUserId
PerformedAt (DateTime)
PlannedAt (DateTime?)   — for VisitPlanned
```

Domain methods: `AddAction(action)`, `RequestReturn(notes)`,
`Resolve(notes)`, `Escalate()`.

Domain events: `VaseIncidentOpened`, `VaseIncidentEscalated`, `VaseReturnRequested`.

### REQ-VASR-02 `[MISSING]` Automatic incident opening on 7-day offline threshold — P0

Extend `VaseHeartbeatMonitorService` (EP-15 S-15-03) with a secondary check: after marking
a vase offline, query vases that have been offline for ≥ 7 days and have no open
`VaseIncident`. For each: create a `VaseIncident` with `IncidentType = ProlongedOffline`
and dispatch `VaseIncidentOpened`.

On `VaseIncidentOpened`, the event handler must:
- Send an email/push to the vendor explaining the issue and requesting a response.
- Create an initial `VaseIncidentAction` of type `EmailSent` recording the auto-notification.

### REQ-VASR-03 `[MISSING]` Admin portal — incident list & dashboard — P1

Platform admins need a dedicated page showing all open vase incidents.

Required: new Admin Portal page `VaseIncidents/List.cshtml`:
- Columns: vase serial number, vendor name, offline since, days offline, status, last action,
  next planned action.
- Filter by status (Open / FollowUpSent / ReturnRequested / Escalated).
- Sort by "days offline" descending to surface worst cases first.
- "Open incident" from Vendors/Details.cshtml vase table (see REQ-VORD-05).

### REQ-VASR-04 `[MISSING]` Admin portal — incident detail & action log — P1

Required: new Admin Portal page `VaseIncidents/Details.cshtml`:
- Vase info (serial, vendor, offline-since).
- Chronological action timeline.
- "Add action" form: action type selector, notes, optional planned date.
- "Request Return" button → sets `Status = ReturnRequested` + creates `ReturnRequested`
  action + emails vendor.
- "Resolve" button → marks incident resolved + notes.
- "Escalate" button → marks incident Escalated, surfaces it prominently in the list.

### REQ-VASR-05 `[MISSING]` Return-request tracking — P1

Once a return is requested and the vase is shipped back:
- Admin records `ReturnShipped` action with tracking number.
- On physical receipt, admin records `ReturnReceived`.
- System automatically calls `SmartVase.UnassignFromVendor()` and marks the vase
  `RegistrationStatus = UnderMaintenance`.

### REQ-VASR-06 `[MISSING]` Backend API for vase incidents — P1

Required endpoints:

| Verb | Path | Policy | Description |
|---|---|---|---|
| `GET` | `/api/v1/admin/vase-incidents` | PlatformAdmin | List incidents (paginated, filterable) |
| `GET` | `/api/v1/admin/vase-incidents/{id}` | PlatformAdmin | Incident detail + action log |
| `POST` | `/api/v1/admin/vase-incidents/{id}/actions` | PlatformAdmin | Add action to incident |
| `PUT` | `/api/v1/admin/vase-incidents/{id}/status` | PlatformAdmin | Change incident status |

---

## Area 5 — Customer Social Sign-In (Google SSO)

> Scope: let customers authenticate with a public SSO provider (Google first) instead of
> creating an email/password account manually. Keycloak brokers the OAuth flow; the platform
> gap is provisioning the local `Customer` on first social login and wiring the IdP config
> reproducibly. Covers the **Web Customer App** (this repo) and the **mobile app** (Keycloak /
> backend config only — mobile client code is not in this repo). Vendor Portal is a fast follow.

### REQ-SSO-01 `[MISSING]` Google Identity Provider in Keycloak realm — P0

The `flowershop` realm has no social Identity Provider configured, and the realm itself is built
manually (no committed export), so a Google IdP cannot be reproduced across environments.

Required: add a Google `identityProvider` to the realm with `trustEmail: true` and a first-broker-
login flow that automatically links an existing user by verified email (Google asserts verified
email, so no password prompt). Deliver as a committed realm export
(`docker/keycloak/flowershop-realm.json`) imported on container startup (`start-dev --import-realm`),
with `clientId` / `clientSecret` sourced from env-var placeholders (never committed).

### REQ-SSO-02 `[MISSING]` Just-in-time local Customer provisioning on first social login — P0

Email/password registration (`CustomerOnboardingService`) creates a Keycloak user **and** a local
`Customer` + `User`, then writes the `customer_id` Keycloak attribute so it appears in every JWT.
A Google first-login creates only a Keycloak user — no local `Customer`, no `customer_id` claim —
so every `[Authorize(CustomerAccess)]` endpoint fails.

Required: a backend `ICustomerProvisioningService` invoked from `TenantResolutionMiddleware` that,
for an authenticated customer-type identity with no linked local user, atomically creates the
`Customer` + `User`, writes the `customer_id` attribute back to Keycloak, and injects the
`customer_id` claim into the in-flight request. Idempotent and safe under concurrent first-requests
(in-transaction re-check + unique email). Reuses the existing placeholder-phone + address so users
flow into the existing `RequireCompleteProfileAttribute` profile-completion path.

### REQ-SSO-03 `[MISSING]` "Continue with Google" button in Customer App login — P1

`AccountController.Login(provider)` already forwards `provider` as `kc_idp_hint`, but no UI entry
point exists. Required: a Google-branded "Continue with Google" button on the Customer App login
page linking to `Account/Login?provider=google&returnUrl=...`.

### REQ-SSO-04 `[DOCS]` Mobile app Google sign-in documentation — P2

`API-CONTRACTS.md` documents Google as a Keycloak IdP rendered on the hosted login page. Once
REQ-SSO-01 is live, the existing MAUI PKCE flow surfaces the Google button automatically. Required:
update the contract note to reflect that Google sign-in is live (not merely "configured").

---

## Summary Table

| ID | Area | Priority | Status |
|---|---|---|---|
| REQ-VONB-01 | Vendor Portal sign-up page | P0 | MISSING |
| REQ-VONB-02 | Online address for freelancers | P1 | MISSING |
| REQ-VONB-03 | Billing address in registration | P1 | MISSING |
| REQ-VONB-04 | Stripe customer creation on registration | P0 | MISSING |
| REQ-VONB-05 | Stripe SetupIntent for card capture at sign-up | P1 | MISSING |
| REQ-VONB-06 | Terms & Conditions acceptance tracking | P1 | MISSING |
| REQ-VONB-07 | Admin approval gating | P2 | MISSING |
| REQ-VSUB-01 | VendorPlatformSubscription domain aggregate | P0 | MISSING |
| REQ-VSUB-02 | 30-day trial initiation on registration | P0 | MISSING |
| REQ-VSUB-03 | Stripe Subscription webhook handling | P0 | MISSING |
| REQ-VSUB-04 | Access enforcement by subscription status | P1 | MISSING |
| REQ-VSUB-05 | Admin portal — subscription view per vendor | P1 | MISSING |
| REQ-VSUB-06 | Admin portal — subscription list with filters | P1 | MISSING |
| REQ-VSUB-07 | Vendor portal — billing / subscription page | P1 | MISSING |
| REQ-VSUB-08 | IStripeService subscription methods | P0 | MISSING |
| REQ-VORD-01 | VaseOrder domain aggregate | P0 | MISSING |
| REQ-VORD-02 | VaseOrder API endpoints | P0 | MISSING |
| REQ-VORD-03 | Vendor portal — order new vases | P1 | MISSING |
| REQ-VORD-04 | Admin portal — vase order management | P1 | MISSING |
| REQ-VORD-05 | Admin portal — vase inventory per vendor | P1 | MISSING |
| REQ-VASR-01 | VaseIncident domain aggregate | P0 | MISSING |
| REQ-VASR-02 | Automatic incident on 7-day offline threshold | P0 | MISSING |
| REQ-VASR-03 | Admin portal — incident list & dashboard | P1 | MISSING |
| REQ-VASR-04 | Admin portal — incident detail & action log | P1 | MISSING |
| REQ-VASR-05 | Return-request tracking | P1 | MISSING |
| REQ-VASR-06 | Backend API for vase incidents | P1 | MISSING |
| REQ-SSO-01 | Google Identity Provider in Keycloak realm | P0 | DONE |
| REQ-SSO-02 | JIT local Customer provisioning on social login | P0 | DONE |
| REQ-SSO-03 | "Continue with Google" button in Customer App | P1 | DONE (on-map login modal) |
| REQ-SSO-04 | Mobile app Google sign-in documentation | P2 | DONE |
