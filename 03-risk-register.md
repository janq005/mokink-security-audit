# 03 — Risk Register

Each risk is written as **cause → consequence** (an arrow), rated on
**likelihood** and **impact**, combined into a **risk level**, matched to the
**control** that should address it, and tagged to a **NIST CSF** function.

### Rating anchors

**Likelihood** — *what has to go wrong, and how hard is that?*
High = one common mistake away · Medium = needs a specific weakness + someone to
find it · Low = needs several things to line up.

**Impact** — *who is hurt, and how much?*
High = money lost, personal data breached, or disclosure required · Medium =
disruption, recoverable · Low = minor, contained.

**Risk level** = the two combined; the higher of the two pulls it up (a rare but
catastrophic event still ranks High).

---

| ID | Risk (cause → consequence) | Likelihood | Impact | Level | Related control | CSF |
|----|----------------------------|------------|--------|-------|-----------------|-----|
| **R1** | SMTP credentials (in Vercel) leaked/compromised → attacker sends password-reset mail as mokink → user account takeover | Medium | High | **High** | 2FA + rotation on mail account; alert on reset volume | Protect |
| **R2** | Database data exposed via leaked connection string or app bug → personal data of tutors & students breached (no fallback) | Medium | High | **High** | Restrict DB access; string server-only; encryption at rest; tested backups | Protect / Recover |
| **R3** | Weak/reused passwords + MFA not enforced → credential stuffing / brute force → account takeover (esp. admin) | High | High | **High** | Enforce MFA; password rules; login rate-limiting | Protect |
| **R4** | Vercel account compromised → **all** secrets + hosting taken at once (single point of failure) | Medium | High | **High** | MFA on Vercel; limit account access; key rotation | Protect |
| **R5** | Google OAuth misconfiguration (redirect URI, client secret) abused → limited auth bypass | Low | Medium | **Low–Med** | Correct OAuth config; protect client secret | Protect |
| **R6** | Stripe API key compromised or webhook spoofed → fraudulent charges / fake payment confirmations | Low | High | **Medium** | Secure + rotate key; verify webhook signatures; monitor charges | Protect / Detect |

---

## Notes on specific ratings

**R2 is really two risks sharing one provider.** The *confidentiality* risk
(data leak) and the *availability* risk (outage, no fallback) are distinct. The
headline risk is confidentiality — personal data of both tutors and students —
which is why impact is High even though a direct Neon breach is unlikely; leaks
usually come via a leaked connection string or an app-level bug, not by breaking
the provider.

**R3 is the top risk.** Credential stuffing is the most common attack on the
public internet, and the admin surface (`app/api/admin/*`: refunds, role changes,
user suspension, bulk email) makes admin takeover catastrophic. Its final
severity depends on whether MFA is *enforced* — resolved in C1/F1.

**R6 is not "no risk".** Nothing has zero risk; the correct framing is "low
residual risk due to a control" — here, that mokink stores no card data and
verifies webhook signatures (both confirmed in `04`).

## Risk theme

Four of six risks are **High**, and R1, R3, R4 share one root: **missing or
unenforced multi-factor authentication across critical accounts and the login
path.** That cluster is the audit's central story.
