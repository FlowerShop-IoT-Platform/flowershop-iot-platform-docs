# FlowerShop Backend — Gap Requirements

**Scope:** Mobile API (FMF app) + Vendor Order Management
**Source:** Gap analysis against `docs/API-CONTRACTS.md` v1.2 and codebase scan (2026-03-21)
**Status:** Draft

---

## Legend

- `[MISSING]` — feature/code does not exist at all
- `[STUB]` — interface/class exists but implementation is empty or placeholder
- `[PARTIAL]` — implementation exists but does not meet API contract

---

## 1. Database — Pending Migration

### REQ-DB-01 `[MISSING]`
Generate and apply EF Core migration `AddCartAndOrderLines` that creates:
- `Carts` table
- `CartItems` table
- `Orders` table (multi-line schema with `LinesSubtotal`; drops `BouquetId`/`BouquetPrice` from old single-item design)
- `OrderLines` table
- `Subscriptions` table
- `Favourites` table
- `Notifications` table
- `DeviceRegistrations` table
- `SavedAddresses` table

All EF configurations exist (`CartConfiguration`, `OrderConfiguration`, `SubscriptionConfiguration`, etc.) but no migration has been generated.

### REQ-DB-02 `[MISSING]`
Add `QrCode` (8-char alphanumeric short code) column to `Bouquets` table. Must have a unique index. Add a migration for this.

### REQ-DB-03 `[MISSING]`
Add `OpeningHours` (varchar) and `Description` (text) columns to `FlowerShops` and `FreelanceFlorists` tables. Add migration.

---

## 2. Customer Authentication & Registration

### REQ-AUTH-01 `[PARTIAL]`
`POST /api/auth/register` must create the customer account in Keycloak **and** create a `Customer` entity in the local DB with the `customer_id` claim populated in the Keycloak JWT. Verify `CustomerRegistrationController` covers both steps atomically.

### REQ-AUTH-02 `[MISSING]`
`POST /api/auth/resend-verification` — delegates to Keycloak admin API to trigger a resend of the email verification link. Rate-limited (1 per 60 s per user). Returns 202 always.

### REQ-AUTH-03 `[MISSING]`
`POST /api/auth/password-reset/request` — delegates to Keycloak admin API to initiate password reset. Returns 202 always (does not reveal whether email exists). No auth required.

### REQ-AUTH-04 `[MISSING]`
`POST /api/auth/password-reset/confirm` — delegates to Keycloak to complete the reset using the token from the email link. Returns 200 on success, 400 if token is invalid or expired.

---

## 3. App Configuration

### REQ-CFG-01 `[STUB]`
`GET /api/config` — `ConfigController` exists but must return a fully populated `AppConfigDto` including `stripePublishableKey` (from app settings / env var), `mapTileProvider`, `supportEmail`, `termsUrl`, `privacyPolicyUrl`, `minAppVersion`. Response must include `Cache-Control: public, max-age=3600`.

---

## 4. Bouquet Discovery

### REQ-DISC-01 `[PARTIAL]`
`GET /api/bouquets/nearby` — `GetNearbyBouquetsQueryHandler` must perform an actual geo-spatial distance filter using the Haversine formula (or PostGIS `ST_DWithin`). Currently the handler may return all available bouquets without distance filtering.

### REQ-DISC-02 `[MISSING]`
`GET /api/bouquets/nearby` — the response items must include `isFavourite: true/false` for authenticated requests. This requires joining with the `Favourites` table for the requesting customer.

### REQ-DISC-03 `[MISSING]`
`GET /api/bouquets/nearby` — freelance florist bouquet locations must have their exact coordinates fuzzed to within ~500 m (`exactAddressHidden: true`) before being returned.

### REQ-DISC-04 `[PARTIAL]`
`GET /api/bouquets/{id}` — must include `isVendorCurrentlyOpen` (boolean | null) computed server-side from the vendor's `OpeningHours` string and the current server time (Warsaw / CET timezone). Returns `null` if `OpeningHours` is not configured.

### REQ-DISC-05 `[PARTIAL]`
`GET /api/bouquets/{id}` — must include `isFavourite` for authenticated requests, `aiTags`, `vendorOpeningHours`, and `distanceKm` (only when `lat`/`lng` query params are provided).

### REQ-DISC-06 `[MISSING]`
`GET /api/bouquets/qr/{code}` — `ResolveBouquetByQrQueryHandler` must look up bouquets by the 8-character alphanumeric `QrCode` field (REQ-DB-02). Currently there is no `QrCode` field on the `Bouquet` aggregate. The handler must return 410 Gone when the bouquet is `sold` or `expired`.

### REQ-DISC-07 `[MISSING]`
When a `Bouquet` is created, generate and store a unique 8-character alphanumeric QR short code (`QrCode` field). The `CreateBouquetCommandHandler` must populate this field.

---

## 5. Vendor Profile & Opening Hours

### REQ-VND-01 `[MISSING]`
Add `OpeningHours` (string, e.g. `"Mon-Sat 09:00-20:00, Sun 10:00-16:00"`) and `Description` fields to the `FlowerShop` and `FreelanceFlorist` domain aggregates and their EF configurations.

### REQ-VND-02 `[MISSING]`
Implement `IsCurrentlyOpen(DateTimeOffset now)` helper that parses the `OpeningHours` string and returns `bool?` (`null` if not configured). Used by `GET /api/bouquets/{id}` (REQ-DISC-04) and `GET /api/vendors/{id}`.

### REQ-VND-03 `[PARTIAL]`
`GET /api/vendors/{id}` (`PublicVendorsController`) — response must include `isCurrentlyOpen`, `openingHours`, `description`, `activeBouquetCount`, and `rating` (rating can default to `null` for MVP). For freelance florists, `exactAddressHidden: true` and coordinates must be fuzzed.

---

## 6. Order Lifecycle

### REQ-ORD-01 `[DONE — EP-04]`
Integrate Stripe. Implement `IStripeService` with:
- `CreatePaymentIntentAsync(decimal amount, string currency, string customerId)` → returns `(clientSecret, ephemeralKey, customerId)`
- `RefundPaymentAsync(string paymentIntentId)` → initiates a refund

Register `StripeService` in DI. Stripe secret key comes from `appsettings` / env var.

### REQ-ORD-02 `[DONE — EP-04]`
`CreateOrderCommandHandler` must call `IStripeService.CreatePaymentIntentAsync` after reserving bouquets and return `stripeClientSecret`, `stripeEphemeralKey`, and `stripeCustomerId` in `OrderCreatedDto`. Implemented; `PaymentClientSecret`/`StripeEphemeralKey`/`StripeCustomerId` are returned in `OrderCreatedDto`.

### REQ-ORD-03 `[DONE — EP-04]`
Implement `StripeWebhookController` at `POST /api/stripe/webhook`. Handle `payment_intent.succeeded` and `payment_intent.payment_failed` events. On `succeeded`: confirm order payment (delegate to `ConfirmPaymentCommand`). On `failed`: cancel the order, release bouquet reservations. Verify Stripe webhook signature using `Stripe-Signature` header. Implemented in commit `0aa528e`. **Resilience gaps (returns 200 on failure, no dedup table, inline processing) are tracked in EP-17 REQ-PAY-02/03.**

### REQ-ORD-04 `[DONE — EP-04]`
Implement idempotency for `POST /api/orders`. The `Idempotency-Key` header (UUID v4) must be stored in Redis with the serialized response. If the same key is received within 24 h, return the cached response without re-executing the handler. Implemented via `IdempotencyMiddleware`. **Note:** this is response-caching only; idempotency keys on the Stripe API calls themselves are EP-17 REQ-PAY-01.

### REQ-ORD-05 `[DONE — EP-04]`
Implement `OrderReservationExpiryService` — a `BackgroundService` that runs every 30 s, finds orders with `Status = Created` and `BouquetReservedUntil < now`, cancels them, and releases bouquet reservations. Must dispatch a `BouquetBecameAvailable` SignalR event for each released bouquet. Implemented. **Critical gap (C1): it can cancel a charged order with no refund — fixed in EP-17 REQ-PAY-04.**

### REQ-ORD-06 `[DONE — EP-04]`
`GET /api/orders/{id}` — response must include `deliveryQrCode` (generated on demand when status is `Paid` or later). Implemented: a geo `geo:` QR is built on order creation for delivery orders and lazily generated in `GetOrderDetailQueryHandler`.

### REQ-ORD-07 `[PARTIAL — EP-04]`
`POST /api/orders/{id}/cancel` — when the order status is `Paid`, call `IStripeService.RefundPaymentAsync` and set `refundStatus` in the response. The refund **call** is wired in `CancelOrderCommandHandler`, but the `RefundStatus` property, `CancelOrderResponseDto`, and `refundStatus` in the response body were never implemented, and the refund result status is not checked. Residual tracked in EP-17 T-17-008.

---

## 7. Vendor Order Management

### REQ-VOM-01 `[MISSING]`
Add `PUT /api/vendor/orders/{id}/status` endpoint (VendorAccess policy). Allowed transitions:
- `Paid` → `Preparing`
- `Preparing` → `InDelivery` (delivery orders only)
- `Preparing` → `ReadyForPickup` (pickup orders only)
- `InDelivery` or `ReadyForPickup` → `Delivered`

Only the vendor who owns the order's bouquets may update it.

### REQ-VOM-02 `[MISSING]`
Implement `UpdateOrderStatusCommand` and `UpdateOrderStatusCommandHandler`. On each status change:
- Append a `TimelineEntry` to `Order.Timeline`.
- Set `EstimatedDeliveryAt` when transitioning to `InDelivery` (vendor-supplied or auto-computed).
- Dispatch `OrderStatusChanged` SignalR event to the order's group.
- Dispatch `OrderStatusChanged` push notification to the customer.

### REQ-VOM-03 `[MISSING]`
Add `GET /api/vendor/orders` (VendorAccess) — returns the vendor's orders paginated, filterable by `status`. Used by the vendor portal to manage fulfillment queue.

### REQ-VOM-04 `[MISSING]`
When order status transitions to `Delivered`, mark all bouquets in the order as `Sold` in the `Bouquets` table and dispatch `BouquetBecameUnavailable` (reason: `sold`) SignalR event.

---

## 8. Favourites

### REQ-FAV-01 `[PARTIAL]`
`GET /api/favourites` — handler must join with bouquet data (thumbnail, status, freshness, price, location, distanceKm). Bouquets that are no longer available must still be returned with their last known status.

### REQ-FAV-02 `[PARTIAL]`
`POST /api/favourites/{bouquetId}` — returns 409 if already favourited.

### REQ-FAV-03 `[PARTIAL]`
`DELETE /api/favourites/{bouquetId}` — returns 204 always (idempotent).

---

## 9. Subscriptions

### REQ-SUB-01 `[PARTIAL]`
`POST /api/subscriptions` — `SubscriptionHandlers` must validate:
- `flowerType` value exists in the flower types reference list.
- `radiusKm` is one of `5`, `10`, `15`.
- `notificationFrequency` is one of `immediate`, `daily`, `weekly`.
- `shopId` exists (for type `shop`).

### REQ-SUB-02 `[PARTIAL]`
`PUT /api/subscriptions/disable-all` — set `IsActive = false` on all of the customer's subscriptions and return `{ disabledCount, message }`.

---

## 10. Push Notifications & Device Registration

### REQ-NOTIF-01 `[STUB]`
Implement `INotificationServiceClient` / `FirebaseNotificationService` that wraps the Firebase Admin SDK (FCM). Inject the Firebase service account key from app settings / env var. Implement `SendAsync(deviceToken, title, body, data)`.

### REQ-NOTIF-02 `[STUB]`
`POST /api/notifications/device` — `DeviceRegistrationService` must upsert the device token in `DeviceRegistrations` (update `AppVersion` and `LastSeenAt` if token already exists).

### REQ-NOTIF-03 `[STUB]`
`DELETE /api/notifications/device` — remove the matching `DeviceRegistration` record.

### REQ-NOTIF-04 `[PARTIAL]`
`GET /api/notifications` — `NotificationHandlers.GetNotificationsQueryHandler` must return paginated results including `unreadCount` in the envelope.

### REQ-NOTIF-05 `[PARTIAL]`
`PUT /api/notifications/{id}/read` and `PUT /api/notifications/read-all` — mark one or all as `IsRead = true`.

---

## 11. Customer Profile & Saved Addresses

### REQ-PRF-01 `[PARTIAL]`
`GET /api/profile` — response must include `stats` (totalOrders, activeSubscriptions, totalSpent, currency), `preferredFlowerTypes`, and `notificationSettings`. These require queries against Orders, Subscriptions, and a NotificationSettings sub-record on Customer.

### REQ-PRF-02 `[PARTIAL]`
`PUT /api/profile` — must propagate `firstName`/`lastName` changes to Keycloak via `IKeycloakAdminClient`.

### REQ-PRF-03 `[PARTIAL]`
`DELETE /api/profile` — request must include `confirmationText: "DELETE"`. Marks the customer as `PendingDeletion` with a `ScheduledDeletionDate` 30 days out. Returns 202 with the scheduled date. GDPR compliance.

### REQ-PRF-04 `[PARTIAL]`
Saved addresses — `GET/POST/PUT/DELETE /api/profile/addresses`. Max 10 addresses per customer (return 409 on overflow). One address may be flagged `isDefault`.

---

## 12. Reference Data

### REQ-REF-01 `[PARTIAL]`
`GET /api/reference/flower-types` — returns localised labels based on `Accept-Language` header. Response must include `Cache-Control: public, max-age=86400` and `ETag`.

### REQ-REF-02 `[PARTIAL]`
`GET /api/reference/freshness-labels` — same caching and localisation requirements as REQ-REF-01.

---

## 13. Real-time (SignalR)

### REQ-RT-01 `[STUB]`
`RealTimeNotificationService` must implement `IBouquetBecameAvailableDispatcher` — call `Clients.Group(radiusGroup).SendAsync("BouquetBecameAvailable", payload)` when a bouquet is activated or a vase comes online. The radius group name must be computed to match the format used by `BouquetHub.JoinRadiusGroup`.

### REQ-RT-02 `[STUB]`
`RealTimeNotificationService` must dispatch `BouquetBecameUnavailable` to all affected radius groups when a bouquet is sold, reserved, or its vase goes offline.

### REQ-RT-03 `[STUB]`
`RealTimeNotificationService` must dispatch `FreshnessUpdated` to radius groups when the `FreshnessScore` on a bouquet changes.

### REQ-RT-04 `[STUB]`
`RealTimeNotificationService` must dispatch `OrderStatusChanged` to `order-{orderId}` group when an order status changes (triggered by `UpdateOrderStatusCommandHandler`, REQ-VOM-02).

### REQ-RT-05 `[MISSING]`
Domain event handlers must invoke `RealTimeNotificationService` when the following domain events fire: `BouquetActivated`, `BouquetSold`, `BouquetReserved`, `BouquetReservationReleased`, `VaseWentOffline`, `VaseCameOnline`, `FreshnessScoreUpdated`, `OrderStatusChanged`.

---

## 14. Freshness Engine

### REQ-FRESH-01 `[PARTIAL]`
When the MQTT service receives a sensor payload (temperature, humidity, CO₂) for a vase, compute a `FreshnessScore` (0–100) and `FreshnessLabel` using the freshness-labels reference table. Update the bouquet's `FreshnessScore` and `FreshnessLabel` fields. Dispatch `FreshnessScoreUpdated` domain event.

### REQ-FRESH-02 `[MISSING]`
When `FreshnessScore` drops below 20 (label: "Poor"), automatically set the bouquet's `Status` to `Expired`, dispatch `BouquetBecameUnavailable` (reason: `removed`), and notify the vendor.

---

## 15. Performance & Reliability

### REQ-PERF-01 `[MISSING]`
Configure ASP.NET Core rate limiting middleware:
- Anonymous endpoints: 60 req/min per IP
- Authenticated endpoints: 120 req/min per user
- Order creation (`POST /api/orders`): 5 req/min per user
- Auth registration: 3 req/min per IP
- Resend verification: 1 req/60 s per user

Return 429 with `Retry-After`, `X-RateLimit-Limit`, `X-RateLimit-Remaining` headers.

### REQ-PERF-02 `[MISSING]`
Configure Redis-backed response caching:
- `GET /api/config`: 1 h TTL
- `GET /api/reference/flower-types`: 24 h TTL
- `GET /api/reference/freshness-labels`: 24 h TTL
- `GET /api/vendors/{id}`: 1 h TTL

Use `ICacheService` (interface already exists).

### REQ-PERF-03 `[MISSING]`
Add `ETag` / `If-None-Match` support to: `GET /api/bouquets/{id}`, `GET /api/vendors/{id}`, `GET /api/reference/*`, `GET /api/profile`. Return 304 Not Modified when the ETag matches.

### REQ-PERF-04 `[MISSING]`
Add `Cache-Control` headers to all cacheable endpoints per the policy table in API-CONTRACTS.md § 19.

---

## 16. Cross-cutting Concerns

### REQ-CC-01 `[MISSING]`
Implement `LocalisationMiddleware` (or action filter) that reads the `Accept-Language` header (`pl-PL` default, `en-US` supported) and sets the thread culture before the request reaches the handler.

### REQ-CC-02 `[MISSING]`
Add `X-Request-Id` response header to all API responses (use `Activity.Current?.TraceId` or a middleware-generated UUID).

### REQ-CC-03 `[MISSING]`
Add `GET /api/health` endpoint (no auth). Returns `{ status: "healthy", version, timestamp }`. Returns 503 if DB or Redis is unavailable.

### REQ-CC-04 `[MISSING]`
Add a global exception-handling middleware that maps unhandled exceptions to RFC 7807 Problem Details responses (type, title, status, detail, traceId).

### REQ-CC-05 `[MISSING]`
Subscription Matching Engine — implement `SubscriptionMatchingService` (background service). When a `BouquetActivated` domain event fires, query all active subscriptions whose criteria match the bouquet (flower type, price range, radius, vendor). For each match: create a `Notification` record, send a push notification via `INotificationServiceClient` (if `notificationFrequency = immediate`), and publish the subscription match to a digest queue (for `daily`/`weekly`).

---

## 17. Messaging Infrastructure

### REQ-INFRA-01 `[STUB]` — P1
`RabbitMQEventBus` class exists in `ServiceCollectionExtensions.cs` and is registered when `UseRabbitMQ: true`, but every method (`PublishAsync`, `PublishBatchAsync`) returns `Task.CompletedTask` — no message is actually published to RabbitMQ. `InMemoryEventBus` is the real in-process fallback. The gap: implement `RabbitMQEventBus` using `RabbitMQ.Client` (or MassTransit) so that domain events are published to durable RabbitMQ exchanges and consumed by background subscribers (subscription matching engine, digest scheduler). This also requires the `RabbitMQ.Client` NuGet package to be added to `FlowerShop.Infrastructure.csproj`.

---

## Summary Table

| ID | Area | Priority | Status |
|---|---|---|---|
| REQ-DB-01 | Database Migration | P0 | MISSING |
| REQ-DB-02 | QR Short Code Column | P0 | MISSING |
| REQ-DB-03 | Opening Hours / Description Columns | P1 | MISSING |
| REQ-AUTH-01 | Customer Registration (Keycloak + DB) | P0 | PARTIAL |
| REQ-AUTH-02 | Resend Verification Endpoint | P1 | MISSING |
| REQ-AUTH-03 | Password Reset Request | P1 | MISSING |
| REQ-AUTH-04 | Password Reset Confirm | P1 | MISSING |
| REQ-CFG-01 | App Config Endpoint | P0 | STUB |
| REQ-DISC-01 | Geo-spatial Nearby Filter | P0 | PARTIAL |
| REQ-DISC-02 | isFavourite in Nearby Results | P1 | MISSING |
| REQ-DISC-03 | Freelancer Location Privacy | P1 | MISSING |
| REQ-DISC-04 | isVendorCurrentlyOpen | P1 | MISSING |
| REQ-DISC-05 | Bouquet Detail Completeness | P1 | PARTIAL |
| REQ-DISC-06 | QR Code Resolution | P0 | MISSING |
| REQ-DISC-07 | QR Code Generation on Create | P0 | MISSING |
| REQ-VND-01 | Opening Hours Domain Model | P1 | MISSING |
| REQ-VND-02 | IsCurrentlyOpen Helper | P1 | MISSING |
| REQ-VND-03 | Public Vendor Endpoint Completeness | P1 | PARTIAL |
| REQ-ORD-01 | Stripe Service | P0 | DONE (EP-04, commit 0aa528e) |
| REQ-ORD-02 | Stripe Keys in OrderCreatedDto | P0 | DONE (EP-04) |
| REQ-ORD-03 | Stripe Webhook Handler | P0 | DONE (EP-04, commit 0aa528e) |
| REQ-ORD-04 | Idempotency-Key for POST /api/orders | P0 | DONE (EP-04); hardening in EP-17 REQ-PAY-01 |
| REQ-ORD-05 | Reservation Expiry Background Service | P0 | DONE (EP-04); refund-on-expiry gap in EP-17 REQ-PAY-04 |
| REQ-ORD-06 | Delivery QR Code Generation | P1 | DONE (EP-04) |
| REQ-ORD-07 | Cancel with Refund | P1 | PARTIAL (refund wired; refundStatus response → EP-17 T-17-008) |
| REQ-VOM-01 | Vendor Order Status Endpoint | P0 | MISSING |
| REQ-VOM-02 | UpdateOrderStatusCommand | P0 | MISSING |
| REQ-VOM-03 | Vendor Order List Endpoint | P1 | MISSING |
| REQ-VOM-04 | Mark Bouquets Sold on Delivery | P1 | MISSING |
| REQ-FAV-01 | Favourites List Handler | P1 | PARTIAL |
| REQ-FAV-02 | Add Favourite (409 on duplicate) | P1 | PARTIAL |
| REQ-FAV-03 | Remove Favourite (idempotent) | P1 | PARTIAL |
| REQ-SUB-01 | Subscription Validation | P1 | PARTIAL |
| REQ-SUB-02 | Disable All Subscriptions | P1 | PARTIAL |
| REQ-NOTIF-01 | FCM Push Notification Service | P0 | STUB |
| REQ-NOTIF-02 | Device Registration Upsert | P0 | STUB |
| REQ-NOTIF-03 | Device Deregistration | P1 | STUB |
| REQ-NOTIF-04 | Notification Feed (unreadCount) | P1 | PARTIAL |
| REQ-NOTIF-05 | Mark Read / Mark All Read | P1 | PARTIAL |
| REQ-PRF-01 | Profile Stats & Settings | P1 | PARTIAL |
| REQ-PRF-02 | Profile Update → Keycloak Sync | P1 | PARTIAL |
| REQ-PRF-03 | Account Deletion (GDPR) | P1 | PARTIAL |
| REQ-PRF-04 | Saved Addresses CRUD | P1 | PARTIAL |
| REQ-REF-01 | Flower Types (cache + l10n) | P2 | PARTIAL |
| REQ-REF-02 | Freshness Labels (cache + l10n) | P2 | PARTIAL |
| REQ-RT-01 | SignalR BouquetBecameAvailable | P1 | STUB |
| REQ-RT-02 | SignalR BouquetBecameUnavailable | P1 | STUB |
| REQ-RT-03 | SignalR FreshnessUpdated | P2 | STUB |
| REQ-RT-04 | SignalR OrderStatusChanged | P1 | STUB |
| REQ-RT-05 | Domain Event → SignalR Wiring | P1 | MISSING |
| REQ-FRESH-01 | Freshness Score Computation | P1 | PARTIAL |
| REQ-FRESH-02 | Auto-expire on Poor Freshness | P2 | MISSING |
| REQ-PERF-01 | Rate Limiting Middleware | P1 | MISSING |
| REQ-PERF-02 | Redis Response Caching | P2 | MISSING |
| REQ-PERF-03 | ETag / If-None-Match | P2 | MISSING |
| REQ-PERF-04 | Cache-Control Headers | P2 | MISSING |
| REQ-CC-01 | Localisation Middleware | P2 | MISSING |
| REQ-CC-02 | X-Request-Id Header | P2 | MISSING |
| REQ-CC-03 | GET /api/health | P1 | MISSING |
| REQ-CC-04 | Global Exception Middleware | P1 | MISSING |
| REQ-CC-05 | Subscription Matching Engine | P1 | MISSING |
| REQ-INFRA-01 | RabbitMQ EventBus Implementation | P1 | STUB |
