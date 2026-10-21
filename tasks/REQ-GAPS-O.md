# Requirement Gaps — Option O (Customer bouquet requests: map pins, vendor offers, accepted order with countdown)

> Companion to REQ-GAPS.md (mobile API, orders, notifications), REQ-GAPS-B.md (customer web app)
> and REQ-GAPS-C.md (vendor platform subscription).
> Requirement refs point at the tasks in `tasks-ep37-customer-bouquet-requests.json`.
> Analysis date: 2026-10-20
> Status: REQ-BRQ-01 … 22 planned (EP-37, see `EPICS-O.md`).

## Context

Today the marketplace only goes one way: vendors put bouquets in smart vases and customers buy what
is listed (`Order` → `OrderLine` → `BouquetId` + `VaseId`). A customer who wants something specific
(a reference photo from Pinterest, "like this but in blue") has no way to ask.

This epic adds the reverse flow, a **bouquet request**:

```
Customer: photos + note + budget + when + where ──▶ pin on the vendors' map (approximate location)
Vendors in range: offer = price + ready/delivery time + notes (substitutions) + photos
Customer: compares offers ──▶ accepts ONE ──▶ others rejected ──▶ pays ──▶ Order at that vendor
Order: countdown to the delivery time · phases · "need more time" · late alerts · delivered
```

What exists and what is missing:
- `SavedAddress` has street/city/postal code but **no coordinates**. Geocoding already exists
  (`IGeocodingService` / `IAddressLookupService` in `Infrastructure/Geocoding`: Nominatim, cached,
  rate-limited, with search and reverse lookup), so the pin reuses it and adds a manual adjustment.
- `Order`/`OrderLine` require a `BouquetId` and `VaseId`, so a custom order needs a new line type.
- `Notification` is **customer-only** (`CustomerId`) and push is a no-op stub
  (`NotificationServiceClient`; no FCM client, EP-06 pending). Vendors have no notification inbox
  (only `VendorMessage`).
- SignalR has one hub (`BouquetHub`), customer-facing.
- Stripe checkout for orders is fully wired (`StripeService`) and can be reused.

**Principles**
1. **Customer privacy first.** Vendors see an approximate location (fuzzed to a ~500 m cell) and
   district only. The exact address, name and phone are revealed to the chosen vendor after
   payment, and never to the others.
2. **Requests are vendor-only content.** They never appear on the public/customer map, and a
   customer sees only their own requests.
3. **One winner, atomically.** Accepting an offer rejects all other offers in the same transaction.
4. **Time is a contract.** Every accepted order has a committed delivery time. Deviations are
   announced, never silent.

---

## A. Customer creates a request

### REQ-BRQ-01 — Who can request — P0
Only signed-in customers (`CustomerAccess`, realm role `User`) with a **verified email** may
create requests. Limits: ≤ 3 open requests per customer, ≤ 10 per rolling 7 days. Suspended
customers cannot create them.
**Accept**: anonymous → 401; limits → 429 `request-limit` with `Retry-After`.

### REQ-BRQ-02 — Request content — P0
- 1–5 reference images (JPEG/PNG/WebP ≤ 8 MB raw; re-encoded ≤ 1600 px JPEG, EXIF incl. GPS
  stripped, stored privately via `IFileStorageService` and served only through authorised
  endpoints);
- description ≤ 1000 chars ("like the first photo but with white roses");
- optional budget range (min/max PLN), optional occasion chip, optional size S/M/L;
- **desired delivery time**: ≥ now + 3 h, ≤ now + 30 days, in Europe/Warsaw shown to the user, UTC
  stored;
- **offer deadline** (default: min(desired − 2 h, now + 48 h)), editable within those bounds;
- delivery or pickup (default delivery).

**Accept**: validation errors are ProblemDetails with field names; images show in previews before
submit.

### REQ-BRQ-03 — Address as the map pin — P0
The request needs a delivery address chosen from saved addresses or entered new (saved as
`SavedAddress` on consent). `SavedAddress` gains `Latitude`/`Longitude` and a `GeocodedAt`. The existing
`IGeocodingService` (Nominatim, cached, rate-limited) geocodes the address, and
`IAddressLookupService` powers address search and the reverse lookup for a dragged pin. The customer then confirms or drags the pin on a Leaflet map (the same library as
EP-13). Delivery requests outside Poland, or without a confirmed pin, are rejected.
**Accept**: geocoding failure → the customer places the pin manually; coordinates are stored with
6 decimals and never sent to vendors before acceptance (REQ-BRQ-06).

### REQ-BRQ-04 — Manage own requests — P0
The customer can list their requests (state, offers count, deadline) and edit a request while it
has no offers (changing details afterwards notifies the vendors with offers and marks those
offers "request changed"). They can also cancel an open request, which rejects all offers and
notifies the vendors. A request with no offers by the deadline expires; the customer gets
"No offers yet — extend or cancel?" 2 h before the deadline.
**Request states**: `Open → OffersReceived → Accepted → (Order) | Expired | Cancelled`.

---

## B. Vendors find requests

### REQ-BRQ-05 — Vendor map and list — P0
In the **vendor app** (and Vendor Portal, REQ-BRQ-20), a "Requests" map layer and a list show open
requests whose approximate location is inside the vendor's **service radius**. The radius is a
new vendor setting, default 15 km, 1–50 km, measured from the shop location or the freelancer's
base. Each pin card shows the first image thumbnail, budget, desired time, time left to offer,
distance and the number of competing offers (count only).
**Accept**: only `VendorAccess` users with an **active platform subscription** (EP-20) see
requests. Customers can never query this layer. The geo query uses the same index approach as
bouquet discovery (EP-03) and is paged/bounded by the viewport.

### REQ-BRQ-06 — Location privacy — P0
Before payment, vendors receive a location fuzzed to a fixed ~500 m grid cell (deterministic per
request, so repeated reads don't triangulate) and the district/city name, never the street,
customer name, email or phone.
**Accept**: API tests assert that no exact coordinates or PII appear in any vendor-facing DTO
before the order is paid.

### REQ-BRQ-07 — New request alerts — P1
Vendors in range get "New bouquet request 3.2 km away, Sat 14:00, budget 150–200 PLN" (in-app
inbox + SignalR + push when available). They can opt out, or set quiet hours and a minimum budget.
**Accept**: alert fan-out is rate-limited per vendor (≤ 20/hour, then a digest).

---

## C. Offers

### REQ-BRQ-08 — Make an offer — P0
A vendor opens a request and submits an **offer**:
- price (PLN, incl. VAT; delivery fee shown separately or "included");
- **ready time** and, for delivery, the **delivery time** (≤ desired time + 2 h; later times are
  flagged "later than requested");
- notes ≤ 1000 chars, e.g. substitutions ("peonies are out of season, I'd use garden roses");
- 0–5 images (substitute flowers, a sample of their work, an AI Florist preview);
- optional link to an AI Florist recipe (EP-28) for the vendor's own use (not shown to the customer).

**Reconstruct with AI Florist**: from the request, the vendor can open an EP-35 reconstruction of
the customer's images (REQ-INSP-11). It shows what they have, what is missing, an estimated cost
and a draft substitution note to copy into the offer.

One active offer per vendor per request. Offer validity defaults to the request's offer deadline.
**Accept**: offers after the deadline or on non-open requests → 409 `request-closed`.

### REQ-BRQ-09 — Change or withdraw an offer — P0
Before acceptance, the vendor can revise an offer (a new revision is kept; the customer sees
"updated" and the latest revision) or withdraw it. After acceptance, only the "need more time"
flow (REQ-BRQ-15) changes times.
**Accept**: revising an offer the customer is accepting at the same moment → one of them gets
409 `version-conflict`; the accepted revision is the one the customer saw (`offerRevision` in the
accept call).

### REQ-BRQ-10 — Customer compares offers — P0
The customer sees all offers for a request: vendor name, rating (if any), distance, price, ready
and delivery time, the "later than requested" flag, notes, images. They can sort by price, time
or distance and ask a question (P2: request-scoped message thread, REQ-BRQ-21).
**Accept**: offers of vendors that were suspended in the meantime are hidden.

---

## D. Acceptance and order

### REQ-BRQ-11 — Accept exactly one offer — P0
`POST /customer/bouquet-requests/{id}/offers/{offerId}/accept { offerRevision }`. In one
transaction: the request becomes `Accepted`, the chosen offer `Accepted`, every other offer
`Rejected` (reason `another-offer-accepted`), and an `Order` is created at that vendor. Rejected
vendors are notified. A second accept → 409 `already-accepted`.
**Accept**: concurrency test with two parallel accepts yields one order.

### REQ-BRQ-12 — Payment — P0
Acceptance creates the order in `Created` and starts the existing Stripe flow (PaymentIntent,
BLIK/card as in EP-18). The vendor's commitment starts when the order is `Paid`. If payment does
not complete within 30 min, the acceptance lapses: the order is cancelled, the request returns to
`OffersReceived` with the other offers restored, and the vendors are notified.
**Accept**: webhook-driven, idempotent; refund rules of EP-17 apply to later cancellations.

### REQ-BRQ-13 — Custom order line — P0
`OrderLine` today needs `BouquetId` + `VaseId`. Add a line kind `Custom` (nullable bouquet/vase,
`BouquetRequestId`, `BouquetOfferId`, snapshot of title/notes/first image) or a sibling
`CustomOrderLine`. Vendor order management (EP-05/EP-12) shows custom orders with their
reference images and offer notes.
**Accept**: existing bouquet orders and reports are unchanged (migration keeps current rows valid).

### REQ-BRQ-14 — Reveal details to the chosen vendor — P0
After `Paid`, the chosen vendor sees the exact address, pin, recipient name and phone (if given),
and the full request. Other vendors lose access to the request images.
**Accept**: image endpoints check the caller's relation to the request on every read.

---

## E. Fulfilment, countdown and lateness

### REQ-BRQ-15 — Phases and "need more time" — P0
Custom orders move through `Paid → InProgress → Ready → OutForDelivery → Delivered` (pickup:
`Ready → PickedUp`). The vendor app has one-tap phase buttons. **Need more time** proposes a new
ready/delivery time with a reason (chips + ≤ 300 chars). The customer accepts, or cancels with a
full refund if the new time is > 60 min after the committed time; no answer within 30 min =
accepted.
**Accept**: each phase change and time change is an audit entry; times never move silently.

### REQ-BRQ-16 — Countdown — P0
Customer and vendor screens show a live countdown to the committed delivery (or pickup) time and
the current phase. When late, they show "Late by 12 min".
**Accept**: server time is the source (`serverNow` in the DTO); the client corrects drift.

### REQ-BRQ-17 — SLA monitor and alerts — P0
A background service (like the vase heartbeat monitor, EP-15) checks custom orders every minute:
- not `InProgress` by `ready − preparationLead` (vendor setting, default 60 min) → vendor alert
  "Start soon";
- not `Ready` at ready time → vendor alert; customer informed "Your florist is finishing up";
- not `Delivered` at delivery time → both alerted "Running late"; repeated at +30 min;
- late > 2 h → Admin Portal escalation and customer offered cancel + refund.

**Accept**: alerts are idempotent per (order, rule); stops on Delivered/Cancelled.

---

## F. Notifications

### REQ-BRQ-18 — Notification events and channels — P0
Events: new request in range (vendor), new/updated/withdrawn offer (customer), offer accepted /
rejected (vendor), acceptance lapsed (both), request about to expire / expired (customer), phase
changes (customer), more time requested / answered (both), late / at-risk alerts (both),
delivered (customer).
Channels: in-app inbox (customer: existing `Notification` with new `NotificationType` values;
vendor: new `VendorNotification` inbox), SignalR (new `MarketplaceHub` with per-user groups), push
when FCM exists (EP-06, **dependency**), and transactional email for acceptance, late and
cancellation.
**Accept**: every event writes the inbox row via the Outbox pattern (`OutboxDispatcherService`),
so delivery survives restarts; the user's per-event preferences are honoured.

---

## G. Safety, admin and parity

### REQ-BRQ-19 — Abuse protection and moderation — P1
Report a request or offer (reason chips). Admin Portal `Requests` page: list/filter, view
images, cancel a request or suspend the author. Automatic hiding after 3 reports pending review.
Text fields are length-capped and HTML-escaped; images are re-encoded.
**Accept**: admin actions are audit-logged and notify the affected users.

### REQ-BRQ-20 — Vendor Portal parity — P2
Vendor Portal `Pages/Requests/*`: map + list, offer form, accepted orders in the existing order
management.

### REQ-BRQ-21 — Request-scoped messages — P2
A per-offer thread between customer and vendor before acceptance (text only, ≤ 500 chars,
rate-limited), closed on acceptance and moved into the order afterwards.

### REQ-BRQ-22 — API contract and clients — P0
New §23 "Bouquet requests" in `API-CONTRACTS.md` (customer and vendor routes, DTOs, states,
SignalR events, errors). Clients: customer web app (`FlowerShop.CustomerApp`) pages **and the
customer mobile app** (follow-up epic in its repo), plus vendor app EP-M17 (map layer, offer
composer, phase buttons, countdown).
**Accept**: `contracts-check` run.

---

## Cross-cutting

- **Migrations**: `BouquetRequests`, `BouquetRequestImages`, `BouquetOffers` (+ revisions,
  images), `SavedAddress.Latitude/Longitude/GeocodedAt`, vendor `ServiceRadiusKm` +
  `PreparationLeadMinutes`, `OrderLine` custom kind, `VendorNotifications`, `OrderTimeChanges`.
- **Tenant isolation**: an offer belongs to a vendor (VendorId + VendorType). The request belongs to
  a customer. Every query is scoped by the caller's role and relation.
- **Data retention**: request images are deleted 90 days after the request closes (orders keep
  the snapshot thumbnail); GDPR erase covers requests, offers and messages.
- **Time zones**: all times stored UTC, displayed Europe/Warsaw.

## Decisions
- **Decided (2026-10-20)**: payment at acceptance (REQ-BRQ-12). Customer clients are the web
  Customer App and the customer mobile app.
- **Decided (2026-10-20)**: platform commission on custom orders is the same as for bouquet orders
  (MVP, not live yet; revisit before launch).
- **Decided (2026-10-20)**: vendors without an active platform subscription cannot see requests
  at all (REQ-BRQ-05).
- The customer mobile app repository doesn't exist yet. Its follow-up (T-37-032) is synced in an
  upcoming iteration.

## Explicitly out of scope
- Auctions/bidding wars (vendors see only the offer count, not competing prices).
- Customer-to-customer visibility of requests.
- Courier integrations (vendors deliver themselves or set pickup).
- Supplier procurement for requested flowers (EP-35's procurement port is the future hook).
