# Epics, Stories & Tasks — Option E (Freshness Visibility)

> Companion to EPICS.md (EP-01–09), EPICS-B.md (EP-10–18), EPICS-C.md (EP-19–22/23) and
> EPICS-D.md (EP-24, Showcase Deployment).
> Epic is numbered EP-25 to continue the sequence.
> Task tracker (ground truth): `docs/tasks/tasks-ep25-bouquet-freshness-visibility.json`.
> Analysis date: 2026-07-15

---

## EP-25: Bouquet Freshness Visibility

**Priority**: P2-medium
**Dependencies**: EP-03 (bouquet discovery / map card), EP-13 (Customer Web App map)
**Requirement refs**: REQ-FRESHVIS-01 through REQ-FRESHVIS-03
**State**: done

> Vendors declare how long a bouquet stays fresh; customers see it on the map at a glance.

### Concept

A vendor picks a **freshness rating of 1–3 flowers** when publishing a bouquet, where the number is the
count of days the bouquet is guaranteed to stay fresh (`1` = up to one day, `3` = three days for sure).
Customers see the rating on the map bouquet card as **three flower pictograms** — coloured flowers for the
current level and greyed flowers for the remainder. A rating of `2` renders as two coloured flowers and
one grey.

The rating **decays by one flower per full day** since the bouquet was created: a bouquet published at
rating `3` shows 3 coloured flowers on day 0, 2 on day 1, 1 on day 2, and 0 (all grey) from day 3. The
decay is computed at read time from the creation timestamp — there is no background job and no stored
countdown.

### Explicitly NOT part of this epic

- **No price impact.** Setting or decaying the freshness rating never changes the price. Freshness-based
  *discounting* is a separate, pre-existing feature (`BouquetFreshnessUpdateService` +
  `FreshnessPricingStrategy`, configured per vendor). The two are intentionally decoupled — this epic is
  display-only.
- **Not the sensor score.** `Bouquet.FreshnessScore` (0–100, sensor/MQTT-derived, `<20` ⇒ expired) is a
  different field and is left untouched. The vendor rating lives on the new `Bouquet.InitialFreshnessDays`.

### Domain model

| Member | Meaning |
|---|---|
| `Bouquet.MaxFreshnessScale` (const = 3) | Number of pictograms the rating is shown against. |
| `Bouquet.InitialFreshnessDays` (int?, 1–3) | Vendor-declared rating captured at creation; `null` when unset. |
| `Bouquet.SetInitialFreshnessDays(int)` | Sets/updates the rating (clamped 1–3; `≤0` clears it). Never touches price. |
| `Bouquet.CurrentFreshnessDays(DateTime utcNow)` | Decayed level = `Initial − fullDaysElapsed`, floored at 0; `null` when unset. |

### Stories & tasks

See `docs/tasks/tasks-ep25-bouquet-freshness-visibility.json` (S-25-01; T-25-001 … T-25-005) for the
authoritative breakdown. Summary:

1. **T-25-001** — `InitialFreshnessDays` + decay computation on the Bouquet aggregate (+ unit tests).
2. **T-25-002** — persistence: migration `AddBouquetInitialFreshnessDays` + snapshot.
3. **T-25-003** — expose `currentFreshnessDays` / `maxFreshnessScale` / `initialFreshnessDays` on read
   models and populate every bouquet query/command handler.
4. **T-25-004** — vendors set the rating at publish time (Set Price modal → `PUT /bouquets/{id}/price`),
   plus catalog-creation pass-through.
5. **T-25-005** — flower pictograms on the customer map card, the customer bouquet details page, and the
   vendor's own bouquet cards.

### API surface

`initialFreshnessDays`, `currentFreshnessDays` and `maxFreshnessScale` are added to the customer `nearby`
list items and `bouquet detail` payloads (see `docs/API-CONTRACTS.md`). Vendors set the rating via the
optional `freshnessDays` field on `PUT /api/v1/bouquets/{id}/price` and on bouquet creation.
