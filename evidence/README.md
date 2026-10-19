# Evidence

Screenshots and command output referenced by the control tests (`../04-control-tests.md`).

| File | Supports | Shows |
|------|----------|-------|
| `e-vercel-2fa-off.png` | C2 / F2 | Vercel 2FA setup prompt with "Skip" — 2FA not enabled |
| `e-email-2fa-off.png` | C3 / F3 | Email account — "Dvigubos autentifikacijos: Išjungta", no recovery email |
| `e-neon-backup-6h.png` | C7 / F4 | Neon Backup & Restore — 6-hour window, no snapshot schedule |
| `e-neon-restore-3-branch.png` | C7 / F4 | Restore test — branch `restore-test-2026-10-19` created from production |
| `e-neon-restore-1-tables.png` | C7 / F4 | Restore test — 48 tables present on the restored branch |
| `e-neon-restore-2-data.png` | C7 / F4 | Restore test — `users` = 86 rows (verified on restored branch) |
| `e-vercel-sandbox-allow-all.png` | O1 | v0 Sandbox Network Policy set to `allow-all` |

## Command-output evidence (recorded inline in `04-control-tests.md`)

- **C4** — `git log --all --name-only` (no committed env files) and
  `git grep` across `git rev-list --all` (all matches are placeholders / dummies / docs).
- **C5** — `grep` hit on `stripe.webhooks.constructEvent` at
  `app/api/webhooks/stripe/route.ts:86–89`.
- **C6** — `grep` for `NEXT_PUBLIC_*` database vars returned nothing.
- **C8** — `grep` for `rateLimit` across auth routes incl. `lib/auth-authorize.ts:26`.

> Replace these notes with pasted terminal captures / screenshots of each command
> run if you want the evidence fully self-contained in the repo.
