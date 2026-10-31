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
| Production | `https://api.findmyflowers.pl` |

All REST endpoints are prefixed with `/api/v1` (the API is versioned — controllers route on `api/v{version:apiVersion}/...` with the current version `1.0`). Dates use ISO 8601 UTC format. Monetary amounts are decimals with 2 fractional digits.

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
| Production | `https://auth.findmyflowers.pl` |

### OIDC Client Configuration

| Parameter | Value |
|---|---|
| `client_id` | `fmf-mobile` |
| `response_type` | `code` |
| `scope` | `openid profile email offline_access` |
| `code_challenge_method` | `S256` |
| `redirect_uri` | `pl.findmyflowers.app://auth/callback` |
| `post_logout_redirect_uri` | `pl.findmyflowers.app://auth/logout-callback` |

**Google OAuth (AUTH-03):** Google social login is **live** as an Identity Provider in the `flowershop` Keycloak realm (committed export: `docker/keycloak/flowershop-realm.json`). The mobile app uses the standard OIDC Authorization Code + PKCE flow — Keycloak renders the Google login button on its hosted login page, so no additional client-side Google SDK integration or app code change is required. Google emails are trusted as verified, so no separate email-verification step applies, and a Google login is auto-linked to any existing account with the same verified email. On the first authenticated API call, the backend provisions the local `Customer` and the `customer_id` claim just-in-time (`CustomerProvisioningService`), so all `CustomerAccess` endpoints work immediately without a re-login.

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

1. **Unregister device** — `DELETE /api/v1/notifications/devices/{deviceToken}` with the current FCM/APNs token in the path.
2. **End Keycloak session** — redirect to Keycloak End Session endpoint with `id_token_hint` and `post_logout_redirect_uri`.
3. **Clear local state** — remove access token, refresh token, and ID token from memory and secure storage. Clear any cached data.

```
GET {keycloakBase}/realms/flowershop/protocol/openid-connect/logout
    ?id_token_hint=<id_token>
    &post_logout_redirect_uri=pl.findmyflowers.app://auth/logout-callback
```

---

## 1. Auth & Registration Endpoints

### POST /api/v1/customers/register

Creates a new customer account atomically (Keycloak user + local `Customer`/`User` record) and returns a JWT so the app can proceed without a separate login. The account can be used immediately; email verification is handled by Keycloak out-of-band (see below).

**Auth:** None

**Request Body (`CustomerRegisterRequest`):**

```json
{
  "username": "jan.kowalski",
  "email": "jan@example.com",
  "password": "SecureP@ss1",
  "customerName": "Jan Kowalski",
  "preferredVendorType": "shop",
  "phoneNumber": "+48123456789"
}
```

**Validation Rules:**

| Field | Rules |
|---|---|
| `username` | Required, unique |
| `email` | Valid email, unique, max 255 chars |
| `password` | Min 8 chars, at least 1 uppercase, 1 lowercase, 1 digit, 1 special char |
| `customerName` | Required — customer display name |
| `preferredVendorType` | Optional — `shop` or `freelance` |
| `phoneNumber` | Optional |

**Response 201 (`AuthResponse`):**

```json
{
  "token": "eyJhbGciOi...",
  "tokenType": "Bearer",
  "expiresIn": 86400,
  "username": "jan.kowalski"
}
```

**Response 400:** Registration failed (e.g. username/email already registered) — returned as a Problem Details object.

**Email Verification:** Handled entirely by Keycloak. The verification link in the email points to Keycloak, which marks the account as verified. The mobile app checks `email_verified` in the JWT claims and displays a reminder banner if `false`. No custom verify endpoint is needed.

---

### POST /api/v1/auth/resend-verification

Triggers Keycloak to resend the email verification link. Rate-limited (policy `auth-email`) to prevent abuse.

**Auth:** Required (user must be logged in but unverified)

**Response 202:** Verification email queued by Keycloak.
**Response 204:** No content — the user is already verified (treated as an idempotent success).
**Response 429:** Rate limited — max 1 request per 60 seconds.
**Response 503:** Keycloak could not send the email (e.g. SMTP misconfigured) — Problem Details.

---

### GET /api/v1/auth/me

Returns the caller's sign-in account read **live from Keycloak** (not from JWT claims), so email changes and verification are visible without re-login. Used by the Vendor Portal verification banner and Profile page.

**Auth:** Required (any role)

**Response 200:**
```json
{
  "username": "rosegardenadmin",
  "email": "owner@example.com",
  "emailVerified": false
}
```

**Response 401:** Missing/invalid token.
**Response 404:** No Keycloak user matches the token `sub` (e.g. dev tokens without Keycloak).
**Response 503:** Keycloak unreachable — Problem Details. Clients should fall back to JWT claims.

---

### POST /api/v1/auth/password-reset/request

Initiates a password reset flow. Keycloak sends the reset email; the user completes the reset on Keycloak's hosted page (there is no backend `confirm` endpoint). Rate-limited (policy `auth-email`).

**Auth:** None

**Request Body:**

```json
{
  "email": "jan@example.com"
}
```

**Response 202:** Always returned (does not reveal whether email exists).

---

### POST /api/v1/auth/logout

Revokes the caller's current JWT so it cannot be reused after logout. The token's `jti` claim is added to a server-side blacklist until the token's natural expiry.

**Auth:** Required

**Response 204:** Token revoked.
**Response 400:** Token has no `jti` claim.

---

## 2. App Configuration Endpoint

### GET /api/v1/config

Returns environment-specific configuration the mobile app needs at startup. Called once on app launch and cached for the session.

**Auth:** None

**Response 200 (`AppConfigDto`):**

```json
{
  "stripePublishableKey": "pk_live_...",
  "supportEmail": "contact@findmyflowers.pl",
  "termsUrl": "https://findmyflowers.pl/terms",
  "privacyPolicyUrl": "https://app.findmyflowers.pl/Home/Privacy",
  "minAppVersion": "1.0.0",
  "forceUpdate": false
}
```

| Field | Description |
|---|---|
| `stripePublishableKey` | Stripe publishable key for PaymentSheet initialization |
| `supportEmail` | Customer support contact |
| `termsUrl` | Link to current Terms of Service |
| `privacyPolicyUrl` | Link to current Privacy Policy |
| `minAppVersion` | Minimum supported app version; app should show an update prompt if current version is lower |
| `forceUpdate` | When `true`, the app must block usage until updated (hard force-update); when `false`, the update prompt is advisory |

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

### GET /api/v1/bouquets/nearby

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
      "initialFreshnessDays": 3,
      "currentFreshnessDays": 2,
      "maxFreshnessScale": 3,
      "thumbnailUrl": "https://api.findmyflowers.pl/bouquets/b3f1a2c4/thumb.jpg",
      "location": {
        "latitude": 52.2297,
        "longitude": 21.0122,
        "displayAddress": "Śródmieście, Warsaw",
        "exactAddressHidden": false
      },
      "distanceKm": 0.8,
      "availableFrom": "2025-04-10T09:15:00Z",
      "isFavourite": false,
      "listingChannel": "vase",
      "deliveryOnly": false
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
- **Online listings (EP-39, §25):** `listingChannel` is `"vase"` or `"online"`; `deliveryOnly` is `true` for online
  listings. They have no vase (`vaseId: null`), are fuzzed like freelance florists (`exactAddressHidden: true`, also for
  any online-only vendor) and are returned only when the customer is within delivery range
  (`min(radiusKm, Delivery:MaxDistanceKm)`); show the distance as "delivers to you", not "walk here". The map lists
  only bouquets a customer can buy: vase-less catalog bouquets are not returned. The same two fields are on
  `GET /bouquets/{id}/detail` and `GET /bouquets/available`.
- **Freshness visibility (EP-25):** `initialFreshnessDays` is the vendor-declared rating
  (1–`maxFreshnessScale`, where `maxFreshnessScale` is always 3), declared when the vendor sets the
  price. `currentFreshnessDays` is the value after daily decay — the initial rating minus one per full
  day since the rating was declared (not since the bouquet was created; the two usually differ),
  floored at 0. Render it as `maxFreshnessScale` flower pictograms with `currentFreshnessDays` coloured
  and the rest greyed (e.g. `2` of `3` → two coloured + one grey). All three fields are `null`/absent when
  the vendor did not declare a rating. This is a display-only signal and is independent of `freshnessScore`
  (the 0–100 sensor reading) and of price — freshness-based discounts are handled separately by the vendor
  pricing strategy and never driven by these fields.
- **`freshnessDays` semantics changed (EP-25):** on the vendor-facing catalog read models
  (`AvailableBouquetReadModel`, `BouquetDetails`) the pre-existing `freshnessDays` field now carries the
  vendor-declared rating (0 when unset), replacing its former meaning of estimated age/shelf-life in days.
  Customer-facing endpoints are unaffected — use `initialFreshnessDays`/`currentFreshnessDays` there.

**Response 400:** Invalid coordinates or radius value.

**Geocoding Note (DISC-09):** This endpoint requires numeric `lat`/`lng` coordinates. When the user denies location permission and enters an address manually, the mobile app must geocode the address client-side using the **Google Places SDK** (the same SDK used for delivery address autocomplete in CHK-04) before calling this endpoint. The backend does not provide a geocoding endpoint.

---

### GET /api/v1/bouquets/{id}/detail

Returns full detail for a single bouquet (customer-facing `BouquetDetailDto`). Note the `/detail` suffix — the plain `GET /api/v1/bouquets/{id}` route is the **vendor** endpoint and returns a different DTO; the mobile client must use `/detail`.

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
  "initialFreshnessDays": 3,
  "currentFreshnessDays": 2,
  "maxFreshnessScale": 3,
  "imageUrl": "https://api.findmyflowers.pl/bouquets/b3f1a2c4/full.jpg",
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

### GET /api/v1/bouquets/qr/{code}

Resolves a bouquet from a scanned QR code.

**Auth:** Optional (unauthenticated users can scan, prompted to login before purchase)

**Path Parameters:** `code` — 8-character alphanumeric short code (e.g., `A3F8K2M1`) or full UUID. Both formats are accepted.

**QR payload format:** Smart vases now display a **full URL** built by the backend (`VaseQrUrl.ForBouquet`), not a short `/b/{code}` link:

```
{CustomerApp:BaseUrl}/Bouquets/Details/{bouquetId}?addToCart=true&vaseId={deviceId}
```

Scanning it opens the **customer web app** bouquet Details page (image + price + Add-to-Cart), which auto-adds the bouquet to the cart because of `addToCart=true`. The `{CustomerApp:BaseUrl}` comes from backend config so the domain can change without reflashing devices. The mobile app can still resolve a bouquet programmatically by extracting the `{bouquetId}` (or a short code) and calling this endpoint, which returns `BouquetDetailDto`.

**Response 200 (`BouquetDetailDto`):** Same schema as `GET /api/v1/bouquets/{id}/detail`
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

### GET /api/v1/favourites

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
      "initialFreshnessDays": 3,
      "currentFreshnessDays": 2,
      "maxFreshnessScale": 3,
      "thumbnailUrl": "https://api.findmyflowers.pl/bouquets/b3f1a2c4/thumb.jpg",
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

### POST /api/v1/favourites/{bouquetId}

Adds a bouquet to the customer's favourites. **Idempotent** — favouriting an already-favourited bouquet succeeds.

**Auth:** Required
**Response 204:** No content (added, or already present).
**Response 404:** Bouquet does not exist.

---

### DELETE /api/v1/favourites/{bouquetId}

Removes a bouquet from the customer's favourites. Idempotent — removing a non-existent favourite succeeds.

**Auth:** Required
**Response 204:** No content.

---

## 5. Order Endpoints

### POST /api/v1/orders/preview

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
      "bouquetImageUrl": "https://api.findmyflowers.pl/bouquets/b3f1a2c4/full.jpg",
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
      "bouquetImageUrl": "https://api.findmyflowers.pl/bouquets/b3f1a2c4/full.jpg",
      "unitPrice": 89.00
    }
  ],
  "linesSubtotal": 89.00,
  "deliveryFee": 0.00,
  "totalAmount": 89.00,
  "currency": "PLN",
  "vendorName": "Flora Elegance",
  "deliveryMethod": "pickup",
  "pickupAvailable": true
}
```

**Response 400/422:** See the delivery error table at the end of this section.

**Notes:**
- This endpoint does NOT reserve the bouquet. Reservation happens only on `POST /api/v1/orders`.
- `deliveryFee` is `0.00` when `deliveryMethod` is `pickup`; flat `25.00 PLN` when delivery falls within the vendor's 10 km service radius.
- `pickupAvailable` is `false` when any line is an online listing (EP-39, each such line has `deliveryOnly: true`):
  hide the Pickup choice and ask for a delivery address.
- `deliveryDistanceKm` and `resolvedDeliveryAddress` are present only on delivery previews. The server geocodes the address via Nominatim (cached for 10 minutes per address) and returns the resolved latitude/longitude so the client can surface a map preview.

---

### POST /api/v1/orders

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
- The Stripe publishable key is obtained from `GET /api/v1/config` (`stripePublishableKey` field).
- The app never handles raw card data (SEC-05).
- Supported payment methods: card, Apple Pay, Google Pay.

**Response 409 (already reserved):**

```json
{
  "type": "https://findmyflowers.pl/errors/conflict",
  "title": "Bouquet Already Reserved",
  "status": 409,
  "detail": "This bouquet has been reserved by another customer."
}
```

**Response 409 (price changed):**

```json
{
  "type": "https://findmyflowers.pl/errors/price-changed",
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
| `pickup-not-offered`        | 422  | `deliveryMethod=pickup` for an online listing (EP-39) — create only |
| `mixed-listing-channels`    | 422  | The order mixes online listings and vase bouquets (EP-39) — create only |
| `bouquet-not-listed`        | 409  | The bouquet is neither an online listing nor in a vase, so it can't be bought (EP-39; replaces "not placed in a vase") |
| `bouquet-unavailable`       | 409  | Another buyer took the bouquet first (EP-39 concurrency lock; `detail` names the bouquet) — create only |

All error responses use the shape `{ "error": "<code>" }`. `POST /api/v1/cart/items` returns 409
`{ "error": "bouquet-reserved", "message" }` when another customer holds the bouquet and 409 `bouquet-not-listed` for an
unlisted bouquet (EP-39; both were 400 before). Cart items and order lines carry `vaseId: null` for online listings.

**Side effects during the order lifecycle:**
- **On `POST /orders/{id}/confirm-payment` for a delivery order:** the order advances from `paid` → `preparing`, and the smart vase hosting the bouquet is sent an MQTT `prepare_delivery` card carrying `display_mode=delivery`, `delivery_status=PREPARING`, `delivery_qr=geo:<lat>,<lon>?q=<url-encoded-address>`, and `delivery_address=<street, postal city>`.
- **When the vendor transitions the order to `in_delivery`** (vendor portal `PUT /api/v1/vendor/orders/{id}/status`): the bouquet is unassigned from its vase (so the vase can host a new bouquet), and the vase receives an MQTT `delivery_started` card resetting it to `display_mode=idle` with banner "Waiting for bouquet".

---

### POST /api/v1/orders/{id}/confirm-payment

Confirms that the Stripe payment succeeded. Called after Stripe SDK reports success on the client. The server also receives a Stripe webhook independently; this endpoint ensures the order transitions promptly without waiting for the webhook.

**Auth:** Required
**Idempotent:** Yes — safe to retry. Duplicate confirmations for an already-paid order return 204.

**Request Body (`ConfirmPaymentRequest`):**

```json
{
  "paymentIntentId": "pi_stripe_...",
  "stripeCustomerId": "cus_..."
}
```

| Field | Rules |
|---|---|
| `paymentIntentId` | Required — the Stripe PaymentIntent id |
| `stripeCustomerId` | Optional — the Stripe customer id, used to persist the saved card / customer association |

**Response 204:** Payment confirmed (order transitions to `paid`). No response body.
**Response 400:** Payment failed/declined, order not found, or order not in a confirmable state.

---

### POST /api/v1/orders/{id}/cancel

Cancels an order. Only allowed while the order is in a cancellable state.

**Auth:** Required

**Request Body (`CancelOrderRequest`):**

```json
{
  "reason": "Changed my mind"
}
```

| Field | Rules |
|---|---|
| `reason` | Required — free-text cancellation reason |

**Response 200 (`CancelOrderResponseDto`):**

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
| `preparing` | No | Contact support: `contact@findmyflowers.pl` |
| `in_delivery` | No | Contact support |
| `delivered` | No | Use report-issue endpoint (post-MVP) |
| `cancelled` | No | Already cancelled (returns 409) |

**Response 409:** Order is not in a cancellable state.
**Response 404:** Order not found or does not belong to the authenticated customer.

---

### POST /api/v1/orders/{id}/confirm-delivery

Customer confirms they received the delivery, completing the order (transitions to `delivered`).

**Auth:** Required
**Request Body:** None.

**Response 204:** Delivery confirmed.
**Response 400:** Order not found, not owned by the customer, or not in a state that can be confirmed.

---

### POST /api/v1/orders/{orderId}/lines/{bouquetId}/feedback

Submits feedback (1–5 star rating + optional comment) for a single bouquet line in a **delivered** order.

**Auth:** Required

**Request Body (`SubmitFeedbackRequest`):**

```json
{
  "rating": 5,
  "comment": "Beautiful and fresh!",
  "isAnonymous": false
}
```

| Field | Rules |
|---|---|
| `rating` | Required — integer 1–5 |
| `comment` | Optional free text |
| `isAnonymous` | If `true`, the vendor does not see the customer identity |

**Response 201:** Feedback recorded — body `{ "id": "<feedbackId>" }`.
**Response 404:** `order-not-found` or `bouquet-not-in-order`.
**Response 409:** `order-not-delivered` or `feedback-already-exists`.
**Response 400:** Validation error (e.g. rating out of range).

---

### GET /api/v1/orders/me/pending-feedback

Returns delivered order lines the customer has not yet reviewed, so the app can prompt for ratings.

**Auth:** Required

**Response 200:** Array of `PendingFeedbackDto`.

---

### GET /api/v1/orders

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
      "bouquetThumbnailUrl": "https://api.findmyflowers.pl/...",
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

### GET /api/v1/orders/{id}

Returns full detail of a single order.

**Auth:** Required

**Response 200 (delivery order):**

```json
{
  "orderId": "ord-8821",
  "bouquetId": "b3f1a2c4-...",
  "bouquetImageUrl": "https://api.findmyflowers.pl/bouquets/b3f1a2c4/full.jpg",
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
  "bouquetImageUrl": "https://api.findmyflowers.pl/bouquets/b3f1a2c4/full.jpg",
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

### POST /api/v1/orders/{id}/report-issue

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

### GET /api/v1/subscriptions

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
      "frequency": "instant",
      "createdAt": "2025-03-01T10:00:00Z"
    },
    {
      "subscriptionId": "sub-002",
      "type": "flower_type",
      "isActive": true,
      "flowerType": "rose",
      "flowerTypeLabel": "Roses",
      "centerLatitude": 52.2297,
      "centerLongitude": 21.0122,
      "radiusKm": 5,
      "frequency": "daily",
      "createdAt": "2025-03-15T14:30:00Z"
    },
    {
      "subscriptionId": "sub-003",
      "type": "price_range",
      "isActive": true,
      "maxPrice": 50.00,
      "currency": "PLN",
      "centerLatitude": 52.2297,
      "centerLongitude": 21.0122,
      "radiusKm": 10,
      "frequency": "weekly",
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
      "centerLatitude": 52.2297,
      "centerLongitude": 21.0122,
      "radiusKm": 5,
      "frequency": "instant",
      "createdAt": "2025-03-25T10:00:00Z"
    }
  ],
  "totalCount": 4,
  "page": 1,
  "pageSize": 20
}
```

**Notes:**
- The notification cadence field is `frequency` (values `instant`, `daily`, `weekly`).
- Geo-scoped subscription types carry `centerLatitude` + `centerLongitude` (the search origin) plus `radiusKm`.

---

### GET /api/v1/subscriptions/{id}

Returns a single subscription by ID.

**Auth:** Required

**Response 200:** Single subscription object (same schema as items in `GET /api/v1/subscriptions`).

**Response 404:** Subscription not found or does not belong to the customer.

---

### POST /api/v1/subscriptions

Creates a new subscription.

**Auth:** Required
**Request Body (by type):**

Shop subscription:
```json
{
  "type": "shop",
  "shopId": "vend-42",
  "frequency": "instant"
}
```

Flower type subscription:
```json
{
  "type": "flower_type",
  "flowerType": "rose",
  "centerLatitude": 52.2297,
  "centerLongitude": 21.0122,
  "radiusKm": 5,
  "frequency": "instant"
}
```

Price range subscription:
```json
{
  "type": "price_range",
  "maxPrice": 50.00,
  "centerLatitude": 52.2297,
  "centerLongitude": 21.0122,
  "radiusKm": 10,
  "frequency": "weekly"
}
```

Combined subscription (flower type + price range):
```json
{
  "type": "combined",
  "flowerType": "rose",
  "maxPrice": 60.00,
  "centerLatitude": 52.2297,
  "centerLongitude": 21.0122,
  "radiusKm": 5,
  "frequency": "instant"
}
```

| Field | Rules |
|---|---|
| `type` | `shop`, `flower_type`, `price_range`, `combined` |
| `frequency` | `instant` (default), `daily`, `weekly` |
| `centerLatitude` / `centerLongitude` | Search origin for geo-scoped types (`flower_type`, `price_range`, `combined`) |
| `radiusKm` | Search radius (required for `flower_type`, `price_range`, `combined`) |
| `flowerType` | Required for `flower_type` and `combined` — use values from `GET /api/v1/reference/flower-types` |
| `shopId` | Required for `shop` |
| `maxPrice` | Required for `price_range` and `combined` |
| `currency` | Optional currency for `maxPrice` |

**Response 201:** Full subscription object (same as GET schema)
**Response 400:** Invalid parameters or missing required fields (e.g. unknown `frequency`).

---

### PUT /api/v1/subscriptions/{id}

Updates an existing subscription (toggle active, change frequency or criteria).

**Auth:** Required
**Request Body:** Partial subscription object (only changed fields required)

```json
{
  "isActive": false,
  "frequency": "daily"
}
```

**Response 204:** Subscription updated.
**Response 400:** Invalid parameters.

---

### PUT /api/v1/subscriptions/disable-all

Deactivates all subscriptions for the authenticated customer. Sets `isActive: false` on every subscription without deleting them (SUB-09).

**Auth:** Required

**Response 204:** All active subscriptions deactivated. No response body.

---

### DELETE /api/v1/subscriptions/{id}

Deletes a subscription.

**Auth:** Required
**Response 204:** No content
**Response 400:** Subscription not found or does not belong to the customer.

---

## 7. Vendor Endpoints

### GET /api/v1/vendors/{id}

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

### GET /api/v1/vendors/{id}/bouquets

Returns all currently available bouquets for a specific vendor.

**Auth:** Optional

**Query Parameters:**

| Param | Type | Required | Description |
|---|---|---|---|
| `page` | int | No | Page number (default: `1`) |
| `pageSize` | int | No | Items per page (default: `20`, max: `50`) |

**Response 200:** Same paginated schema as `GET /api/v1/bouquets/nearby` items.

---

### GET /api/v1/vendor/settings/pricing-strategy

Returns the vendor's current freshness-based pricing strategy, or the platform default if none is configured.

**Auth:** Required (`VendorAccess`)

**Response 200:**

```json
{
  "isDefault": false,
  "autoAdjustPrice": true,
  "tiers": [
    {
      "afterHours": 24,
      "cumulativeDiscountPercent": 20.0,
      "description": "After 24h in vase: -20% off original price"
    },
    {
      "afterHours": 48,
      "cumulativeDiscountPercent": 40.0,
      "description": "After 48h in vase: -40% off original price"
    },
    {
      "afterHours": 72,
      "cumulativeDiscountPercent": 70.0,
      "description": "After 72h in vase: -70% off original price"
    }
  ]
}
```

**Notes:**
- `isDefault: true` means no custom strategy is stored — the platform Standard preset is being used.
- Discount percentages are cumulative off the `BasePrice` set when the vendor last called `SetPrice` or `UpdatePrice` on the bouquet.

**Response 403:** No VendorId in token.

---

### PUT /api/v1/vendor/settings/pricing-strategy

Sets a custom freshness pricing strategy for the vendor. Replaces any previously stored strategy.

**Auth:** Required (`VendorAccess`)

**Request Body:**

```json
{
  "autoAdjustPrice": true,
  "tiers": [
    { "afterHours": 24, "cumulativeDiscountPercent": 15 },
    { "afterHours": 48, "cumulativeDiscountPercent": 35 },
    { "afterHours": 72, "cumulativeDiscountPercent": 60 }
  ]
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `autoAdjustPrice` | bool | No | Default `true`. Set `false` to store tiers without activating auto-adjustment. |
| `tiers` | array | Yes | At least one tier. `null` or empty resets to platform default. |
| `tiers[].afterHours` | int | Yes | Hours since bouquet placement at which this tier activates. Must be > 0 and strictly ascending. |
| `tiers[].cumulativeDiscountPercent` | decimal | Yes | Total discount off the original base price (not compounded). Must be in (0, 100]. Must be strictly ascending across tiers. |

**Response 204:** Strategy saved.

**Response 400:** Validation error (e.g., hours not ascending, discount out of range).

**Response 403:** No VendorId in token.

---

### DELETE /api/v1/vendor/settings/pricing-strategy

Resets the vendor's pricing strategy to the platform default (Standard 3-day schedule: −20%/−40%/−70% at 24h/48h/72h).

**Auth:** Required (`VendorAccess`)

**Response 204:** Strategy reset.

**Response 403:** No VendorId in token.

---

### PUT /api/v1/vendors/{id}/opening-hours

Saves the vendor's weekly schedule. Read back as `weeklyOpeningHours` on `GET /api/v1/vendors/{id}` (vendor-scoped detail).

**Auth:** Required (`VendorAccess`; a vendor may only change its own record, platform admins any)

**Request:**

```json
{ "schedule": [ { "day": "Monday", "isOpen": true, "openTime": "09:00", "closeTime": "18:00" },
                { "day": "Sunday", "isOpen": false, "openTime": null, "closeTime": null } ] }
```

**Response 200:** Saved. **Response 400:** Unknown/duplicate day, times not `HH:mm`, or closing not after opening (`detail` names the day). **Response 403:** Another vendor's record. **Response 404:** Vendor not found.

---

### PUT /api/v1/vendors/{id}/location

Moves the vendor's shop (traditional shop: physical location; freelance florist: home location, privacy radius kept). Customers' map places every bouquet at the vendor's location, so this moves all of the vendor's active bouquets.

**Auth:** Required (`VendorAccess`; own record only, platform admins any)

**Request:**

```json
{ "address": "Marszałkowska 10", "city": "Warszawa", "postalCode": "00-001", "country": "Polska",
  "latitude": 52.2297, "longitude": 21.0122 }
```

`address` may also be a whole address line or, for a map pin with no street address, the coordinates as text. For shops the parts are stored as one line: `"Marszałkowska 10, 00-001 Warszawa, Polska"`.

**Response 200:** Saved. **Response 400:** Coordinates out of range or `(0,0)`, or `address` empty. **Response 403:** Another vendor's record. **Response 404:** Vendor not found.

---

### GET /api/v1/vendors/address-lookup/search?query={text}

Validates a typed address (OpenStreetMap Nominatim). Returns up to five candidates, most relevant first.

**Auth:** Anonymous (used by the sign-up wizard too). Rate-limited per user/IP: 20 requests/minute (`429`).

**Response 200:**

```json
[ { "latitude": 52.2297, "longitude": 21.0122, "displayName": "Marszałkowska 10, Śródmieście, Warszawa, 00-001, Polska",
    "street": "Marszałkowska 10", "city": "Warszawa", "postalCode": "00-001", "country": "Polska" } ]
```

**Response 400:** `query` shorter than 3 or longer than 300 characters. **Response 404:** No match. **Response 503:** Geocoder unavailable.

---

### GET /api/v1/vendors/address-lookup/reverse?latitude={lat}&longitude={lng}

Resolves a map point to the nearest address. Same auth, rate limit and candidate shape as search; the returned coordinates are the requested point, not the snapped building.

**Response 200:** One candidate. **Response 400:** Coordinates out of range. **Response 404:** No address at that point (e.g. open water). **Response 503:** Geocoder unavailable.

---

## 8. Customer Profile Endpoints

### GET /api/v1/profile

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

### PUT /api/v1/profile

Updates the customer's profile.

**Auth:** Required

**Request Body (`UpdateProfileRequest`):** Partial object — only changed fields required.

```json
{
  "name": "Jan Kowalski",
  "phoneNumber": "+48123456789",
  "preferredFlowerTypes": ["rose", "tulip", "peony"]
}
```

| Field | Rules |
|---|---|
| `name` | Optional — single display name (there is no separate first/last name) |
| `phoneNumber` | Optional |
| `preferredFlowerTypes` | Optional — array of flower type keys |

**Response 204:** Profile updated. No response body.

---

### PUT /api/v1/profile/notification-settings

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

**Response 204:** Notification settings updated. No response body.

**Notes:**
- Setting `pushEnabled: false` stops all push notifications from being delivered to the device but does **not** deactivate individual subscriptions. To deactivate all subscriptions, use `PUT /api/v1/subscriptions/disable-all`.
- Setting `subscriptionDigestEnabled: false` stops daily/weekly digest emails for subscriptions with `frequency` set to `daily` or `weekly`.

---

### POST /api/v1/profile/phone/send-code

Sends an SMS verification code to the supplied phone number. Stores the phone on the customer, resets its verified flag, then dispatches the code via the SMS provider. Rate-limited (policy `phone-verify`).

**Auth:** Required

**Request Body (`SendPhoneCodeRequest`):**

```json
{
  "phoneNumber": "+48123456789"
}
```

**Response 202:** Code dispatched.
**Response 400:** `invalid-phone-format` (or other validation error).
**Response 502:** `sms-send-failed` — the SMS provider rejected the send.

---

### POST /api/v1/profile/phone/verify-code

Verifies the 6-digit code sent by `send-code`. On success the customer's `PhoneVerified` flag is set.

**Auth:** Required

**Request Body (`VerifyPhoneCodeRequest`):**

```json
{
  "code": "123456"
}
```

**Response 204:** Phone verified.
**Response 400:** Code invalid or expired.

---

### DELETE /api/v1/profile

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

### GET /api/v1/profile/addresses

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

### POST /api/v1/profile/addresses

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

### PUT /api/v1/profile/addresses/{id}

Updates a saved address.

**Auth:** Required
**Request Body:** Partial address object.
**Response 204:** Address updated. No response body.

---

### DELETE /api/v1/profile/addresses/{id}

Deletes a saved address.

**Auth:** Required
**Response 204:** No content.

---

## 10. Push Notifications Endpoints

### POST /api/v1/notifications/devices

Registers a device for push notifications (Firebase Cloud Messaging for Android, APNs for iOS). Upserts by device token.

**Auth:** Required

**Request Body (`RegisterDeviceRequest`):**

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
| `appVersion` | Optional |

**Response 204:** Device registered / updated. No response body.

---

### DELETE /api/v1/notifications/devices/{deviceToken}

Unregisters the current device from push notifications (e.g., on sign-out). The token is passed in the path — there is no request body.

**Auth:** Required

**Response 204:** No content.

---

### GET /api/v1/notifications

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

### PUT /api/v1/notifications/{id}/read

Marks a notification as read.

**Auth:** Required
**Response 204:** No content.

---

### PUT /api/v1/notifications/read-all

Marks all notifications as read.

**Auth:** Required
**Response 204:** No content.

---

## 11. Reference Data Endpoints

### GET /api/v1/reference/flower-types

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

### GET /api/v1/reference/freshness-labels

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

### GET /health

Returns backend health status. Useful for mobile app to detect connectivity issues. (Health checks are mapped at the unversioned root: `/health`, plus `/health/ready` and `/health/live` — they are **not** under the `/api/v1` prefix.)

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
| **Authenticated** | Pass `access_token` query string param | Customers group, radius groups, order groups |
| **Anonymous** | Connect without token | Customers group, radius groups |

Anonymous connections can subscribe to `JoinCustomersGroup` (or `JoinRadiusGroup`) to receive real-time bouquet availability updates (AUTH-04, DISC-04). Note that bouquet availability broadcasts (`BouquetBecameAvailable` / `BouquetBecameUnavailable` / `FreshnessUpdated`) are currently emitted to the flat `customers` group; radius groups still exist for geo-scoped subscriptions. Order-related groups (`JoinOrderGroup`) require authentication.

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

Unsubscribe from radius updates. Takes the same `(lat, lng, radiusKm)` arguments used to join — the group name is derived from them, so they must match.

```csharp
await connection.InvokeAsync("LeaveRadiusGroup", latitude, longitude, radiusKm);
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

#### JoinCustomersGroup / LeaveCustomersGroup

Join (or leave) the flat `customers` broadcast group used by the web map and mobile clients to receive `BouquetBecameAvailable` / `BouquetBecameUnavailable` / `FreshnessUpdated` / price / view-count broadcasts. No arguments. On joining, the server also pushes the current offline-vase state so stale markers can be hidden immediately.

```csharp
await connection.InvokeAsync("JoinCustomersGroup");
await connection.InvokeAsync("LeaveCustomersGroup");
```

#### JoinCustomerGroup

Join a per-customer group (`customer-{customerId}`) and the flat `customers` group.

```csharp
await connection.InvokeAsync("JoinCustomerGroup", customerId);
```

#### JoinVendorGroup

Vendor-portal subscription. Joins `vendor-{vendorId}` and `vendortype-{vendorType}`. If the JWT carries a `vendor_id` claim it must match `vendorId`.

```csharp
await connection.InvokeAsync("JoinVendorGroup", vendorId, vendorType);
```

#### StartViewingBouquet / StopViewingBouquet

Increment / decrement the real-time view counter for a bouquet (drives `BouquetViewCountChanged`).

```csharp
await connection.InvokeAsync("StartViewingBouquet", bouquetId);
await connection.InvokeAsync("StopViewingBouquet", bouquetId);
```

---

### Server → Client

#### BouquetBecameAvailable

A bouquet became available. Broadcast to the flat `customers` group (not per-radius). The payload is **flat and minimal** — the client should fetch full detail via REST if it needs location, freshness, distance, etc.

```json
{
  "bouquetId": "b3f1a2c4-...",
  "vendorId": "vend-42",
  "vendorType": "shop",
  "status": "available",
  "price": 89.00,
  "currency": "PLN",
  "imageUrl": "https://api.findmyflowers.pl/...",
  "timestamp": "2025-04-10T09:15:00Z"
}
```

**Notes:**
- Not all fields are present on every emission — a status-transition emission may carry only `bouquetId`, `vendorId`, `vendorType`, `status`, `timestamp`, while a newly-published bouquet also carries `price`, `currency`, `imageUrl`.
- There is no nested `location`, `distanceKm`, `freshnessScore`, `vendorName`, `vaseId`, or `availableFrom` on this event — fetch `GET /api/v1/bouquets/{id}/detail` for those.
- The client subscribes via `JoinCustomersGroup` (web map) / `JoinCustomerGroup` (authenticated).

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

A bouquet's freshness score has changed (pushed periodically by the vase sensors). Broadcast to the `customers` group and the owning `vendor-{vendorId}` group.

```json
{
  "bouquetId": "b3f1a2c4-...",
  "freshnessScore": 78,
  "freshnessLabel": "Fresh",
  "timestamp": "2025-04-10T09:15:00Z"
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

> **Not yet implemented** (planned EP-07). Documented here for the mobile client to code against; the backend does not currently emit this event.

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
  "type": "https://findmyflowers.pl/errors/bouquet-not-found",
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
  "type": "https://findmyflowers.pl/errors/validation-error",
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

For 409 price-changed errors (on `POST /api/v1/orders`), the response includes the current price:

```json
{
  "type": "https://findmyflowers.pl/errors/price-changed",
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
| `Idempotency-Key` | For POST /api/v1/orders | Client-generated UUID v4 |

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
| `POST /api/v1/customers/register` | No | No | Duplicate returns 400 naturally |
| `POST /api/v1/auth/resend-verification` | Yes | Yes | Rate-limited; no side effects on repeat |
| `POST /api/v1/orders/preview` | Yes | Yes | Read-only, no side effects |
| `POST /api/v1/orders` | No | **Yes with `Idempotency-Key`** | Same key returns cached 201 response |
| `POST /api/v1/orders/{id}/confirm-payment` | Yes | Yes | Duplicate confirmation returns 204 |
| `POST /api/v1/orders/{id}/cancel` | Yes | Yes | Duplicate cancellation returns 409 harmlessly |
| `POST /api/v1/subscriptions` | No | No | Duplicate returns 400 if identical |
| `POST /api/v1/favourites/{id}` | Yes | Yes | Idempotent — duplicate returns 204 |
| `POST /api/v1/profile/addresses` | No | No | May create duplicates — client should check |
| `POST /api/v1/notifications/devices` | Yes | Yes | Upserts by device token |
| All `GET` endpoints | Yes | Yes | Always safe |
| All `PUT` endpoints | Yes | Yes | Same payload produces same result |
| All `DELETE` endpoints | Yes | Yes | Deleting already-deleted returns 204/404 |

**Usage:**

```
POST /api/v1/orders
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
| `GET /api/v1/config` | `public, max-age=3600` | No | Cache for 1h; serve stale if offline |
| `GET /api/v1/reference/flower-types` | `public, max-age=86400` | Yes | Cache for 24h; serve stale if offline |
| `GET /api/v1/reference/freshness-labels` | `public, max-age=86400` | Yes | Cache for 24h; serve stale if offline |
| `GET /api/v1/vendors/{id}` | `public, max-age=3600` | Yes | Cache for 1h; serve stale if offline |
| `GET /api/v1/bouquets/{id}/detail` | `private, max-age=60` | Yes | Short cache; show stale with banner if offline |
| `GET /api/v1/bouquets/nearby` | `private, no-cache` | No | Cache last result; show with "May be outdated" banner |
| `GET /api/v1/orders` | `private, no-cache` | No | Cache last result for offline viewing |
| `GET /api/v1/orders/{id}` | `private, max-age=30` | Yes | Short cache; show stale with banner |
| `GET /api/v1/profile` | `private, max-age=300` | Yes | Cache for 5 min |
| `GET /health` | `no-store` | No | Never cache |
| CDN images (`thumbnailUrl`, `imageUrl`) | `public, max-age=604800` | No | Cache for 7 days |

### Conditional Requests

For endpoints that return `ETag`, the mobile app should send `If-None-Match` on subsequent requests:

```
GET /api/v1/bouquets/b3f1a2c4-.../detail
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

---

## 21. AI Florist (EP-28)

Vendor-only (`VendorAccess` policy). The tenant always comes from the JWT (`vendor_id`,
`vendor_type`) via `ITenantContextService`; `X-Vendor-Id` never overrides it. A session of another
vendor answers **404**, exactly like a missing one. All routes are under `/api/v1/vendor`.
Enums are strings, ids are plain GUIDs, money is a JSON number in PLN (2 dp).

| Method | Path | Body / query | Success |
|---|---|---|---|
| GET | `/design-assistant/usage` | — | 200 `DesignUsageDto` (feature flags, quota, limits, styles) |
| POST | `/design-sessions` | `{ "inputMode": "Photos" \| "Manual" }`, optional `Idempotency-Key` | 201 `DesignSessionDto` (same key → same session) |
| GET | `/design-sessions?skip=0&take=20` | — | 200 `DesignSessionSummaryDto[]`, newest first |
| GET | `/design-sessions/{id}` | — | 200 `DesignSessionDto` — the polling target |
| DELETE | `/design-sessions/{id}` | — | 204; queued work cancelled, private media purged |
| POST | `/design-sessions/{id}/photos` | multipart: file field **`photo`** (JPEG/PNG/WebP ≤ 8 MB raw; re-encoded server-side to ≤ 1600 px JPEG, EXIF stripped), optional form field `clientPhotoId` | 201 `DesignSessionDto` (replayed `clientPhotoId` is a no-op) |
| DELETE | `/design-sessions/{id}/photos/{photoId}` | — | 200 `DesignSessionDto` |
| GET | `/design-sessions/{id}/photos/{photoId}/content` | — | 200 `image/jpeg`, `Cache-Control: private, no-store` |
| POST | `/design-sessions/{id}/recognize` | `{ "expectedVersion": 3 }`, **required** `Idempotency-Key` | 202 `DesignSessionDto` (state `Recognizing`) |
| PUT | `/design-sessions/{id}/inventory` | `{ "expectedVersion": 4, "confirmed": true, "items": [ { "id": null\|guid, "kind": "Flower\|Greenery\|Material", "name": "Pink rose", "color": "Pink", "quantity": 12, "unit": "Stem\|Item", "unitPrice": 4.00\|null } ] }` | 200 `DesignSessionDto` |
| POST | `/design-sessions/{id}/recipes` | `{ "expectedVersion": 5, "size": "M", "mode": "AvailableOnly", "count": 3, "style": "Soft & natural"\|null, "costCeiling": 100\|null }`, **required** `Idempotency-Key` | 202 `DesignSessionDto` (state `Composing`) |
| POST | `/design-sessions/{id}/recipes/{recipeId}/preview` | `{}`, **required** `Idempotency-Key` | 202 `DesignSessionDto` (preview `Queued`); 200 if a preview is already `Ready` |
| GET | `/design-sessions/{id}/recipes/{recipeId}/preview/content` | — | 200 image bytes, `private, no-store` |
| PUT | `/design-sessions/{id}/recipes/{recipeId}/saved` | `{ "saved": true }` | 200 `DesignSessionDto` |
| GET | `/design-sessions/{id}/recipes/{recipeId}/cost?sellingPrice=120` | optional `sellingPrice` | 200 `RecipeCostDto` (margin only when cost is complete) |
| GET | `/design-recipes?saved=true` | — | 200 `DesignRecipeDto[]` (saved recipes across sessions, newest first) |
| GET | `/design-cost-profile` | — | 200 `DesignCostProfileDto` (created with defaults on first read) |
| PUT | `/design-cost-profile` | `{ "expectedVersion": 1, "basis": "Gross", "laborMinutes": 30, "hourlyRate": 40, "overheadPercent": 10 }` | 200 `DesignCostProfileDto` |

**Session states**: `Draft → Recognizing → InventoryReady → InventoryConfirmed → Composing → RecipesReady`
(manual entry: `Draft → InventoryConfirmed` by saving inventory with `confirmed: true`). `Deleted` is
never returned (404). Editing confirmed stock bumps `stockRevision` and drops **unsaved** recipes.

**Preview states** (per recipe): `NotRequested | Queued | Running | Ready | Failed | OutcomeUnknown`.
Previews never change the session state; a recipe is usable and saveable without one.

**Polling**: while `state` is `Recognizing`/`Composing` or any preview is `Queued`/`Running`, GET the
session every 2 s for the first 60 s, then every 5 s; stop when backgrounded and re-read on resume.
After 120 s show "Still working" with leave/resume. Never re-POST a billable start to "retry" a
poll — reuse the same `Idempotency-Key` if the original response was lost.

**Inventory rules**: ≤ 100 lines, quantity 1–9999, flowers/greenery in `Stem`, materials in `Item`
(or `Stem`), name ≤ 80 chars. A `null` `unitPrice` is filled from the vendor's remembered price
for the same name+colour when one exists; explicit prices are remembered for next time.
Size **M** = 11–21 flower+greenery stems (materials excluded).

**Cost**: `total` is `null` and `isComplete` is `false` while any line lacks a price —
`knownSubtotal` is shown as "Known cost" with "N prices missing". Margin = (price − total) ÷
price × 100 (gross margin, not markup). A `costCeiling` needs every stock line priced (422
`cost-profile-incomplete` otherwise).

**Errors**: RFC 7807 ProblemDetails `{ status, title, detail, code }`; `detail` is safe to show.

| Status | `code` |
|---|---|
| 400 | `invalid-request`, `invalid-inventory`, `invalid-photo`, `photo-limit` |
| 403 | `feature-disabled` (kill switch / not in pilot) |
| 404 | `not-found` (absent or another vendor's) |
| 409 | `version-conflict` (refresh, keep the user's edits), `invalid-state`, `idempotency-conflict` |
| 422 | `insufficient-stock`, `cost-profile-incomplete`, `invalid-brief`, `must-include-missing`, `refine-limit` (EP-33) |
| 429 | `quota-exhausted`, `spend-cap-reached`, `too-many-active` (+ `Retry-After` seconds) |

Example `DesignSessionDto` (abridged):

```json
{
  "id": "6f0c…", "version": 7, "stockRevision": 1, "state": "RecipesReady", "inputMode": "Photos",
  "createdAt": "2026-10-14T09:00:00Z", "updatedAt": "2026-10-14T09:01:10Z",
  "photos": [{ "id": "…", "clientPhotoId": "a1", "status": "Uploaded",
               "contentUrl": "/api/v1/vendor/design-sessions/6f0c…/photos/…/content", "width": 1600, "height": 1200 }],
  "inventory": { "confirmed": true, "items": [
    { "id": "…", "kind": "Flower", "name": "Pink rose", "color": "Pink", "quantity": 12, "unit": "Stem",
      "unitPrice": 4.0, "confidence": null, "estimated": false } ] },
  "settings": { "size": "M", "mode": "AvailableOnly", "count": 3, "style": "Soft & natural", "costCeiling": null },
  "recipes": [{ "id": "…", "sessionId": "6f0c…", "title": "Blush garden", "description": "…",
    "assemblyInstructions": "1. …", "palette": ["Pink", "White", "Sage"], "stockRevision": 1,
    "matchesCurrentStock": true, "totalStems": 14,
    "lines": [{ "inventoryItemId": "…", "kind": "Flower", "name": "Pink rose", "color": "Pink", "quantity": 7, "unit": "Stem" }],
    "cost": { "currency": "PLN", "basis": "Gross", "lines": [ … ], "flowersSubtotal": 52.0, "materialsSubtotal": 12.0,
              "labor": 0, "overhead": 0, "knownSubtotal": 64.0, "total": 64.0, "isComplete": true, "unknownItemIds": [] },
    "preview": { "state": "NotRequested", "contentUrl": null, "errorMessage": null, "updatedAt": null },
    "saved": false, "createdAt": "…", "savedAt": null }],
  "jobs": [{ "id": "…", "kind": "Composition", "state": "Completed", "recipeId": null,
             "submittedAt": "…", "completedAt": "…", "errorCode": null, "errorMessage": null }],
  "activeJob": null,
  "notice": null
}
```

`DesignUsageDto`: `{ enabled, previewsEnabled, recognitionsUsed, recognitionsLimit, recognitionsRemaining,
compositionsUsed, compositionsLimit, compositionsRemaining, previewsUsed, previewsLimit, previewsRemaining,
resetsAt, maxPhotos, maxPhotoBytes, maxLongEdgePixels, sizes: ["S", "M", "L"], modes: ["AvailableOnly"],
styles: ["Soft & natural", …], minStems: 11, maxStems: 21 }`.

`RecipeCostDto`: `{ cost: CostBreakdownDto, sellingPrice, marginAmount, marginPercent, marginUnavailableReason }`.

`DesignCostProfileDto`: `{ version, basis, laborMinutes, hourlyRate, overheadPercent, prices: [{ name, color, unitPrice, updatedAt }] }`.

`DesignSessionSummaryDto`: `{ id, state, inputMode, inventoryItemCount, recipeCount, savedRecipeCount, createdAt, updatedAt }`.

Canonical copy for the mobile client: `flower-shop-vendor-mobile-app/docs/superpowers/specs/2026-10-14-ai-bouquet-assistant-api-contract.md`.

### 21.1 Feedback loop & smarter wizard (EP-33)

Additive. Vendor text (house style, brief, refine notes, corrections) is only ever sent to models as
labelled JSON data; every recipe — refined or repaired — still goes through the code validator, the
size rule, the cost ceiling and the must-include check.

| Method | Path | Body / query | Success |
|---|---|---|---|
| POST | `/design-sessions/{id}/recipes` | adds optional `"occasion": "birthday\|romantic\|wedding\|sympathy\|thank-you\|celebration\|everyday"`, `"colourPreference"` (≤ 60), `"mustInclude": [inventoryItemId]`, `"leaveOut": [inventoryItemId]`, `"briefNote"` (≤ 200); `"size": "S\|M\|L"` | 202 `DesignSessionDto`; 422 `invalid-brief` before any paid call when the picks can't make the size |
| POST | `/design-sessions/{id}/recipes/{recipeId}/refine` | `{ "chips": ["more-greenery"\|"fewer-stems"\|"cheaper"\|"more-colour"\|"softer-colours"\|"swap-main-flower"], "note": "…" }` (≤ 200), **required** `Idempotency-Key` | 202 `DesignSessionDto` (state `Composing`; `activeJob.refineOfRecipeId`). One revised recipe is appended right after the original with `parentRecipeId` and `generation + 1`; counts as one composition; 422 `refine-limit` after 5 generations |
| POST | `/design-sessions/{id}/recipes/{recipeId}/preview` | optional `{ "corrections": ["wrong-colours"\|"extra-flowers"\|"missing-flowers"\|"wrong-size"\|"unrealistic"\|"other"], "note": "…" }` | With corrections: a re-render of a `Ready` preview (counts as a preview; the new image replaces the old) |
| PUT | `/design-sessions/{id}/recipes/{recipeId}/feedback` | `{ "target": "Recipe\|Preview", "rating": "Up\|Down", "reasons": [chip], "comment": "…" }` (≤ 300; reasons kept only for `Down`) | 200 `DesignSessionDto`; idempotent, last write wins; another vendor's recipe → 404; rating a preview that isn't `Ready` → 409 |
| DELETE | `/design-sessions/{id}/recipes/{recipeId}/feedback?target=Recipe` | — | 200 `DesignSessionDto` |
| POST | `/design-sessions/{id}/recipes/{recipeId}/signals` | `{ "kind": "Copied" }` (the only client-reported signal) | 204 |
| GET | `/design-style-profile` | — | 200 `DesignStyleProfileDto` (created on first read) |
| PUT | `/design-style-profile` | `{ "expectedVersion": 1, "notes": "loose garden look", "avoid": [{ "name": "Carnations", "stockItemId": null }], "favourites": [stockItemId], "defaultStyle": "Garden", "defaultColourPreference": "soft pastels", "learnFromRatings": true, "improveAiConsent": false }` | 200 `DesignStyleProfileDto`; 409 `version-conflict` |
| DELETE | `/design-feedback` | — ("Forget my ratings") | 200 `DesignStyleProfileDto` with `ratingCount: 0` |

**Recipe fields** (`DesignRecipeDto`): `why` (one sentence, `recipe-v2`; null from older prompts),
`size` (`S\|M\|L`), `parentRecipeId`, `generation`, `feedback` / `previewFeedback`
(`{ target, rating, reasons, comment, updatedAt }` or null). Inventory items add `hintApplied` (recognition
renamed the line using one of the shop's earlier fixes). `settings` adds `occasion`, `colourPreference`,
`mustInclude`, `leaveOut`, `briefNote`. Jobs add `refineOfRecipeId`.

**Sizes** (`usage.sizeRules`): S 7–12 stems (labour × 0.75), M 11–21 (× 1), L 19–31 (× 1.5). `usage` also
lists `occasions`, `refineChips`, `recipeFeedbackReasons`, `previewFeedbackReasons`.

**Implicit signals** (server-side, same transaction as the action): `Saved`, `PreviewRequested`, `Refined`,
`StockUsed` (`POST /vendor/stock/consume` with `reference = "design-recipe:{recipeId}"`) and the client's `Copied`.

**House style**: avoid-listed **stock items** are removed in code before composition (a hard rule, also
for the brief pre-flight); free avoid names are guidance. With "Learn from my ratings" on and at least
three ratings, the composer receives a vendor-scoped taste summary (up to 3 liked/made recipes, top 3 👎
reasons of 30 days; no comments, no other shop's data).

**Repair**: when fewer valid recipes came back than asked, the server makes one repair call with the
validator's reasons (free for the vendor, reserved against the global $ cap; `BouquetDesign:Composition:RepairEnabled`).

**New 422 codes**: `invalid-brief`, `must-include-missing`, `refine-limit`.

**Admin (PlatformAdmin)**: `GET /api/v1/admin/analytics/ai-florist?from&to&step&promptVersion` → `AiFloristQualityDto`
`{ from, to, rows: [{ step, provider, model, promptVersion, jobs, failureRate, p50LatencyMs, p95LatencyMs, spentUsd,
recipes, ratings, thumbsUpRate, savedRate, usedStockRate, refineRate, validatorRejectionRate, repairAttempts,
repairSuccessRate, recognisedLines, correctedLineRate, looksRightRate, rerenderRate }], trend: [{ date, step,
promptVersion, jobs, positiveRate }], promptVersions }` (aggregates only).
`GET /api/v1/admin/analytics/ai-florist/feedback-export?from&to` → `text/csv` of feedback from vendors who
switched on "Help improve the AI Florist" (contact data and shop names scrubbed; audit-logged).

### 21.2 Visual stock start and two wizard modes (EP-34)

Additive. `InventoryItemDto` gains `mainPhotoId` (the linked stock item's main photo, for a thumbnail via
`GET /vendor/stock/photos/{photoId}/content`), `fallbackUnitPrice` and `fallbackSource` (`"stock"` |
`"remembered"`: the price used when `unitPrice` is empty). They are filled on `GET /design-sessions/{id}`
and `PUT …/inventory`, with one query per kind of data (no per-row calls).

`POST /design-sessions` accepts `inputMode: "Stock"` as the EP-34 name of `"Manual"` (mode 1, "start from my
stock"). `DesignUsageDto.inputModes` lists what new sessions may use: `["Manual", "Photos"]` while
`BouquetDesign:LegacyPhotosMode=true` (default, until the vendor app migrates, EP-M14), then `["Manual"]`
— a new `Photos` session then answers 400 `invalid-request`. Existing `Photos` sessions and `/recognize`
keep working. Bench counting moved to the Stock page (§22.2).

### 21.3 Reconstruct a bouquet from photos (EP-35)

Mode 2 of the wizard: photos of **one** finished bouquet (e.g. a customer's picture) → what it is made of →
matched to the vendor's stock → Matched / Short / Missing → "use matched items" → the normal §21 compose
flow. **Never changes stock levels.** Create the session with `inputMode: "BouquetPhotos"` (listed in
`usage.inputModes`), upload 1–5 photos with the §21 photo endpoint, then:

| Method | Path | Body | Success |
|---|---|---|---|
| POST | `/vendor/design-sessions/{id}/analyze` | `{ expectedVersion }`, **required** `Idempotency-Key` | 202 `DesignSessionDto` (state `Analyzing`). Job kind `BouquetAnalysis`; counts as **one recognition**; same key → same job. Ends in `AnalysisReady` (or back to `Draft` with a failed job) |
| PUT | `/vendor/design-sessions/{id}/analysis/items/{itemId}` | `{ stockItemId?, notInStock, quantity? }` | 200 `DesignSessionDto`. Pick another of **your** stock items (scored in code; another vendor's id → 404), mark "not in my stock", or set the needed quantity (1–200). No model call; a different item than the AI's match is stored as an EP-33 correction |
| PUT | `/vendor/design-sessions/{id}/analysis/items/{itemId}` | `{ rename: { name, variety? } }` or `{ clearRename: true }` (EP-41) | 200 `DesignSessionDto`. One-tap rename ("Could be: Seeded eucalyptus", the photo's look-alike): re-matched to your stock in code and recorded as your §21.5 verdict (same flower type → `correct` with the variety; another type → `wrong`, actual name). The AI's answer is kept as `aiName`/`aiVariety`; `clearRename` goes back to it |
| GET | `/vendor/design-sessions/{id}/media/crops/{itemId}` | — | image (`private, no-store`): the item's crop from your photo (EP-41, §21.6). 404 when the line has no crop or the photos have expired |
| POST | `/vendor/design-sessions/{id}/analysis/use` | `{ expectedVersion }` | 200 `DesignSessionDto` (state `InventoryConfirmed`). Linked items become confirmed inventory lines linked by `stockItemId`, quantity capped at what's in stock, priced from the stock cost; Missing items never become lines. 409 `insufficient-stock` when nothing is linked |

`DesignSessionDto.analysis` (null until analysed): `{ purpose: "Inspiration", analyzedAt, items: [{ id, kind, name,
colors[], variety, quantity, detectedQuantity, quantityMin, quantityMax, confidence, confidenceScore, verdict, visibility,
match: StockMatchDto|null, alternatives: StockMatchDto[≤2], status: "Matched|Short|Missing", shortfall, notInStock }] (EP-40
`confidenceScore`/`verdict`: §21.5), missing: [{ name, kind,
colors[], variety, quantity, unit }], warnings[] ("mixed-bouquets", "low-quality", "not-a-bouquet", "colours-only" — the free Palette analyzer measured colours only and its item names are placeholders; show it), palette[],
procurement: { available: false, providers: [] } }`; EP-41 adds `aiName`, `aiVariety`, `atlas`, `boxes`, `cropUrl` and
`imageMatches` to each item (§21.6). `StockMatchDto`: `{ stockItemId, name, color, variety, score
(0–100, not a probability), band: "High|Medium|Low", factors[], available, mainPhotoId }`. Bands: High ≥ 80, Medium
50–79, Low < 50 — a Low match is shown but never linked unless the florist chooses it. `factors` are display texts
("name ✓", "colour close|similar|differs|unknown", "variety …", "right NN% of the time", "photo …"); the model's raw
self-rating is **not** a factor (the former `"AI NN % sure"` is no longer emitted — clients should drop it from older
payloads). The same colour text as the stock item is always "colour close"; colour phrases are read by their main
clause ("Bright yellow with dark brown center" = yellow). `missing` lists Missing items
and the shortfall of Short ones (the "to buy" list; supplier ordering is a later epic — `procurement.available` is
false). `sourceRequestId` is reserved for EP-37 ("Reconstruct with AI Florist" from a customer request).

New states `Analyzing`, `AnalysisReady`; recipes composed after "use" are steered towards the bouquet (the composer
receives the matched lines, palette and missing items as a target: one recipe "close to the photo", one "best
substitute"). Errors: 409 `invalid-state` (not a `BouquetPhotos` session, no photos, or busy), 429 `quota-exhausted`
/ `spend-cap-reached`, 403 `feature-disabled`, 400 `invalid-request` (missing `Idempotency-Key`, bad quantity).

### 21.4 Cost assessor (EP-36)

"What does this bouquet cost me?" — separate from design sessions. 1–5 photos of one **finished** bouquet are analysed
by the §21.3 engine (purpose `Assessment`: countable stems, hidden fillers, min/likely/max), matched to **your** stock
in code, and priced in code. The model never returns prices. Own monthly quota `BouquetDesign:Assessment:MonthlyLimit`
(default 30) under the AI Florist kill switch, pilot list and global $ cap; `DesignUsageDto` gains `assessmentsUsed`,
`assessmentsLimit`, `assessmentsRemaining`.

| Method | Path | Body | Success |
|---|---|---|---|
| GET | `/vendor/cost-assessments?skip&take` | — | 200 `CostAssessmentSummaryDto[]`, newest first (≤ 50) |
| POST | `/vendor/cost-assessments` | `{ note? }` (≤ 200), optional `Idempotency-Key` (same key → same assessment) | 201 `CostAssessmentDto` (state `Draft`) |
| GET / DELETE | `/vendor/cost-assessments/{id}` | — | 200 `CostAssessmentDto` / 204 (photos purged). Another vendor's → 404 |
| POST | `/vendor/cost-assessments/{id}/photos` | multipart field **`photo`** (≤ 5 live) | 201 `CostAssessmentDto` |
| DELETE | `/vendor/cost-assessments/{id}/photos/{photoId}` | — | 200 `CostAssessmentDto` |
| GET | `/vendor/cost-assessments/{id}/photos/{photoId}/content` | — | image (`private, no-store`) |
| POST | `/vendor/cost-assessments/{id}/analyze` | `{ expectedVersion }`, **required** `Idempotency-Key` | 200 `CostAssessmentDto` — **synchronous**: `Ready`, or `Failed` with `errorMessage`. Same key → replay, never billed twice. A `Ready` assessment is not analysed again (409 `invalid-state`); a `Failed` one may be |
| PUT | `/vendor/cost-assessments/{id}/lines` | `{ expectedVersion, lines: [{ id?, stockItemId?, kind, name, color?, variety?, quantity, quantityMin?, quantityMax?, unit?, unitCostOverride?, rememberPrice }] }` | 200 `CostAssessmentDto`. Replaces all lines (≤ 40; quantity 1–200; min ≤ quantity ≤ max). `stockItemId` must be one of your active items (else 404). `rememberPrice` stores the override as the remembered price for name\|colour\|variety. Lines entered by hand make a `Draft`/`Failed` assessment `Ready`. 409 `version-conflict` on a stale version |
| PUT | `/vendor/cost-assessments/{id}/selling-price` | `{ expectedVersion, sellingPrice: number\|null }` | 200 `CostAssessmentDto` |
| POST | `/vendor/cost-assessments/{id}/design` | — | 200 `DesignSessionDto`: a `BouquetPhotos` design in `AnalysisReady` with the photos copied and the lines as the analysis (stock links kept as the florist's choices); no model call. Same assessment → same design |

`CostAssessmentDto`: `{ id, version, state: "Draft|Analyzing|Ready|Failed", note, photos: DesignPhotoDto[], analysis:
DesignAnalysisDto|null (§21.3, purpose "Assessment"), lines: [{ id, stockItemId, kind, name, color, variety, quantity,
quantityMin, quantityMax, unit, unitCost, unitCostSource: "Override|Stock|Remembered|Unknown", unitCostOverride, lineCost,
mainPhotoId, available }], sellingPrice, cost: { currency, basis, flowersSubtotal, materialsSubtotal, labor, overhead,
knownSubtotal, total|null, isComplete, missingPrices, minTotal|null, maxTotal|null, marginAmount|null, marginPercent|null,
targetMarginPercent, suggestedPrice|null, unavailableReason }, provider, model, promptVersion, errorMessage, createdAt,
analyzedAt }`. Unit cost: the override, else the linked stock item's unit cost, else the remembered price, else unknown.
Totals reuse the §21 cost breakdown (labour from the cost profile, overhead on the subtotal); `total`, the range, margin
(gross, as `RecipeCostDto`) and `suggestedPrice` (the price at `targetMarginPercent`, whole PLN) are null while any price
is missing. `DesignCostProfileDto` gains `targetMarginPercent` (default 60; PUT accepts it, 0–95, omitted keeps it).

"Deduct from stock" is the existing `POST /vendor/stock/consume` (§22) with `reference = "cost-assessment:{id}"`,
for bouquets already made. Errors: 409 `invalid-state` / `version-conflict`, 429 `quota-exhausted` / `spend-cap-reached`
(with `Retry-After`), 403 `feature-disabled`, 502 `provider-failed` (state `Failed`; counted only if submitted),
400 `invalid-request` / `invalid-inventory` / `invalid-photo`, 422 `photo-limit`.

### 21.5 Recognition confidence from human feedback (EP-40)

The model's own "NN % sure" (`confidence`, still returned, **deprecated for display**) is replaced by a calibrated
`confidenceScore` on every recognised line: how often this kind of answer was right when florists checked it. It is
computed in code (Beta-Binomial: the model's raw-confidence bucket prior → anonymous platform counts per provider ×
model × prompt version × task × flower type, used only with ≥ 3 contributing vendors → this shop's own counts), never
by the model. Florists' verdicts feed it.

| Method | Path | Body | Success |
|---|---|---|---|
| PUT | `/vendor/design-sessions/{id}/analysis/items/{itemId}/verdict` | `RecognitionVerdictRequest` | 200 `DesignSessionDto` |
| PUT | `/vendor/design-sessions/{id}/inventory/items/{itemId}/verdict` | `RecognitionVerdictRequest` (recognised Photos-mode lines only) | 200 `DesignSessionDto` |
| PUT | `/vendor/stock/counts/{countId}/lines/{lineId}/verdict` | `RecognitionVerdictRequest` | 200 `StockCountDto` |

`RecognitionVerdictRequest`: `{ "identity": "correct|wrong|unsure", "actualName"?: "Scabiosa", "actualColor"?, "actualVariety"?,
"notInBouquet"?: false, "actualQuantity"?: 12 }`. "Right" means the **flower type** matches (colour and variety are
kept but don't decide it). `wrong` needs `actualName` or `notInBouquet: true` (a false positive). The verdict is about
recognition, **not** stock: "not in my stock" (`PUT …/analysis/items/{itemId}` `notInStock`) is unchanged and separate.
Idempotent: one verdict per line, a repeat updates it. No model call. Another vendor's session/count → 404; unknown
line → 404; bad body → 400 `invalid-request`. A wrong verdict with `actualName` also stores the EP-33 rename
correction (hints). Five or more verdicts on one review within 2 s are kept as implicit ("mark all right" bursts).

**Implicit verdicts** (weight 0.25, never overwriting an explicit one) are written by the existing confirms: Photos-mode
`PUT …/inventory` with `confirmed: true` (kept → right, renamed to another type → wrong, removed → not in the bouquet,
count compared), `POST …/analysis/use` (only lines it used; Missing lines and "not in my stock" give none),
`PUT /vendor/cost-assessments/{id}/lines` (EP-36) and `POST /vendor/stock/counts/{id}/apply` (applied lines).

Line DTOs (`DesignAnalysisItemDto`, `InventoryItemDto` for recognised lines, `StockCountLineDto`) gain:

```
confidenceScore: {
  identity: { score: 0.72, band: "High|Medium|Low", basis: "shop|platform|model|none", checks: 12.5 },
  count:    { … } | null
} | null,                       // null for lines made before EP-40 and for manual lines
verdict: { identity: "correct|wrong|unsure", actualName, actualColor, actualVariety, notInBouquet,
           actualQuantity, countCorrect, weight: "explicit|implicit", updatedAt } | null
```

Bands: High ≥ 0.85, Medium ≥ 0.6, Low < 0.6. `basis: "none"` = not enough checks yet — show "not enough checks yet",
not a number. `checks` is the weighted number of verdicts behind the basis (an implicit confirm counts 0.25). Scores are
computed at the end of the job and recomputed from the current cells on `GET` of the session / count. Decisions now
use the score: `StockMatcher` multiplies by the calibrated identity score (factor text `"right NN% of the time"` once
there are checks); an identity band **Low on real checks** (basis ≠ none) is never auto-linked; a stock count line's
`needsCheck` is true when the count band is Low or there are no checks yet (lines counted before EP-40 keep the old
"raw < 0.6" rule); the Cost Assessor widens the min/max quantity of lines with a Low count score by 25 % before the cost range.

`DesignStyleProfileDto` gains `shareRecognitionFeedback` (default **true**; PUT accepts it, omitted keeps it): "Help
improve recognition for everyone". Off → the vendor's verdicts stay out of the next platform rebuild (hourly) but still
feed its own shop cells; it still receives platform scores. Shared data is counts per flower type only — no vendor id,
names typed by florists, photos, notes, prices or stock.

**Admin (PlatformAdmin)**: `AiFloristQualityDto` gains `calibration: [{ provider, model, promptVersion, task, verdicts,
explicit, implicit, unchecked|null, reliability: [{ bucket, checked, meanRaw, observed }], eceRaw, eceCalibrated,
types: [{ typeKey, checked, accuracy }] (top 20, worst first), countAccuracy, countChecked }]` — groups, buckets and types
with fewer than 3 vendors are left out. The feedback export adds one `RecognitionVerdict` row per verdict of consenting
vendors (`source, type_key, detected_name, identity, actual_name, count_correct, weight`, scrubbed).
`POST /api/v1/admin/analytics/ai-florist/recognition-verdicts/backfill` → `{ fromCorrections, fromConfirmedSessions,
skipped, platformCells }`: one-off, idempotent conversion of existing corrections and confirmed Photos-mode sessions into
implicit verdicts, then a cell rebuild.

Config `BouquetDesign:Confidence:*`: `ScoresEnabled` (rollout switch — `false` in `appsettings.json`: verdicts are
captured but `confidenceScore` stays null and every decision keeps the model's own rating; switch on after the pilot's
calibration check), `ImplicitWeight` 0.25, `HalfLifeDays` 120, `PriorStrength` 10, `ShopStrength` 10,
`MinChecks` 5, `MinVendorsPerCell` 3, `MaxPerVendorPerCell` 30 (per `CapWindowDays` 30), `HighBand` 0.85, `MediumBand`
0.6, `ColdStartFactor` 0.8, `InheritFrom` (`"provider|model|prompt": "provider|model|prompt"` — a new version uses its
predecessor's bucket prior at `InheritStrength` 0.5), `BurstLines` 5 / `BurstSeconds` 2, `RefreshMinutes` 60.
Migration `AddRecognitionConfidence` (tables `RecognitionVerdicts`, `RecognitionConfidenceCells`,
`RecognitionConfidenceShopCells`; column `DesignStyleProfiles.ShareRecognitionFeedback`).

### 21.6 Flower atlas, item crops and image matching (EP-41)

Each review line can show three pictures: **in your photo** (a crop), **looks like** (a reviewed reference image from the
platform flower atlas, with its credit) and **your stock** (the matched item's main photo, `match.mainPhotoId`). Images
from the atlas are only ever ones a platform admin approved (§26). Everything here is computed in code; nothing is
applied to a line without the florist's tap.

**Analysis items** (`DesignAnalysisItemDto`, sessions and cost assessments) gain:

```
aiName: "Astilbe" | null,          // set when the florist renamed the line; `name`/`variety` are the new ones
aiVariety: null,
boxes: [{ photoIndex: 0, x: 0.12, y: 0.30, w: 0.18, h: 0.22 }],   // normalised 0–1, ≤ 3, photoIndex into the live photos in order
cropUrl: "/api/v1/vendor/design-sessions/{id}/media/crops/{itemId}" | null,   // cost assessments: /vendor/cost-assessments/{id}/media/crops/{itemId}
atlas: {
  flowerId, varietyId | null, typeKey: "flower:carnation",
  name: "Goździk",                 // request language (Accept-Language pl → Polish when known), else English
  nameEn, namePl, scientificName, varietyName | null, description,
  imageUrl | null,                 // public R2 URL, or /api/v1/atlas/images/{imageId}/content (anonymous) on local storage
  attribution | null, sourceUrl | null, licence | null (SPDX, e.g. CC-BY-SA-4.0), author | null,
  couldBe: { varietyId, name: "Eucalyptus", variety: "Seeded" } | null   // a common variety the AI didn't name
} | null,
imageMatches: {                    // Phase 2, null while BouquetDesign:Embedding:Provider is None
  model: "clip-vit-b32",
  atlas: [{ flowerId | null, typeKey, name, similarity: 0.91, source: "atlas|shop" }],   // ≤ 3, grouped by flower
  stock: [{ stockItemId, name, similarity, mainPhotoId | null }],                         // ≤ 3, your active items only
  disagreement: { flowerId, typeKey, name: "Amaranthus", similarity: 0.11 /* the margin */, source: "atlas" } | null,
  stockSuggestion: { stockItemId, name, similarity, mainPhotoId } | null                 // "Your stock may have this"
} | null
```

Show `attribution` (or author + licence) with every atlas image — CC-BY and CC-BY-SA require it — and link `sourceUrl`.
`couldBe` and `disagreement` are one-tap renames (`PUT …/analysis/items/{itemId}` with `rename`, §21.3);
`stockSuggestion` is a one-tap `stockItemId` choice. `couldBe` is null once the line is renamed. Recognised Photos-mode
inventory lines (`InventoryItemDto`, `estimated` or scored) and `StockCountLineDto` gain `atlas` too.

**Boxes and crops.** Prompt `bouquet-analysis-v2` asks the vision model for boxes; it rolls out as an EP-33 candidate
(`BouquetDesign:Analysis:CandidatePromptVersion` = `bouquet-analysis-v2`, `CandidatePercent` 0 → pilot). Code drops
boxes on a photo index that wasn't sent, clamps them to the photo, drops ones under 2 % or over 80 % of the photo and
keeps ≤ 3. The first box is cropped (padding 8 %, long edge ≤ 384 px, JPEG) from the container's own processed photo and
stored privately under its prefix; crops follow the source photos' retention and are deleted with the session or
assessment. v1 and the free Palette analyzer give no boxes (`boxes: []`, `cropUrl: null`).

**Image matching (Phase 2, off by default).** With `BouquetDesign:Embedding:Provider` = `Onnx` or `Http`, every crop is
fingerprinted and compared with approved atlas images and **your own** stock photos (vendor-scoped, filtered by vendor
first; vectors of different models are never compared). The match score gains an image factor
(`text × (1 − StockWeight) + image × StockWeight`, factor text `"photo looks like your …"`; unchanged without a
fingerprint; the flower-type gate still applies). A verdict (§21.5) on a line with a crop stores the crop's fingerprint
id; with `Embedding:LearnFromVerdicts` a ✓ or "it's actually …" makes the crop a labelled example of your private
neighbourhood (`source: "shop"` neighbours) — never the shared atlas.

| Method | Path | Auth | Success |
|---|---|---|---|
| GET | `/vendor/design-sessions/{id}/media/crops/{itemId}` | VendorAccess | image, `private, no-store` |
| GET | `/vendor/cost-assessments/{id}/media/crops/{itemId}` | VendorAccess | image, `private, no-store` |
| GET | `/vendor/atlas/vocabulary` | VendorAccess | `[{ name, language: "en|pl", kind, typeKey | null, source: "atlas|stock" }]` — the "What is it?" names: atlas names (request language first), then your active stock names |
| GET | `/api/v1/atlas/images/{imageId}/content` | anonymous | image (`public, max-age=86400`); approved images only, else 404 |
| GET | `/api/v1/atlas/attributions` | anonymous | `[{ flower, variety, imageUrl, sourceUrl, author, licence, attribution, source }]` — credits of every approved image |

Config `BouquetDesign:Embedding:*`: `Provider` (`None` default | `Onnx` | `Http`), `Model` (`clip-vit-b32`), `Dimensions`
(512), `StockWeight` 0.3, `DisagreeMargin` 0.05, `MinSimilarity` 0.6, `StockSuggestSimilarity` 0.85, `SimilarityFloor`
0.6 / `SimilarityCeiling` 0.95 (image factor 0 → 1), `LearnFromVerdicts` false, `TimeoutSeconds` 30, `Onnx:ModelPath`,
`Onnx:ImageSize` 224, `Onnx:Mean`/`Std` (CLIP defaults), `Http:Endpoint` (`POST {Endpoint}/embed` with
`{ model, images: [base64] }` → `{ model, dimensions, vectors }`), `Http:ApiKey`. Migrations `AddFlowerAtlas`
(`AtlasFlowers`, `AtlasImages`) and `AddImageEmbeddings` (`AtlasImageEmbeddings`, `StockPhotoEmbeddings`,
`AnalysisCropEmbeddings`, `RecognitionVerdicts.CropEmbeddingId`; installs pgvector and an HNSW cosine index when the
server offers it, otherwise similarity is computed in code).

---

## 22. Vendor Stock (EP-30)

Vendor-only (`VendorAccess` policy), under `/api/v1/vendor/stock`. Tenant from the JWT; another
vendor's item answers **404**. Independent of `BouquetDesign:Enabled`. Quantities are whole units
(flowers and greenery in stems); money is a JSON number in PLN (2 dp). Errors are ProblemDetails with
a stable `code` extension; show `detail` to the user.

| Method | Path | Body / query | Success |
|---|---|---|---|
| GET | `/vendor/stock` | `?search=&kind=Flower\|Greenery\|Material&status=all\|InStock\|Low\|Out&includeArchived=false` | 200 `StockOverviewDto` (summary is always over all active items) |
| POST | `/vendor/stock` | `{ kind, name, color?, unit?, quantity, unitCost?, lowStockThreshold?, supplier?, notes?, note? }` | 201 `StockItemDto` (`quantity > 0` records an `Initial` movement) |
| GET | `/vendor/stock/{id}` | — | 200 `StockItemDetailsDto` (item + 20 most recent movements) |
| PUT | `/vendor/stock/{id}` | `{ expectedVersion, kind, name, color?, unit?, unitCost?, lowStockThreshold?, supplier?, notes? }` | 200 `StockItemDto` |
| DELETE | `/vendor/stock/{id}` | — | 204 (archived; history kept) |
| POST | `/vendor/stock/{id}/restore` | — | 200 `StockItemDto` |
| POST | `/vendor/stock/{id}/movements` | `{ type: Received\|Used\|Wasted\|Counted, quantity, unitCost?, note? }`, optional `Idempotency-Key` | 200 `StockItemDto` (Counted: `quantity` is the counted total) |
| GET | `/vendor/stock/{id}/movements` | `?skip=0&take=50` (max 100) | 200 `StockMovementDto[]`, newest first |
| GET | `/vendor/stock/movements` | `?skip=0&take=50` (max 100) | 200 `StockMovementDto[]` — whole shop |
| POST | `/vendor/stock/consume` | `{ reason: Used\|Wasted, note?, reference?, lines: [{ stockItemId?, name?, color?, quantity }] }`, optional `Idempotency-Key` | 200 `StockConsumeResultDto` — all-or-nothing |

Rules: `unit` defaults to `Stem` (`Item` for materials); flowers/greenery must be `Stem`. Name +
colour (trimmed, case-insensitive) is unique per vendor and is the same key the assistant uses, so
`consume` lines may name an item instead of its id. A movement or consume replayed with the same
`Idempotency-Key` is applied once (`replayed: true` on consume). Unit costs of stock items fill
missing prices in assistant sessions (before remembered assistant prices).

| Status | `code` |
|---|---|
| 400 | `invalid-request` |
| 404 | `not-found` |
| 409 | `duplicate-item`, `version-conflict`, `archived` |
| 422 | `insufficient-stock`, `unknown-item`, `item-limit` (500 items per vendor) |

```json
{
  "summary": { "totalItems": 3, "lowStockCount": 1, "outOfStockCount": 1, "totalValue": 44.0, "unpricedCount": 1 },
  "items": [{
    "id": "…", "version": 4, "kind": "Flower", "name": "Rose", "color": "Pink", "unit": "Stem",
    "quantity": 8, "unitCost": 4.5, "lowStockThreshold": 10, "supplier": "Hurtownia", "notes": null,
    "isArchived": false, "status": "Low", "stockValue": 36.0,
    "lastReceivedAt": "2026-10-18T03:58:00Z", "createdAt": "…", "updatedAt": "…" }]
}
```

`StockMovementDto`: `{ id, stockItemId, itemName, itemColor, unit, type: Initial|Received|Used|Wasted|Counted,
delta, quantityAfter, unitCost, note, reference, occurredAt }`. `StockItemDetailsDto`: `{ item, recentMovements }`.
`StockConsumeResultDto`: `{ items: StockItemDto[], replayed }`. `totalValue` adds priced items only;
`unpricedCount` says how many items with stock on hand have no cost.

### 22.1 Variety, appearance and photos (EP-31)

A colour word is ambiguous (many reds; a rose can be red-edged white), so items carry an optional
**variety** (cultivar, ≤ 60 chars, part of the identity key `name|colour|variety`) and an
**appearance** suggested by AI from a photo and confirmed by the florist.

| Method | Path | Body / query | Success |
|---|---|---|---|
| POST | `/vendor/stock/{id}/photos` | multipart `photo` (JPEG/PNG/WebP, re-encoded) + `clientPhotoId?` (replay = no-op) | 201 `StockPhotoDto` (first photo becomes main; ≤ 5 per item) |
| DELETE | `/vendor/stock/{id}/photos/{photoId}` | — | 204 (next photo becomes main) |
| PUT | `/vendor/stock/{id}/photos/{photoId}/primary` | — | 204 |
| GET | `/vendor/stock/photos/{photoId}/content` | — | 200 `image/jpeg` (private — send the token; never bind the URL) |
| POST | `/vendor/stock/photos/{photoId}/describe` | optional `Idempotency-Key` | 200 `StockAppearanceSuggestionDto` (nothing is saved) |
| POST | `/vendor/stock/photo-suggestions` | multipart `photo` + `clientPhotoId?`, optional `Idempotency-Key` | 200 `StockPhotoSuggestionDto` — pending photo (24 h) + suggestion, or `notice` when AI is off / over quota |
| GET | `/vendor/stock/ai-usage` | — | 200 `StockAiUsageDto` |

- Items: `StockItemDto` adds `variety`, `appearance`, `primaryPhoto`; `StockItemDetailsDto` adds `photos`.
- `appearance`: `{ description?, pattern: Solid|Bicolour|Edged|Gradient|Speckled|Multicolour,
  palette: [{ name, hex: "#RRGGBB", share: 0–100 }] (≤ 5, shares ≤ 100), source: Manual|Ai, summary, updatedAt }`.
- POST `/vendor/stock` also accepts `variety?`, `appearance?`, `pendingPhotoId?` (attaches the pending photo).
  PUT accepts `variety?`, `appearance?` — an **omitted** field is left unchanged; an explicit `null` clears it.
- `consume` lines accept `variety?`; `stockItemId` wins over the key.
- Suggestion: `{ photoId, kind?, name?, variety?, colorLabel?, description?, pattern, palette, confidence?, provider, replayed }`.
  Usage: `{ enabled, provider, used, limit, remaining, resetsAt }`.
- Recognition is gated like the assistant (`BouquetDesign:Enabled`, `PilotVendorIds`, global $ cap) but has
  its **own monthly quota** (`BouquetDesign:Limits:StockRecognitionsPerMonth`, default 100). The ledger row
  is written before the provider call; a replayed key returns the stored suggestion, a key reused for
  another photo is refused. Provider: `BouquetDesign:StockRecognition:Provider` = `Palette` (free, local
  dominant colours) or `OpenAI`.
- Assistant: stock lines carry `stockItemId` + `variety`; recognition links bench photos to stock items,
  recipes know each item's palette, and previews use the stock items' main photos as references.

| Status | `code` (EP-31) |
|---|---|
| 400 | `invalid-photo`, `invalid-request` (e.g. a key reused for another photo) |
| 403 | `feature-disabled` |
| 409 | `in-progress` |
| 422 | `photo-limit` |
| 429 | `quota-exhausted`, `spend-cap-reached` |
| 502 | `provider-failed` (the reservation is released) |

### 22.2 Count stock from photos (EP-34)

Bench recognition on the Stock page. The EP-28 recognizer (inventory prompt + catalogue linking) estimates
counts; **nothing changes stock until apply**. Billed to the stock AI quota (`Limits:StockRecognitionsPerMonth`,
shared with §22.1 recognitions) under the AI Florist kill switch, pilot list and global $ cap.

| Method | Path | Body | Success |
|---|---|---|---|
| POST | `/vendor/stock/counts` | multipart, 1–5 files in field **`photos`**; **required** `Idempotency-Key` | 201 `StockCountDto` (state `Ready` or `Failed`); same key → same count |
| GET | `/vendor/stock/counts/{countId}` | — | 200 `StockCountDto`; another vendor's → 404 |
| POST | `/vendor/stock/counts/{countId}/apply` | `{ "lines": [{ "stockItemId": guid, "countedQuantity": 14 }] }`, **required** `Idempotency-Key` | 200 `StockCountDto` (state `Applied`). One `Counted` movement per line (`reference = "stock-count:{countId}"`), all or nothing; the same key replays, another key → 409 |

`StockCountDto`: `{ id, state: "Recognising|Ready|Failed|Applied", lines: [{ lineId, stockItemId|null, kind, name,
color, variety, unit, currentQuantity|null, countedQuantity, delta|null, confidence, needsCheck, mainPhotoId, confidenceScore,
verdict, checkNote|null, suggestedStockItemId|null }], photoCount, errorMessage, appliedLines, createdAt, appliedAt }`. `needsCheck` (EP-40) is true when the
calibrated count score is Low or there are no checks yet, or when the line has a `checkNote`; counts made before EP-40 keep "below 0.6 confidence".
One line per flower: the same flower listed twice (same stock item, or same type and colour) is merged with the
**larger** quantity and `checkNote` "Listed twice: 10 + 3". A link (the model's, a name+colour match or a remembered
rename) whose colour clearly differs from the item's colour/palette is dropped: the line keeps the detected colour,
`stockItemId` is null, `suggestedStockItemId` names the item and `checkNote` says "Colour differs from your Rose (Red)". Verdicts:
`PUT /vendor/stock/counts/{countId}/lines/{lineId}/verdict` (§21.5); applying writes implicit verdicts for applied lines.
Errors: 422 `unknown-item` (a line not linked to an active item of yours), 429 `quota-exhausted`/`spend-cap-reached`,
403 `feature-disabled`, 502 `provider-failed`. `StockItemDto` also gains `mainPhotoId`.

---

## 23. Customer Bouquet Requests (EP-37)

Customers post a bouquet request (1–5 reference images, note, budget, desired time) pinned at the delivery address;
vendors in range with platform access send offers; the customer accepts exactly one, which creates a **custom order**
paid through the existing Stripe flow (§8). Vendor-facing DTOs never contain the street, the exact pin, the recipient or
the customer's identity before the order is paid — the location is snapped to a deterministic ~500 m grid cell.
Errors are ProblemDetails with `code` (and `field` for validation): 400 `invalid-request` / `invalid-photo`, 403
`email-not-verified` / `subscription-required` / `out-of-range`, 404 `not-found`, 409 `invalid-state` / `version-conflict`
/ `request-closed` / `already-accepted` / `offer-exists`, 422 `photo-limit`, 429 `request-limit` (`Retry-After`).

**Request states**: `Draft → Open → OffersReceived → AwaitingPayment → Accepted → Fulfilled`, or `Expired` /
`Cancelled`. **Phases** after payment: `Paid → InProgress → Ready → OutForDelivery → Delivered` (pickup:
`Ready → PickedUp`). **Offer states**: `Active | Accepted | Rejected | Withdrawn | Lapsed`.

### 23.1 Customer (`CustomerAccess`, verified email)

| Method | Path | Body | Success |
|---|---|---|---|
| GET | `/customer/bouquet-requests?skip&take` | — | `CustomerRequestSummaryDto[]` |
| POST | `/customer/bouquet-requests` | `{ description (≤1000), budgetMin?, budgetMax?, occasion?, size? (S/M/L), desiredAt (≥ now+3 h, ≤ now+30 d), offerDeadline? (default min(desired−2 h, now+48 h)), fulfilment ("Delivery"\|"Pickup"), savedAddressId? \| street, city, postalCode, addressNotes?, recipientName?, recipientPhone?, latitude?, longitude?, saveAddress? }`, optional `Idempotency-Key` | 201 `CustomerRequestDto` (state `Draft`). A pin wins; else the saved address pin; else the address is geocoded (failure → 400 `field: pin`). Poland only. Limits: ≤ 3 open, ≤ 10 per 7 days → 429 `request-limit` |
| GET / PUT | `/customer/bouquet-requests/{id}` | PUT: the same body + `expectedVersion` | `CustomerRequestDto`. Editing with offers marks them "request changed" and tells those vendors |
| POST / DELETE | `/customer/bouquet-requests/{id}/images[/{imageId}]` | multipart field **`image`** | `CustomerRequestDto` (1–5; re-encoded, EXIF/GPS stripped, private) |
| GET | `/customer/bouquet-requests/{id}/images/{imageId}` | — | image (`private, no-store`) |
| POST | `/customer/bouquet-requests/{id}/publish` | — | `CustomerRequestDto` (`Open`; needs ≥ 1 image). Vendors in range are alerted |
| POST | `/customer/bouquet-requests/{id}/cancel` | — | `CustomerRequestDto` (only before an accept; offers rejected, vendors told) |
| GET | `/customer/bouquet-requests/{id}/offers?sort=price\|time\|distance` | — | `CustomerOfferDto[]` — latest revision of active offers (and the accepted one); suspended vendors hidden |
| GET | `/customer/bouquet-requests/{id}/offers/{offerId}/images/{imageId}` | — | image |
| POST | `/customer/bouquet-requests/{id}/offers/{offerId}/accept` | `{ offerRevision }` | `AcceptOfferResultDto { request, orderId, totalAmount, currency, paymentDueAt, paymentClientSecret, stripeEphemeralKey, stripeCustomerId }`. One transaction: request `AwaitingPayment`, offer `Accepted`, others `Rejected` (`another-offer-accepted`), custom `Order` created. Pay within **30 min** (§8 PaymentIntent; confirm via `POST /orders/{orderId}/confirm-payment` or the webhook). Repeating the same accept returns the same order. Another accept → 409 `already-accepted`; a stale revision → 409 `version-conflict` |
| POST | `/customer/bouquet-requests/{id}/offers/{offerId}/report` | `{ reason? }` | `true` |
| POST | `/customer/bouquet-requests/{id}/time-change/answer` | `{ accept }` | `CustomerRequestDto`. Declining is allowed only when the new time is > 60 min after the agreed one and cancels with a full refund; unanswered after 30 min = accepted |
| POST | `/customer/bouquet-requests/{id}/order/cancel` | — | `CustomerRequestDto`. While `AwaitingPayment`: gives up the acceptance (offers come back). When > 2 h late: cancels with a full refund |

`CustomerRequestDto`: `{ id, version, state, description, budgetMin, budgetMax, occasion, size, desiredAt, offerDeadline,
fulfilment, address { street, city, postalCode, notes, recipientName, recipientPhone }, latitude, longitude, images[{ id,
contentUrl, width, height }], activeOfferCount, acceptedOfferId, acceptedVendorName, order: CustomOrderStatusDto, hidden,
cancelReason, createdAt, closedAt }`. `CustomOrderStatusDto`: `{ orderId, phase, phases[{ phase, at }], committedReadyAt,
committedDeliveryAt, paymentDueAt, paidAt, pendingTimeChange, timeChanges[{ id, requestedAt, oldReadyAt, oldDeliveryAt,
newReadyAt, newDeliveryAt, reasons[], note, status: Pending|Accepted|AutoAccepted|Declined, answeredAt, autoAcceptAt,
canDecline }], lateByMinutes, serverNow, nextPhases[] }` — clients count down from `serverNow`. `CustomerOfferDto`: `{ id,
revision, updated, state, vendorId, vendorName, vendorRating, vendorReviewCount, distanceKm, price, deliveryFee (null =
included), total, readyAt, deliveryAt, laterThanRequested, notes, images[], requestChanged, updatedAt }`.

### 23.2 Vendor (`VendorAccess`; seeing requests needs platform access, EP-20, and a shop/base location)

| Method | Path | Body | Success |
|---|---|---|---|
| GET | `/vendor/bouquet-requests?minLat&maxLat&minLng&maxLng&take` | — | `VendorRequestSummaryDto[]`: open requests within the service radius (fuzzed point, district, distance, thumbnail, budget, times, offer count, `hasMyOffer`), nearest first |
| GET | `/vendor/bouquet-requests/{id}` | — | `VendorRequestDto`. Readable by vendors in range while it takes offers, by vendors with an offer until it's paid, and by the chosen vendor; after payment everyone else gets 404 |
| GET | `/vendor/bouquet-requests/{id}/images/{imageId}` | — | image (same access) |
| POST | `/vendor/bouquet-requests/{id}/offers` | `{ price, deliveryFee?, readyAt, deliveryAt? (delivery: required, ≥ readyAt, ≤ desired + 2 h), notes? (≤1000), designRecipeId? }`, optional `Idempotency-Key` | 201 `VendorRequestDto`. One active offer per vendor (409 `offer-exists`); closed request or past the deadline → 409 `request-closed`. Later than desired → `laterThanRequested` |
| PUT | `/vendor/bouquet-requests/{id}/offers/{offerId}` | the same + `expectedRevision` | `VendorRequestDto` (new revision; 409 `version-conflict`) |
| POST | `/vendor/bouquet-requests/{id}/offers/{offerId}/withdraw` | — | `VendorRequestDto` |
| POST / DELETE / GET | `/vendor/bouquet-requests/{id}/offers/{offerId}/images[/{imageId}]` | multipart `image` | `VendorRequestDto` / image (0–5) |
| POST | `/vendor/bouquet-requests/{id}/reconstruct[?fresh=true]` | — | `DesignSessionDto` — "Reconstruct with AI Florist" (§21.3): a `BouquetPhotos` design with the request's images copied privately and `sourceRequestId` set; its `analysis.estimatedCost` is the §21.4 cost of the matched items. Without `fresh` your newest live design for the request opens (one is created if none); `fresh=true` always creates a new one (the earlier stays in the wizard). `GET /vendor/bouquet-requests/{id}` returns `lastReconstruction { sessionId, createdAt, analyzedAt? }` so the app can ask which to open |
| POST | `/vendor/bouquet-requests/{id}/report` | `{ reason? }` | `true` (hidden after 3 reports) |
| GET | `/vendor/custom-orders?skip&take` | — | `VendorRequestDto[]` of your paid custom orders (revealed) |
| POST | `/vendor/bouquet-requests/{id}/phase` | `{ phase }` | `VendorRequestDto` — the next phase only (`order.nextPhases`) |
| POST | `/vendor/bouquet-requests/{id}/time-change` | `{ newReadyAt, newDeliveryAt?, reasons[] (≤5), note? (≤300) }` | `VendorRequestDto` (one pending change at a time) |
| GET / PUT | `/vendor/marketplace-settings` | PUT `{ expectedVersion, serviceRadiusKm (1–50, default 15), preparationLeadMinutes (0–720, default 60), requestAlertsEnabled, quietHoursStart?, quietHoursEnd? (0–23, Europe/Warsaw), minBudget? }` | `VendorMarketplaceSettingsDto` (+ `hasLocation`, `latitude`/`longitude` — the matching location for the map's shop marker and radius circle, null without one — `subscriptionActive`) |
| GET | `/vendor/notifications?unreadOnly&skip&take` | — | `VendorNotificationListDto { items[{ id, type, title, body, bouquetRequestId, orderId, isRead, createdAt }], unreadCount }` |
| PUT | `/vendor/notifications/{id}/read`, `/vendor/notifications/read-all` | — | `true` |

`VendorRequestDto`: `{ id, state ("Open"|"Closed" before payment), description, budgetMin, budgetMax, occasion, size,
desiredAt, offerDeadline, fulfilment, fuzzedLatitude, fuzzedLongitude, district, distanceKm, images[], offerCount, myOffer:
VendorOfferDto|null, canOffer, revealed, address|null, latitude|null, longitude|null, order: CustomOrderStatusDto|null }` —
`address`, `latitude`, `longitude` and `order` are set only for the chosen vendor after payment. The existing vendor order
endpoints (§9) show custom orders with `isCustom`, `customTitle`, `customNotes`, `bouquetRequestId`, and hide the customer
name and address until paid; `PUT /vendor/orders/{id}/status` refuses custom orders (use `phase`).

### 23.3 Admin (`PlatformAdmin`)

`GET /admin/bouquet-requests?state&hidden&skip&take` → `AdminRequestDto[]` (reports first; `escalated` = > 2 h late);
`POST /admin/bouquet-requests/{id}/hide|unhide|cancel { reason? }` (cancel refunds a paid order; audit-logged, users
notified); `GET /admin/bouquet-requests/{id}/images/{imageId}`.

### 23.4 Real time and alerts

SignalR hub **`/hubs/marketplace`** (authenticated; each connection joins `customer:{customer_id}` or `vendor:{vendor_id}`
from its token). Events: vendors — `request.new`, `request.digest`, `request.changed`, `request.closed`, `offer.accepted`,
`offer.rejected`, `offer.lapsed`, `order.paid`, `order.timeChange`, `order.cancelled`, `order.startSoon`, `order.notReady`,
`order.late`; customers — `offer.created`, `offer.updated`, `offer.withdrawn`, `request.expiring`, `request.expired`,
`request.lapsed`, `request.cancelled`, `order.paid`, `order.phase`, `order.timeChange`, `order.late`. Payloads:
`{ requestId, orderId, title, body, … }`. Each event also writes an inbox row in the same transaction as the change
(customers: `GET /notifications`, type `BouquetRequest`, `bouquetRequestId`; vendors: §23.2). New-request alerts honour
the vendor's opt-out, quiet hours and minimum budget, max 20 per hour then a digest. A background monitor (every minute)
sends the "no offers yet" reminder 2 h before the deadline, expires requests, lapses unpaid acceptances after 30 min,
auto-accepts unanswered time changes after 30 min, and raises SLA alerts once per order and rule (start soon, not ready,
late, +30 min, > 2 h with an admin escalation). Push (FCM, EP-06) and real email delivery are not wired yet — the email
channel calls `IEmailService.SendMarketplaceEmailAsync` (no-op stub today).

---

## 24. Docs Portal Handoff (EP-38)

`docs.findmyflowers.pl` is served by `FlowerShop.DocsPortal` behind Keycloak (client `flowershop-docs`). Readers are
`developer` (everything), `vendor` (vendor manuals) or `customer` (customer manual); several roles see the union. Apps
open the docs **without putting a token in a URL**: they trade their current access token for a one-time code.

| Method | Path | Auth | Body | Success |
|---|---|---|---|---|
| POST | `/docs/handoff` | Bearer (any signed-in role) | `{ returnPath? }` — relative docs path, e.g. `/manuals/vendor-portal-manual.html#s8`; empty = docs home | `{ url, expiresAt }`. `url` = `https://docs.findmyflowers.pl/auth/handoff?code=…&next=…`; open it in a browser tab (mobile: Custom Tabs / SFSafariViewController). The code is single-use and expires after 60 s |
| POST | `/docs/handoff/redeem` | docs portal only (client-credentials token of `flowershop-docs`) | `{ code }` | `{ subject, name, roles[], returnPath, sessionExpiresAt }` |
| POST | `/docs/roles` | docs portal only | `{ subject }` | `{ subject, roles[], known, active }` (session re-validation every 15 min; `known = false` = identity provider unavailable) |

Roles follow the Keycloak composites: realm `User` → `customer`, `VendorAdmin` → `vendor`, `PlatformAdmin` or group
`developers` → `developer`. Errors (ProblemDetails `code`): 400 `invalid-request` (`returnPath` not relative — absolute
URLs, `//host`, backslashes and control characters are refused), 403 `no-docs-access` (no docs role), 404
`invalid-code` (unknown, used or expired), 429 rate limit (20 handoffs per user per minute).

**Client behaviour**: on any failure (expired token after a refresh attempt, 4xx/5xx, network) open
`https://docs.findmyflowers.pl/signin?returnUrl=<returnPath>` instead — the docs portal signs the user in with Keycloak
and continues to the page. Never log the code or the URL that carries it, and keep both out of analytics events.

**Docs portal endpoints** (browser): `/signin?returnUrl=&prompt=login`, `/signout?returnUrl=&switch=true`
(RP-initiated logout), `/auth/handoff?code=&next=`, `/search.json` (the reader's search index), `/health` (public).
Content responses are `Cache-Control: private, no-store` and `X-Robots-Tag: noindex`; forbidden pages return 403 with
an account switch, unknown ones 404.

---

## 25. Online-only vendors (EP-39)

Vendors with sales mode `OnlineOnly` (online flower shops and freelance florists with no counter and no collections)
sell finished bouquets for **delivery only, without a smart vase**. An online listing is one physical bouquet: it sells
once, then it is gone. It comes off the map when its declared freshness runs out or after
`OnlineListings:MaxListingDays` (default 3), whichever is first. The bouquet row is the lock against double selling:
two buyers racing for one listing get one success and one 409.

### 25.1 Sales mode

| Method | Path | Auth | Body | Success | Errors |
|---|---|---|---|---|---|
| GET | `/vendor/settings/sales-mode` | `VendorAccess` | — | `{ salesMode, assignedVaseCount, canSwitchToOnlineOnly }` | 404 `vendor-not-found` |
| PUT | `/vendor/settings/sales-mode` | `VendorAccess` | `{ salesMode: "InStore" \| "OnlineOnly" }` | same shape | 400 `invalid-sales-mode`; 409 `vendor-has-vases` (return the vases first, EP-22) |
| PUT | `/admin/vendors/{id}/sales-mode` | `PlatformAdmin` | same | same | ProblemDetails 400 / 404 / 409 (`type: vendor-has-vases`) |

`POST /vendor-accounts/register-with-admin` accepts an optional `salesMode` (default `InStore`). Switching back to
`InStore` is always allowed; existing online listings stay until sold or expired. Vendor list/detail DTOs
(`GET /vendors`, `GET /vendors/{id}`) include `salesMode`.

### 25.2 Create an online listing

`POST /vendor/online-listings` — `VendorAccess`, `multipart/form-data`, optional `Idempotency-Key` (a retry with the same
key returns the listing the first call created).

| Field | Rules |
|---|---|
| `name` | required, ≤ 200 |
| `price` | PLN, > 0 |
| `freshnessDays` | required, 1–3; drives expiry |
| `photo` | required, JPEG/PNG/WebP, ≤ 5 MB; stored like `POST /bouquets/{id}/photo` |
| `description`, `category` | optional |
| `flowerTypes` | optional; repeated field or comma-separated; used for customer bouquet alerts |

**Response 201:**

```json
{
  "bouquetId": "6f0c…",
  "name": "21 red roses",
  "description": "",
  "price": 199.00,
  "currency": "PLN",
  "imageUrl": "https://img.findmyflowers.pl/bouquets/6f0c….jpg?v=1761222000",
  "status": "Available",
  "listingChannel": "online",
  "freshnessDays": 2,
  "createdAt": "2026-10-23T09:00:00Z",
  "expiresAt": "2026-10-25T09:00:00Z"
}
```

Errors: 400 `photo-required` / `photo-too-large` / `invalid-listing` (validation problem details for field rules);
403 `vendor-pending-approval` or `subscription-inactive` (same gates as `POST /bouquets`); 409 `vendor-not-online` (the
vendor's sales mode is `InStore`).

Everything else reuses the bouquet endpoints: `GET /vendors/{id}/bouquets` (vendor bouquets carry `listingChannel`,
`deliveryOnly`, `expiresAt`), `PUT /bouquets/{id}`, `PUT /bouquets/{id}/price`, `POST /bouquets/{id}/photo`.
`DELETE /bouquets/{id}` on an online listing marks it `Removed` (kept for order history) and returns 400 while a customer
holds it in checkout. Online listings can never be placed in a vase.

### 25.3 Buying and fulfilment

- Discovery: §3 (`listingChannel`, `deliveryOnly`, fuzzed pin, delivery-range filter).
- Cart and checkout: §5 (`pickupAvailable`, `pickup-not-offered`, `mixed-listing-channels`, `bouquet-not-listed`, 409
  on a lost race; `vaseId: null` on cart items and order lines). Delivery fee and range are the platform's
  `Delivery:*` settings.
- Vendor orders: `Paid → Preparing → InDelivery → Delivered` as usual, with no vase or MQTT steps; `ReadyForPickup` is
  refused with `pickup-not-offered`. Delivered marks the bouquet `Sold`.
- Events: `BouquetBecameAvailable` and `BouquetSold` are raised for online listings with `vaseId: null`, so the SignalR
  new-bouquet push (§13) and customer bouquet alerts work unchanged.

---

## 26. Flower atlas (admin) (EP-41)

The platform flower atlas behind §21.6: about 150 cut flowers and greenery (`docs/atlas/atlas-seed.json`), PL/EN names
and aliases (also the recognition vocabulary — `FlowerTypeKey` takes the seed list's names), varieties, and reference
images from **Wikimedia Commons and iNaturalist under CC0, CC-BY or CC-BY-SA only**, or our own photos — never Google
Images. Every image starts `Unreviewed`; vendors see it only after it is approved. **PlatformAdmin** only; writes are
audit-logged. Base path `/api/v1/admin/atlas`.

| Method | Path | Body | Success |
|---|---|---|---|
| GET | `/flowers` | — | `AtlasFlowerSummaryDto[]`: `{ id, typeKey, kind, nameEn, namePl, scientificName, varieties, approvedImages, unreviewedImages, hasPrimary, primaryImageUrl }` |
| GET | `/flowers/{id}` | — | `AtlasFlowerDto` (below) |
| POST | `/flowers` | `{ kind, nameEn, namePl?, scientificName?, aliasesEn?, aliasesPl?, description?, colours?, searchCommons?, searchINaturalist? }` | 201 `AtlasFlowerDto`; the type key is `FlowerTypeKey` of `nameEn` (409 `duplicate` if taken) |
| PUT | `/flowers/{id}` | same + `expectedVersion` | 200 `AtlasFlowerDto`; 409 `invalid-state` on a stale version, 409 `duplicate` |
| DELETE | `/flowers/{id}` | — | 204; its images and fingerprints go with it |
| POST / PUT / DELETE | `/flowers/{id}/varieties[/{varietyId}]` | `{ name, aliases?, colours?, notes?, common }` | 200 `AtlasFlowerDto`. Names unique per entry (409 `duplicate`); `common` → offered as "Could be"; deleting keeps the images on the flower |
| POST | `/seed?updateExisting=false` | — | `{ created, existing, updated, invalid }` — creates the seed list's missing entries (idempotent); `updateExisting` overwrites names/aliases/descriptions |
| POST | `/import` | `{ flowerId?, maxFlowers?, perSource? }` | `{ flowers, added, alreadyImported, rejected, failed, remaining, messages[] }` — 2 (`perSource` 1–4) unreviewed candidates per source for one entry, or for the next `maxFlowers` (default 10) entries with fewer than 4 images; call again while `remaining > 0`. Idempotent by source URL; NC/ND/unknown licences are skipped (`rejected`) and logged, never downloaded |
| POST | `/images/{imageId}/review` | `{ action: "approve|reject|set-primary", moveToVarietyId? }` | 200 `AtlasFlowerDto`. The first approved image of a flower/variety becomes its main one (exactly one); a refused licence can't be approved (422 `licence-not-allowed`); `moveToVarietyId` re-files it (empty Guid: the flower). Approving fingerprints the image when Phase 2 is on |
| POST | `/flowers/{id}/images` | multipart `photo`, `varietyId?`, `source` (`Own` \| `Licensed`), `author?`, `licence?`, `attribution?` (required for `Licensed`), `sourceUrl?`, `approve` | 201 `AtlasFlowerDto` |
| DELETE | `/images/{imageId}` | — | 200 `AtlasFlowerDto` |
| GET | `/images/{imageId}/content` | — | image of any status (`private, no-store`) |
| POST | `/embeddings/backfill?batchSize=100` | — | `{ atlasImages, stockPhotos, failed, enabled }` — one batch of fingerprints for approved atlas images and existing stock photos without one (no-op while `Embedding:Provider` is None) |

`AtlasFlowerDto`: `{ id, version, typeKey, kind, nameEn, namePl, scientificName, aliasesEn[], aliasesPl[], description,
colours[], searchCommons, searchINaturalist, varieties: [{ id, name, aliases[], colours[], notes, common }], images: [{ id,
varietyId, url, width, height, source: "WikimediaCommons|INaturalist|Own|Licensed", sourceUrl, author, licence (SPDX),
attribution, status: "Unreviewed|Approved|Rejected", isPrimary, reviewedAt, createdAt, licenceAllowed,
attributionRequired }], updatedAt }`. Errors: 404 `not-found`, 409 `duplicate` / `invalid-state`, 422
`licence-not-allowed`, 400 `invalid-request` / `invalid-photo`, 502 `source-failed` (no sources configured).

Config `Atlas:AllowedLicences` (licence families; empty = `CC0`, `CC-BY`, `CC-BY-SA`; NC/ND are refused whatever is
listed), `Atlas:Import:CandidatesPerSource` 2, `MaxFlowersPerRun` 10, `EnoughImages` 4, `MaxLongEdge` 1024,
`MaxImageBytes` 400000, `MaxDownloadBytes` 15000000, `UserAgent` (Wikimedia API etiquette), `CommonsApiUrl`,
`INaturalistApiUrl`. Images are stored with `IFileStorageService` under `atlas/{flowerId}/{imageId}.jpg` (public R2 bucket,
`img.findmyflowers.pl/atlas/...`).
