# Requirement Gaps — Option Q (Online-only vendors: bouquets without a smart vase)

> Companion to REQ-GAPS.md (mobile API: discovery, orders), REQ-GAPS-C.md (onboarding, vases) and
> REQ-GAPS-O.md (EP-37 custom orders, which already sell without a vase).
> Requirement refs point at the tasks in `tasks-ep39-online-only-vendors.json`.
> Analysis date: 2026-10-23
> Status: implemented in this repo (EP-39, see `EPICS-Q.md`); mobile follow-ups REQ-ONL-17/18 open. Section H decided
> 2026-10-23.

## Context

The smart vase exists to stop **online and offline customers buying the same physical bouquet**.
The bouquet sits in a shop and the vase shows the QR code and its state (available, reserved, sold).
Some vendors only sell online: online flower shops and freelance florists with no shop counter.
Nobody walks in and nobody collects, so the vase has nothing to protect. Today these vendors cannot
sell at all, because the purchase path requires a vase.

### What the code does today

| Area | Behaviour | Where |
|---|---|---|
| Domain | `Bouquet.VaseId` is already nullable. `CreateCatalog` makes vase-less bouquets that go straight to `Available` when given a price. | `Domain/Aggregates/Bouquet/Bouquet.cs` |
| Map | Vase-less `Available` bouquets **are shown**: the offline-vase filter is `b.VaseId == null \|\| !offline`. | `GetNearbyBouquetsQueryHandler`, `GetAvailableBouquetsQueryHandler` |
| Cart | **Hard block**: `"Bouquet is not placed in a vase"`. | `CartCommandHandlers.cs:65` |
| Checkout | **Hard block**: `"… is not placed in a vase"`. | `OrderCommandHandlers.cs:225` |
| Persistence | `CartItem.VaseId` and `OrderLine.VaseId` are non-nullable (domain + `IsRequired()`). | `Cart.cs`, `Order.cs`, `CartConfiguration`, `OrderConfiguration` |
| Read models | `line.VaseId.Value` / `i.VaseId.Value` dereferences. | `VendorOrderHandlers.cs:345`, `OrderQueryHandlers.cs:157`, `CartQueryHandlers.cs:46` |
| Side effects | Every MQTT command (cart add, `prepare_delivery`, `delivery_started`, `delivery_complete`, reservation release) and vase state reset is **already guarded** by `VaseId != null`. | cart/order/vendor-order handlers, `OrderPaidVaseNotificationHandler`, `CartReservationReleaser` |
| Events | `BouquetBecameAvailable` and `BouquetSold` are raised **only when `VaseId != null`**. They drive SignalR "new bouquet" pushes and customer bouquet-alert matching (`SubscriptionMatchingService`). | `Bouquet.SetPrice`, `Bouquet.Sell` |
| Vendor UX | "Add a bouquet" goes to `/Vases/Index`. Creating a bouquet means placing it in a vase. The dashboard is built around vases. | VendorPortal `_Layout.cshtml`, `Index.cshtml.cs` |
| Concurrency | `Bouquet.Version` exists as a column but is **not a concurrency token**, and `Reserve` doesn't increment it. Two simultaneous online buyers can both reserve. | `FlowerShopDbContext` (Bouquets) |
| Precedent | EP-37 custom orders (`Order.CreateCustom`, `CustomOrderLine`) already sell, pay and deliver with **no vase**. | `Order.cs` |

So the vase is only *structurally* required in two guards, two non-nullable columns and three
`.Value` reads. The rest of the purchase path already works without one.

### Virtual vase vs. vase-optional listing

A virtual vase (a `SmartVase` row with no device) would keep `VaseId` non-null. But a `SmartVase`
is an IoT asset, and most of the code that reads it would treat a virtual one wrongly:

| A virtual vase would… | Because |
|---|---|
| hide its bouquets from the map after 30 s | `GetEffectivelyOfflineVaseIdsAsync(30 s)` — unless we fake heartbeats |
| open asset-recovery cases after 7 days offline | EP-22 `VaseHealthMonitoringService` → `VaseIncident` |
| receive MQTT commands for a device that doesn't exist | all the vase-guarded commands above |
| hold only **one** bouquet | placing a bouquet removes the previous one (`GetActiveBouquetInVaseAsync` → `Remove`), so 10 listings need 10 fake vases |
| distort fleet data | vase inventory, `AssignedVaseCount`, vase analytics, admin fleet views, active-vase dashboard tiles |
| need a fake `DeviceIdentifier`, registration status and assignment | `SmartVase.Create` / `AssignToVendor` |

Each of these needs an `IsVirtual` check inside IoT code that should never have to know about it.
That is more code and more risk than making the vase optional, and it hides the real business
concept.

**Decision (proposed): no virtual vase.** Add an explicit **listing channel** on the bouquet and a
**sales mode** on the vendor. Purchase rules check "is this bouquet purchasable?" instead of "does it
have a vase?". Vase-specific side effects stay exactly as they are, behind the guards that already
exist. The bouquet's own status (`Available → Reserved → Sold`) becomes the only lock against double
selling. That is all an online-only bouquet needs, as long as it is made concurrency-safe
(REQ-ONL-06).

```
                     ┌──────────── Bouquet (one physical unit) ────────────┐
 Vendor.SalesMode    │ ListingChannel = Vase    → VaseId required          │
  InStore ──────────▶│   lock: vase display + bouquet status               │
  OnlineOnly ───────▶│ ListingChannel = Online  → VaseId null              │
                     │   lock: bouquet status (optimistic concurrency)     │
                     └─────────────────────────────────────────────────────┘
 IsPurchasable = Status == Available && (Channel == Online || VaseId != null)
```

---

## A. Vendor sales mode

### REQ-ONL-01 — Vendor sales mode — P0
`IVendor` gains `SalesMode` (`InStore` = 0, the default for every existing vendor; `OnlineOnly` = 1).
It applies to both `FlowerShop` and `FreelanceFlorist` and is stored on their tables.
`SetSalesMode(mode)` returns `Result`. Switching to `OnlineOnly` fails with `vendor-has-vases` while
the vendor has assigned vases. The vendor must return them first through the EP-22 return flow, or
an admin unassigns them. Switching back to `InStore` is always allowed, and existing online listings
stay valid until sold or expired.
**Accept**: domain tests for both transitions; migration defaults existing rows to `InStore`.

### REQ-ONL-02 — Choose the mode at registration and in settings — P1
The vendor registration API (`POST /api/v1/vendor-accounts/register-with-admin`) accepts an optional
`salesMode` (default `InStore`). The Vendor Portal sign-up wizard asks "How do you sell?" with two
choices: "In my shop, with smart vases" and "Online only, delivery to customers". The Vendor Portal
Settings page shows the mode and lets the vendor switch it (`GET`/`PUT /api/v1/vendor/settings/sales-mode`),
with the REQ-ONL-01 error shown as a friendly message.
**Accept**: an online-only registration never shows vase-ordering prompts (EP-21).

### REQ-ONL-03 — Admin visibility and override — P2
The Admin Portal vendor list and details show the sales mode. A platform admin can change it, with
the same rule as REQ-ONL-01.

---

## B. Domain: online listings

### REQ-ONL-04 — Listing channel on the bouquet — P0
`Bouquet` gains `ListingChannel` (`Vase` = 0, the default for every existing row; `Online` = 1).
A new factory `Bouquet.ListOnline(vendorId, vendorType, name, description, price, currency, imageUrl,
flowerTypes, category, freshnessDays)` creates an `Online` bouquet with `VaseId = null`. Rules:
price > 0, an image is required (customers can't see it in person), and freshness 1..3 is required
because it drives expiry (REQ-ONL-08). Status goes straight to `Available`. A pricing-history entry
and `BouquetBecameAvailable` are recorded.
Invariants: an `Online` bouquet can never get a vase (assignment returns a failure). A `Vase`
bouquet keeps its current behaviour.
`Bouquet.IsPurchasable` = `Status == Available && (ListingChannel == Online || VaseId != null)`.
**Accept**: domain tests for the factory, the invariants and `IsPurchasable`.

### REQ-ONL-05 — Events don't depend on the vase — P0
`BouquetBecameAvailable.VaseId` and `BouquetSold.VaseId` become `VaseId?`, and both events are
raised for `Online` bouquets as well (in `ListOnline`, `SetPrice` and `Sell`). Their handlers then
work for online listings with no other change: the SignalR new-bouquet push, customer bouquet-alert
matching (`SubscriptionMatchingService`) and the sold notification. Any handler that uses the vase id
must handle null.
**Accept**: an online listing triggers a matching customer alert and a `BouquetHub` push.

### REQ-ONL-06 — Concurrency-safe reservation — P0
With no vase display, the bouquet row is the only lock. Map `Bouquet.Version` as a concurrency token
and increment it in `Reserve`, `ReleaseReservation`, `Sell`, `Remove`, `Expire` and the price
changes. When two buyers race for the same bouquet, exactly one wins. The other gets the existing
"no longer available" / `ReservedByAnotherCustomer` result (HTTP 409), not a 500. This also fixes the
same race for vase bouquets.
**Accept**: an integration test with two parallel add-to-cart (and two parallel checkout) calls on
one bouquet gives one success and one 409.

### REQ-ONL-07 — Nullable vase on cart and order lines — P0
`CartItem.VaseId` and `OrderLine.VaseId` become `VaseId?`, with nullable columns and `IsRequired()`
removed. DTOs `VaseId` become `Guid?`: cart, order, vendor order line and customer app models,
including `VaseIdDtoJsonConverter`. The three `.Value` reads become null-safe. Migration
`AddOnlineOnlyListings` adds `SalesMode` (two vendor tables), `ListingChannel` (Bouquets), and makes
`CartItems.VaseId` and `OrderLines.VaseId` nullable. Existing data is unchanged.
**Accept**: the migration applies on a copy of prod data; existing orders still read.

### REQ-ONL-08 — Online listing lifetime — P1
An `Online` bouquet is a real, finished bouquet waiting in the florist's studio. It sells once,
just like a vase bouquet. It has no sensors, so a background pass (the existing freshness or
heartbeat hosted service, or a new `OnlineListingExpiryService`) expires `Available` online
listings whose `CurrentFreshnessDays` reached 0, or that are older than
`OnlineListings:MaxListingDays` (default 3). A reserved bouquet is never expired in the middle of
checkout; it is expired after the reservation is released. The vendor can remove a listing at any
time (`Remove(VendorRequest)`, which is already vase-agnostic).
**Accept**: unit test of the expiry rule; expired listings disappear from the map.

---

## C. Purchase flow

### REQ-ONL-09 — Cart and checkout use `IsPurchasable` — P0
Replace the two "not placed in a vase" guards (`AddToCartCommand`, `CreateOrderCommand`) with
`bouquet.IsPurchasable`. The new failure code is `bouquet-not-listed` (HTTP 409/422 per the existing
mapping). It replaces the vase message and also covers legacy vase-less `Vase`-channel bouquets.
Cart and order lines take `bouquet.VaseId` (null for online).
**Accept**: an online bouquet can be added to the cart, checked out and paid with Stripe (stub and
real), and a legacy vase-less catalog bouquet still cannot.

### REQ-ONL-10 — Delivery only for online listings — P0
An order containing any `Online` line must use `DeliveryMethod.Delivery`. `Pickup` fails with
`pickup-not-offered`. `POST /api/v1/orders/preview` reports `pickupAvailable: false` so clients can
hide the choice. The delivery address, geocoding, fee and `Delivery:MaxDistanceKm` range checks are
the existing ones (distance from the vendor location, `VendorLocationResolver`). An order cannot mix
`Vase` and `Online` lines. The existing one-vendor-per-order rule makes this rare, but it must still
be checked, because it would make the pickup rule ambiguous.
**Accept**: handler tests for pickup refused, mixed lines refused and out-of-range delivery refused.

### REQ-ONL-11 — Vendor fulfilment without vase side effects — P1
Vendor order transitions (`Paid → Preparing → InDelivery → Delivered`) work for online lines. Vase
steps (clear vase, MQTT `delivery_started`/`delivery_complete`, vase state reset) are skipped by the
existing null guards. `Delivered` still marks the bouquet `Sold`. `ReadyForPickup` is not offered
for online orders.
**Accept**: a handler test walks an online order from paid to delivered with no
`ISmartVaseRepository`/`IMqttService` call.

---

## D. Discovery

### REQ-ONL-12 — The map shows only purchasable bouquets — P0
`GetNearbyBouquets` and `GetAvailableBouquets` include a bouquet only when `IsPurchasable` (and the
vase is online, for `Vase` bouquets). This **fixes a current leak**: vase-less catalog bouquets with
a price appear on the map today, but customers can't buy them.
**Accept**: a query test with one vase bouquet, one online bouquet and one vase-less catalog
bouquet returns the first two.

### REQ-ONL-13 — Delivery-only presentation and location — P1
`NearbyBouquetDto` and `BouquetDetailDto` gain `listingChannel` (`"vase"`/`"online"`) and
`deliveryOnly: true` for online listings. The map pin sits at the vendor's dispatch location, and an
online-only vendor's pin is fuzzed like freelance florists' already are (`ExactAddressHidden`). An
online shop has no counter to walk to, so the exact address is not useful to the customer and is a
privacy cost to the vendor. Online listings are returned only when the customer is within delivery
range: `min(radiusKm, Delivery:MaxDistanceKm)`. The distance shown is "delivers to you", not "walk
here".
**Accept**: contract update; a customer outside delivery range does not see online listings.

---

## E. Vendor API and portals

### REQ-ONL-14 — Create an online listing — P0
`POST /api/v1/vendor/online-listings` (`VendorAccess`, multipart: fields + 1 photo) runs
`ListOnlineBouquetCommand`. The command checks the same gates as `CreateBouquetCommand` (approval
EP-19, subscription EP-20), stores the photo through `IFileStorageService` (R2) with the same key
scheme as `/bouquets/{id}/photo`, and returns the bouquet. It refuses with `vendor-not-online` when
the vendor's `SalesMode` is `InStore`. Listing, editing, price change, photo replacement and removal
reuse the existing bouquet endpoints, which are already vase-agnostic or become so through REQ-ONL-07.
`Idempotency-Key` is supported, as in EP-30 stock movements.
**Accept**: controller tests for 201, 403 (other tenant), 409 `vendor-not-online`, 402/403 subscription
inactive.

### REQ-ONL-15 — Vendor Portal for online-only vendors — P0
When `SalesMode == OnlineOnly`:
- "Add a bouquet" (app bar and nav) opens the new `Pages/Bouquets/Create` (name, photo, price,
  freshness, flower types, description) instead of `/Vases/Index`.
- The "My Vases" nav, the dashboard vase tiles and active-vase counts, and the vase-ordering prompts
  (EP-21) are hidden. The `Vases/*` pages redirect to the Bouquets page with an explanation.
- The Bouquets list shows an "Online" badge, the expiry countdown (REQ-ONL-08) and "Remove listing".
  `SetPrice` hides vase-only actions.
- AI Florist (EP-28/33) "save recipe" can prefill the Create form (name, description, flower types,
  suggested price).
**Accept**: Playwright/portal tests for both modes; an InStore vendor's portal is unchanged.

### REQ-ONL-16 — Customer App — P1
The Customer App map and detail pages show a "Delivery only" badge for online listings, and checkout
hides Pickup when `pickupAvailable` is false. Any "find the vase / scan the QR in the shop" wording is
hidden for online listings. The QR short-code share link (`/b/{code}`) keeps working.

---

## F. Mobile follow-ups

### REQ-ONL-17 — Vendor mobile app — P1
EP-M19 in the vendor app repo: the sales mode in settings, an "Add online bouquet" flow (camera photo
→ `POST /vendor/online-listings`), hiding vase screens for online-only vendors, and the "Online" badge
with expiry. The API is ready after S-39-05.

### REQ-ONL-18 — Customer mobile app — P1
Customer mobile app repo: the "Delivery only" badge, hiding pickup when `pickupAvailable` is false,
and handling `bouquet-not-listed` / `pickup-not-offered`.

---

## G. Quality

### REQ-ONL-19 — Tests, contract and docs — P1
Domain, handler and integration tests for REQ-ONL-01…14. `docs/API-CONTRACTS.md` gets a new §25
"Online-only vendors (EP-39)" and changes to §3 (`listingChannel`, `deliveryOnly`) and §5
(`pickupAvailable`, the new error codes). The Vendor Portal and Customer App manuals get an "online
only" section (vendor/customer audiences, EP-38), plus a changelog entry.

---

## H. Decisions (decided 2026-10-23: all four as proposed)

1. **Hybrid vendors.** Should a shop with vases also be able to list online-only, made-to-order
   bouquets that are not on display (`SalesMode = Hybrid`, with a per-bouquet choice)? The model
   supports it: `ListingChannel` is per bouquet, so it would be one more enum value and a picker on
   Create. *Proposed: not in v1.* **Decided: not in v1.**
2. **Collection by appointment.** Should some online-only florists allow pickup from their studio?
   It is out of scope per the brief ("no collections expected"). If it is needed later, add a
   per-vendor `OffersPickup` flag rather than reintroducing a vase. **Decided: no pickup in v1.**
3. **Repeatable products.** Online shops often sell the same arrangement many times ("21 red roses").
   The `Bouquet` aggregate is one physical unit, and v1 keeps it that way: one listing, one sale.
   A catalog product with quantity or made-to-order capacity would be a separate epic (it could
   reuse EP-30 stock to check capacity). **Decided: one listing, one sale in v1.**
4. **Per-vendor delivery radius.** v1 uses the platform-wide `Delivery:MaxDistanceKm` and fee.
   Online shops may want their own radius and fee. **Decided: platform-wide radius and fee in v1.**
