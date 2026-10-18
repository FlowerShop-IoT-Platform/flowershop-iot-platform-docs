# Epics, Stories & Tasks — Option I (Phone verification abuse protection)

> Companion to EPICS.md (EP-01–09), EPICS-B.md (EP-10–18), EPICS-C.md (EP-19–23),
> EPICS-D.md (EP-24), EPICS-E.md (EP-25), EPICS-F.md (EP-26), EPICS-G.md (EP-27) and EPICS-H.md (EP-28).
> Epic is numbered EP-29 to continue the sequence (EP-28 is the AI Florist, `EPICS-H.md`).
> Task tracker (ground truth): `docs/tasks/tasks-ep29-phone-verification-abuse-protection.json`.
> Requirements: `docs/tasks/REQ-GAPS-I.md` (REQ-SMSABUSE-01 … REQ-SMSABUSE-13).
> Analysis date: 2026-10-16

---

## EP-29: Phone verification abuse protection

**Priority**: P1-high
**Dependencies**: EP-07 customer profile phone verification (`ISmsVerificationService`,
`/api/v1/profile/phone/*`); EP-24 Fly.io deployment; EP-27 Turnstile site key (reused); a Twilio
Verify account (operator runbook `docs/deployment/TWILIO-VERIFY-SETUP.md`)
**Requirement refs**: REQ-SMSABUSE-01 through REQ-SMSABUSE-13
**State**: done — all six stories implemented on one branch, one commit per story. Two migrations
(`AddPhoneVerificationAttempts`, `UniqueVerifiedCustomerPhone`). Real SMS delivery switches on when
the operator completes `docs/deployment/TWILIO-VERIFY-SETUP.md`; until then production answers an
honest 503 "temporarily unavailable".

> Make "verify your phone" safe to expose to the Internet: a normal customer verifies one number
> with one code; a bot farm hits a wall long before it costs real money.

### Concept

The phone-verification feature works, but production quietly runs the `Null` placeholder (fake
"code sent", public magic code `123456`). Switching to Twilio Verify turns every "Send code" click
into a paid SMS, which makes the endpoint a target for **SMS pumping** (toll fraud) and for
**mass fake-account verification**. The defence is layered — each layer is cheap for a real user:

| Layer | Stops | Normal user notices |
|---|---|---|
| Honest provider in prod | Fake success, magic-code bypass | Nothing (or a clear "unavailable") |
| Real client IP + registration throttle | One host minting hundreds of accounts | Nothing |
| Country allow-list (+ Twilio Geo Permissions) | Pumping to premium foreign ranges | Nothing (PL customers) |
| ≤ 2 numbers / account / 30 days | One account spraying many numbers | Nothing |
| Per-account & per-number daily caps | Many accounts hammering one number | Nothing |
| Global daily budget | Any attack's worst-case bill | Only during an attack |
| One verified number per account | Re-using one SIM for many accounts | Nothing |
| Verified email + Turnstile | Scripted accounts reaching the SMS step | Invisible captcha |

### Design decisions

- **Ledger, not counters in cache**: every send-code is a `PhoneVerificationAttempt` row. Limits are
  plain count queries over indexed columns, survive restarts and multiple machines, and the table
  doubles as an audit trail for support.
- **Check before mutate**: the guard runs *before* `Customer.UpdatePhoneNumber`, so a refused request
  never resets an existing verified number.
- **Fail closed where money is involved**: Turnstile provider errors and an exhausted budget refuse;
  a Keycloak lookup failure counts as "email not verified".
- **Config-switchable friction**: email gate and Turnstile are off unless configured, so dev, tests
  and the future native mobile app keep working; production turns them on.
- **Uniqueness in two places**: an application check (no SMS spent on a taken number, clean 409)
  plus a partial unique DB index as the race-proof backstop.
- **Twilio-side controls are part of the plan, not the code**: Geo Permissions (PL only) and Fraud
  Guard are documented in the runbook and complement the in-app allow-list.

### Explicitly NOT part of this epic

- Moving verification to first checkout (product decision).
- Twilio Lookup line-type intelligence (paid); Keycloak registration captcha (Keycloak config).

### Stories & tasks

See `docs/tasks/tasks-ep29-phone-verification-abuse-protection.json` (S-29-01 … S-29-06;
T-29-001 … T-29-015). Summary:

1. **S-29-01 — Honest SMS provider in production + Twilio runbook**
   - T-29-001 — `NullSmsVerificationService` refuses in Production (503, no magic code).
   - T-29-002 — specific refusal messages in the Customer App.
   - T-29-003 — `TWILIO-VERIFY-SETUP.md` runbook + secrets table rows.
2. **S-29-02 — Real client IP behind Fly + registration throttling**
   - T-29-004 — `Fly-Client-IP` resolution middleware (flag-gated).
   - T-29-005 — per-IP `customer-register` rate limit.
3. **S-29-03 — Send ledger with per-account, per-number and country limits**
   - T-29-006 — `PhoneVerificationAttempt` aggregate + repo + migration `AddPhoneVerificationAttempts`.
   - T-29-007 — `PhoneVerificationGuard` + `Sms:Limits` options.
   - T-29-008 — guard + ledger wired into the send handler; error mapping.
   - T-29-009 — tests.
4. **S-29-04 — Global daily SMS budget**
   - T-29-010 — budget check, warning/critical logs, 503.
5. **S-29-05 — One verified phone number per account**
   - T-29-011 — check on send and verify (409).
   - T-29-012 — partial unique index + de-duplication migration.
6. **S-29-06 — Bot friction: verified email + Turnstile**
   - T-29-013 — verified-email gate (403).
   - T-29-014 — server-side Turnstile verification.
   - T-29-015 — Turnstile widget in the Customer App phone forms.

### What shipped, and the things worth knowing

- **Refusal codes** (API body = code, so clients can show specific messages):
  `sms-not-configured` / `sms-daily-budget-exhausted` → 503, `sms-send-failed` → 502,
  `phone-limit-*` → 429, `phone-already-in-use` → 409, `email-not-verified` → 403,
  `phone-country-not-allowed` / `captcha-failed` / `invalid-phone-format` → 400.
- **Order of checks in the send handler**: format → captcha → verified email → global budget →
  country → distinct numbers → per-account/day → per-number/day → uniqueness → update phone → send.
  Everything before "update phone" leaves the customer's current (possibly verified) number alone.
- **Ledger counts only provider-reaching attempts** (`Sent`, `ProviderFailed`). Refusals — and
  `sms-not-configured` — are recorded as `Rejected` for audit but never use up a customer's limits.
- **New config**: `Sms:Limits:*` (defaults in `appsettings.json`, `RequireVerifiedEmail=true` in
  Production), `ForwardedHeaders:TrustFlyClientIp` (Production), `RateLimiting:CustomerRegisterPerHour`
  (default 10), `Turnstile:SecretKey` (Fly secret, blank = skipped). The Customer App reuses
  `ContactForm:TurnstileSiteKey` for the widget.
- **Migration caveat**: `UniqueVerifiedCustomerPhone` un-verifies all but the oldest account sharing a
  verified number (test accounts verified with the old magic code). Those customers verify again.
- **Mobile app note**: once `Turnstile__SecretKey` is set, `send-code` requires `captchaToken`; a
  native client will need a Turnstile-compatible flow (or its own key) before it calls the endpoint.

### Rollout

Apply migrations → deploy API (with `Sms:Limits` defaults) → deploy Customer App → operator follows
`TWILIO-VERIFY-SETUP.md` and sets the `Sms__*` and `Turnstile__SecretKey` Fly secrets. Until the
Twilio secrets are set, "Send code" shows an honest "temporarily unavailable" message.
