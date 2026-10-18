# Epics, Stories & Tasks — Option K (Stock photos & AI appearance)

> Companion to EPICS-J.md (EP-30 Stock management) and EPICS-H.md (EP-28 AI Bouquet Assistant).
> Epic is numbered EP-31 to continue the sequence.
> Task tracker (ground truth): `docs/tasks/tasks-ep31-stock-photos-ai-appearance.json`.
> Requirements: `docs/tasks/REQ-GAPS-K.md` (REQ-STKPH-01 … REQ-STKPH-13).
> Mobile counterpart: vendor app repo, EP-M12 (`docs/tasks/tasks-ep-m12-stock-photos.json`).
> Analysis date: 2026-10-18

---

## EP-31: Stock photos & AI appearance recognition

**Priority**: P1-high
**Dependencies**: EP-30 (stock), EP-28 (private media store, image processing, provider plumbing,
usage gate and $ cap); vendor app EP-M12 for the app UI
**Requirement refs**: REQ-STKPH-01 through REQ-STKPH-13
**State**: in progress — implemented and tested in both repos (backend, Vendor Portal, vendor app EP-M12); deploy (T-31-015) open.

> A colour word is a guess. A photo is the truth; the AI turns it into words the florist confirms.

### Concept

```
 Photo ──upload──▶ private store ──describe (AI, quota)──▶ suggestion ──florist applies──▶ appearance
                                                                                             │
   variety + colour label + palette swatches + pattern + description  ◀──────────────────────┘
        │
        ├──▶ assistant composer: real shades for recipes
        ├──▶ assistant previews: stock photos as reference images
        └──▶ assistant recognition: bench photos matched to stock items
```

### Design decisions

- **Variety joins the identity**: key = name|colour[|variety]; items without a variety keep their
  existing key, so EP-30 data needs no key migration.
- **Suggest, never overwrite**: AI output is a suggestion; only an explicit apply writes it, and the
  appearance records `source = Ai`.
- **Synchronous describe with a ledger**: one photo, one short call. A ledger row is the reservation
  (inserted under the assistant's advisory lock), so quota and the shared $ cap stay exact; an
  `Idempotency-Key` replay returns the stored suggestion.
- **Free by default**: the `Palette` provider measures dominant colours with SkiaSharp — real
  swatches with no AI spend; `OpenAI` adds variety/name and a richer description.
- **Same private storage as the assistant** (`stock/{vendorId}/…` keys), same image pipeline.
- **Backwards-compatible updates**: `variety`/`appearance` omitted from a PUT are left untouched.

### Stories & tasks

See `docs/tasks/tasks-ep31-stock-photos-ai-appearance.json` (S-31-01 … S-31-07; T-31-001 … T-31-015).

1. **S-31-01 — Trackers and contracts** — T-31-001 docs · T-31-002 contracts.
2. **S-31-02 — Variety and appearance** — T-31-003 domain · T-31-004 commands/API.
3. **S-31-03 — Item photos** — T-31-005 entity/migration/storage · T-31-006 endpoints, pending photos, retention.
4. **S-31-04 — AI appearance recognition** — T-31-007 ledger/quota · T-31-008 providers · T-31-009 tests.
5. **S-31-05 — Assistant integration** — T-31-010 stock-linked lines · T-31-011 previews + composition · T-31-012 bench matching.
6. **S-31-06 — Vendor Portal UI** — T-31-013.
7. **S-31-07 — Vendor app and rollout** — T-31-014 EP-M12 · T-31-015 deploy.

### Rollout

Deploy the API (migration runs on start) → portal → app. Stock recognition uses the free `Palette`
provider until `BouquetDesign__StockRecognition__Provider=OpenAI` (with `BouquetDesign__OpenAI__ApiKey`).
Kill switch: `BouquetDesign__Enabled=false` stops AI describe; photos keep working.
