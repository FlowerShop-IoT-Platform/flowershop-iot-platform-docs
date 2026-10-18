# Requirement Gaps — Option J (Stock management)

> Companion to REQ-GAPS.md (Mobile API), REQ-GAPS-B.md (Portals/Analytics/IoT),
> REQ-GAPS-C.md (Vendor onboarding/subscription/vase), REQ-GAPS-D.md (Showcase deployment),
> REQ-GAPS-F.md (Customer portal mobile redesign), REQ-GAPS-G.md (Contact us),
> REQ-GAPS-H.md (AI Florist) and REQ-GAPS-I.md (Phone verification abuse protection).
> Requirement refs point at the tasks in `tasks-ep30-stock-management.json`; the mobile side is
> tracked in the vendor app repo (`docs/tasks/tasks-ep-m11-stock.json`).
> Analysis date: 2026-10-18
> Status: REQ-STOCK-01 … 11 implemented (EP-30, see `EPICS-J.md`); production deploy pending.

## Context

The AI Florist (EP-28) introduced "stock" — but only as a **per-session snapshot**: the
florist photographs or types what is on the bench, confirms it, and the snapshot dies with the
session (unit prices are remembered in `DesignCostProfile`, quantities are not). EP-28 explicitly
left "automatic stock subtraction" out of scope.

Vendors have no place to keep a running stock of flowers, greenery and materials: what arrived from
the wholesaler, what went into bouquets, what wilted, what is running low and what the stock is
worth. They want the same feature on the desktop (Vendor Portal) and on the phone (vendor app).

Goal: a vendor-owned, tenant-isolated **Stock** with an auditable movement ledger, usable from both
apps, and wired into the assistant (stock feeds the assistant; a recipe that was made can be
deducted from stock).

---

## REQ-STOCK-01 — Vendor stock catalogue — P0
A `StockItem` belongs to one vendor (`VendorId` + `VendorType`): kind (Flower / Greenery /
Material), name, optional colour, unit (flowers and greenery are counted in **stems**, materials in
stems or **items**), quantity on hand (whole units, ≥ 0), optional unit cost (PLN), optional
low-stock threshold, optional supplier and notes. Name + colour (trimmed, case-insensitive) is the
**catalogue key** — the same key the assistant uses — and is unique per vendor.
**Accept**: duplicate key → 409 `duplicate-item`; invalid fields → 400 `invalid-request`; at most
500 items per vendor; another vendor's item answers 404.

## REQ-STOCK-02 — Stock movements ledger — P0
Every quantity change is a `StockMovement` row: `Received` (+, optional new unit cost),
`Used` (−, went into bouquets), `Wasted` (−, wilted or damaged), `Counted` (stock-take: sets the
absolute count, delta may be ±), and `Initial` (opening quantity on create). Each row stores delta,
quantity after, unit cost (receipts), note, reference, acting user and time. Quantity never goes
below zero.
**Accept**: using/wasting more than on hand → 422 `insufficient-stock`, nothing changes; a receipt
with a unit cost updates the item's unit cost and `lastReceivedAt`; the ledger reproduces the
current quantity.

## REQ-STOCK-03 — Safe concurrent edits and replay-safe movements — P0
Detail edits carry `expectedVersion` (stale → 409 `version-conflict`). Concurrent movements on one
item are serialized by the row version (loser gets 409 and retries). A movement or batch sent with
an `Idempotency-Key` is applied once; a replay returns the current state.
**Accept**: two edits from the same version → second 409; same key twice → one ledger row.

## REQ-STOCK-04 — Overview, filters and low-stock status — P0
`GET /vendor/stock` returns items plus a summary: item count, low-stock count (quantity ≤
threshold, > 0), out-of-stock count, stock value (Σ quantity × unit cost over priced items) and
unpriced count. Filters: search (name/colour/supplier), kind, status (`low`, `out`), archived.
**Accept**: each item reports `status` = `InStock` | `Low` | `Out`; value ignores unpriced items
and says how many were ignored (never counts them as zero silently).

## REQ-STOCK-05 — Archive and restore — P1
Removing an item archives it (hidden by default, history kept, key stays reserved); restore brings
it back. Archived items accept no movements.
**Accept**: movement on an archived item → 409 `archived`; re-adding an archived key → 409 that
points at restore.

## REQ-STOCK-06 — Movement history — P1
Per-item history (`GET /vendor/stock/{id}/movements`) and vendor-wide recent activity
(`GET /vendor/stock/movements`), newest first, paged (max 100).

## REQ-STOCK-07 — Batch consumption (deduct a recipe) — P0
`POST /vendor/stock/consume` deducts several lines in one all-or-nothing step, reason `Used` or
`Wasted`, with a note and reference (e.g. `design-recipe:{id}`). A line names a `stockItemId` or a
name + colour, matched by catalogue key; duplicates are summed first.
**Accept**: any unknown line → 422 `unknown-item`, any short line → 422 `insufficient-stock`; the
message names every problem line; on failure nothing is deducted.

## REQ-STOCK-08 — Stock feeds the AI Florist — P1
When the assistant fills a missing unit price (manual save or recognition), a matching stock item's
unit cost is used first, then the remembered assistant price. The vendor app can import current
stock (quantity > 0) into a session's inventory, and deduct a made recipe from stock (REQ-STOCK-07).
**Accept**: a session line "Rose / Pink" without price gets the stock unit cost; the assistant still
works when the vendor has no stock.

## REQ-STOCK-09 — Vendor Portal stock screens — P0
Portal (desktop + its mobile layout): Stock list with summary tiles, search/kind/status filters and
low/out badges; add item; item page with details edit, quick movements (receive / use / waste /
count) and history; archive/restore; vendor-wide activity. Linked from the sidebar and the More menu.

## REQ-STOCK-10 — Vendor app stock screens — P0
Vendor app (EP-M11): Stock list with summary + filters, item detail with movement buttons and
history, add/edit, reachable from More and a Home quick action; assistant integration from
REQ-STOCK-08. Errors use the shared ProblemDetails mapping; every async command catches timeouts.

## REQ-STOCK-11 — Contracts and docs — P1
`API-CONTRACTS.md` gets a Stock section; `docs/contracts/outbound.md` lists the new endpoints; both
repos' `CLAUDE.md` mention the feature.

---

## Explicitly out of scope

- Batches/lots with per-delivery expiry and FIFO valuation (cost is the latest receipt cost).
- Supplier purchase orders, shopping lists (EP-28 board screen 06) and automatic re-ordering.
- Automatic deduction when a saleable `Bouquet` is created or sold.
- Push notifications on low stock (push is still a stub, EP-06).
