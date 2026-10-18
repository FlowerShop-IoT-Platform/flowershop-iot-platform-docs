# Epics, Stories & Tasks — Option L (AI Florist wizard in the Vendor Portal)

> Companion to EPICS-H.md (EP-28 AI Florist) and EPICS-K.md (EP-31 Stock photos & AI appearance).
> Epic is numbered EP-32 to continue the sequence.
> Task tracker (ground truth): `docs/tasks/tasks-ep32-vendor-portal-ai-florist.json`.
> Requirements: `docs/tasks/REQ-GAPS-L.md` (REQ-WIZ-01 … REQ-WIZ-10).
> Mobile counterpart (already shipped): vendor app EP-M10 "Bouquet assistant".
> Analysis date: 2026-10-18

---

## EP-32: AI Florist wizard in the Vendor Portal

**Priority**: P1-high
**Dependencies**: EP-28 (AI Florist API), EP-30/31 (stock, consume, stock photos)
**Requirement refs**: REQ-WIZ-01 through REQ-WIZ-10
**State**: in progress — implemented and tested; deploy (T-32-009) open.

> The same four steps as the phone, on the screen where the shop already manages its stock.

### Concept

```
/Design (hub: allowance, start, recent, saved, cost settings)
   │ Start from photos            │ Start from my stock
   ▼                              ▼
1 Photos ─Recognise─▶ (polling) ─▶ 2 Check stock ◀─ Fill from my stock
                                     │ Confirm
                                     ▼
                                  3 Design (style, cost ceiling) ─Create─▶ (polling)
                                     ▼
                                  4 Recipes: preview · save · margin · copy · use stock
```

### Design decisions

- **Client only**: no API change — the portal calls §21 like the vendor app does and §22 consume.
- **Server-rendered steps, light JS**: each action is a form post (works without JS); a small script
  polls the page's `Status` handler and reloads when the session version changes.
- **Step from state**: the page derives the step from the session state; "Edit stock" / "New recipes"
  are explicit `?step=` links, allowed only where the API would accept the action.
- **Idempotency on the web**: each billable form carries a one-time key; when the API was
  unreachable the re-rendered form keeps the same key, so a retry can never be billed twice.
- **Private media via proxy**: photos and previews go through `/Design/Media/…` with the session
  token, `Cache-Control: private, no-store`.
- **Testable rules**: `DesignWizard` is pure and linked into `FlowerShop.Domain.Tests`.

### Stories & tasks

See `docs/tasks/tasks-ep32-vendor-portal-ai-florist.json` (S-32-01 … S-32-04; T-32-001 … T-32-009).

1. **S-32-01 — Trackers** — T-32-001.
2. **S-32-02 — Portal API client and wizard rules** — T-32-002 client · T-32-003 rules · T-32-004 consume.
3. **S-32-03 — Wizard pages** — T-32-005 hub/nav · T-32-006 session wizard · T-32-007 saved, cost, media.
4. **S-32-04 — Tests and rollout** — T-32-008 tests · T-32-009 deploy.

### Rollout

Merge → Vendor Portal deploy. Uses whatever providers the API is configured with
(`BouquetDesign__*__Provider`); the kill switch `BouquetDesign__Enabled=false` hides the start buttons.
