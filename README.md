# Security & Resilience Audit — mokink.lt

A self-directed security audit of a live production web application
(mokink.lt / *TavoMokytojas* — a tutoring marketplace), structured the way a
regulated-sector reviewer would approach it: scope → asset inventory → risk
assessment → control testing with evidence → findings → remediation.

The methodology is modelled on **DORA** (third-party register, critical-function
identification, incident classification) and **NIST CSF 2.0** (Identify / Protect
/ Detect / Respond / Recover), the frameworks a GRC / operational-resilience
function works to day to day.

**Author:** Titas Jankūnas
**System audited:** mokink.lt (Next.js · TypeScript · PostgreSQL/Neon · Stripe · Vercel)
**Audit date:** 19 October 2026

---

## Headline

**mokink.lt is well-engineered at the code layer; the account-layer gaps this
audit found were identified, remediated, and verified the same day.**

The application already did the hard technical things correctly — Stripe webhook
signatures verified, no live secrets in Git, the database connection string
server-only, authentication routes rate-limited, and a full TOTP two-factor
system built. The weaknesses were **operational**, and all three High findings
have since been closed:

- **MFA existed but wasn't enforced**, including for admin accounts →
  **fixed**: 2FA now enforced on the authorization path for admins, the Google
  sign-in path blocked for admins, and pre-existing sessions invalidated. Verified
  live with three bypass tests (F1).
- **The Vercel account (all secrets) and the email account (password-reset
  channel) had no 2FA** → **fixed**: 2FA enabled on both; recovery email set on
  the mail account (F2, F3).
- **Backups covered only a 6-hour window and had never been restore-tested** →
  restore now **tested and verified** via an isolated point-in-time branch; the
  short retention window is the one item still open (F4).

One lower-severity item — an endpoint that reveals whether an email has 2FA
enabled — is documented as a formally **accepted residual risk** (rate-limited,
reveals no credential, mitigation costs more than it's worth).

---

## Results at a glance

| # | Control | Result at test | Finding | Status |
|---|---------|----------------|---------|--------|
| C1 | 2FA enforcement (esp. admin) | ⚠️ Partial | **F1** | ✅ Remediated |
| C2 | Vercel account MFA | ❌ Fail | **F2** | ✅ Remediated |
| C3 | Email account 2FA | ❌ Fail | **F3** | ✅ Remediated |
| C4 | No secrets in Git history | ✅ Pass | — | — |
| C5 | Stripe webhook signature verification | ✅ Pass | — | — |
| C6 | DB connection string not client-exposed | ✅ Pass | — | — |
| C7 | Backups exist & restore tested | ⚠️ Partial | **F4** | ◐ Restore tested; retention open |
| C8 | Login rate-limiting | ✅ Pass | — | — |

**At audit time:** 4 Passes · 3 Fails/Partials on identity & access · 1 Partial on
recovery. **Now:** all 3 identity & access findings remediated and verified; F4's
restore tested, retention the one open item; 1 residual risk formally accepted.

---

## How to read this repo

| File | What's in it |
|------|--------------|
| [`01-scope.md`](01-scope.md) | What was audited, the critical function, and what was deliberately excluded |
| [`02-third-party-register.md`](02-third-party-register.md) | DORA-style register of ICT providers, criticality, and fallback |
| [`03-risk-register.md`](03-risk-register.md) | Risks (threat → impact), likelihood × impact ratings, mapped to NIST CSF |
| [`04-control-tests.md`](04-control-tests.md) | The 8 controls tested — method, evidence, design vs. operating effectiveness |
| [`05-findings.md`](05-findings.md) | F1–F4 written up: observation, risk, severity, recommendation |
| [`06-remediation-plan.md`](06-remediation-plan.md) | Prioritised fixes with owners and target dates |
| [`07-nist-csf-mapping.md`](07-nist-csf-mapping.md) | Coverage across the five CSF functions |
| [`evidence/`](evidence/) | Screenshots and command output referenced by the control tests |

---

## A note on method

Two findings in this audit were **corrected by the evidence** rather than
confirmed by assumption:

- An initial read of "no MFA" was overturned on inspection of the code — a
  complete TOTP system exists; the real issue is **enforcement**, not absence
  (F1).
- An initial read of "no login rate-limiting" was overturned once the core
  authorization library was inspected — the login path **is** rate-limited (C8
  passes).

Recording these reversals is deliberate. An assessment that only confirms its
own starting assumptions is not an assessment. Every Pass and every finding here
rests on a specific piece of evidence, cited in `04-control-tests.md`.
