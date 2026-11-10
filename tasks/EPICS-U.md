# Epics, Stories & Tasks — Option U (Florist operating system: covering every business flow)

> Companion to EPICS-J.md (stock), EPICS-O.md (customer requests) and EPICS-Q.md (online-only vendors).
> Epics are numbered EP-43 … EP-58 to continue the sequence.
> Task trackers (ground truth): `docs/tasks/tasks-ep43-*.json` … `docs/tasks/tasks-ep58-*.json`.
> Requirements: `docs/tasks/REQ-GAPS-U.md` (REQ-FOS-01 … REQ-FOS-103).
> Mobile counterparts (vendor app, customer app) are raised as follow-up tasks inside each epic.
> Analysis date: 2026-11-09

**Goal:** a Polish florist or flower shop runs a normal day, and a peak day, entirely in FMF — not only sells through it.

| Epic | Priority | Stories | Tasks | Tracker |
|---|---|---|---|---|
| EP-43 Polish-first experience, with English and Ukrainian | p1-high | 7 | 16 | `tasks-ep43-polish-localisation.json` |
| EP-44 Florist payouts and Polish payment methods | p1-high | 6 | 9 | `tasks-ep44-payments-and-payouts.json` |
| EP-45 Push alerts for florists and customers | p1-high | 5 | 7 | `tasks-ep45-push-alerts.json` |
| EP-46 One order book: every channel, one production board | p1-high | 5 | 8 | `tasks-ep46-unified-order-book.json` |
| EP-47 Made-to-order catalogue and gift order details | p1-high | 4 | 9 | `tasks-ep47-catalogue-and-gift-details.json` |
| EP-48 Delivery slots, zones, capacity, pre-orders and closures | p1-high | 4 | 6 | `tasks-ep48-delivery-slots-zones-preorders.json` |
| EP-49 Fulfilment proof and recipe-linked production | p1-high | 5 | 5 | `tasks-ep49-fulfilment-proof-recipes.json` |
| EP-50 Invoices, KSeF and accounting export | p1-high | 5 | 7 | `tasks-ep50-invoices-ksef-accounting.json` |
| EP-51 Courier dispatch and pickup partners | p2-medium | 4 | 4 | `tasks-ep51-courier-dispatch-pickup.json` |
| EP-52 Purchasing, suppliers and demand forecast | p2-medium | 4 | 7 | `tasks-ep52-purchasing-and-supply.json` |
| EP-53 Customer–florist conversations | p1-high | 5 | 5 | `tasks-ep53-customer-conversations.json` |
| EP-54 Customer book, reminders and repeat sales | p2-medium | 5 | 7 | `tasks-ep54-customer-book-repeat-sales.json` |
| EP-55 Events, weddings, funerals and business accounts | p2-medium | 4 | 5 | `tasks-ep55-events-funerals-business.json` |
| EP-56 Locations, decision analytics and audit | p2-medium | 3 | 4 | `tasks-ep56-shop-operations.json` |
| EP-57 Public florist profiles, SEO and social channels | p2-medium | 4 | 7 | `tasks-ep57-public-profiles-seo-social.json` |
| EP-58 Team, shifts and bouquet authorship | p1-high | 7 | 16 | `tasks-ep58-team-shifts-authorship.json` |

---

## EP-43: Polish-first experience, with English and Ukrainian

**Priority**: P1-high
**Dependencies**: none (EP-45 push texts, EP-53 conversations and EP-57 SEO pages build on it)
**Requirement refs**: REQ-FOS-01 through REQ-FOS-10
**State**: in progress — implemented 2026-11-10 (S-43-01 … S-43-06 done; S-43-07 done except T-43-016, the vase display language, which waits on firmware EP-08). §H decided: Admin Portal Polish + English now; machine-drafted translations with native-speaker sign-off before release; Ukrainian ships with Polish. Translations are drafts until signed off.

> Every portal page, email and AI Florist text in Polish by default, with English and Ukrainian as full second languages.

### Concept

```
culture cookie / user preference / Accept-Language
        │
 RequestLocalization (pl-PL default · en-GB · uk-UA)  ── portals (resx SharedResources) ── <html lang>
        │                                               ├─ plural rules (pl and uk have several forms)
        │                                               └─ formatting: 25,00 zł · 14.02.2027 · 24 h (PLN in every language)
 User.PreferredLanguage (pl | en | uk) ─▶ API emails / push / SMS / AI Florist output language
 Vendor.LanguagesSpoken ─▶ public profile badge + customer filter "speaks Ukrainian"
```

### Design decisions
- Polish is the default for everyone; English and Ukrainian are selectable through a switcher and the stored preference, and are offered automatically when the browser asks for them.
- Ukrainian is a full third language (uk-UA, Cyrillic), not a partial one: the same pages, emails and alerts as Polish and English, enforced by the same key-parity guard.
- Resource files (`.resx`) per portal plus one shared set, not a database of strings: translations ship with the build and are reviewed in PRs. A machine-translated draft is allowed, but each language is signed off by a native speaker before release.
- Prices stay in PLN in every language; only formatting changes.
- AI Florist recipe names, notes and offer drafts follow the vendor's language; prompts stay in English internally.

### What was built
- **Infrastructure** (`src/portals/FlowerShop.Portals.Localization`): request localisation for the three portals (cookie → stored preference → `Accept-Language` → Polish), `GET /language/{code}` switcher partial, `<html lang>`, PLN/date formatting, CLDR plural rules (`L.Plural`), localised validation messages, `Accept-Language` forwarded to the API. Admin Portal: Polish and English.
- **Preference**: `User.PreferredLanguage`, vendor `LanguagesSpoken` (migration `AddLanguagePreferences`), `GET/PUT /api/v1/me/language`, `GET/PUT /api/v1/vendor/settings/languages`, Keycloak `locale` mirror and `ui_locales` (API contract §27).
- **Translations**: Vendor Portal (~1,380 keys) and Customer App (~560) in pl/en/uk, Admin Portal (~750) in pl/en — `Resources/SharedResource*.resx`, English text as key.
- **Messages and AI**: marketplace inbox/real-time/email texts in the recipient's language (`NoteText`, `MessageCatalog`); AI Florist recipes in the vendor's language; Ukrainian atlas names.
- **Ukrainian specifics**: Twilio Verify / Stripe / Turnstile / Keycloak locales, Cyrillic search, "Florist speaks" filter and "Speaks Ukrainian" badge.
- **Guard and docs**: `TranslationGuardTests` (parity, plurals, placeholders, case-only duplicates, used keys, hard-coded markup scan); Polish and Ukrainian manuals under `docs/manuals/pl|uk`, Help opens the user's language.
- **Follow-ups raised**: vendor app EP-M23, customer app EP-15, smart vase firmware EP-08 (display language).

### Stories & tasks
See `tasks-ep43-polish-localisation.json` (S-43-01 … S-43-07; T-43-001 … T-43-016).
1. **S-43-01 — Trackers** — T-43-001 (done).
2. **S-43-02 — Localisation infrastructure** (done) — T-43-002 Request localisation in the three portals · T-43-003 PreferredLanguage on User.
3. **S-43-03 — Vendor Portal translation** (done) — T-43-004 Extract Vendor Portal strings to resources · T-43-005 Ukrainian translation of the Vendor Portal.
4. **S-43-04 — Customer App translation** (done) — T-43-006 Extract Customer App strings to resources · T-43-007 Ukrainian translation of the Customer App · T-43-008 Admin Portal strings to resources.
5. **S-43-05 — Emails, notifications and AI output** (done) — T-43-009 Email and inbox templates per language · T-43-010 AI Florist output language.
6. **S-43-06 — Ukrainian specifics** (done) — T-43-011 Cyrillic coverage and third-party locales · T-43-012 Vendor languages spoken.
7. **S-43-07 — Guard, manuals and mobile** (in progress) — T-43-013 Missing-translation guard test · T-43-014 Polish and Ukrainian manuals · T-43-015 Mobile follow-up (done) · T-43-016 Smart vase display language (todo, waits on firmware EP-08).

---

## EP-44: Florist payouts and Polish payment methods

**Priority**: P1-high
**Dependencies**: EP-17/18 payment resilience and card checkout; EP-20 vendor billing; EP-37 custom-request payments
**Requirement refs**: REQ-FOS-11 through REQ-FOS-16
**State**: proposed. Trackers only (S-44-01). REQ-GAPS-U §B open decisions.

> Money reaches the florist automatically, with a payout statement, and customers can pay the way Poles pay.

### Concept

```
Customer pays (BLIK / card / P24 / wallet)
        │  PaymentIntent  transfer_data.destination = vendor's connected account
        │                 application_fee_amount  = platform fee rule (0 % by default)
        ▼
 Stripe ─▶ connected account balance ─▶ payout to florist's bank (Stripe schedule)
 refund / dispute ─▶ reverse_transfer + refund_application_fee
 Vendor Portal Payments: gross · fees · refunds · net · payouts · CSV
```

### Design decisions
- Destination charges on Stripe Connect Express are proposed (§H.1): the existing Stripe integration, webhooks and refund retry stay; the florist's KYC and payouts are handled by Stripe.
- The platform fee is a configurable rule that defaults to 0 %: revenue stays the SaaS subscription until a commission is decided (§H.2).
- A vendor without a verified connected account can still use the portal but cannot take paid online orders (§H.3).

### Stories & tasks
See `tasks-ep44-payments-and-payouts.json` (S-44-01 … S-44-06; T-44-001 … T-44-009).
1. **S-44-01 — Trackers** — T-44-001 (done).
2. **S-44-02 — Connected accounts** — T-44-002 Stripe Connect Express onboarding.
3. **S-44-03 — Destination charges and refunds** — T-44-003 Destination charges with platform fee rule · T-44-004 Refund and dispute handling.
4. **S-44-04 — Payout statement and admin view** — T-44-005 Vendor payout statement + CSV · T-44-006 Admin connected-account view and pre-go-live settlement list.
5. **S-44-05 — Payment methods** — T-44-007 Przelewy24 and wallets · T-44-008 Pay-later option.
6. **S-44-06 — Rollout and contract** — T-44-009 Onboarding campaign and contract.

---

## EP-45: Push alerts for florists and customers

**Priority**: P1-high
**Dependencies**: EP-06 (DeviceRegistration, Notification aggregate); EP-37 (VendorNotification, SLA monitor); EP-43 (texts in Polish)
**Requirement refs**: REQ-FOS-17 through REQ-FOS-23
**State**: proposed. Trackers only (S-45-01). REQ-GAPS-U §C open decisions.

> Real phone notifications for the events that make or lose a sale.

### Concept

```
domain event ─▶ outbox ─▶ AlertDispatcher ─▶ preferences + quiet hours
                                            ├─▶ Notification row (in-app inbox)
                                            ├─▶ FCM HTTP v1 (Android / iOS via APNs) ─▶ device tokens
                                            └─▶ email fallback (opt-in)
```

### Design decisions
- Completes EP-06 T-06-007/T-06-008 instead of duplicating them; those tasks are closed by this epic.
- One dispatcher decides channels from an alert catalogue and the user's preferences; handlers only raise events.
- Vase alerts are sent to in-store vendors only (EP-42 rule).

### Stories & tasks
See `tasks-ep45-push-alerts.json` (S-45-01 … S-45-05; T-45-001 … T-45-007).
1. **S-45-01 — Trackers** — T-45-001 (done).
2. **S-45-02 — FCM delivery** — T-45-002 FirebaseNotificationService.
3. **S-45-03 — Alert catalogue and dispatcher** — T-45-003 AlertDispatcher with vendor alerts · T-45-004 Customer alerts.
4. **S-45-04 — Preferences** — T-45-005 Notification preferences and quiet hours.
5. **S-45-05 — Web push and mobile** — T-45-006 Vendor Portal Web Push · T-45-007 Mobile follow-up.

---

## EP-46: One order book: every channel, one production board

**Priority**: P1-high
**Dependencies**: EP-05/12 vendor orders; EP-48 slots/capacity (calendar load); EP-58 staff and shifts (assignment)
**Requirement refs**: REQ-FOS-24 through REQ-FOS-29
**State**: proposed. Trackers only (S-46-01). REQ-GAPS-U §D open decisions.

> Phone, walk-in, Instagram and FMF orders in one list and one production board, so FMF replaces the notebook.

### Concept

```
FMF checkout ─┐
 phone / walk-in / Instagram / WhatsApp ─┼─▶ Order (Source, Contact, DueAt, Payments[])
 custom request (EP-37) ─┘                         │
                                Production board: To make → Made → Ready → Out → Done
                                Calendar: daily load vs capacity (EP-48) · job card print
```

### Design decisions
- Manual orders are real `Order`s with `Source != Fmf`, so analytics, invoices (EP-50) and the board treat all channels the same.
- A manual order does not need an FMF customer account: it carries an order contact (name, phone, optional email). A contact that matches a customer later can be linked (§H.1).
- No platform fee and no Stripe on manual orders; payments are recorded, not processed.

### Stories & tasks
See `tasks-ep46-unified-order-book.json` (S-46-01 … S-46-05; T-46-001 … T-46-008).
1. **S-46-01 — Trackers** — T-46-001 (done).
2. **S-46-02 — Order model for all channels** — T-46-002 Order source, contact and DueAt.
3. **S-46-03 — Manual orders and payments** — T-46-003 Manual order commands and API · T-46-004 Recorded payments and balances · T-46-005 Vendor Portal New order page.
4. **S-46-04 — Production board and calendar** — T-46-006 Production board page · T-46-007 Calendar view.
5. **S-46-05 — Contract and mobile** — T-46-008 Contract, changelog, vendor app follow-up.

---

## EP-47: Made-to-order catalogue and gift order details

**Priority**: P1-high
**Dependencies**: EP-39 (`IsPurchasable`, nullable vase on lines); EP-48 (earliest delivery slot); EP-49 (photo before dispatch keeps the real-photo promise)
**Requirement refs**: REQ-FOS-30 through REQ-FOS-36
**State**: proposed. Trackers only (S-47-01). REQ-GAPS-U §E open decisions.

> Repeatable bouquet types in S/M/L next to one-off real bouquets, plus the gift details every flower order needs.

### Concept

```
CatalogueItem (vendor) ── sizes S/M/L (price, lead time, optional house recipe)
        │                       example photos (real, own work)
 Cart line: Bouquet (one-off) | CatalogueItem+size | AddOn
        │
 Order gift details: recipient · card message · anonymous sender · delivery date + window
```

### Design decisions
- Catalogue photos must be the florist's own work and are labelled "example — made fresh for you"; the finished-bouquet photo (EP-49) keeps FMF's real-photo promise.
- One-off real bouquets stay the hero of the map; catalogue items appear on the florist page, in search and as a "made to order" map filter.
- Gift details apply to every order type, including custom requests and manual orders.

### Stories & tasks
See `tasks-ep47-catalogue-and-gift-details.json` (S-47-01 … S-47-04; T-47-001 … T-47-009).
1. **S-47-01 — Trackers** — T-47-001 (done).
2. **S-47-02 — Catalogue items** — T-47-002 CatalogueItem domain, persistence and API · T-47-003 Vendor Portal catalogue pages · T-47-004 Save a recipe as catalogue item.
3. **S-47-03 — Buying from the catalogue** — T-47-005 Customer catalogue browsing · T-47-006 Catalogue lines in cart and orders.
4. **S-47-04 — Gift details and add-ons** — T-47-007 Gift details on orders · T-47-008 Add-ons · T-47-009 Send to a phone number.

---

## EP-48: Delivery slots, zones, capacity, pre-orders and closures

**Priority**: P1-high
**Dependencies**: EP-46 (board/calendar); EP-47 (lead times, delivery date on orders); opening hours (`Pages/Settings/OpeningHours`)
**Requirement refs**: REQ-FOS-37 through REQ-FOS-42
**State**: proposed. Trackers only (S-48-01). REQ-GAPS-U §F open decisions.

> Customers pick a real delivery window; florists control how much they take on each day, including peak days.

### Concept

```
Vendor: zones (rings or postcodes → fee, min order) · weekly slot template · capacity per slot
        closures & special days · pre-order campaigns (date, cap, cut-off)
        │
 Checkout: address → zone → fee  ·  slot picker (lead time, cut-off, capacity left)
        │
 Order holds a slot reservation ─▶ released on lapse / cancel ─▶ board + calendar load
```

### Design decisions
- The global `Delivery:*` settings remain the default for vendors who set nothing, so nothing breaks on release.
- Capacity is counted in orders per slot (v1), not in florist minutes.
- A slot is reserved when the order is created and released with the existing reservation lapse.

### Stories & tasks
See `tasks-ep48-delivery-slots-zones-preorders.json` (S-48-01 … S-48-04; T-48-001 … T-48-006).
1. **S-48-01 — Trackers** — T-48-001 (done).
2. **S-48-02 — Zones and slots model** — T-48-002 Delivery zones and slot templates.
3. **S-48-03 — Checkout slot picker** — T-48-003 Slot availability query and reservation · T-48-004 Customer slot picker UI.
4. **S-48-04 — Closures and pre-orders** — T-48-005 Closures and special days · T-48-006 Pre-order campaigns.

---

## EP-49: Fulfilment proof and recipe-linked production

**Priority**: P1-high
**Dependencies**: EP-46 (board statuses); EP-47 (catalogue sizes with recipes); EP-45 (alerts); EP-30 stock consume; EP-28/33 recipes
**Requirement refs**: REQ-FOS-43 through REQ-FOS-47
**State**: proposed. Trackers only (S-49-01). REQ-GAPS-U §G open decisions.

> The customer sees the finished bouquet before it leaves and a photo at the door; stock follows the recipe automatically.

### Concept

```
Board: To make ─▶ Made  (upload finished photo) ─▶ customer approves / asks a change (N min, silence = OK)
                         └─ recipe on line ─▶ stock consume (all-or-nothing, idempotent)
 Out for delivery ─▶ delivered: photo + recipient name (+ existing QR) ─▶ customer alert
```

### Design decisions
- Approval is on by default for custom requests and catalogue items, off for one-off real bouquets (already photographed).
- Silence within the approval window counts as approval, so photos never block a delivery slot.
- Stock deduction happens at Made, with a manual override and an undo movement.

### Stories & tasks
See `tasks-ep49-fulfilment-proof-recipes.json` (S-49-01 … S-49-05; T-49-001 … T-49-005).
1. **S-49-01 — Trackers** — T-49-001 (done).
2. **S-49-02 — Finished photo and approval** — T-49-002 Order photos and approval window.
3. **S-49-03 — Proof of delivery** — T-49-003 POD photo and recipient name.
4. **S-49-04 — Recipe-linked stock** — T-49-004 Recipe on order lines + consume at Made.
5. **S-49-05 — Evidence** — T-49-005 Dispute evidence and portfolio consent.

---

## EP-50: Invoices, KSeF and accounting export

**Priority**: P1-high
**Dependencies**: EP-44 (fees and payouts in the export); EP-46 (manual orders); EP-20/21 (platform charges to vendors)
**Requirement refs**: REQ-FOS-48 through REQ-FOS-53
**State**: proposed. Trackers only (S-50-01). REQ-GAPS-U §H open decisions.

> Invoices go out automatically through the florist's own invoicing app (which talks to KSeF), and the accountant gets a monthly export.

### Concept

```
Order paid / Done ─▶ IInvoicingProvider (vendor's account: Fakturownia | ifirma | inFakt)
                         └─ provider sends to KSeF ─▶ KSeF number + PDF link stored on the order
 Platform → vendor charges (plan, vases, fees) ─▶ platform's own provider account ─▶ KSeF
 Monthly export per vendor: sales, VAT, payment methods, fees, payouts (CSV)
```

### Design decisions
- FMF never holds the vendor's KSeF credentials: invoices are issued through the vendor's invoicing-app account using its API token (stored encrypted).
- Fiscal receipts from a cash register (kasa fiskalna) for walk-in sales stay out of scope (§H.2).
- The platform's own invoices to vendors move to a KSeF-capable provider; Stripe stays the payment collector.

### Stories & tasks
See `tasks-ep50-invoices-ksef-accounting.json` (S-50-01 … S-50-05; T-50-001 … T-50-007).
1. **S-50-01 — Trackers** — T-50-001 (done).
2. **S-50-02 — Provider port and connection** — T-50-002 IInvoicingProvider + Fakturownia adapter · T-50-003 Second provider adapter.
3. **S-50-03 — Invoice data and issuing** — T-50-004 NIP and invoice details at checkout · T-50-005 Automatic issuing with retries.
4. **S-50-04 — Platform invoices** — T-50-006 KSeF invoices for vendor charges.
5. **S-50-05 — Accountant export** — T-50-007 Monthly export CSV + summary.

---

## EP-51: Courier dispatch and pickup partners

**Priority**: P2-medium
**Dependencies**: EP-48 (slots), EP-49 (POD), EP-58 (staff roles)
**Requirement refs**: REQ-FOS-54 through REQ-FOS-58
**State**: proposed. Trackers only (S-51-01). REQ-GAPS-U §I open decisions.

> Deliveries assigned to the florist's own courier or booked with a courier partner, with tracking, plus after-hours pickup points.

### Concept

```
Order (slot) ─▶ assign: own courier (employee) | partner (ICourierProvider: quote · book · track · cancel)
                         └─ route sheet per slot (ordered stops, map link, phone)
 Pickup point: shop | kwiatomat / locker partner ─▶ pickup code by SMS
```

### Design decisions
- One partner adapter first (§H.1), behind a port, so others can be added.
- Partner fees are shown to the vendor before booking and can be passed to the customer through zone fees (EP-48).

### Stories & tasks
See `tasks-ep51-courier-dispatch-pickup.json` (S-51-01 … S-51-04; T-51-001 … T-51-004).
1. **S-51-01 — Trackers** — T-51-001 (done).
2. **S-51-02 — Own couriers** — T-51-002 Courier assignment and route sheet.
3. **S-51-03 — Courier partner** — T-51-003 ICourierProvider and first adapter.
4. **S-51-04 — Pickup points** — T-51-004 Kwiatomat/locker pickup pilot.

---

## EP-52: Purchasing, suppliers and demand forecast

**Priority**: P2-medium
**Dependencies**: EP-30/31 stock; EP-35 missing list and supplier port; EP-49 (recipes on orders); EP-48 (pre-orders)
**Requirement refs**: REQ-FOS-59 through REQ-FOS-64
**State**: proposed. Trackers only (S-52-01). REQ-GAPS-U §J open decisions.

> From "what is low" to "what to buy on Monday", with real purchase costs feeding AI Florist prices.

### Concept

```
low stock + recipes of upcoming orders + pre-orders + events ─▶ Reorder list (editable)
        └─▶ export / send to supplier (PDF, CSV, text)
 Supplier delivery ─▶ Purchase receipt ─▶ Received movements + cost history ─▶ AI Florist unit costs
 IFlowerSupplierCatalog adapter (price feed) · forecast (history + Polish calendar)
```

### Design decisions
- Reorder suggestions are always editable suggestions, never automatic purchases.
- Forecasting starts with a transparent rule (last years' same weeks + holiday uplift), not a black-box model.

### Stories & tasks
See `tasks-ep52-purchasing-and-supply.json` (S-52-01 … S-52-04; T-52-001 … T-52-007).
1. **S-52-01 — Trackers** — T-52-001 (done).
2. **S-52-02 — Reorder list and suppliers** — T-52-002 Reorder calculation and page · T-52-003 Supplier book.
3. **S-52-03 — Receipts and costs** — T-52-004 Purchase receipt · T-52-005 Waste report.
4. **S-52-04 — Price feed and forecast** — T-52-006 Supplier price feed adapter · T-52-007 Demand forecast.

---

## EP-53: Customer–florist conversations

**Priority**: P1-high
**Dependencies**: EP-37 (requests, offers, `MarketplaceHub`, PII rule); EP-45 (push); EP-43 (texts)
**Requirement refs**: REQ-FOS-65 through REQ-FOS-70
**State**: proposed. Trackers only (S-53-01). REQ-GAPS-U §K open decisions.

> A chat with photos on every custom request and order, without leaking contact details before payment.

### Concept

```
Conversation (request × vendor | order) ── messages: text + photos (private media)
        │  before payment: phone / e-mail / links masked server-side
        ├─▶ MarketplaceHub (SignalR) live
        ├─▶ push (EP-45) · email digest if unread 30 min
        └─▶ report → admin moderation · retention 12 months
```

### Design decisions
- One conversation per (request, vendor) and per order — never a free-form inbox between strangers.
- Contact details are masked until the customer pays, matching EP-37's "no PII until paid" rule.
- The one-shot `VendorMessage` stays for customers without an order, with its admin escalation.

### Stories & tasks
See `tasks-ep53-customer-conversations.json` (S-53-01 … S-53-05; T-53-001 … T-53-005).
1. **S-53-01 — Trackers** — T-53-001 (done).
2. **S-53-02 — Conversations** — T-53-002 Conversation aggregate, API and masking.
3. **S-53-03 — Live delivery** — T-53-003 SignalR, push and digest.
4. **S-53-04 — UIs** — T-53-004 Vendor Portal and Customer App thread UIs.
5. **S-53-05 — Safety and retention** — T-53-005 Moderation, rate limits, retention.

---

## EP-54: Customer book, reminders and repeat sales

**Priority**: P2-medium
**Dependencies**: EP-46 (manual-order contacts); EP-45 (push); EP-47 (catalogue items for subscriptions); EP-44 (off-session payments)
**Requirement refs**: REQ-FOS-71 through REQ-FOS-76
**State**: proposed. Trackers only (S-54-01). REQ-GAPS-U §L open decisions.

> Florists know their customers, customers are reminded of the dates that matter, and flowers can be bought on repeat.

### Concept

```
Customer book (per vendor): FMF customers + manual contacts · history · notes · dates · consent
 Reminders: birthdays, anniversaries, imieniny (name → dates) ─▶ alert N days before + suggestions
 Recurring: catalogue item × cadence ─▶ Stripe off-session ─▶ orders on the board
 Vouchers & codes · stamp-card loyalty · "last chance" = FreshnessPricingStrategy applied
```

### Design decisions
- Each florist sees only customers who ordered from them; marketing to them needs recorded consent (RODO).
- Discounts and loyalty are funded by the vendor; the platform fee (if any) is on the discounted price.
- "Last chance" uses the vendor's existing pricing strategy only when `AutoAdjustPrice` is on.

### Stories & tasks
See `tasks-ep54-customer-book-repeat-sales.json` (S-54-01 … S-54-05; T-54-001 … T-54-007).
1. **S-54-01 — Trackers** — T-54-001 (done).
2. **S-54-02 — Customer book** — T-54-002 Customer book pages and API.
3. **S-54-03 — Reminders** — T-54-003 Occasion dates, name-day dataset and reminder job.
4. **S-54-04 — Repeat purchases** — T-54-004 Recurring bouquet subscriptions · T-54-005 Discount codes and vouchers · T-54-006 Stamp-card loyalty.
5. **S-54-05 — Last chance** — T-54-007 Apply freshness pricing automatically.

---

## EP-55: Events, weddings, funerals and business accounts

**Priority**: P2-medium
**Dependencies**: EP-46 (orders/board), EP-44 (deposits), EP-50 (monthly invoices), EP-53 (conversation), EP-28/36 (recipes and cost assessor)
**Requirement refs**: REQ-FOS-77 through REQ-FOS-81
**State**: proposed. Trackers only (S-55-01). REQ-GAPS-U §M open decisions.

> Quotes with deposits for weddings and events, a proper funeral range, and business customers with standing orders.

### Concept

```
Enquiry ─▶ Quote (items × qty, AI recipes, mood board, PDF in Polish, versions)
        ─▶ customer accepts online ─▶ deposit (zadatek) ─▶ balance due date ─▶ event tasks on the board
 Funeral: wieniec / wiązanka · ribbon text · funeral home / cemetery delivery · urgent flag
 Business account: several orderers · standing orders · monthly invoice
```

### Design decisions
- Quotes reuse AI Florist recipes and the cost assessor for pricing, which competitors cannot.
- Standing orders share the recurring engine from EP-54.

### Stories & tasks
See `tasks-ep55-events-funerals-business.json` (S-55-01 … S-55-04; T-55-001 … T-55-005).
1. **S-55-01 — Trackers** — T-55-001 (done).
2. **S-55-02 — Event quotes** — T-55-002 Quote model, builder and PDF · T-55-003 Online acceptance and deposit.
3. **S-55-03 — Funerals** — T-55-004 Funeral category, ribbon text, venue delivery.
4. **S-55-04 — Business customers** — T-55-005 Business accounts and standing orders.

---

## EP-56: Locations, decision analytics and audit

**Priority**: P2-medium
**Dependencies**: EP-46 (sources), EP-49 (recipe costs), EP-37 (request response time), EP-58 (staff per location, per-florist stats)
**Requirement refs**: REQ-FOS-82 through REQ-FOS-84
**State**: proposed. Trackers only (S-56-01). REQ-GAPS-U §N open decisions.

> Owners can run more than one shop, see the numbers that drive decisions and trace money-related changes.

### Concept

```
Vendor ─▶ Locations (address, hours, stock, vases, staff roster) ─▶ orders routed to a location
 Analytics: sales by channel and location · margin per bouquet · vase utilisation · request response & win rate · waste %
 Audit log: refunds · payout settings · prices · role changes ─▶ owner view
```

### Design decisions
- Locations are a child entity of the vendor (one tenant), not separate vendors (§H.1).
- Per-florist figures live in EP-58; this epic adds the shop-level and location-level view.

### Stories & tasks
See `tasks-ep56-shop-operations.json` (S-56-01 … S-56-03; T-56-001 … T-56-004).
1. **S-56-01 — Trackers** — T-56-001 (done).
2. **S-56-02 — Locations** — T-56-002 Location entity and filters.
3. **S-56-03 — Analytics and audit** — T-56-003 Decision analytics · T-56-004 Audit log.

---

## EP-57: Public florist profiles, SEO and social channels

**Priority**: P2-medium
**Dependencies**: EP-43 (Polish pages); EP-47 (catalogue on the profile); EP-49 (photos for portfolio, with consent)
**Requirement refs**: REQ-FOS-85 through REQ-FOS-91
**State**: proposed. Trackers only (S-57-01). REQ-GAPS-U §O open decisions.

> Florists and live bouquets found on Google in Polish, with profiles, photo reviews and links from Instagram and Google Business Profile.

### Concept

```
/kwiaciarnia/{slug}  profile: photos · hours · live bouquets · catalogue · ratings & photo reviews
 /kwiaty/{miasto}[/{dzielnica}]   /okazje/{dzien-matki|walentynki|…}   (server-rendered, Polish slugs)
 schema.org: Florist/LocalBusiness · Product/Offer · AggregateRating   ·  sitemap.xml · canonical
 Link-in-bio page + QR  ·  Meta catalogue feed of live listings
```

### Design decisions
- Stable pages (florist, city, occasion) carry the ranking; one-off bouquet pages that expire redirect to the florist page, not 404.
- Reviews are only from paid orders (verified), and florists can reply but not delete.

### Stories & tasks
See `tasks-ep57-public-profiles-seo-social.json` (S-57-01 … S-57-04; T-57-001 … T-57-007).
1. **S-57-01 — Trackers** — T-57-001 (done).
2. **S-57-02 — Profiles and pages** — T-57-002 Public florist profile · T-57-003 City and occasion pages, sitemap, redirects · T-57-004 Structured data.
3. **S-57-03 — Reviews** — T-57-005 Photo reviews and florist replies.
4. **S-57-04 — Social channels** — T-57-006 Link-in-bio page and QR · T-57-007 Meta catalogue feed.

---

## EP-58: Team, shifts and bouquet authorship

**Priority**: P1-high
**Dependencies**: EP-19 vendor registration (owner account); EP-46 (production board, order assignment); EP-49 ("Made" step records the author); EP-43 (invitation e-mails per language)
**Requirement refs**: REQ-FOS-92 through REQ-FOS-103
**State**: proposed. Trackers only (S-58-01). REQ-GAPS-U §P decided 2026-11-09.

> A medium or large shop runs its team in FMF: staff sign in with their own accounts, two clear roles, a work calendar showing who is on shift, and every bouquet credited to the florist who made it.

### Concept

```
Owner/Manager ─▶ Staff page: invite · role · deactivate · activity
        └─ invitation ─▶ Keycloak account (vendor_id, vendor_type, role VendorStaff) ─▶ staff signs in to the Vendor Portal / vendor app
 Roles:  Owner/Manager = everything      Florist = board, orders, bouquets, photos, stock, AI Florist
         (API policies first; portal hides what the API would refuse)
 Work calendar: weekly roster · copy week · leave / days off · optional clock in/out
        └─ "On shift now" on dashboard and board · assign orders only to people on shift
 Authorship: bouquet / online listing / order line ─▶ MadeBy (who marked it Made, or picked) ─▶ board · job card · per-florist stats
 Per-florist view: bouquets made · ratings · hours ─▶ hours export for payroll
```

### Design decisions
- Two roles in v1, as asked by shops: **Owner/Manager** (full access, including staff, shifts, prices, payouts, billing and settings) and **Florist** (front line: production board, orders, making bouquets and listings, finished-bouquet photos, stock movements, AI Florist; no prices, payouts, billing, staff or settings). Courier and cashier variants can follow later (§H.1).
- The existing `Employee` aggregate and `EmployeesController` are reused and fixed rather than replaced; the unused `ShopAdmin`/`ShopEmployee` policies are replaced by the two roles.
- Staff get their own Keycloak accounts tied to the shop (`vendor_id`, `vendor_type` claims and a `VendorStaff` role), so tenant resolution keeps working and every action has a real author. The vendor owner's `VendorAdmin` account is an Owner/Manager automatically.
- Authorship is recorded automatically at the "Made" step (EP-49) and can be corrected by an Owner/Manager; it is never required to sell.
- Clock in/out is optional per shop: the roster alone answers "who is on shift".
- Every staff member, florists included, sees the whole team calendar (decided §H.2).
- Showing the florist's first name to customers is a per-shop option, off by default (decided §H.3).
- Staff accounts are seats in the vendor's SaaS plan: each plan includes a number of seats and extra seats are billed, because several florists working in the app at once add real infrastructure load (decided §H.4).

### Stories & tasks
See `tasks-ep58-team-shifts-authorship.json` (S-58-01 … S-58-07; T-58-001 … T-58-016).
1. **S-58-01 — Trackers** — T-58-001 (done).
2. **S-58-02 — Staff sign-in** — T-58-002 Keycloak accounts for staff · T-58-003 Tenant resolution for staff tokens.
3. **S-58-03 — Two roles** — T-58-004 Role model and policies · T-58-005 Role tests across the API.
4. **S-58-04 — Staff page** — T-58-006 Vendor Portal Staff page · T-58-007 Activity logging.
5. **S-58-05 — Bouquet authorship** — T-58-008 MadeBy on bouquets, listings and order lines.
6. **S-58-06 — Work calendar** — T-58-009 Shift roster model and API · T-58-010 Calendar pages · T-58-011 On shift now and assignment.
7. **S-58-07 — Hours, stats and alerts** — T-58-012 Clock in/out and hours export · T-58-013 Per-florist stats · T-58-014 Shift and assignment alerts · T-58-015 Staff seats on the vendor plan · T-58-016 Vendor app follow-up.

---

## Programme order

1. **Wave 1 — before Walentynki / Dzień Kobiet 2027:** EP-43, EP-44, EP-45, EP-49, EP-47, EP-50.
2. **Wave 2 — before Dzień Matki 2027:** EP-46, EP-58, EP-48, EP-53, EP-57.
3. **Wave 3 — before Wszystkich Świętych 2027:** EP-55, EP-54, EP-52, EP-51, EP-56.

Confirm each epic's §H decisions in REQ-GAPS-U before its first implementation story.
