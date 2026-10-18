# Requirement Gaps — Option L (AI Florist wizard in the Vendor Portal)

> Companion to REQ-GAPS-H.md (AI Florist, EP-28) and REQ-GAPS-K.md (Stock photos & AI appearance, EP-31).
> Requirement refs point at the tasks in `tasks-ep32-vendor-portal-ai-florist.json`.
> Analysis date: 2026-10-18
> Status: REQ-WIZ-01 … 10 implemented (EP-32, see `EPICS-L.md`); production deploy pending.

## Context

The AI Florist (EP-28) shipped with a backend API (`/api/v1/vendor/design-*`) and one client: the
vendor mobile app's "Bouquet assistant" (EP-M10). Vendors who manage their shop from a desktop — the
Vendor Portal already has Stock (EP-30) with photos and AI appearance (EP-31) — have no way to start
a design there. With real OpenAI providers switched on in production (2026-10-18) the feature is
ready for pilot use, so the portal gets the same flow, named **AI Florist wizard**.

No API or domain change: the portal is a new client of the existing contract (`API-CONTRACTS.md` §21)
and of the stock consume endpoint (§22).

---

## REQ-WIZ-01 — Entry points — P0
A sidebar item **AI Florist wizard** (`/Design`), an entry in the mobile-width "More" menu, a button
on the Stock page and a dashboard quick action. The hub shows this month's AI allowance
(recognitions, recipe sets, previews, reset date), "Start from photos", "Start from my stock",
recent designs (continue / delete) and links to saved recipes and cost settings.
**Accept**: when the assistant is disabled for the shop (`enabled=false`) the hub says so and the
start buttons are hidden; saved recipes stay reachable.

## REQ-WIZ-02 — Step 1: stock photos — P0
Upload 1–`maxPhotos` photos (multi-file input, JPEG/PNG/WebP), see thumbnails through a same-origin
proxy (the browser cannot send the API token), delete one, then **Recognise stock**. "Enter stock
by hand instead" skips recognition.
**Accept**: 6th photo / invalid file shows the server's message; thumbnails never come from a public URL.

## REQ-WIZ-03 — Working states and polling — P0
While the session is `Recognizing`/`Composing` or a preview is `Queued`/`Running`, the page polls a
portal status handler: every 2 s for the first 60 s, then every 5 s, paused while the tab is hidden;
it reloads when the session version changes. After 120 s it shows "Still working" (the vendor can leave
and come back). Polling never re-posts a billable start.
**Accept**: closing the tab mid-job and reopening the design resumes the right step.

## REQ-WIZ-04 — Step 2: check your stock — P0
Editable table (kind, name, colour, variety, quantity, unit, unit price) with estimates flagged
("Counts are estimates"), add row, remove row, and **Fill from my stock**: stock items with
something on hand are merged in (same `stockItemId`, or same name|colour|variety for hand-typed rows —
quantity updated, price filled only when empty, `stockItemId` linked); nothing is saved until
**Confirm stock**. Prices accept `4,50` and `4.50`. Unchanged, already confirmed stock is not re-sent
(re-sending would drop unsaved recipes).
**Accept**: a version conflict keeps the vendor's rows and asks them to confirm again; a bad price names the row.

## REQ-WIZ-05 — Step 3: choose your design — P0
Style (from `usage.styles`, or "No preference") and an optional cost ceiling; size M, available stock
only, 3 recipes (first release, as on mobile). **Create recipes** sends a one-time `Idempotency-Key`
(reused when the first answer was lost).
**Accept**: 422 `insufficient-stock` / `cost-profile-incomplete` and 429 quota answers show the
server's `detail` (plus "Try again in …" from `Retry-After`).

## REQ-WIZ-06 — Step 4: recipes — P0
Recipe cards: title, description, palette, stem summary, lines, assembly instructions, cost
("Known cost … · N prices missing" when incomplete — never a made-up total) and the AI disclaimer.
Per recipe: **Generate preview** (state shown; failed/unknown outcomes can be retried with a new key),
**Save / Unsave**, **Margin** for a selling price (server-computed), **Copy recipe** (plain text) and
**Use stock** (all-or-nothing deduction via `POST /vendor/stock/consume`, reason `Used`, reference
`design-recipe:{id}`, confirmed first). "Edit stock" and "New recipes" go back a step.
**Accept**: previews render through the same-origin proxy; a recipe that no longer matches the stock is flagged.

## REQ-WIZ-07 — Saved recipes — P1
`/Design/Saved` lists saved recipes across designs (newest first) with cost, preview and a link to
the design.

## REQ-WIZ-08 — Cost settings — P1
`/Design/CostProfile`: basis (Gross/Net), labour minutes (0–1440), hourly rate, overhead % (0–100),
and the remembered prices list. Version conflict keeps the edits.

## REQ-WIZ-09 — Errors and tenant safety — P0
All calls carry the vendor's session token; the tenant comes from the JWT. ProblemDetails `detail` is
shown as-is; 404 → "This design is no longer available"; transport failure → "We couldn't reach the
server". Media proxy returns 404 for anything the API refuses.

## REQ-WIZ-10 — Tests and docs — P1
Unit tests for the pure wizard rules (step resolution, polling need, cost text, amount parsing,
stock merge, change detection); a Node test for the polling script; `CLAUDE.md` updated.

## Explicitly out of scope
- API/domain changes; new sizes or modes (the API still only offers M / AvailableOnly).
- Turning a recipe into a sellable bouquet listing (follow-up).
- Live push (SignalR) instead of polling.
