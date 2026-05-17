# FlowerShop IoT Platform — Epic Features Guide

> What each epic delivers, why it matters, and how to use the new functionality via the API and portal UIs.

**Last updated**: 2026-04-06

---

## EP-01 · Database Foundation & Migrations

### Value

Establishes the schema for every customer-facing feature that follows. Without these tables, carts, orders, subscriptions, favourites, notifications, and saved addresses cannot be persisted.

### What was delivered

| Table | Purpose |
|-------|---------|
| **Carts / CartItems** | DB-backed shopping cart that persists across devices |
| **Orders / OrderLines** | Multi-line orders with delivery info and timeline |
| **Subscriptions** | Customer notification preferences (flower type, price, radius) |
| **Favourites** | Saved bouquets for quick access |
| **Notifications** | In-app notification feed |
| **DeviceRegistrations** | Push notification token storage (iOS / Android) |
| **SavedAddresses** | Customer delivery addresses (max 10) |
| **Bouquet.QrShortCode** | 8-char alphanumeric code for QR-based bouquet lookup |
| **Vendor OpeningHours / Description** | Extended vendor profile data |

### How to verify

```bash
# Check all tables exist in dev database
psql -h localhost -U postgres -d flowershop_iot_dev \
  -c "\dt" | grep -E "carts|cart_items|orders|order_lines|subscriptions|favourites|notifications|device_registrations|saved_addresses"
```

---

## EP-02 · Customer Authentication & Registration

### Value

Enables customers to create accounts, receive JWTs with a `customer_id` claim, and access protected endpoints (cart, orders, profile, favourites).

### API usage

**Register a new customer:**
```http
POST /api/v1/customers/register
Content-Type: application/json

{
  "username": "jane",
  "email": "jane@example.com",
  "password": "SecurePass123!",
  "customerName": "Jane Doe",
  "preferredVendorType": "FlowerShop"
}
```
Returns a JWT that the mobile app or web client stores and sends as `Authorization: Bearer <token>` on every subsequent request.

**Dev-mode token (no Keycloak):**
```http
POST /api/v1/auth/dev-token
{ "username": "testcustomer", "role": "User" }
```

---

## EP-03 · Bouquet Discovery, QR & Vendor Profile

### Value

Powers the core customer experience — finding nearby bouquets on a map, scanning QR codes on physical vases, and seeing whether a vendor is currently open.

### API usage

**Browse nearby bouquets (geo-spatial):**
```http
GET /api/v1/bouquets/nearby?lat=52.23&lng=21.01&radiusKm=5&sortBy=distance&page=1&pageSize=20
```
Returns bouquets within 5 km, sorted by distance. Each item includes `distanceKm`, `isFavourite` (if authenticated), `freshnessScore`, and `price`. Freelance florist coordinates are fuzzed for privacy.

**Apply filters:**
```http
GET /api/v1/bouquets/nearby?lat=52.23&lng=21.01&radiusKm=10&vendorType=FlowerShop&flowerType=roses&maxPrice=80&search=birthday
```

**Scan a QR code:**
```http
GET /api/v1/bouquets/qr/A7xK2m9P
```
Returns the full bouquet detail. Returns `410 Gone` if the bouquet has been sold or expired.

**View vendor profile:**
```http
GET /api/v1/vendors/{vendorId}/public
```
Response includes `isCurrentlyOpen` (computed from the vendor's structured opening hours), `openingHours`, `description`, and `activeBouquetCount`.

### Customer App UI

The **Home page** (`http://localhost:5003/`) displays a Leaflet.js interactive map centered on Warsaw. Markers appear for nearby bouquets; clicking one shows name, price, and a link to the detail page.

---

## EP-04 · Order Lifecycle & Stripe

### Value

Completes the purchase flow end-to-end: cart → preview → order → Stripe payment → delivery. Adds automatic reservation expiry, idempotent order creation, refunds on cancellation, and delivery QR codes.

### API usage — full purchase flow

**Step 1 — Add to cart:**
```http
POST /api/v1/cart/items
Authorization: Bearer <customer-token>
{ "bouquetId": "..." }
```
Returns `409 Conflict` with `vendor-mismatch` if the bouquet belongs to a different vendor than existing cart items.

**Step 2 — Preview order:**
```http
POST /api/v1/orders/preview
{
  "bouquetIds": ["id1", "id2"],
  "deliveryMethod": "delivery",
  "street": "ul. Marszałkowska 1",
  "city": "Warszawa",
  "postalCode": "00-001"
}
```
Returns `linesSubtotal`, `deliveryFee` (15.00 PLN default for delivery), and `totalAmount`.

**Step 3 — Create order:**
```http
POST /api/v1/orders
Idempotency-Key: unique-client-key-123
{
  "bouquetIds": ["id1", "id2"],
  "deliveryMethod": "delivery",
  "expectedTotalAmount": 145.00,
  "street": "ul. Marszałkowska 1",
  "city": "Warszawa",
  "postalCode": "00-001"
}
```
Returns `orderId`, `paymentClientSecret`, `stripeEphemeralKey`, and `stripeCustomerId`. Bouquets are reserved for 15 minutes.

**Step 4 — Confirm payment (after Stripe client-side):**
```http
POST /api/v1/orders/{orderId}/confirm-payment
{ "paymentIntentId": "pi_xxx", "stripeCustomerId": "cus_xxx" }
```

**Step 5 — Cancel (with automatic refund if paid):**
```http
POST /api/v1/orders/{orderId}/cancel
{ "reason": "Changed my mind" }
```
If the order was already paid, a Stripe refund is automatically initiated.

### Stripe webhook

The platform listens at `POST /api/stripe/webhook` for:
- `payment_intent.succeeded` → confirms payment automatically
- `payment_intent.payment_failed` → cancels the order

### Idempotency

Any `POST /api/v1/orders` with the same `Idempotency-Key` header within 24 hours returns the cached response instead of creating a duplicate order.

### Reservation expiry

A background service runs every 30 seconds. Orders stuck in `Created` status past their 15-minute reservation window are automatically cancelled, and bouquets become available again (with a SignalR push to connected clients).

### Delivery QR code

Once an order reaches `Paid` status, `GET /api/v1/orders/{id}` includes a `deliveryQrCode` field — a base64 PNG data URI the customer can show at pickup/delivery.

---

## EP-05 · Vendor Order Management

### Value

Gives vendors full visibility and control over incoming orders — view, filter, advance status, and mark delivered.

### API usage

**List vendor orders:**
```http
GET /api/v1/vendor/orders?status=Paid&page=1&pageSize=20
Authorization: Bearer <vendor-token>
```

**View order detail:**
```http
GET /api/v1/vendor/orders/{orderId}
```

**Advance order status:**
```http
PUT /api/v1/vendor/orders/{orderId}/status
{
  "status": "InPreparation",
  "estimatedDeliveryAt": "2026-04-06T16:00:00Z"
}
```

**Valid status transitions (vendor-driven):**
```
Paid → InPreparation → ReadyForPickup → InDelivery → Delivered
Any active status → Cancelled
```

When an order reaches `Delivered`, all bouquets in the order are automatically marked as `Sold`.

### Vendor Portal UI

Navigate to **Orders** in the Vendor Portal sidebar (`http://localhost:5002/Orders`). The page shows tabs for Pending / Confirmed / Ready / Completed. Click an order to see line items and a timeline. Use the status buttons to advance the order through its lifecycle.

---

## EP-06 · Favourites, Subscriptions & Notifications

### Value

Lets customers save bouquets for later, set up alerts for new bouquets matching their preferences, and receive push notifications.

### API usage

**Favourites:**
```http
POST   /api/v1/favourites/{bouquetId}      # Add (idempotent)
DELETE /api/v1/favourites/{bouquetId}      # Remove (idempotent)
GET    /api/v1/favourites?page=1&pageSize=20
```

**Subscriptions — get notified when matching bouquets appear:**
```http
POST /api/v1/subscriptions
{
  "type": "ByFlowerType",
  "flowerType": "roses",
  "maxPrice": 100,
  "frequency": "instant",
  "centerLatitude": 52.23,
  "centerLongitude": 21.01,
  "radiusKm": 10
}
```
Frequency options: `instant` (push immediately), `daily`, `weekly`.

**Disable all subscriptions at once:**
```http
PUT /api/v1/subscriptions/disable-all
```

**Register device for push notifications:**
```http
POST /api/v1/notifications/devices
{ "deviceToken": "fcm-token-xxx", "platform": "Android", "appVersion": "1.2.0" }
```

**Notification feed:**
```http
GET /api/v1/notifications?unreadOnly=true&page=1&pageSize=20
PUT /api/v1/notifications/{id}/read
PUT /api/v1/notifications/read-all
```

### Customer App UI

- **Favourites** page at `/Favourites` — card grid of saved bouquets with availability badges.
- **Subscriptions** page at `/Subscriptions` — list existing subscriptions and create new ones with a form sidebar.
- Heart toggle buttons appear on bouquet cards throughout the app.

---

## EP-07 · Customer Profile & Saved Addresses

### Value

Full customer self-service: edit profile, manage delivery addresses, configure notification preferences, and initiate GDPR-compliant account deletion.

### API usage

**Get profile with stats:**
```http
GET /api/v1/profile
```
Returns name, email, phone, preferred flower types, notification settings, plus aggregated stats (total orders, active subscriptions, total spent).

**Update profile:**
```http
PUT /api/v1/profile
{ "firstName": "Jane", "lastName": "Smith", "phone": "+48123456789" }
```

**Saved addresses (max 10):**
```http
POST /api/v1/profile/addresses
{ "street": "ul. Nowy Świat 15", "city": "Warszawa", "postalCode": "00-029", "isDefault": true }

GET    /api/v1/profile/addresses
DELETE /api/v1/profile/addresses/{id}
```

**Request account deletion (30-day grace period):**
```http
DELETE /api/v1/profile
{ "confirmationText": "DELETE" }
```
Returns `202 Accepted`. The account enters `PendingDeletion` status with a 30-day countdown.

### Customer App UI

- **Profile** page at `/Profile` — edit name, view order stats, manage notification preferences.
- **Addresses** sub-page at `/Profile/Addresses` — list, add, and delete saved addresses.

---

## EP-08 · Real-time Events & Freshness Engine

### Value

Powers the live map experience and ensures bouquet freshness scores stay accurate. Customers see bouquets appear/disappear in real-time. Vendors get instant vase status alerts.

### How it works

**Freshness scoring** runs every hour (plus on each MQTT sensor reading):
```
Score = 100
  − (temperature − 25°C) × 3    (if temp > 25°C)
  − (40% − humidity) × 1.5      (if humidity < 40%)
  − hours_since_placed × 0.5

Labels: "Just picked" (90+), "Very fresh" (70-89), "Fresh" (50-69),
        "Good" (30-49), "Last chance" (<30)
```
Bouquets that drop below score 20 are automatically expired.

**SignalR real-time events** (connect to `/hubs/bouquet`):

```javascript
// Browser — join a location group
connection.invoke("JoinRadiusGroup", 52.23, 21.01, 5.0);

// Listen for events
connection.on("BouquetBecameAvailable", (data) => {
    // Add marker to map
});
connection.on("BouquetBecameUnavailable", (data) => {
    // Remove marker from map
});
connection.on("FreshnessUpdated", (data) => {
    // Update freshness badge on bouquet card
});
connection.on("OrderStatusChanged", (data) => {
    // Update order tracking UI
});
```

**Domain events wired to SignalR:**
- New bouquet activated → `BouquetBecameAvailable` broadcast to radius groups
- Bouquet sold → `BouquetBecameUnavailable` broadcast
- Vase goes offline → `VaseStatusChanged` to vendor group
- Freshness score changes → `FreshnessUpdated` to radius groups

---

## EP-09 · Performance, Reliability & Cross-cutting

### Value

Production-hardening: rate limiting prevents abuse, ETag caching reduces bandwidth, the global exception handler ensures consistent error responses, and subscription matching triggers instant push notifications when matching bouquets appear.

### Middleware pipeline

| Middleware | Header / Behaviour |
|------------|-------------------|
| **GlobalExceptionMiddleware** | Returns RFC 7807 Problem Details JSON for all unhandled exceptions |
| **RequestIdMiddleware** | Sets `X-Request-Id` response header (correlates with logs) |
| **LocalisationMiddleware** | Reads `Accept-Language`, defaults to `pl-PL` |
| **ETagMiddleware** | Computes SHA-256 ETag on GET responses; returns `304 Not Modified` when `If-None-Match` matches |
| **IdempotencyMiddleware** | Caches POST responses by `Idempotency-Key` header for 24 hours |

### Rate limiting policies

| Policy | Limit | Applied to |
|--------|-------|-----------|
| `fixed` (global) | 100 req/min | All endpoints |
| `auth` | 10 req/min | Login, registration |
| `orders` | 20 req/min (sliding) | Order creation |

Exceeded limits return `429 Too Many Requests` with a `Retry-After` header.

### Subscription matching engine

A background service listens for `BouquetBecameAvailable` domain events. For each new bouquet, it finds customers with matching active subscriptions (by flower type, vendor, price range, radius). Subscriptions with `frequency: instant` trigger an immediate push notification.

---

## EP-10 · Analytics API

### Value

Fixes the critical VendorId.New() bug (all analytics queries were returning data for random vendors) and delivers 6 working analytics endpoints covering vendor performance, vase health, real-time dashboards, platform-wide insights, peer benchmarks, and CSV export.

### API usage

**Vendor dashboard analytics:**
```http
GET /api/v1/analytics/vendor?period=30
Authorization: Bearer <vendor-token>
```
Returns: total revenue, order count, average order value, top bouquets by revenue, and daily trend data.

**Vase performance:**
```http
GET /api/v1/analytics/vases?vaseId={id}&period=7
```
Returns: uptime percentage, scan count, orders attributed, average freshness, environment metrics (temp, humidity).

**Real-time snapshot:**
```http
GET /api/v1/analytics/realtime
```
Returns: active vases (online count), available bouquets, open orders, hourly revenue.

**Platform insights (admin only):**
```http
GET /api/v1/analytics/platform?period=30
Authorization: Bearer <admin-token>
```
Returns: total vendors, customers, GMV (gross marketplace volume), top vendors by revenue.

**Peer benchmarks:**
```http
GET /api/v1/analytics/benchmarks?period=30
```
Returns: revenue percentile vs. peers, performance ratios, strengths and improvement areas.

**Export to CSV:**
```http
GET /api/v1/analytics/export?exportType=vendor&format=csv&period=30
```
Returns CSV file with order and revenue data for the requested period.

---

## EP-11 · Admin Portal — Customer Management & Platform Analytics

### Value

Platform administrators can now manage customers (view, suspend, reactivate), oversee all orders across the platform, view platform-wide analytics, and suspend/reactivate vendors.

### Admin Portal UI (`http://localhost:5001`)

**Dashboard** (`/`) — now shows platform statistics summary (total vendors, customers, orders) with quick-access links to management sections.

**Customers** (`/Customers/List`) — searchable, paginated table of all registered customers. Click a customer to see their profile details. Use **Suspend** / **Reactivate** buttons to manage account status.

**Orders** (`/Orders/List`) — platform-wide order list with status filter tabs. Useful for monitoring order volume and identifying stuck orders.

**Vendors** (`/Vendors/Details/{id}`) — updated with **Suspend** and **Reactivate** buttons. Suspending a vendor disables their storefront and vase connections.

### API endpoints (PlatformAdmin policy)

```http
GET    /api/v1/admin/customers?search=jane&page=1&pageSize=20
GET    /api/v1/admin/customers/{id}
POST   /api/v1/admin/customers/{id}/suspend
POST   /api/v1/admin/customers/{id}/reactivate
GET    /api/v1/admin/orders?status=Paid&page=1&pageSize=20
POST   /api/v1/admin/vendors/{id}/suspend
POST   /api/v1/admin/vendors/{id}/reactivate
```

---

## EP-12 · Vendor Portal — Orders, Analytics & Profile

### Value

Vendors can manage incoming orders, view revenue analytics, edit their business profile, configure opening hours, and upload bouquet photos — all from the portal UI. Critical bugs fixed: VendorId now reads from JWT instead of session; mock-vase upload replaced with real API call.

### Vendor Portal UI (`http://localhost:5002`)

**Orders** (`/Orders`) — tabbed view (Pending / Confirmed / Ready / Completed). Click an order to see line items and timeline. Advance the status with one click: Confirm → Ready → Complete.

**Analytics** (`/Analytics`) — six stat cards showing revenue, order count, bouquets sold, average order value, active vases, and available bouquets.

**Settings > Profile** (`/Settings/Profile`) — edit business name, description, email, and phone.

**Settings > Opening Hours** (`/Settings/OpeningHours`) — weekly schedule editor. Select operating days and set open/close times. These hours feed the `isCurrentlyOpen` flag shown to customers.

### Bugs fixed

| Bug | Fix |
|-----|-----|
| VendorId read from session | `ICurrentVendorService` reads `vendor_id` JWT claim (falls back to session) |
| Mock-vase raw HttpClient upload | Uses `ApiClient.UploadBouquetPhotoAsync` calling `POST /api/v1/bouquets/{id}/photo` |

---

## EP-13 · Customer Web App — Authentication & Shopping

### Value

Transforms the customer web app from a read-only browser into a full shopping experience with authentication, cart management, checkout, and an interactive map.

### Customer App UI (`http://localhost:5003`)

**Home page** (`/`) — Leaflet.js interactive map centered on Warsaw (52.23, 21.01). Nearby bouquet markers load automatically. Click a marker to see the bouquet name, price, and a link to the detail page.

**Cart** (`/Cart`) — view items with prices, remove individual items, clear all, and proceed to checkout. Cart badge in the navigation shows item count.

**Checkout** (`/Checkout`) — order preview showing line items, delivery fee, and total. "Place Order" creates the order. Redirects to a confirmation page on success.

**Confirmation** (`/Checkout/Confirmation/{orderId}`) — displays order ID and a thank-you message.

**Protected routes**: Cart, Checkout, Orders, Favourites, Profile, and Subscriptions require authentication. Unauthenticated users are redirected to `/Account/Login`.

---

## EP-14 · Customer Web App — Profile, Favorites & Subscriptions

### Value

Gives logged-in customers full self-service: order history with timeline tracking, favourite bouquets, profile editing with saved addresses, and subscription management.

### Customer App UI

**Orders** (`/Orders`) — list with status filter tabs (All / Pending / Confirmed / In Delivery / Completed / Cancelled). Click an order to see full detail including line items, status timeline, estimated delivery, and delivery QR code.

**Favourites** (`/Favourites`) — card grid of saved bouquets with availability badges ("Available", "Sold", "Expired"). Toggle favourites from any bouquet card.

**Profile** (`/Profile`) — view and edit name, phone, and notification preferences. Sidebar shows account stats (total orders, active subscriptions).

**Addresses** (`/Profile/Addresses`) — list saved delivery addresses with an inline add form. Delete addresses with one click.

**Subscriptions** (`/Subscriptions`) — list active subscriptions with type, criteria, and frequency. Create new subscriptions with a form. "Disable All" button turns off all notifications at once.

---

## EP-15 · IoT Integration

### Value

Moves IoT from simulation to production-readiness: real photo uploads, authenticated MQTT connections, smarter offline detection, and over-the-air firmware updates.

### Bouquet photo upload

```http
POST /api/v1/bouquets/{id}/photo
Authorization: Bearer <vendor-token>
Content-Type: multipart/form-data

# Form field: "photo" — jpg, png, or webp, max 5 MB
```
Returns `{ "photoUrl": "/uploads/bouquets/{bouquetId}_{filename}" }`. The Vendor Portal's vase details page now uses this endpoint instead of the mock-vase workaround.

### MQTT broker authentication

The MQTT client now reads `Mqtt:Username` and `Mqtt:Password` from configuration and passes credentials when connecting to Mosquitto. Configure in `appsettings.Development.json`:
```json
{
  "Mqtt": {
    "Host": "localhost",
    "Port": 1883,
    "Username": "flowershop",
    "Password": "dev123"
  }
}
```

### Stale vase detection

`ISmartVaseRepository.GetStaleVasesAsync(cutoffMinutes)` queries vases whose last heartbeat exceeds the threshold and aren't already marked offline. The `VaseHealthMonitoringService` uses this every 15 seconds to detect and report offline vases.

### OTA firmware updates

```http
POST /api/v1/smartvases/{id}/firmware
Authorization: Bearer <admin-token>
{
  "firmwareUrl": "https://firmware.flowershop.dev/v2.1.0/vase.bin",
  "version": "2.1.0"
}
```
Returns `202 Accepted`. The server publishes an MQTT message to `vase/{serialNumber}/ota` containing the download URL. The vase firmware handles the update autonomously.

---

## Quick Reference — Service URLs

| Service | URL | Credentials |
|---------|-----|-------------|
| API + Swagger | `http://localhost:8080/swagger` | — |
| Admin Portal | `http://localhost:5001` | platformadmin / AdminPass123! |
| Vendor Portal | `http://localhost:5002` | (vendor accounts) |
| Customer App | `http://localhost:5003` | (customer accounts) |
| SignalR Hub | `ws://localhost:8080/hubs/bouquet` | — |
| Stripe Webhook | `POST http://localhost:8080/api/stripe/webhook` | Stripe-Signature header |
| Health Check | `http://localhost:8080/health` | — |
