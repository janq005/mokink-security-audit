# 02 — Third-Party ICT Register

Modelled on the DORA **register of information** (Art. 28). The core question
each row answers: *what do we depend on, what does it touch, how badly would its
failure or compromise hurt us, and what is our fallback?*

Criticality is judgement, not a formula — rated by the cost to the critical
function (booking & payment) if the provider failed or was breached.

| Provider | Service provided | Data it touches | Hosted | Criticality | Fallback |
|----------|------------------|-----------------|--------|-------------|----------|
| **Neon** | Managed PostgreSQL database | Personal data of tutors **and students**; booking records | Neon cloud | **Critical** | None |
| **Stripe** | Payment processing | Cardholder data (held **at Stripe**); mokink stores only payment references/tokens | Stripe cloud | **Critical** | None |
| **Email + password auth** | Primary login path, built in-house | Credentials (email + password hashes); TOTP secrets | mokink / Vercel | **Critical** | None |
| **Hostinger SMTP** (`lib/email.ts`) | Transactional + marketing email, incl. **password-reset & verification mail** | Email content; account-recovery messages | Hostinger / Google acct | **Critical** | None |
| **Vercel** | App hosting + secrets store | Serves the app; holds **all** environment variables and keys | Vercel cloud | **Critical** | None |
| **Google OAuth** | External identity provider; second login path | Authentication only (Google holds credentials) | Google | **Important** | None (email login remains) |

## Reading the register

**Why email auth and Google OAuth are rated differently.** Google OAuth's
security is Google's responsibility; if it is unavailable, the email+password
path still works, so its residual risk to mokink is limited — **Important**, not
Critical. The in-house email+password path is the opposite: **no external
provider secures it**, so it is Critical and carries the highest *owned* risk.

**Why Hostinger SMTP is Critical, not Important.** It was initially a candidate
for "Important" (email delivery). On inspection, **password-reset and
account-verification emails route through the same SMTP sender** (`lib/email.ts`).
That places the email channel inside the authentication chain: compromise enables
password-reset interception and account takeover. The dependency the rating hung
on was resolved before the rating was set.

**Why Vercel's fallback is "None", not "backup deployment".** Vercel is *both*
the host *and* the secrets store. A previous deployment or code backup does not
fall back the **secrets store** or **account access** — if the Vercel account is
compromised or unavailable, hosting and every key go down together. An optimistic
fallback that would not actually work hides risk; "None" states it honestly.

## The register is itself a finding

Six providers, five Critical, **fallback "None" across the board.** That
uniformity is **concentration risk** — mokink is wholly dependent on single
providers with no contingency. Naming it is the point of the register; it feeds
directly into risks R2 and R4.
