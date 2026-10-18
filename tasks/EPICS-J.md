# Epics, Stories & Tasks — Option J (Stock management)

> Companion to EPICS.md (EP-01–09), EPICS-B.md (EP-10–18), EPICS-C.md (EP-19–23),
> EPICS-D.md (EP-24), EPICS-E.md (EP-25), EPICS-F.md (EP-26), EPICS-G.md (EP-27),
> EPICS-H.md (EP-28) and EPICS-I.md (EP-29). Epic is numbered EP-30 to continue the sequence.
> Task tracker (ground truth): `docs/tasks/tasks-ep30-stock-management.json`.
> Requirements: `docs/tasks/REQ-GAPS-J.md` (REQ-STOCK-01 … REQ-STOCK-11).
> Mobile counterpart: vendor app repo, EP-M11 (`docs/tasks/tasks-ep-m11-stock.json`).
> Analysis date: 2026-10-18

---

## EP-30: Stock management

**Priority**: P1-high
**Dependencies**: EP-02 (vendor auth / tenant context), EP-12 (Vendor Portal), EP-28 (assistant —
shares the catalogue key and consumes stock prices); vendor mobile app EP-M11 for the app UI
**Requirement refs**: REQ-STOCK-01 through REQ-STOCK-11
**State**: in progress — backend, Vendor Portal and vendor app (EP-M11) implemented and tested; deployment (T-30-014) remains.

> One stock per shop. Every change leaves a line in the ledger.

### Concept

The assistant's stock is a throw-away snapshot of what is on the bench. EP-30 adds the shop's
**running stock**: items (flowers, greenery, materials) with quantity on hand, unit cost and a
low-stock threshold, changed only through movements — *received*, *used*, *wasted*, *counted* —
so the history explains every number.

```
 Delivery ──Received──▶ ┌────────────┐ ──Used──▶ bouquets (manual, or "deduct recipe" from the assistant)
                        │ StockItem  │ ──Wasted─▶ bin
 Stock-take ─Counted──▶ └────────────┘
                              │ unit cost
                              ▼
               AI Florist fills missing prices
```

### Design decisions

- **Counter + ledger**: `StockItem.Quantity` is the fast read; each change appends a
  `StockMovement` (delta, quantity after) in the same `SaveChanges`. The item row version serializes
  concurrent movements, so the ledger always adds up.
- **Same catalogue key as the assistant** (`name|colour`, trimmed, lower-case) — unique per vendor —
  so stock, assistant sessions and recipe deduction match without ids.
- **Whole units** like the assistant; flowers and greenery in stems.
- **Replay-safe**: movements accept `Idempotency-Key`, stored on the ledger row with a filtered
  unique index, on top of the response-cache `IdempotencyMiddleware`.
- **All-or-nothing batch consume** for recipes: no half-deducted bouquets.
- **Archive, never delete**: history stays; the key stays reserved and can be restored.
- **Independent of `BouquetDesign:Enabled`**: Stock works when the assistant is switched off.

### Explicitly NOT part of this epic

- Lots/expiry/FIFO valuation, supplier purchase orders, shopping lists, auto re-order.
- Automatic deduction on saleable bouquet creation or sale; low-stock push notifications.

### Stories & tasks

See `docs/tasks/tasks-ep30-stock-management.json` (S-30-01 … S-30-06; T-30-001 … T-30-014).

1. **S-30-01 — Trackers and contracts** — T-30-001 docs · T-30-002 API + outbound contracts.
2. **S-30-02 — Domain** — T-30-003 `StockItem` + `StockMovement` · T-30-004 domain tests.
3. **S-30-03 — Persistence** — T-30-005 EF config + repository · T-30-006 migration `AddStockManagement`.
4. **S-30-04 — Application + API** — T-30-007 commands/queries · T-30-008 `VendorStockController` ·
   T-30-009 assistant price fallback · T-30-010 flow tests.
5. **S-30-05 — Vendor Portal** — T-30-011 ApiClient + models · T-30-012 pages + navigation.
6. **S-30-06 — Vendor app and rollout** — T-30-013 EP-M11 (app repo) · T-30-014 deploy.

### API surface (all `VendorAccess`, under `/api/v1/vendor`)

`GET/POST stock`, `GET/PUT/DELETE stock/{id}`, `POST stock/{id}/restore`,
`POST stock/{id}/movements`, `GET stock/{id}/movements`, `GET stock/movements`, `POST stock/consume`.
Details: `docs/API-CONTRACTS.md` §22.

### Rollout

Deploy the API (migration `AddStockManagement` runs on start; additive tables only) → deploy the
Vendor Portal → ship the vendor app build with EP-M11. Rollback: redeploy the previous images; the
new tables are ignored by older code.
