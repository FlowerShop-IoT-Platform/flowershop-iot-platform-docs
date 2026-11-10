# Requirement Gaps — Option U (Florist operating system: covering every business flow)

> Companion to the competitor and gap analysis "FindMyFlowers — Polish Market Competitor & Gap Analysis"
> (reviewed 2026-11-08 against the code) and to REQ-GAPS-J/O/Q (stock, customer requests, online-only vendors).
> Requirement refs point at the tasks in `tasks-ep43-*.json` … `tasks-ep58-*.json`.
> Analysis date: 2026-11-09
> Status: proposed (EP-43 … EP-58, see `EPICS-U.md`). Each section's §H decisions are open unless marked decided.

## Context

FMF adds sales channels (smart vase, online listings, custom requests) and has the strongest AI tooling in the
Polish market, but florists choose software that runs their whole day: taking orders from every channel, making,
delivering, getting paid, invoicing, buying flowers and keeping customers. Competitors (Bukiet.online + Bukiet CRM,
Floraboard, Kwiatostan CRM, Poczta Kwiatowa, Wolt/Glovo) each cover part of that. A review of `main` on 2026-11-08
found these gaps; each epic below states what already exists so the work is incremental.

**Goal:** a Polish florist or flower shop can run a normal day, and a peak day, entirely in FMF.

### Already in place (not repeated as requirements)

Stock with unit cost, waste ledger, low stock, recipe consumption, photo stock count (EP-30/31/34); AI Florist,
bouquet reconstruction, cost assessor, atlas, calibrated confidence (EP-28/32–36/40/41); custom requests with offers,
SLA monitor and 30-minute payment (EP-37); online-only listings (EP-39); BLIK + card via Stripe; post-order ratings
(EP-27); freshness price tiers as vendor suggestions; employee roles API; vendor SaaS billing (EP-20); vase ordering and
recovery (EP-21/22); Google SSO (EP-23).

### Epic map

| Epic | Business flow | Priority | Requirements |
|---|---|---|---|
| EP-43 Polish-first experience, with English and Ukrainian | Every portal page, email and AI Florist text in Polish by default, with English and Ukrainian as full second languages. | P1 | REQ-FOS-01 … REQ-FOS-10 |
| EP-44 Florist payouts and Polish payment methods | Money reaches the florist automatically, with a payout statement, and customers can pay the way Poles pay. | P1 | REQ-FOS-11 … REQ-FOS-16 |
| EP-45 Push alerts for florists and customers | Real phone notifications for the events that make or lose a sale. | P1 | REQ-FOS-17 … REQ-FOS-23 |
| EP-46 One order book: every channel, one production board | Phone, walk-in, Instagram and FMF orders in one list and one production board, so FMF replaces the notebook. | P1 | REQ-FOS-24 … REQ-FOS-29 |
| EP-47 Made-to-order catalogue and gift order details | Repeatable bouquet types in S/M/L next to one-off real bouquets, plus the gift details every flower order needs. | P1 | REQ-FOS-30 … REQ-FOS-36 |
| EP-48 Delivery slots, zones, capacity, pre-orders and closures | Customers pick a real delivery window; florists control how much they take on each day, including peak days. | P1 | REQ-FOS-37 … REQ-FOS-42 |
| EP-49 Fulfilment proof and recipe-linked production | The customer sees the finished bouquet before it leaves and a photo at the door; stock follows the recipe automatically. | P1 | REQ-FOS-43 … REQ-FOS-47 |
| EP-50 Invoices, KSeF and accounting export | Invoices go out automatically through the florist's own invoicing app (which talks to KSeF), and the accountant gets a monthly export. | P1 | REQ-FOS-48 … REQ-FOS-53 |
| EP-51 Courier dispatch and pickup partners | Deliveries assigned to the florist's own courier or booked with a courier partner, with tracking, plus after-hours pickup points. | P2 | REQ-FOS-54 … REQ-FOS-58 |
| EP-52 Purchasing, suppliers and demand forecast | From "what is low" to "what to buy on Monday", with real purchase costs feeding AI Florist prices. | P2 | REQ-FOS-59 … REQ-FOS-64 |
| EP-53 Customer–florist conversations | A chat with photos on every custom request and order, without leaking contact details before payment. | P1 | REQ-FOS-65 … REQ-FOS-70 |
| EP-54 Customer book, reminders and repeat sales | Florists know their customers, customers are reminded of the dates that matter, and flowers can be bought on repeat. | P2 | REQ-FOS-71 … REQ-FOS-76 |
| EP-55 Events, weddings, funerals and business accounts | Quotes with deposits for weddings and events, a proper funeral range, and business customers with standing orders. | P2 | REQ-FOS-77 … REQ-FOS-81 |
| EP-56 Locations, decision analytics and audit | Owners can run more than one shop, see the numbers that drive decisions and trace money-related changes. | P2 | REQ-FOS-82 … REQ-FOS-84 |
| EP-57 Public florist profiles, SEO and social channels | Florists and live bouquets found on Google in Polish, with profiles, photo reviews and links from Instagram and Google Business Profile. | P2 | REQ-FOS-85 … REQ-FOS-91 |
| EP-58 Team, shifts and bouquet authorship | A medium or large shop runs its team in FMF: staff sign in with their own accounts, two clear roles, a work calendar showing who is on shift, and every bouquet credited to the florist who made it. | P1 | REQ-FOS-92 … REQ-FOS-103 |

---

## A. EP-43 — Polish-first experience, with English and Ukrainian

**Why:** All three portals render in English (`<html lang="en">`, hard-coded strings); only the API sets `pl-PL` culture (`LocalisationMiddleware`). Every Polish competitor (Bukiet.online, Kwiatostan, Poczta Kwiatowa) is Polish-first, and home florists will not adopt an English back office. Ukrainians are one of the largest groups of residents in Poland, both as customers and as florists and shop staff, and none of the Polish florist tools covered in the gap review offers a Ukrainian interface.

### REQ-FOS-01 — Localisation infrastructure in all portals — P0
Vendor Portal, Customer App and Admin Portal use ASP.NET Core request localisation with `pl-PL` as default and `en-GB` and `uk-UA` as further cultures, resolved from a culture cookie, then the signed-in user's stored preference, then `Accept-Language`. A language switcher (Polski / English / Українська) in the header and account settings sets the cookie and the preference. `<html lang>` follows the culture. Dates, times, numbers and money use the culture (`25,00 zł`), and a plural-aware text helper covers Polish and Ukrainian plural forms (e.g. 1 bukiet / 2 bukiety / 5 bukietów; 1 букет / 2 букети / 5 букетів).
**Accept**: a first visit with no headers renders Polish; `Accept-Language: uk` renders Ukrainian; switching language persists across sign-in; `<html lang>` matches; plural helper tests for pl and uk.

### REQ-FOS-02 — Stored language preference — P1
`User` (and so customers, vendor admins and employees) gets `PreferredLanguage` (`pl`/`en`/`uk`), editable in profile/settings and returned in the profile APIs. The Keycloak login theme has Polish, English and Ukrainian messages and follows the same choice.
**Accept**: migration adds the column with default `pl`; profile endpoints read and write all three values; contract updated.

### REQ-FOS-03 — Vendor Portal translated — P0
All `Pages/**` texts, validation messages, status names (order, request, stock movement), empty states, toasts and the ProblemDetails messages the portal shows are translated into Polish and Ukrainian (English is the source). Enum display names come from resources, not `ToString()`.
**Accept**: no untranslated text on any Vendor Portal page in `pl` or `uk` (checked by the REQ guard and a manual walk-through per language).

### REQ-FOS-04 — Customer App translated — P0
All `Views/**`, checkout, cart, request and order-tracking texts, Stripe return pages and error pages are translated into Polish and Ukrainian. Map and bouquet cards use translated freshness wording.
**Accept**: as above for the Customer App, in both languages.

### REQ-FOS-05 — Emails and notifications in the recipient's language — P1
`EmailService` templates, vendor inbox (`VendorNotification`) texts, push texts (EP-45) and SMS texts exist in Polish, English and Ukrainian and are chosen by the recipient's preference, not the sender's.
**Accept**: unit test renders each template in all three languages; no template falls back to another language silently.

### REQ-FOS-06 — AI Florist output in the vendor's language — P1
Recipe names, florist notes, offer drafts and cost-assessor explanations are produced in the vendor's language: Polish, English or Ukrainian (prompt instruction + Polish and Ukrainian common names from the flower atlas where present; missing names are added to the atlas seed).
**Accept**: deterministic provider and one hosted-provider snapshot test produce Polish text for a `pl` vendor and Ukrainian text for a `uk` vendor.

### REQ-FOS-07 — Ukrainian specifics — P1
Cyrillic is covered by the portals' fonts (no fallback boxes) and by generated PDFs (job cards, quotes, payout statements); search and name matching handle Cyrillic input; third-party screens (Stripe checkout, Keycloak, Turnstile, Twilio Verify SMS) use their Ukrainian locale where the provider offers one, else Polish. Vendors can declare the languages they speak (`LanguagesSpoken`), shown as a badge on their public profile (EP-57) and usable as a customer filter ("speaks Ukrainian").
**Accept**: a Ukrainian-language walk-through of sign-up, checkout and a job-card PDF shows no missing glyphs; the language filter returns only vendors who declared Ukrainian.

### REQ-FOS-08 — Missing-translation guard — P1
A test checks `pl`, `en` and `uk` resource key parity and scans `.cshtml` files for hard-coded user-visible strings outside an allow-list.
**Accept**: the test fails on a planted untranslated string or a key missing from any one language.

### REQ-FOS-09 — Polish and Ukrainian manuals and help — P2
The vendor and customer manuals in the docs portal get Polish and Ukrainian versions; the Help menu opens the version matching the user's language.
**Accept**: `scripts/docs/check_audiences.py` passes with the new pages.

### REQ-FOS-10 — Mobile apps — P1
The vendor app and customer mobile app use the same language preference and ship Polish and Ukrainian strings (follow-up in their repos).
**Accept**: follow-up trackers raised in the mobile repos.

**Open decisions (§H, EP-43)**
1. Should the Admin Portal be translated in this epic (proposed: infrastructure yes, strings later, P3; Polish only)?
2. Who reviews translations (proposed: a native-speaking florist pilot user signs off the Polish glossary, and a native Ukrainian speaker signs off the Ukrainian one)?
3. Ship Ukrainian together with Polish (proposed: Customer App and Vendor Portal in the same release, so the guard can require all three from day one) or as a follow-up release?

---

## B. EP-44 — Florist payouts and Polish payment methods

**Why:** Every customer payment lands in the platform's Stripe account (`StripeService` creates plain PaymentIntents; there are no transfers or connected accounts), so florists must be settled by hand. Bukiet.online pays florists directly through TPay. BLIK and card are live; Przelewy24, wallets and instalments are not.

### REQ-FOS-11 — Connected account onboarding — P0
Each vendor gets a Stripe Connect Express account, created and linked from Vendor Portal Payments ("Set up payouts"), with Stripe-hosted onboarding (KYC, bank account). Status (`NotStarted`, `Pending`, `Enabled`, `Restricted`) is stored on the vendor and updated from `account.updated` webhooks.
**Accept**: a test vendor completes onboarding in test mode; status shows Enabled; restricted accounts show the Stripe requirement text.

### REQ-FOS-12 — Destination charges with a platform fee rule — P0
All customer PaymentIntents (cart orders, online listings, custom-request orders) set `transfer_data.destination` to the vendor's account and `application_fee_amount` from `Payments:PlatformFee` (percent + fixed, per plan; default 0). Orders cannot be paid when the vendor's account is not Enabled; the customer sees a clear message and the listing is hidden from purchase.
**Accept**: handler tests for fee calculation; an order in test mode shows the transfer on the connected account.

### REQ-FOS-13 — Refunds and disputes reverse the transfer — P0
Refunds use `reverse_transfer` and `refund_application_fee`; the existing refund retry (`NeedsRefundRetry`) covers the new parameters; dispute webhooks mark the order and notify the vendor with the evidence deadline.
**Accept**: refund test in Stripe test mode leaves the connected balance correct.

### REQ-FOS-14 — Payout statement — P1
Vendor Portal Payments shows per period: gross sales, platform fees, Stripe fees, refunds, disputes, net, and the payouts with arrival dates; CSV export. Admin Portal shows each vendor's account status and any blocked payouts.
**Accept**: statement totals equal the sum of the period's balance transactions in a test.

### REQ-FOS-15 — More Polish payment methods — P2
Add Przelewy24 and card wallets (Apple Pay, Google Pay) for PLN PaymentIntents, and an instalment / pay-later option the processor supports for PLN (§H.4). The existing BLIK in-flight handling is generalised to any asynchronous method.
**Accept**: checkout offers BLIK, P24, card and wallets for PLN; async methods do not lapse mid-flight.

### REQ-FOS-16 — Go-live for existing vendors — P1
Existing vendors are asked to complete onboarding with a dashboard banner and email; orders paid before go-live are listed for manual settlement in the Admin Portal.
**Accept**: admin report lists pre-go-live orders not yet settled.

**Open decisions (§H, EP-44)**
1. Stripe Connect Express (proposed) or a Polish split-payment provider (Przelewy24 Marketplace, TPay split)? Stripe keeps the current code; a Polish provider may be cheaper for BLIK.
2. Platform fee: stay at 0 % with SaaS only (proposed for launch), or a commission on FMF-sourced orders only?
3. Should a vendor without verified payouts be able to list at all (proposed: list yes, take payment no)?
4. Which pay-later option for PLN (depends on what the processor offers at the time)?

---

## C. EP-45 — Push alerts for florists and customers

**Why:** `NotificationServiceClient` is a no-op stub and EP-06 T-06-007/T-06-008 (Firebase) are still todo. Florists hear about new custom requests only through the inbox and email, and custom requests are won by the first good offer.

### REQ-FOS-17 — Firebase Cloud Messaging adapter — P0
`INotificationServiceClient` gets an FCM HTTP v1 implementation (service-account secret in Fly secrets, `Push:Provider=Fcm`, stub kept for dev). Invalid or unregistered tokens are removed from `DeviceRegistration`.
**Accept**: integration test with a fake FCM endpoint; stale token removed on `UNREGISTERED`.

### REQ-FOS-18 — Vendor alert catalogue — P0
New paid order; new bouquet request in the service radius; offer accepted / rejected / lapsed; SLA warning before a promised time; customer message (EP-53); low stock; vase offline (in-store only); new review. Each alert has a deep link and a Polish and English text.
**Accept**: each alert type has a test that raises the event and asserts one push per device.

### REQ-FOS-19 — Customer alert catalogue — P1
Order status changes (paid, ready, out for delivery, delivered), finished-bouquet photo to approve (EP-49), offer received on a request, bouquet-alert subscription match, florist message.
**Accept**: as above for customers.

### REQ-FOS-20 — Preferences and quiet hours — P1
Per-user toggles per alert group and quiet hours (default 22:00–07:00, vendor SLA alerts exempt). Settings pages in both portals and the apps.
**Accept**: a muted group sends nothing; quiet hours delay non-urgent alerts.

### REQ-FOS-21 — Reliable delivery — P1
Alerts are dispatched from the outbox with an idempotency key per (event, user); retries with back-off; delivery outcome logged for the admin.
**Accept**: replaying an outbox message does not send a second push.

### REQ-FOS-22 — Browser notifications in the Vendor Portal — P2
Optional Web Push for new requests and paid orders while the portal is open or installed as a PWA.
**Accept**: a vendor who allows notifications gets one for a new request.

### REQ-FOS-23 — Mobile token registration — P1
Vendor app and customer app register tokens on sign-in and remove them on sign-out (follow-up in their repos).
**Accept**: follow-up trackers raised.

**Open decisions (§H, EP-45)**
1. Is Web Push for the Vendor Portal in scope for v1 (proposed: P3, after mobile)?

---

## D. EP-46 — One order book: every channel, one production board

**Why:** The Vendor Portal lists FMF orders only (`Pages/Orders`); there is no way to record a phone or Instagram order, no due-time view and no record of cash or terminal payments. Bukiet CRM and Floraboard both lead with exactly this.

### REQ-FOS-24 — Order source and contact — P0
`Order` gets `Source` (`Fmf`, `Phone`, `WalkIn`, `Instagram`, `Facebook`, `WhatsApp`, `Email`, `Other`), `DueAt` and an optional `OrderContact` (name, phone, email). `CustomerId` becomes optional for non-FMF sources.
**Accept**: migration; domain tests for each source; FMF orders unchanged.

### REQ-FOS-25 — Create and edit manual orders — P0
Vendor Portal "New order": contact, items (free text with price, a catalogue item from EP-47, or a saved recipe), quantity, due date/time, pickup or delivery (address, slot from EP-48), card message and notes. Editable until Done; cancel with reason.
**Accept**: page-model and handler tests; an Instagram order appears in the list and on the board.

### REQ-FOS-26 — Offline payments and balances — P0
Record payments on any order: cash, card terminal, bank transfer, other; deposits and partial payments; balance due shown on the order and the board. Online (Stripe) payments appear in the same list read-only.
**Accept**: balance = total − payments; refund of a recorded payment is a negative entry.

### REQ-FOS-27 — Production board — P0
Board with columns To make → Made → Ready → Out for delivery → Done, grouped by due day (today, tomorrow, later), filters by source, florist and delivery/pickup; one-click status moves that call the existing order transitions; printable job card (items, recipe, card message, address, due time).
**Accept**: status moves on the board equal the order API transitions; job card prints on A6/A5.

### REQ-FOS-28 — Calendar view — P1
Month and week calendar of due orders with daily load against capacity (EP-48) and peak days highlighted.
**Accept**: calendar counts match the board.

### REQ-FOS-29 — API and contract — P1
Endpoints under `/api/v1/vendor/orders` for manual create/update, payments and board moves; contract section; the vendor app can use the same.
**Accept**: contract updated; endpoints tenant-scoped.

**Open decisions (§H, EP-46)**
1. Link a manual-order contact to an FMF customer automatically by phone/email (proposed: suggest, vendor confirms)?
2. Should manual orders count toward the vendor's analytics and AI Florist taste signals (proposed: analytics yes, taste no)?

---

## E. EP-47 — Made-to-order catalogue and gift order details

**Why:** Today a listing is one physical bouquet that sells once (vase bouquet or EP-39 online listing). Most online flower orders are "this style, made today", and every competitor offers a catalogue. Orders also lack a card message, a separate recipient, anonymous sending, a delivery date and add-ons (`Order` has only `DeliveryAddress` with notes).

### REQ-FOS-30 — CatalogueItem aggregate — P0
Vendor-scoped item: name, description, category (bouquet, box, plant, funeral, add-on), 1–5 example photos (private upload, public thumbnails), sizes S/M/L each with price and lead time, availability days, optional link to a saved AI Florist recipe per size, active/archived.
**Accept**: domain tests; Vendor Portal catalogue CRUD.

### REQ-FOS-31 — Customer browsing — P0
Catalogue items show on the florist page, in a "made to order" map/search filter with earliest delivery (EP-48), and in bouquet search by occasion and price.
**Accept**: a catalogue item with an S size at 129 zł is found by filter and shows the earliest slot.

### REQ-FOS-32 — Cart and checkout for catalogue lines — P0
Cart and `OrderLine` support catalogue lines (item, size, price snapshot, no `BouquetId`); `IsPurchasable` is extended; the board (EP-46) shows the size and recipe.
**Accept**: checkout of a mixed cart is rejected if vendors differ (one vendor per order, as today).

### REQ-FOS-33 — Gift details on orders — P0
Recipient name and phone (separate from the buyer), card message (≤ 250 characters, printed on the job card), anonymous sender flag (the recipient is not told who sent it), delivery date and time window.
**Accept**: the job card and vendor order view show the card message; the recipient phone is hidden from the vendor until paid (as EP-37 PII rule).

### REQ-FOS-34 — Add-ons — P1
Vendor-defined extras (glass vase, chocolates, balloon, greeting card) with price, optional stock link (decrements stock on Made) and a per-order limit.
**Accept**: add-on totals included in the order and invoice.

### REQ-FOS-35 — Send to a phone number — P2
The buyer can give only the recipient's phone; the recipient gets an SMS link to confirm the address and a time window; if not confirmed in time, the buyer is notified and refunded by rule.
**Accept**: end-to-end test with the SMS stub.

### REQ-FOS-36 — Save a recipe as a catalogue item — P2
From AI Florist saved recipes, "Add to catalogue" pre-fills an item with sizes from the recipe sizes (EP-33 S/L) and the preview as a draft photo the florist must replace with a real one before publishing.
**Accept**: publishing is blocked while only an AI preview photo is attached.

**Open decisions (§H, EP-47)**
1. Allow a stock or AI image on catalogue items at all (proposed: no — own photos only, AI preview as draft)?
2. One vendor per order stays (proposed: yes)?

---

## F. EP-48 — Delivery slots, zones, capacity, pre-orders and closures

**Why:** Delivery is one global rule: a flat `Delivery:FeePln` = 25 zł within `Delivery:MaxDistanceKm` = 10 km (`DeliveryOptions`). There are no time windows, no capacity, no same-day cut-off, no closed days and no pre-orders, which is where flower orders fail on Walentynki, Dzień Kobiet and Dzień Matki.

### REQ-FOS-37 — Delivery zones per vendor — P0
Zones as distance rings or postcode lists, each with a fee, minimum order value and optional free-delivery threshold. Address checks use the existing geocoder.
**Accept**: zone tests for ring and postcode; vendors without zones use the global default.

### REQ-FOS-38 — Delivery and pickup slots — P0
Weekly slot template (e.g. 9–12, 12–15, 15–18) per vendor, same-day cut-off time, per-item lead time (EP-47), capacity per slot. Pickup slots for in-store vendors.
**Accept**: slot generator tests including cut-off and lead time.

### REQ-FOS-39 — Slot picker and reservation — P0
Checkout shows available slots for the address and items; the chosen slot is reserved with the order, released on lapse or cancel; full slots are not offered. Online listings and custom requests use the same picker for their delivery time.
**Accept**: concurrency test: two checkouts for the last place in a slot — one wins.

### REQ-FOS-40 — Closures and special days — P1
Closed dates and special opening hours, with Polish holiday presets (e.g. 1 Jan, Easter, 1 Nov, 24–26 Dec). Closures hide purchase on listings and block slots.
**Accept**: a closed day offers no slots and the map shows "opens on …".

### REQ-FOS-41 — Pre-order campaigns for peak days — P1
The vendor opens pre-orders for a date (e.g. 14 Feb) with a daily cap, cut-off and optional dedicated catalogue items; customers see "Order for Walentynki" ahead of time; the board shows the campaign load.
**Accept**: cap reached → campaign shows sold out.

### REQ-FOS-42 — Capacity on the board and calendar — P1
EP-46 calendar shows slot load vs capacity and warns when manual orders exceed it.
**Accept**: calendar load equals reserved slots.

**Open decisions (§H, EP-48)**
1. Should online-only vendors (EP-39) get pickup slots (EP-39 decided no pickup; revisit)?
2. Capacity in orders per slot (proposed) or florist minutes per bouquet size?

---

## G. EP-49 — Fulfilment proof and recipe-linked production

**Why:** "It didn't look like the picture" is the top complaint about catalogue florists, and FMF's brand is the real bouquet. The order has a timeline and a delivery QR, but no finished-bouquet photo, no approval and no proof-of-delivery photo. Stock can be consumed by recipe (`POST /vendor/stock/consume`) but nothing links an order line to a recipe.

### REQ-FOS-43 — Finished-bouquet photo on the order — P0
At Made, the vendor uploads 1–3 photos (portal or app); stored as private media, shown to the customer and on the order.
**Accept**: upload test; photo visible to the order's customer only.

### REQ-FOS-44 — Customer approval window — P0
Per vendor setting and order type, the customer can approve or ask for a change (text) within N minutes (default 20); silence = approved; a change request notifies the florist and keeps the order in Made.
**Accept**: timer test with a fake clock.

### REQ-FOS-45 — Proof of delivery — P1
At delivery the courier or florist adds a photo and optional recipient name; scanning the existing delivery QR completes it; the buyer is notified.
**Accept**: order Done with POD data in the timeline.

### REQ-FOS-46 — Recipe on order lines — P1
Order lines can link a saved recipe (from catalogue size, offer or picked manually). At Made, stock is consumed with the existing all-or-nothing consume and an idempotency key; shortages show what is missing and allow override.
**Accept**: consume runs once per line even when Made is clicked twice.

### REQ-FOS-47 — Photos as evidence — P2
Finished and delivery photos are attached to Stripe dispute evidence and, with customer consent, to the florist's public portfolio (EP-57).
**Accept**: dispute evidence upload test in Stripe test mode.

**Open decisions (§H, EP-49)**
1. Default approval window length (proposed 20 min, vendor can set 10–60)?

---

## H. EP-50 — Invoices, KSeF and accounting export

**Why:** KSeF e-invoicing applies to most firms from 1 April 2026 and to the smallest sellers from 1 January 2027. FMF has no invoices at all: Stripe receipts only, no company tax number (NIP) at checkout, and the platform's own charges to vendors (SaaS plan, vases) are Stripe invoices, not KSeF invoices. Bukiet auto-invoices after fulfilment.

### REQ-FOS-48 — Invoicing provider port and adapters — P0
`IInvoicingProvider` (issue invoice/receipt, get status, get PDF, cancel/correct) with adapters for Fakturownia and one more provider (§H.1); the vendor connects their account in Settings with an API token; a connection test.
**Accept**: adapter tests against recorded responses; token stored encrypted.

### REQ-FOS-49 — Invoice details at checkout — P0
"I need an invoice" with company name, NIP (checksum validation), address; stored on the order; also on manual orders and custom-request payments.
**Accept**: invalid NIP rejected; data reaches the provider call.

### REQ-FOS-50 — Automatic issuing — P0
Per vendor rule: issue on payment or on completion; invoice number, KSeF number/status and PDF link stored on the order; failures retried and shown on the order and dashboard.
**Accept**: retry test; KSeF status shown when the provider returns it.

### REQ-FOS-51 — VAT settings — P1
VAT rate per catalogue item/add-on/delivery fee with a vendor default, and support for VAT-exempt small businesses (the vendor declares the exemption basis).
**Accept**: invoice lines carry the configured rates.

### REQ-FOS-52 — Platform invoices to vendors — P0
SaaS subscription, vase orders and platform fees are invoiced to vendors through the platform's own KSeF-capable provider, using the billing address and NIP from registration (EP-19).
**Accept**: a subscription renewal produces a KSeF invoice in the provider sandbox.

### REQ-FOS-53 — Accountant export — P1
Monthly CSV per vendor: orders (all sources), invoice numbers, VAT, payment method, platform and Stripe fees, refunds, payouts; plus a summary PDF. Sent by email to an accountant address if set.
**Accept**: export totals match the payout statement (EP-44).

**Open decisions (§H, EP-50)**
1. Which providers first (proposed: Fakturownia, then ifirma; check API access and KSeF support at build time)?
2. Fiscal receipts for walk-in/manual cash sales: out of scope (proposed) or via a receipt-printer integration later?
3. VAT rates for flowers vary by product type; the vendor sets rates — FMF gives no tax advice in the UI (proposed).

---

## I. EP-51 — Courier dispatch and pickup partners

**Why:** Orders end at "Out for delivery" with a fixed fee; there is no courier assignment, route sheet, partner booking or tracking. FMF should partner on logistics (Wolt Drive, Glovo, Stuart) and 24/7 pickup (1,300+ kwiatomaty) rather than build a fleet.

### REQ-FOS-54 — Own-courier assignment and route sheet — P1
Assign an order to an employee with the Courier role; route sheet per slot with ordered stops, navigation links, recipient phone and card notes; printable and in the vendor app.
**Accept**: route sheet lists all assigned orders for the slot.

### REQ-FOS-55 — Courier partner adapter — P1
`ICourierProvider` with quote, book, track, cancel and webhooks; first adapter per §H.1; status updates move the order (picked up, delivered, failed).
**Accept**: adapter contract tests with recorded responses.

### REQ-FOS-56 — Tracking link — P2
Buyer and recipient get a tracking link (partner's or FMF's own with courier status).
**Accept**: link shown in the order and alert.

### REQ-FOS-57 — After-hours pickup points — P3
Pickup point type for partner kwiatomaty/lockers: the florist loads the order, the customer receives a code by SMS; pilot with one operator.
**Accept**: pilot flow documented and tested with a stub.

### REQ-FOS-58 — Simple route ordering — P3
Nearest-neighbour ordering of stops for the route sheet.
**Accept**: unit test on a fixed set of addresses.

**Open decisions (§H, EP-51)**
1. First courier partner: Wolt Drive, Glovo On-Demand or Stuart (depends on API access and coverage in pilot cities)?
2. Kwiatomat pilot partner and commercial model?

---

## J. EP-52 — Purchasing, suppliers and demand forecast

**Why:** Stock, waste and low-stock exist (EP-30) and AI Florist produces a missing-flowers list (EP-35), but there is no reorder list from upcoming orders, no supplier book, no one-step delivery receipt and `IFlowerSupplierCatalog` is a no-op. Kwiatostan sells AI reorder and forecasting; Floraboard rolls recipes into supplier orders.

### REQ-FOS-59 — Reorder list — P1
Computed per item from low-stock thresholds, recipes of orders due in the next N days, pre-order campaigns and events; editable quantities; export as PDF/CSV or copyable text for WhatsApp.
**Accept**: unit test of the calculation with fixtures.

### REQ-FOS-60 — Supplier book — P2
Suppliers with contacts, usual delivery days and items they supply.
**Accept**: CRUD with tenant scoping.

### REQ-FOS-61 — Purchase receipt — P1
Record a delivery in one step (supplier, delivery-note number, lines with quantity and unit cost) creating `Received` movements and a cost history per item.
**Accept**: one receipt creates N movements atomically with an idempotency key.

### REQ-FOS-62 — Supplier price feed — P2
Implement `IFlowerSupplierCatalog` with one source (§H.1) so AI Florist and the cost assessor can use current wholesale prices when stock cost is missing.
**Accept**: assessor uses the feed price when no stock cost exists.

### REQ-FOS-63 — Waste report — P2
Wasted value by item and week, and as a share of purchases.
**Accept**: report totals match `Wasted` movements.

### REQ-FOS-64 — Demand forecast — P3
Weekly forecast per item from order history plus Polish holiday and name-day calendar uplift; shown next to the reorder list.
**Accept**: forecast explained per item ("Dzień Matki +60 %").

**Open decisions (§H, EP-52)**
1. First price source: Floriday (needs a buyer account) or a CSV price-list import from Polish wholesalers (proposed first)?

---

## K. EP-53 — Customer–florist conversations

**Why:** Custom requests and weddings need clarification before an offer, but today there is only a one-shot contact email (`VendorMessage`, EP-27, "no threading") and offer notes. EP-37 deferred optional messages. Instagram DMs are the status quo FMF must replace.

### REQ-FOS-65 — Conversation model — P0
`Conversation` aggregate per (bouquet request, vendor) and per order; messages with text and up to 5 photos; read receipts; tenant and customer scoping.
**Accept**: domain tests; only the two parties can read.

### REQ-FOS-66 — Contact masking before payment — P0
Before payment on a request, phone numbers, e-mails and links are masked server-side with a notice to both sides; after payment masking stops for that order.
**Accept**: masking tests for Polish phone formats and obfuscated e-mails.

### REQ-FOS-67 — Real-time and alerts — P1
New messages over `MarketplaceHub`, push (EP-45), and an e-mail digest when unread for 30 minutes.
**Accept**: unread digest test with a fake clock.

### REQ-FOS-68 — Inbox and thread UIs — P0
Vendor Portal: request and order pages get a thread panel; inbox merges `VendorNotification` and conversations. Customer App: thread on request and order pages.
**Accept**: page-model tests for both portals.

### REQ-FOS-69 — Moderation and limits — P1
Report a message; admin view in the Admin Portal; rate limits per user; block a user from a conversation.
**Accept**: rate limit returns 429 with Retry-After.

### REQ-FOS-70 — Retention — P1
Messages and photos deleted 12 months after the conversation closes (GDPR); documented in the privacy notice.
**Accept**: retention job test.

**Open decisions (§H, EP-53)**
1. May a customer message a vendor on a request before the vendor sends an offer (proposed: yes, vendor-initiated questions only)?

---

## L. EP-54 — Customer book, reminders and repeat sales

**Why:** FMF has favourites, bouquet-alert subscriptions and ratings, but no florist customer list, no occasion or name-day (imieniny) reminders, no recurring bouquets, no discount codes or vouchers and no loyalty. Freshness price tiers exist (−20/−40/−70 % at 24/48/72 h) but are only suggestions to the vendor.

### REQ-FOS-71 — Florist customer book — P1
Per vendor: customers and manual contacts with order history, totals, notes, important dates and marketing consent; search; CSV export of consenting contacts.
**Accept**: tenant-scoping tests; consent required for export.

### REQ-FOS-72 — Occasion and name-day reminders — P1
Customers save dates (birthday, anniversary, name day by name); reminder N days before with bouquets from favourite florists; a Polish name-day dataset as reference data.
**Accept**: name → dates lookup test; reminder job test.

### REQ-FOS-73 — Recurring bouquets — P2
Weekly/biweekly/monthly subscription to a catalogue item and size, paid off-session; skip, pause, cancel; each occurrence becomes an order with a slot (EP-48).
**Accept**: renewal creates an order and payment; failed payment pauses with notice.

### REQ-FOS-74 — Discount codes and gift vouchers — P2
Vendor-funded codes (% or amount, minimum order, usage limit, expiry); digital gift vouchers bought by customers and redeemed at the same vendor.
**Accept**: code validation tests; voucher balance tracking.

### REQ-FOS-75 — Stamp-card loyalty — P3
Optional per vendor: every Nth paid order gets a set discount.
**Accept**: counter test.

### REQ-FOS-76 — "Last chance" markdowns — P2
When `AutoAdjustPrice` is on, the freshness tier price is applied automatically to the listing; customers can filter "last chance"; the map shows a badge.
**Accept**: price changes at tier boundaries in a fake-clock test.

**Open decisions (§H, EP-54)**
1. Gift vouchers are prepayments with VAT consequences; launch with discount codes only (proposed) and add vouchers after accountant review?

---

## M. EP-55 — Events, weddings, funerals and business accounts

**Why:** Weddings, events, funerals and office flowers are the highest-value and most regular work for Polish florists, and FMF has nothing specific for them: no quote, no deposit (zadatek), no ribbon text, no standing orders or monthly invoicing. Details Flowers, Curate and Floranext make event proposals standard abroad.

### REQ-FOS-77 — Event quotes — P1
Enquiry (type, date, venue, guests, budget, inspiration photos) → quote with items and quantities, optional linked recipes and costs, mood board, terms; Polish PDF; versions; sent by link.
**Accept**: PDF snapshot test; versions keep history.

### REQ-FOS-78 — Acceptance and deposit — P1
Customer accepts online and pays a deposit (percentage set by vendor); balance due date with reminders; cancellation terms shown and stored.
**Accept**: deposit and balance recorded as order payments (EP-46).

### REQ-FOS-79 — Event tasks on the board — P2
On the event date the board shows production, setup and teardown tasks with assignees.
**Accept**: board shows event tasks.

### REQ-FOS-80 — Funeral range — P2
Catalogue category for wreaths and sympathy arrangements with ribbon text field, delivery to a funeral home or cemetery with exact time, and an urgent flag that changes slot rules.
**Accept**: ribbon text appears on the job card.

### REQ-FOS-81 — Business accounts and standing orders — P2
Company customer with several authorised orderers, standing orders (e.g. weekly reception flowers), and one monthly invoice (EP-50).
**Accept**: monthly invoice aggregates the period's orders.

**Open decisions (§H, EP-55)**
1. Default deposit percentage and refund terms are the vendor's choice; should FMF offer a template text (proposed: yes, editable)?

---

## N. EP-56 — Locations, decision analytics and audit

**Why:** There is one location per vendor. Analytics pages exist but do not show channel split, margin per bouquet or response time to requests, and money-related changes are not audited. Staff, roles and shifts moved to EP-58.

### REQ-FOS-82 — Multiple locations — P2
Location entity with address, hours, stock, vases and staff (EP-58 roster per location); orders, stock and the board filter by location; customer map shows each location.
**Accept**: a two-location vendor sees separate stock.

### REQ-FOS-83 — Decision analytics — P2
Sales by channel and location, margin per bouquet (price − recipe cost), vase utilisation, request response time and win rate, waste %; export.
**Accept**: figures match underlying data in tests.

### REQ-FOS-84 — Audit log — P2
Money-related and permission changes (refunds, payout settings, price changes, role changes from EP-58) logged with user and time, visible to the owner.
**Accept**: log entries for refund, payout settings and role change.

**Open decisions (§H, EP-56)**
1. Locations as children of one vendor (proposed) or a vendor per location with a parent account?

---

## O. EP-57 — Public florist profiles, SEO and social channels

**Why:** The Customer App has no public florist page (only `GET /vendors/{id}/public` in the API), no city or occasion pages, no structured data and no sitemap, so FMF only reaches people who already know it. Ratings exist (`BouquetFeedback`) but are not public and have no photos.

### REQ-FOS-85 — Public florist profile — P1
Server-rendered profile with Polish slug: description, photos, hours, location (approximate for home florists, as today), live bouquets, catalogue items, ratings and reviews.
**Accept**: profile renders without sign-in; home florist location stays approximate.

### REQ-FOS-86 — City and occasion pages — P1
City/district pages listing florists and live bouquets, and occasion pages published ahead of each peak; Polish slugs; sitemap.xml; canonical tags.
**Accept**: sitemap lists active profiles, cities and occasions.

### REQ-FOS-87 — Structured data — P1
schema.org JSON-LD for Florist/LocalBusiness, Product/Offer (price, availability) and AggregateRating.
**Accept**: Rich Results test passes on sample pages.

### REQ-FOS-88 — Expired listing handling — P1
Sold or expired bouquet URLs redirect (301) to the florist profile with a notice.
**Accept**: redirect test.

### REQ-FOS-89 — Photo reviews and replies — P2
`BouquetFeedback` gains an optional customer photo (moderated) and a florist reply; reviews show on the profile.
**Accept**: moderation queue in the Admin Portal.

### REQ-FOS-90 — Link-in-bio, QR and Google Business Profile — P2
A short link-in-bio page per florist with live bouquets and catalogue, QR download, and a guide to linking it from Instagram, Facebook and Google Business Profile.
**Accept**: page loads fast on mobile (Lighthouse ≥ 90).

### REQ-FOS-91 — Meta catalogue feed — P3
Per-vendor product feed of live listings and catalogue items for Instagram/Facebook shops.
**Accept**: feed validates in Meta's format.

**Open decisions (§H, EP-57)**
1. Slug format and whether home florists get a public profile URL with their name (privacy radius, EP-39 rules)?

---

## P. EP-58 — Team, shifts and bouquet authorship

**Why:** The backend has an `Employee` aggregate (roles Employee/Manager/Admin, a permission list, an activity log) and an `EmployeesController` (invite, list, change role, deactivate, activity), but none of it works end to end: accepting an invitation creates no Keycloak account (`AcceptEmployeeInvitationCommandHandler` returns a placeholder token), the invite/role/deactivate endpoints require a `ShopAdmin` role that vendor owners (`VendorAdmin`) don't have, the permissions are never checked, and nothing writes to the activity log. The Vendor Portal has no staff screen. Bouquets and orders don't record who made them, and there are no shifts or work calendar. For medium and large shops these are everyday needs: who is working today, who made this bouquet, and keeping front-line florists out of prices, payouts and settings.

### REQ-FOS-92 — Staff accounts that can sign in — P0
Accepting an invitation creates (or links) a Keycloak user with `vendor_id`/`vendor_type` attributes and a `VendorStaff` role, sets a password or Google sign-in, and returns real tokens; `TenantResolutionMiddleware` sets the vendor context for staff tokens with the staff member's user id. Deactivating an employee disables the Keycloak user and ends their sessions. Invitation e-mails expire after 7 days and can be resent.
**Accept**: integration test: invite → accept → sign in to the Vendor Portal as staff; deactivated staff can no longer obtain a token.

### REQ-FOS-93 — Two roles enforced by the API — P0
`EmployeeRole` becomes `OwnerManager` and `Florist` (existing `Admin`/`Manager` map to OwnerManager, `Employee` to Florist). API policies `VendorManage` (VendorAdmin or OwnerManager) and `VendorWork` (also Florist) replace `ShopAdmin`/`ShopEmployee`. Florists are refused (403 `staff-role-forbidden`) on prices and price strategies, payouts and payments, billing and plan, staff and shifts management, vendor settings, vase ordering and sales-mode changes. The permission list on `Employee` is derived from the role (no per-person permission editing in v1).
**Accept**: API tests per role for every refused area and for the allowed florist flows.

### REQ-FOS-94 — Staff page in the Vendor Portal — P0
Owner/Manager sees a Staff page: list with role, status and last sign-in; invite (name, e-mail, phone, role); change role; deactivate/reactivate; resend invitation; open a person's activity. Navigation and pages hide manager-only areas for florists.
**Accept**: page-model tests for both roles; a florist does not see Staff, Payments, Billing or Settings in the menu.

### REQ-FOS-95 — Bouquet and order authorship — P0
Bouquets (vase and online listings), catalogue order lines and custom-request orders record `MadeBy` (user id + name snapshot) and `MadeAt`. It is set when a staff member marks the item Made (EP-49) or creates the listing, and an Owner/Manager can correct it. The production board, order details and job card show the author; customers see only the shop name unless the shop turns on the per-shop option "Show florist's first name to customers" (off by default), which then shows "Made by Anna" on the bouquet page, order tracking and finished-bouquet photo.
**Accept**: domain tests; author shown on the board and job card; correction logged.

### REQ-FOS-96 — Activity log that is actually written — P1
Key actions write to the employee activity log: sign-in, bouquet made, listing created, price set (by managers), order status changes, stock movements, photo uploaded. The log is visible to Owner/Manager per person and for the shop.
**Accept**: each listed action creates one log entry with the acting user.

### REQ-FOS-97 — Work calendar and shifts — P1
Owner/Manager builds a weekly roster: shifts per person (start, end, optional location from EP-56 and note), copy a week, recurring patterns, leave and days off. Calendar views by week and by day; conflicts (overlap, leave) are flagged. Everyone in the shop, florists included, sees the whole team calendar in the portal and the vendor app, with their own shifts highlighted ("My shifts"); only Owner/Manager can edit it.
**Accept**: roster tests for overlap and leave; copy-week creates the same shifts a week later.

### REQ-FOS-98 — Who is on shift now — P1
Dashboard and production board show who is on shift now and next. Orders on the board (EP-46) can be assigned to a florist; assignment suggests people on shift at the order's due time and warns when assigning someone off shift.
**Accept**: on-shift list matches the roster at a fixed clock; warning shown for an off-shift assignee.

### REQ-FOS-99 — Optional clock in/out and hours — P2
Per shop setting: staff clock in and out (portal or app); hours worked vs rostered per person and period; CSV export for payroll. No payroll calculation in FMF.
**Accept**: hours export totals equal the clock records.

### REQ-FOS-100 — Per-florist view — P2
Owner/Manager sees per florist: bouquets made, orders completed, average customer rating of their bouquets (from `BouquetFeedback`), waste they recorded, hours; a florist sees only their own figures.
**Accept**: figures match the underlying records; a florist cannot read a colleague's stats via the API.

### REQ-FOS-101 — Shift and assignment alerts — P2
Staff get push/e-mail (EP-45) when their roster changes, the day before a shift, and when an order is assigned to them.
**Accept**: alert raised for each case in tests.

### REQ-FOS-102 — Staff seats in the SaaS plan — P1
Each vendor platform plan (EP-20) includes a number of staff seats (`Billing:Seats:IncludedPerPlan`, the owner's account is always free) and bills extra seats as a quantity on the Stripe subscription (`Billing:Seats:ExtraSeatPriceId`). Inviting beyond the allowance shows the price and asks the Owner/Manager to confirm, or to upgrade the plan; the quantity is updated with proration. Pending invitations and active staff use a seat; deactivating someone frees it at the next change. Billing page shows seats used / included / extra. Admin Portal can grant extra seats manually. Seat counts and prices are business settings, not code.
**Accept**: invite within allowance: no charge; invite beyond it: confirmation, subscription quantity +1 in Stripe test mode; deactivation lowers the quantity; seats shown on Billing.

### REQ-FOS-103 — Vendor app — P1
The vendor app supports staff sign-in, hides manager-only areas for florists, shows My shifts and lets a florist mark items Made with authorship (follow-up in the vendor app repo).
**Accept**: follow-up tracker raised.

**Decisions (§H, EP-58) — decided 2026-11-09**
1. Two roles for v1: Owner/Manager and Florist. Courier and cashier can come later as variants of Florist.
2. Florists see the whole team calendar, not only their own shifts (read-only).
3. Showing the florist's first name to customers is a per-shop option, off by default; each shop decides.
4. Yes: staff accounts count as seats in the SaaS plan, with a number included per plan and extra seats billed (REQ "Staff seats in the SaaS plan"). Several florists working at once in a large shop add real infrastructure load. Included seat counts and the extra-seat price are business settings to fill in before release.

---

## Suggested order (Polish peak calendar)

1. **Before Walentynki (14 Feb 2027) and Dzień Kobiet (8 Mar):** EP-43 Polish UI, EP-44 payouts, EP-45 push, EP-49 photo
   before dispatch, EP-47 gift details and catalogue, EP-50 KSeF invoices (the smallest sellers join KSeF on 1 Jan 2027).
2. **Before Dzień Matki (26 May):** EP-46 order book, EP-58 team/shifts/authorship, EP-48 slots/zones/pre-orders, EP-53 conversations,
   EP-57 profiles and SEO.
3. **Before Wszystkich Świętych (1 Nov):** EP-55 events/funerals/business, EP-54 reminders and repeat sales, EP-52
   purchasing, EP-51 couriers, EP-56 locations/analytics/audit.
