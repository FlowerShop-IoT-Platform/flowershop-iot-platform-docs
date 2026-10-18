# Requirement Gaps — Option G (Contact Us across portals)

> Companion to REQ-GAPS.md (Mobile API), REQ-GAPS-B.md (Portals/Analytics/IoT),
> REQ-GAPS-C.md (Vendor onboarding/subscription/vase), REQ-GAPS-D.md (Showcase deployment)
> and REQ-GAPS-F.md (Customer portal mobile redesign).
> Requirement refs point at the tasks in `tasks-ep27-contact-us.json`.
> Design: `docs/superpowers/specs/2026-07-20-contact-us-portals-design.md`.
> Analysis date: 2026-07-20

## Context

On 2026-07-20 a **Cloudflare Email Routing** rule was created that forwards all mail arriving at
`contact@findmyflowers.pl` into the operator's Gmail inbox (receive-only forwarding). None of the
three portals currently expose a way for a user to reach the platform. This epic adds a visible
**"Contact us"** affordance to every portal.

Because the backend `IEmailService` is a **stub** (sends nothing) and Cloudflare routing is
**receive-only** (the app cannot send mail), "Contact us" is implemented as a client-side
**`mailto:` link** — **presentation only**: no API, DTO, handler, migration, auth, routing or
service change, and no message is stored. The user's own email client composes the message to
`contact@findmyflowers.pl`, which Cloudflare forwards to Gmail.

---

## REQ-CONTACT-01 — Vendor Portal "Contact us" link — P2
Add a "Contact us" affordance to the Vendor Portal in both the top navbar (before Logout) and the
footer (beside Privacy). It is a `mailto:contact@findmyflowers.pl` link with a pre-filled subject
`FlowerShop Vendor Portal enquiry` and a `bi-envelope` icon.
**Accept**: on every Vendor Portal page the link is visible in the navbar and footer; clicking it
opens the user's email client pre-addressed to `contact@findmyflowers.pl` with the subject
pre-filled. No backend call is made.

## REQ-CONTACT-02 — Customer App "Contact us" link — P2
Add a "Contact us" affordance to the Customer App in the account dropdown (before Logout), the top
navbar, and the footer (beside Privacy). It is a `mailto:contact@findmyflowers.pl` link with a
pre-filled subject `FlowerShop Customer enquiry` and a `bi-envelope` icon. The mobile bottom tab
bar is intentionally **not** changed (it already holds its full five tabs).
**Accept**: the link is visible for authenticated users (dropdown + navbar + footer) and for
anonymous users (navbar + footer); clicking it opens a pre-addressed email; the bottom tab bar is
unchanged; the EP-26 layout is not regressed.

## REQ-CONTACT-03 — Admin Portal "Contact us" link — P3
Add a "Contact us" affordance to the Admin Portal in both the top navbar (before Logout) and the
footer (beside Privacy). It is a `mailto:contact@findmyflowers.pl` link with a pre-filled subject
`FlowerShop Admin Portal enquiry` and a `bi-envelope` icon.
**Accept**: on every Admin Portal page the link is visible in the navbar and footer; clicking it
opens a pre-addressed email. No backend call is made.

---

# Extension — Contact form + Customer↔Vendor messaging (2026-07-20)

> Because a `mailto:` link only works when the user has a configured email client (and the browser
> gives JS no success signal), the "Contact us" link is upgraded to open a **modal form** (primary
> path) with the `mailto:` link retained inside as a fallback. Delivery uses a **Cloudflare Worker
> → Brevo** pipeline that keeps the operator's private Gmail hidden. A separate flow lets a
> signed-in customer message the **vendor** of a bouquet (recipient shown openly), logged in the DB
> for admin escalation.

## Platform contact form (pieces 1–2)

### REQ-CONTACT-04 — Contact modal form — P2
Clicking "Contact us" in any portal opens a Bootstrap modal with a form: **Your email** (required),
**Subject**, **Message** (required), a hidden **honeypot** field, and a **Turnstile** widget. The
modal also contains an "or use your own email app" `mailto:` fallback link.
**Accept**: the modal opens from the existing nav/footer triggers; the `mailto:` fallback still
works; the form validates required fields client-side.

### REQ-CONTACT-05 — Sender prefill & block-send rule — P2
The **Your email** field is prefilled from the session: Vendor/Admin from the `email` JWT claim,
Customer from `User.FindFirst("email")`. When no email is available (anonymous), the field is blank
and required and the **Send button stays disabled until a syntactically valid email is entered**.
**Accept**: signed-in users see their email prefilled; anonymous users cannot send until a valid
email is typed.

### REQ-CONTACT-06 — Worker → Brevo delivery, Gmail hidden — P2
The form POSTs to a Cloudflare Worker that sends via Brevo to `contact@findmyflowers.pl`
(From `noreply@findmyflowers.pl`, Reply-To = the sender). The operator's Gmail
(`karkinos79@gmail.com`) appears in no email header — it is only the hidden Cloudflare forwarding
destination. All email secrets live in the Worker, none in the .NET apps.
**Accept**: a submitted form arrives in the operator's Gmail; the message headers show only
`findmyflowers.pl` addresses; replying reaches the sender.

### REQ-CONTACT-07 — Auto-reply & spam protection — P2
The Worker sends an automatic acknowledgement to the sender (From `noreply@findmyflowers.pl`) and
protects the endpoint with: honeypot drop, server-side Turnstile verification, and a per-IP
rate-limit.
**Accept**: the sender receives an acknowledgement; a filled honeypot or failed Turnstile is
rejected; repeated rapid submissions are throttled.

### REQ-CONTACT-08 — Operator setup guide — P3
A `SETUP.md` documents the one-time operator steps that cannot be automated: Brevo account + domain
verification (DKIM/SPF in Cloudflare), Turnstile keys, `wrangler deploy`, and Gmail "Send mail as".
**Accept**: following SETUP.md end-to-end produces working delivery; until done, the form degrades
to the `mailto:` fallback.

## Customer → Vendor direct message (piece 3)

### REQ-CONTACT-09 — "Contact vendor" from a bouquet — P2
A signed-in customer sees a **Contact vendor** button on the bouquet details page that opens a
modal to message the vendor who provided that bouquet. Anonymous users are prompted to sign in.
**Accept**: a signed-in customer can open the modal and send a message about a specific bouquet;
the vendor's identity/address is shown openly (not hidden).

### REQ-CONTACT-10 — Persist every vendor message — P2
Each customer→vendor message is stored as a **`VendorMessage`** record (customer, customer email,
vendor, vendor email, bouquet, text, status, timestamps) via a new aggregate + migration
`AddVendorMessages`, for troubleshooting and escalation.
**Accept**: sending a message creates a queryable DB record with status `Sent` and `SentAt` set.

### REQ-CONTACT-11 — Deliver via Worker → Brevo (vendor as recipient) — P2
`POST /api/v1/vendor-messages` (CustomerAccess) resolves the bouquet's vendor, persists the record,
then calls the Worker → Brevo to email the vendor (To = vendor, Reply-To = customer,
From `noreply@findmyflowers.pl`). No auto-reply for this flow.
**Accept**: the vendor receives the email; a reply reaches the customer; a matching DB record
exists. Vendor resolution never uses `VendorId.New()` — it reads the bouquet's real vendor.

### REQ-CONTACT-12 — Admin Messages page & escalation — P2
A new Admin Portal **Messages** page lists vendor messages (customer, vendor, bouquet, text,
sentAt, status) and lets an admin **Escalate** (vendor ignored it) and **Resolve** (with a note).
Status lifecycle: **Sent → Escalated → Resolved**.
**Accept**: an admin can view logged messages and move one through Escalated and Resolved; changes
persist and are reflected in the list.
