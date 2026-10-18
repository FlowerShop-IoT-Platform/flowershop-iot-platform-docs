# Epics, Stories & Tasks — Option H (AI Florist)

> Companion to EPICS.md (EP-01–09), EPICS-B.md (EP-10–18), EPICS-C.md (EP-19–23),
> EPICS-D.md (EP-24), EPICS-E.md (EP-25), EPICS-F.md (EP-26) and EPICS-G.md (EP-27).
> Epic is numbered EP-28 to continue the sequence.
> Task tracker (ground truth): `docs/tasks/tasks-ep28-ai-bouquet-assistant.json`.
> Requirements: `docs/tasks/REQ-GAPS-H.md` (REQ-AIB-01 … REQ-AIB-19).
> Design: `docs/superpowers/specs/2026-10-13-ai-bouquet-assistant-design.md`.
> Mobile counterpart: vendor app repo, EP-M10 (`docs/tasks/tasks-ep-m10-ai-bouquet-assistant.json`).
> Analysis date: 2026-10-14

---

## EP-28: AI Florist

**Priority**: P1-high
**Dependencies**: EP-02 (vendor auth / tenant context), EP-15 (R2 storage pattern), EP-24 (Fly.io
deployment); vendor mobile app EP-M10 for the UI
**Requirement refs**: REQ-AIB-01 through REQ-AIB-19
**State**: in-progress — backend implemented and tested (pilot prototype on free providers); Fly
changes, provider evaluation and pilot rollout remain.

> Stock first. Recipes first. Preview on demand.

### Concept

A florist photographs what is on the bench (or types it in), corrects the estimated counts, picks a
style and an optional cost ceiling, and gets up to three bouquet recipes that are guaranteed to fit
the confirmed stock, each with an honest cost (missing prices are shown as missing, never as zero).
An illustrative image is generated only when the florist taps *Generate preview* for one recipe.

```
Photos/manual ─▶ Recognition job ─▶ Estimated stock ─▶ Florist confirms ─▶ Composition job
                  (billable)                                                (billable)
                                                                                 │
   Save / share ◀── Recipe detail (cost, margin) ◀── Validated recipes (≤3) ◀───┘
                         │
                         └─▶ Preview job (billable, on demand, cached) ─▶ private image
```

### Design decisions

- **Code is the gatekeeper**: every proposed recipe (LLM or rule-based) is re-validated against the
  confirmed stock snapshot; invalid ones are dropped, never "repaired" with another paid call.
- **Durable jobs in PostgreSQL**, not the RabbitMQ stub or the outbox: `FOR UPDATE SKIP LOCKED`
  claims, renewable leases, no transaction across provider calls.
- **The job row is the spend reservation and the idempotency record** — one table instead of three,
  with quota checks serialized by an advisory lock.
- **No blind retries**: a post-submission loss is `OutcomeUnknown` (spend kept reserved); BFL's
  polling URL is persisted so a crash resumes polling instead of paying twice.
- **Private media**: separate private R2 bucket, authenticated byte endpoints, EXIF stripped.
- **Free by default**: `Fake` recogniser, rule-based composer and SkiaSharp placeholder previews let
  the whole flow be tested in the app before any paid provider is chosen.
- **Worker in the API process for the pilot** — the API Fly machine must stay awake.

### Explicitly NOT part of this epic

- Board screen 06 "Additions · later release" (supplier items, shopping list).
- S/L sizes, flowerbox, recipe templates/regeneration history, hand-off to saleable bouquet creation.
- Automatic stock subtraction, supplier ordering, publishing.
- SignalR push of job progress (polling is authoritative), product analytics events.

### Stories & tasks

See `docs/tasks/tasks-ep28-ai-bouquet-assistant.json` (S-28-01 … S-28-06; T-28-001 … T-28-022).

1. **S-28-01 — Contracts, specs and trackers** (done)
   - T-28-001 docs placed in both repos · T-28-002 wire contract (`API-CONTRACTS.md` §21, `outbound.md`).
2. **S-28-02 — Domain** (done)
   - T-28-003 `DesignSession` · T-28-004 `RecipeValidator` · T-28-005 `CostCalculator` + `DesignCostProfile` ·
     T-28-006 31 domain tests.
3. **S-28-03 — Jobs, idempotency, quotas, persistence** (done)
   - T-28-007 `DesignJob` · T-28-008 EF config, repository, migration `AddBouquetDesignAssistant` ·
     T-28-009 usage service + job starter.
4. **S-28-04 — API and private media** (done)
   - T-28-010 commands/queries · T-28-011 `BouquetDesignController` · T-28-012 private storage + upload
     processing · T-28-013 per-caller idempotency cache key.
5. **S-28-05 — Providers and worker** (done)
   - T-28-014 free defaults · T-28-015 OpenAI/BFL adapters · T-28-016 processor + worker + retention ·
     T-28-017 32 flow/provider tests.
6. **S-28-06 — Deployment, cost and validation** (in progress)
   - T-28-018 deployment guide (done) · T-28-019 Fly changes + deploy · T-28-020 provider evaluation ·
     T-28-021 pilot rollout · T-28-022 follow-ups (deferred).

### What shipped, and the things worth knowing

- **New API surface** (all `VendorAccess`, under `/api/v1/vendor`): `design-assistant/usage`,
  `design-sessions[/{id}]`, `…/photos`, `…/recognize`, `…/inventory`, `…/recipes`,
  `…/recipes/{rid}/{preview,saved,cost}`, `design-recipes`, `design-cost-profile`.
- **New config**: `BouquetDesign:*` (flags, pilot list, limits, providers, storage, worker, retention);
  secrets `BouquetDesign__OpenAI__ApiKey`, `BouquetDesign__Bfl__ApiKey`,
  `BouquetDesign__Storage__AccessKeyId/SecretAccessKey`.
- **Operational gotcha**: the worker lives in the API process. With Fly's `auto_stop_machines =
  'suspend'` and `min_machines_running = 0`, queued jobs stall whenever the machine sleeps. The
  deployment guide switches the API to one always-on machine.
- **Pre-existing bug fixed**: `IdempotencyMiddleware` cached responses under the bare key, so two
  users sending the same key could replay each other's response; it is now scoped per caller + path.

### Rollout

Apply migration (automatic on API start) → deploy API with free providers and private R2 bucket →
test end-to-end in the vendor app (EP-M10) → run the provider evaluation → enable paid providers for
a test vendor via `PilotVendorIds` → 2–3 pilot shops → 10 shops. Rollback: set
`BouquetDesign__Enabled=false` (stops new paid work, keeps recipes readable); never drop tables.
Details: `docs/deployment/AI-BOUQUET-ASSISTANT.md`.
