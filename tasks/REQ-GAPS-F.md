# Requirement Gaps — Option F (Customer Portal Mobile Redesign)

> Companion to REQ-GAPS.md (Mobile API), REQ-GAPS-B.md (Portals/Analytics/IoT),
> REQ-GAPS-C.md (Vendor onboarding/subscription/vase) and REQ-GAPS-D.md (Showcase deployment).
> Requirement refs point at the tasks in `tasks-ep26-customer-portal-mobile-redesign.json`.
> Design: `docs/superpowers/specs/2026-07-18-customer-portal-mobile-redesign-design.md`.
> Mockups: `docs/assets/customer-portal-redesign/`.
> Analysis date: 2026-07-18

## Context

The `FlowerShop.CustomerApp` (ASP.NET MVC, live at `app.findmyflowers.pl`) is functional but its
mobile presentation is dated: a large opaque filter card covers most of the map, the palette clashes
(blue header + green button + grey body), and it reads as default Bootstrap. This epic restyles the
customer portal for mobile to a professional, Airbnb-style experience — **presentation only**: no API,
DTO, handler, migration, auth or routing changes, and every existing form field, JS handler, SignalR
hook (EP-08) and E2E `data-testid` (EP-16) is preserved.

Single brand accent: **rose `#E5326E`** (replaces the blue+green mix). Freshness pictograms (EP-25)
keep their green.

---

## Foundation

### REQ-CXUI-01 — Design-token & typography system — P1
A single set of CSS tokens (colour, radius, shadow, spacing) and a system-font type scale defined in
`wwwroot/css/site.css`, consumed by every customer view. Base font ≥ 15px on mobile; standardised
button / card / chip / status-badge / pill component classes.
**Accept**: tokens exist and are used; no view hardcodes the old blue/green primaries for chrome.

### REQ-CXUI-02 — Bottom tab navigation + slim header — P1
Fixed bottom tab bar on mobile (`≤ md`): Explore · Saved · Orders · Cart · Profile, active tab in
rose. Non-map pages get a slim back+title top bar. Desktop retains the top navbar. Cart badge count
still updates.
**Accept**: on a phone viewport the five tabs route correctly and the active tab is highlighted; cart
count still reflects `#cartBadge`.

## Explore / Map (screen 01–03)

### REQ-CXUI-03 — Floating search pill + filter chips — P1
Replace the map-covering filter card header with a floating rounded **search pill** (shows address +
"within N km") and a circular filter button badged with the active-filter count. Add a horizontally
scrolling **filter chip** row (radius, flower type, freshness, top-rated) over the map.
**Accept**: the map is fully visible behind the pill/chips; tapping the filter button opens the sheet
(REQ-CXUI-05). Mockup `01-explore-map.png`.

### REQ-CXUI-04 — Price-pill map markers — P1
Replace the green-bordered circular photo markers with white rounded **price-pill** markers that turn
dark when selected, matching the Airbnb pattern. Preserve `window.bouquetMarkerMap`, `markers[]` and
SignalR add/remove behaviour.
**Accept**: markers show price, selected state renders dark, real-time marker removal (EP-08) still
works. Mockup `01`/`03`.

### REQ-CXUI-05 — Filters bottom sheet — P1
Move all filters into a draggable **bottom sheet**: address search, "Use my location", radius chips
(replacing the `<select>`), flower-type chips, freshness chips (EP-25), sticky "Show N bouquets"
button. Keep the existing field ids/names and form-submit behaviour.
**Accept**: sheet opens/closes; radius chips drive the existing `radiusFilter` value; submitting
reloads results exactly as today. Mockup `02-filters-sheet.png`.

### REQ-CXUI-06 — Floating bouquet card — P1
Replace the Leaflet popup with an Airbnb-style **floating card**: large rounded photo, ♡ + ✕ overlay,
carousel dots, name, verified vendor, rating + distance, freshness flowers, price with strike-through
original, Details + Add-to-cart. Reuse `addBouquetToCart` and `renderFreshnessFlowers`.
**Accept**: selecting a marker shows the card; Add-to-cart updates the cart badge; "See reviews" modal
still works. Mockup `03-bouquet-card.png`.

## Bouquet details (screen 04)

### REQ-CXUI-07 — Bouquet details redesign — P2
Hero photo with carousel dots and back/♡ overlay, title + availability badge, rating, vendor card,
freshness row (EP-25), description, tag chips, and a **sticky add-to-cart bar** showing price.
**Accept**: details render for a real bouquet; sticky bar adds to cart. Mockup `04-bouquet-details.png`.

## Cart & checkout (screen 05–06)

### REQ-CXUI-08 — Cart redesign — P2
Line-item **cards** (thumbnail + rounded qty steppers), a summary card (subtotal / delivery / total),
and a sticky **Checkout · total** bar. Apply to both the cart page and the `_Layout` offcanvas.
**Accept**: quantity changes and totals behave as today. Mockup `05-cart.png`.

### REQ-CXUI-09 — Checkout redesign — P2
Sectioned cards: delivery address, payment-method rows (Card / BLIK, EP-18) with selection control,
order summary, sticky **Pay** bar. No change to the Stripe/BLIK flow.
**Accept**: checkout completes end-to-end unchanged. Mockup `06-checkout.png`.

## Account (screen 07–09)

### REQ-CXUI-10 — Favourites redesign — P3
Two-column **card grid**: photo, filled-heart toggle, freshness flowers, price.
**Accept**: unfavouriting removes the card as today. Mockup `07-favourites.png`.

### REQ-CXUI-11 — Orders redesign — P3
Order **cards**: number, colour-coded status badge, thumbnail, item summary, total; details page
restyled to match.
**Accept**: order list and details render for real orders. Mockup `08-orders.png`.

### REQ-CXUI-12 — Profile redesign — P3
Avatar header and grouped **settings cards** (personal details, saved addresses, payment methods,
alert subscriptions, order history), rose sign-out.
**Accept**: every profile link routes as today. Mockup `09-profile.png`.

## Cross-cutting

### REQ-CXUI-13 — Behaviour & test-hook preservation — P1
The redesign changes markup/CSS/light JS only. Preserve all form field ids/names, JS handlers,
`window.API_BASE` load order, SignalR marker logic, `renderFreshnessFlowers`, and E2E `data-testid`
selectors (`bouquet-list`, `bouquet-card`, cart badge/link). Re-add `data-testid`s on any relocated
control a Playwright spec targets.
**Accept**: the EP-16 Playwright suite passes with no change to test expectations; real-time updates
(EP-08) still function.
