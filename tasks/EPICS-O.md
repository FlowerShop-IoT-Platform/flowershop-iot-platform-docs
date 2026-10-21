# Epics, Stories & Tasks — Option O (Customer bouquet requests)

> Companion to EPICS.md (EP-04 orders + Stripe, EP-05 vendor orders, EP-06 notifications),
> EPICS-B.md (EP-13/14 customer web app) and EPICS-C.md (EP-20 vendor platform subscription).
> Epic is numbered EP-37 to continue the sequence.
> Task tracker (ground truth): `docs/tasks/tasks-ep37-customer-bouquet-requests.json`.
> Requirements: `docs/tasks/REQ-GAPS-O.md` (REQ-BRQ-01 … REQ-BRQ-22).
> Mobile counterparts: vendor app EP-M17 (T-37-024); customer mobile app follow-up (T-37-032).
> Analysis date: 2026-10-20

---

## EP-37: Customer bouquet requests (map pins, vendor offers, accepted order with countdown)

**Priority**: P1-high
**Dependencies**: EP-04/EP-17/EP-18 (Stripe orders, refunds), EP-05/EP-12 (vendor order management),
EP-13 (customer web app, Leaflet), EP-20 (active vendor subscription), EP-06 (FCM, for push only)
**Requirement refs**: REQ-BRQ-01 through REQ-BRQ-22
**State**: in progress — implemented and tested (`BouquetRequest`/`BouquetOffer`/`VendorMarketplaceSettings`/`VendorNotification`, custom order line, migration `AddBouquetRequests`, §23 endpoints, `/hubs/marketplace`, background monitor, Customer App `Requests`, Vendor Portal `Pages/Requests/*`, Admin Portal `Pages/Requests`). Open: real email/push delivery (T-37-022, stub email + EP-06), optional messages (T-37-027, deferred), vendor app EP-M17 (T-37-024), customer mobile app (T-37-032).

> Customers can now ask for the bouquet they want instead of only buying what is listed. Vendors
> nearby compete with offers. The customer picks one and both sides follow a countdown to delivery.

### Flow

```
 Customer (web app / mobile)                     Vendors in radius (vendor app / portal)
 ───────────────────────────                     ──────────────────────────────────────
 photos + note + budget + time + address pin ──▶ map pin at fuzzed ~500 m cell, no PII
                                                  "New request 3.2 km · Sat 14:00 · 150–200 PLN"
                      ◀── offer: price · ready/delivery time · substitution notes · photos
 compare offers ─▶ ACCEPT one  ══ one transaction ══ others Rejected · Order(Created, Custom line)
 pay (Stripe card/BLIK) ─▶ Paid ──▶ exact address revealed to the chosen vendor only
 countdown ◀──────────── Paid → InProgress → Ready → OutForDelivery → Delivered
           ◀──────────── "Need more time" (accept / cancel+refund if > 60 min)
 SLA monitor: start-soon · not ready · running late · +30 min · > 2 h → admin escalation
```

### Design decisions
- **Privacy by construction.** Requests store an exact and a deterministic fuzzed location. Vendor
  DTOs only ever contain the fuzzed one until the order is `Paid`, and tests assert it. Customers
  never see other customers' requests.
- **Vendors must be subscribed.** Only vendors with an active EP-20 platform subscription see
  requests or make offers, inside their own service radius (new vendor setting, default 15 km).
- **One winner.** Accept is a single transaction with optimistic concurrency, and the accept call
  carries `offerRevision`. Parallel accepts produce one order.
- **Pay to commit.** Acceptance creates the order. The vendor's commitment starts at `Paid`. An
  unpaid acceptance lapses after 30 min and the other offers come back.
- **Custom order line.** `OrderLine` requires `BouquetId`/`VaseId` today. A `Custom` kind holds
  the request/offer references and a snapshot, so existing orders and reports stay unchanged.
- **Notifications don't wait for FCM.** Push is a no-op stub (EP-06 pending), so inbox
  (`Notification` for customers, new `VendorNotification` for vendors), SignalR (`MarketplaceHub`)
  and email ship first. Push plugs in later. All events go through the Outbox.
- **AI help for pricing a picture.** "Reconstruct with AI Florist" (EP-35 mode 2) on a request
  maps the customer's images to the vendor's stock. It shows what is missing and an estimated cost,
  and drafts the substitution note for the offer.
- **Address pin needs coordinates.** `SavedAddress` gains lat/lng through the existing
  `IGeocodingService` / `IAddressLookupService` (`Infrastructure/Geocoding`), always with a
  manual pin adjustment.

### Stories & tasks
See `tasks-ep37-customer-bouquet-requests.json` (S-37-01 … S-37-08; T-37-001 … T-37-032).
1. **S-37-01 — Trackers** — T-37-001.
2. **S-37-02 — Requests and addresses** — T-37-002 aggregate · T-37-003 images · T-37-004 geocoding + pin · T-37-005 endpoints + limits · T-37-006 expiry.
3. **S-37-03 — Vendor discovery** — T-37-007 radius + lead · T-37-008 fuzzed location + query · T-37-009 fan-out.
4. **S-37-04 — Offers** — T-37-010 aggregate · T-37-011 vendor endpoints · T-37-012 comparison.
5. **S-37-05 — Acceptance, payment and custom order** — T-37-013 atomic accept · T-37-014 custom line · T-37-015 payment + lapse · T-37-016 reveal.
6. **S-37-06 — Fulfilment, countdown and SLA** — T-37-017 phases + more time · T-37-018 countdown · T-37-019 SLA monitor.
7. **S-37-07 — Notifications** — T-37-020 catalogue + vendor inbox · T-37-021 MarketplaceHub · T-37-022 email + push.
8. **S-37-08 — Clients, moderation, contract and tests** — T-37-023 customer web app · T-37-024 EP-M17 · T-37-025 portal parity · T-37-026 moderation · T-37-027 messages · T-37-028 contract · T-37-029 tests · T-37-030 manuals · T-37-031 "Reconstruct with AI Florist" entry · T-37-032 customer mobile app.

**Suggested order**: S-37-02 → S-37-03 → S-37-04 → S-37-05 → S-37-06 → S-37-07 → S-37-08
(T-37-028 contract drafted early, after S-37-04, so the vendor app can start).

### Decisions
All decided (2026-10-20): payment at acceptance; commission the same as bouquet orders (MVP);
unsubscribed vendors can't see requests; web Customer App now, customer mobile app follow-up once
its repository exists.
