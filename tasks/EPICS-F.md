# Epics, Stories & Tasks — Option F (Customer Portal Mobile Redesign)

> Companion to EPICS.md (EP-01–09), EPICS-B.md (EP-10–18), EPICS-C.md (EP-19–23),
> EPICS-D.md (EP-24) and EPICS-E.md (EP-25).
> Epic is numbered EP-26 to continue the sequence.
> Task tracker (ground truth): `docs/tasks/tasks-ep26-customer-portal-mobile-redesign.json`.
> Requirements: `docs/tasks/REQ-GAPS-F.md` (REQ-CXUI-01 … REQ-CXUI-13).
> Design: `docs/superpowers/specs/2026-07-18-customer-portal-mobile-redesign-design.md`.
> Mockups: `docs/assets/customer-portal-redesign/`.
> Analysis date: 2026-07-18

---

## EP-26: Customer Portal Mobile Redesign

**Priority**: P1-high
**Dependencies**: EP-13/EP-14 (Customer Web App), EP-08 (real-time markers), EP-16 (E2E Playwright),
EP-18 (card/BLIK checkout), EP-25 (freshness pictograms)
**Requirement refs**: REQ-CXUI-01 through REQ-CXUI-13
**State**: in-progress — design system, bottom nav and map redesign implemented; content pages
restyled via the shared CSS; `dotnet build` + EP-16 Playwright re-run pending a .NET environment
(T-26-013).

> Give the customer portal a professional, Airbnb-style mobile experience without changing any
> behaviour, contract or data.

### Concept

The `FlowerShop.CustomerApp` works but looks dated on mobile — a large opaque filter card hides most
of the map, the palette clashes (blue header + green button + grey body), and it reads as default
Bootstrap. This epic restyles the whole customer portal to a **map-first, app-like** design in the
spirit of Airbnb: a floating **search pill**, horizontal **filter chips**, white **price-pill**
markers, a draggable **filter bottom sheet**, a **floating bouquet card**, and one consistent
card/typography **design system** with a **bottom tab bar** across every screen.

It is deliberately **presentation-only**: no API, DTO, handler, migration, auth or routing change.
Every existing form field, JS handler, SignalR real-time hook (EP-08) and E2E `data-testid` (EP-16)
is preserved.

### Design decisions

- **Scope**: the full customer portal (map, details, cart, checkout, favourites, orders, profile) —
  it only reads as one app if every screen shares the system.
- **Accent**: a single **rose `#E5326E`** replaces the blue+green mix. Freshness flower pictograms
  (EP-25) keep their green `#2e7d5b`.
- **Approach**: visual/layout redesign that **preserves all behaviour**.

### Explicitly NOT part of this epic

- No API / DTO / handler / migration changes; read models are consumed as-is.
- No new features, filters, sort orders, payment methods or screens.
- No Leaflet / map-provider swap; no auth or route changes; not a native (MAUI/RN) app.

### Screens & mockups

Rendered phone mockups live in `docs/assets/customer-portal-redesign/` (source `mockups.html`):

| # | Screen | Mockup |
|---|--------|--------|
| 00 | Current state (before) | `00-current-state.jpg` |
| 01 | Explore / Map | `01-explore-map.png` |
| 02 | Filters (bottom sheet) | `02-filters-sheet.png` |
| 03 | Bouquet card (map) | `03-bouquet-card.png` |
| 04 | Bouquet details | `04-bouquet-details.png` |
| 05 | Cart | `05-cart.png` |
| 06 | Checkout | `06-checkout.png` |
| 07 | Saved / Favourites | `07-favourites.png` |
| 08 | Orders | `08-orders.png` |
| 09 | Profile | `09-profile.png` |

### Stories & tasks

See `docs/tasks/tasks-ep26-customer-portal-mobile-redesign.json` (S-26-01 … S-26-05; T-26-001 …
T-26-013) for the authoritative breakdown. Summary:

1. **S-26-01 — Design system foundation & shared chrome**
   - T-26-001 — tokens, type scale and component classes in `site.css` (rose accent).
   - T-26-002 — mobile bottom tab bar + slim header in `_Layout.cshtml`.
2. **S-26-02 — Explore / Map redesign**
   - T-26-003 — floating search pill + filter chips.
   - T-26-004 — price-pill markers (replace green circles), SignalR-safe.
   - T-26-005 — filter bottom sheet (radius chips replace the `<select>`).
   - T-26-006 — floating bouquet card (replace the Leaflet popup).
3. **S-26-03 — Details, cart & checkout**
   - T-26-007 — bouquet details with sticky add-to-cart bar.
   - T-26-008 — cart (page + offcanvas).
   - T-26-009 — checkout (Card / BLIK).
4. **S-26-04 — Account screens**
   - T-26-010 — favourites grid.
   - T-26-011 — orders list & details.
   - T-26-012 — profile & addresses.
5. **S-26-05 — Behaviour preservation**
   - T-26-013 — preserve field ids / handlers / real-time hooks and re-run the EP-16 Playwright suite.

### Rollout

Deploy-only (no migration, no server change). Ship the Customer App image; the API is unchanged.
Verify on a real mobile viewport (`verify` skill / Playwright) and re-run the EP-16 suite before
release.
