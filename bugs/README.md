# Bug reports and bugfix test sessions

One file per troubleshooting session, written by the `/bugfix-test` skill (`.claude/skills/bugfix-test/`):

- `YYYY-MM-DD-bugfix-test-session.md`: the fixes verified (Pass / Partial / Fail / Blocked, with evidence) and the
  **new bugs found** in that session (`NB-YYYY-MM-DD-n`).
- `screenshots/YYYY-MM-DD-<session>/`: the screenshots those files refer to, with personal data redacted.

Format: `.claude/skills/bugfix-test/session-template.md`. There is no production environment; "cloud" means the dev
environment at `*.findmyflowers.pl`. Mobile-app triage documents live in
`flower-shop-vendor-mobile-app/docs/bugs/`.
