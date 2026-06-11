# Epics, Stories & Tasks — Option B
## Admin Portal, Vendor Portal, Customer Web App, Analytics API, IoT Integration

> Companion to EPICS.md (Option C — Mobile API, EP-01 through EP-09).
> Epics are numbered EP-10 through EP-15 to avoid collision.

---

## Dependency Map

```
EP-09 (Performance/Crosscutting — from Option C)
  └── EP-10 (Analytics API)
        └── EP-11 (Admin Portal Analytics UI)
              └── EP-12 (Vendor Portal — Orders & Analytics UI)

EP-05 (Vendor Order Management — from Option C)
  └── EP-12 (Vendor Portal — Orders & Analytics UI)

EP-01 (Database Migrations — from Option C)
  └── EP-13 (Customer Web App — Auth & Shopping)

EP-13 (Customer Web App — Auth & Shopping)
  └── EP-14 (Customer Web App — Profile, Favorites, Subscriptions)

EP-10 (Analytics API)
  └── EP-14 (Customer Web App — real-time map)

EP-15 (IoT Integration) — independent, can run in parallel
```

---

## EP-10: Analytics API — Fix Tenant Context & Implement Handlers

**Priority**: P0-critical (VendorId.New() bug makes all analytics endpoints broken)
**Dependencies**: Option C EP-09 (Redis cache available)
**Requirement refs**: REQ-ANA-01 through REQ-ANA-07

### Stories
- S-10-01: Fix VendorId.New() bug in AnalyticsController (REQ-ANA-01) — P0
- S-10-02: Implement vendor and vase analytics handlers (REQ-ANA-02, REQ-ANA-03, REQ-ANA-04)
- S-10-03: Implement platform and benchmark handlers (REQ-ANA-05, REQ-ANA-06)
- S-10-04: Implement analytics export handler (REQ-ANA-07)

### Tasks
| ID | Story | Title |
|----|-------|-------|
| T-10-001 | S-10-01 | Inject ITenantContextService into AnalyticsController; replace VendorId.New() with Current.VendorId in 4 actions |
| T-10-002 | S-10-02 | Implement GetVendorAnalyticsQueryHandler (revenue, orders, avg order value, top bouquets, daily trend) |
| T-10-003 | S-10-02 | Implement GetVaseAnalyticsQueryHandler (uptime %, scan count, orders attributed, freshness avg per vase) |
| T-10-004 | S-10-02 | Implement GetRealtimeAnalyticsQueryHandler (active vases, available bouquets, open orders — cache-backed) |
| T-10-005 | S-10-03 | Implement GetPlatformInsightsQueryHandler (total vendors, customers, GMV, top vendors, geographic distribution) |
| T-10-006 | S-10-03 | Implement GetBenchmarkAnalyticsQueryHandler (revenue percentile, freshness score vs. peer group) |
| T-10-007 | S-10-04 | Implement ExportAnalyticsQueryHandler (CSV streaming export of orders/revenue for the requested period) |

---

## EP-11: Admin Portal — Customer Management, Platform Analytics & Vendor Actions

**Priority**: P1-high
**Dependencies**: EP-10 (platform analytics endpoint), Option C EP-09
**Requirement refs**: REQ-ADM-01 through REQ-ADM-06

### Stories
- S-11-01: Customer management pages (list + detail + suspend/reactivate) (REQ-ADM-01, REQ-ADM-02)
- S-11-02: Platform analytics dashboard UI (REQ-ADM-03)
- S-11-03: Vendor suspend/reactivate and direct-by-id detail (REQ-ADM-04, REQ-ADM-05)
- S-11-04: Platform order oversight (REQ-ADM-06)

### Tasks
| ID | Story | Title |
|----|-------|-------|
| T-11-001 | S-11-01 | Add GET /api/v1/admin/customers (paginated, searchable) backend endpoint + handler |
| T-11-002 | S-11-01 | Add GET /api/v1/admin/customers/{id}, POST /suspend, POST /reactivate backend endpoints |
| T-11-003 | S-11-01 | Implement Customers/List.cshtml and Customers/Details.cshtml in AdminPortal with ApiClient methods |
| T-11-004 | S-11-02 | Add platform analytics section to Admin Portal dashboard calling GET /api/analytics/platform |
| T-11-005 | S-11-03 | Add POST /api/v1/admin/vendors/{id}/suspend and /reactivate backend endpoints |
| T-11-006 | S-11-03 | Add Suspend/Reactivate buttons to Vendors/Details.cshtml; add ApiClient.SuspendVendorAsync/ReactivateVendorAsync |
| T-11-007 | S-11-03 | Add GET /api/v1/admin/vendors/{id} endpoint; refactor Vendors/Details.cshtml.cs to call GetVendorByIdAsync |
| T-11-008 | S-11-04 | Add GET /api/v1/admin/orders (paginated, filterable) + handler; add Orders/List.cshtml to AdminPortal |

---

## EP-12: Vendor Portal — Order Management, Analytics & Profile

**Priority**: P0-critical (vendors cannot manage orders without this)
**Dependencies**: Option C EP-05 (vendor order API), EP-10 (analytics API fix)
**Requirement refs**: REQ-VND-01 through REQ-VND-07, REQ-IOT-01, REQ-IOT-06

> **PR #7 (merged 2026-06-04):** Added freshness-based dynamic pricing — `FreshnessPricingStrategy` value object, `Bouquet.BasePrice` + `AppliedFreshnessTierIndex`, `SetVendorFreshnessPricingStrategyCommand`, `GET/PUT/DELETE /api/v1/vendor/settings/pricing-strategy`, and `VaseHealthMonitoringService` price-adjustment integration. Backend is complete; portal UI is tracked in S-12-05 below.

### Stories
- S-12-01: Order list and detail pages with status advancement (REQ-VND-01, REQ-VND-02)
- S-12-02: Fix VendorId session bug and mock-vase upload bug (REQ-VND-05, REQ-VND-06, REQ-IOT-01)
- S-12-03: Analytics/revenue dashboard page (REQ-VND-04)
- S-12-04: Opening hours and vendor profile edit (REQ-VND-03, REQ-VND-07)
- S-12-05: Freshness pricing strategy settings UI (PR #7 backend; portal UI pending)

### Tasks
| ID | Story | Title |
|----|-------|-------|
| T-12-001 | S-12-01 | Create Orders/Index.cshtml.cs — list orders with status filter; add ApiClient.GetVendorOrdersAsync |
| T-12-002 | S-12-01 | Create Orders/Details.cshtml.cs — order lines, timeline, status-advance form; add ApiClient.UpdateOrderStatusAsync |
| T-12-003 | S-12-02 | Create ICurrentVendorService extracting VendorId from JWT claim; replace session reads across all page models |
| T-12-004 | S-12-02 | Fix Vases/Details.cshtml.cs: replace mock-vase HttpClient call with _apiClient.UploadBouquetPhotoAsync(vaseId, bytes) |
| T-12-005 | S-12-03 | Create Analytics/Index.cshtml.cs calling GET /api/analytics/vendor; render revenue/orders chart with Chart.js |
| T-12-006 | S-12-04 | Create Settings/OpeningHours.cshtml.cs with weekly schedule editor; add ApiClient.UpdateOpeningHoursAsync |
| T-12-007 | S-12-04 | Create Settings/Profile.cshtml.cs for vendor name/contact/description edit; add ApiClient.UpdateVendorProfileAsync |
| T-12-008 | S-12-05 | Add ApiClient.GetPricingStrategyAsync / SetPricingStrategyAsync / ResetPricingStrategyAsync to VendorPortal ApiClient (calls GET/PUT/DELETE /api/v1/vendor/settings/pricing-strategy) |
| T-12-009 | S-12-05 | Create Settings/PricingStrategy.cshtml.cs: load current strategy (preset selector + custom tier table); POST to PUT endpoint; Reset button calls DELETE endpoint |

---

## EP-13: Customer Web App — Authentication Integration & Shopping

**Priority**: P0-critical (no authenticated features work without this)
**Dependencies**: Option C EP-01, EP-04 (Stripe order creation)
**Requirement refs**: REQ-CWA-01 through REQ-CWA-03, REQ-CWA-08

### Stories
- S-13-01: Wire authentication to protected routes; read customerId from JWT (REQ-CWA-01)
- S-13-02: Cart UI (REQ-CWA-02)
- S-13-03: Checkout and order placement UI (REQ-CWA-03)
- S-13-04: Leaflet.js map view with real-time pins (REQ-CWA-08)

### Tasks
| ID | Story | Title |
|----|-------|-------|
| T-13-001 | S-13-01 | Add ICurrentCustomerService reading customer_id claim from HttpContext.User; register in DI |
| T-13-002 | S-13-01 | Decorate Cart, Checkout, Orders, Favourites, Profile controllers with [Authorize]; redirect anon to /Account/Login |
| T-13-003 | S-13-01 | Remove customerId Guid from ApiClient method signatures; inject ICurrentCustomerService instead |
| T-13-004 | S-13-02 | Create CartController (GET /Cart, POST /Cart/Add, POST /Cart/Remove, POST /Cart/Clear) with cart view |
| T-13-005 | S-13-02 | Add cart item count badge to _Layout.cshtml via a ViewComponent calling GetCartCountAsync |
| T-13-006 | S-13-03 | Create CheckoutController: GET /Checkout (order preview + Stripe PublishableKey), POST /Checkout/PlaceOrder |
| T-13-007 | S-13-03 | Load Stripe.js on checkout page; initialize PaymentElement with clientSecret from CreateOrder response |
| T-13-008 | S-13-03 | Create Checkout/Confirmation view for post-payment success redirect (Stripe return_url) |
| T-13-009 | S-13-04 | Add Leaflet.js map view to Home/Index.cshtml — initialize map, geocode browser location, load nearby pins |
| T-13-010 | S-13-04 | Connect Leaflet map to SignalR BouquetHub: add/remove pins on BouquetBecameAvailable/Unavailable events |

---

## EP-14: Customer Web App — Profile, Favorites & Subscriptions

**Priority**: P1-high
**Dependencies**: EP-13 (authenticated session)
**Requirement refs**: REQ-CWA-04 through REQ-CWA-07

### Stories
- S-14-01: Order history and tracking pages (REQ-CWA-04)
- S-14-02: Favorites UI (REQ-CWA-05)
- S-14-03: Customer profile and saved addresses (REQ-CWA-06)
- S-14-04: Subscriptions management (REQ-CWA-07)

### Tasks
| ID | Story | Title |
|----|-------|-------|
| T-14-001 | S-14-01 | Create OrdersController: GET /Orders (list with status filter), GET /Orders/{id} (detail + timeline) |
| T-14-002 | S-14-01 | Order detail view: show order lines, timeline events, estimated delivery, delivery QR code (if available) |
| T-14-003 | S-14-02 | Add heart/unfavorite toggle button to bouquet cards; add ApiClient.AddFavouriteAsync / RemoveFavouriteAsync |
| T-14-004 | S-14-02 | Create FavouritesController: GET /Favourites listing saved bouquets with availability status badges |
| T-14-005 | S-14-03 | Create ProfileController: GET /Profile (stats + notification prefs), POST /Profile/Update (name/prefs) |
| T-14-006 | S-14-03 | Add saved addresses sub-page: GET /Profile/Addresses, POST /Profile/Addresses/Add, DELETE /Profile/Addresses/{id} |
| T-14-007 | S-14-04 | Create SubscriptionsController: GET /Subscriptions, POST /Subscriptions/Create, DELETE /Subscriptions/{id} |
| T-14-008 | S-14-04 | Add disable-all button to subscriptions page calling DELETE /api/v1/subscriptions/all |

---

## EP-15: IoT Integration — Production Paths & Security

**Priority**: P1-high (P0 for REQ-IOT-01 mock-vase bug)
**Dependencies**: none (can run in parallel)
**Requirement refs**: REQ-IOT-01 through REQ-IOT-05

### Stories
- S-15-01: Real bouquet photo upload endpoint (REQ-IOT-01, REQ-IOT-02)
- S-15-02: MQTT broker authentication (REQ-IOT-03)
- S-15-03: Vase heartbeat offline detection (REQ-IOT-04)
- S-15-04: OTA firmware update endpoint (REQ-IOT-05)

### Tasks
| ID | Story | Title |
|----|-------|-------|
| T-15-001 | S-15-01 | Implement POST /api/v1/bouquets/{id}/photo multipart endpoint in BouquetsController; store file, return photoUrl |
| T-15-002 | S-15-01 | Add ApiClient.UploadBouquetPhotoAsync(bouquetId, bytes, contentType) to VendorPortal ApiClient |
| T-15-003 | S-15-01 | Fix Vases/Details.cshtml.cs to call _apiClient.UploadBouquetPhotoAsync instead of raw HttpClient mock |
| T-15-004 | S-15-02 | Add Mqtt:Username and Mqtt:Password to IConfiguration; pass credentials in MqttDeviceCommunicationService connection |
| T-15-005 | S-15-02 | Update docker-compose.dev.yml Mosquitto config to require password file; add dev credentials to appsettings.Development |
| T-15-006 | S-15-03 | Implement VaseHeartbeatMonitorService (BackgroundService): poll every 5 min, mark stale vases offline, dispatch VaseWentOfflineEvent |
| T-15-007 | S-15-03 | Add ISmartVaseRepository.GetStaleVasesAsync(cutoffMinutes) and implement in SmartVaseRepository |
| T-15-008 | S-15-04 | Add POST /api/v1/admin/vases/{id}/firmware endpoint; publish MQTT message to vase/{serial}/ota with download URL |

---

## EP-16: E2E Testing — Playwright Suite for All Three Portals

**Priority**: P2-medium
**Dependencies**: None — independent of all other epics. Tests for unimplemented features are marked `[Trait("Status", "pending")]` and skipped until the relevant epic is complete.
**Requirement refs**: none (quality infrastructure)

### Stories
- S-16-01: Test project setup and shared infrastructure
- S-16-02: Admin Portal E2E tests — auth, vendor management, vase registration (AP-01 to AP-06)
- S-16-03: Vendor Portal E2E tests — auth, vase list, bouquet price (VP-01 to VP-06)
- S-16-04: Customer App E2E tests — browsing (CA-01 to CA-03)

### Tasks
| ID | Story | Title |
|----|-------|-------|
| T-16-001 | S-16-01 | Create `FlowerShop.Tests.E2E` project with Playwright .NET + xunit, `PlaywrightFixture`, `TestConfig`, `appsettings.E2E.json` |
| T-16-002 | S-16-02 | Admin Portal page objects (LoginPage, VendorListPage, VendorCreatePage, VendorDetailsPage, VaseListPage, VaseRegisterPage) |
| T-16-003 | S-16-02 | Admin Portal auth tests — AP-01 valid login, AP-02 invalid login |
| T-16-004 | S-16-02 | Admin Portal vendor management tests — AP-03 create vendor, AP-04 validation errors |
| T-16-005 | S-16-02 | Admin Portal vase registration tests — AP-05 register vase, AP-06 duplicate serial error |
| T-16-006 | S-16-03 | Vendor Portal page objects (LoginPage, VaseListPage, VaseDetailsPage, BouquetListPage, BouquetSetPricePage) |
| T-16-007 | S-16-03 | Vendor Portal auth tests — VP-01 valid login, VP-02 invalid login, VP-03 session expiry redirect |
| T-16-008 | S-16-03 | Vendor Portal vase & bouquet tests — VP-04 view vases, VP-05 set price, VP-06 invalid price error |
| T-16-009 | S-16-04 | Customer App page objects (HomePage, BouquetDetailsPage) |
| T-16-010 | S-16-04 | Customer App browsing tests — CA-01 home page, CA-02 bouquet details, CA-03 non-existent bouquet 404 |

---

## EP-17: Payment Resilience & Correctness Hardening

**Priority**: P0-critical
**Dependencies**: EP-04 (extends the functional Stripe flow with failure-mode hardening).
**Requirement refs**: REQ-PAY-01 to REQ-PAY-10 (REQ-GAPS-B.md Area 6)
**Source**: review against [systemdesign.one — Design a Payment System](https://newsletter.systemdesign.one/p/design-a-payment-system); full findings in `docs/STRIPE-PAYMENT-RESILIENCE-REVIEW.md` and tasks in `tasks-ep17-payment-resilience.json`.
**Status (2026-06-11): ✅ COMPLETE — all 6 stories / 18 tasks done.** P0 merged to `main` via PR #11 (`91df302` S-17-01, `ec10e3f` S-17-02, `80ed1b2` S-17-03) — all four money-loss paths C1–C4 closed. P1 + P2 on branch `feat/ep17-payment-resilience-p1` (`eb7b84e` S-17-04, `a68aafb` S-17-05, `2086bfa` S-17-06, `20c1878` T-17-012). 288 tests pass (167 unit + 121 domain). Migrations: `AddProcessedStripeEvents`, `AddOrderNeedsRefundRetry`, `AddOrderPaymentProcessing`, `AddOrderDisputed`. Notes: T-17-007 implemented as "never cancel a charged/in-flight order; leave orphaned-charge refunds to the reconciliation worker (H1)" per the review's C1 guidance; T-17-018's `IAlertService` logs at Critical with stable alert names — wiring to a real on-call/metrics backend is a deployment-stack follow-up outside EP-17 code.

> EP-04 made the payment flow work; EP-17 makes it correct under failure. The review found four reachable money-loss paths (C1–C4). **C1 is live today**: a paid customer can be auto-cancelled by the reservation-expiry worker with no refund. C1–C4 are ship-blockers; goal is zero money-loss (no double-charge, no double-refund, no orphaned charge, no silently-dropped webhook).

### Stories
- ✅ S-17-01: Idempotent Stripe API calls — no double-charge / no double-refund (C3, M1) — **DONE**
- ✅ S-17-02: Reliable webhook ingestion — store-first, dedup, retry-on-failure (C2, H2, H3) — **DONE**
- ✅ S-17-03: No orphaned charges — expiry must refund, plus reconciliation safety net (C1, C4, H1) — **DONE**
- ✅ S-17-04: Atomic & race-safe payment confirmation (H5, H6) — **DONE**
- ✅ S-17-05: Asynchronous payment methods (BLIK) handled correctly (H4) — **DONE**
- ✅ S-17-06: Defensive hardening — circuit breaker, amount checks, dispute events, alerting (M2–M6) — **DONE**

### Tasks
| ID | Story | Title | Status |
|----|-------|-------|--------|
| T-17-001 | S-17-01 | Attach a deterministic IdempotencyKey to `PaymentIntent.Create`; wire through the dead `CreateOrderCommand.IdempotencyKey` | ✅ done |
| T-17-002 | S-17-01 | Attach a deterministic IdempotencyKey to `Refund.Create` | ✅ done |
| T-17-003 | S-17-01 | Fix truncating money-to-minor-unit conversion (`Math.Round`) | ✅ done |
| T-17-004 | S-17-02 | Add `processed_stripe_events` table for webhook dedup + audit | ✅ done |
| T-17-005 | S-17-02 | Rework `StripeWebhookController` to store-first, then `200`; stop returning `200` on failure | ✅ done |
| T-17-006 | S-17-02 | Process stored webhook events asynchronously via a worker (mirror of the outbox) | ✅ done |
| T-17-007 | S-17-03 | Verify-with-Stripe before expiring a `Created` order; refund if charged (fixes live C1) | ✅ done |
| T-17-008 | S-17-03 | Check `refund.Status`; only mark refunded on success, else flag + alert | ✅ done |
| T-17-009 | S-17-03 | Add daily payment reconciliation worker | ✅ done |
| T-17-010 | S-17-04 | Add optimistic-concurrency token to `Order` | ✅ done (Order.Version, no migration) |
| T-17-011 | S-17-04 | Make payment confirmation atomic in a single transaction | ✅ done (ExecuteInTransactionAsync) |
| T-17-012 | S-17-04 | Move inline MQTT/vase side effects to the `OrderPaidEvent` consumer | ✅ done (OrderPaidVaseNotificationHandler) |
| T-17-013 | S-17-05 | Handle `payment_intent.processing`; webhook as source of truth for BLIK | ✅ done |
| T-17-014 | S-17-06 | Assert amount + currency match the order before confirming | ✅ done |
| T-17-015 | S-17-06 | Wrap Stripe calls in Polly timeout + bounded retry + circuit breaker | ✅ done |
| T-17-016 | S-17-06 | Pin the Stripe API version | ✅ done (SDK-pinned + AppInfo + surfaced) |
| T-17-017 | S-17-06 | Handle `charge.refunded` / `charge.dispute.created` / `payment_intent.canceled` | ✅ done |
| T-17-018 | S-17-06 | Route payment failures to alerting, not silent logs | ✅ done (IAlertService; real on-call backend = deploy follow-up) |

---

## Summary

| Epic | Priority | Stories | Tasks | Key Dependency |
|------|----------|---------|-------|----------------|
| EP-10 Analytics API | P0-critical | 4 | 7 | EP-09 |
| EP-11 Admin Portal | P1-high | 4 | 8 | EP-10 |
| EP-12 Vendor Portal | P0-critical | 4 | 7 | EP-05, EP-10 |
| EP-13 Customer Web Shopping | P0-critical | 4 | 10 | EP-01, EP-04 |
| EP-14 Customer Web Profile | P1-high | 4 | 8 | EP-13 |
| EP-15 IoT Integration | P1-high | 4 | 8 | none |
| EP-16 E2E Testing | P2-medium | 4 | 10 | none |
| EP-17 Payment Resilience & Correctness | P0-critical | 6 | 18 | EP-04 |
| **Total** | | **34** | **76** | |
