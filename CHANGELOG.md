# Changelog

## 0.1.0 - 2026-11-27

Initial release.

- `prelaunch-audit` orchestrator: stack discovery (AI SDKs included), safety rules, risk
  level, one findings table, go / no-go gate with plain meanings for each verdict,
  plain-English summary, report kept out of git.
- `secret-scan`: gitleaks history and folder scans, env files, public-prefix variables
  (names only), AI, email and SMS key shapes in the built bundle and the live site
  (Next.js and Vite), restricting Google browser keys, rotate-first leak response.
- `database-access-audit`: Supabase RLS, policies, grants, keys, storage, advisors, custom
  SMTP; logged-out write test; two-account isolation test.
- `firebase-access-audit`: Firestore and Realtime Database rules, Admin SDK, emulator
  tests, storage rules, App Check, open bucket checks.
- `server-code-review`: auth on routes, server actions and serverless functions, the
  middleware-only trap, IDOR, validation, XSS and SQL injection, uploads, sign-in
  redirects and cookies, CORS, SSRF, errors, logs, caching, dependency audit.
- `payments-and-abuse-review`: webhooks, cron, signing keys, database-backed rate limits,
  SMS pumping, budgets that only alert, AI features and prompt injection.
- `deploy-checklist`: env vars per environment, live payment keys, preview protection,
  tests in build, headers, leftover admin/debug routes and exposed files, uptime alerts,
  plan limits, backups, rollback, email DNS, privacy policy.
- `ui-launch-check`: tools in a temp folder, multi-size screenshots, overflow and
  tap-size checks, axe-core, keyboard focus, CSP errors, meta tags, search engine
  settings, links, Lighthouse.
- `GLOSSARY.md`: everyday meanings for the technical words.
