# 05 — Findings

Four findings. Each traces to a tested control (`04`) and a risk (`03`).
Severity = how badly the gap exposes the critical function.

---

## F1 — Two-factor authentication is implemented but not enforced

**Severity: High** (admin accounts) / Medium (general users)
**Control:** C1 · **Risk:** R3, R4

**Observation.** mokink has a complete TOTP 2FA system (enroll, verify, disable,
backup codes, test coverage). Enforcement is **opt-in per user** via a
`totpEnabled` flag; login succeeds with password alone when the flag is off.
No role-based requirement exists, so **privileged/admin accounts can operate
without 2FA**.

**Risk.** The admin API surface (`app/api/admin/*` — refunds, role changes, user
suspension, bulk email, outreach) is protected by password + rate-limiting only,
unless the admin voluntarily enabled 2FA. Admin account takeover → full platform
compromise.

**Design:** Pass (mechanism exists and is well-built).
**Operating:** Partial/Fail (not enforced where it matters most).

**Evidence.** `app/api/auth/check-2fa/route.ts` (per-user `totpEnabled`, no
enforcement/role gate); `app/api/account/2fa/*`.

**Recommendation.** Enforce 2FA for all admin/privileged roles — block login
completion without a verified second factor. Strongly prompt, then phase in, for
all users. **Low effort — the entire mechanism already exists; only an
enforcement check is missing.**

**✅ Remediated — 2026-10-19 (commit `b456fcf`).** 2FA is now enforced on the
authorization path, not the UI probe. Enforcement lives in
`authorizeCredentials()`; admin rights derive from the `mfa` value set *after*
that check, so the `check-2fa` probe endpoint is advisory only and cannot be used
to bypass it. The Google OAuth path is now blocked for admin accounts (admin
login requires email + password + TOTP). Pre-existing sessions were invalidated
on deploy. Verified live against mokink.lt via three tests, all passing:
(1) password login presents the TOTP field and admits only on a valid code;
(2) admin Google sign-in returns the "email + password + 2FA only" error;
(3) a pre-deploy session is refused at `/admin`.

> **Accepted residual risk — 2FA-status enumeration.** The `check-2fa` endpoint
> lets an unauthenticated caller learn whether a given email has 2FA enabled
> (including the admin address). It is rate-limited to 20 requests/IP/hour and
> documented in code as a deliberate trade-off for progressive-disclosure UX. It
> reveals no credential and does not aid bypass. **Disposition: risk accepted**
> (low severity; the mitigation — always showing the code field and removing the
> endpoint — costs login UX for marginal benefit). Revisit if threat model
> changes.

---

## F2 — No MFA on the Vercel account

**Severity: High**
**Control:** C2 · **Risk:** R4

**Observation.** The Vercel account hosting mokink and storing all environment
variables / secrets has 2FA disabled; Vercel actively prompts to enable it, and
the "skip" option had been taken.

**Risk.** A compromised Vercel password grants full access to hosting **and**
every secret (DB string, SMTP, Stripe keys) at once — the single-point-of-failure
/ concentration risk. Account takeover here compromises the entire system.

**Design & Operating:** Fail — password alone.

**Evidence.** `evidence/e-vercel-2fa-off.png` (pre-remediation state).

**Recommendation.** Enable authenticator-app 2FA on the Vercel account
immediately; store backup codes securely. One-click remediation.

**✅ Remediated — 2026-10-19.** Authenticator-app 2FA enabled on the Vercel
account; the "secure your account" prompt no longer appears.

---

## F3 — No 2FA on the email account controlling mokink's mail channel

**Severity: High**
**Control:** C3 · **Risk:** R1

**Observation.** The account managing `info@mokink.lt` (address redacted) has 2FA
disabled and no recovery email configured.

**Risk.** This mailbox sends mokink's password-reset and verification mail.
Compromise enables interception of reset mail and takeover of user accounts.
The absent recovery email also impedes recovery after a compromise — two gaps
compounding.

**Design & Operating:** Fail — password-only, no second factor, no recovery path.

**Evidence.** `evidence/e-email-2fa-off.png` (pre-remediation; address redacted).

**Recommendation.** Enable authenticator-app 2FA on this account immediately;
set a recovery email; store backup codes securely.

**✅ Remediated — 2026-10-19.** 2FA enabled on the account controlling the mail
channel; recovery email configured.

---

## F4 — Limited backup retention and untested restore

**Severity: Medium**
**Control:** C7 · **Risk:** R2 (CSF: Recover)

**Observation.** Neon provides point-in-time restore, but only within a **6-hour**
window on the current plan. No scheduled snapshots are configured ("No snapshots,
no schedule set"), and no restore has ever been performed or tested.

**Risk.** An incident not detected within 6 hours — delayed-discovery data
corruption, a bad migration, accidental deletion — cannot be recovered; the clean
state falls outside the window. Untested restore means recovery capability is
unproven.

**Design:** Pass (PITR exists).
**Operating:** Partial (short window, no scheduled snapshots).

**Restore tested — 2026-10-19.** Performed a point-in-time restore of production
to an isolated Neon branch (`restore-test-2026-10-19`, parent `production`) at the
08:52 restore point (~3h back). Branch provisioned in **≈1m56s** (RTO; Neon uses
copy-on-write, so this is near-instant provisioning, not a full data copy —
a traditional `pg_dump` restore of the same data would take materially longer).
Verified on the restored branch: **48 tables** present, `users` = **86 rows**,
`MAX("createdAt")` = **2026-10-12 09:53** — confirming the branch reflects the
chosen past state, not a clone of the present. **Recovery mechanism works.**

This closes the "untested restore" gap. The finding **remains open on retention**:
the 6-hour PITR window is too short to survive a delayed-discovery incident, and
no scheduled snapshots exist.

**Evidence.** `evidence/e-neon-backup-6h.png` (6h window);
`evidence/e-neon-restore-3-branch.png` (branch created from production);
`evidence/e-neon-restore-1-tables.png` (48 tables restored);
`evidence/e-neon-restore-2-data.png` (86 users).

**Recommendation.** (1) Extend retention and enable scheduled snapshots (plan
upgrade, or an independent periodic `pg_dump` to object storage as a cheap
alternative). (2) **Perform and document a test restore** to a scratch branch;
measure actual RTO/RPO.

---

## Observations (informational, not scored)

- **O1 — Permissive sandbox network policy.** The v0 project's Sandbox Network
  Policy is set to `allow-all` (no domain allow-listing).
  Evidence: `evidence/e-vercel-sandbox-allow-all.png`. Low severity for the
  current use; worth tightening if the sandbox handles sensitive data. (CSF:
  Protect.)

## Severity roll-up

| Finding | Severity | Theme | Status |
|---------|----------|-------|--------|
| F1 — 2FA not enforced | High (admin) | Identity & access | ✅ Remediated (commit `b456fcf`, verified live) |
| F2 — Vercel no MFA | High | Identity & access | ✅ Remediated |
| F3 — Email no 2FA | High | Identity & access | ✅ Remediated |
| F4 — Backups short/untested | Medium | Recovery | ◐ Restore tested; retention still open |

All three High (identity & access) findings were remediated the same day the
audit was run. F4's recovery mechanism was verified by a live restore test; the
only item still open is the 6-hour retention window (plan-level). One residual
risk is formally **accepted** (the `check-2fa` enumeration vector under F1).

Three of four findings are one theme: **MFA exists or is possible everywhere, but
is enforced nowhere that counts.**
