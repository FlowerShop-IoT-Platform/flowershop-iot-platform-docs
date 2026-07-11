# Epics, Stories & Tasks — Option C (Addendum)
## Vendor Self-Registration, Platform Subscription, Vase Ordering & Asset Recovery

> Companion to EPICS.md (EP-01–09, Mobile API) and EPICS-B.md (EP-10–18, Portals & Analytics).
> Epics are numbered EP-19 through EP-22 to avoid collision.
> Requirement refs point to REQ-GAPS-C.md.
> Analysis date: 2026-06-27

---

## Dependency Map

```
EP-01 (DB Foundation — from Option C)
  └── EP-19 (Vendor Self-Registration)
        └── EP-20 (Vendor Platform Subscription)

EP-15 (IoT — VaseHeartbeatMonitorService)
  └── EP-22 (Vase Asset Recovery)

EP-19 (self-registration = vendor portal login)
  └── EP-21 (Vase Ordering — vendor portal UI)

EP-11 (Admin Portal scaffolding)
  └── EP-21 (Vase Ordering — admin portal UI)
  └── EP-22 (Vase Asset Recovery — admin portal UI)
```

---

## EP-19: Vendor Self-Registration & Billing Onboarding

**Priority**: P0-critical
**Dependencies**: EP-01 (migrations), EP-04 / EP-17 Stripe service available
**Requirement refs**: REQ-VONB-01 through REQ-VONB-07

> The backend API endpoint for registration (`POST /api/v1/vendor-accounts/register-with-admin`)
> exists but is missing: Vendor Portal UI, billing address, Stripe customer creation, payment
> method capture, T&C tracking, and trial subscription kick-off. This epic fills those gaps.

### Stories

- **S-19-01**: Vendor Portal multi-step sign-up wizard (REQ-VONB-01)
- **S-19-02**: Extend registration API — billing address, online address, T&C (REQ-VONB-02, 03, 06)
- **S-19-03**: Stripe customer creation and payment method capture on registration (REQ-VONB-04, 05)
- **S-19-04**: Admin approval gating (optional, REQ-VONB-07)

### Tasks

| ID | Story | Requirement | Title |
|----|-------|-------------|-------|
| T-19-001 | S-19-01 | REQ-VONB-01 | Add "Sign Up" link to `VendorPortal/Pages/Login.cshtml` pointing to `/Account/Register` |
| T-19-002 | S-19-01 | REQ-VONB-01 | Create `VendorPortal/Pages/Account/Register.cshtml` — step 1: vendor type radio, business name, physical address (shop) / residential address (florist) |
| T-19-003 | S-19-01 | REQ-VONB-01 | Register step 2 (same page, JS-driven or separate page): billing address form (street, city, postal code, country) |
| T-19-004 | S-19-01 | REQ-VONB-01 | Register step 3: Stripe Payment Element (uses SetupIntent client secret from API); load `stripe.js`, initialize Payment Element |
| T-19-005 | S-19-01 | REQ-VONB-01 | Register step 4: T&C checkbox, privacy policy link, submit button; on success redirect to `/` (dashboard) |
| T-19-006 | S-19-02 | REQ-VONB-02 | Add `OnlineAddress` (URL, nullable) to `VendorWithAdminRegisterRequest` DTO, `FreelanceFlorist` aggregate, and EF config; generate migration `AddFreelanceFloristOnlineAddress` |
| T-19-007 | S-19-02 | REQ-VONB-03 | Add `BillingAddress` (street, city, postalCode, country) value object to both vendor aggregates and EF configs; generate migration `AddVendorBillingAddress` |
| T-19-008 | S-19-02 | REQ-VONB-06 | Add `TermsAcceptedAt` (DateTime?) and `TermsVersion` (string?) to both vendor aggregates; persist from registration request; generate migration `AddVendorTermsAccepted` |
| T-19-009 | S-19-03 | REQ-VONB-04 | Add `CreateCustomerAsync(email, name, billingAddress)` to `IStripeService` and implement in `StripeService`; returns Stripe `customerId` |
| T-19-010 | S-19-03 | REQ-VONB-04 | Add `StripeCustomerId` (string?) to both vendor aggregates and EF configs; migration `AddVendorStripeCustomerId`; call `CreateCustomerAsync` inside `VendorOnboardingService` after vendor creation |
| T-19-011 | S-19-03 | REQ-VONB-05 | Add `CreateSetupIntentAsync(stripeCustomerId)` to `IStripeService`; implement in `StripeService` |
| T-19-012 | S-19-03 | REQ-VONB-05 | Add `POST /api/v1/vendor-accounts/setup-intent` (anonymous) returning `{ clientSecret }` for the in-progress registration; VendorPortal step 3 calls this endpoint |
| T-19-013 | S-19-04 | REQ-VONB-07 | Add `ApprovalStatus` (enum: Pending / Approved / Rejected) to both vendor aggregates; default = `Pending` for self-registered, `Approved` for admin-created; migration `AddVendorApprovalStatus` |
| T-19-014 | S-19-04 | REQ-VONB-07 | Add `POST /api/v1/admin/vendors/{id}/approve` and `/reject` endpoints (PlatformAdmin); add Approve/Reject buttons to `AdminPortal/Pages/Vendors/Details.cshtml` |
| T-19-015 | S-19-04 | REQ-VONB-07 | Enforce `ApprovalStatus = Approved` in `CreateBouquetCommandHandler` — return `Result.Failure("vendor-pending-approval")` → 403 for Pending/Rejected vendors |

---

## EP-20: Vendor Platform Subscription Management (SaaS Billing)

**Priority**: P0-critical
**Dependencies**: EP-19 (Stripe customer must exist before creating a Stripe Subscription)
**Requirement refs**: REQ-VSUB-01 through REQ-VSUB-08

> Smart note: the existing `Subscription` aggregate is for **customer** bouquet-alert preferences
> (radius, flower type, shop) and is completely unrelated. This epic adds `VendorPlatformSubscription`,
> a distinct aggregate for recurring SaaS billing.

### Stories

- **S-20-01**: `VendorPlatformSubscription` domain aggregate + persistence (REQ-VSUB-01)
- **S-20-02**: Trial initiation on vendor registration (REQ-VSUB-02)
- **S-20-03**: Stripe Subscription methods + webhook handling (REQ-VSUB-03, REQ-VSUB-08)
- **S-20-04**: Access enforcement by subscription status (REQ-VSUB-04)
- **S-20-05**: Admin portal — subscription views (REQ-VSUB-05, REQ-VSUB-06)
- **S-20-06**: Vendor portal — billing page (REQ-VSUB-07)

### Tasks

| ID | Story | Requirement | Title |
|----|-------|-------------|-------|
| T-20-001 | S-20-01 | REQ-VSUB-01 | Create `VendorPlatformSubscription` aggregate in `FlowerShop.Domain/Aggregates/VendorSubscription/`; define all properties, `Status` enum (Trial / Active / PastDue / Cancelled / Suspended), and domain methods |
| T-20-002 | S-20-01 | REQ-VSUB-01 | Create `VendorSubscriptionId` strongly-typed ID value object |
| T-20-003 | S-20-01 | REQ-VSUB-01 | Add `IVendorSubscriptionRepository` (GetByVendorIdAsync, AddAsync, UpdateAsync) to domain; implement `VendorSubscriptionRepository` in Infrastructure |
| T-20-004 | S-20-01 | REQ-VSUB-01 | Create `VendorPlatformSubscriptionConfiguration` (EF Core); generate migration `AddVendorPlatformSubscriptions` |
| T-20-005 | S-20-02 | REQ-VSUB-02 | Add `CreateSubscriptionWithTrialAsync(customerId, priceId, trialDays)` to `IStripeService` and implement in `StripeService` |
| T-20-006 | S-20-02 | REQ-VSUB-02 | Add domain events: `VendorTrialStarted`, `VendorSubscriptionActivated`, `VendorSubscriptionCancelled`, `VendorSubscriptionSuspended` |
| T-20-007 | S-20-02 | REQ-VSUB-02 | In `VendorOnboardingService.RegisterVendorWithAdminAsync`, after Stripe customer creation: call `CreateSubscriptionWithTrialAsync`, create `VendorPlatformSubscription` in `Trial` status, persist it |
| T-20-008 | S-20-03 | REQ-VSUB-08 | Add `CancelSubscriptionAsync`, `UpdateSubscriptionPaymentMethodAsync`, `GetSubscriptionAsync` to `IStripeService` and implement in `StripeService` |
| T-20-009 | S-20-03 | REQ-VSUB-03 | Extend `StripeWebhookController` / `StripeWebhookProcessorService` to handle: `customer.subscription.updated`, `invoice.paid`, `invoice.payment_failed`, `customer.subscription.deleted`, `customer.subscription.trial_will_end` |
| T-20-010 | S-20-03 | REQ-VSUB-03 | Implement command handlers (`ActivateVendorSubscriptionCommand`, `SuspendVendorSubscriptionCommand`, `CancelVendorSubscriptionCommand`) called from webhook handlers |
| T-20-011 | S-20-03 | REQ-VSUB-03 | On `invoice.payment_failed`: notify vendor via push and email (use `INotificationServiceClient`); on `trial_will_end`: send 3-day warning notification |
| T-20-012 | S-20-04 | REQ-VSUB-04 | Inject `IVendorSubscriptionRepository` into `CreateBouquetCommandHandler`; guard: return `Result.Failure("subscription-inactive")` → 403 if status is `Cancelled` or `Suspended` |
| T-20-013 | S-20-05 | REQ-VSUB-05 | Add `GET /api/v1/admin/vendors/{id}/subscription` endpoint (PlatformAdmin) returning `VendorSubscriptionDto` |
| T-20-014 | S-20-05 | REQ-VSUB-05 | Extend `AdminPortal/Pages/Vendors/Details.cshtml` with a Subscription section: plan, status badge, trial start/end, billing dates |
| T-20-015 | S-20-05 | REQ-VSUB-06 | Add `GET /api/v1/admin/subscriptions` endpoint (PlatformAdmin, paginated, filterable by status / trialExpiringSoon) |
| T-20-016 | S-20-05 | REQ-VSUB-06 | Create `AdminPortal/Pages/Subscriptions/List.cshtml` — all vendor subscriptions with status filters and trial-expiry highlighting |
| T-20-017 | S-20-06 | REQ-VSUB-07 | Add `GET /api/v1/vendor/subscription` and `DELETE /api/v1/vendor/subscription` endpoints (VendorAccess) |
| T-20-018 | S-20-06 | REQ-VSUB-07 | Create `VendorPortal/Pages/Billing/Index.cshtml.cs`: show plan, status, trial countdown, next billing date; "Update payment method" (new SetupIntent flow); "Cancel" button |

---

## EP-21: Smart Vase Hardware Ordering & Fulfillment

**Priority**: P1-high
**Dependencies**: EP-19 (vendor portal login), EP-11 (Admin Portal scaffolding)
**Requirement refs**: REQ-VORD-01 through REQ-VORD-05

> Smart vases are physical assets that the platform ships to vendors. Currently there is no
> ordering or procurement flow — vases are registered by admins directly. This epic adds the
> full vendor-request → admin-fulfillment → delivery loop.

### Stories

- **S-21-01**: `VaseOrder` domain aggregate + persistence (REQ-VORD-01)
- **S-21-02**: VaseOrder API endpoints (REQ-VORD-02)
- **S-21-03**: Vendor portal — order new vases flow (REQ-VORD-03)
- **S-21-04**: Admin portal — vase order management (REQ-VORD-04)
- **S-21-05**: Admin portal — vase inventory per vendor (REQ-VORD-05)

### Tasks

| ID | Story | Requirement | Title |
|----|-------|-------------|-------|
| T-21-001 | S-21-01 | REQ-VORD-01 | Create `VaseOrder` aggregate in `FlowerShop.Domain/Aggregates/VaseOrder/`; define all properties, `VaseOrderStatus` enum, and domain methods (`Confirm`, `MarkShipped`, `MarkDelivered`, `Cancel`) |
| T-21-002 | S-21-01 | REQ-VORD-01 | Create `VaseOrderId` strongly-typed ID value object |
| T-21-003 | S-21-01 | REQ-VORD-01 | Add domain events: `VaseOrderPlaced`, `VaseOrderConfirmed`, `VaseOrderShipped`, `VaseOrderDelivered`, `VaseOrderCancelled` |
| T-21-004 | S-21-01 | REQ-VORD-01 | Add `IVaseOrderRepository` (AddAsync, GetByIdAsync, GetByVendorIdAsync, GetAllAsync) to domain; implement in Infrastructure |
| T-21-005 | S-21-01 | REQ-VORD-01 | Create `VaseOrderConfiguration` (EF Core); generate migration `AddVaseOrders` |
| T-21-006 | S-21-02 | REQ-VORD-02 | Create `PlaceVaseOrderCommand` + `PlaceVaseOrderCommandHandler` (validates quantity ≥ 1, shipping address required) |
| T-21-007 | S-21-02 | REQ-VORD-02 | Create `UpdateVaseOrderStatusCommand` + `UpdateVaseOrderStatusCommandHandler` (admin only, enforces status machine) |
| T-21-008 | S-21-02 | REQ-VORD-02 | Add `VendorVaseOrdersController` at `/api/v1/vendor/vase-orders` (VendorAccess): `POST` (place order) + `GET` (list own orders) |
| T-21-009 | S-21-02 | REQ-VORD-02 | Add `AdminVaseOrdersController` at `/api/v1/admin/vase-orders` (PlatformAdmin): `GET` (all orders, filterable) + `GET /{id}` (detail) + `PUT /{id}/status` (advance status + tracking number) |
| T-21-010 | S-21-03 | REQ-VORD-03 | Add "Order New Vases" button to `VendorPortal/Pages/Vases/Index.cshtml` |
| T-21-011 | S-21-03 | REQ-VORD-03 | Create `VendorPortal/Pages/Vases/Order.cshtml.cs` — quantity selector (1–10), shipping address form, submit → `POST /api/v1/vendor/vase-orders` |
| T-21-012 | S-21-03 | REQ-VORD-03 | Create `VendorPortal/Pages/Vases/Orders.cshtml.cs` — list vendor's vase orders with status badges and tracking numbers |
| T-21-013 | S-21-04 | REQ-VORD-04 | Create `AdminPortal/Pages/VaseOrders/List.cshtml.cs` — list all orders, filter by status; add nav link to sidebar |
| T-21-014 | S-21-04 | REQ-VORD-04 | Create `AdminPortal/Pages/VaseOrders/Details.cshtml.cs` — order detail, status-advance buttons (Confirm / Mark Shipped + tracking input / Mark Delivered), admin notes |
| T-21-015 | S-21-05 | REQ-VORD-05 | Add `GET /api/v1/admin/vendors/{id}/vase-summary` endpoint returning counts: ordered, delivered, online, offline, offline > 7 days; include per-vase list with `LastHeartbeat` |
| T-21-016 | S-21-05 | REQ-VORD-05 | Extend `AdminPortal/Pages/Vendors/Details.cshtml` with a "Vase Inventory" table using the vase-summary endpoint; highlight offline-7d rows in red; link to incident if one is open |

---

## EP-22: Smart Vase Asset Recovery & Offline Escalation

**Priority**: P1-high
**Dependencies**: EP-15 (VaseHeartbeatMonitorService must exist), EP-11 / EP-21 (Admin Portal)
**Requirement refs**: REQ-VASR-01 through REQ-VASR-06

> Smart vases are expensive IoT assets. A vase that has been offline for 7+ days is either
> abandoned, broken, or the vendor has stopped operating. This epic adds a structured case-
> management workflow: automatic incident creation, follow-up tracking (email, call, visit),
> return-request flow, and admin UI to manage and audit every step.

### Stories

- **S-22-01**: `VaseIncident` domain aggregate + persistence (REQ-VASR-01)
- **S-22-02**: Automatic incident creation on 7-day offline threshold (REQ-VASR-02)
- **S-22-03**: Backend API for incidents (REQ-VASR-06)
- **S-22-04**: Admin portal — incident list dashboard (REQ-VASR-03)
- **S-22-05**: Admin portal — incident detail & action log (REQ-VASR-04)
- **S-22-06**: Return-request tracking through to asset recovery (REQ-VASR-05)

### Tasks

| ID | Story | Requirement | Title |
|----|-------|-------------|-------|
| T-22-001 | S-22-01 | REQ-VASR-01 | Create `VaseIncident` aggregate in `FlowerShop.Domain/Aggregates/VaseIncident/`; define `IncidentType` and `IncidentStatus` enums; `VaseIncidentAction` owned entity with `ActionType` enum |
| T-22-002 | S-22-01 | REQ-VASR-01 | Create `VaseIncidentId` strongly-typed ID value object |
| T-22-003 | S-22-01 | REQ-VASR-01 | Add domain methods to `VaseIncident`: `AddAction`, `RequestReturn`, `Resolve`, `Escalate`; each mutates `Status` and appends an action entry |
| T-22-004 | S-22-01 | REQ-VASR-01 | Add domain events: `VaseIncidentOpened`, `VaseIncidentEscalated`, `VaseReturnRequested` |
| T-22-005 | S-22-01 | REQ-VASR-01 | Add `IVaseIncidentRepository` (AddAsync, GetByIdAsync, GetByVaseIdAsync, GetOpenIncidentsAsync) to domain; implement in Infrastructure |
| T-22-006 | S-22-01 | REQ-VASR-01 | Create `VaseIncidentConfiguration` (EF Core, with owned-entity collection for Actions); generate migration `AddVaseIncidents` |
| T-22-007 | S-22-02 | REQ-VASR-02 | In `VaseHeartbeatMonitorService`, add a secondary pass (run every hour) querying `ISmartVaseRepository.GetVasesOfflineSinceAsync(cutoffDays: 7)` |
| T-22-008 | S-22-02 | REQ-VASR-02 | For each vase in that result with no open `VaseIncident`: create `VaseIncident`, persist, dispatch `VaseIncidentOpened` |
| T-22-009 | S-22-02 | REQ-VASR-02 | Implement `VaseIncidentOpenedHandler`: send vendor email/push notification ("Your vase has been offline for 7+ days — please reconnect or contact support"); add initial `EmailSent` action to the incident |
| T-22-010 | S-22-02 | REQ-VASR-02 | Add `GetVasesOfflineSinceAsync(cutoffDays)` to `ISmartVaseRepository` and implement in `SmartVaseRepository` |
| T-22-011 | S-22-03 | REQ-VASR-06 | Create `AdminVaseIncidentsController` at `/api/v1/admin/vase-incidents` (PlatformAdmin): `GET` (list, paginated, filterable by status); `GET /{id}` (detail); `POST /{id}/actions` (add action); `PUT /{id}/status` (change status) |
| T-22-012 | S-22-03 | REQ-VASR-06 | Implement `GetVaseIncidentsQuery` + `GetVaseIncidentsQueryHandler`; `AddVaseIncidentActionCommand` + handler; `UpdateVaseIncidentStatusCommand` + handler |
| T-22-013 | S-22-04 | REQ-VASR-03 | Create `AdminPortal/Pages/VaseIncidents/List.cshtml.cs` — columns: vase serial, vendor name, offline-since, days offline, status, last action type, next planned action |
| T-22-014 | S-22-04 | REQ-VASR-03 | Filter bar on incident list: status, vendorId; default sort: days offline descending |
| T-22-015 | S-22-04 | REQ-VASR-03 | Add "Incidents" nav item to `AdminPortal/Pages/Shared/_Layout.cshtml`; show badge count for Open + Escalated incidents |
| T-22-016 | S-22-05 | REQ-VASR-04 | Create `AdminPortal/Pages/VaseIncidents/Details.cshtml.cs` — vase info header, chronological action timeline |
| T-22-017 | S-22-05 | REQ-VASR-04 | "Add action" form on details page: `ActionType` dropdown (EmailSent, CallMade, VisitPlanned, VisitCompleted, AdminNote), notes textarea, optional `PlannedAt` date picker |
| T-22-018 | S-22-05 | REQ-VASR-04 | "Request Return" button: confirms via modal → calls `PUT /{id}/status` (ReturnRequested) → emails vendor with return instructions; records `ReturnRequested` action |
| T-22-019 | S-22-05 | REQ-VASR-04 | "Escalate" button: confirms → marks `Escalated` → surfaces prominently in list (red badge) |
| T-22-020 | S-22-05 | REQ-VASR-04 | "Resolve" button: text input for resolution notes → marks `Resolved` + `ResolvedAt` |
| T-22-021 | S-22-06 | REQ-VASR-05 | On "ReturnShipped" action: admin inputs tracking number; update incident action record |
| T-22-022 | S-22-06 | REQ-VASR-05 | On "ReturnReceived" action: auto-call `SmartVase.UnassignFromVendor()` and set `RegistrationStatus = UnderMaintenance`; close the incident with status `Resolved` |
| T-22-023 | S-22-06 | REQ-VASR-05 | If vendor responds (admin records `VendorResponded` action) and the vase comes back online within 14 days, automatically resolve the incident without return |

---

## EP-23: Customer Social Sign-In (Google SSO) ✅ Done

**Priority**: P0-critical
**Dependencies**: EP-02 (Customer Auth — Keycloak, customer_id claim, CustomerOnboardingService)
**Requirement refs**: REQ-SSO-01 through REQ-SSO-04
**Design doc**: `docs/superpowers/specs/2026-07-05-google-sso-customer-design.md`
**Setup guide**: `docker/keycloak/README.md` (Google Cloud + local + production)
**Status**: Implemented and browser-verified (Playwright) on branch `feat/ep23-google-sso-customer`
against live Keycloak + a real Google OAuth client. See `tasks-ep23-google-sso-customer.json`
`completionNotes` for the per-task deltas.

> Customers can sign in with Google instead of manually creating an email/password account.
> Keycloak already brokers social logins and both clients forward `kc_idp_hint`, so the OAuth
> handshake is not the work. The gaps are: no Google IdP in the (manually-built, non-exported)
> realm, and a Google first-login produces a Keycloak user with no local `Customer` and no
> `customer_id` claim — so every `CustomerAccess` endpoint fails. This epic adds a committed
> realm export with a Google IdP (auto-linking existing users by verified email, skipping email
> verification), backend just-in-time `Customer` provisioning wired into `TenantResolutionMiddleware`,
> and a "Continue with Google" button. Covers the Web Customer App (this repo) and the mobile app
> (Keycloak/backend config only). Vendor Portal is a fast follow reusing the same mechanism.

### Stories

- **S-23-01**: Google Identity Provider in Keycloak realm — committed export (REQ-SSO-01)
- **S-23-02**: Backend just-in-time Customer provisioning on first social login (REQ-SSO-02)
- **S-23-03**: Customer App "Continue with Google" button (REQ-SSO-03)
- **S-23-04**: Mobile app Google sign-in documentation (REQ-SSO-04)

### Tasks

| ID | Story | Requirement | Title |
|----|-------|-------------|-------|
| T-23-001 | S-23-02 | REQ-SSO-02 | Write failing unit tests for `CustomerProvisioningService` (provision / auto-link / idempotent / not-for-vendor) |
| T-23-002 | S-23-02 | REQ-SSO-02 | Implement `ICustomerProvisioningService` — atomic Customer + User creation, `customer_id` write-back to Keycloak; register in DI |
| T-23-003 | S-23-02 | REQ-SSO-02 | Invoke provisioning from `TenantResolutionMiddleware`; inject `customer_id` claim into the in-flight request |
| T-23-004 | S-23-01 | REQ-SSO-01 | Export the `flowershop` realm to `docker/keycloak/flowershop-realm.json` (or author a minimal export) |
| T-23-005 | S-23-01 | REQ-SSO-01 | Add Google `identityProvider` to the export — `trustEmail`, auto-link-by-verified-email, env-var placeholder secrets |
| T-23-006 | S-23-01 | REQ-SSO-01 | Switch `keycloak-dev` to `start-dev --import-realm`, mount import dir, add `GOOGLE_CLIENT_ID`/`SECRET` env vars |
| T-23-007 | S-23-03 | REQ-SSO-03 | Add "Continue with Google" button to the Customer App login (`?provider=google`) — already existed; evolved into T-23-011 |
| T-23-008 | S-23-02 | REQ-SSO-02 | ⏳ Integration test: social identity provisions on first API call — **deferred** (unit + manual Playwright E2E cover it) |
| T-23-009 | S-23-04 | REQ-SSO-04 | Update `API-CONTRACTS.md` AUTH-03 note to reflect Google sign-in is live |
| T-23-010 | S-23-03 | REQ-SSO-02 | Blank the placeholder phone (`+10000000000`) in the customer profile form (commit c40a8b9) |
| T-23-011 | S-23-03 | REQ-SSO-03 | In-context login modal on the map — Google + email options, all redirect to Keycloak (commit 5a977b0) |

> **Delivery notes:** All stories done. T-23-007's button already existed, so the UI work went
> into the on-map login modal (T-23-011). T-23-008 (automated integration test) is the only open
> item, deferred in favour of 5 unit tests plus a full manual Playwright E2E run. A literal
> iframe-embed of the Keycloak/Google login on the map is not possible (`X-Frame-Options: DENY`),
> so the modal presents our own buttons that redirect — provider redirect is inherent to OAuth.

---

## Epic Dependency Map (Full — all epics)

```
EP-01 (DB Foundation)
  ├── EP-02 (Customer Auth) ──► EP-23 (Customer Google SSO)
  ├── EP-03 (Bouquet Discovery)
  ├── EP-04 (Orders + Stripe) ──► EP-05 (Vendor Order Mgmt)
  │                                      └── EP-12 (Vendor Portal)
  ├── EP-06 (Fav/Sub/Notifications)
  ├── EP-07 (Customer Profile)
  ├── EP-08 (Real-time + Freshness) ← needs EP-05
  ├── EP-09 (Performance) ──► EP-10 (Analytics API)
  │                                  └── EP-11 (Admin Portal)
  │                                        ├── EP-21 (Vase Ordering — admin UI)
  │                                        └── EP-22 (Asset Recovery — admin UI)
  └── EP-19 (Vendor Self-Registration)
        └── EP-20 (Vendor Platform Subscription)

EP-13 (Customer Web Shopping)
  └── EP-14 (Customer Web Profile)

EP-15 (IoT Integration)
  └── EP-22 (Vase Asset Recovery — extends heartbeat monitor)

EP-16 (E2E Testing) — independent
EP-17 (Payment Resilience) ──► EP-18 (Payment Visibility & Card)
EP-21 (Vase Ordering — vendor portal UI) ← needs EP-19 (vendor login)
```

---

## Summary

| Epic | Priority | Stories | Tasks | Key Dependency | Status |
|------|----------|---------|-------|----------------|--------|
| EP-19 Vendor Self-Registration | P0-critical | 4 | 15 | EP-01, Stripe (EP-04/17) | ✅ Done |
| EP-20 Vendor Platform Subscription | P0-critical | 6 | 18 | EP-19 | ✅ Done |
| EP-21 Vase Ordering & Fulfillment | P1-high | 5 | 16 | EP-19, EP-11 | ✅ Done |
| EP-22 Vase Asset Recovery | P1-high | 6 | 23 | EP-15, EP-11/21 | ✅ Done |
| EP-23 Customer Google SSO | P0-critical | 4 | 11 | EP-02 | ✅ Done (1 test deferred) |
| **Total (EP-19–23)** | | **25** | **83** | | |
| **Grand total (EP-01–23)** | | **99** | **269** | | |

### Implementation status (2026-06-28)

All four epics (21 stories, 72 tasks) are **implemented and browser-verified** on branch
`feat/ep19-22-vendor-onboarding-subscription-vase` (PR #20). Per-task state is tracked in the
`tasks-ep19..22-*.json` files (all marked `done`); see each file's `completionNotes` for specifics.

Migrations (one per epic, applied to `flowershop_iot_dev`): `AddVendorOnboardingBilling`,
`AddVendorPlatformSubscriptions`, `AddVaseOrdersAndIncidents`.

Notable deltas from the original plan:
- **EP-20 T-20-018** ("Update payment method") shipped initially as a placeholder link and was
  completed afterwards as a real `Billing/PaymentMethod` page — a Stripe **SetupIntent** Payment
  Element plus an editable, persisted **billing address** (endpoints
  `POST /api/v1/vendor/subscription/payment-method/setup-intent` and
  `PUT /api/v1/vendor/subscription/billing-address`). Card capture degrades gracefully when
  `Stripe:PublishableKey` is unset; billing-address save verified persisting to the DB.
- **EP-19** required a fix to `VendorPortal/Filters/SessionExpiredFilter.cs` to whitelist the
  anonymous `/Account/Register` wizard (it was otherwise redirected to `/Login`).
- **EP-22** extended the existing `VaseHealthMonitoringService` (in `Infrastructure/Monitoring/`)
  rather than the `VaseHeartbeatMonitorService` named in the plan.
