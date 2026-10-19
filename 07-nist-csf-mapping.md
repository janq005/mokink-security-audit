# 07 — NIST CSF 2.0 Coverage Mapping

Where this audit's controls and findings land across the five Cybersecurity
Framework functions. Useful as a one-glance view of posture by function.

| Function | What it covers | mokink controls tested | Posture |
|----------|----------------|------------------------|---------|
| **Identify** | Knowing what you have | Asset inventory, third-party register, risk register (`01`–`03`) | ✅ Documented |
| **Protect** | Stopping bad things | C1 2FA, C2 Vercel MFA, C3 email 2FA, C4 secrets, C5 webhook verify, C6 DB exposure, C8 rate-limiting | ✅ Strong — code controls passed; account-MFA gaps found and remediated |
| **Detect** | Noticing something's wrong | C5 (webhook verification); charge monitoring (recommended) | ⚠️ Partial — limited monitoring/alerting |
| **Respond** | Acting on an incident | Incident playbook present in repo (`docs/INCIDENT_PLAYBOOK.md`) | ✅ Documented |
| **Recover** | Returning to normal | C7 backups / restore | ⚠️ Partial — short window, untested restore (F4) |

## Reading the map

- **Identify** and **Respond** are the quiet strengths — the registers here plus
  mokink's own in-repo incident playbook and secret-rotation doc.
- **Protect** was bimodal at audit time: *code-level* controls passed (webhook
  verification, no leaked secrets, server-only DB string, rate-limiting), while
  *account-level* controls (MFA) failed. That split was the audit's headline —
  and the account-level gaps have since been remediated (F1–F3), bringing the
  function up to strong.
- **Detect** is the thinnest function — there's little active monitoring or
  alerting beyond signature verification. A natural next audit cycle: add
  logging/alerting on password-reset volume and unusual Stripe activity.
- **Recover** is held back by F4.

## Suggested next cycle

1. Close F1–F4 (see `06`).
2. Build out **Detect**: alerting on password-reset spikes (R1) and anomalous
   charges (R6); centralised auth logging.
3. Re-test and update this mapping — a second pass with the fixes in place turns
   the ⚠️ rows green and demonstrates the audit loop end to end.
