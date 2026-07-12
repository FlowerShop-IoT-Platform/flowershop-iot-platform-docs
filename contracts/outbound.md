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

### 2b. Customer Endpoints (Policy: `CustomerAccess`)

| Method | Path | Request Body | Response | Counterpart |
|--------|------|-------------|----------|-------------|
| GET | `/api/v1/cart` | — | `CartDto` | Mobile App |
| POST | `/api/v1/cart/items` | `{ bouquetId }` | `CartDto` | Mobile App |
| DELETE | `/api/v1/cart/items/{bouquetId}` | — | 204 | Mobile App |
| DELETE | `/api/v1/cart` | — | 204 | Mobile App |
| POST | `/api/v1/orders/preview` | `{ bouquetIds[], deliveryMethod, street?, city?, postalCode?, notes? }` | `OrderPreviewDto` | Mobile App |
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
| DELETE | `/api/v1/bouquets/{id}` | — | 204 | Vendor Portal |
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
**Planned implementation**: `RabbitMQEventBus` — EP-09 (currently stub, events silently dropped)
**RabbitMQ connection**: `amqp://flowershop:dev123@localhost:5672/` (dev)
**Counterpart repos**: None currently (all in-process); future: analytics service, notification worker

> When RabbitMQ is wired, these events will become inter-service contracts.

### 4a. Bouquet Events

| Event Class | Key Fields | Published By | Consumed By |
|-------------|-----------|-------------|-------------|
| `BouquetCatalogCreated` | `BouquetId, Name, VendorId, VendorType, CreatedAt` | `CreateBouquetCommandHandler` | `RealTimeNotificationService` |
| `BouquetPlacedInVase` | `BouquetId, VaseId, VendorId, VendorType, PlacedAt` | `Bouquet` aggregate | — |
| `BouquetPriceSet` | `BouquetId, Price (PriceInfo), SetBy, VendorType, Method (PricingMethod)` | `Bouquet` aggregate | `MqttService`, `RealTimeNotificationService` |
| `BouquetPriceUpdated` | `BouquetId, PreviousPrice, NewPrice, UpdatedBy, Method, Reason` | `UpdateBouquetCommandHandler` | `RealTimeNotificationService` |
| `BouquetBecameAvailable` | `BouquetId, VaseId, VendorId, VendorType` | `Bouquet` aggregate | `RealTimeNotificationService` |
| `BouquetReserved` | `BouquetId, CustomerId, ReservedAt, ReservationExpires` | `Bouquet` aggregate | `RealTimeNotificationService` |
| `BouquetSold` | `BouquetId, VaseId, VendorId, SoldTo, OrderId, Price, SoldAt` | `Bouquet` aggregate | `RealTimeNotificationService` |
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

### 5b. Auth Policies

| Policy Name | Required Claims / Roles | Used On |
|-------------|------------------------|---------|
| `CustomerAccess` | Role must be `User` + valid `customer_id` claim | All `/api/v1/cart`, `/api/v1/orders`, `/api/v1/profile`, `/api/v1/favourites`, `/api/v1/subscriptions`, `/api/v1/notifications` |
| `VendorAccess` | Role must be `VendorAdmin` + valid `vendor_id` + `vendor_type` claims | All vendor bouquet / vase / analytics / employee endpoints |
| `PlatformAdmin` | Role must be `PlatformAdmin` | All `/api/v1/admin/*`, vase registration, platform analytics |
| `ShopAdmin` | Role must be `ShopAdmin` (employee sub-role) | Employee management write operations |

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
| `EventBus:Type` | string | `InMemory` or `RabbitMQ` | `InMemory` |
| `EventBus:RabbitMQ:ConnectionString` | string | RabbitMQ AMQP URL | `amqp://flowershop:dev123@localhost:5672/` |
| `Features:EnableRealTimeUpdates` | bool | Enable SignalR hub | `true` |
| `Features:EnableAnalytics` | bool | Enable analytics handlers | `true` |
| `Features:EnableVaseHealthMonitoring` | bool | Enable background heartbeat monitor | `false` (dev) |
| `Features:MaxVasesPerVendor` | int | Business rule limit | `10` |
| `Features:MaxBouquetsPerVendor` | int | Business rule limit | `50` |
| `Cors:AllowedOrigins` | string[] | CORS origins | `["*"]` (dev) |

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
| `RabbitMQEventBus.PublishAsync` returns `Task.CompletedTask` — all events silently dropped when RabbitMQ is configured | High | EP-09 S-09-06 |
| `INotificationServiceClient` is a no-op stub — no FCM pushes sent | High | EP-06 |
| `AnalyticsController` uses `VendorId.New()` — all analytics return wrong tenant data | High | EP-10 T-10-001 |
| `CacheService` uses `IMemoryCache` not Redis | Medium | EP-09 T-09-003 |
| `IStripeService` not wired — `CreateOrderCommandHandler` never calls Stripe | High | EP-04 |
| `DeliveryLocationUpdated` SignalR event in API contract but not implemented | Low | EP-07 |
| MQTT config sending disabled (`smartvase/+/+/config`) — firmware bug | Low | firmware fix needed |
| Customer App portal cart path mismatch vs current API | Medium | EP-13 |
| DB migration not run: Cart, CartItem, OrderLines, Subscription, Favourite, Notification, SavedAddress tables missing | Critical | EP-01 |
