# 01 — Audit Scope

## System under audit

**mokink.lt** (codebase: *TavoMokytojas*) is a live tutoring marketplace.
Students find tutors, book sessions, and pay through the platform. Tutors manage
profiles, availability, and subscriptions. The platform is built on Next.js /
TypeScript, with a PostgreSQL database hosted on Neon, payments via Stripe, and
hosting on Vercel.

## Critical or important function

> **Booking and payment.**

If matching or payment processing fails or is compromised, the platform loses its
purpose, its revenue, and user trust. This is the "critical or important
function" the rest of the audit is oriented around — controls and risks are
prioritised by their effect on it.

## In scope

| Asset | Description |
|-------|-------------|
| Next.js application | The mokink.lt web app and its API routes |
| Neon (PostgreSQL) | Production database — personal data of tutors and students, bookings |
| GitHub repository | Source control for the application |
| Vercel | Hosting and environment-variable / secrets store |
| Email + password auth | In-house primary login path (`lib/auth-authorize.ts`, TOTP support) |
| Google OAuth | Secondary external login path |
| Stripe | Payment processing |
| Hostinger SMTP (`lib/email.ts`) | Transactional + marketing email, incl. password-reset mail |

## Out of scope — and why

Stated explicitly, because defining the boundary is part of the audit and
reflects the **cloud shared-responsibility model**: the provider secures the
platform, the customer secures what they put on it and how they configure it.

| Excluded | Reason |
|----------|--------|
| Stripe's internal security / PCI compliance | Provider responsibility; mokink stores no card data (see C-tests) |
| Google's OAuth infrastructure | Provider responsibility |
| Neon / Vercel / Hostinger physical & platform security | Provider responsibility under shared-responsibility model |
| Penetration testing of third-party services | Not authorised; out of a self-audit's remit |

## Approach

1. **Identify** — inventory assets and third-party providers (`02`).
2. **Assess** — enumerate risks, rate likelihood × impact, map to NIST CSF (`03`).
3. **Test** — check 8 controls against the live system, collect evidence, rate
   design vs. operating effectiveness (`04`).
4. **Report** — write findings, severity, and a prioritised remediation plan
   (`05`, `06`).

Evidence is drawn from: the live Vercel, Neon, and email-account dashboards;
and read-only inspection of the application source (`git log`, `grep`, file
reads). No changes were made to production during evidence collection.
