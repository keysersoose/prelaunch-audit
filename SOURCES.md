# Sources

Public pages that back the facts and commands in each skill. Some checks come from
hands-on experience shipping web apps; those are marked "field experience" below and have
no single source. Tool flags change, so check the current docs before relying on a flag.

## Plugin format and install

- https://code.claude.com/docs/en/plugins-reference (version pins, directory listing fields, icon)
- https://code.claude.com/docs/en/plugins/marketplace-reference.md (plugin.json version wins over marketplace version)
- https://code.claude.com/docs/en/plugin-marketplaces
- https://code.claude.com/docs/en/plugins/install.md (install from the shell, one-step install, auto-update off for third-party marketplaces)
- https://code.claude.com/docs/en/skills (frontmatter, `${CLAUDE_PLUGIN_ROOT}`)
- https://code.claude.com/docs/en/commands (`/security-review`, `/code-review`)
- https://code.claude.com/docs/en/mcp (`--scope local` is the default and stays out of the repo; `--scope project` writes `.mcp.json`)

## prelaunch-audit (orchestrator)

- OWASP API Security Top 10 (2023): https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- OWASP Top 10 (2025): https://owasp.org/Top10/2025/
- OWASP Top 10 for LLM Applications: https://genai.owasp.org/llm-top-10/
- Semgrep getting started: https://semgrep.dev/docs/getting-started/
- Claude Code settings and permissions: https://code.claude.com/docs/en/settings
- Risk-based gate, verdict meanings, report shape and known-gaps list: field experience.

## secret-scan

- gitleaks (commands, `.gitleaksignore`, pre-commit hook): https://github.com/gitleaks/gitleaks
- gitleaks releases and checksums: https://github.com/gitleaks/gitleaks/releases
- TruffleHog: https://github.com/trufflesecurity/trufflehog
- GNU grep `-o` / `--only-matching` (print only the name, not the value): https://www.gnu.org/software/grep/manual/grep.html#General-Output-Control
- Next.js environment variables (`NEXT_PUBLIC_`): https://nextjs.org/docs/app/guides/environment-variables
- Vite env variables (`VITE_`): https://vite.dev/guide/env-and-mode
- Expo environment variables (`EXPO_PUBLIC_`): https://docs.expo.dev/guides/environment-variables/
- Supabase API keys (publishable, secret, legacy JWT keys): https://supabase.com/docs/guides/api/api-keys
- Google Cloud API keys (restrict by site and by API): https://cloud.google.com/docs/authentication/api-keys
- Removing sensitive data from a repository: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository
- GitHub secret scanning and push protection: https://docs.github.com/en/code-security/secret-scanning/introduction/about-secret-scanning
- Next.js production checklist (`.env*` in `.gitignore`): https://nextjs.org/docs/app/guides/production-checklist

## database-access-audit (Supabase)

- Row level security: https://supabase.com/docs/guides/database/postgres/row-level-security
- RLS disabled in public (lint 0013): https://github.com/supabase/splinter/blob/main/docs/0013_rls_disabled_in_public.md
- Securing your data: https://supabase.com/docs/guides/database/secure-data
- Production checklist: https://supabase.com/docs/guides/security/production-checklist
- Advisors: https://supabase.com/docs/guides/observability/advisors
- CLI reference (`supabase db advisors`, `supabase db dump`, `--remote` in v4): https://supabase.com/docs/reference/cli/introduction
- Database linter: https://supabase.com/docs/guides/database/database-linter
- Security definer views (lint 0010): https://supabase.com/docs/guides/database/database-advisors?lint=0010_security_definer_view
- Storage access control: https://supabase.com/docs/guides/storage/security/access-control
- Backups (Storage files not included): https://supabase.com/docs/guides/platform/backups
- MCP server, read-only mode and `project_ref`: https://supabase.com/docs/guides/ai-tools/mcp
- Custom SMTP (built-in sender is for testing, low limit): https://supabase.com/docs/guides/auth/auth-smtp
- `getClaims()` for server-side JWT checks: https://supabase.com/docs/reference/javascript/auth-getclaims
- Supabase agent skills: https://github.com/supabase/agent-skills and https://supabase.com/blog/supabase-agent-skills
- Auth CAPTCHA: https://supabase.com/docs/guides/auth/auth-captcha
- Auth rate limits: https://supabase.com/docs/guides/auth/rate-limits
- Password security: https://supabase.com/docs/guides/auth/password-security
- SSL enforcement: https://supabase.com/docs/guides/platform/ssl-enforcement
- Going into production: https://supabase.com/docs/guides/deployment/going-into-prod
- Write access missed when only reads are tested (case study): https://wiz.io/blog/exposed-moltbook-database-reveals-millions-of-api-keys
- Research on exposed Supabase tables: https://www.upguard.com/labs/research/everything-everywhere-systemic-data-exposure-in-supabase-apps
- OWASP broken object level authorization: https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/
- OWASP broken access control: https://owasp.org/Top10/A01_2021-Broken_Access_Control/
- Two-account test idea: https://dev.to/roidayan/is-your-vibe-coded-app-leaking-your-users-data-a-10-minute-check-no-jargon-53jb

## firebase-access-audit

- Get started with security rules: https://firebase.google.com/docs/firestore/security/get-started
- Realtime Database security rules: https://firebase.google.com/docs/database/security
- Test rules with the emulator: https://firebase.google.com/docs/firestore/security/test-rules-emulator
- Rules unit tests: https://firebase.google.com/docs/rules/unit-tests
- Firestore REST API: https://firebase.google.com/docs/firestore/use-rest-api
- Verify ID tokens: https://firebase.google.com/docs/auth/admin/verify-id-tokens
- App Check: https://firebase.google.com/docs/app-check
- Vercel Blob (public access): https://vercel.com/docs/vercel-blob
- Amazon S3 blocking public access: https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html

## server-code-review

- OWASP broken function level authorization: https://owasp.org/API-Security/editions/2023/en/0xa5-broken-function-level-authorization/
- OWASP mass assignment cheat sheet: https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html
- OWASP XSS prevention cheat sheet: https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html
- OWASP SQL injection prevention cheat sheet: https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html
- OWASP file upload cheat sheet (magic numbers, type allow-list): https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
- File signatures (magic numbers) for PNG, JPEG, PDF, ZIP and more: https://en.wikipedia.org/wiki/List_of_file_signatures
- OWASP unvalidated redirects cheat sheet: https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html
- OWASP CSRF prevention cheat sheet: https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
- Next.js server actions security (Origin and Host compared): https://nextjs.org/docs/app/guides/data-security
- Next.js middleware bypass advisory (CVE-2025-29927): https://github.com/vercel/next.js/security/advisories/GHSA-f82v-jwr5-mffw
- MDN CORS: https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
- Supabase Edge Functions `verify_jwt`: https://supabase.com/docs/guides/functions/function-configuration
- Firebase callable functions (`request.auth`): https://firebase.google.com/docs/functions/callable
- OWASP SSRF prevention cheat sheet: https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
- OWASP error handling cheat sheet: https://cheatsheetseries.owasp.org/cheatsheets/Error_Handling_Cheat_Sheet.html
- OWASP logging cheat sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- Firebase session management and revocation: https://firebase.google.com/docs/auth/admin/manage-sessions
- Vercel function limits (request body size): https://vercel.com/docs/functions/limitations
- MDN Cache-Control: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control
- GNU find `-prune`: https://www.gnu.org/software/findutils/manual/html_mono/find.html#Directories
- npm audit: https://docs.npmjs.com/cli/v10/commands/npm-audit
- Semgrep CLI and install: https://semgrep.dev/docs/getting-started/cli
- Semgrep metrics (`--config auto` needs metrics; `--metrics=off`): https://semgrep.dev/docs/metrics
- CodeQL: https://codeql.github.com/docs/
- Six-app review of AI-built code (small sample): https://dev.to/glitchbound/after-the-lovable-data-exposure-study-what-i-found-in-the-code-of-six-apps-43a2
- Clean switch-off for missing settings, trimming pasted env values: field experience.

## payments-and-abuse-review

- Stripe webhooks (signature, raw body, duplicates, order): https://docs.stripe.com/webhooks
- Stripe idempotent requests: https://docs.stripe.com/api/idempotent_requests
- Vercel securing cron jobs (`CRON_SECRET`): https://vercel.com/docs/cron-jobs/manage-cron-jobs#securing-cron-jobs
- OWASP unrestricted resource consumption: https://owasp.org/API-Security/editions/2023/en/0xa4-unrestricted-resource-consumption/
- Twilio Verify toll fraud (SMS pumping) prevention: https://www.twilio.com/docs/verify/preventing-toll-fraud
- Firebase phone sign-in and SMS region policy: https://firebase.google.com/docs/auth/web/phone-auth
- Google Cloud budgets only alert, they do not cap: https://cloud.google.com/billing/docs/how-to/budgets
- OWASP Top 10 for LLM Applications (prompt injection, output handling): https://genai.owasp.org/llm-top-10/
- Database-backed rate limits on serverless, one signing key per purpose: field experience.

## deploy-checklist

- Vercel environment variables: https://vercel.com/docs/environment-variables
- Netlify CLI `env:list` prints values unless filtered: https://cli.netlify.com/commands/env/
- Vercel build command: https://vercel.com/docs/project-configuration#buildcommand
- Vercel deployment protection (Standard Protection on all plans, Password Protection paid): https://vercel.com/docs/deployment-protection
- Vercel instant rollback: https://vercel.com/docs/instant-rollback
- Vercel production checklist: https://vercel.com/docs/production-checklist
- Vercel Hobby plan is for non-commercial use: https://vercel.com/docs/plans/hobby
- Supabase Free projects pause when inactive: https://supabase.com/docs/guides/platform/billing-on-supabase
- Firebase pricing plans and Spark quotas: https://firebase.google.com/pricing
- Netlify production checklist: https://docs.netlify.com/resources/checklists/production-checklist
- Next.js content security policy: https://nextjs.org/docs/app/guides/content-security-policy
- MDN CSP: https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP
- MDN Cross-Origin-Opener-Policy (popup sign-in): https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Opener-Policy
- Vite `public/` folder is served as-is: https://vite.dev/guide/assets#the-public-directory
- curl `-w` write-out: https://curl.se/docs/manpage.html#-w
- Sentry for Next.js: https://docs.sentry.io/platforms/javascript/guides/nextjs/
- Supabase backups (Pro 7 days daily, Storage not included): https://supabase.com/docs/guides/platform/backups
- Firestore export and import: https://firebase.google.com/docs/firestore/manage-data/export-import
- Firestore indexes: https://firebase.google.com/docs/firestore/query-data/indexing
- Google email sender rules (SPF, DKIM, DMARC): https://support.google.com/mail/answer/81126
- Stripe go-live checklist (live keys, live webhook endpoint): https://docs.stripe.com/get-started/checklist/go-live
- ZAP baseline scan (passive, staging only): https://www.zaproxy.org/docs/docker/baseline-scan/
- One check command with a quick mode, clean-env tests, hidden newline in pasted secrets, demo flags ignored in production, uptime alert owner: field experience.

## ui-launch-check

- Node ESM resolves bare imports from the script's own folder: https://nodejs.org/api/esm.html#resolution-algorithm
- Playwright emulation (viewports, touch): https://playwright.dev/docs/emulation
- Playwright accessibility testing with axe: https://playwright.dev/docs/accessibility-testing
- Playwright mocking: https://playwright.dev/docs/mock
- WCAG 2.2 target size minimum (2.5.8): https://www.w3.org/WAI/WCAG22/Understanding/target-size-minimum.html
- WCAG 2.2 focus visible: https://www.w3.org/WAI/WCAG22/Understanding/focus-visible.html
- MDN scrollWidth: https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollWidth
- MDN clientWidth: https://developer.mozilla.org/en-US/docs/Web/API/Element/clientWidth
- MDN SecurityPolicyViolationEvent: https://developer.mozilla.org/en-US/docs/Web/API/SecurityPolicyViolationEvent
- axe-core: https://github.com/dequelabs/axe-core
- Google: block indexing with noindex: https://developers.google.com/search/docs/crawling-indexing/block-indexing
- Google: robots.txt introduction: https://developers.google.com/search/docs/crawling-indexing/robots/intro
- Lighthouse CLI: https://github.com/GoogleChrome/lighthouse
- linkinator: https://github.com/JustinBeckwith/linkinator
- Testing the production build and freeing stale ports: field experience.
