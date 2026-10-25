# Requirement Gaps — Option T (Online-only vendors: a vase-free vendor UI)

> Companion to REQ-GAPS-Q.md (EP-39 online-only vendors, REQ-ONL-15/17 first hid the main vase entry points) and
> REQ-GAPS-C.md (EP-21 vase ordering, EP-22 asset recovery).
> Requirement refs point at the tasks in `tasks-ep42-online-only-vase-free-ui.json`.
> Analysis date: 2026-10-25
> Status: proposed (EP-42, see `EPICS-T.md`). Vendor app counterpart EP-M22. Section H open.

## Context

An online-only vendor (`SalesMode = OnlineOnly`, EP-39) has no smart vases and can't have any while that mode is set
(`vendor-has-vases`). Every vase screen, tile, prompt, link, script and sentence is therefore dead weight for them.
EP-39 hid the main entry points (nav "My Vases", More menu vase items, dashboard vase tiles, `Vases/*` redirect via
`OnlineOnlyVasesFilter`, "Add a bouquet" → `Bouquets/Create`). A sweep of the Vendor Portal and the vendor app on
2026-10-25 found what is still left, and one design gap that shows the whole vase UI to an online-only vendor.

**Goal:** a vendor who registers as online-only, and any vendor who switches to it, never sees smart-vase
functionality in the Vendor Portal or the vendor app. Switching back to `InStore` brings all of it back unchanged.
In-store vendors see no change.

### What is still visible today (Vendor Portal)

| # | Surface | Where |
|---|---|---|
| 1 | **Fail-open sales mode.** `SalesModeService.GetAsync` returns `InStore` when the API call fails or returns nothing, and it is only read on the first page render. A brief API error, or the first pages after a new online-only sign-up, show the full vase portal: nav, dashboard vase tiles, "Order smart vase" prompt. | `Services/SalesModeService.cs` |
| 2 | Analytics "Online Vases … of N registered" tile, plus a `GetMyVasesAsync` call on every load, not gated. | `Pages/Analytics/Index.cshtml:55`, `Index.cshtml.cs:38` |
| 3 | Bouquets page renders the vase-picker modal `#createBouquetModal` ("Select a vase for your bouquet", "You don't have any vases yet. Go to My Vases") for every vendor. The online-only view never opens it, but it is in the page, and its data may be fetched. | `Pages/Bouquets/Index.cshtml:395-450`, `Index.cshtml.cs` |
| 4 | Cost assessor: "To list it, place the bouquet in a vase on My bouquets…" is shown to online-only vendors. | `Pages/Design/Assess/Details.cshtml:240` |
| 5 | `_Layout` loads `vase-updates.js` (SignalR `VaseStatusChanged` toasts) and the device-heartbeat script for every vendor. | `Pages/Shared/_Layout.cshtml:126`, `wwwroot/js/business.js` |
| 6 | Help: `DocsHelpTopics` maps vase pages to manual sections s5/s6. The vendor portal manual's vase chapters are not marked "in-store only", and there is no online-only quick start. | `Services/DocsHelpTopics.cs`, `docs/manuals/vendor-portal-manual.html` |
| 7 | Copy: registration success, onboarding banners, verification-pending and empty states are written for vase vendors. Each needs checking for vase wording. | `Pages/Account/*`, `Pages/Index.cshtml`, `Pages/Bouquets/*` |

### What is still visible today (vendor app, EP-M22)

| # | Surface | Where (vendor app repo) |
|---|---|---|
| 1 | `ISalesModeService.Current` is `InStore` until it is read. The first Home render after sign-in or sign-up can flash the vase tile and "Manage Vases" before the mode arrives. A failed read keeps them shown. | `Services/SalesModeService.cs` |
| 2 | Off-tab vase routes (`vases`, `vasedetail`, `vaseupload`, `createbouquet`, `qrscanner`, `vaseorders`, `vaseorders/place`) have no guard. A notification tap, deep link or back stack can still open them. | `AppShell.xaml.cs` |
| 3 | Analytics "Active Vases" tile, not gated. | `Views/AnalyticsPage.xaml:61` |
| 4 | Alerts empty state: "…when your vases go offline… or new vases are assigned." | `Views/AlertsPage.xaml:125` |
| 5 | The bouquet map calls `GetMyVasesAsync` for online-only vendors. | `ViewModels/BouquetMapViewModel.cs:78` |
| 6 | Help topics for the vase screens. | `HelpTopics` |

### What the API does today

The API never refuses vase actions for an online-only vendor. `POST /vendor/vase-orders` (EP-21), placing a bouquet in
a vase and vase photo upload check tenant ownership but not `SalesMode`, so a stale browser tab or an old app build
can still order vases. Read endpoints (`GET /smartvases/my-vases`) return an empty list, which is correct.

### Approach

1. **Know the mode before the first page.** The portal reads the sales mode when the vendor signs in, and after
   registration. The app reads it the same way and remembers it per vendor. "Unknown" is a state of its own: vase
   entry points are hidden until the mode is known, and nothing is redirected while it is unknown.
2. **One rule, every surface.** Every vase surface asks the same rule (`SalesModeView.ShowVases` in the portal,
   `SalesModes` in the app). A static test keeps new vase links and copy from slipping in ungated.
3. **The server agrees with the UI.** Vase-only write actions are refused for online-only vendors with one error
   code, so a hidden feature can't be reached by other means.

Out of scope: the Admin Portal (admins keep their vase and fleet views; an online-only vendor simply has no vases),
customer surfaces (done in EP-39), and the Hybrid mode (REQ-GAPS-Q §H.1, not in v1).

---

## A. Sales mode known up front

### REQ-NOV-01 — Read the sales mode at sign-in, and an "unknown" state — P0
The Vendor Portal stores the vendor's sales mode in the session as part of signing in: the Login handler, the docs or
handoff return, and the end of the registration wizard (whose request already carries `salesMode`).
`ISalesModeService` returns `InStore`, `OnlineOnly` or `null` (unknown, e.g. the API failed) and retries on the next
request while unknown. `SalesModeView` gains `ShowVaseEntryPoints(mode)`: true only for a known `InStore`. Nav, the
More menu, dashboard vase tiles and vase-ordering prompts use it, so unknown hides them (proposed, §H.1).
`OnlineOnlyVasesFilter` keeps redirecting only for a known `OnlineOnly`. On unknown, a vase page that was opened
directly still renders, because an in-store vendor must not be locked out. Switching mode in Settings already updates
the session (`Remember`) and keeps doing so.
**Accept**: unit tests for the three states. A newly registered online-only vendor's first portal page shows no vase
UI. With the sales-mode API down, no vase nav or prompt is shown and `/Vases` still opens for an in-store vendor.

### REQ-NOV-02 — One documented rule for vase surfaces — P1
`SalesModeView` documents the full list of vase surfaces (nav, More, dashboard, analytics, Bouquets create modal,
assessor wording, real-time scripts, Help topics, `Vases/*`). Every view checks `ShowVases` / `ShowVaseEntryPoints`
instead of an inline sales-mode comparison.
**Accept**: no `.cshtml` compares the sales mode string directly (covered by REQ-NOV-11).

---

## B. Vendor Portal sweep

### REQ-NOV-03 — Analytics without vases — P0
For online-only vendors, `Analytics/Index` hides the "Online Vases" tile and skips `GetMyVasesAsync`. The space shows
online listing metrics the page already has (active listings, sold, expired), or the tile row closes up.
**Accept**: page-model test: online-only, no vase call, no vase tile; in-store unchanged.

### REQ-NOV-04 — Bouquets page without the vase picker — P1
For online-only vendors, the `#createBouquetModal` vase picker and its "Go to My Vases" link are not rendered, and the
page model does not load vases. Empty states and help text say "Add online bouquet" (EP-39 wording already used in
the empty state).
**Accept**: page-model test: online-only, `Vases` not loaded, modal partial not rendered.

### REQ-NOV-05 — AI Florist and cost assessor wording — P1
Where a flow tells the vendor to "place the bouquet in a vase" (cost assessor `Details`, and anywhere else the sweep
finds it), online-only vendors instead get "List it online", linking to `Bouquets/Create` with the price prefilled
(the existing `SalesModeView.ListOnlinePrefill` rule, or a sibling for an assessment).
**Accept**: unit test of the prefill. The cost assessor shows the link for online-only and the vase sentence for
in-store.

### REQ-NOV-06 — No vase real-time scripts — P2
`_Layout` loads `vase-updates.js` and the device-heartbeat script only when `ShowVases`. Customer-request real-time
updates (`MarketplaceHub`, EP-37) are unaffected.
**Accept**: layout test or a rendered-HTML assertion for both modes.

### REQ-NOV-07 — Help and manual — P2
`DocsHelpTopics` never sends an online-only vendor to a vase section. The vendor portal manual (and the vendor app
manual, EP-M22) marks the vase chapters "In-store vendors (smart vases)" and gets an "Selling online only" quick start
(listing, delivery, expiry) near the top, which the Help menu opens for online-only vendors.
**Accept**: unit test of the topic mapping per mode. `scripts/docs/check_audiences.py` passes.

### REQ-NOV-08 — Copy sweep — P1
Review vendor-facing copy for online-only vendors for vase wording: registration success and verification pages,
onboarding and approval banners, dashboard and list empty states, Billing/plan text and vendor emails from the API
(`IEmailService` templates). Each mention is gated by the REQ-NOV-02 rule or rewritten to be mode-neutral.
**Accept**: the REQ-NOV-11 scan finds no ungated vase wording; reviewed list noted in the task comments.

---

## C. API

### REQ-NOV-09 — Refuse vase-only actions for online-only vendors — P1
Vase-only **write** actions fail for an `OnlineOnly` vendor with HTTP 409 and code `vendor-online-only`, in the same
`{ error, message }` shape the EP-39 sales-mode endpoints use: placing a vase order (`POST /vendor/vase-orders`),
creating or placing a bouquet in a vase (`POST /bouquets` with a vase, `POST /smartvases/{id}/bouquet-photo`), and
vendor-side vase assignment or QR linking. The check is one shared rule in the Application layer
(`SalesModeRules.EnsureVasesAllowed`). Reads are unchanged: an online-only vendor has no vases, so they return empty.
EP-22 return requests for vases assigned before a switch can't happen, because the switch is refused while vases are
assigned. Admin endpoints are not affected.
**Accept**: handler tests per action for both modes; contract §25 lists the new code.

---

## D. Mobile

### REQ-NOV-10 — Vendor mobile app (EP-M22) — P0
In the vendor app repo: the mode read at sign-in and sign-up and remembered per vendor, with unknown hiding vase entry
points (REQ-NOV-01); a Shell navigation guard that sends online-only vendors away from the vase routes with a message
(like `OnlineOnlyVasesFilter`); the Analytics, Alerts, bouquet map and Help sweep; `vendor-online-only` mapped to a
friendly message; a static guard test; a device run with a newly registered online-only vendor. The API is ready
after S-42-04.

---

## E. Quality

### REQ-NOV-11 — Regression guard — P1
A test in `FlowerShop.Domain.Tests`, next to the `SalesModeView` tests, scans
`src/portals/FlowerShop.VendorPortal/Pages/**/*.cshtml` for vase links (`/Vases`, `asp-page="/Vases`), vase scripts
and the words "vase"/"vases". It requires each hit to be in an allow-listed file that is gated: `Pages/Vases/*`,
gated blocks marked with a `@* vase-surface *@` comment, or Settings/registration texts that explain the modes. A new
ungated vase surface fails the build. Page-model tests cover both modes for Index, Analytics, Bouquets, More and
Assess/Details.
**Accept**: the scan passes on the swept portal and fails on a planted ungated `/Vases` link.

### REQ-NOV-12 — Contract, manuals and changelog — P2
`docs/API-CONTRACTS.md` §25 gets `vendor-online-only` and the list of refused actions. The vendor portal manual gets
the online-only quick start (REQ-NOV-07). The changelog gets an entry.

---

## H. Open decisions (proposed, not yet decided)

1. **Unknown sales mode.** When the mode can't be read, should vase entry points be hidden (proposed: fail closed for
   nav and prompts, but no redirects) or shown as today (fail open)? Hiding them can briefly take "My Vases" out of
   the nav for an in-store vendor during an API error. Showing them pollutes an online-only vendor's UI, which this
   epic is about.
2. **Sales mode as a JWT claim.** Should Keycloak carry a `sales_mode` claim, so no extra API call is needed? *Proposed:
   no.* The mode can change in the middle of a session, a claim would be stale until the token refreshes, and it would
   add a Keycloak attribute write to the sales-mode switch. The session/API read (REQ-NOV-01) is enough.
3. **Switching back to InStore.** Proposed: every vase surface comes back at once (the session is updated on switch),
   and the dashboard shows "Order your first smart vase" as the next step.
4. **Server-side refusal.** Is REQ-NOV-09 in scope for this epic (proposed: yes, P1), or UI-only?
