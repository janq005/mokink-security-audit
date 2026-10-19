# 04 — Control Testing

Each control is tested on two layers:

- **Design effectiveness** — is the control correctly set up (in config / code)?
- **Operating effectiveness** — is it actually working, for everyone, right now?

The gap between "it exists" and "it works for real" is where findings live. No
control is rated without evidence.

---

## C1 — Two-factor authentication enforcement ⚠️ Partial → **F1**

**Control:** Users — especially privileged/admin accounts — are required to use
MFA.

**Method:** Located the 2FA implementation across the codebase and read the
login-path logic.
```
grep -rln "check-2fa|twoFactor|totp|otp|2fa" app/ lib/
cat app/api/auth/check-2fa/route.ts
```

**Evidence:** A complete TOTP system exists — `app/api/account/2fa/{enroll,
verify,disable}`, `lib/totp.ts`, `lib/__tests__/totp.test.ts`, backup-codes page.
The sign-in probe `app/api/auth/check-2fa/route.ts` returns `required: true` **only
when the user's `totpEnabled` flag is set**. No role-based enforcement is present
in `lib/auth-authorize.ts`.

- **Design:** Pass — mechanism is well-built (TOTP, backup codes, enumeration-aware).
- **Operating (at time of test):** Partial/Fail — enforcement was **opt-in per
  user**; login succeeded with password alone when the flag was off. Admin
  accounts could operate without 2FA.

**Result: Partial → Remediated.** → Finding **F1**. Fixed same day (commit
`b456fcf`): 2FA now enforced on the authorization path for admin roles, Google
sign-in blocked for admins, old sessions invalidated; verified live via three
bypass tests. See F1 in `05-findings.md`.

---

## C2 — MFA on the Vercel account ❌ Fail → **F2**

**Control:** The account hosting mokink and storing all secrets requires MFA.

**Method:** Vercel account → security/authentication.

**Evidence:** `evidence/e-vercel-2fa-off.png` — Vercel prompts "Secure Your
Account with 2FA" with a "Skip securing my account" option; the prompt is shown
only when 2FA is not enabled.

- **Design & Operating (at time of test):** Fail — account protected by password alone.

**Result: Fail → Remediated.** → Finding **F2**. 2FA enabled on the Vercel
account same day.

---

## C3 — 2FA on the email account ❌ Fail → **F3**

**Control:** The account controlling mokink's mail channel (`info@mokink.lt`,
managed via a linked Google account — address redacted) requires 2FA.

**Method:** Account security settings.

**Evidence:** `evidence/e-email-2fa-off.png` — "Dvigubos autentifikacijos
nustatymai: ❌ Išjungta" (2FA disabled). Recovery email also unset ("Atkūrimo
el. paštas: –").

- **Design & Operating (at time of test):** Fail — password-only, no second
  factor, no recovery path.

**Result: Fail → Remediated.** → Finding **F3**. This is the password-reset
channel (R1) — a high-consequence account. 2FA enabled and recovery email set
same day.

---

## C4 — No live secrets in Git history ✅ Pass

**Control:** No secret values are committed to the repository.

**Method:** Full-history scan for committed env files and for secret value patterns.
```
git --no-pager log --all --oneline --name-only | grep -iE '(^|/)\.env($|\.local|\.production)'
git --no-pager grep -IE 'sk_live_|sk_test_|whsec_|postgres://[^ ]+:[^ ]+@' $(git rev-list --all)
```

**Evidence:** No real `.env` / `.env.local` / `.env.production` file in history.
All pattern matches are **placeholders** (`sk_test_your_stripe_secret_key`),
**CI dummies** (`sk_test_ci_dummy`, `whsec_test_ci_dummy`), test fixtures
(`sk_test_dummy`), or **documentation** (`docs/secret-rotation.md`,
`docs/INCIDENT_PLAYBOOK.md`). The tell: placeholder strings read as English
words; real keys read as line-noise. Secrets live only in Vercel env vars.

- **Design & Operating:** Pass.

**Result: Pass.** Positive note: the repo already contains a documented
**secret-rotation procedure** and **incident playbook** — mature Protect/Respond
artifacts rarely seen in junior projects.

---

## C5 — Stripe webhook signature verification ✅ Pass

**Control:** Incoming Stripe webhooks are cryptographically verified before the
app acts on them.

**Method:**
```
grep -rn "constructEvent|STRIPE_WEBHOOK_SECRET|stripe.webhooks" app/ lib/ pages/
```

**Evidence:** `app/api/webhooks/stripe/route.ts:86–89` calls
`stripe.webhooks.constructEvent(...)` with `process.env.STRIPE_WEBHOOK_SECRET`.
Forged payment-confirmation events cannot be accepted without the signing secret.

- **Design & Operating:** Pass. Residual Stripe risk (R6) is well-controlled.

**Result: Pass.**

---

## C6 — DB connection string not exposed to the client ✅ Pass

**Control:** The database URL is server-only, never shipped to the browser
(in Next.js, any `NEXT_PUBLIC_*` var is sent to the client).

**Method:**
```
grep -rn "NEXT_PUBLIC_DATABASE|NEXT_PUBLIC.*POSTGRES|NEXT_PUBLIC.*DATABASE_URL" .
```

**Evidence:** No matches. No `NEXT_PUBLIC_`-prefixed database variable exists.

- **Design & Operating:** Pass.

**Result: Pass.**

---

## C7 — Backups exist and restore works ⚠️ Partial → **F4**

**Control:** Recoverable backups exist, with a retention window adequate for
delayed-discovery incidents, and restore has been tested.

**Method:** Neon dashboard → Backup & Restore.

**Evidence:** `evidence/e-neon-backup-6h.png` — point-in-time restore limited to
a **6-hour** window; "No snapshots, no schedule set"; scheduled snapshots gated
behind a plan upgrade. No restore has ever been performed.

- **Design:** Pass — PITR mechanism exists.
- **Operating:** Partial — 6-hour window too short to survive a slow-to-detect
  incident; no scheduled snapshots. **Restore now tested** (below).

**Restore test (2026-10-19):** PITR of production to an isolated branch
`restore-test-2026-10-19` at the 08:52 point; branch ready in **≈1m56s** (RTO).
Verified: 48 tables, `users`=86, `MAX("createdAt")`=2026-10-12 — a genuine
historical state. Evidence: `evidence/e-neon-restore-{1-tables,2-data,3-branch}.png`.

**Result: Partial** — recovery mechanism verified; finding stands on the 6-hour
retention limit. → Finding **F4**.

---

## C8 — Login rate-limiting ✅ Pass

**Control:** Authentication endpoints throttle attempts to resist brute
force / credential stuffing.

**Method:**
```
grep -rn "rateLimit" app/api/auth/ lib/auth*
```

**Evidence:** A shared limiter (`lib/rate-limit.ts`) is applied across auth routes
— `register` (5/hr), `forgot-password` (5/15min), `reset-password` (5/15min),
`check-2fa` (20/hr) — and, critically, within the core authorization library
**`lib/auth-authorize.ts:26`**, which the login path runs through. The login path
is rate-limited.

- **Design & Operating:** Pass. Uses bcrypt for password hashing (noted in
  `check-2fa` source comment).

**Result: Pass.** *(An earlier provisional "Partial" was overturned once
`auth-authorize.ts` was inspected — evidence over assumption.)*

---

## Summary

| # | Control | Design | Operating (at test) | Result → Status |
|---|---------|--------|---------------------|-----------------|
| C1 | 2FA enforcement | Pass | Partial/Fail | ⚠️ Partial → ✅ Remediated |
| C2 | Vercel MFA | Fail | Fail | ❌ Fail → ✅ Remediated |
| C3 | Email 2FA | Fail | Fail | ❌ Fail → ✅ Remediated |
| C4 | Secrets in Git | Pass | Pass | ✅ Pass |
| C5 | Stripe webhook verify | Pass | Pass | ✅ Pass |
| C6 | DB client exposure | Pass | Pass | ✅ Pass |
| C7 | Backups / restore | Pass | Partial | ⚠️ Partial (restore tested; retention open) |
| C8 | Login rate-limiting | Pass | Pass | ✅ Pass |
