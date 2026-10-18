# Requirement Gaps — Option I (Phone verification abuse protection)

> Companion to REQ-GAPS.md (Mobile API), REQ-GAPS-B.md (Portals/Analytics/IoT),
> REQ-GAPS-C.md (Vendor onboarding/subscription/vase), REQ-GAPS-D.md (Showcase deployment),
> REQ-GAPS-F.md (Customer portal mobile redesign), REQ-GAPS-G.md (Contact us) and REQ-GAPS-H.md (AI Bouquet Assistant).
> Requirement refs point at the tasks in `tasks-ep29-phone-verification-abuse-protection.json`.
> Analysis date: 2026-10-16
> Status: all 13 requirements implemented (EP-29 done — see `EPICS-I.md`).

## Context

Customer phone verification (EP-07 follow-up, `POST /api/v1/profile/phone/send-code` +
`/verify-code`) is about to be switched from the `NullSmsVerificationService` placeholder to the real
Twilio Verify provider on the public site `findmyflowers.pl`. Investigation on 2026-10-16 found:

- **Production silently uses the placeholder.** `Sms:Provider` defaults to `Null`, no Fly secret sets
  it, and `NullSmsVerificationService` reports success without sending — so customers see
  "Verification code sent" but nothing arrives — and accepts the public magic code `123456`, so
  anybody can mark any number as verified.
- **No cost/abuse controls.** The only guard is a per-user 5-requests/10-minutes rate limit shared by
  send and verify. There is no limit per destination number, per country, per account over time, nor
  a global budget. Every SMS costs money (Twilio Verify ≈ $0.05 + carrier fee), so the endpoint is
  an **SMS pumping (toll-fraud)** target: bots trigger sends to premium/attacker-controlled numbers.
- **Fake accounts are free.** `POST /api/v1/customers/register` is anonymous and not rate-limited;
  behind Fly's proxy `HttpContext.Connection.RemoteIpAddress` is the proxy, so any per-IP limit
  would bucket all users together.
- **A verified number is not unique.** Hundreds of accounts can each "verify" the same number.

Goal: limit one account to 1–2 numbers, make mass fake-account verification uneconomic, and cap the
worst-case SMS bill — **without adding friction for a normal customer** (one number, one or two
codes, an invisible captcha).

---

## REQ-SMSABUSE-01 — No fake success in production — P0
When no real SMS provider is configured in the `Production` environment, `send-code` must fail
honestly (`sms-not-configured`, HTTP 503) instead of pretending to send, and the magic code `123456`
must never verify. Dev/test behaviour of `NullSmsVerificationService` is unchanged.
**Accept**: with `ASPNETCORE_ENVIRONMENT=Production` and `Sms:Provider=Null`, send-code returns 503
and verify-code with `123456` returns 400; in Development both behave as before.

## REQ-SMSABUSE-02 — Operator runbook for Twilio Verify — P2
Document the one-time Twilio setup (trial account, Verify service, Geo Permissions = Poland only,
Fraud Guard, Fly secrets) and list the SMS secrets in `SECRETS-AND-ENV-VARS.md`.
**Accept**: following `docs/deployment/TWILIO-VERIFY-SETUP.md` end to end delivers a real SMS.

## REQ-SMSABUSE-03 — Real client IP behind Fly — P1
The API resolves the caller's IP from Fly's `Fly-Client-IP` header (set by Fly's edge proxy, not by
the client) when `ForwardedHeaders:TrustFlyClientIp=true`, so per-IP rate-limit partitions work.
**Accept**: with the flag on, a request carrying `Fly-Client-IP: 203.0.113.7` is partitioned as
`ip:203.0.113.7`; with the flag off the header is ignored.

## REQ-SMSABUSE-04 — Throttle anonymous customer registration — P1
`POST /api/v1/customers/register` gets a per-IP rate limit (default 10 per hour) so one host cannot
mint hundreds of accounts.
**Accept**: the 11th registration from one IP inside an hour returns 429.

## REQ-SMSABUSE-05 — Phone verification send ledger — P1
Every send-code request is recorded as a `PhoneVerificationAttempt` (customer, E.164 number,
requested-at, outcome `Sent | Rejected | ProviderFailed`, reason) via migration
`AddPhoneVerificationAttempts`. The ledger is the data source for all limits below.
**Accept**: each send-code call — allowed, refused or failed at the provider — writes one row.

## REQ-SMSABUSE-06 — Destination country allow-list — P0
Codes are only sent to numbers whose E.164 prefix is in `Sms:Limits:AllowedCountryPrefixes`
(default `["+48"]`). Others are refused with `phone-country-not-allowed` (400) before any SMS.
Complements Twilio Geo Permissions (defence in depth).
**Accept**: `+44…` is refused without calling the provider; `+48…` proceeds.

## REQ-SMSABUSE-07 — At most N numbers per account — P1
An account may request codes for at most `MaxDistinctNumbersPerCustomer` (default 2) different
numbers within `DistinctNumbersWindowDays` (default 30). A third number is refused with
`phone-limit-numbers-per-account` (429). Re-sending to an already-used number is not a new number.
**Accept**: numbers A and B succeed, C is refused, A again succeeds.

## REQ-SMSABUSE-08 — Daily send caps per account and per number — P1
At most `MaxSendsPerCustomerPerDay` (default 6) sends per account and `MaxSendsPerNumberPerDay`
(default 5) sends to one destination number **across all accounts** per rolling 24 h. Refusals:
`phone-limit-account-daily` / `phone-limit-number-daily` (429).
**Accept**: the 6th send to one number from 6 different accounts is refused.

## REQ-SMSABUSE-09 — Global daily SMS budget — P0
A platform-wide cap `Sms:Limits:GlobalDailySendBudget` (default 200; 0 = unlimited) on successful
sends per rolling 24 h. When reached, send-code returns `sms-daily-budget-exhausted` (503) and logs
a critical alert; at 80 % a warning is logged. Caps the worst-case bill of a bot attack.
**Accept**: with budget 3, the 4th send of the day returns 503 and the provider is not called.

## REQ-SMSABUSE-10 — One verified phone number per account — P1
A phone number can be verified on at most one customer account. Send-code refuses a number already
verified elsewhere (`phone-already-in-use`, 409, no SMS spent) and verify-code re-checks (race).
Enforced in the DB by a partial unique index on `CustomerPhone WHERE phone_verified`; the migration
de-duplicates existing data first (keeps the oldest account verified).
**Accept**: account B cannot verify a number account A has verified; the index exists.

## REQ-SMSABUSE-11 — Verified email before SMS — P2
When `Sms:Limits:RequireVerifiedEmail=true` (on in Production), send-code is refused with
`email-not-verified` (403) until the account's Keycloak email is verified. Google SSO accounts are
already verified, so they are unaffected.
**Accept**: an account with an unverified email gets 403 and no SMS is sent.

## REQ-SMSABUSE-12 — Turnstile on "Send code" — P2
When `Turnstile:SecretKey` is configured on the API, send-code requires a valid Cloudflare
Turnstile token (`captchaToken`), verified server-side; failure → `captcha-failed` (400). The
Customer App renders the (usually invisible) widget in the phone modal and the Complete-profile page
using the existing `ContactForm:TurnstileSiteKey`. Blank secret = check skipped (dev, mobile).
**Accept**: with a secret set, a request without a valid token is refused before any SMS.

## REQ-SMSABUSE-13 — Honest, specific UI messages — P2
The Customer App maps each refusal to a clear message (service unavailable, too many attempts,
number linked to another account, country not supported, verify your email first, captcha failed)
instead of a generic "check the phone number".
**Accept**: each refusal code shows its own message on the profile page.

---

### Explicitly NOT part of this epic
- Moving phone verification from the profile to first checkout (product decision; tracked separately).
- Twilio Lookup line-type intelligence (paid per lookup) — Fraud Guard + Geo Permissions first.
- Keycloak self-registration captcha (configured in Keycloak, not this codebase) — noted in runbook.
