# FlowerShop Backend — Epics, Stories & Tasks

**Scope:** Mobile API (FMF) + Vendor Order Management
**Source:** REQ-GAPS.md (2026-03-21)

> **Status (2026-07-12):** This was the original mobile-API plan. Most of it has since shipped —
> the Cart/Order/Subscription/Favourite/Notification/etc. migration is **applied**
> (`AddCartOrderAndRelatedTables`), Stripe is wired (EP-04), and the discovery/orders/profile/
> real-time work is largely built. The per-task `state` fields in the companion
> `tasks-ep0X-*.json` files lag the code — **treat the codebase as ground truth** for what is done.

Execution order: EP-01 must complete before any other epic. EP-04 depends on EP-01. EP-05 depends on EP-04. All others are independent after EP-01.

---

## EP-01 · Database Foundation & Migrations
**Priority:** P0 · **Blocks:** all other epics

Everything else depends on the DB schema being correct. No application logic can be tested without these tables.

### S-01-01 · Generate and apply pending migration
Aggregates, repositories, and EF configurations for Cart, Order, Subscription, Favourite, Notification, DeviceRegistration, and SavedAddress exist but have no migration.

| Task | Requirement | Description |
|---|---|---|
| T-01-001 | REQ-DB-01 | Run `dotnet ef migrations add AddCartAndOrderLines` and verify generated SQL covers: Carts, CartItems, Orders (multi-line), OrderLines, Subscriptions, Favourites, Notifications, DeviceRegistrations, SavedAddresses |
| T-01-002 | REQ-DB-01 | Apply migration to dev DB; verify all 9 new tables exist with correct columns and FK constraints |
| T-01-003 | REQ-DB-01 | Update `FlowerShopDbContextModelSnapshot` — ensure it reflects the new schema |

### S-01-02 · QR short-code column
| Task | Requirement | Description |
|---|---|---|
| T-01-004 | REQ-DB-02 | Add `QrCode varchar(8) NOT NULL UNIQUE` to `Bouquet` aggregate and `BouquetConfiguration`; generate migration `AddBouquetQrCode` |
| T-01-005 | REQ-DB-02 | Seed existing bouquets with generated QR codes (data migration in the migration's `Up()`) |

### S-01-03 · Vendor profile columns
| Task | Requirement | Description |
|---|---|---|
| T-01-006 | REQ-DB-03 | Add `OpeningHours varchar(300)` and `Description text` to `FlowerShop` and `FreelanceFlorist` aggregates and EF configurations; generate migration `AddVendorProfileColumns` |

---

## EP-02 · Customer Authentication & Registration
**Priority:** P0–P1 · **Depends on:** EP-01

### S-02-01 · Verify and fix customer registration
| Task | Requirement | Description |
|---|---|---|
| T-02-001 | REQ-AUTH-01 | Audit `CustomerRegistrationController` — confirm it creates the Keycloak user, creates a `Customer` entity in DB, and links the `customer_id` to the Keycloak user as a custom attribute |
| T-02-002 | REQ-AUTH-01 | Fix `CustomerOnboardingService` if the Keycloak user creation and local `Customer` record creation are not atomic (wrap in a try/rollback pattern) |
| T-02-003 | REQ-AUTH-01 | Add integration test: register → login → verify `customer_id` claim is present in JWT |

### S-02-02 · Email verification resend
| Task | Requirement | Description |
|---|---|---|
| T-02-004 | REQ-AUTH-02 | Add `POST /api/auth/resend-verification` to `AuthController`; delegate to `IKeycloakAdminClient.SendVerificationEmailAsync(userId)` |
| T-02-005 | REQ-AUTH-02 | Apply rate limiting: 1 request per 60 s per authenticated user (use REQ-PERF-01 middleware once available, or a per-endpoint `[EnableRateLimiting]` attribute) |

### S-02-03 · Password reset flow
| Task | Requirement | Description |
|---|---|---|
| T-02-006 | REQ-AUTH-03 | Add `POST /api/auth/password-reset/request` (no auth); delegate to `IKeycloakAdminClient.SendPasswordResetEmailAsync(email)`; always return 202 |
| T-02-007 | REQ-AUTH-04 | Add `POST /api/auth/password-reset/confirm`; delegate to Keycloak token endpoint or admin API; return 200 / 400 |

---

## EP-03 · Bouquet Discovery, QR & Vendor Profile
**Priority:** P0–P1 · **Depends on:** EP-01

### S-03-01 · Geo-spatial nearby filter
| Task | Requirement | Description |
|---|---|---|
| T-03-001 | REQ-DISC-01 | Implement Haversine distance computation as a static helper `GeoHelper.DistanceKm(lat1, lng1, lat2, lng2)` |
| T-03-002 | REQ-DISC-01 | Rewrite `GetNearbyBouquetsQueryHandler` to: (1) load available bouquets with vase locations, (2) filter by Haversine distance ≤ radiusKm, (3) sort by distance ascending, (4) paginate |
| T-03-003 | REQ-DISC-01 | Apply `vendorType`, `search`, `flowerType`, `maxPrice` filters in the handler |

### S-03-02 · isFavourite & freelancer privacy
| Task | Requirement | Description |
|---|---|---|
| T-03-004 | REQ-DISC-02 | In `GetNearbyBouquetsQueryHandler`, when `customerId` is present, load the customer's favourite bouquet IDs from `IFavouriteRepository` and set `IsFavourite` on each result item |
| T-03-005 | REQ-DISC-03 | In `GetNearbyBouquetsQueryHandler` and `GetBouquetDetailQueryHandler`, fuzz coordinates for freelance florists: add random offset ≤ 0.005° (~500 m) before returning; set `ExactAddressHidden = true` |

### S-03-03 · Bouquet detail completeness
| Task | Requirement | Description |
|---|---|---|
| T-03-006 | REQ-DISC-04 | Implement `OpeningHoursParser.IsOpen(string openingHours, DateTimeOffset now, string timezone = "Europe/Warsaw")` — parses human-readable opening hours string and returns `bool?` |
| T-03-007 | REQ-DISC-04 | In `GetBouquetDetailQueryHandler`, populate `IsVendorCurrentlyOpen` using `OpeningHoursParser.IsOpen` |
| T-03-008 | REQ-DISC-05 | Add `IsFavourite`, `AiTags`, `VendorOpeningHours`, and `DistanceKm` (optional) to `BouquetDetailDto` and populate in handler |

### S-03-04 · QR code generation & resolution
| Task | Requirement | Description |
|---|---|---|
| T-03-009 | REQ-DISC-07 | Add `GenerateQrCode()` factory method to `Bouquet` aggregate that produces a cryptographically random 8-char alphanumeric code |
| T-03-010 | REQ-DISC-07 | Update `CreateBouquetCommandHandler` to call `GenerateQrCode()` and persist the code |
| T-03-011 | REQ-DISC-06 | Add `GetByQrCodeAsync(string code)` to `IBouquetRepository` and `BouquetRepository` |
| T-03-012 | REQ-DISC-06 | Update `ResolveBouquetByQrQueryHandler` to look up by QR short code; return 410 when status is `Sold` or `Expired` |

### S-03-05 · Vendor profile & opening hours
| Task | Requirement | Description |
|---|---|---|
| T-03-013 | REQ-VND-01 | Add `OpeningHours` and `Description` properties to `FlowerShop` and `FreelanceFlorist` aggregates with setter methods |
| T-03-014 | REQ-VND-02 | Wire `OpeningHoursParser.IsOpen` into `GetPublicVendorQueryHandler` to populate `IsCurrentlyOpen` |
| T-03-015 | REQ-VND-03 | Complete `PublicVendorDto` with `isCurrentlyOpen`, `openingHours`, `description`, `activeBouquetCount`; populate all fields in `GetPublicVendorQueryHandler` |

---

## EP-04 · Order Lifecycle & Stripe
**Priority:** P0 · **Depends on:** EP-01 · **Status (verified 2026-06-10): DONE** — implemented in commits `0aa528e`, `82847f1` + outbox series; flow manually tested against a Stripe test account (BLIK). Only S-04-06/T-04-019 is **partial** (refund call wired, but `RefundStatus`/`CancelOrderResponseDto`/`refundStatus` response never built — residual moved to EP-17 T-17-008). Correctness-under-failure hardening is **EP-17**.

### S-04-01 · Stripe service
| Task | Requirement | Description |
|---|---|---|
| T-04-001 | REQ-ORD-01 | Add `Stripe.net` NuGet package to `FlowerShop.Infrastructure` |
| T-04-002 | REQ-ORD-01 | Define `IStripeService` interface in `FlowerShop.Application`: `CreatePaymentIntentAsync`, `RefundPaymentAsync` |
| T-04-003 | REQ-ORD-01 | Implement `StripeService` in `FlowerShop.Infrastructure.Payments`; read `Stripe:SecretKey` from configuration |
| T-04-004 | REQ-ORD-01 | Register `IStripeService → StripeService` in `ServiceCollectionExtensions` |

### S-04-02 · Order creation with Stripe
| Task | Requirement | Description |
|---|---|---|
| T-04-005 | REQ-ORD-02 | Inject `IStripeService` into `CreateOrderCommandHandler`; after persisting the order, call `CreatePaymentIntentAsync` with `TotalAmount`, `Currency`, Stripe customer ID |
| T-04-006 | REQ-ORD-02 | Add `StripeClientSecret`, `StripeEphemeralKey`, `StripeCustomerId` to `OrderCreatedDto` |
| T-04-007 | REQ-ORD-02 | Store `StripePaymentIntentId` and `StripeCustomerId` on the `Order` aggregate for later webhook correlation |

### S-04-03 · Stripe webhook handler
| Task | Requirement | Description |
|---|---|---|
| T-04-008 | REQ-ORD-03 | Add `StripeWebhookController` at `POST /api/stripe/webhook`; bypass `[Authorize]` and read raw body |
| T-04-009 | REQ-ORD-03 | Verify `Stripe-Signature` header using `Stripe:WebhookSecret` from configuration |
| T-04-010 | REQ-ORD-03 | Handle `payment_intent.succeeded` → dispatch `ConfirmPaymentCommand` |
| T-04-011 | REQ-ORD-03 | Handle `payment_intent.payment_failed` → dispatch `CancelOrderCommand` |

### S-04-04 · Idempotency for order creation
| Task | Requirement | Description |
|---|---|---|
| T-04-012 | REQ-ORD-04 | Implement `IdempotencyMiddleware` or an `IdempotencyFilter` that: reads `Idempotency-Key` header on `POST /api/orders`, checks Redis for a cached response keyed by `idempotency:{key}`, returns cached response if found, stores the response in Redis with 24 h TTL after successful execution |

### S-04-05 · Reservation expiry background service
| Task | Requirement | Description |
|---|---|---|
| T-04-013 | REQ-ORD-05 | Implement `OrderReservationExpiryService : BackgroundService` in `FlowerShop.Infrastructure` |
| T-04-014 | REQ-ORD-05 | Service loops every 30 s; queries `IOrderRepository.GetExpiredReservationsAsync(now)` and for each: cancels the order, releases bouquet reservations, dispatches `BouquetBecameAvailable` SignalR event |
| T-04-015 | REQ-ORD-05 | Add `GetExpiredReservationsAsync` to `IOrderRepository` and `OrderRepository` |
| T-04-016 | REQ-ORD-05 | Register `OrderReservationExpiryService` as a hosted service in `ServiceCollectionExtensions` |

### S-04-06 · Delivery QR code & cancel with refund
| Task | Requirement | Description |
|---|---|---|
| T-04-017 | REQ-ORD-06 | Add `IQrCodeGenerator` interface; implement `QRCoderQrCodeGenerator` using `QRCoder` NuGet package; generates a base64 PNG data URI from a string payload |
| T-04-018 | REQ-ORD-06 | In `GetOrderDetailQueryHandler`, when `order.Status >= Paid`, call `IQrCodeGenerator.GenerateForOrder(orderId)` and include in `DeliveryQrCode` field of `OrderDetailDto` |
| T-04-019 | REQ-ORD-07 | In `CancelOrderCommandHandler`, when order status is `Paid`, call `IStripeService.RefundPaymentAsync(order.StripePaymentIntentId)` and set `RefundStatus = "refund_initiated"` on the order; return `refundStatus` in the cancel response DTO |

---

## EP-05 · Vendor Order Management
**Priority:** P0 · **Depends on:** EP-04

### S-05-01 · Vendor order status API
| Task | Requirement | Description |
|---|---|---|
| T-05-001 | REQ-VOM-01 | Create `VendorOrdersController` at `PUT /api/vendor/orders/{id}/status` (VendorAccess policy) |
| T-05-002 | REQ-VOM-01 | Validate the requesting vendor owns the order (via VendorId on order lines) |
| T-05-003 | REQ-VOM-02 | Implement `UpdateOrderStatusCommand(OrderId, VendorId, NewStatus, EstimatedDeliveryAt?)` and `UpdateOrderStatusCommandHandler` |
| T-05-004 | REQ-VOM-02 | Enforce allowed status transitions in `Order.TransitionStatus(newStatus)` domain method; return a `Result` error for invalid transitions |
| T-05-005 | REQ-VOM-02 | Append `TimelineEntry` on each transition; set `EstimatedDeliveryAt` when transitioning to `InDelivery` |

### S-05-02 · Real-time & notification on status change
| Task | Requirement | Description |
|---|---|---|
| T-05-006 | REQ-VOM-02 | In `UpdateOrderStatusCommandHandler`, call `IRealTimeNotificationService.SendOrderStatusChangedAsync(orderId, newStatus, label, estimatedDeliveryAt)` |
| T-05-007 | REQ-VOM-02 | In `UpdateOrderStatusCommandHandler`, call `INotificationServiceClient.SendAsync` to deliver push notification to the customer's registered device(s) |

### S-05-03 · Mark bouquets sold on delivery
| Task | Requirement | Description |
|---|---|---|
| T-05-008 | REQ-VOM-04 | When order transitions to `Delivered`, set each bouquet's `Status = Sold` and `SoldAt = now` via `IBouquetRepository.UpdateAsync` |
| T-05-009 | REQ-VOM-04 | Dispatch `BouquetBecameUnavailable` (reason: `sold`) SignalR event for each bouquet |

### S-05-04 · Vendor order list
| Task | Requirement | Description |
|---|---|---|
| T-05-010 | REQ-VOM-03 | Add `GET /api/vendor/orders` to `VendorOrdersController`; paginated, filterable by `status`; returns only orders belonging to the requesting vendor |
| T-05-011 | REQ-VOM-03 | Add `GetByVendorIdAsync(vendorId, statusFilter, page, pageSize)` to `IOrderRepository` and `OrderRepository` |

---

## EP-06 · Favourites, Subscriptions & Notifications
**Priority:** P1 · **Depends on:** EP-01

### S-06-01 · Complete favourites handlers
| Task | Requirement | Description |
|---|---|---|
| T-06-001 | REQ-FAV-01 | Complete `GetFavouritesQueryHandler` — join with bouquet data (thumbnail, status, freshness, price, location), compute `DistanceKm` when coordinates provided, include `SavedAt` |
| T-06-002 | REQ-FAV-02 | Complete `AddFavouriteCommandHandler` — return `Result.Failure("already-favourited")` if already exists; controller maps to 409 |
| T-06-003 | REQ-FAV-03 | Confirm `RemoveFavouriteCommandHandler` returns success even if not found (idempotent) |

### S-06-02 · Complete subscription validation
| Task | Requirement | Description |
|---|---|---|
| T-06-004 | REQ-SUB-01 | Add `CreateSubscriptionCommandValidator` (FluentValidation): validate `flowerType` against reference list, `radiusKm` ∈ {5, 10, 15}, `notificationFrequency` ∈ {immediate, daily, weekly}, `shopId` exists |
| T-06-005 | REQ-SUB-02 | Complete `DisableAllSubscriptionsCommandHandler` — sets `IsActive = false` on all customer subscriptions; returns count |

### S-06-03 · Push notification service
| Task | Requirement | Description |
|---|---|---|
| T-06-006 | REQ-NOTIF-01 | Add `FirebaseAdminSDK` NuGet package (or `Google.Apis.FirebaseCloudMessaging.v1`) |
| T-06-007 | REQ-NOTIF-01 | Implement `FirebaseNotificationService : INotificationServiceClient`; reads Firebase service account JSON from configuration; implement `SendAsync(deviceToken, title, body, data)` |
| T-06-008 | REQ-NOTIF-01 | Register `INotificationServiceClient → FirebaseNotificationService` in DI |

### S-06-04 · Device registration
| Task | Requirement | Description |
|---|---|---|
| T-06-009 | REQ-NOTIF-02 | Complete `DeviceRegistrationService.RegisterAsync` — upsert by `(CustomerId, DeviceToken)`: update `AppVersion`, `Platform`, `LastSeenAt` if exists; insert if not |
| T-06-010 | REQ-NOTIF-03 | Complete `DeviceRegistrationService.UnregisterAsync` — delete by `DeviceToken` |

### S-06-05 · Notification feed
| Task | Requirement | Description |
|---|---|---|
| T-06-011 | REQ-NOTIF-04 | Update `GetNotificationsQueryHandler` to return `UnreadCount` in the paginated envelope |
| T-06-012 | REQ-NOTIF-05 | Complete `MarkNotificationReadCommandHandler` and `MarkAllNotificationsReadCommandHandler` |

---

## EP-07 · Customer Profile & Saved Addresses
**Priority:** P1 · **Depends on:** EP-01

### S-07-01 · Complete profile endpoints
| Task | Requirement | Description |
|---|---|---|
| T-07-001 | REQ-PRF-01 | Complete `GetProfileQueryHandler` — add stats query (total orders, active subscriptions, total spent from `IOrderRepository`); add `NotificationSettings` from `Customer.NotificationSettings` |
| T-07-002 | REQ-PRF-02 | In `UpdateProfileCommandHandler`, call `IKeycloakAdminClient.UpdateUserAsync(userId, firstName, lastName)` to sync name changes |
| T-07-003 | REQ-PRF-03 | Implement `DeleteAccountCommandHandler` — validate `confirmationText == "DELETE"`, set `Customer.Status = PendingDeletion`, set `ScheduledDeletionDate = now + 30 days`, return 202 |

### S-07-02 · Saved addresses CRUD
| Task | Requirement | Description |
|---|---|---|
| T-07-004 | REQ-PRF-04 | Complete `GetSavedAddressesQueryHandler` — return plain array (not paginated) |
| T-07-005 | REQ-PRF-04 | Complete `CreateSavedAddressCommandHandler` — enforce max 10 addresses (return 409 on overflow); handle `isDefault` flag (clear existing default if new one set) |
| T-07-006 | REQ-PRF-04 | Complete `UpdateSavedAddressCommandHandler` and `DeleteSavedAddressCommandHandler` |

---

## EP-08 · Real-time Events & Freshness Engine
**Priority:** P1 · **Depends on:** EP-01, EP-05

### S-08-01 · Wire domain events to SignalR
| Task | Requirement | Description |
|---|---|---|
| T-08-001 | REQ-RT-01 | Implement `IRealTimeNotificationService.SendBouquetBecameAvailableAsync(payload)` in `RealTimeNotificationService` — sends to all radius groups that cover the bouquet's vase location |
| T-08-002 | REQ-RT-02 | Implement `SendBouquetBecameUnavailableAsync(bouquetId, vendorType, reason)` — sends to radius groups |
| T-08-003 | REQ-RT-03 | Implement `SendFreshnessUpdatedAsync(bouquetId, freshnessScore, freshnessLabel)` — sends to radius groups |
| T-08-004 | REQ-RT-04 | Implement `SendOrderStatusChangedAsync(orderId, newStatus, label, estimatedDeliveryAt)` — sends to `order-{orderId}` group |
| T-08-005 | REQ-RT-05 | Register MediatR notification handlers that invoke `RealTimeNotificationService` on domain events: `BouquetActivatedEvent`, `BouquetSoldEvent`, `BouquetReservedEvent`, `BouquetReservationReleasedEvent`, `VaseStatusChangedEvent` |

### S-08-02 · Freshness scoring from MQTT
| Task | Requirement | Description |
|---|---|---|
| T-08-006 | REQ-FRESH-01 | In `MqttDeviceCommunicationService`, when a sensor payload is received for a vase, compute `FreshnessScore` using a configurable formula (temp/humidity/CO₂ weighted average); look up `FreshnessLabel` from the reference table |
| T-08-007 | REQ-FRESH-01 | Update the associated bouquet's `FreshnessScore` and `FreshnessLabel` via `IBouquetRepository`; dispatch `FreshnessScoreUpdatedEvent` |
| T-08-008 | REQ-FRESH-02 | In the `FreshnessScoreUpdatedEvent` handler, if `FreshnessScore < 20`, set bouquet `Status = Expired`, dispatch `BouquetBecameUnavailable` (reason: `removed`), and push a vendor notification |

---

## EP-09 · Performance, Reliability & Cross-cutting
**Priority:** P1–P2 · **Depends on:** EP-01

### S-09-01 · App config & reference data caching
| Task | Requirement | Description |
|---|---|---|
| T-09-001 | REQ-CFG-01 | Complete `ConfigController.GetConfig` — populate all fields from `IConfiguration` / env vars; add `Cache-Control: public, max-age=3600` response header |
| T-09-002 | REQ-REF-01 | In `ReferenceDataController`, add `Cache-Control: public, max-age=86400` and `ETag` to flower-types and freshness-labels responses |
| T-09-003 | REQ-PERF-02 | Implement `CacheService : ICacheService` using `StackExchange.Redis`; wrap `GET /api/config`, `GET /api/reference/*` responses with Redis cache |

### S-09-02 · Rate limiting
| Task | Requirement | Description |
|---|---|---|
| T-09-004 | REQ-PERF-01 | Configure `Microsoft.AspNetCore.RateLimiting` middleware in `Program.cs` with four policies: `anonymous` (60/min/IP), `authenticated` (120/min/user), `order-create` (5/min/user), `auth-register` (3/min/IP) |
| T-09-005 | REQ-PERF-01 | Apply `[EnableRateLimiting]` attributes to relevant controllers/actions; return 429 with `Retry-After`, `X-RateLimit-Limit`, `X-RateLimit-Remaining` headers |

### S-09-03 · ETag & Cache-Control
| Task | Requirement | Description |
|---|---|---|
| T-09-006 | REQ-PERF-03 | Implement `ETagMiddleware` or use `ResponseCachingMiddleware` — compute ETag from response body hash; return 304 when `If-None-Match` matches for: `GET /api/bouquets/{id}`, `GET /api/vendors/{id}`, `GET /api/profile` |
| T-09-007 | REQ-PERF-04 | Add `Cache-Control` response headers per the policy table in API-CONTRACTS.md § 19 |

### S-09-04 · Cross-cutting middleware
| Task | Requirement | Description |
|---|---|---|
| T-09-008 | REQ-CC-01 | Add `LocalisationMiddleware` that reads `Accept-Language` header and sets `CultureInfo.CurrentCulture` / `CurrentUICulture` for the request; default `pl-PL` |
| T-09-009 | REQ-CC-02 | Add `RequestIdMiddleware` that sets `X-Request-Id` response header using `HttpContext.TraceIdentifier` |
| T-09-010 | REQ-CC-03 | Add `GET /api/health` to a `HealthController` (no auth); checks DB connectivity via `DbContext.Database.CanConnectAsync()` and Redis ping; returns 200 or 503 |
| T-09-011 | REQ-CC-04 | Add `GlobalExceptionMiddleware` that catches unhandled exceptions and returns RFC 7807 Problem Details JSON; log the exception with `ILogger` |

### S-09-05 · Subscription matching engine
| Task | Requirement | Description |
|---|---|---|
| T-09-012 | REQ-CC-05 | Implement `SubscriptionMatchingService : BackgroundService` that subscribes to `BouquetActivatedEvent` via `IEventBus`; for each new bouquet, queries `ISubscriptionRepository.FindMatchingAsync(bouquet)` |
| T-09-013 | REQ-CC-05 | For each matching subscription with `notificationFrequency = immediate`: create a `Notification` record and call `INotificationServiceClient.SendAsync` to the customer's device(s) |
| T-09-014 | REQ-CC-05 | Add `FindMatchingAsync(BouquetId, VendorId, FlowerType, Price, Location)` to `ISubscriptionRepository` — queries active subscriptions by type, filtering by `shopId`, `flowerType`, `maxPrice`, and radius |

---

## Epic Dependency Map

```
EP-01  ──► EP-02
       ──► EP-03
       ──► EP-04 ──► EP-05
       ──► EP-06
       ──► EP-07
       ──► EP-08 (also needs EP-05)
       ──► EP-09
```

## Task Count per Epic

| Epic | Stories | Tasks | Priority |
|---|---|---|---|
| EP-01 Database Foundation | 3 | 6 | P0 |
| EP-02 Authentication | 3 | 7 | P0–P1 |
| EP-03 Bouquet Discovery & QR | 5 | 15 | P0–P1 |
| EP-04 Order Lifecycle & Stripe | 6 | 19 | P0 |
| EP-05 Vendor Order Management | 4 | 11 | P0 |
| EP-06 Favourites, Subscriptions & Notifications | 5 | 12 | P1 |
| EP-07 Customer Profile | 2 | 6 | P1 |
| EP-08 Real-time & Freshness | 2 | 8 | P1 |
| EP-09 Performance & Cross-cutting | 5 | 14 | P1–P2 |
| **Total** | **35** | **98** | |
