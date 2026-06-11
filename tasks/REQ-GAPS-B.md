# Requirements Gap Analysis — Option B
## Admin Portal, Vendor Portal, Customer Web App, Analytics API, IoT Integration

> Companion to REQ-GAPS.md (Option C — Mobile API).
> Scope: Razor Pages portals, ASP.NET MVC customer web app, analytics backend, IoT/mock-vase cleanup.
> Priority legend: P0=critical blocker, P1=high value, P2=nice to have.

---

## Area 1: Analytics API — Tenant Context & Handler Stubs

### REQ-ANA-01 [BUG] Fix VendorId.New() in AnalyticsController — P0
`AnalyticsController` calls `VendorId.New()` in four actions (`GetVendorAnalytics`, `GetVaseAnalytics`, `GetRealtimeAnalytics`, `ExportAnalytics`) instead of reading the authenticated vendor from `ITenantContextService`. Every call returns data for a random vendor ID. Must inject `ITenantContextService` and use `Current.VendorId`.

### REQ-ANA-02 [MISSING] Implement GetVendorAnalyticsQueryHandler — P1
`GetVendorAnalyticsQuery` exists but no handler class is registered. The `VendorAnalyticsDto` returned by `GET /api/analytics/vendor` is never populated. Handler must aggregate: total revenue, order count, average order value, top bouquets, daily trend series — all scoped to the authenticated vendor and the requested period.

### REQ-ANA-03 [MISSING] Implement GetVaseAnalyticsQueryHandler — P1
`GetVaseAnalyticsQuery` exists but no handler. Must return per-vase metrics: uptime %, scan count, orders attributed, freshness average — for the authenticated vendor, optionally filtered by a single VaseId.

### REQ-ANA-04 [MISSING] Implement GetRealtimeAnalyticsQueryHandler — P1
`GetRealtimeAnalyticsQuery` exists but no handler. Real-time dashboard data: active vases count, bouquets currently available, open orders, live freshness scores. Should read from cache (Redis) when available to avoid DB hammering.

### REQ-ANA-05 [MISSING] Implement GetPlatformInsightsQueryHandler — P1
`GetPlatformInsightsQuery` exists but no handler. PlatformAdmin-only endpoint returning: total vendors, total customers, GMV (last 30/90 days), top-performing vendors, geographic heatmap data.

### REQ-ANA-06 [MISSING] Implement GetBenchmarkAnalyticsQueryHandler — P2
`GetBenchmarkAnalyticsQuery` passes `null` as VendorId and no handler exists. Anonymous peer-comparison benchmarks: revenue percentile, average freshness score vs. peers of same VendorType and city.

### REQ-ANA-07 [MISSING] Implement ExportAnalyticsQueryHandler — P2
`ExportAnalyticsQuery` exists but no handler. Must generate CSV (or XLSX) from the vendor's order/revenue data for the requested period. Requires a streaming file response; do not buffer entire result in memory.

---

## Area 2: Admin Portal — Customer Management

### REQ-ADM-01 [MISSING] Customer list page — P1
No `/Customers` section exists in AdminPortal. Platform admins need to list customers (paginated), search by email, and view registration date, order count, and status. Requires `GET /api/v1/admin/customers` endpoint (add to backend) and a corresponding `ApiClient` method.

### REQ-ADM-02 [MISSING] Customer detail and suspend/reactivate — P1
No page to view a customer's profile or soft-suspend their account. Requires a `GET /api/v1/admin/customers/{id}` and `POST /api/v1/admin/customers/{id}/suspend` / `reactivate` backend endpoints, plus AdminPortal pages for the detail view and action buttons.

### REQ-ADM-03 [MISSING] Platform analytics dashboard — P1
`AdminPortal/Pages/Index.cshtml.cs` shows only three counters (TotalVendors, TotalVases, PendingInstallations). A proper platform analytics section is missing: GMV summary, active customers, order volume chart, top vendors, geographic coverage. The `GET /api/analytics/platform` endpoint exists (REQ-ANA-05) but AdminPortal has no UI for it.

### REQ-ADM-04 [MISSING] Vendor suspend / reactivate actions — P1
Vendor Details page only allows updating location. There are no suspend or reactivate buttons for admin to change a vendor's status. Requires `POST /api/v1/admin/vendors/{id}/suspend` and `POST /api/v1/admin/vendors/{id}/reactivate` backend endpoints and corresponding AdminPortal page handler methods.

### REQ-ADM-05 [PARTIAL] Vendor Details loads from full list — P2
`Vendors/Details.cshtml.cs` loads the entire vendor list and calls `FirstOrDefault`. This is wasteful and fragile. A dedicated `GET /api/v1/admin/vendors/{id}` endpoint and `ApiClient.GetVendorByIdAsync(id)` should be used instead.

### REQ-ADM-06 [MISSING] Order oversight — platform-level order list — P2
No page exists for platform admins to view all orders across vendors (for dispute resolution, fraud detection). Requires `GET /api/v1/admin/orders` (paginated, filterable by vendor/status/date) backend endpoint and AdminPortal list page.

---

## Area 3: Vendor Portal — Order Management

### REQ-VND-01 [MISSING] Order list page for vendors — P0
VendorPortal has no `/Orders` section. Vendors cannot see incoming orders. The `GET /api/v1/vendor/orders` endpoint is planned in EP-05 (T-05-010), but the portal UI is entirely absent. Requires: Orders/Index.cshtml listing orders with status filter, pagination, and status badges.

### REQ-VND-02 [MISSING] Order detail page and status advancement — P0
Vendors need to open an order, see its lines and timeline, and advance the status (Preparing → InDelivery/ReadyForPickup → Delivered). Requires: Orders/Details.cshtml with a status-advance button calling `PUT /api/v1/vendor/orders/{id}/status` and mapping the allowed transitions.

### REQ-VND-03 [MISSING] Opening hours management UI — P1
No page exists for vendors to set their shop opening hours. The domain supports `OpeningHours` (EP-01, REQ-DB-03) but VendorPortal has no UI. Requires a Settings or Profile page with a weekly schedule editor calling `PUT /api/v1/vendor/profile/hours`.

### REQ-VND-04 [MISSING] Analytics/revenue dashboard in Vendor Portal — P1
VendorPortal `Index.cshtml.cs` shows basic stats (vases, bouquets) but no revenue, order volume, or trend data. A separate Analytics page should call `GET /api/analytics/vendor` and render the data (once REQ-ANA-01/02 are fixed).

### REQ-VND-05 [BUG] VendorId read from session / hardcoded — P0
`VendorPortal/Pages/Bouquets/Index.cshtml.cs` reads VendorId from `HttpContext.Session.GetString("VendorId")` (stored on login). All vendor-scoped API calls should extract VendorId from the authenticated JWT claim (`vendor_id`) via `ITenantContextService` or an `ICurrentVendorService` helper, not from session. This is both a security gap and a reliability issue.

### REQ-VND-06 [BUG] Bouquet image upload calls mock-vase endpoint — P0
`VendorPortal/Pages/Vases/Details.cshtml.cs` uploads bouquet images to `http://localhost:8080/api/mock-vase/{id}/bouquet-image` — a hardcoded development mock. Production upload must go through the real `POST /api/v1/bouquets/{id}/photo` endpoint using the authenticated `ApiClient`.

### REQ-VND-07 [MISSING] Vendor profile edit page — P1
VendorPortal has no page where vendors can update their own display name, contact info, or description. Requires a Settings/Profile page calling `PUT /api/v1/vendor/profile`.

---

## Area 4: Customer Web App — Authentication & Protected Features

### REQ-CWA-01 [PARTIAL] OIDC login wired but no protected routes — P0
`AccountController` implements OIDC login/logout but no controller action is decorated with `[Authorize]`. Cart, checkout, order tracking, and favorites all require authentication. The `customerId` Guid is passed as a raw URL parameter in `ApiClient` (`/api/v1/cart/{customerId}/items`) — it must be read from the authenticated user's `customer_id` JWT claim instead.

### REQ-CWA-02 [MISSING] Cart UI — P0
`CustomerApp.ApiClient` has `AddToCartAsync`, `RemoveFromCartAsync`, `GetCartAsync`, `ClearCartAsync`, but no `CartController` or cart view exists. Requires: GET /Cart (cart view), POST /Cart/Add, DELETE /Cart/Remove, and a cart item count in the nav header.

### REQ-CWA-03 [MISSING] Checkout / order placement UI — P0
No checkout flow exists. Requires: GET /Checkout (summary with Stripe PaymentSheet initialization), POST /Checkout/PlaceOrder calling `POST /api/v1/orders`, confirmation page after payment success. The Stripe JS SDK must be loaded on the checkout page.

### REQ-CWA-04 [MISSING] Order history and tracking UI — P1
No order list or detail page for customers. Requires: GET /Orders listing past orders with status, GET /Orders/{id} showing order detail with timeline and estimated delivery.

### REQ-CWA-05 [MISSING] Favorites UI — P1
No favorites section. Requires: a heart button on bouquet cards calling the Favorites API, GET /Favourites page listing saved bouquets with their current availability status.

### REQ-CWA-06 [MISSING] Profile page — P1
No customer profile UI. Requires: GET /Profile showing name, email, notification settings, and saved addresses; PUT /Profile for editing name/preferences; GET/POST /Profile/Addresses for saved address CRUD.

### REQ-CWA-07 [MISSING] Subscription management UI — P2
No subscriptions section. Requires: GET /Subscriptions listing active subscriptions, POST /Subscriptions/Create, DELETE /Subscriptions/{id}, and a "Disable all" button.

### REQ-CWA-08 [MISSING] Map view with Leaflet.js and real-time pins — P1
`HomeController` renders a static bouquet list. A Leaflet.js map with bouquet pins filtered by user's current location (browser geolocation API) is missing. Map pins should reflect real-time availability via SignalR `BouquetBecameAvailable` / `BouquetBecameUnavailable` events.

---

## Area 5: IoT Integration — Remove Mocks, Add Production Paths

### REQ-IOT-01 [BUG] Remove mock-vase endpoint dependency — P0
`VendorPortal/Pages/Vases/Details.cshtml.cs` uploads to `http://localhost:8080/api/mock-vase/{id}/bouquet-image` using a new `HttpClient()` instance (no auth, no base URL config). This must use the shared, authenticated `_apiClient` and the real bouquet photo endpoint.

### REQ-IOT-02 [MISSING] Add bouquet photo upload API endpoint — P1
The real backend endpoint `POST /api/v1/bouquets/{id}/photo` (multipart/form-data) is referenced in the portal but may not be implemented. Verify `BouquetsController` has the photo upload action and that it stores the file (local storage or blob), returning the `photoUrl`.

### REQ-IOT-03 [MISSING] MQTT broker authentication — P1
Mosquitto broker in `docker-compose.dev.yml` accepts anonymous connections. Production deployment needs username/password (or certificate-based) authentication. `MqttDeviceCommunicationService` must pass credentials from `IConfiguration` (`Mqtt:Username`, `Mqtt:Password`).

### REQ-IOT-04 [MISSING] Vase heartbeat / offline detection background service — P1
When a vase stops sending MQTT heartbeats, no automatic offline detection exists. A `BackgroundService` polling `ISmartVaseRepository.GetStaleVasesAsync(cutoffMinutes: 10)` should mark stale vases as offline and dispatch `VaseWentOfflineEvent`.

### REQ-IOT-05 [MISSING] OTA firmware update endpoint — P2
No endpoint exists to push a firmware update to a smart vase. Requires `POST /api/v1/admin/vases/{id}/firmware` (upload binary) and MQTT publish to `vase/{serialNumber}/ota` with the download URL.

---

## Area 6: Payment Resilience & Correctness Hardening

> Extends EP-04 (which made the Stripe flow *functional*) with the failure-mode hardening it lacks.
> Source: review against [systemdesign.one — Design a Payment System](https://newsletter.systemdesign.one/p/design-a-payment-system), full findings in `docs/STRIPE-PAYMENT-RESILIENCE-REVIEW.md` (findings C1–C4 critical, H1–H6 high, M1–M6 medium).
>
> **STATUS (2026-06-11): EP-17 COMPLETE — all 10 REQ-PAY done.** C1–C4 closed (P0, merged via PR #11); H/M items done (P1+P2, PR #12). Each requirement below keeps its original gap description for context and is tagged `[DONE]` with the implementing artifact. The "C1 is live today" warning was the original finding — it is now fixed (REQ-PAY-04).

### REQ-PAY-01 [DONE — S-17-01] Idempotent Stripe API calls — P0 (findings C3, M1)
> Done: deterministic `IdempotencyKey` on `PaymentIntent.Create` (`pi_create:{orderId}`, or the client key) and `Refund.Create` (`refund:{orderId}`); `StripeService.ToMinorUnits` uses `Math.Round(AwayFromZero)`.
`StripeService` attaches no `IdempotencyKey` to `PaymentIntent.Create` or `Refund.Create`, so a retried `CreateOrderCommand` creates a second PaymentIntent and a retried `CancelOrderCommand` issues a second refund. The `CreateOrderCommand.IdempotencyKey` field already exists but is never read in the handler. Pass deterministic keys (`pi_create:{orderId}`, `refund:{orderId}`) on every Stripe POST. Also fix the truncating `(long)(amount * 100)` minor-unit conversion (use `Math.Round`).

### REQ-PAY-02 [DONE — S-17-02] Reliable webhook ingestion — P0 (findings C2, H2)
> Done: `StripeWebhookController` now verifies → stores → `200` (or `500` on store failure); `StripeWebhookProcessorService` applies transitions async; catch-all-200 removed.
`StripeWebhookController` catches all processing errors and returns `200`, so Stripe never retries — one failed webhook permanently loses a state transition. It also runs the full `ConfirmPaymentCommand` (DB writes per bouquet, vase updates, MQTT) inline under Stripe's ~20s timeout. Rework to: verify signature → persist raw event → return `200`; process asynchronously via a worker; return `500` on transient processing failure so Stripe re-delivers.

### REQ-PAY-03 [DONE — S-17-02] Persisted webhook-event dedup & audit log — P0 (finding H3)
> Done: `ProcessedStripeEvents` table (EventId PK), `IStripeEventStore`/`EfStripeEventStore` insert-if-absent + mark processed/failed. Migration `AddProcessedStripeEvents`.
Idempotency rests entirely on the `Status == Created` guard, which breaks for events that arrive when the order is not `Created` (refunds, disputes, async BLIK transitions) and leaves no audit trail. Add a `processed_stripe_events(event_id PK, type, received_at, payload, processed_at, error)` table; insert-or-skip on event id gives true idempotency independent of aggregate state.

### REQ-PAY-04 [DONE — S-17-03] No orphaned charges — expiry must refund; check refund status — P0 (findings C1, C4)
> Done: `OrderReservationExpiryService` verifies the PaymentIntent at Stripe before cancelling and never cancels a charged/in-flight order (orphaned-charge refunds handled by the reconciliation worker per the review's C1 guidance). `CancelOrderCommandHandler` branches on `RefundResult.Status` → `MarkRefundCompleted`/`MarkRefundFailed` (+ `NeedsRefundRetry` flag + alert).
`OrderReservationExpiryService` cancels `Created` orders via `Order.CancelDueToReservationExpiry()`, which never refunds and bypasses `CancelOrderCommandHandler`'s refund logic — so a charged-but-unconfirmed order is cancelled with no refund. Never auto-cancel a `Created` order without first verifying the PaymentIntent at Stripe and refunding any successful charge. Separately, `CancelOrderCommandHandler` logs success regardless of `refund.Status` — branch on it and flag failed/pending refunds for retry + alert.

### REQ-PAY-05 [DONE — S-17-03] Daily payment reconciliation worker — P1 (finding H1)
> Done: `PaymentReconciliationService` (24h) converges paid-ambiguous orders via `IOrderRepository.GetOrdersForReconciliationAsync` — confirms charged-but-Created orders, retries failed refunds, alerts per divergence.
No reconciliation exists, so a single dropped webhook is invisible forever. Add a scheduled job that queries Stripe for orders in paid-ambiguous states (`Created` with a PaymentIntentId, recently cancelled, refund-pending/failed) and converges local state — confirming paid orders, refunding orphaned charges, retrying failed refunds — emitting an alert per divergence.

### REQ-PAY-06 [DONE — S-17-04] Atomic & race-safe payment confirmation — P1 (findings H5, H6)
> Done: `Order.Version` mapped `.IsConcurrencyToken()`; confirm wrapped in `IUnitOfWork.ExecuteInTransactionAsync`; MQTT moved to `OrderPaidVaseNotificationHandler` (T-17-012).
The client `confirm-payment` and webhook paths can run concurrently; both read `Status == Created` (TOCTOU) and `Order` has no mapped EF concurrency token, so both can proceed. `ConfirmPaymentCommandHandler` also issues many independent `SaveChanges` (per-bouquet, outbox, order), so a mid-loop crash leaves bouquets `Sold` while the order stays `Created`. Map an optimistic-concurrency token on `Order`; wrap the paid transition + bouquet updates + outbox write in one transaction; move MQTT/vase side effects to the `OrderPaidEvent` consumer.

### REQ-PAY-07 [DONE — S-17-05] Correct handling of asynchronous payment methods (BLIK) — P1 (finding H4)
> Done: `Order.PaymentProcessing` flag + `MarkPaymentProcessing()`; webhook handles `payment_intent.processing`; in-flight orders exempt from expiry; confirm only on `succeeded`. Migration `AddOrderPaymentProcessing`.
BLIK is enabled for PLN but is asynchronous: `payment_intent.processing` arrives before `succeeded`/`failed`. The client confirm call can return before funds are confirmed, only `succeeded`/`failed` are handled, and the 15-min expiry can cancel a slow-but-valid payment. Handle `payment_intent.processing`, treat the webhook as source of truth for BLIK, and exempt in-flight intents from reservation expiry.

### REQ-PAY-08 [DONE — S-17-06] Defensive amount check & money-event coverage — P2 (findings M2, M5)
> Done: webhook asserts amount+currency before confirming (alert on mismatch); handlers for `charge.refunded`, `charge.dispute.created` (`Order.Disputed`, migration `AddOrderDisputed`), `payment_intent.canceled`.
The webhook trusts `payment_intent.succeeded` without asserting `Amount`/`Currency` match the order, and ignores `charge.refunded`, `charge.dispute.created`, and `payment_intent.canceled`. Add an amount/currency equality check before confirming, and handlers for the missing money events (refund completion, chargebacks, cancellation).

### REQ-PAY-09 [DONE — S-17-06] Stripe resilience policy & pinned API version — P2 (findings M3, M4)
> Done: Polly pipeline (15s timeout + 3× backoff retry + circuit breaker) around all `StripeService` calls; API version pinned by the SDK + `AppInfo` set + `PinnedApiVersion` surfaced/logged.
No timeout/retry/circuit breaker around Stripe calls (the design recommends a breaker to protect *our* system), and the Stripe API version is not pinned. Add a Polly pipeline (timeout + bounded retry + circuit breaker — enabled only after REQ-PAY-01 idempotency keys land) and pin `StripeConfiguration.ApiVersion`.

### REQ-PAY-10 [DONE (seam) — S-17-06] Alerting on payment anomalies — P2 (finding M6)
Every "manual refund required", failed refund, webhook processing failure, amount mismatch, and reconciliation divergence is currently a silent log line. Route all of these to structured alerts (metric + on-call notification) per `docs/deployment/OBSERVABILITY.md`.
> Done: `IAlertService`/`LoggingAlertService` raises structured Critical alerts with stable names (`PAYMENT_REFUND_FAILED`, `PAYMENT_AMOUNT_MISMATCH`, `PAYMENT_DISPUTE_CREATED`, `PAYMENT_WEBHOOK_POISON`, `PAYMENT_EXPIRY_VERIFY_FAILED`, `PAYMENT_ORPHANED_CHARGE_REFUND_FAILED`, `PAYMENT_RECONCILIATION_DRIFT`, `PAYMENT_REFUND_RETRY_FAILED`) at every failure point.
> **FOLLOW-UP (not EP-17):** wire `IAlertService` to a real on-call/metrics backend (PagerDuty / OpenTelemetry) per `docs/deployment/OBSERVABILITY.md`. Today it only logs at Critical.
