# FMF — API Contracts

**Document:** API-CONTRACTS.md
**App:** Find My Flowers (FMF)
**Version:** 1.2 (MVP)

This document defines the REST API endpoints, WebSocket events, and request/response schemas that the FMF mobile app consumes from the Flower Shop IoT Platform backend.

---

## Base URL

| Environment | URL |
|---|---|
| Development | `http://localhost:8080` |
| Staging | `https://api-staging.bloomplatform.pl` |
| Production | `https://api.bloomplatform.pl` |

All REST endpoints are prefixed with `/api`. Dates use ISO 8601 UTC format. Monetary amounts are decimals with 2 fractional digits.

---

## Authentication

All authenticated endpoints require a Bearer token in the `Authorization` header:

```
Authorization: Bearer <access_token>
```

Tokens are issued by Keycloak. FMF uses **OIDC Authorization Code + PKCE** flow.

### Keycloak OIDC Endpoints

| Purpose | URL |
|---|---|
| Discovery | `{keycloakBase}/realms/flowershop/.well-known/openid-configuration` |
| Authorization | `{keycloakBase}/realms/flowershop/protocol/openid-connect/auth` |
| Token | `{keycloakBase}/realms/flowershop/protocol/openid-connect/token` |
| End Session | `{keycloakBase}/realms/flowershop/protocol/openid-connect/logout` |
| UserInfo | `{keycloakBase}/realms/flowershop/protocol/openid-connect/userinfo` |

| Environment | Keycloak Base URL |
|---|---|
| Development | `http://localhost:8090` |
| Staging | `https://auth-staging.bloomplatform.pl` |
| Production | `https://auth.bloomplatform.pl` |

### OIDC Client Configuration

| Parameter | Value |
|---|---|
| `client_id` | `fmf-mobile` |
| `response_type` | `code` |
| `scope` | `openid profile email offline_access` |
| `code_challenge_method` | `S256` |
| `redirect_uri` | `pl.bloomplatform.fmf://auth/callback` |
| `post_logout_redirect_uri` | `pl.bloomplatform.fmf://auth/logout-callback` |

**Google OAuth (AUTH-03):** Google social login is configured as an Identity Provider in Keycloak. The mobile app uses the standard OIDC Authorization Code + PKCE flow — Keycloak renders the Google login button on its hosted login page. No additional client-side Google SDK integration is required.

### Token Lifecycle

| Token | Lifetime | Storage |
|---|---|---|
| Access token | 5 minutes | In-memory only |
| Refresh token | 30 days | Platform secure storage (iOS Keychain / Android Keystore) |
| ID token | 5 minutes | In-memory only |

**Silent Renewal:** Use the refresh token to obtain a new access token before expiry. The mobile app should proactively refresh when the access token has < 30 seconds remaining.

**Token Refresh Request:**

```
POST {keycloakBase}/realms/flowershop/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&client_id=fmf-mobile
&refresh_token=<refresh_token>
```

### JWT Access Token Claims

```json
{
  "sub": "a1b2c3d4-...",
  "preferred_username": "jan.kowalski",
  "email": "jan@example.com",
  "email_verified": true,
  "given_name": "Jan",
  "family_name": "Kowalski",
  "realm_access": {
    "roles": ["User"]
  },
  "customer_id": "cust-123",
  "iss": "{keycloakBase}/realms/flowershop",
  "exp": 1712745900,
  "iat": 1712745600
}
```

### Sign-Out Sequence

The mobile app must perform the following steps on sign-out (PRF-07):

1. **Unregister device** — `DELETE /api/notifications/device` with the current FCM/APNs token.
2. **End Keycloak session** — redirect to Keycloak End Session endpoint with `id_token_hint` and `post_logout_redirect_uri`.
3. **Clear local state** — remove access token, refresh token, and ID token from memory and secure storage. Clear any cached data.

```
GET {keycloakBase}/realms/flowershop/protocol/openid-connect/logout
    ?id_token_hint=<id_token>
    &post_logout_redirect_uri=pl.bloomplatform.fmf://auth/logout-callback
```

---

## 1. Auth & Registration Endpoints

### POST /api/auth/register

Creates a new customer account. The account is created in Keycloak, which sends a verification email. The user can log in immediately but will see a reminder banner until email is verified.

**Auth:** None

**Request Body:**

```json
{
  "email": "jan@example.com",
  "password": "SecureP@ss1",
  "firstName": "Jan",
  "lastName": "Kowalski",
  "acceptedTermsVersion": "1.0",
  "acceptedPrivacyPolicyVersion": "1.0"
}
```

**Validation Rules:**

| Field | Rules |
|---|---|
| `email` | Valid email, unique, max 255 chars |
| `password` | Min 8 chars, at least 1 uppercase, 1 lowercase, 1 digit, 1 special char |
| `firstName` | Required, 1–100 chars |
| `lastName` | Required, 1–100 chars |
| `acceptedTermsVersion` | Required |
| `acceptedPrivacyPolicyVersion` | Required |

**Response 201:**

```json
{
  "userId": "a1b2c3d4-...",
  "email": "jan@example.com",
  "emailVerified": false,
  "message": "Account created. A verification email has been sent."
}
```

**Response 409:** Email already registered.

**Email Verification:** Handled entirely by Keycloak. The verification link in the email points to Keycloak, which marks the account as verified. The mobile app checks `email_verified` in the JWT claims and displays a reminder banner if `false`. No custom verify endpoint is needed.

---

### POST /api/auth/resend-verification

Triggers Keycloak to resend the email verification link. Rate-limited to prevent abuse.

**Auth:** Required (user must be logged in but unverified)

**Response 202:** Verification email sent (always returned; does not reveal internal state).
**Response 429:** Rate limited — max 1 request per 60 seconds.

---

### POST /api/auth/password-reset/request

Initiates a password reset flow. Keycloak sends the reset email.

**Auth:** None

**Request Body:**

```json
{
  "email": "jan@example.com"
}
```

**Response 202:** Always returned (does not reveal whether email exists).

---

### POST /api/auth/password-reset/confirm

Completes the password reset using the token from the email link.

**Auth:** None

**Request Body:**

```json
{
  "token": "reset-token-from-email",
  "newPassword": "NewSecureP@ss2"
}
```

**Response 200:** Password updated.
**Response 400:** Token invalid or expired.

---

## 2. App Configuration Endpoint

### GET /api/config

Returns environment-specific configuration the mobile app needs at startup. Called once on app launch and cached for the session.

**Auth:** None

**Response 200:**

```json
{
  "stripePublishableKey": "pk_live_...",
  "mapTileProvider": "google",
  "supportEmail": "support@bloomplatform.pl",
  "termsUrl": "https://bloomplatform.pl/terms",
  "privacyPolicyUrl": "https://bloomplatform.pl/privacy",
  "minAppVersion": "1.0.0"
}
```

| Field | Description |
|---|---|
| `stripePublishableKey` | Stripe publishable key for PaymentSheet initialization |
| `mapTileProvider` | Map provider hint (reserved for future use) |
| `supportEmail` | Customer support contact |
| `termsUrl` | Link to current Terms of Service |
| `privacyPolicyUrl` | Link to current Privacy Policy |
| `minAppVersion` | Minimum supported app version; app should show a force-update prompt if current version is lower |

---

## 3. Bouquet Endpoints

### Bouquet Status Values

All bouquet-related endpoints use the following `status` field values:

| Status | Display Label | Description |
|---|---|---|
| `available` | Available | Bouquet is available for purchase |
| `reserved` | Reserved | Bouquet is reserved during an active checkout (up to 10 minutes) |
| `sold` | Sold | Bouquet has been purchased |
| `expired` | No Longer Available | Bouquet was removed by vendor or freshness dropped below threshold |
| `vase_offline` | Temporarily Unavailable | The smart vase went offline; bouquet may return |

---

### GET /api/bouquets/nearby

Returns bouquets available within a radius of the given location.

**Auth:** Optional (unauthenticated users can browse)

**Query Parameters:**

| Param | Type | Required | Description |
|---|---|---|---|
| `lat` | double | Yes | Customer latitude |
| `lng` | double | Yes | Customer longitude |
| `radiusKm` | int | Yes | Search radius: `5`, `10`, or `15` |
| `vendorType` | string | No | Filter: `shop`, `freelance`, or omit for all |
| `search` | string | No | Filter by vendor name or neighbourhood |
| `page` | int | No | Page number (default: `1`) |
| `pageSize` | int | No | Items per page (default: `20`, max: `50`) |

**Response 200:**

```json
{
  "items": [
    {
      "bouquetId": "b3f1a2c4-...",
      "vaseId": "v-001",
      "vendorId": "vend-42",
      "vendorName": "Flora Elegance",
      "vendorType": "shop",
      "price": 89.00,
      "currency": "PLN",
      "status": "available",
      "freshnessScore": 84,
      "freshnessLabel": "Very Fresh",
      "thumbnailUrl": "https://cdn.bloomplatform.pl/bouquets/b3f1a2c4/thumb.jpg",
      "location": {
        "latitude": 52.2297,
        "longitude": 21.0122,
        "displayAddress": "Śródmieście, Warsaw",
        "exactAddressHidden": false
      },
      "distanceKm": 0.8,
      "availableFrom": "2025-04-10T09:15:00Z",
      "isFavourite": false
    }
  ],
  "totalCount": 42,
  "page": 1,
  "pageSize": 20
}
```

**Notes:**
- `isFavourite` is always `false` for unauthenticated requests.
- `exactAddressHidden` is `true` for freelance florists; coordinates are approximate within ~500 m (SEC-06).

**Response 400:** Invalid coordinates or radius value.

**Geocoding Note (DISC-09):** This endpoint requires numeric `lat`/`lng` coordinates. When the user denies location permission and enters an address manually, the mobile app must geocode the address client-side using the **Google Places SDK** (the same SDK used for delivery address autocomplete in CHK-04) before calling this endpoint. The backend does not provide a geocoding endpoint.

---

### GET /api/bouquets/{id}

Returns full detail for a single bouquet.

**Auth:** Optional

**Query Parameters:**

| Param | Type | Required | Description |
|---|---|---|---|
| `lat` | double | No | Customer latitude (used to compute `distanceKm`) |
| `lng` | double | No | Customer longitude (used to compute `distanceKm`) |

**Response 200:**

```json
{
  "bouquetId": "b3f1a2c4-...",
  "vaseId": "v-001",
  "vendorId": "vend-42",
  "vendorName": "Flora Elegance",
  "vendorType": "shop",
  "vendorOpeningHours": "Mon-Sat 09:00-20:00, Sun 10:00-16:00",
  "isVendorCurrentlyOpen": true,
  "price": 89.00,
  "currency": "PLN",
  "status": "available",
  "freshnessScore": 84,
  "freshnessLabel": "Very Fresh",
  "imageUrl": "https://cdn.bloomplatform.pl/bouquets/b3f1a2c4/full.jpg",
  "aiTags": [
    { "type": "rose", "label": "Roses", "count": 8 },
    { "type": "tulip", "label": "Tulips", "count": 5 },
    { "type": "eucalyptus", "label": "Eucalyptus", "count": null }
  ],
  "location": {
    "latitude": 52.2297,
    "longitude": 21.0122,
    "displayAddress": "Śródmieście, Warsaw",
    "exactAddressHidden": false
  },
  "distanceKm": 0.8,
  "availableFrom": "2025-04-10T09:15:00Z",
  "isFavourite": false
}
```

**Notes:**
- `exactAddressHidden: true` for freelance florists — location is approximate within ~500 m (SEC-06).
- `isFavourite` is `false` for unauthenticated requests.
- `distanceKm` is included only when `lat` and `lng` query params are provided; omitted otherwise.
- `isVendorCurrentlyOpen` is computed server-side from `vendorOpeningHours` and the server's current time. Returns `null` if opening hours are not configured.

**Response 404:** Bouquet not found or no longer available.

---

### GET /api/bouquets/qr/{code}

Resolves a bouquet from a scanned QR code.

**Auth:** Optional (unauthenticated users can scan, prompted to login before purchase)

**Path Parameters:** `code` — 8-character alphanumeric short code (e.g., `A3F8K2M1`) or full UUID. Both formats are accepted.

QR codes displayed on smart vases encode the URL `https://fmf.bloomplatform.pl/b/{code}`. The mobile app intercepts this deep link and extracts `{code}` to call this endpoint.

**Response 200:** Same schema as `GET /api/bouquets/{id}`
**Response 404:** Code not found or expired.
**Response 410:** Bouquet was available but is now sold/reserved.

---

### Image Dimensions

| Image field | Resolution | Aspect ratio | Typical size |
|---|---|---|---|
| `thumbnailUrl` | 400 × 400 px | 1:1 (square) | 15–30 KB |
| `imageUrl` | 1200 × 900 px | 4:3 | 80–200 KB |

All images are served via CDN with `Cache-Control: public, max-age=604800` (7 days). The mobile app should use these dimensions for placeholder/skeleton sizing before the image loads.

---

## 4. Favourites Endpoints

### GET /api/favourites

Returns all favourited bouquets for the authenticated customer.

**Auth:** Required

**Query Parameters:**

| Param | Type | Required | Description |
|---|---|---|---|
| `lat` | double | No | Customer latitude (used to compute `distanceKm`) |
| `lng` | double | No | Customer longitude (used to compute `distanceKm`) |
| `page` | int | No | Page number (default: `1`) |
| `pageSize` | int | No | Items per page (default: `20`, max: `50`) |

**Response 200:**

```json
{
  "items": [
    {
      "bouquetId": "b3f1a2c4-...",
      "vendorId": "vend-42",
      "vendorName": "Flora Elegance",
      "vendorType": "shop",
      "price": 89.00,
      "currency": "PLN",
      "status": "available",
      "freshnessScore": 84,
      "freshnessLabel": "Very Fresh",
      "thumbnailUrl": "https://cdn.bloomplatform.pl/bouquets/b3f1a2c4/thumb.jpg",
      "location": {
        "latitude": 52.2297,
        "longitude": 21.0122,
        "displayAddress": "Śródmieście, Warsaw",
        "exactAddressHidden": false
      },
      "distanceKm": 0.8,
      "availableFrom": "2025-04-10T09:15:00Z",
      "savedAt": "2025-04-09T14:00:00Z"
    }
  ],
  "totalCount": 3,
  "page": 1,
  "pageSize": 20
}
```

**Notes:**
- `distanceKm` is included only when `lat` and `lng` query params are provided.
- Bouquets that are no longer available still appear in favourites with their last known `status` (e.g., `sold`, `expired`). The app should dim or badge unavailable favourites.

---

### POST /api/favourites/{bouquetId}

Adds a bouquet to the customer's favourites.

**Auth:** Required
**Response 201:** No content.
**Response 409:** Already favourited.

---

### DELETE /api/favourites/{bouquetId}

Removes a bouquet from the customer's favourites.

**Auth:** Required
**Response 204:** No content.

---

## 5. Order Endpoints

### POST /api/orders/preview

Returns cost breakdown for a potential order **without** creating the order or reserving the bouquet. Used to display the order summary screen (CHK-06) before the customer commits.

**Auth:** Required
**Idempotent:** Yes — safe to call multiple times.

**Request Body (delivery):**

```json
{
  "bouquetIds": ["b3f1a2c4-..."],
  "deliveryMethod": "delivery",
  "street": "ul. Marszałkowska 10",
  "city": "Warsaw",
  "postalCode": "00-001",
  "notes": "3rd floor, flat 12"
}
```

**Request Body (pickup):**

```json
{
  "bouquetIds": ["b3f1a2c4-..."],
  "deliveryMethod": "pickup"
}
```

**Response 200 (delivery):**

```json
{
  "lines": [
    {
      "bouquetId": "b3f1a2c4-...",
      "bouquetName": "Spring Roses Bouquet",
      "bouquetImageUrl": "https://cdn.bloomplatform.pl/bouquets/b3f1a2c4/full.jpg",
      "unitPrice": 89.00
    }
  ],
  "linesSubtotal": 89.00,
  "deliveryFee": 25.00,
  "totalAmount": 114.00,
  "currency": "PLN",
  "vendorName": "Flora Elegance",
  "deliveryMethod": "delivery",
  "deliveryDistanceKm": 4.82,
  "resolvedDeliveryAddress": {
    "street": "ul. Marszałkowska 10",
    "city": "Warsaw",
    "postalCode": "00-001",
    "notes": "3rd floor, flat 12",
    "latitude": 52.2321,
    "longitude": 21.0151,
    "displayName": "Marszałkowska 10, 00-001 Warszawa, Poland"
  }
}
```

**Response 200 (pickup):**

```json
{
  "lines": [
    {
      "bouquetId": "b3f1a2c4-...",
      "bouquetName": "Spring Roses Bouquet",
      "bouquetImageUrl": "https://cdn.bloomplatform.pl/bouquets/b3f1a2c4/full.jpg",
      "unitPrice": 89.00
    }
  ],
  "linesSubtotal": 89.00,
  "deliveryFee": 0.00,
  "totalAmount": 89.00,
  "currency": "PLN",
  "vendorName": "Flora Elegance",
  "deliveryMethod": "pickup"
}
```

**Response 400/422:** See the delivery error table at the end of this section.

**Notes:**
- This endpoint does NOT reserve the bouquet. Reservation happens only on `POST /api/orders`.
- `deliveryFee` is `0.00` when `deliveryMethod` is `pickup`; flat `25.00 PLN` when delivery falls within the vendor's 10 km service radius.
- `deliveryDistanceKm` and `resolvedDeliveryAddress` are present only on delivery previews. The server geocodes the address via Nominatim (cached for 10 minutes per address) and returns the resolved latitude/longitude so the client can surface a map preview.

---

### POST /api/orders

Creates a new order and reserves the bouquet. The bouquet is locked for 10 minutes. If payment is not confirmed within that window, the reservation is released automatically.

**Auth:** Required
**Idempotent:** No — use `Idempotency-Key` header to prevent duplicate orders on retry (see section 18).

**Request Headers:**

| Header | Required | Description |
|---|---|---|
| `Idempotency-Key` | Required | Client-generated UUID v4, unique per order attempt |

**Request Body:**

```json
{
  "bouquetIds": ["b3f1a2c4-..."],
  "deliveryMethod": "delivery",
  "street": "ul. Marszałkowska 10",
  "city": "Warsaw",
  "postalCode": "00-001",
  "notes": "3rd floor, flat 12",
  "expectedTotalAmount": 114.00
}
```

| Field | Rules |
|---|---|
| `bouquetIds` | One or more bouquet UUIDs; all must belong to the same vendor |
| `deliveryMethod` | `delivery` or `pickup` |
| `street`, `city`, `postalCode`, `notes` | Required when `deliveryMethod` is `delivery`; ignored for `pickup`. The server re-geocodes the address and stores the resolved latitude/longitude on the order so couriers can route without re-geocoding. |
| `expectedTotalAmount` | Server validates the computed total matches. Returns 409 with `price-changed` type if it doesn't. |

**Response 201:**

```json
{
  "orderId": "ord-8821",
  "status": "created",
  "bouquetReservedUntil": "2025-04-10T10:25:00Z",
  "stripeClientSecret": "pi_stripe_..._secret_...",
  "stripeEphemeralKey": "ek_live_...",
  "stripeCustomerId": "cus_...",
  "bouquetPrice": 89.00,
  "deliveryFee": 12.00,
  "totalAmount": 101.00,
  "currency": "PLN"
}
```

**Stripe Integration Notes:**
- Use `stripeClientSecret` with the Stripe Mobile SDK PaymentSheet.
- `stripeEphemeralKey` and `stripeCustomerId` are needed for the PaymentSheet configuration.
- The Stripe publishable key is obtained from `GET /api/config` (`stripePublishableKey` field).
- The app never handles raw card data (SEC-05).
- Supported payment methods: card, Apple Pay, Google Pay.

**Response 409 (already reserved):**

```json
{
  "type": "https://bloomplatform.pl/errors/conflict",
  "title": "Bouquet Already Reserved",
  "status": 409,
  "detail": "This bouquet has been reserved by another customer."
}
```

**Response 409 (price changed):**

```json
{
  "type": "https://bloomplatform.pl/errors/price-changed",
  "title": "Price Changed",
  "status": 409,
  "detail": "The bouquet price has changed since your preview.",
  "currentTotalAmount": 105.00,
  "currency": "PLN"
}
```

**Response 400/422:** See the delivery error table below.

#### Delivery error codes (preview + create)

| Error code                | HTTP | When |
|---------------------------|------|------|
| `delivery-address-required` | 400  | `deliveryMethod=delivery` but street/city/postalCode missing |
| `geocoding-failed`          | 422  | The geocoder returned no match or errored |
| `delivery-out-of-range`     | 422  | Resolved address is more than 10 km from the vendor location |
| `vendor-location-unknown`   | 422  | Vendor has no configured location (can't compute distance) |

All error responses use the shape `{ "error": "<code>" }`.

**Side effects during the order lifecycle:**
- **On `POST /orders/{id}/confirm-payment` for a delivery order:** the order advances from `paid` → `preparing`, and the smart vase hosting the bouquet is sent an MQTT `prepare_delivery` card carrying `display_mode=delivery`, `delivery_status=PREPARING`, `delivery_qr=geo:<lat>,<lon>?q=<url-encoded-address>`, and `delivery_address=<street, postal city>`.
- **When the vendor transitions the order to `in_delivery`** (vendor portal `PUT /api/v1/vendor/orders/{id}/status`): the bouquet is unassigned from its vase (so the vase can host a new bouquet), and the vase receives an MQTT `delivery_started` card resetting it to `display_mode=idle` with banner "Waiting for bouquet".

---

### POST /api/orders/{id}/confirm-payment

Confirms that the Stripe payment succeeded. Called after Stripe SDK reports success on the client. The server also receives a Stripe webhook independently; this endpoint ensures the order transitions promptly without waiting for the webhook.

**Auth:** Required
**Idempotent:** Yes — safe to retry. Duplicate confirmations for an already-paid order return 200 with the same response.

**Request Body:**

```json
{
  "paymentIntentId": "pi_stripe_..."
}
```

**Response 200:**

```json
{
  "orderId": "ord-8821",
  "status": "paid",
  "paidAt": "2025-04-10T10:11:00Z"
}
```

**Response 402:** Payment failed or was declined.
**Response 404:** Order not found.
**Response 409:** Order already cancelled.

---

### POST /api/orders/{id}/cancel

Cancels an order. Only allowed while the order is in a cancellable state.

**Auth:** Required

**Response 200:**

```json
{
  "orderId": "ord-8821",
  "status": "cancelled",
  "cancelledAt": "2025-04-10T10:15:00Z",
  "refundStatus": "not_applicable"
}
```

| `refundStatus` value | Meaning |
|---|---|
| `not_applicable` | Order was not yet paid — no refund needed |
| `refund_initiated` | Payment was made — Stripe refund has been initiated |
| `refund_completed` | Refund processed (may appear on subsequent GET) |

**Cancellable statuses:**

| Order status | Can cancel? | Notes |
|---|---|---|
| `created` | Yes | Reservation released immediately |
| `paid` | Yes | Automatic Stripe refund initiated |
| `preparing` | No | Contact support: `support@bloomplatform.pl` |
| `in_delivery` | No | Contact support |
| `delivered` | No | Use report-issue endpoint (post-MVP) |
| `cancelled` | No | Already cancelled (returns 409) |

**Response 409:** Order is not in a cancellable state.
**Response 404:** Order not found or does not belong to the authenticated customer.

---

### GET /api/orders

Returns all orders for the authenticated customer.

**Auth:** Required
**Query Parameters:**

| Param | Type | Required | Description |
|---|---|---|---|
| `status` | string | No | Filter: `active`, `completed`, `cancelled` |
| `page` | int | No | Page number (default: `1`) |
| `pageSize` | int | No | Items per page (default: `20`, max: `50`) |

**Status filter mapping:**

| Filter value | Includes order statuses |
|---|---|
| `active` | `created`, `paid`, `preparing`, `in_delivery` |
| `completed` | `delivered` |
| `cancelled` | `cancelled` |
| _(omitted)_ | All statuses |

**Response 200:**

```json
{
  "items": [
    {
      "orderId": "ord-8821",
      "bouquetThumbnailUrl": "https://cdn.bloomplatform.pl/...",
      "vendorName": "Flora Elegance",
      "totalAmount": 101.00,
      "currency": "PLN",
      "status": "in_delivery",
      "deliveryMethod": "delivery",
      "createdAt": "2025-04-10T10:10:00Z"
    }
  ],
  "totalCount": 5,
  "page": 1,
  "pageSize": 20
}
```

---

### GET /api/orders/{id}

Returns full detail of a single order.

**Auth:** Required

**Response 200 (delivery order):**

```json
{
  "orderId": "ord-8821",
  "bouquetId": "b3f1a2c4-...",
  "bouquetImageUrl": "https://cdn.bloomplatform.pl/bouquets/b3f1a2c4/full.jpg",
  "vendorId": "vend-42",
  "vendorName": "Flora Elegance",
  "bouquetPrice": 89.00,
  "deliveryFee": 12.00,
  "totalAmount": 101.00,
  "currency": "PLN",
  "status": "in_delivery",
  "deliveryMethod": "delivery",
  "deliveryAddress": {
    "street": "ul. Marszałkowska 10",
    "city": "Warsaw",
    "postalCode": "00-001",
    "notes": "3rd floor, flat 12"
  },
  "pickupAddress": null,
  "estimatedDeliveryAt": "2025-04-10T11:00:00Z",
  "deliveryQrCode": "data:image/png;base64,...",
  "timeline": [
    { "status": "created", "timestamp": "2025-04-10T10:10:00Z", "label": "Order Placed" },
    { "status": "paid", "timestamp": "2025-04-10T10:11:00Z", "label": "Payment Confirmed" },
    { "status": "preparing", "timestamp": "2025-04-10T10:20:00Z", "label": "Preparing" },
    { "status": "in_delivery", "timestamp": "2025-04-10T10:23:00Z", "label": "Out for Delivery" },
    { "status": "delivered", "timestamp": null, "label": "Delivered" }
  ],
  "createdAt": "2025-04-10T10:10:00Z"
}
```

**Response 200 (pickup order):**

```json
{
  "orderId": "ord-8822",
  "bouquetId": "b3f1a2c4-...",
  "bouquetImageUrl": "https://cdn.bloomplatform.pl/bouquets/b3f1a2c4/full.jpg",
  "vendorId": "vend-42",
  "vendorName": "Flora Elegance",
  "bouquetPrice": 89.00,
  "deliveryFee": 0.00,
  "totalAmount": 89.00,
  "currency": "PLN",
  "status": "paid",
  "deliveryMethod": "pickup",
  "deliveryAddress": null,
  "pickupAddress": {
    "street": "ul. Nowy Świat 15",
    "city": "Warsaw",
    "postalCode": "00-029",
    "vendorName": "Flora Elegance",
    "openingHours": "Mon-Sat 09:00-20:00"
  },
  "estimatedDeliveryAt": null,
  "deliveryQrCode": "data:image/png;base64,...",
  "timeline": [
    { "status": "created", "timestamp": "2025-04-10T10:10:00Z", "label": "Order Placed" },
    { "status": "paid", "timestamp": "2025-04-10T10:11:00Z", "label": "Payment Confirmed" },
    { "status": "ready_for_pickup", "timestamp": "2025-04-10T10:20:00Z", "label": "Ready for Pickup" },
    { "status": "delivered", "timestamp": null, "label": "Picked Up" }
  ],
  "createdAt": "2025-04-10T10:10:00Z"
}
```

**Order Status Values:**

| Status | Label (delivery) | Label (pickup) | Description |
|---|---|---|---|
| `created` | Order Placed | Order Placed | Order created, awaiting payment |
| `paid` | Payment Confirmed | Payment Confirmed | Stripe payment succeeded |
| `preparing` | Preparing | Preparing | Vendor is preparing the bouquet |
| `ready_for_pickup` | _(not used)_ | Ready for Pickup | Pickup only: bouquet ready at vendor location |
| `in_delivery` | Out for Delivery | _(not used)_ | Delivery only: courier has picked up the bouquet |
| `delivered` | Delivered | Picked Up | Customer has received the bouquet |
| `cancelled` | Cancelled | Cancelled | Order was cancelled (payment failed, timeout, or manual) |

**`deliveryQrCode` availability:**

| Order status | `deliveryQrCode` value |
|---|---|
| `created` | `null` — not yet generated |
| `paid` and later | Base64 PNG data URI — available for both delivery (courier verification) and pickup (shop verification) |
| `cancelled` | `null` |

**Notes:**
- `bouquetPrice` is always present as a separate field so the app can display the itemised cost breakdown.
- For pickup orders, `deliveryAddress` is `null` and `pickupAddress` contains the vendor's location. For delivery orders, `pickupAddress` is `null`.
- `deliveryFee` is `0.00` for pickup orders.
- `estimatedDeliveryAt` is `null` for pickup orders and for delivery orders before `in_delivery` status.

**Response 404:** Order not found or does not belong to the authenticated customer.

---

### POST /api/orders/{id}/report-issue

Reports an issue with a delivered order (TRK-08). **Reserved for post-MVP.**

**Auth:** Required

**Request Body:**

```json
{
  "issueType": "damaged",
  "description": "Several stems were broken on arrival."
}
```

`issueType` values: `damaged`, `wrong_bouquet`, `late_delivery`, `other`

**Response 201:**

```json
{
  "issueId": "iss-001",
  "orderId": "ord-8821",
  "status": "submitted",
  "createdAt": "2025-04-10T14:00:00Z"
}
```

**Response 400:** Order is not in `delivered` status.
**Response 404:** Order not found.

> **Note:** This endpoint is planned for post-MVP. The mobile app should hide the "Report Issue" UI element until this endpoint is available.

---

## 6. Subscription Endpoints

### GET /api/subscriptions

Returns all subscriptions for the authenticated customer.

**Auth:** Required

**Query Parameters:**

| Param | Type | Required | Description |
|---|---|---|---|
| `page` | int | No | Page number (default: `1`) |
| `pageSize` | int | No | Items per page (default: `20`, max: `50`) |

**Response 200:**

```json
{
  "items": [
    {
      "subscriptionId": "sub-001",
      "type": "shop",
      "isActive": true,
      "shopId": "vend-42",
      "shopName": "Flora Elegance",
      "notificationFrequency": "immediate",
      "createdAt": "2025-03-01T10:00:00Z"
    },
    {
      "subscriptionId": "sub-002",
      "type": "flower_type",
      "isActive": true,
      "flowerType": "rose",
      "flowerTypeLabel": "Roses",
      "radiusKm": 5,
      "notificationFrequency": "daily",
      "createdAt": "2025-03-15T14:30:00Z"
    },
    {
      "subscriptionId": "sub-003",
      "type": "price_range",
      "isActive": true,
      "maxPrice": 50.00,
      "currency": "PLN",
      "radiusKm": 10,
      "notificationFrequency": "weekly",
      "createdAt": "2025-03-20T08:00:00Z"
    },
    {
      "subscriptionId": "sub-004",
      "type": "combined",
      "isActive": true,
      "flowerType": "rose",
      "flowerTypeLabel": "Roses",
      "maxPrice": 60.00,
      "currency": "PLN",
      "radiusKm": 5,
      "notificationFrequency": "immediate",
      "createdAt": "2025-03-25T10:00:00Z"
    }
  ],
  "totalCount": 4,
  "page": 1,
  "pageSize": 20
}
```

---

### GET /api/subscriptions/{id}

Returns a single subscription by ID.

**Auth:** Required

**Response 200:** Single subscription object (same schema as items in `GET /api/subscriptions`).

**Response 404:** Subscription not found or does not belong to the customer.

---

### POST /api/subscriptions

Creates a new subscription.

**Auth:** Required
**Request Body (by type):**

Shop subscription:
```json
{
  "type": "shop",
  "shopId": "vend-42",
  "notificationFrequency": "immediate"
}
```

Flower type subscription:
```json
{
  "type": "flower_type",
  "flowerType": "rose",
  "radiusKm": 5,
  "notificationFrequency": "immediate"
}
```

Price range subscription:
```json
{
  "type": "price_range",
  "maxPrice": 50.00,
  "radiusKm": 10,
  "notificationFrequency": "weekly"
}
```

Combined subscription (flower type + price range):
```json
{
  "type": "combined",
  "flowerType": "rose",
  "maxPrice": 60.00,
  "radiusKm": 5,
  "notificationFrequency": "immediate"
}
```

| Field | Rules |
|---|---|
| `type` | `shop`, `flower_type`, `price_range`, `combined` |
| `notificationFrequency` | `immediate`, `daily`, `weekly` |
| `radiusKm` | `5`, `10`, or `15` (required for `flower_type`, `price_range`, `combined`) |
| `flowerType` | Required for `flower_type` and `combined` — use values from `GET /api/reference/flower-types` |
| `shopId` | Required for `shop` |
| `maxPrice` | Required for `price_range` and `combined` |

**Response 201:** Full subscription object (same as GET schema)
**Response 400:** Invalid parameters or missing required fields.

---

### PUT /api/subscriptions/{id}

Updates an existing subscription (toggle active, change frequency).

**Auth:** Required
**Request Body:** Partial subscription object (only changed fields required)

```json
{
  "isActive": false,
  "notificationFrequency": "daily"
}
```

**Response 200:** Updated subscription object
**Response 404:** Subscription not found or does not belong to the customer.

---

### PUT /api/subscriptions/disable-all

Deactivates all subscriptions for the authenticated customer. Sets `isActive: false` on every subscription without deleting them (SUB-09).

**Auth:** Required

**Response 200:**

```json
{
  "disabledCount": 4,
  "message": "All subscriptions have been deactivated."
}
```

---

### DELETE /api/subscriptions/{id}

Deletes a subscription.

**Auth:** Required
**Response 204:** No content
**Response 404:** Subscription not found or does not belong to the customer.

---

## 7. Vendor Endpoints

### GET /api/vendors/{id}

Returns public vendor profile.

**Auth:** Optional

**Response 200:**

```json
{
  "vendorId": "vend-42",
  "vendorType": "shop",
  "name": "Flora Elegance",
  "description": "Warsaw's finest fresh bouquets since 2012.",
  "openingHours": "Mon-Sat 09:00-20:00",
  "isCurrentlyOpen": true,
  "location": {
    "latitude": 52.2297,
    "longitude": 21.0122,
    "displayAddress": "ul. Nowy Świat 15, Warsaw",
    "exactAddressHidden": false
  },
  "activeBouquetCount": 3,
  "rating": 4.8
}
```

**Notes:**
- For freelance florists, `exactAddressHidden` is `true` and coordinates are approximate (within ~500 m).
- `isCurrentlyOpen` is computed server-side from `openingHours`. Returns `null` if opening hours are not configured.

**Response 404:** Vendor not found.

---

### GET /api/vendors/{id}/bouquets

Returns all currently available bouquets for a specific vendor.

**Auth:** Optional

**Query Parameters:**

| Param | Type | Required | Description |
|---|---|---|---|
| `page` | int | No | Page number (default: `1`) |
| `pageSize` | int | No | Items per page (default: `20`, max: `50`) |

**Response 200:** Same paginated schema as `GET /api/bouquets/nearby` items.

---

## 8. Customer Profile Endpoints

### GET /api/profile

Returns the authenticated customer's profile.

**Auth:** Required

**Response 200:**

```json
{
  "customerId": "cust-123",
  "firstName": "Jan",
  "lastName": "Kowalski",
  "email": "jan@example.com",
  "emailVerified": true,
  "memberSince": "2025-01-15T00:00:00Z",
  "stats": {
    "totalOrders": 12,
    "activeSubscriptions": 3,
    "totalSpent": 1240.00,
    "currency": "PLN"
  },
  "preferredFlowerTypes": ["rose", "tulip"],
  "notificationSettings": {
    "pushEnabled": true,
    "emailEnabled": true,
    "subscriptionDigestEnabled": true
  }
}
```

---

### PUT /api/profile

Updates the customer's profile.

**Auth:** Required

**Request Body:** Partial object — only changed fields required.

```json
{
  "firstName": "Jan",
  "lastName": "Kowalski",
  "preferredFlowerTypes": ["rose", "tulip", "peony"]
}
```

**Response 200:** Updated profile object.

---

### PUT /api/profile/notification-settings

Updates notification preferences.

**Auth:** Required

**Request Body:**

```json
{
  "pushEnabled": true,
  "emailEnabled": false,
  "subscriptionDigestEnabled": true
}
```

**Response 200:** Updated notification settings object.

**Notes:**
- Setting `pushEnabled: false` stops all push notifications from being delivered to the device but does **not** deactivate individual subscriptions. To deactivate all subscriptions, use `PUT /api/subscriptions/disable-all`.
- Setting `subscriptionDigestEnabled: false` stops daily/weekly digest emails for subscriptions with `notificationFrequency` set to `daily` or `weekly`.

---

### DELETE /api/profile

Requests account deletion (GDPR). Marks the account for deletion; completed within 30 days.

**Auth:** Required

**Request Body:**

```json
{
  "confirmationText": "DELETE"
}
```

**Response 202:**

```json
{
  "message": "Account deletion requested. Your data will be removed within 30 days.",
  "scheduledDeletionDate": "2025-05-10T00:00:00Z"
}
```

---

## 9. Saved Addresses Endpoints

Saved addresses are a small, bounded collection (practical limit: 10 per customer). They return a plain array, not the paginated envelope.

### GET /api/profile/addresses

Returns the customer's saved delivery addresses.

**Auth:** Required

**Response 200:**

```json
[
  {
    "addressId": "addr-001",
    "label": "Home",
    "street": "ul. Marszałkowska 10",
    "city": "Warsaw",
    "postalCode": "00-001",
    "notes": "3rd floor, flat 12",
    "isDefault": true
  }
]
```

---

### POST /api/profile/addresses

Creates a new saved address. Maximum 10 addresses per customer.

**Auth:** Required

**Request Body:**

```json
{
  "label": "Work",
  "street": "ul. Złota 59",
  "city": "Warsaw",
  "postalCode": "00-120",
  "notes": "Reception desk",
  "isDefault": false
}
```

**Response 201:** Full address object.
**Response 409:** Maximum address limit (10) reached.

---

### PUT /api/profile/addresses/{id}

Updates a saved address.

**Auth:** Required
**Request Body:** Partial address object.
**Response 200:** Updated address object.

---

### DELETE /api/profile/addresses/{id}

Deletes a saved address.

**Auth:** Required
**Response 204:** No content.

---

## 10. Push Notifications Endpoints

### POST /api/notifications/device

Registers a device for push notifications (Firebase Cloud Messaging for Android, APNs for iOS).

**Auth:** Required

**Request Body:**

```json
{
  "deviceToken": "firebase-or-apns-token-...",
  "platform": "android",
  "appVersion": "1.0.0"
}
```

| Field | Rules |
|---|---|
| `platform` | `android` or `ios` |
| `deviceToken` | Required, FCM registration token or APNs device token |

**Response 200:** Device registered / updated.

---

### DELETE /api/notifications/device

Unregisters the current device from push notifications (e.g., on sign-out).

**Auth:** Required

**Request Body:**

```json
{
  "deviceToken": "firebase-or-apns-token-..."
}
```

**Response 204:** No content.

---

### GET /api/notifications

Returns the in-app notification feed for the authenticated customer.

**Auth:** Required

**Query Parameters:**

| Param | Type | Required | Description |
|---|---|---|---|
| `page` | int | No | Page number (default: `1`) |
| `pageSize` | int | No | Items per page (default: `20`, max: `50`) |
| `unreadOnly` | bool | No | Filter to unread only (default: `false`) |

**Response 200:**

```json
{
  "items": [
    {
      "notificationId": "notif-001",
      "type": "subscription_match",
      "title": "New roses available!",
      "body": "Flora Elegance has fresh roses for 89.00 PLN",
      "bouquetId": "b3f1a2c4-...",
      "orderId": null,
      "subscriptionId": "sub-002",
      "newStatus": null,
      "isRead": false,
      "createdAt": "2025-04-10T09:20:00Z"
    },
    {
      "notificationId": "notif-002",
      "type": "order_update",
      "title": "Order out for delivery",
      "body": "Your order ord-8821 is on its way!",
      "bouquetId": null,
      "orderId": "ord-8821",
      "subscriptionId": null,
      "newStatus": "in_delivery",
      "isRead": true,
      "createdAt": "2025-04-10T10:23:00Z"
    }
  ],
  "totalCount": 15,
  "unreadCount": 3,
  "page": 1,
  "pageSize": 20
}
```

**Notification fields by type:**

| Field | `subscription_match` | `order_update` | `promotion` | `system` |
|---|---|---|---|---|
| `bouquetId` | present | `null` | present or `null` | `null` |
| `orderId` | `null` | present | `null` | `null` |
| `subscriptionId` | present | `null` | `null` | `null` |
| `newStatus` | `null` | present | `null` | `null` |

---

### Push Notification Payload (FCM / APNs)

Push notifications sent to the device include a `data` payload for deep linking (SUB-07). The mobile app should parse `data` to navigate to the correct screen.

**Subscription match (new bouquet):**

```json
{
  "notification": {
    "title": "New roses available!",
    "body": "Flora Elegance has fresh roses for 89.00 PLN"
  },
  "data": {
    "type": "subscription_match",
    "bouquetId": "b3f1a2c4-...",
    "subscriptionId": "sub-002"
  }
}
```

**Order status update:**

```json
{
  "notification": {
    "title": "Order out for delivery",
    "body": "Your order ord-8821 is on its way!"
  },
  "data": {
    "type": "order_update",
    "orderId": "ord-8821",
    "newStatus": "in_delivery"
  }
}
```

**Deep link mapping:**

| `data.type` | Navigate to | Key field |
|---|---|---|
| `subscription_match` | Bouquet detail screen | `data.bouquetId` |
| `order_update` | Order tracking screen | `data.orderId` |
| `promotion` | Promotions screen or bouquet detail | `data.bouquetId` (optional) |
| `system` | Notifications list | — |

---

### PUT /api/notifications/{id}/read

Marks a notification as read.

**Auth:** Required
**Response 204:** No content.

---

### PUT /api/notifications/read-all

Marks all notifications as read.

**Auth:** Required
**Response 204:** No content.

---

## 11. Reference Data Endpoints

### GET /api/reference/flower-types

Returns the list of supported flower types for subscriptions and filtering.

**Auth:** None
**Cacheable:** Yes — cache client-side for up to 24 hours (see section 19).

**Response 200:**

```json
[
  { "type": "rose", "label": "Roses" },
  { "type": "tulip", "label": "Tulips" },
  { "type": "peony", "label": "Peonies" },
  { "type": "lily", "label": "Lilies" },
  { "type": "sunflower", "label": "Sunflowers" },
  { "type": "eucalyptus", "label": "Eucalyptus" },
  { "type": "orchid", "label": "Orchids" },
  { "type": "hydrangea", "label": "Hydrangeas" },
  { "type": "chrysanthemum", "label": "Chrysanthemums" },
  { "type": "gerbera", "label": "Gerberas" }
]
```

---

### GET /api/reference/freshness-labels

Returns freshness score ranges and their display labels.

**Auth:** None
**Cacheable:** Yes — cache client-side for up to 24 hours (see section 19).

**Response 200:**

```json
[
  { "min": 80, "max": 100, "label": "Very Fresh", "color": "#22C55E" },
  { "min": 60, "max": 79, "label": "Fresh", "color": "#84CC16" },
  { "min": 40, "max": 59, "label": "Moderate", "color": "#EAB308" },
  { "min": 20, "max": 39, "label": "Declining", "color": "#F97316" },
  { "min": 0, "max": 19, "label": "Poor", "color": "#EF4444" }
]
```

---

## 12. Health Check Endpoint

### GET /api/health

Returns backend health status. Useful for mobile app to detect connectivity issues.

**Auth:** None

**Response 200:**

```json
{
  "status": "healthy",
  "version": "1.0.0",
  "timestamp": "2025-04-10T09:00:00Z"
}
```

**Response 503:** Service unavailable.

---

## 13. SignalR Hub

**Hub URL:** `{baseUrl}/hubs/bouquet`

### Authentication

The SignalR hub supports both authenticated and anonymous connections:

| Connection type | How to connect | Available groups |
|---|---|---|
| **Authenticated** | Pass `access_token` query string param | Radius groups, order groups |
| **Anonymous** | Connect without token | Radius groups only |

Anonymous connections can subscribe to `JoinRadiusGroup` to receive real-time bouquet availability updates (AUTH-04, DISC-04). Order-related groups (`JoinOrderGroup`) require authentication.

### Connection Setup (.NET MAUI)

```csharp
// Authenticated connection
var connection = new HubConnectionBuilder()
    .WithUrl($"{baseUrl}/hubs/bouquet", options =>
    {
        options.AccessTokenProvider = () => Task.FromResult(accessToken);
    })
    .WithAutomaticReconnect(new[] { TimeSpan.Zero, TimeSpan.FromSeconds(2), TimeSpan.FromSeconds(5), TimeSpan.FromSeconds(10), TimeSpan.FromSeconds(30) })
    .Build();

// Anonymous connection (unauthenticated user browsing the map)
var connection = new HubConnectionBuilder()
    .WithUrl($"{baseUrl}/hubs/bouquet")
    .WithAutomaticReconnect(new[] { TimeSpan.Zero, TimeSpan.FromSeconds(2), TimeSpan.FromSeconds(5), TimeSpan.FromSeconds(10), TimeSpan.FromSeconds(30) })
    .Build();
```

### Reconnection Behaviour

The hub automatically reconnects on network interruption (REL-02). The client must:
1. Re-join radius/order groups after reconnection via the `Reconnected` event handler.
2. Fetch latest data via REST to fill any events missed during disconnection.
3. Show a "Reconnecting..." indicator during the reconnection window.

```csharp
connection.Reconnected += async (connectionId) =>
{
    // Re-join groups after reconnect
    await connection.InvokeAsync("JoinRadiusGroup", lastLatitude, lastLongitude, lastRadiusKm);
    // Refresh bouquet data via REST to catch missed events
    await RefreshNearbyBouquets();
};
```

### Client → Server

#### JoinRadiusGroup

Subscribe to real-time updates for a location and radius. Available to both authenticated and anonymous connections.

```csharp
await connection.InvokeAsync("JoinRadiusGroup", latitude, longitude, radiusKm);
```

#### LeaveRadiusGroup

Unsubscribe from radius updates.

```csharp
await connection.InvokeAsync("LeaveRadiusGroup");
```

#### JoinOrderGroup

Subscribe to updates for a specific order. **Requires authentication.**

```csharp
await connection.InvokeAsync("JoinOrderGroup", orderId);
```

#### LeaveOrderGroup

Unsubscribe from order updates.

```csharp
await connection.InvokeAsync("LeaveOrderGroup", orderId);
```

---

### Server → Client

#### BouquetBecameAvailable

A new bouquet is available within the subscribed radius. Payload matches the nearby endpoint item schema so the app can add a map pin without an extra REST call.

```json
{
  "bouquetId": "b3f1a2c4-...",
  "vaseId": "v-001",
  "vendorId": "vend-42",
  "vendorName": "Flora Elegance",
  "vendorType": "shop",
  "price": 89.00,
  "currency": "PLN",
  "status": "available",
  "freshnessScore": 84,
  "freshnessLabel": "Very Fresh",
  "thumbnailUrl": "https://cdn.bloomplatform.pl/...",
  "location": {
    "latitude": 52.2297,
    "longitude": 21.0122,
    "displayAddress": "Śródmieście, Warsaw",
    "exactAddressHidden": false
  },
  "distanceKm": 0.8,
  "availableFrom": "2025-04-10T09:15:00Z"
}
```

**Notes:**
- `distanceKm` is computed using the coordinates the client sent in `JoinRadiusGroup`.
- The event uses the same nested `location` structure as the REST response.

#### BouquetBecameUnavailable

A bouquet is no longer available (sold, reserved, vase offline).

```json
{
  "bouquetId": "b3f1a2c4-...",
  "vendorType": "shop",
  "reason": "reserved"
}
```

`reason` values: `sold`, `reserved`, `vase_offline`, `removed`

**Notes:**
- `vendorType` is included so the app can correctly update filtered views (DISC-11).

#### FreshnessUpdated

A bouquet's freshness score has changed (pushed periodically by the vase sensors).

```json
{
  "bouquetId": "b3f1a2c4-...",
  "freshnessScore": 78,
  "freshnessLabel": "Fresh"
}
```

#### OrderStatusChanged

An order's status has changed.

```json
{
  "orderId": "ord-8821",
  "newStatus": "in_delivery",
  "label": "Out for Delivery",
  "estimatedDeliveryAt": "2025-04-10T11:00:00Z"
}
```

#### DeliveryLocationUpdated

Real-time delivery courier location update (sent while order status is `in_delivery`).

```json
{
  "orderId": "ord-8821",
  "courierLatitude": 52.2310,
  "courierLongitude": 21.0145,
  "estimatedMinutesRemaining": 8
}
```

---

## 14. Error Schema

All API errors follow the [RFC 7807](https://tools.ietf.org/html/rfc7807) Problem Details format:

```json
{
  "type": "https://bloomplatform.pl/errors/bouquet-not-found",
  "title": "Bouquet Not Found",
  "status": 404,
  "detail": "Bouquet 'b3f1a2c4-...' does not exist or is no longer available.",
  "traceId": "00-abc123-def456-00"
}
```

### Validation Error Extension

For 400 validation errors, the response includes field-level details:

```json
{
  "type": "https://bloomplatform.pl/errors/validation-error",
  "title": "Validation Error",
  "status": 400,
  "detail": "One or more validation errors occurred.",
  "traceId": "00-abc123-def456-00",
  "errors": {
    "email": ["Email is required.", "Email format is invalid."],
    "password": ["Password must be at least 8 characters."]
  }
}
```

### Price Changed Error Extension

For 409 price-changed errors (on `POST /api/orders`), the response includes the current price:

```json
{
  "type": "https://bloomplatform.pl/errors/price-changed",
  "title": "Price Changed",
  "status": 409,
  "detail": "The bouquet price has changed since your preview.",
  "traceId": "00-abc123-def456-00",
  "currentTotalAmount": 105.00,
  "currency": "PLN"
}
```

### Common Error Codes

| HTTP Status | type | Meaning |
|---|---|---|
| 400 | `validation-error` | Request body or query params invalid |
| 401 | `unauthorized` | Missing or invalid access token |
| 402 | `payment-failed` | Payment was declined or failed |
| 403 | `forbidden` | Token valid but insufficient permissions |
| 404 | `not-found` | Resource does not exist |
| 409 | `conflict` | Bouquet already reserved; concurrent order conflict; duplicate resource |
| 409 | `price-changed` | Price changed between preview and order creation |
| 410 | `gone` | QR code bouquet no longer available |
| 429 | `rate-limited` | Too many requests — see `Retry-After` header |
| 500 | `internal-error` | Unexpected server error |

---

## 15. Rate Limiting

| Scope | Limit | Window |
|---|---|---|
| Anonymous endpoints | 60 requests | per minute per IP |
| Authenticated endpoints | 120 requests | per minute per user |
| Order creation | 5 requests | per minute per user |
| Auth registration | 3 requests | per minute per IP |
| Resend verification | 1 request | per 60 seconds per user |

When rate-limited, the response includes:

| Header | Description |
|---|---|
| `Retry-After` | Seconds until the limit resets |
| `X-RateLimit-Limit` | Maximum requests in the window |
| `X-RateLimit-Remaining` | Remaining requests in the window |

---

## 16. Pagination Convention

List endpoints use one of two response formats:

### Paginated envelope

Used for potentially large or growing collections:

```json
{
  "items": [ ... ],
  "totalCount": 42,
  "page": 1,
  "pageSize": 20
}
```

| Param | Default | Max |
|---|---|---|
| `page` | 1 | — |
| `pageSize` | 20 | 50 |

**Endpoints using paginated envelope:** bouquets/nearby, favourites, orders, subscriptions, notifications.

### Plain array

Used for small, bounded collections where pagination is unnecessary:

**Endpoints using plain array:** profile/addresses (max 10), reference/flower-types, reference/freshness-labels.

---

## 17. HTTP Headers Convention

### Request Headers

| Header | Required | Description |
|---|---|---|
| `Authorization` | For auth endpoints | `Bearer <access_token>` |
| `Content-Type` | For POST/PUT | `application/json` |
| `Accept-Language` | Optional | `pl-PL` (default) or `en-US` |
| `X-App-Version` | Recommended | Mobile app version, e.g. `1.0.0` |
| `X-Platform` | Recommended | `android` or `ios` |
| `Idempotency-Key` | For POST /api/orders | Client-generated UUID v4 |

### Response Headers

| Header | Description |
|---|---|
| `X-Request-Id` | Unique request identifier for debugging |
| `X-RateLimit-Limit` | Rate limit ceiling |
| `X-RateLimit-Remaining` | Remaining requests in window |
| `Retry-After` | Seconds to wait (only on 429) |
| `Cache-Control` | Caching directive (see section 19) |
| `ETag` | Entity tag for conditional requests (see section 19) |

---

## 18. Idempotency

Non-idempotent `POST` endpoints that create resources or have side effects require an `Idempotency-Key` header to safely support retries (REL-04).

| Endpoint | Idempotent | Retry Safe | Notes |
|---|---|---|---|
| `POST /api/auth/register` | No | No | Duplicate returns 409 naturally |
| `POST /api/auth/resend-verification` | Yes | Yes | Rate-limited; no side effects on repeat |
| `POST /api/orders/preview` | Yes | Yes | Read-only, no side effects |
| `POST /api/orders` | No | **Yes with `Idempotency-Key`** | Same key returns cached 201 response |
| `POST /api/orders/{id}/confirm-payment` | Yes | Yes | Duplicate confirmation returns same 200 |
| `POST /api/orders/{id}/cancel` | Yes | Yes | Duplicate cancellation returns 409 harmlessly |
| `POST /api/subscriptions` | No | No | Duplicate returns 409 if identical |
| `POST /api/favourites/{id}` | Yes | Yes | Duplicate returns 409 harmlessly |
| `POST /api/profile/addresses` | No | No | May create duplicates — client should check |
| `POST /api/notifications/device` | Yes | Yes | Upserts by device token |
| All `GET` endpoints | Yes | Yes | Always safe |
| All `PUT` endpoints | Yes | Yes | Same payload produces same result |
| All `DELETE` endpoints | Yes | Yes | Deleting already-deleted returns 204/404 |

**Usage:**

```
POST /api/orders
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json
```

The server caches the response for a given `Idempotency-Key` for 24 hours. If the same key is sent again, the server returns the cached response without creating a new order.

---

## 19. Caching & Offline Support

The backend includes caching headers to support offline mode (REL-01) and performance targets (PERF-02, PERF-03).

### Cache-Control Policy

| Endpoint | `Cache-Control` | `ETag` | Offline Strategy |
|---|---|---|---|
| `GET /api/config` | `public, max-age=3600` | No | Cache for 1h; serve stale if offline |
| `GET /api/reference/flower-types` | `public, max-age=86400` | Yes | Cache for 24h; serve stale if offline |
| `GET /api/reference/freshness-labels` | `public, max-age=86400` | Yes | Cache for 24h; serve stale if offline |
| `GET /api/vendors/{id}` | `public, max-age=3600` | Yes | Cache for 1h; serve stale if offline |
| `GET /api/bouquets/{id}` | `private, max-age=60` | Yes | Short cache; show stale with banner if offline |
| `GET /api/bouquets/nearby` | `private, no-cache` | No | Cache last result; show with "May be outdated" banner |
| `GET /api/orders` | `private, no-cache` | No | Cache last result for offline viewing |
| `GET /api/orders/{id}` | `private, max-age=30` | Yes | Short cache; show stale with banner |
| `GET /api/profile` | `private, max-age=300` | Yes | Cache for 5 min |
| `GET /api/health` | `no-store` | No | Never cache |
| CDN images (`thumbnailUrl`, `imageUrl`) | `public, max-age=604800` | No | Cache for 7 days |

### Conditional Requests

For endpoints that return `ETag`, the mobile app should send `If-None-Match` on subsequent requests:

```
GET /api/bouquets/b3f1a2c4-...
If-None-Match: "abc123"
```

**Response 304:** Not Modified — use cached version (saves bandwidth).

### Offline Mode Behaviour

When the device is offline, the app should:
1. Serve cached data for all previously visited screens.
2. Display a persistent "You are offline" banner at the top of the screen.
3. Mark all cached bouquet data with "May be outdated" indicators.
4. Disable actions that require network: checkout, favourites toggle, subscription changes.
5. Queue notification read/mark operations and sync when back online.

---

## 20. Localisation

The backend supports content localisation via the `Accept-Language` request header (I18N-01, I18N-04).

### Supported Locales

| Locale | Status |
|---|---|
| `pl-PL` | Default — returned if no `Accept-Language` header is sent |
| `en-US` | Secondary language for MVP |

### Localised Fields

The following response fields are returned in the requested language:

| Field | Endpoints | Example (pl-PL) | Example (en-US) |
|---|---|---|---|
| `freshnessLabel` | bouquets/nearby, bouquets/{id}, favourites | "Bardzo Świeży" | "Very Fresh" |
| Timeline `label` | orders/{id} | "Zamówienie Złożone" | "Order Placed" |
| Error `title` | all errors | "Bukiet Nie Znaleziony" | "Bouquet Not Found" |
| Error `detail` | all errors | (Polish message) | (English message) |
| Flower type `label` | reference/flower-types | "Róże" | "Roses" |
| Freshness `label` | reference/freshness-labels | "Bardzo Świeży" | "Very Fresh" |
| Notification `title` and `body` | notifications, push payloads | (Polish) | (English) |
| Bouquet status display label | (client-side mapping) | "Dostępny" | "Available" |

### Non-Localised Fields

The following fields are always returned as-is, regardless of locale:

- All identifiers (`bouquetId`, `orderId`, etc.)
- Status values (`available`, `in_delivery`, etc.) — these are enum keys, not display text
- Enum/type values (`shop`, `freelance`, `rose`, etc.)
- Vendor-authored content (`vendorName`, `description`, `vendorOpeningHours`) — stored in the vendor's language
- Monetary values and currency codes
- Coordinates, dates, URLs
