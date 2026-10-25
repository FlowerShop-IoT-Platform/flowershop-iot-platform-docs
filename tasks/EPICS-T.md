# Epics, Stories & Tasks — Option T (Online-only vendors: a vase-free vendor UI)

> Companion to EPICS-Q.md (EP-39 online-only vendors) and EPICS-C.md (EP-21 vase ordering, EP-22 asset recovery).
> Epic is numbered EP-42 to continue the sequence.
> Task tracker (ground truth): `docs/tasks/tasks-ep42-online-only-vase-free-ui.json`.
> Requirements: `docs/tasks/REQ-GAPS-T.md` (REQ-NOV-01 … REQ-NOV-12).
> Mobile counterpart: vendor app EP-M22 (T-42-015).
> Analysis date: 2026-10-25

---

## EP-42: Online-only vendors — a vase-free vendor UI

**Priority**: P1-high
**Dependencies**: EP-39 (sales mode, `SalesModeView`, `OnlineOnlyVasesFilter`), EP-21/22 (vase screens being hidden)
**Requirement refs**: REQ-NOV-01 through REQ-NOV-12
**State**: proposed. Trackers only (S-42-01). REQ-GAPS-T §H open: unknown mode fails closed, no JWT claim,
switch-back behaviour, server refusal in scope.

> An online-only vendor will never have a smart vase, so they should never see one: no vase menu, tile, prompt,
> picker, toast, help chapter or sentence. That holds in the Vendor Portal and the vendor app, and from the first
> page after sign-up.

### Concept

```
 sign-in / registration ─▶ session.SalesMode = InStore | OnlineOnly        (unknown = null, retried)
                                   │
           SalesModeView.ShowVaseEntryPoints(mode)  — true only for a known InStore
                                   │
   nav · More · dashboard · analytics · Bouquets picker · assessor wording · vase scripts · Help topics
                                   │
   Vases/* ─▶ OnlineOnlyVasesFilter (known OnlineOnly only) ─▶ Bouquets + message
   API vase writes ─▶ SalesModeRules.EnsureVasesAllowed ─▶ 409 vendor-online-only
   static scan test: no ungated vase link, script or wording in Pages/**/*.cshtml
```

### Design decisions
- **Fix fail-open first.** Most of the leftover pollution comes from `SalesModeService` falling back to `InStore`
  when the mode can't be read, which shows the whole vase portal. The mode is now read at sign-in. When it is unknown,
  vase entry points are hidden but no page is redirected, so an in-store vendor can still open `/Vases` during an API
  error (proposed, §H.1).
- **One rule per client.** Every vase surface asks `SalesModeView` (portal) or `SalesModes` (app). There are no inline
  mode comparisons in views.
- **The server agrees with the UI.** Hidden features are also refused by the API (`vendor-online-only`). A stale tab
  or an old app build can't order a vase for an online-only vendor.
- **A guard instead of vigilance.** A static scan fails the build when a new ungated vase link, script or sentence
  appears in the portal views. The app gets the same guard for XAML and routes.
- **No JWT claim.** The mode can change in the middle of a session, so a claim would go stale (proposed, §H.2).
- **Out of scope**: Admin Portal fleet views, customer surfaces (EP-39), Hybrid mode.

### Stories & tasks
See `tasks-ep42-online-only-vase-free-ui.json` (S-42-01 … S-42-07; T-42-001 … T-42-015).
1. **S-42-01 — Trackers** — T-42-001 (done).
2. **S-42-02 — Sales mode known up front** — T-42-002 mode at sign-in/registration, unknown state, `ShowVaseEntryPoints` · T-42-003 tests.
3. **S-42-03 — Vendor Portal sweep** — T-42-004 Analytics · T-42-005 Bouquets vase picker · T-42-006 AI Florist/assessor wording · T-42-007 vase real-time scripts · T-42-008 copy sweep (registration, banners, empty states, billing, emails).
4. **S-42-04 — API refuses vase-only actions** — T-42-009 `vendor-online-only` guard · T-42-010 tests.
5. **S-42-05 — Help, contract and docs** — T-42-011 Help topics + manual quick start · T-42-012 contract §25 + changelog.
6. **S-42-06 — Regression guard** — T-42-013 static vase-surface scan · T-42-014 per-mode page-model tests.
7. **S-42-07 — Mobile follow-up** — T-42-015 vendor app EP-M22.

**Suggested order**: S-42-02 → S-42-03 → S-42-04 → S-42-06 → S-42-05 → S-42-07. Confirm §H before S-42-02 and S-42-04.

### Rollout
1. Ship S-42-02 + S-42-03 together. In-store vendors see no change. Online-only vendors lose the leftover vase UI.
2. Ship S-42-04 after the vendor app maps `vendor-online-only` (EP-M22), so old builds show a sensible message.
3. Check by registering a new online-only test vendor on prod. Walk every portal page and app tab, and expect no vase
   UI. Then switch it to in-store and expect all of it back.
