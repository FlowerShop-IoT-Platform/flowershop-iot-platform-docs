# FlowerShop IoT Platform — Cross-Repo Contract Inventory

> **Purpose**: Authoritative record of all integration touch-points between this repo and external systems (firmware, mobile app, portals, Keycloak, Stripe, Firebase FCM, RabbitMQ).
> **Last scanned**: 2026-03-21
> **Counterpart repos**: `flower-shop-firmware` (ESP32), `flower-shop-mobile` (Flutter/React Native), portal front-ends (served from this repo), `keycloak-config`

---

## Table of Contents

1. [MQTT Topics](#1-mqtt-topics)
2. [REST API Endpoints](#2-rest-api-endpoints)
3. [SignalR Hub — BouquetHub](#3-signalr-hub--boughethub)
4. [Domain / Integration Events (Event Bus)](#4-domain--integration-events-event-bus)
5. [Keycloak / JWT Claims & Auth Policies](#5-keycloak--jwt-claims--auth-policies)
6. [Shared Config Keys & Environment Variables](#6-shared-config-keys--environment-variables)
7. [Firebase FCM Push Notifications](#7-firebase-fcm-push-notifications)
8. [Portal API Clients (Inbound REST)](#8-portal-api-clients-inbound-rest)

---

## 1. MQTT Topics

**Broker**: Mosquitto — configured via `MQTT__BrokerHost` / `MQTT__BrokerPort`
**Counterpart repo**: `flower-shop-firmware` (ESP32 smart vase firmware)

> QoS default is `AtLeastOnce (1)` unless noted. Retain flag is `false` unless noted.

### 1a. Backend → Device (Publish)

| Topic | Direction | QoS | Retain | Payload Schema | Owner | Status | File |
|-------|-----------|-----|--------|----------------|-------|--------|------|
| `smartvase/{vendorId}/{deviceId}/commands` | **Publish** | 1 | false | See command payload variants below | This repo | Active | `Infrastructure/Services/MqttService.cs`, `Application/EventHandlers/OrderPaidVaseNotificationHandler.cs` |
| `smartvase/{vendorId}/{deviceId}/wifi` | **Publish** | TBD | false | `{ "ssid": string, "password": string, "securityType": string, "isHidden": bool, "priority": int }` | This repo | **STUB — not implemented** | `Application/Services/IMqttService.cs` |
| `smartvase/{vendorId}/{deviceId}/config` | **Publish** | 1 | false | `{ "sensor_interval": ulong, "heartbeat_interval": ulong }` | This repo | **DISABLED** — firmware ESP32 const-variable bug | `Infrastructure/IoT/MqttDeviceCommunicationService.cs` |
| `devices/{deviceId}/display` | **Publish** | 1 | false | `{ "status": string, "price": decimal, "qr_code": string, "freshness_score": int, "custom_message": string, "theme": string, "timestamp": long }` | This repo | Active (legacy format) | `Infrastructure/IoT/MqttDeviceCommunicationService.cs` |
| `devices/{deviceId}/alerts` | **Publish** | 1 | false | `{ "type": string, "severity": string, "message": string, "timestamp": long }` | This repo | Active | `Infrastructure/IoT/MqttDeviceCommunicationService.cs` |
| `devices/{deviceId}/content` | **Publish** | 1 | false | `{ "status": string, "price": decimal, "qr_code_data": string, "freshness_score": int, "custom_message": string, "theme": string, "timestamp": long }` | This repo | Active | `Infrastructure/IoT/MqttDeviceCommunicationService.cs` |

**`status` enum values**: `available`, `reserved`, `sold`, `removed`
**`theme` enum values**: `default`, `elegant`, `modern`, `rustic`
**`type` (alert) enum values**: `low_battery`, `connection_issue`, `sensor_malfunction`, `maintenance_required`
**`severity` enum values**: `low`, `medium`, `high`, `critical`

#### `smartvase/{vendorId}/{deviceId}/commands` payload variants

All command payloads share `command` (string) and `timestamp` (long). There is **no separate `/display` topic** — display updates ride the `commands` topic.

> ⚠️ **Timestamp units differ:** the **display** command (`SendVaseDisplayCommandAsync`) uses `timestamp` in **seconds** (`ToUnixTimeSeconds`); all **generic** commands (`SendVaseCommandAsync`, e.g. `prepare_delivery`) use `timestamp` in **milliseconds** (`ToUnixTimeMilliseconds`).

**1. Display update** (`command: "updateDisplay"`) — nested `payload` object, timestamp in **seconds**:

```json
{
  "command": "updateDisplay",
  "timestamp": 1712745600,
  "payload": {
    "status": "AVAILABLE",
    "price": 89.00,
    "currency": "PLN",
    "bouquetId": "b3f1a2c4-...",
    "qrCode": "https://app.findmyflowers.pl/Bouquets/Details/{bouquetId}?addToCart=true&vaseId={deviceId}",
    "message": "..."
  }
}
```

**2. Generic command** (flat payload; optional `correlation_id`; params merged at top level; timestamp in **milliseconds**):

```json
{ "command": "restart|ping|configure|...", "timestamp": 1712745600000, "correlation_id": "..." }
```

**3. Order-driven `prepare_delivery`** (from `OrderPaidVaseNotificationHandler` on `OrderPaidEvent`; flat params, timestamp in **milliseconds**):

```json
{
  "command": "prepare_delivery",
  "timestamp": 1712745600000,
  "display_mode": "delivery",
  "delivery_qr": "geo:<lat>,<lon>?q=<url-encoded-address>",
  "delivery_status": "PREPARING",
  "delivery_address": "<street>, <postalCode> <city>",
  "order_info": "ORDER #A1B2C3D4",
  "customer_id": "<customerId>"
}
```

`display_mode` / `delivery_qr` / `delivery_status` / `delivery_address` are present only for delivery orders that have a generated delivery QR code; `order_info` and `customer_id` are always present.

### 1b. Device → Backend (Subscribe)

| Topic Pattern | Direction | QoS | Payload Schema | Owner | Status | Handler |
|---------------|-----------|-----|----------------|-------|--------|---------|
| `smartvase/platform/heartbeat` | **Subscribe** | 1 | See heartbeat schema below | Firmware | Active | `MqttDeviceCommunicationService.HandleHeartbeat()` |
| `smartvase/+/+/heartbeat` | **Subscribe** | 1 | See heartbeat schema below | Firmware | Active | `MqttDeviceCommunicationService.HandleHeartbeat()` |
| `smartvase/+/+/status` | **Subscribe** | 1 | `{ "device_id": string, "state": string, ... }` | Firmware | Active | `MqttDeviceCommunicationService` |
| `smartvase/+/+/sensors` | **Subscribe** | 1 | See sensor schema below | Firmware | Active | `MqttDeviceCommunicationService.HandleSensorData()` |
| `devices/+/heartbeat` | **Subscribe** | 1 | Legacy heartbeat | Firmware | Active (legacy compat) | `MqttDeviceCommunicationService` |
| `devices/+/status` | **Subscribe** | 1 | Legacy status | Firmware | Active (legacy compat) | `MqttDeviceCommunicationService` |
| `devices/+/bouquet_detected` | **Subscribe** | 1 | `{ "device_id": string, ... }` | Firmware | **STUB handler** | `MqttDeviceCommunicationService.HandleBouquetDetected()` |
| `devices/+/bouquet_removed` | **Subscribe** | 1 | `{ "device_id": string, ... }` | Firmware | **STUB handler** | `MqttDeviceCommunicationService.HandleBouquetRemoved()` |

**Heartbeat payload schema** (Device → Backend):
```json
{
  "device_id": "vase-001",
  "shop_id": "shop-42",
  "timestamp": 1234567890,
  "state": "online|offline",
  "wifi_rssi": -65,
  "free_heap": 45000,
  "uptime": 3600000,
  "bouquet_present": true,
  "bouquet_price": 89.99,
  "temperature": 22.5,
  "humidity": 65.0,
  "sensor_errors": []
}
```

**Sensor data payload schema** (Device → Backend):
```json
{
  "temperature": 22.5,
  "humidity": 65.0,
  "distance": 25.0,
  "bouquet_present": true,
  "state": "active|idle",
  "bouquet_price": 89.99
}
```

---

## 2. REST API Endpoints

**Base URL**: `http://localhost:8080` (dev) — `https://api.findmyflowers.pl` (prod)
**API version prefix**: `/api/v1/`
**Counterpart repos**: `flower-shop-mobile` (mobile app), portals (served from this repo), firmware heartbeat endpoint

### 2a. Public / Anonymous Endpoints

| Method | Path | Auth | Request Body | Response | Counterpart |
|--------|------|------|-------------|----------|-------------|
| POST | `/api/v1/auth/dev-token` | None | `{ username, vendorId?, vendorType?, role? }` | `{ token, tokenType, expiresIn, username, vendorId?, vendorType? }` | Dev tooling |
| POST | `/api/v1/admin-accounts/register` | None | `{ username, email, password, firstName?, lastName?, masterKey? }` | `{ userId, username, email, role, token, expiresIn, message }` | Admin bootstrap |
| POST | `/api/v1/vendor-accounts/register-with-admin` | None | `{ vendorType, businessName, description?, contactEmail, contactPhone?, latitude, longitude, fullAddress?, adminUsername, adminPassword, onlineAddress?, billingAddress?{street,city,postalCode,country}, acceptedTerms?, termsVersion? }` (EP-19 billing/T&C fields) | `{ vendorId, vendorType, adminUsername, stripeCustomerId?, token, expiresIn }` | Admin Portal, Vendor Portal sign-up |
| POST | `/api/v1/vendor-accounts/setup-intent` | None | `{ vendorId }` | `{ clientSecret }` (Stripe SetupIntent — EP-19; 404 if vendor unknown, 400 if no Stripe customer) | Vendor Portal sign-up wizard |
| POST | `/api/v1/customers/register` | None | `{ username, email, password, customerName, preferredVendorType?, phoneNumber? }` | `AuthResponse` `{ token, tokenType, expiresIn, username }` 201 | Mobile App |
| POST | `/api/v1/auth/password-reset/request` | None | `{ email }` | 202 (always; reset completes on Keycloak hosted page — no `/confirm` endpoint) | Mobile App |
| POST | `/api/v1/auth/resend-verification` | Bearer (any authenticated) | — | 202 queued / 204 already-verified / 429 (rate-limit `auth-email`) | Mobile App |
| POST | `/api/v1/auth/logout` | Bearer (any authenticated) | — | 204 (jti blacklisted until expiry) / 400 no-jti | Mobile App |
| GET | `/api/v1/bouquets/nearby` | None | Query: `lat, lng, radiusKm, search?, vendorType?, flowerType?, maxPrice?, sortBy, page, pageSize` | `NearbyBouquetsResult` | Mobile App |
| GET | `/api/v1/bouquets/{id}` | None | — | `BouquetDetailDto` | Mobile App, Portals |
| GET | `/api/v1/bouquets/qr/{code}` | None | Query: `lat?, lng?` | `BouquetDetailDto` | Mobile App (QR scan) |
| GET | `/api/v1/bouquets/available` | None | Query: `page, pageSize, category?, priceRange?` | `IEnumerable<AvailableBouquetReadModel>` | Customer App |
| GET | `/api/v1/vendors/{id}/public` | None | — | `PublicVendorDto` | Mobile App |
| GET | `/api/v1/vendors/{id}/bouquets` | None | Query: `page, pageSize` | `VendorPublicBouquetsResult` | Mobile App |
| POST | `/api/v1/smartvases/heartbeat/{deviceId}` | None | `{ temperature, humidity, batteryLevel, sensorErrors[]? }` | `HeartbeatResponseDto` | Firmware |
| POST | `/api/mock-vase/{vaseId}/bouquet-image` | None | `IFormFile` (multipart) | `{ imageUrl }` | Admin Portal, Vendor Portal |
| GET | `/api/mock-vase/image/{vaseId}` | None | — | JPEG | Portals |
| GET | `/api/mock-vase/{vaseId}/status` | None | — | `VaseStatusResponse` | Dev tooling |
| GET | `/api/v1/reference/flower-types` | None | — | `List<{ key, label }>` | Mobile App |
| GET | `/api/v1/reference/freshness-labels` | None | — | `List<{ minScore, maxScore, label, colour }>` | Mobile App |
| GET | `/api/v1/config` | None | — | `{ stripePublishableKey, supportEmail, termsUrl, privacyPolicyUrl, minAppVersion, forceUpdate }` | Mobile App |
| POST | `/api/v1/vendor-accounts/accept-invitation/{token}` | None | `{ temporaryPassword, newPassword, acceptTerms, deviceInfo? }` | `InvitationAcceptanceResult` | Email link → Mobile/Browser |
| POST | `/api/v1/docs/handoff` | Bearer (any signed-in role; rate limit `docs-handoff` 20/min) | `{ returnPath? }` (relative docs path) | `{ url, expiresAt }` — `https://docs.findmyflowers.pl/auth/handoff?code=…&next=…`, code single-use, 60 s (EP-38) | Vendor Portal, Customer App, vendor app (EP-M18), customer mobile app |
| POST | `/api/v1/docs/handoff/redeem` | Bearer, `DocsPortalClient` (client-credentials token of `flowershop-docs`) | `{ code }` | `{ subject, name, roles[], returnPath, sessionExpiresAt }` / 404 `invalid-code` (EP-38) | FlowerShop.DocsPortal |
| POST | `/api/v1/docs/roles` | Bearer, `DocsPortalClient` | `{ subject }` | `{ subject, roles[], known, active }` — session re-validation every 15 min (EP-38) | FlowerShop.DocsPortal |

### 2b. Customer Endpoints (Policy: `CustomerAccess`)

| Method | Path | Request Body | Response | Counterpart |
|--------|------|-------------|----------|-------------|
| GET | `/api/v1/cart` | — | `CartDto` | Mobile App |
| POST | `/api/v1/cart/items` | `{ bouquetId }` | `CartDto` (`items[].vaseId` null for online listings) / 409 `bouquet-reserved` or `bouquet-not-listed` (EP-39) | Mobile App |
| DELETE | `/api/v1/cart/items/{bouquetId}` | — | 204 | Mobile App |
| DELETE | `/api/v1/cart` | — | 204 | Mobile App |
| POST | `/api/v1/orders/preview` | `{ bouquetIds[], deliveryMethod, street?, city?, postalCode?, notes? }` | `OrderPreviewDto` (+ `pickupAvailable`, `lines[].deliveryOnly`, EP-39) | Mobile App |
| POST | `/api/v1/orders` | `{ bouquetIds[], deliveryMethod, expectedTotalAmount, street?, city?, postalCode?, notes? }` + `Idempotency-Key` header | `OrderCreatedDto` | Mobile App |
| POST | `/api/v1/orders/{id}/confirm-payment` | `{ paymentIntentId, stripeCustomerId? }` | 204 | Mobile App (Stripe) |
| POST | `/api/v1/orders/{id}/cancel` | `{ reason }` | `CancelOrderResponseDto` (200) | Mobile App |
| POST | `/api/v1/orders/{id}/confirm-delivery` | — | 204 | Mobile App |
| POST | `/api/v1/orders/{orderId}/lines/{bouquetId}/feedback` | `{ rating, comment?, isAnonymous }` | `{ id }` 201 | Mobile App |
| GET | `/api/v1/orders/me/pending-feedback` | — | `IReadOnlyList<PendingFeedbackDto>` | Mobile App |
| GET | `/api/v1/orders` | Query: `status?, page, pageSize` | `OrderListResult` | Mobile App |
| GET | `/api/v1/orders/{id}` | — | `OrderDetailDto` | Mobile App |
| GET | `/api/v1/favourites` | Query: `page, pageSize` | `FavouritesResult` | Mobile App |
| POST | `/api/v1/favourites/{bouquetId}` | — | 204 | Mobile App |
| DELETE | `/api/v1/favourites/{bouquetId}` | — | 204 | Mobile App |
| GET | `/api/v1/subscriptions` | Query: `page, pageSize` | `SubscriptionListResult` | Mobile App |
| GET | `/api/v1/subscriptions/{id}` | — | `SubscriptionDto` | Mobile App |
| POST | `/api/v1/subscriptions` | `{ type, frequency, flowerType?, maxPrice?, currency?, shopId?, centerLatitude?, centerLongitude?, radiusKm? }` | `SubscriptionDto` | Mobile App |
| PUT | `/api/v1/subscriptions/{id}` | `{ type?, frequency?, flowerType?, maxPrice?, currency?, shopId?, centerLatitude?, centerLongitude?, radiusKm?, isActive? }` | 204 | Mobile App |
| PUT | `/api/v1/subscriptions/disable-all` | — | 204 | Mobile App |
| DELETE | `/api/v1/subscriptions/{id}` | — | 204 | Mobile App |
| GET | `/api/v1/notifications` | Query: `unreadOnly, page, pageSize` | `NotificationListResult` | Mobile App |
| PUT | `/api/v1/notifications/{id}/read` | — | 204 | Mobile App |
| PUT | `/api/v1/notifications/read-all` | — | 204 | Mobile App |
| POST | `/api/v1/notifications/devices` | `{ deviceToken, platform, appVersion? }` | 204 | Mobile App |
| DELETE | `/api/v1/notifications/devices/{deviceToken}` | — | 204 | Mobile App |
| GET | `/api/v1/profile` | — | `CustomerProfileDto` | Mobile App |
| PUT | `/api/v1/profile` | `{ name?, phoneNumber?, preferredFlowerTypes[]? }` | 204 | Mobile App |
| PUT | `/api/v1/profile/notification-settings` | `{ pushEnabled, emailEnabled, subscriptionDigestEnabled }` | 204 | Mobile App |
| POST | `/api/v1/profile/phone/send-code` | `{ phoneNumber }` | 202 (400 `invalid-phone-format`, 502 `sms-send-failed`) | Mobile App |
| POST | `/api/v1/profile/phone/verify-code` | `{ code }` | 204 (400 invalid/expired) | Mobile App |
| DELETE | `/api/v1/profile` | — | 204 | Mobile App |
| GET | `/api/v1/profile/addresses` | — | `List<SavedAddressDto>` | Mobile App |
| POST | `/api/v1/profile/addresses` | `{ label, street, city, postalCode, notes?, isDefault }` | `SavedAddressDto` | Mobile App |
| PUT | `/api/v1/profile/addresses/{id}` | `{ label?, street?, city?, postalCode?, notes?, isDefault? }` | 204 | Mobile App |
| DELETE | `/api/v1/profile/addresses/{id}` | — | 204 | Mobile App |

### 2c. Vendor Endpoints (Policy: `VendorAccess`)

| Method | Path | Request Body | Response | Counterpart |
|--------|------|-------------|----------|-------------|
| POST | `/api/v1/bouquets` | `CreateBouquetCommand` (name, description, price, currency, vaseId?, imageUrls[], flowerTypes[], ...) | `BouquetDetails` 201 | Vendor Portal |
| GET | `/api/v1/vendors/{vendorId}/bouquets` | Query: `status?` | `List<BouquetDetails>` | Vendor Portal |
| PUT | `/api/v1/bouquets/{id}` | `UpdateBouquetCommand` | `BouquetDetails` | Vendor Portal |
| DELETE | `/api/v1/bouquets/{id}` | — | 204 (online listings are marked Removed, not deleted; 400 while a customer holds it, EP-39) | Vendor Portal |
| POST | `/api/v1/vendor/online-listings` | multipart `name, price, freshnessDays (1..3), description?, category?, flowerTypes?, photo` + optional `Idempotency-Key` | `OnlineListingDto` 201 / 409 `vendor-not-online` / 403 `vendor-pending-approval`, `subscription-inactive` (EP-39, §25) | Vendor Portal, vendor mobile app (EP-M19) |
| GET/PUT | `/api/v1/vendor/settings/sales-mode` | PUT `{ salesMode: "InStore" \| "OnlineOnly" }` | `{ salesMode, assignedVaseCount, canSwitchToOnlineOnly }` / 409 `vendor-has-vases` (EP-39) | Vendor Portal, vendor mobile app (EP-M19) |
| PUT | `/api/v1/bouquets/{id}/price` | `{ price, currency, notes? }` | `BouquetDetails` | Vendor Portal |
| GET | `/api/v1/bouquets/{id}/analytics` | Query: `period` | `BouquetAnalyticsDto` | Vendor Portal |
| GET | `/api/v1/smartvases/my-vases` | — | `List<VaseDetailsDto>` | Vendor Portal |
| GET | `/api/v1/smartvases/{id}` | — | `VaseDetailsDto` | Vendor Portal |
| PATCH | `/api/v1/smartvases/{id}/status` | `{ status, price?, qrCodeData?, freshnessScore?, customMessage? }` | 200 | Vendor Portal |
| GET | `/api/v1/analytics/vendor` | Query: `period, includeComparisons` | `VendorAnalyticsDto` | Vendor Portal — **BUG: returns random VendorId data** |
| GET | `/api/v1/analytics/vases` | Query: `vaseId?, period, metrics?` | `IEnumerable<VasePerformanceMetrics>` | Vendor Portal — **BUG: returns random VendorId data** |
| GET | `/api/v1/analytics/realtime` | Query: `refreshRate` | `RealtimeAnalyticsDto` | Vendor Portal — **BUG: returns random VendorId data** |
| GET | `/api/v1/analytics/export` | Query: `exportType, format, period` | File (CSV/XLSX) | Vendor Portal — **BUG: returns random VendorId data** |
| GET | `/api/v1/analytics/benchmarks` | Query: `benchmarkType, vendorType?, period` | `BenchmarkAnalyticsDto` | Vendor Portal |
| GET | `/api/v1/vendors/{vendorId}/employees` | Query: `includeInactive` | `List<EmployeeDto>` | Vendor Portal |
| POST | `/api/v1/vendors/{vendorId}/employees` | `{ fullName, email, phone?, role, permissions[]? }` | `EmployeeInvitationResult` 201 | Vendor Portal |
| GET | `/api/v1/vendors/{vendorId}/employees/{employeeId}` | — | `EmployeeDetailsDto` | Vendor Portal |
| PUT | `/api/v1/vendors/{vendorId}/employees/{employeeId}/role` | `{ newRole, additionalPermissions[]? }` | 200 | Vendor Portal |
| DELETE | `/api/v1/vendors/{vendorId}/employees/{employeeId}` | `{ reason, notifyEmployee }` | 200 | Vendor Portal |
| GET | `/api/v1/vendors/{vendorId}/employees/{employeeId}/activity` | Query: `pageNumber, pageSize` | `PagedResult<EmployeeActivityDto>` | Vendor Portal |
| GET | `/api/v1/vendor/payments` | Query: `filter?, page, pageSize` | `VendorPaymentListResult` (EP-18) — tenant-scoped, no Stripe-internal ids | Vendor Portal (payments dashboard) |
| GET | `/api/v1/vendor/payments/metrics` | Query: `fromDate?, toDate?` | `VendorPaymentMetricsDto` (EP-18) | Vendor Portal (payments dashboard) |
| GET | `/api/v1/vendor/design-assistant/usage` | — | `DesignUsageDto` (EP-28 — flags, monthly quota, limits, styles) | Vendor mobile app |
| POST/GET | `/api/v1/vendor/design-sessions` | POST `{ inputMode }` + optional `Idempotency-Key`; GET `?skip&take` | 201 `DesignSessionDto` / `DesignSessionSummaryDto[]` (EP-28) | Vendor mobile app |
| GET/DELETE | `/api/v1/vendor/design-sessions/{id}` | — | `DesignSessionDto` (polling target) / 204 | Vendor mobile app |
| POST/DELETE | `/api/v1/vendor/design-sessions/{id}/photos[/{photoId}]` | multipart `photo` + `clientPhotoId` | `DesignSessionDto` | Vendor mobile app |
| GET | `/api/v1/vendor/design-sessions/{id}/photos/{photoId}/content`, `/recipes/{rid}/preview/content` | — | private image bytes (`private, no-store`) | Vendor mobile app |
| POST | `/api/v1/vendor/design-sessions/{id}/recognize`, `/recipes`, `/recipes/{rid}/preview` | `{ expectedVersion }` / `{ expectedVersion, size, mode, count, style?, costCeiling? }` / `{}` + **required** `Idempotency-Key` | 202 `DesignSessionDto` (billable, quota-checked) | Vendor mobile app |
| PUT | `/api/v1/vendor/design-sessions/{id}/inventory` | `{ expectedVersion, confirmed, items[] }` | `DesignSessionDto` | Vendor mobile app |
| PUT/GET | `/api/v1/vendor/design-sessions/{id}/recipes/{rid}/saved` / `/cost?sellingPrice=` | `{ saved }` | `DesignSessionDto` / `RecipeCostDto` | Vendor mobile app |
| GET | `/api/v1/vendor/design-recipes?saved=true` | — | `DesignRecipeDto[]` | Vendor mobile app |
| GET/PUT | `/api/v1/vendor/design-cost-profile` | `{ expectedVersion, basis, laborMinutes, hourlyRate, overheadPercent }` | `DesignCostProfileDto` | Vendor mobile app |
| PUT/DELETE | `/api/v1/vendor/design-sessions/{id}/recipes/{rid}/feedback` | PUT `{ target: Recipe\|Preview, rating: Up\|Down, reasons[], comment? }`; DELETE `?target=` | `DesignSessionDto` (EP-33) | Vendor Portal, vendor mobile app (EP-M13) |
| POST | `/api/v1/vendor/design-sessions/{id}/recipes/{rid}/refine` | `{ chips[], note? }` + **required** `Idempotency-Key` | 202 `DesignSessionDto` (one composition, EP-33) | Vendor Portal, vendor mobile app (EP-M13) |
| POST | `/api/v1/vendor/design-sessions/{id}/recipes/{rid}/preview` | optional `{ corrections[], note? }` = re-render with corrections (EP-33) | 202 `DesignSessionDto` | Vendor Portal, vendor mobile app (EP-M13) |
| POST | `/api/v1/vendor/design-sessions/{id}/recipes/{rid}/signals` | `{ kind: "Copied" }` | 204 (EP-33) | Vendor Portal, vendor mobile app (EP-M13) |
| GET/PUT | `/api/v1/vendor/design-style-profile` | PUT `{ expectedVersion, notes?, avoid[{ name, stockItemId? }], favourites[], defaultStyle?, defaultColourPreference?, learnFromRatings, improveAiConsent }` | `DesignStyleProfileDto` (EP-33) | Vendor Portal, vendor mobile app (EP-M13) |
| DELETE | `/api/v1/vendor/design-feedback` | — ("Forget my ratings") | `DesignStyleProfileDto` (EP-33) | Vendor Portal, vendor mobile app (EP-M13) |
| POST | `/api/v1/vendor/design-sessions/{id}/analyze` | `{ expectedVersion }` + **required** `Idempotency-Key` | 202 `DesignSessionDto` — `BouquetAnalysis` job (EP-35, one recognition) | Vendor Portal, vendor mobile app (EP-M15) |
| PUT | `/api/v1/vendor/design-sessions/{id}/analysis/items/{itemId}` | `{ stockItemId?, notInStock, quantity? }` or EP-41 `{ rename: { name, variety? } }` / `{ clearRename: true }` | `DesignSessionDto` (EP-35; rename re-matches and records a verdict, EP-41) | Vendor Portal, vendor mobile app (EP-M15, EP-M21) |
| GET | `/api/v1/vendor/design-sessions/{id}/media/crops/{itemId}`, `/api/v1/vendor/cost-assessments/{id}/media/crops/{itemId}` | — | image — private crop of an analysed item (EP-41) | Vendor Portal, vendor mobile app (EP-M21) |
| GET | `/api/v1/vendor/atlas/vocabulary` | — | `[{ name, language, kind, typeKey, source }]` — "What is it?" names (EP-41) | Vendor Portal, vendor mobile app (EP-M21) |
| GET | `/api/v1/atlas/images/{imageId}/content`, `/api/v1/atlas/attributions` | — (anonymous) | approved atlas image / image credits (EP-41) | Vendor Portal, vendor mobile app (EP-M21) |
| POST | `/api/v1/vendor/design-sessions/{id}/analysis/use` | `{ expectedVersion }` | `DesignSessionDto` — matched items become confirmed inventory (EP-35) | Vendor Portal, vendor mobile app (EP-M15) |
| PUT | `/api/v1/vendor/design-sessions/{id}/analysis/items/{itemId}/verdict`, `/inventory/items/{itemId}/verdict` | `{ identity: correct\|wrong\|unsure, actualName?, actualColor?, actualVariety?, notInBouquet?, actualQuantity? }` | `DesignSessionDto` — recognition verdict, idempotent per line (EP-40) | Vendor Portal, vendor mobile app (EP-M20) |
| GET/POST | `/api/v1/vendor/cost-assessments` | POST `{ note? }` (+ optional `Idempotency-Key`) | `CostAssessmentSummaryDto[]` / 201 `CostAssessmentDto` (EP-36) | Vendor Portal, vendor mobile app (EP-M16) |
| GET/DELETE | `/api/v1/vendor/cost-assessments/{id}` | — | `CostAssessmentDto` / 204 (EP-36) | Vendor Portal, vendor mobile app (EP-M16) |
| POST/DELETE | `/api/v1/vendor/cost-assessments/{id}/photos[/{photoId}]` | multipart `photo` | `CostAssessmentDto` (EP-36) | Vendor Portal, vendor mobile app (EP-M16) |
| POST | `/api/v1/vendor/cost-assessments/{id}/analyze` | `{ expectedVersion }` + **required** `Idempotency-Key` | `CostAssessmentDto` — synchronous, own monthly quota (EP-36) | Vendor Portal, vendor mobile app (EP-M16) |
| PUT | `/api/v1/vendor/cost-assessments/{id}/lines` · `/selling-price` | `{ expectedVersion, lines[…] }` · `{ expectedVersion, sellingPrice }` | `CostAssessmentDto` (EP-36) | Vendor Portal, vendor mobile app (EP-M16) |
| POST | `/api/v1/vendor/cost-assessments/{id}/design` | — | `DesignSessionDto` — "Open in AI Florist" (EP-36) | Vendor Portal, vendor mobile app (EP-M16) |
| POST/GET | `/api/v1/vendor/stock/counts[/{countId}]` | POST multipart `photos` (1–5) + **required** `Idempotency-Key` | 201/200 `StockCountDto` (EP-34, stock AI quota) | Vendor Portal, vendor mobile app (EP-M14) |
| GET/POST/PUT | `/api/v1/customer/bouquet-requests[/{id}]` | `{ description, budget…, desiredAt, fulfilment, address/pin… }` (+ `expectedVersion` on PUT, optional `Idempotency-Key`) | `CustomerRequestDto` (EP-37, §23.1) | Customer App, customer mobile app (follow-up) |
| POST/DELETE/GET | `/api/v1/customer/bouquet-requests/{id}/images[/{imageId}]` | multipart `image` | `CustomerRequestDto` / private image (EP-37) | Customer App, customer mobile app (follow-up) |
| POST | `/api/v1/customer/bouquet-requests/{id}/publish` · `/cancel` · `/order/cancel` · `/time-change/answer` | — / `{ accept }` | `CustomerRequestDto` (EP-37) | Customer App, customer mobile app (follow-up) |
| GET/POST | `/api/v1/customer/bouquet-requests/{id}/offers[/{offerId}/accept\|report]` | accept `{ offerRevision }` | `CustomerOfferDto[]` / `AcceptOfferResultDto` (custom order + PaymentIntent, EP-37) | Customer App, customer mobile app (follow-up) |
| GET | `/api/v1/vendor/bouquet-requests[/{id}]` · `/vendor/custom-orders` | bbox query | `VendorRequestSummaryDto[]` / `VendorRequestDto` (fuzzed, no PII before payment, EP-37 §23.2) | Vendor Portal, vendor mobile app (EP-M17) |
| POST/PUT | `/api/v1/vendor/bouquet-requests/{id}/offers[/{offerId}[/withdraw\|images]]` | `{ price, deliveryFee?, readyAt, deliveryAt?, notes? }` (+ `expectedRevision`) | `VendorRequestDto` (EP-37) | Vendor Portal, vendor mobile app (EP-M17) |
| POST | `/api/v1/vendor/bouquet-requests/{id}/phase` · `/time-change` · `/reconstruct` · `/report` | `{ phase }` / `{ newReadyAt, newDeliveryAt?, reasons[], note? }` / — | `VendorRequestDto` / `DesignSessionDto` (EP-37) | Vendor Portal, vendor mobile app (EP-M17) |
| GET/PUT | `/api/v1/vendor/marketplace-settings` · `/vendor/notifications[/{id}/read\|read-all]` | settings `{ expectedVersion, serviceRadiusKm, … }` | `VendorMarketplaceSettingsDto` / `VendorNotificationListDto` (EP-37) | Vendor Portal, vendor mobile app (EP-M17) |
| GET/POST | `/api/v1/admin/bouquet-requests[/{id}/hide\|unhide\|cancel]` | `{ reason? }` | `AdminRequestDto[]` (EP-37) | Admin Portal |
| WS | `/hubs/marketplace` (SignalR) | — | `request.*`, `offer.*`, `order.*` events (EP-37, §23.4) | Customer App, Vendor Portal, mobile apps |
| POST | `/api/v1/vendor/stock/counts/{countId}/apply` | `{ lines[{ stockItemId, countedQuantity }] }` + **required** `Idempotency-Key` | `StockCountDto` — Counted movements, all or nothing (EP-34) | Vendor Portal, vendor mobile app (EP-M14) |
| PUT | `/api/v1/vendor/stock/counts/{countId}/lines/{lineId}/verdict` | same body as the design verdicts | `StockCountDto` — recognition verdict, no stock change (EP-40) | Vendor Portal, vendor mobile app (EP-M20) |
| GET/POST | `/api/v1/vendor/stock` | GET `?search&kind&status&includeArchived`; POST `{ kind, name, color?, unit?, quantity, unitCost?, lowStockThreshold?, supplier?, notes? }` | `StockOverviewDto` / 201 `StockItemDto` (EP-30) | Vendor Portal, vendor mobile app |
| GET/PUT/DELETE | `/api/v1/vendor/stock/{id}` | PUT `{ expectedVersion, …fields }` | `StockItemDetailsDto` / `StockItemDto` / 204 archive | Vendor Portal, vendor mobile app |
| POST | `/api/v1/vendor/stock/{id}/restore` | — | `StockItemDto` | Vendor Portal, vendor mobile app |
| POST/GET | `/api/v1/vendor/stock/{id}/movements` | POST `{ type, quantity, unitCost?, note? }` + optional `Idempotency-Key`; GET `?skip&take` | `StockItemDto` / `StockMovementDto[]` | Vendor Portal, vendor mobile app |
| GET | `/api/v1/vendor/stock/movements` | `?skip&take` | `StockMovementDto[]` (whole shop) | Vendor Portal |
| POST | `/api/v1/vendor/stock/consume` | `{ reason, note?, reference?, lines[] }` + optional `Idempotency-Key` | `StockConsumeResultDto` (all-or-nothing) | Vendor mobile app (assistant "I made it") |
| POST/DELETE/PUT | `/api/v1/vendor/stock/{id}/photos[/{photoId}[/primary]]` | multipart `photo` + `clientPhotoId?` | `StockPhotoDto` / 204 (EP-31) | Vendor Portal, vendor mobile app |
| GET | `/api/v1/vendor/stock/photos/{photoId}/content` | — | private JPEG bytes | Vendor Portal (proxy), vendor mobile app |
| POST | `/api/v1/vendor/stock/photos/{photoId}/describe` | optional `Idempotency-Key` | `StockAppearanceSuggestionDto` (EP-31, AI) | Vendor Portal, vendor mobile app |
| POST | `/api/v1/vendor/stock/photo-suggestions` | multipart `photo` + `clientPhotoId?`, optional `Idempotency-Key` | `StockPhotoSuggestionDto` (pending photo + suggestion) | Vendor Portal, vendor mobile app |
| GET | `/api/v1/vendor/stock/ai-usage` | — | `StockAiUsageDto` | Vendor Portal, vendor mobile app |
| GET | `/api/v1/vendor-accounts/debug/token-claims` | — | Token claims | Dev tooling |

### 2d. Platform Admin Endpoints (Policy: `PlatformAdmin`)

| Method | Path | Request Body | Response | Counterpart |
|--------|------|-------------|----------|-------------|
| GET | `/api/v1/admin-accounts/me` | — | `AdminInfoResponse` | Admin Portal |
| POST | `/api/v1/admin/vendors` | `CreateVendorAccountRequest` | `VendorRegistrationResultDto` 201 | Admin Portal |
| GET | `/api/v1/admin/vendors/pending-installation` | — | `List<VendorPendingInstallationDto>` | Admin Portal |
| POST | `/api/v1/admin/vendors/{vendorId}/assign-technician` | `{ technicianId, scheduledDate, estimatedDuration, specialInstructions? }` | `TechnicianAssignmentResult` | Admin Portal |
| PATCH | `/api/v1/admin/vendors/{vendorId}/status` | `{ newStatus, reason, notifyVendor }` | 200 | Admin Portal |
| GET | `/api/v1/admin/analytics` | Query: `fromDate?, toDate?, includeVendorBreakdown, includeLocationAnalysis` | `PlatformAnalyticsDto` | Admin Portal |
| PUT | `/api/v1/admin/vendors/{id}/sales-mode` | `{ salesMode }` | `{ salesMode, assignedVaseCount, canSwitchToOnlineOnly }` / 409 `vendor-has-vases` (EP-39) | Admin Portal |
| GET | `/api/v1/admin/analytics/ai-florist` | Query: `from?, to?, step?, promptVersion?` | `AiFloristQualityDto` (aggregates only, EP-33) | Admin Portal |
| GET | `/api/v1/admin/analytics/ai-florist/feedback-export` | Query: `from?, to?` | `text/csv` of consented, scrubbed feedback (audit-logged, EP-33); EP-40 adds verdict rows/columns | Admin Portal |
| POST | `/api/v1/admin/analytics/ai-florist/recognition-verdicts/backfill` | — | `{ fromCorrections, fromConfirmedSessions, skipped, platformCells }` — one-off, idempotent (EP-40) | Operator (curl) |
| GET/POST/PUT/DELETE | `/api/v1/admin/atlas/flowers[/{id}[/varieties[/{varietyId}]]]` | `AtlasFlowerInput` / `AtlasVarietyInput` | `AtlasFlowerSummaryDto[]` / `AtlasFlowerDto` — flower atlas curation (EP-41, `API-CONTRACTS.md` §26) | Admin Portal |
| POST | `/api/v1/admin/atlas/seed`, `/import`, `/images/{imageId}/review`, `/flowers/{id}/images`, `/embeddings/backfill` | see §26 | seed list, licensed import (Wikimedia Commons, iNaturalist), review, own upload, fingerprint backfill (EP-41) | Admin Portal, Operator |
| POST | `/api/v1/admin/reports/vendor-onboarding` | `VendorOnboardingReportRequest` | `VendorOnboardingReportDto` | Admin Portal |
| POST | `/api/v1/smartvases` | `{ deviceId, latitude, longitude, locationName?, vendorId?, vendorType? }` | `VaseRegistrationResult` 201 | Admin Portal |
| GET | `/api/v1/smartvases` | — | `List<VaseDetailsDto>` | Admin Portal |
| GET | `/api/v1/smartvases/check-device/{deviceId}` | — | `bool` | Admin Portal |
| POST | `/api/v1/smartvases/{id}/migrate` | `{ targetVendorId, targetVendorType, reason }` | `VaseMigrationResult` 202 | Admin Portal |
| POST | `/api/v1/smartvases/{id}/ping` | — | `PingVaseResult` | Admin Portal |
| POST | `/api/v1/smartvases/{id}/restart` | — | `VaseCommandResult` | Admin Portal |
| POST | `/api/v1/smartvases/{id}/configure` | `{ brightness?, contrast?, sensorSensitivity?, privacyMode? }` | `VaseCommandResult` | Admin Portal |
| POST | `/api/v1/smartvases/{id}/unassign` | — | `VaseCommandResult` | Admin Portal |
| GET | `/api/v1/analytics/platform` | Query: `period, vendorType?, region?` | `PlatformInsightsDto` | Admin Portal |
| POST | `/api/mock-vase/{vaseId}/ping` | — | `PingResponse` | Admin Portal |

### 2e. Common Response Errors

| HTTP Status | Meaning | Common Error Strings |
|-------------|---------|---------------------|
| 400 | Validation / business rule error | `vendor-mismatch`, `price-changed`, `bouquet-not-available` |
| 401 | Unauthenticated | — |
| 403 | Wrong tenant / insufficient role | — |
| 404 | Resource not found | — |
| 409 | Conflict (idempotency / concurrent update) | `price-changed`, `vendor-mismatch` |
| 410 | Gone (bouquet sold/expired) | — |

---

## 3. SignalR Hub — BouquetHub

**Hub URL**: `/hubs/bouquet`
**Transport**: WebSocket with exponential backoff reconnect (0s, 2s, 5s, 10s, 30s)
**Counterpart repos**: `flower-shop-mobile`, portals (Admin/Vendor/Customer)

### 3a. Client → Server Methods

| Method | Auth Required | Parameters | Group Joined | Purpose |
|--------|---------------|-----------|-------------|---------|
| `JoinRadiusGroup` | No | `lat: double, lng: double, radiusKm: double` | `radius-{lat:0.00}-{lng:0.00}-{radiusKm}` | Subscribe to geo-radius bouquet events |
| `LeaveRadiusGroup` | No | `lat: double, lng: double, radiusKm: double` | — | Unsubscribe from radius group |
| `JoinOrderGroup` | Yes | `orderId: string` | `order-{orderId}` | Subscribe to order status updates |
| `LeaveOrderGroup` | Yes | `orderId: string` | — | Unsubscribe from order group |
| `JoinVendorGroup` | Internal check | `vendorId: string, vendorType: string` | `vendor-{vendorId}`, `vendortype-{vendorType}` | Vendor portal subscription |
| `JoinCustomerGroup` | No | `customerId: string` | `customer-{customerId}`, `customers` | Customer portal subscription |
| `JoinCustomersGroup` | No | — | `customers` | Anonymous web-map subscription; on join server also pushes current offline-vase `BouquetBecameUnavailable` state |
| `LeaveCustomersGroup` | No | — | — | Leave the flat customers group |
| `StartViewingBouquet` | No | `bouquetId: string` | — | Increment real-time view counter |
| `StopViewingBouquet` | No | `bouquetId: string` | — | Decrement real-time view counter |

### 3b. Server → Client Events

| Event Name | Target Group(s) | Payload Schema | Status | Trigger |
|------------|----------------|----------------|--------|---------|
| `BouquetBecameAvailable` | `customers` (all) | `{ bouquetId, vendorId, vendorType, status, price, currency, imageUrl, timestamp }` | Active | Bouquet created/released from cart |
| `BouquetBecameUnavailable` | `customers` (all) | `{ bouquetId, vendorType, reason, timestamp }` | Active | Sold / reserved / vase offline |
| `BouquetPriceChanged` | `customers`, `vendor-{vendorId}` | `{ bouquetId, price, currency, timestamp }` | Active | Vendor updates price |
| `BouquetViewCountChanged` | `customers` | `{ bouquetId, viewCount, timestamp }` | Active | View counter incremented/decremented |
| `BouquetStatusChanged` | `vendor-{vendorId}` | `{ bouquetId, status, vendorType, timestamp }` | Active | Bouquet status changes |
| `BouquetDeleted` | `vendor-{vendorId}`, `customers` | `{ bouquetId, reason, timestamp }` | Active | Vendor deletes bouquet |
| `OrderStatusChanged` | `order-{orderId}`, `customer-{customerId}`, `vendor-{vendorId}` | `{ orderId, status, label, timestamp, estimatedDeliveryAt? }` | Active | Order lifecycle transitions |
| `VaseStatusChanged` | `vendor-{vendorId}` | `{ vaseId, status, timestamp }` | Active | Vase heartbeat monitor detects change |
| `FreshnessUpdated` | `customers`, `vendor-{vendorId}` | `{ bouquetId, freshnessScore, freshnessLabel, timestamp }` | Active | MQTT sensor data → freshness calc (`RealTimeNotificationService.NotifyFreshnessUpdate`) |
| `DeliveryLocationUpdated` | `order-{orderId}` | `{ orderId, courierLatitude, courierLongitude, estimatedMinutesRemaining, timestamp }` | **Not implemented** (EP-07) | Courier location update |

**`status` values for `OrderStatusChanged`**: `created`, `paid`, `preparing`, `ready_for_pickup`, `in_delivery`, `delivered`, `cancelled`
**`reason` values for `BouquetBecameUnavailable`**: `sold`, `reserved`, `vase_offline`, `removed`
**`status` values for `VaseStatusChanged`**: `online`, `offline`, `error`

---

## 4. Domain / Integration Events (Event Bus)

**Current implementation**: `InMemoryEventBus` via MediatR (in-process)
**Planned implementation**: RabbitMQ — EP-09 (not implemented; `EventBus:Type=RabbitMQ` fails at startup)
**Counterpart repos**: None currently (all in-process); future: analytics service, notification worker

> When RabbitMQ is wired, these events will become inter-service contracts.

### 4a. Bouquet Events

| Event Class | Key Fields | Published By | Consumed By |
|-------------|-----------|-------------|-------------|
| `BouquetCatalogCreated` | `BouquetId, Name, VendorId, VendorType, CreatedAt` | `CreateBouquetCommandHandler` | `RealTimeNotificationService` |
| `BouquetPlacedInVase` | `BouquetId, VaseId, VendorId, VendorType, PlacedAt` | `Bouquet` aggregate | — |
| `BouquetPriceSet` | `BouquetId, Price (PriceInfo), SetBy, VendorType, Method (PricingMethod)` | `Bouquet` aggregate | `MqttService`, `RealTimeNotificationService` |
| `BouquetPriceUpdated` | `BouquetId, PreviousPrice, NewPrice, UpdatedBy, Method, Reason` | `UpdateBouquetCommandHandler` | `RealTimeNotificationService` |
| `BouquetBecameAvailable` | `BouquetId, VaseId?` (null for online listings, EP-39), `VendorId, VendorType` | `Bouquet` aggregate (vase bouquets and online listings) | `RealTimeNotificationService`, `SubscriptionMatchingService` |
| `BouquetReserved` | `BouquetId, CustomerId, ReservedAt, ReservationExpires` | `Bouquet` aggregate | `RealTimeNotificationService` |
| `BouquetSold` | `BouquetId, VaseId?` (null for online listings, EP-39), `VendorId, SoldTo, OrderId, Price, SoldAt` | `Bouquet` aggregate | `RealTimeNotificationService` |
| `BouquetRemovedFromVase` | `BouquetId, VaseId, VendorId, Reason (RemovalReason), RemovedBy, PreviousStatus, RemovedAt` | `DeleteBouquetCommandHandler` | `RealTimeNotificationService` |

### 4b. Order Events

| Event Class | Key Fields | Published By | Consumed By |
|-------------|-----------|-------------|-------------|
| `OrderCreatedEvent` | `OrderId, CustomerId, BouquetIds[], VendorId, VendorType, TotalAmount` | `Order` aggregate | `RealTimeNotificationService` |
| `OrderPaidEvent` | `OrderId, CustomerId, BouquetIds[], VendorId, PaymentIntentId` | `Order` aggregate | `RealTimeNotificationService` |
| `OrderCancelledEvent` | `OrderId, CustomerId, BouquetIds[], VendorId, PaymentIntentId, PreviousStatus` | `Order` aggregate | `RealTimeNotificationService` |
| `ReservationExpiredEvent` | `OrderId, CustomerId, BouquetIds[]` | `Order` aggregate | Bouquet release handler |

### 4c. SmartVase Events

| Event Class | Key Fields | Published By | Consumed By |
|-------------|-----------|-------------|-------------|
| `SmartVaseCreated` | `VaseId, DeviceId` | `SmartVase` aggregate | — |
| `VaseAssignedToVendor` | `VaseId, VendorId, VendorType, Configuration` | `SmartVase` aggregate | — |
| `VaseRegistrationCompletedByTechnician` | `VaseId, DeviceId, VendorId, VendorType, TechnicianId, InstallationLocation, CompletedAt, Configuration` | `SmartVase` aggregate | — |
| `VaseHealthStatusChanged` | `VaseId, NewStatus (VaseHealthStatus), PreviousStatus, HealthData` | `SmartVase` aggregate | `RealTimeNotificationService` |
| `VaseWentOffline` | `VaseId, VendorId, VendorType, OfflineAt` | `VaseHealthMonitoringService` | `RealTimeNotificationService` |
| `VaseMigrationInitiated` | `Migration (VaseMigration)` | `SmartVase` aggregate | — |
| `VaseMigrationCompleted` | `VaseId, PreviousAssignment, NewAssignment` | `SmartVase` aggregate | — |
| `VaseUnassignedFromVendor` | `VaseId, PreviousVendorId, PreviousVendorType, UnassignedAt` | `SmartVase` aggregate | — |

### 4d. Vendor Events

| Event Class | Key Fields | Published By | Consumed By |
|-------------|-----------|-------------|-------------|
| `FlowerShopRegistered` | `VendorId, ShopProfile, Location` | `FlowerShop` aggregate | `VendorEventHandlers` → `INotificationServiceClient` |
| `FreelanceFloristRegistered` | `VendorId, FloristProfile, ResidentialLocation` | `FreelanceFlorist` aggregate | `VendorEventHandlers` → `INotificationServiceClient` |
| `VendorRegistered` | `VendorId, VendorType, RegisteredAt` | `VendorEvents` | — |
| `VendorProfileCompleted` | `VendorId, VendorType, CompletedAt` | `VendorEvents` | — |
| `ShopProfileUpdated` | `VendorId, NewProfile, PreviousProfile, UpdatedBy` | `FlowerShop` aggregate | — |
| `ShopLocationUpdated` | `VendorId, NewLocation` | `FlowerShop` aggregate | — |
| `FloristPrivacySettingsUpdated` | `VendorId, NewSettings, PreviousSettings, UpdatedBy` | `FreelanceFlorist` aggregate | — |
| `FloristGrowthStageAdvanced` | `VendorId, Milestone, PreviousStage, RecordedBy` | `FreelanceFlorist` aggregate | `VendorEventHandlers` → `INotificationServiceClient` |
| `VendorTypeChanged` | `VendorId, FromType, ToType` | Application handlers | `INotificationServiceClient` |
| `VaseAssignedToShop` | `VendorId, VaseId, VaseConfiguration` | `FlowerShop` aggregate | — |
| `VaseAssignedToFlorist` | `VendorId, VaseId, HomeVaseConfiguration, BusinessGrowthStage` | `FreelanceFlorist` aggregate | — |

### 4e. Employee Events

| Event Class | Key Fields | Published By | Consumed By |
|-------------|-----------|-------------|-------------|
| `EmployeeInvitationCreated` | `EmployeeId, VendorId, FullName, Email, Role, InvitationToken, CreatedBy` | `Employee` aggregate | Email service |
| `EmployeeInvitationAccepted` | `EmployeeId, VendorId, AcceptedAt` | `Employee` aggregate | Keycloak user creation |
| `EmployeeRoleUpdated` | `EmployeeId, VendorId, PreviousRole, NewRole, UpdatedBy` | `Employee` aggregate | — |
| `EmployeePermissionGranted` | `EmployeeId, VendorId, Permission, GrantedBy` | `Employee` aggregate | — |
| `EmployeePermissionRevoked` | `EmployeeId, VendorId, Permission, RevokedBy` | `Employee` aggregate | — |
| `EmployeeDeactivated` | `EmployeeId, VendorId, DeactivatedBy, Reason` | `Employee` aggregate | Keycloak user disable |

---

## 5. Keycloak / JWT Claims & Auth Policies

**Counterpart repos**: `keycloak-config` (realm configuration), `flower-shop-mobile`, portals

### 5a. JWT Claims Extracted by This Repo

| Claim Name | Type | Source | Description | Required By |
|------------|------|--------|-------------|------------|
| `sub` | `string` (UUID) | Keycloak standard | Keycloak user ID | `TenantResolutionMiddleware` |
| `ClaimTypes.NameIdentifier` | `string` | Standard JWT | Maps to `sub` | Auth handlers |
| `vendor_id` | `string` (GUID) | Keycloak custom | Vendor's ID | `TenantResolutionMiddleware` → `VendorContext` |
| `vendor_type` | `string` | Keycloak custom | `TraditionalShop` or `FreelanceFlorist` | `TenantResolutionMiddleware` → `VendorContext` |
| `customer_id` | `string` (GUID) | Keycloak custom | Customer's ID | `TenantResolutionMiddleware` → `CustomerContext` |
| `ClaimTypes.Role` | `string` | Keycloak standard | User role string | Auth policy checks |
| `realm_access.roles[]` | `string[]` | Keycloak JWT | Array of realm roles | Role-based policy evaluation |
| `docs_roles[]` | `string[]` | Keycloak client-role mapper on `flowershop-docs` | `docs-developer` / `docs-vendor` / `docs-customer` (composites of `PlatformAdmin`+group `developers` / `VendorAdmin` / `User`) | FlowerShop.DocsPortal sign-in (EP-38) |
| `azp` / `client_id` | `string` | Keycloak standard | Client that obtained the token | `DocsPortalClient` policy (must equal `Docs:ClientId`) |
| `groups[]` | `string[]` | Keycloak group mapper (optional) | Group paths; `/developers` grants docs developer. Without it the API reads groups through the admin API | Docs handoff (EP-38) |

### 5b. Auth Policies

| Policy Name | Required Claims / Roles | Used On |
|-------------|------------------------|---------|
| `CustomerAccess` | Role must be `User` + valid `customer_id` claim | All `/api/v1/cart`, `/api/v1/orders`, `/api/v1/profile`, `/api/v1/favourites`, `/api/v1/subscriptions`, `/api/v1/notifications` |
| `VendorAccess` | Role must be `VendorAdmin` + valid `vendor_id` + `vendor_type` claims | All vendor bouquet / vase / analytics / employee endpoints |
| `PlatformAdmin` | Role must be `PlatformAdmin` | All `/api/v1/admin/*`, vase registration, platform analytics |
| `ShopAdmin` | Role must be `ShopAdmin` (employee sub-role) | Employee management write operations |
| `DocsPortalClient` | `azp` or `client_id` = `Docs:ClientId` (`flowershop-docs`, client credentials) | `POST /api/v1/docs/handoff/redeem`, `POST /api/v1/docs/roles` (EP-38) |

### 5c. Keycloak Server Configuration (Dev)

| Key | Value |
|-----|-------|
| Authority URL | `http://keycloak-dev:8080/realms/flowershop` (container) / `http://localhost:8090/realms/flowershop` (host) |
| Client ID | `flowershop-api` |
| Client Secret | `flowershop-secret` |
| Admin Realm | `master` |
| Admin Username | `admin` |
| Admin Password | `admin123` |
| Admin Client ID | `admin-cli` |
| Realm Name | `flowershop` |

> **External repos must configure the same realm name, client ID, and custom claim mappers** (`vendor_id`, `vendor_type`, `customer_id`) in Keycloak.

### 5d. Google SSO / Just-In-Time Customer Provisioning

Customers may sign in with Google (configured as an Identity Provider in the `flowershop` realm; committed export `docker/keycloak/flowershop-realm.json`). On a first Google login the Keycloak user exists but the platform DB does not, and the token carries **no `customer_id` claim**. `TenantResolutionMiddleware` detects this case (token has `email`, no `vendor_id`, no existing `customer_id`, not an admin) and calls `CustomerProvisioningService.EnsureCustomerProvisionedAsync`, which:

1. Reads `email`, `given_name`, `family_name` from the validated token.
2. Creates a local `Customer` + `User` (idempotently — reuses an existing user matched by Keycloak id or email).
3. Writes `customer_id` back to Keycloak via `KeycloakAdminClient.SetUserAttributesAsync`, and assigns the `User` realm role — so subsequent tokens carry the claim natively.
4. **Injects the `customer_id` claim into the current request principal**, so the very first request is authorized without requiring a re-login.

Consequently: `customer_id` may be **absent on the first Google token** and is populated mid-request; Google emails are treated as verified (no separate email-verification step); a Google login auto-links to any existing account with the same verified email.


### 5e. Docs portal client (EP-38)

Confidential client **`flowershop-docs`** (authorization code + PKCE S256, service account for the back-channel). Redirect `https://docs.findmyflowers.pl/signin-oidc`, post-logout `https://docs.findmyflowers.pl/*`. Client roles `docs-developer`, `docs-vendor`, `docs-customer` are emitted as the `docs_roles` claim (id, access and userinfo tokens); an audience mapper adds `flowershop-api` so its client-credentials token is accepted by the API. Composites: realm `User` → `docs-customer`, `VendorAdmin` → `docs-vendor`, `PlatformAdmin` → `docs-developer`; group `developers` → `docs-developer`. Dev: `docker/keycloak/flowershop-realm.json` (secret `dev-docs-portal-secret-change-me`); prod: `scripts/keycloak/setup-docs-client.sh` (idempotent). Customer creation paths verify the `User` realm role (`EnsureRealmRoleAsync`); a nightly job reports customers without it.
---

## 6. Shared Config Keys & Environment Variables

**Counterpart repos**: `keycloak-config`, `flower-shop-firmware`, deployment infrastructure

### 6a. .NET Configuration Keys (IConfiguration paths)

| Config Key | Type | Description | Default (dev) |
|------------|------|-------------|--------------|
| `ConnectionStrings:DefaultConnection` | string | Primary PostgreSQL | `Host=postgres-dev;Database=flowershop_iot_dev;Username=postgres;Password=dev123;` |
| `ConnectionStrings:ReadConnection` | string | Read replica PostgreSQL | `Host=postgres-dev;Database=flowershop_iot_read_dev;...` |
| `ConnectionStrings:Redis` | string | Redis | `redis-dev:6379` |
| `JWT:SecretKey` | string | HMAC signing key (≥32 chars) | Dev key only |
| `JWT:Issuer` | string | JWT issuer claim | `FlowerShopIoTPlatformDev` |
| `JWT:Audience` | string | JWT audience claim | `FlowerShopIoTUsersDev` |
| `JWT:ExpirationHours` | int | Token lifetime | `8760` (dev), `24` (prod) |
| `Authentication:UseKeycloak` | bool | Enable Keycloak vs dev-token flow | `true` |
| `Authentication:Keycloak:Authority` | string | Keycloak OIDC authority URL | See §5c |
| `Authentication:Keycloak:ClientId` | string | API client ID | `flowershop-api` |
| `Authentication:Keycloak:ClientSecret` | string | API client secret | `flowershop-secret` |
| `Authentication:Keycloak:AdminRealm` | string | Keycloak admin realm | `master` |
| `Authentication:Keycloak:AdminUsername` | string | Keycloak admin user | `admin` |
| `Authentication:Keycloak:AdminPassword` | string | Keycloak admin password | `admin123` |
| `Authentication:Keycloak:AdminClientId` | string | Admin CLI client | `admin-cli` |
| `MQTT:BrokerHost` | string | MQTT broker hostname | `mqtt-dev` (container) / `localhost` (host) |
| `MQTT:BrokerPort` | int | MQTT broker port | `1883` |
| `MQTT:Username` | string | MQTT auth username | `flowershop` |
| `MQTT:Password` | string | MQTT auth password | `dev123` |
| `EventBus:Type` | string | Optional; `InMemory` is the only bus — any other value stops startup | `InMemory` (unset) |
| `Features:EnableRealTimeUpdates` | bool | Enable SignalR hub | `true` |
| `Features:EnableAnalytics` | bool | Enable analytics handlers | `true` |
| `Features:EnableVaseHealthMonitoring` | bool | Enable background heartbeat monitor | `false` (dev) |
| `Features:MaxVasesPerVendor` | int | Business rule limit | `10` |
| `Features:MaxBouquetsPerVendor` | int | Business rule limit | `50` |
| `Cors:AllowedOrigins` | string[] | CORS origins | `["*"]` (dev) |
| `BouquetDesign:Enabled` / `PreviewsEnabled` | bool | EP-28 kill switches for new paid AI work (existing recipes stay readable) | `true` |
| `BouquetDesign:PilotVendorIds` | Guid[] | Pilot allow-list; empty = all vendors | `[]` |
| `BouquetDesign:{Recognition,Composition,Preview}:Provider` | string | `Fake`/`OpenAI`; `Deterministic`/`OpenAI`; `Placeholder`/`Bfl`/`OpenAI` | free defaults |
| `BouquetDesign:OpenAI:ApiKey`, `BouquetDesign:Bfl:ApiKey` | secret | Provider keys (Fly secrets only) | empty |
| `BouquetDesign:Storage:*` | object | `Local` (dev folder) or `R2` private bucket `flowershop-design-private` | `Local` |
| `BouquetDesign:Limits:*`, `Worker:*`, `Retention:*` | object | Quotas, $ cap, worker concurrency/lease, retention days | see `BouquetDesignOptions` |
| `BouquetDesign:{Recognition,Composition,Preview}:PromptVersion` | string | Active prompt version (`inventory-v1/v2`, `recipe-v1/v2`, `preview-v1/v2`; EP-33). Empty = newest | `*-v1` (appsettings) |
| `BouquetDesign:{Step}:CandidatePromptVersion` / `CandidatePercent` | string / int | Candidate version for a sticky, vendor-hashed share (0–100) | `*-v2` / `0` |
| `BouquetDesign:Analysis:Provider` / `Model` / `EstimatedUsd` | string / string / decimal | Bouquet analysis for "reconstruct from photos" (EP-35): `Palette` (free) or `OpenAI` | `Palette` / `gpt-4.1-mini` / `0.010` |
| `BouquetDesign:Analysis:ModelAssistedMatching` / `UseInspirationAsPreviewReference` | bool | Model picks among code-ranked stock candidates; bouquet photo as preview reference (EP-35) | `false` / `false` |
| `BouquetDesign:Assessment:MonthlyLimit` | int | Cost assessments per vendor per month (EP-36) | `30` |
| `BouquetDesign:LegacyPhotosMode` | bool | Offer `inputMode: "Photos"` for new wizard sessions (EP-34); switch off after vendor app EP-M14 | `true` |
| `BouquetDesign:Composition:RepairEnabled` | bool | One validator-guided repair call per composition (global $ cap, not vendor quota) | `true` |
| `BouquetDesign:Analysis:CandidatePromptVersion` | string | EP-41 `bouquet-analysis-v2` (item boxes for crops) as a candidate at `CandidatePercent` | `bouquet-analysis-v2` / `0` |
| `BouquetDesign:Embedding:*` | object | Image fingerprints (EP-41 Phase 2): `Provider` (`None`\|`Onnx`\|`Http`), `Model`, `Dimensions`, `StockWeight`, `DisagreeMargin`, `MinSimilarity`, `StockSuggestSimilarity`, `SimilarityFloor`, `SimilarityCeiling`, `LearnFromVerdicts`, `TimeoutSeconds`, `Onnx:{ModelPath,ImageSize,Mean,Std}`, `Http:{Endpoint,ApiKey}` | `None`, `clip-vit-b32`, `512`, `0.3`, `0.05`, `0.6`, `0.85`, `0.6`, `0.95`, `false`, `30` |
| `Atlas:AllowedLicences` / `Atlas:Import:*` | list / object | Flower atlas (EP-41): licence families (empty = CC0, CC-BY, CC-BY-SA; NC/ND always refused); import `CandidatesPerSource`, `MaxFlowersPerRun`, `EnoughImages`, `MaxLongEdge`, `MaxImageBytes`, `MaxDownloadBytes`, `UserAgent`, `CommonsApiUrl`, `INaturalistApiUrl` | `[]`, `2`, `10`, `4`, `1024`, `400000`, `15000000` |
| `BouquetDesign:Confidence:*` | object | Recognition confidence from florists' verdicts (EP-40): `ScoresEnabled` (rollout switch, scores off until the pilot check), `ImplicitWeight`, `HalfLifeDays`, `PriorStrength`, `ShopStrength`, `MinChecks`, `MinVendorsPerCell`, `MaxPerVendorPerCell`, `CapWindowDays`, `HighBand`, `MediumBand`, `ColdStartFactor`, `InheritFrom`, `InheritStrength`, `BurstLines`, `BurstSeconds`, `RefreshMinutes` | `false` (appsettings; code default `true`), `0.25`, `120`, `10`, `10`, `5`, `3`, `30`, `30`, `0.85`, `0.6`, `0.8`, `{}`, `0.5`, `5`, `2`, `60` |
| `OnlineListings:MaxListingDays` / `CheckIntervalMinutes` | int | Online listings (EP-39) expire after this many days at most (or sooner when the declared freshness runs out); expiry pass interval | `3` / `15` |
| `Marketplace:Enabled` | bool | EP-37 kill switch for new requests and offers (existing orders keep working) | `true` |
| `Marketplace:MaxOpenRequestsPerCustomer` / `MaxRequestsPer7Days` | int | Customer request limits (429 `request-limit`) | `3` / `10` |
| `Marketplace:RequireVerifiedEmail` | bool | Verified email needed to create requests | `true` |
| `Marketplace:TimeZone` / `CustomerAppUrl` / `VendorPortalUrl` | string | Quiet hours / email links | `Europe/Warsaw` / empty |
| `Marketplace:MonitorEnabled` | bool | Background monitor (expiry, payment lapse, time-change auto-accept, SLA alerts) | `true` |
| `Docs:PublicUrl` | string | Docs host the handoff URL points at (API, Vendor Portal, Customer App) (EP-38) | `https://docs.findmyflowers.pl` |
| `Docs:ClientId` / `HandoffCodeSeconds` / `HandoffSessionHours` | string / int / int | Docs client allowed to redeem; code TTL; handoff session cap (API) | `flowershop-docs` / `60` / `8` |
| `Docs:ReconciliationEnabled` / `ReconciliationHourUtc` | bool / int | Nightly report of customers lacking realm role `User` (API, Keycloak only) | `true` / `3` |
| `Docs:SitePath`, `Docs:ApiBaseUrl`, `Docs:Keycloak:{Authority,ClientId,ClientSecret}` | string | FlowerShop.DocsPortal: built `_site`, API back-channel, OIDC client | `_site`, `http://localhost:8080`, local Keycloak |
| `Docs:AuthDisabled` | bool | DocsPortal: every request reads as developer — **Development only**, logged and ignored elsewhere | `false` |
| `Docs:RevalidateMinutes` / `SlidingHours` / `AbsoluteHours` | int | DocsPortal session: role re-read, sliding and absolute lifetime | `15` / `8` / `24` |

### 6b. Docker Compose Environment Variables

| Variable | Service | Value (dev) |
|----------|---------|-------------|
| `POSTGRES_DB` | `postgres-dev` | `flowershop_iot_dev` |
| `POSTGRES_USER` | `postgres-dev` | `postgres` |
| `POSTGRES_PASSWORD` | `postgres-dev` | `dev123` |
| `RABBITMQ_DEFAULT_USER` | `rabbitmq-dev` | `flowershop` |
| `RABBITMQ_DEFAULT_PASS` | `rabbitmq-dev` | `dev123` |
| `RABBITMQ_DEFAULT_VHOST` | `rabbitmq-dev` | `/` |
| `KEYCLOAK_ADMIN` | `keycloak-dev` | `admin` |
| `KEYCLOAK_ADMIN_PASSWORD` | `keycloak-dev` | `admin123` |
| `KC_DB_USERNAME` | `keycloak-dev` | `postgres` |
| `KC_DB_PASSWORD` | `keycloak-dev` | `dev123` |
| `MQTT__BrokerHost` | `api-dev` | `mqtt-dev` |
| `MQTT__BrokerPort` | `api-dev` | `1883` |
| `MQTT__Username` | `api-dev` | `flowershop` |
| `MQTT__Password` | `api-dev` | `dev123` |
| `JWT__SecretKey` | `api-dev` | Dev key |
| `JWT__Issuer` | `api-dev` | `FlowerShopIoTPlatformDev` |
| `JWT__Audience` | `api-dev` | `FlowerShopIoTUsersDev` |
| `Authentication__UseKeycloak` | `api-dev` | `true` |
| `Authentication__Keycloak__Authority` | `api-dev` | `http://keycloak-dev:8080/realms/flowershop` |
| `Authentication__Keycloak__ClientId` | `api-dev` | `flowershop-api` |

---

## 7. Firebase FCM Push Notifications

**Interface**: `INotificationServiceClient`
**Implementation status**: **STUB** — pending EP-06
**Counterpart repo**: `flower-shop-mobile` (receives FCM tokens and push messages)

| Method | Parameters | Purpose | Status |
|--------|-----------|---------|--------|
| `SendVendorWelcomeNotificationAsync` | `vendorId, vendorType, vendorName, cancellationToken` | Welcome email/push on vendor registration | Stub |
| `SendGrowthStageAdvancementNotificationAsync` | `vendorId, newStage, previousStage, cancellationToken` | Notify florist of growth milestone | Stub |
| `SendVendorTypeChangeNotificationAsync` | `vendorId, newType, previousType, cancellationToken` | Notify on vendor type transition | Stub |
| `SendComplianceAlertAsync` | `vendorId, alertType, message, cancellationToken` | Platform compliance alert | Stub |
| `SendMaintenanceNotificationAsync` | `message, targetVendors?, cancellationToken` | Broadcast maintenance notice | Stub |
| `SendBouquetAvailabilityNotificationAsync` | `notificationData: object, cancellationToken` | Push to subscribed customers | Stub |

**Device registration endpoint**: `POST /api/v1/notifications/devices` — accepts `{ deviceToken, platform, appVersion? }`

---

## 8. Portal API Clients (Inbound REST)

The three portals (served from this repo) consume the REST API above. This section records which endpoints each portal calls, for dependency tracing.

### Admin Portal (`FlowerShop.AdminPortal`)

| Endpoint Called | Purpose |
|----------------|---------|
| `POST /api/v1/vendor-accounts/register-with-admin` | Create vendor + admin user |
| `GET /api/v1/admin/vendors/pending-installation` | List pending vendors |
| `GET /api/v1/vendors` | All vendors list |
| `GET /api/v1/vendors/{id}` | Vendor details |
| `PUT /api/v1/vendors/{id}/location` | Update vendor location |
| `POST /api/v1/smartvases` | Register new vase |
| `GET /api/v1/smartvases` | All vases list |
| `GET /api/v1/smartvases/{id}` | Vase details |
| `GET /api/v1/smartvases/check-device/{deviceId}` | Device ID check |
| `POST /api/v1/smartvases/{id}/ping` | Ping vase via MQTT |
| `POST /api/v1/smartvases/{id}/restart` | Restart vase |
| `POST /api/v1/smartvases/{id}/configure` | Configure vase |
| `POST /api/v1/smartvases/{id}/unassign` | Unassign vase |
| `POST /api/mock-vase/{id}/bouquet-image` | Upload bouquet image (mock endpoint) |

### Vendor Portal (`FlowerShop.VendorPortal`)

| Endpoint Called | Purpose | Known Issue |
|----------------|---------|-------------|
| `GET /api/v1/vendors/{id}` | Vendor profile | VendorId read from session, not JWT |
| `GET /api/v1/smartvases/my-vases` | Vendor's vases | — |
| `GET /api/v1/smartvases/{id}` | Vase details | — |
| `POST /api/v1/bouquets` | Create bouquet | — |
| `PUT /api/v1/bouquets/{id}` | Update bouquet | — |
| `GET /api/v1/bouquets/{id}` | Get bouquet | — |
| `GET /api/v1/vendors/{id}/bouquets` | List vendor bouquets | — |
| `DELETE /api/v1/bouquets/{id}` | Delete bouquet | — |
| `PUT /api/v1/bouquets/{id}/price` | Set price | — |
| `POST /api/mock-vase/{id}/bouquet-image` | Upload photo | Hardcoded `http://localhost:8080` URL |

### Customer App (`FlowerShop.CustomerApp`)

| Endpoint Called | Purpose | Known Issue |
|----------------|---------|-------------|
| `GET /api/v1/bouquets/available` | Browse bouquets | — |
| `GET /api/v1/bouquets/{id}` | Bouquet detail | — |
| `POST /api/v1/cart/{customerId}/items/{bouquetId}` | Add to cart | Old cart path — mismatches current API |
| `DELETE /api/v1/cart/{customerId}/items/{bouquetId}` | Remove from cart | Old cart path |
| `GET /api/v1/cart/{customerId}/items` | Get cart | Old cart path |
| `DELETE /api/v1/cart/{customerId}` | Clear cart | Old cart path |
| `GET /api/v1/cart/{customerId}/count` | Cart count | Old cart path |

> **Note**: Customer App cart calls use legacy path format `cart/{customerId}/...` which does not match the current API controller route `cart/...` (tenant from JWT). This is a known EP-13 gap.

---

## Known Stubs & Gaps (Contract Risk)

| Gap | Risk Level | Epic |
|-----|-----------|------|
| No RabbitMQ bus — `EventBus:Type=RabbitMQ` fails at startup (the stub that dropped events was removed) | Low | EP-09 S-09-06 |
| `INotificationServiceClient` is a no-op stub — no FCM pushes sent | High | EP-06 |
| `AnalyticsController` uses `VendorId.New()` — all analytics return wrong tenant data | High | EP-10 T-10-001 |
| `CacheService` uses `IMemoryCache` not Redis | Medium | EP-09 T-09-003 |
| `IStripeService` not wired — `CreateOrderCommandHandler` never calls Stripe | High | EP-04 |
| `DeliveryLocationUpdated` SignalR event in API contract but not implemented | Low | EP-07 |
| MQTT config sending disabled (`smartvase/+/+/config`) — firmware bug | Low | firmware fix needed |
| Customer App portal cart path mismatch vs current API | Medium | EP-13 |
| DB migration not run: Cart, CartItem, OrderLines, Subscription, Favourite, Notification, SavedAddress tables missing | Critical | EP-01 |
