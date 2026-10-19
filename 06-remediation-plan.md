# 06 — Remediation Plan

The three High findings were all **low-effort** — the mechanisms existed or were
one toggle away — and were closed the same day the audit ran. Status below.

| Priority | Finding | Action | Status |
|----------|---------|--------|--------|
| 1 | F2 | Enable 2FA on the Vercel account; store backup codes | ✅ Done 2026-10-19 |
| 2 | F3 | Enable 2FA on the email account; set a recovery email | ✅ Done 2026-10-19 |
| 3 | F1 | Enforce 2FA for admin roles (block login without verified second factor); block admin Google path; invalidate old sessions | ✅ Done 2026-10-19 (commit `b456fcf`, verified live) |
| 4 | F1 | Phase in 2FA prompting for all (non-admin) users | ○ Open — planned |
| 5 | F4 | Enable scheduled snapshots / extend retention (plan upgrade, or periodic `pg_dump` to object storage) | ○ Open — the one remaining gap |
| 6 | F4 | Perform & document a test restore; record RTO/RPO | ✅ Done 2026-10-19 (RTO ≈1m56s) |
| 7 | O1 | Tighten sandbox network policy from `allow-all` to an allow-list | ○ Open — low priority |

## Remediation log

A High finding closed with evidence is stronger than one merely reported.

| Date | Finding | Change | Verified by |
|------|---------|--------|-------------|
| 2026-10-19 | F4 | Restore tested — PITR to isolated branch, RTO ≈1m56s, 48 tables / 86 users verified | `evidence/e-neon-restore-*.png` |
| 2026-10-19 | F1 | 2FA enforced on authorization path for admin roles; admin Google sign-in blocked; pre-deploy sessions invalidated | commit `b456fcf`; 3 live bypass tests (password-login, admin-OAuth-blocked, stale-session-refused) all passed |
| 2026-10-19 | F2 | 2FA enabled on the Vercel account | account owner, verified in Vercel security settings |
| 2026-10-19 | F3 | 2FA enabled + recovery email set on the mail-channel account | account owner, verified in account security settings |
| _(open)_ | F4 | Scheduled snapshots / extended retention | — |

> Note on evidence: the "before" (pre-fix) state of F2/F3 is captured in
> `evidence/`. The "after" (enabled) state was verified live by the account owner
> and recorded above; post-fix screenshots were not attached to this public copy.

## Residual risk after remediation

With F1–F3 closed, R1, R3, and R4 drop from **High** toward **Low–Medium**:
account takeover now requires defeating a second factor, not just a password.
R2's confidentiality exposure is unchanged (an access-control / app-security
matter); F4's restore test confirms the **Recover** capability against R2's
availability side, though the 6-hour retention window remains a limitation until
scheduled snapshots are added.

**One residual risk is formally accepted:** the `check-2fa` endpoint allows
2FA-status enumeration of an email (rate-limited, no credential exposed). Low
severity; mitigation costs login UX for marginal gain. See F1 in `05-findings.md`.
