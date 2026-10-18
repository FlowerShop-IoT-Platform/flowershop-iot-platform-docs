# Epics, Stories & Tasks — Option G (Contact Us across portals)

> Companion to EPICS.md (EP-01–09), EPICS-B.md (EP-10–18), EPICS-C.md (EP-19–23),
> EPICS-D.md (EP-24), EPICS-E.md (EP-25) and EPICS-F.md (EP-26).
> Epic is numbered EP-27 to continue the sequence.
> Task tracker (ground truth): `docs/tasks/tasks-ep27-contact-us.json`.
> Requirements: `docs/tasks/REQ-GAPS-G.md` (REQ-CONTACT-01 … REQ-CONTACT-12).
> Design: `docs/superpowers/specs/2026-07-20-contact-us-portals-design.md`.
> Analysis date: 2026-07-20

---

## EP-27: Contact & messaging across portals

**Priority**: P2-medium
**Dependencies**: EP-26 (Customer portal layout — bottom tab bar preserved); Cloudflare Email
Routing rule `contact@findmyflowers.pl` → Gmail (set up 2026-07-20); a Brevo account + Cloudflare
Worker (operator SETUP.md)
**Requirement refs**: REQ-CONTACT-01 through REQ-CONTACT-12
**State**: done — all three pieces implemented and verified. Piece 1 (mailto links) shipped and now
doubles as the in-modal fallback; piece 2 (contact modal + Cloudflare Worker → Brevo) and piece 3
(customer→vendor logged message + Admin escalation, migration `AddVendorMessages`) are built,
tested and deploy-ready. Real email delivery switches on when the operator completes
`infra/cloudflare/contact-worker/SETUP.md`; until then the form shows its `mailto:` fallback and
vendor messages are still persisted for the admin dashboard.

> Let users reach the platform, and let customers reach the vendor behind a bouquet — by email,
> reliably, without exposing the operator's private Gmail.

### Concept

Three coordinated communication features:

1. **Contact us (mailto)** — a visible "Contact us" link in every portal's nav + footer that opens
   the user's email client pre-addressed to `contact@findmyflowers.pl`. Shipped; now also the
   in-modal fallback for piece 2.
2. **Contact-us form (platform)** — a modal form (the *primary* path, since a `mailto:` only works
   with a configured client and the browser gives JS no success signal) delivered via a
   **Cloudflare Worker → Brevo** pipeline that keeps the operator's Gmail hidden.
3. **Customer → Vendor direct message** — a signed-in customer messages the vendor who provided a
   bouquet (recipient shown openly). Every message is **persisted** for troubleshooting and
   **escalatable by an admin** (Sent → Escalated → Resolved).

### Why the pipeline looks the way it does

- The backend `IEmailService` is a **stub**, and Cloudflare routing is **receive-only** — the app
  can't send mail without new infrastructure.
- A `mailto:` gives JS no success/failure signal, so "detect missing client → show form" is not
  implementable; the **form is primary**, `mailto:` is the in-modal fallback.
- The Cloudflare Worker `send_email` binding can only reach **verified destinations**, so
  auto-replies and vendor recipients need a transactional provider → **Brevo** (free tier).
- The **Worker holds every email secret**; none live in the .NET apps (the property the operator
  wanted). Brevo also supplies SMTP for Gmail **"Send mail as"**, hiding the Gmail on manual replies.

### Keeping the operator's Gmail private

For platform contact, `From noreply@findmyflowers.pl`, `To contact@findmyflowers.pl`,
`Reply-To sender`. `karkinos79@gmail.com` is only the hidden Cloudflare forwarding destination — it
appears in no header. Auto-replies come from the Worker; manual replies use Gmail "Send mail as".

### Design decisions

- **Form primary, mailto fallback** (not client-detection).
- **Placement**: nav/menu **and** footer in every portal; per-portal pre-filled subject.
- **Customer bottom tab bar untouched** (5-tab max, EP-26) — link lives in dropdown/navbar/footer.
- **Vendor message entry point**: bouquet details page, **signed-in only** (so we have the
  customer's email + id to log and attribute).
- **Vendor recipient shown openly** — no hiding, unlike the operator contact flow.
- **New `VendorMessage` aggregate** modeled on `VaseIncident`; the only DB change is migration
  `AddVendorMessages`.
- **Spam protection**: honeypot + Turnstile + per-IP rate-limit.

### Explicitly NOT part of this epic

- No in-app inbox / threading / chat — a "message" is one outbound email (+ a DB log for piece 3).
- No push/SMS notifications; no customer-facing message-history UI (only admins see logged messages).
- No auto-escalation timer (admin escalates manually).

### Stories & tasks

See `docs/tasks/tasks-ep27-contact-us.json` (S-27-01 … S-27-05; T-27-001 … T-27-012). Summary:

1. **S-27-01 — Contact us in Vendor & Admin portals** (done)
   - T-27-001 / T-27-002 — navbar item + footer link (mailto).
2. **S-27-02 — Contact us in Customer App** (done)
   - T-27-003 / T-27-004 — dropdown + navbar + footer; verify build + render.
3. **S-27-03 — Cloudflare Worker → Brevo pipeline** (done)
   - T-27-005 — `infra/cloudflare/contact-worker` (Brevo, Turnstile, honeypot, KV rate-limit, CORS)
     plus a dependency-free Node test harness (`npm test`).
   - T-27-006 — operator `SETUP.md` (Brevo, DKIM/SPF, Turnstile, deploy, Send-mail-as).
4. **S-27-04 — Platform contact form** (done)
   - T-27-007 — `_ContactModal.cshtml` in all 3 portals; prefill + block-send; mailto fallback.
5. **S-27-05 — Customer → Vendor direct message** (done)
   - T-27-008 — `VendorMessage` aggregate + EF config + repo + migration `AddVendorMessages`.
   - T-27-009 — `SendVendorMessageCommand` + `VendorMessagesController` + Worker client.
   - T-27-010 — "Contact vendor" modal on bouquet details (signed-in only).
   - T-27-011 — Admin Messages page (Escalate / Resolve).
   - T-27-012 — tests + verification for pieces 2 & 3.

### What shipped, and the two things worth knowing

- **New API surface**: `POST /api/v1/vendor-messages` (CustomerAccess) and, for admins,
  `GET/POST /api/v1/admin/vendor-messages[/{id}/escalate|/resolve]` (PlatformAdmin).
- **New config**: `ContactForm:WorkerUrl` + `ContactForm:TurnstileSiteKey` in each portal;
  `ContactForm:WorkerUrl` + `ContactForm:WorkerToken` (Fly secret) on the API. All blank by default,
  which is a safe, degraded-but-working state.
- **Gotcha fixed during implementation**: customer-scoped MediatR requests must be listed in
  `TenantValidationBehavior`'s allow-list, otherwise the behaviour rejects them for having no
  `VendorId`. `SendVendorMessageCommand` was returning **403** until it was added — worth
  remembering for the next customer-facing command.
- **Delivery never loses a message**: the `VendorMessage` row is written *before* the Worker call, so
  an unconfigured or failing pipeline yields `delivered: false` with the record intact for admins.

### Rollout

Apply migration `AddVendorMessages` → deploy API → `wrangler deploy` the Worker → deploy the three
portals. The mailto fallback keeps "Contact us" working throughout. Operator completes SETUP.md
(Brevo/Turnstile/Gmail) to activate real delivery. EP-16 Playwright suite unaffected (additive UI).
