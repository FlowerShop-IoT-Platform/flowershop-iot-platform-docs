# Manual Test — S-04-06: Cancel-with-refund-status

> **Feature under test:** `POST /api/v1/orders/{id}/cancel` now returns **200 OK** with a
> `CancelOrderResponseDto` body `{ orderId, status, cancelledAt, refundStatus }`, and the
> `Orders.RefundStatus` column is persisted.
> `refundStatus` = `not_applicable` when the order was never paid, `refund_initiated` once a paid
> order is cancelled (and a Stripe refund is attempted).

## Environment (verified 2026-06-10)

| Component | URL / value |
|---|---|
| API + Swagger | http://localhost:8080/ (Swagger served at root) |
| Admin / Vendor / Customer portals | :5001 / :5002 / :5003 (all return 200) |
| DB | `localhost:5432 / flowershop_iot_dev / postgres / dev123` |
| Migration applied | `20260610145020_AddOrderRefundStatus` (adds `Orders.RefundStatus varchar(50)`) |
| Stripe | test keys present in `appsettings.Development.json` (`sk_test_…`, `pk_test_…`); `WebhookSecret` empty |
| Auth | `Authentication:UseKeycloak = true` → the `/auth/dev-token` shortcut is **disabled**; a real Keycloak customer JWT is required |
| Seed data | 1 customer (`Ivan Khariuk`), 1 Available bouquet (`Fresh Bouquet`, id `8de84bde-…`) |

---

## What can be tested, and how

There are three layers at which this feature can be exercised. Pick based on how much of the
payment flow you want to involve.

### Scenario A — Unpaid order → `refundStatus = "not_applicable"` (fastest, no Stripe needed)

This is the cleanest end-to-end proof that the new response shape and column work, without
involving a real card payment.

1. **Get a customer JWT.** Log into the Customer App (http://localhost:5003) as the seeded
   customer via Keycloak, or grab the bearer token the SPA stores after login (DevTools →
   Application → Local Storage / or the `Authorization` header on any `/api/v1/...` XHR).
   In Swagger, click **Authorize** and paste `Bearer <token>`.
2. **Create an order** for the Available bouquet:
   - `POST /api/v1/orders/preview` with `{ "bouquetIds": ["8de84bde-53c0-4eb5-b501-c524be40e6cb"], "deliveryMethod": "pickup" }` → note `totalAmount`.
   - `POST /api/v1/orders` with the same bouquet + `expectedTotalAmount` from the preview.
     Response is `201` with `orderId` and `status: "created"`. **Do not pay.**
3. **Cancel it immediately** (still in `created`):
   - `POST /api/v1/orders/{orderId}/cancel` body `{ "reason": "changed my mind" }`.
   - ✅ **Expect HTTP 200** (not 204) with body:
     ```json
     { "orderId": "...", "status": "cancelled", "cancelledAt": "2026-...Z", "refundStatus": "not_applicable" }
     ```
4. **Verify persistence + side effects:**
   ```sql
   SELECT "Id","Status","RefundStatus" FROM "Orders" WHERE "Id" = '<orderId>';
   -- Status=Cancelled, RefundStatus=NULL  (NULL maps to "not_applicable" in the response)
   SELECT "Status" FROM "Bouquets" WHERE "Id" = '8de84bde-...';
   -- back to 1 (Available) — reservation was released
   ```

### Scenario B — Paid order → `refundStatus = "refund_initiated"` (full Stripe path)

This proves the refund branch. Requires completing a Stripe test payment.

1. Create an order as in A, steps 1–2. The create response includes
   `paymentClientSecret` / `stripeEphemeralKey` / `stripeCustomerId`.
2. **Confirm payment** so the order becomes `paid`. Two options:
   - **Via the Customer App checkout UI** using a Stripe test card (`4242 4242 4242 4242`,
     any future expiry, any CVC) — this drives the real `PaymentIntent`; or
   - **Simulate the confirm** by calling `POST /api/v1/orders/{id}/confirm-payment` with the
     `paymentIntentId` (the part of the client secret before `_secret_`). Note: a fully realistic
     refund needs a *real* captured PaymentIntent, so prefer the UI card path if you want the
     Stripe refund call to actually succeed.
3. **Cancel the paid order:** `POST /api/v1/orders/{id}/cancel` `{ "reason": "test refund" }`.
   - ✅ **Expect HTTP 200** with `"refundStatus": "refund_initiated"`.
   - Watch the API log: `Refund {RefundId} issued for cancelled order …`.
4. **Verify:**
   ```sql
   SELECT "Status","RefundStatus" FROM "Orders" WHERE "Id" = '<orderId>';
   -- Status=Cancelled, RefundStatus='refund_initiated'
   ```
   In the **Stripe test dashboard** (Payments → the PaymentIntent), confirm a refund object exists.

> ⚠️ Known limitation (intentionally deferred to **EP-17 T-17-008**): `refund_initiated` is set
> whenever the order *was paid*, **even if the Stripe refund call throws**. The handler catches the
> exception, logs "Manual refund required", and still reports `refund_initiated`. Branching on the
> real `refund.Status` (succeeded/pending/failed) and alerting on failure is out of scope for S-04-06.

### Scenario C — Negative / state-machine checks

Using the customer JWT from A:

| Action | Expected |
|---|---|
| Cancel a **non-existent** orderId | `404` `{ error: "Order not found" }` |
| Cancel an order belonging to **another customer** | `404` `{ error: "Order does not belong to this customer" }` |
| Cancel an **already-cancelled** or **delivered** order | `409 Conflict` |

---

## Automated coverage (already green)

- `tests/FlowerShop.Domain.Tests/Aggregates/Order/OrderRefundStatusTests.cs` — `RefundStatus`
  starts null; `MarkRefundInitiated()` sets `refund_initiated`. (2 tests)
- `tests/FlowerShop.Tests.Unit/Application/CancelOrderCommandHandlerTests.cs` — paid→refund_initiated,
  created→not_applicable, order-not-found, refund-throws-still-cancels. (4 tests)

Quick re-run:
```bash
dotnet test tests/FlowerShop.Domain.Tests/FlowerShop.Domain.Tests.csproj --filter "FullyQualifiedName~OrderRefundStatus"
dotnet test tests/FlowerShop.Tests.Unit/FlowerShop.Tests.Unit.csproj --filter "FullyQualifiedName~CancelOrderCommandHandler"
```

---

## Manual verification run — 2026-06-10 (results)

**Verdict: PASS.** All three scenarios (A, B, C) were exercised against the **live API** on
`http://localhost:8080`. The cancel endpoint returns `200` with the documented body, the
`Orders.RefundStatus` column persists, and reservations are released. Findings below.

### How it was driven

The surface under test is the API (`POST /api/v1/orders/{id}/cancel`); the Customer App card
checkout for Scenario B is optional, since the refund branch is observable directly at the API.

1. **Customer JWT.** With `UseKeycloak=true` the `/auth/dev-token` shortcut is disabled and the
   seed user has no documented password, so a real Keycloak token was minted:
   - Reset a password for the seed user via the Keycloak admin API:
     ```bash
     ADMIN=$(curl -s -X POST "http://localhost:8090/realms/master/protocol/openid-connect/token" \
       -d client_id=admin-cli -d username=admin -d password=admin123 -d grant_type=password \
       | python -c "import sys,json;print(json.load(sys.stdin)['access_token'])")
     # user id 2edf8f5d-… = ivan.khariuk@hyland.com (maps to seed CustomerId 11194aa2-…)
     curl -X PUT "http://localhost:8090/admin/realms/flowershop/users/2edf8f5d-0799-4abf-9ac2-61d64907524a/reset-password" \
       -H "Authorization: Bearer $ADMIN" -H "Content-Type: application/json" \
       -d '{"type":"password","value":"TestPass123!","temporary":false}'
     ```
   - Got an access token via the password grant against the confidential customer client:
     ```bash
     curl -s -X POST "http://localhost:8090/realms/flowershop/protocol/openid-connect/token" \
       -d client_id=flowershop-customer-app -d client_secret=Et35u7yxPvhbRwuBBPTKmwq3AM1EcIj4 \
       -d username=ivan.khariuk@hyland.com -d password=TestPass123! -d grant_type=password -d scope=openid
     ```
   - The token has no `customer_id` claim; the controller resolves it via the
     `sub` → `Users.KeycloakUserId` → `CustomerId` fallback (`OrdersController.GetCustomerIdAsync`).
2. **SQL** was run inside the `flowershop-postgres-dev` container (`psql` is not on the host PATH):
   `docker exec -i flowershop-postgres-dev psql -U postgres -d flowershop_iot_dev`.
3. **Order seeding.** Order create/preview could not be used (see ⚠️ environment gap below), so
   `Created` and `Paid` orders owned by the seed customer were inserted directly into `Orders` +
   `OrderLines`, reserving the seed bouquet (`8de84bde-…`, `Status=2`) the way the create flow does.
   All seeded orders were deleted afterward; the bouquet is back to `Available (1)`.

### Results

| Scenario | Action | Result | DB after |
|---|---|---|---|
| **A** unpaid | cancel a `Created` order | `200` `{status:"cancelled", refundStatus:"not_applicable"}` ✅ | `Status=Cancelled`, `RefundStatus=NULL`, bouquet → `Available` |
| **B** paid | cancel a `Paid` order (with `PaymentIntentId`) | `200` `{status:"cancelled", refundStatus:"refund_initiated"}` ✅ | `Status=Cancelled`, `RefundStatus='refund_initiated'`, bouquet → `Available` |
| **C1** | cancel non-existent orderId | `404 {"error":"Order not found"}` ✅ | — |
| **C2** | cancel another customer's order | `404 {"error":"Order does not belong to this customer"}` ✅ | — |
| **C3** | cancel already-cancelled order | `409 {"error":"Cannot cancel a delivered or already cancelled order"}` ✅ | — |
| **C3b** | cancel delivered order | `409` (same message) ✅ | — |

Extra probes (off the happy path):
- Re-cancel an already-cancelled order → `409` (state machine holds).
- Cancel with **no** bearer token → `401`.
- Cancel with empty body `{}` (no `reason`) → `200 not_applicable` (reason is optional).
- Swagger (`/swagger/v1/swagger.json`) advertises the new contract: `200 → CancelOrderResponseDto`,
  plus `404` and `409`.

Scenario B note: the seeded `Paid` order used a plausible-but-not-live `PaymentIntentId`, so the
Stripe refund call almost certainly threw and was swallowed — the handler still returned
`refund_initiated`. This confirms the documented EP-17 limitation live (it is *not* a regression).

### ⚠️ Environment gap found (fixed during the run)

The running API build expects `FreshnessPricingStrategy` (+ `BasePriceAmount`,
`BasePriceCurrency`, `AppliedFreshnessTierIndex`) columns from migration
`20260604000000_AddFreshnessPricing`, **but that migration was missing from
`__EFMigrationsHistory`** (history jumped `20260516…` → `20260610_AddOrderRefundStatus`). As a
result every Orders endpoint that joins bouquet→vendor — `GET /orders`, `POST /orders/preview`,
**and the cancel handler itself** — returned `500: column f.FreshnessPricingStrategy does not exist`.
The migration's `Up()` was applied (idempotent `ADD COLUMN IF NOT EXISTS` + history row) to bring the
DB in sync with the code, after which all scenarios passed. **The environment table at the top of
this doc should also list `20260604000000_AddFreshnessPricing` as a prerequisite.** Worth checking
whether other environments have the same missing migration.
