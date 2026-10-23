# Epics, Stories & Tasks — Option Q (Online-only vendors: bouquets without a smart vase)

> Companion to EPICS.md (EP-03 discovery, EP-04 orders), EPICS-C.md (EP-19 onboarding, EP-21/22
> vases) and EPICS-O.md (EP-37 custom orders).
> Epic is numbered EP-39 to continue the sequence.
> Task tracker (ground truth): `docs/tasks/tasks-ep39-online-only-vendors.json`.
> Requirements: `docs/tasks/REQ-GAPS-Q.md` (REQ-ONL-01 … REQ-ONL-19).
> Mobile counterparts: vendor app EP-M19 (T-39-020); customer mobile app (T-39-021).
> Analysis date: 2026-10-23

---

## EP-39: Online-only vendors — sell bouquets without a smart vase

**Priority**: P1-high
**Dependencies**: EP-03 (discovery), EP-04 (orders + Stripe), EP-19 (registration), EP-20 (platform access)
**Requirement refs**: REQ-ONL-01 through REQ-ONL-19
**State**: proposed — design only; open decisions in REQ-GAPS-Q §H.

> The vase stops in-store and online customers buying the same bouquet. Online-only shops and
> freelance florists never have walk-in customers or collections, so they don't need one. They list
> a finished bouquet with a photo, it appears on the map as "Delivery only", and it sells once.

### Concept

```
 Vendor.SalesMode = OnlineOnly
   Vendor Portal "Add a bouquet" ─▶ POST /api/v1/vendor/online-listings (photo, price, freshness)
        ─▶ Bouquet.ListOnline(...)  ListingChannel = Online, VaseId = null, Status = Available
        ─▶ BouquetBecameAvailable ─▶ BouquetHub push + customer bouquet alerts
 Customer map (purchasable only, within delivery range, "Delivery only", fuzzed pin)
   ─▶ cart: IsPurchasable → Reserve (optimistic concurrency: one winner, others 409)
   ─▶ checkout: Delivery required (pickup-not-offered) → Stripe → vendor fulfils → Delivered → Sold
 Expiry: declared freshness reaches 0 or MaxListingDays elapsed → Expired (off the map)
 Vase-only side effects (MQTT, vase state) skipped by the existing VaseId != null guards
```

### Design decisions
- **No virtual vase.** A fake `SmartVase` would be hidden by the 30 s offline map filter. It would
  open EP-22 recovery incidents, receive MQTT commands, hold only one bouquet at a time and skew
  fleet and analytics counts. Every one of those needs an `IsVirtual` check in IoT code. A
  vase-optional listing touches far less: two guards, two columns and three null reads.
- **Say what the bouquet is.** `ListingChannel` (Vase | Online) on the bouquet and `SalesMode`
  (InStore | OnlineOnly) on the vendor make the business rule explicit and checkable:
  `IsPurchasable`. "Has a vase" is no longer a stand-in for "can be sold".
- **The bouquet status is the lock.** With no physical display, the lock is
  `Available → Reserved → Sold`, protected by the `Version` concurrency token. This also closes
  today's double-reserve race for vase bouquets.
- **Same unit semantics.** An online listing is one physical bouquet that sells once, the same as a
  vase bouquet. Repeatable products are a separate epic (open decision 3).
- **Delivery only, existing delivery rules.** No pickup, and the same geocode, fee and
  `MaxDistanceKm` checks. The map shows online listings only to customers they can be delivered to.
- **Fix the existing leak.** Vase-less catalog bouquets with a price show on the map today but
  can't be bought. The map now shows only purchasable bouquets.

### Stories & tasks
See `tasks-ep39-online-only-vendors.json` (S-39-01 … S-39-07; T-39-001 … T-39-023).
1. **S-39-01 — Trackers** — T-39-001.
2. **S-39-02 — Domain and persistence** — T-39-002 vendor SalesMode · T-39-003 ListingChannel + ListOnline + IsPurchasable · T-39-004 vase-agnostic events · T-39-005 bouquet concurrency · T-39-006 nullable line VaseId · T-39-007 migration `AddOnlineOnlyListings` · T-39-008 listing expiry.
3. **S-39-03 — Purchase flow** — T-39-009 IsPurchasable guards · T-39-010 delivery-only rule · T-39-011 null-safe read models · T-39-012 fulfilment without vase side effects.
4. **S-39-04 — Discovery** — T-39-013 purchasable-only map · T-39-014 delivery-only fields, range, pin fuzzing.
5. **S-39-05 — Vendor API and portals** — T-39-015 online listing API · T-39-016 sales mode settings + registration · T-39-017 Vendor Portal · T-39-018 Admin Portal.
6. **S-39-06 — Customer surfaces and mobile** — T-39-019 Customer App · T-39-020 vendor app EP-M19 · T-39-021 customer mobile app.
7. **S-39-07 — Tests, contract and docs** — T-39-022 tests · T-39-023 contract §25, manuals, changelog.

**Suggested order**: S-39-02 → S-39-03 → S-39-04 (T-39-013 can ship first on its own, as a fix) →
S-39-05 → S-39-06 → S-39-07.

### Rollout
1. Ship the migration and the domain changes. Every existing vendor is `InStore` and every bouquet
   is `Vase`, so behaviour is unchanged.
2. Ship T-39-013 (purchasable-only map) and T-39-005 (concurrency). Both are safe fixes on their own.
3. Enable online listings for one pilot vendor. An admin switches its sales mode, and you run one
   real paid delivery end to end.
4. Open the "How do you sell?" choice in registration, then the mobile follow-ups.
