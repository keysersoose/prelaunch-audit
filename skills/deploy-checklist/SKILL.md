---
name: deploy-checklist
description: Check a web app's hosting and launch setup - env vars per environment, live payment keys, preview vs production separation, tests in the build, security headers, HTTPS, leftover admin/debug routes and exposed files, error and uptime monitoring, free-plan limits, backups, a rollback plan, email DNS, privacy policy - and run outside-in curl checks against the live URL. Use when someone says "deploy checklist", "am I ready to go live", "check my Vercel setup", "check my Netlify setup", "production checklist", "check my headers" or "launch day checks".
---

# Deploy checklist

Goal: "merged", "deployed" and "working" are three different states. This module checks
the platform settings and then proves the live site behaves, from outside.

Read-only. Do not change env vars, promote, roll back or deploy. Print env var NAMES only;
never print values. Do not run `vercel env pull`.

Set the target once: `S=https://your-site.example` (production) and, if there is one,
`P=https://your-preview-url.example` (preview or staging).

## Step 1. One command for every pre-ship check

Look for a single entry point (`npm run check` or `scripts/check.sh`). If there is none,
recommend one (do not create it without approval):

```bash
#!/usr/bin/env bash
set -euo pipefail
section() { echo; echo "--- $1"; }
section types;   npx tsc --noEmit
section tests;   env -i PATH="$PATH" HOME="$HOME" npm test   # clean env, like the host build
section audit;   npm audit --omit=dev --audit-level=high
[ "${QUICK:-0}" = "1" ] && exit 0
section secrets; if command -v gitleaks >/dev/null; then gitleaks git --redact .; else echo "skipped: gitleaks not installed"; fi
section build;   npm run build
```

Fast stages first. `QUICK=1` runs only those for small changes; run everything before
merging. Running tests with `env -i` catches tests that pass only because a real secret is
in your shell.

## Step 2. Tests block the deploy

Check the build command on the host. Tests should run first, for example in `vercel.json`:
`{ "buildCommand": "npm test && next build" }`. Then a failing test leaves the last good
version live. Check the lockfile is committed and CI uses `npm ci`.

## Step 3. Environment variables per environment

```bash
vercel env ls      # Vercel: shows names, not values
netlify env:list --json | node -e 'let s="";process.stdin.on("data",d=>s+=d).on("end",()=>console.log(Object.keys(JSON.parse(s)).join("\n")))'   # Netlify: keys only
grep -rhoE 'process\.env\.[A-Z_0-9]+' --include='*.ts' --include='*.tsx' --include='*.js' . \
  | grep -v node_modules | sort -u
```

Plain `netlify env:list` prints values, so always use the `--json` plus keys filter.

- Every variable the code reads is set in Production (and Preview if used).
- Preview must not use production database keys, payment live keys or production webhook
  and callback URLs. Give Preview its own test project and sandbox keys.
- Production must use live payment keys and the live webhook secret. In the Stripe
  dashboard (live mode), check that a webhook endpoint exists for the production URL and
  recent deliveries succeeded. Check by key name and prefix only; never print a value.
- Site URL, auth domain and webhook URL values differ per environment.
- Secrets are marked Sensitive in the host dashboard.
- Look for names containing TEST, E2E, DEBUG, DEMO or BYPASS set in Production. Demo and
  test switches should be ignored by code in production (check `VERCEL_ENV === 'production'`
  or the host's equivalent), not by habit.
- Public-prefix values (`NEXT_PUBLIC_`, `VITE_`) are frozen into the build. Any env change
  needs a redeploy.
- Values added with `echo` carry a hidden newline. Use `printf '%s' "$VALUE" | vercel env add NAME production`.

## Step 4. Preview protection

Vercel: open Settings > Deployment Protection and record the scope (Standard Protection,
All Deployments or None) and the method. Standard Protection with Vercel Authentication is
free on all plans; Password Protection is a paid Pro add-on. Netlify: deploy previews are
public unless you turn on password or team-only protection. Flag None or Disabled as
Medium when previews use real data.

## Step 5. Security headers and HTTPS

```bash
curl -sI "$S/" | grep -iE 'content-security-policy|strict-transport-security|x-content-type-options|x-frame-options|referrer-policy|permissions-policy|cross-origin-opener-policy'
curl -sI "http://${S#https://}/" | head -3          # expect 301 or 308 to https
curl -s -o /dev/null -w '%{http_code}\n' "$S/does-not-exist-$(date +%s)"   # expect 404
```

Expect at least: `X-Content-Type-Options: nosniff`, a `Referrer-Policy`, frame protection
(`frame-ancestors 'none'` in CSP or `X-Frame-Options: DENY`), and HSTS (always use https).
Start HSTS without `includeSubDomains` until you know every subdomain serves HTTPS. If
`script-src` has `'unsafe-inline'`, record it as a known gap and plan nonces or hashes.
After any header change, sign in on the real domain and check the console for CSP errors;
popup sign-in may need `Cross-Origin-Opener-Policy: same-origin-allow-popups`.

## Step 6. Outside-in checks on the live site

```bash
for p in / /login /privacy /terms; do
  printf '%-14s %s\n' "$p" "$(curl -s -o /dev/null -w '%{http_code}' "$S$p")"; done
# protected routes must refuse anonymous callers (use the app's real route names)
for p in /api/<private-route-1> /api/<admin-route>; do
  printf '%-30s %s\n' "$p" "$(curl -s -o /dev/null -w '%{http_code}' -X POST "$S$p")"; done
# leftover admin, debug and seed routes, and files that must never be served
for p in /admin /api/admin /dashboard /api/debug /api/test /api/seed /api/reset /setup /install /.env /.env.local /.env.production /.git/config /config.json /backup.sql /dump.sql /storybook; do
  printf '%-24s %s\n' "$p" "$(curl -s -o /dev/null -w '%{http_code}' "$S$p")"; done
grep -rniE 'admin@|password123|changeme|default_password' supabase/seed.sql prisma/seed.* scripts/ 2>/dev/null
```

Expected: public pages 200, private routes 401 (or 403, 405). Leftover routes and files:
401, 403, 404, or a redirect to sign-in. A 200 on `/.env`, `/.git/config` or a `.sql`
file, or on a debug, seed or reset route, is Critical. A 200 on `/admin` is fine only if it
shows a sign-in screen and its data calls refuse anonymous visitors. In Vite, every file in
`public/` is served as-is. A seeded admin with a known password, or a test user that also
exists in production, is Critical. With a real non-admin test token, admin routes return
403. Save the table with the date; a check nobody can re-run is only an opinion.

Also: sign in once on the real domain (a `redirect_uri_mismatch` means the host is missing
from the OAuth client), do one real end-to-end action as a test user, and the next morning
confirm scheduled jobs returned 200 in the logs.

## Step 7. Monitoring, alerts and plan limits

- Error monitoring (for example Sentry) installed and tested with one deliberate test error.
- An uptime monitor (any free one is fine) checks the home page and one API health route
  every few minutes and alerts a named person by email or phone. Error monitoring only sees
  errors from code that runs, so it cannot tell you the site is down. Write down who gets
  each alert.
- Runtime logs readable (`vercel logs <deployment-url>`); read a day of real logs once and
  look for personal data or tokens.
- Spend alerts on at the host and at every paid API provider.
- Plan limits: Vercel Hobby is for non-commercial use, Supabase Free projects pause after a
  period with no activity, and Firebase Spark has daily quotas (check the current terms).
  A paying-customer launch on a free plan can stop without warning.

## Step 8. Backups and data changes

- Backups exist and someone has restored one once. Supabase: Pro gets 7 days of daily
  backups (PITR is an add-on); Free plan: export it yourself with
  `supabase db dump -f backup.sql` (or `--data-only -f data.sql`; on CLI v4 use `--remote`
  instead of `--linked`/`--db-url`). Storage objects are not included. Firestore: scheduled
  backups or `gcloud firestore export gs://<bucket>/backup`.
- Schema and data changes are additive (expand, migrate, contract: add the new, move the
  data, remove the old later), never a rename or delete in the same release as the code.
  Code rolls back in seconds; data does not.
- Database rules and indexes deploy separately from the app. Deploy a new index and wait
  for it to build before releasing code that queries it.

## Step 9. Rollback plan

Ask the owner to describe it, and check it was practised once:

- Vercel: Deployments, pick a known-good one, Instant Rollback (or `vercel rollback`).
  After a rollback, new pushes do not go live until a deployment is promoted again
  (`vercel promote <url>`).
- A rollback does not undo database changes or a bad env var (put the old value back and
  redeploy). Then fix in git with `git revert <sha>`.

## Step 10. Email DNS and privacy policy

```bash
dig +short TXT example.com | grep -i spf            # Windows: nslookup -type=TXT example.com
dig +short TXT _dmarc.example.com
dig +short TXT <selector>._domainkey.example.com    # DKIM selector from the email provider
```

If the app sends email: SPF and DKIM at least, DMARC recommended (Google requires all
three for bulk senders), and link tracking off for sign-in emails.

If the app collects emails (sign-up, waitlist, newsletter, contact form), it needs a
privacy policy linked from the form and from the footer. Google sign-in, Stripe and app
stores also ask for one. The policy should name the outside services that receive user
data (hosting, database, email, analytics, payments, AI providers) and say how a user can
delete their account and data. Marketing email needs a working unsubscribe link. Check
that the policy matches what the code really does. This is not legal advice. A cookie
banner may be needed if analytics or ad cookies are used (rules differ by country).
Search settings (robots.txt, sitemap, noindex) are checked in the ui-launch-check skill.

## Step 11. Agents and production

If an AI agent works in this repo, production commands (`vercel --prod`, `vercel env`,
`vercel rollback`, `vercel promote`, database deploys) should need human confirmation,
for example in the `permissions.ask` list of `.claude/settings.json`. Agents should use a
low-privilege token, never the owner's.

## Severity guide

| Finding | Severity |
|---------|----------|
| Preview uses production database or payment keys | Critical |
| Secret file, git folder or debug/seed route reachable on the live site | Critical |
| Destructive data change shipped with the code, no backup | Critical |
| Protected API route returns 200 to an anonymous caller | Critical |
| Production uses test payment keys | High |
| App collects emails or personal data and has no privacy policy | High |
| Test or bypass flag active in production | High |
| No tested backup, or no rollback plan | High |
| Tests do not block the deploy | High |
| Missing HSTS, nosniff or frame protection | Medium |
| Previews unprotected while using real data | Medium |
| No error monitoring, spend alerts, or uptime alert to a named person | Medium |
| No SPF/DKIM for a sending domain | Medium |

## What to report

- Env var names per environment (no values) and any cross-environment reuse.
- The headers found and the ones missing.
- The outside-in table: URL, expected code, actual code, date.
- Monitoring, uptime, backup and rollback status, each with "checked" or "not checked".
- DNS results for the sending domain.
