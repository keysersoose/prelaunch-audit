---
name: server-code-review
description: Review a web app's server code route by route - a real login check on every API route, server action and serverless function (not only in middleware), user id from the verified token, ids in URLs (IDOR), input validation, XSS and SQL injection, uploads, sign-in redirects and cookies, CORS, SSRF on outbound fetch, error messages that leak internals, and dependencies. Use when someone says "review my API routes", "check my backend", "are my endpoints protected", "check my server actions", "check my edge functions", "check CORS" or "code review before launch".
---

# Server code review

Goal: every server entry point answers "who is calling, and are they allowed?" before it
does anything, and never trusts what the browser sends. Admin keys (Supabase secret key,
Firebase Admin SDK) skip database rules, so one route that forgets a check exposes
everything. This is the most common serious leak in AI-built apps.

Read-only. Write each answer as **pass** or **fail** with `file:line`. If you could not
verify something, write **unverified**, never pass. Webhooks, cron jobs, rate limits and
AI features are in the `payments-and-abuse-review` skill.

## Step 1. List every entry point

```bash
# Next.js App Router routes, Pages Router API routes, server actions, middleware
find . \( -path ./node_modules -o -path ./.next -o -path ./dist -o -path ./out \) -prune -o \
  \( -name 'route.ts' -o -name 'route.js' -o -path '*/pages/api/*' \) -type f -print
grep -rlE "^['\"]use server['\"]" --include='*.ts' --include='*.tsx' --include='*.js' . | grep -v node_modules
ls middleware.* src/middleware.* 2>/dev/null
# Supabase Edge Functions, Firebase Functions
ls supabase/functions 2>/dev/null; ls functions/src 2>/dev/null
```

If the review is for a change, start from the diff: `git diff --stat main...HEAD` and
`git log --oneline main..HEAD`, and review the changed routes first.

## Step 2. Auth on every entry point

For each entry point, confirm the identity check is on the first lines, before any
database call, file parsing or paid outside API call, and that it returns 401 when missing.

Quick filter (add the helper names this app uses):

```bash
{ find . \( -path ./node_modules -o -path ./.next -o -path ./dist -o -path ./out \) -prune -o \
    \( -name 'route.ts' -o -name 'route.js' -o -path '*/pages/api/*' \) -type f -print
  grep -rlE "^['\"]use server['\"]" --include='*.ts' --include='*.tsx' --include='*.js' . | grep -v node_modules
  find supabase/functions functions/src -type f \( -name '*.ts' -o -name '*.js' \) 2>/dev/null; } | sort -u |
while IFS= read -r f; do
  grep -qE 'getUser|getClaims|verifyIdToken|auth\(|getServerSession|currentUser|context\.auth|request\.auth' "$f" || echo "NO AUTH CALL: $f"
done
grep -nB2 'verify_jwt *= *false' supabase/config.toml 2>/dev/null
grep -rnE "onRequest\(|invoker: *['\"]public" functions/src 2>/dev/null
```

Every file printed must be meant to be public, or be a machine route with its own check.
Server actions (`'use server'`) are public endpoints too; each needs its own check.
A Supabase Edge Function with `verify_jwt = false` accepts calls from anyone: it must check
the caller itself, or be a webhook with a signature check. A Firebase `onRequest` function
is a public URL. An `onCall` function gives you `request.auth`, but you must still refuse
when it is empty.

**Middleware is not enough.** Next.js versions before 15.2.3, 14.2.25, 13.5.9 and 12.3.5
let an attacker skip middleware with an `x-middleware-subrequest` header (check the
advisory for your version; `npm ls next` shows yours). Even on a fixed version, every
route and server action must check login itself, not only `middleware.ts`.

Token checks must be real verification, not decoding: Firebase `verifyIdToken`, Supabase
`supabase.auth.getClaims()` or `getUser(jwt)` on the server (not a cookie session read
alone), or a JWT package that checks signature, issuer, audience and expiry. Trust an
email for admin lists only if `email_verified` is true.

Admin routes: admin status comes from the verified token plus an allow-list kept in a
server-only env variable. An empty list must mean "no admins", not "everyone". Hiding a
button is not security. Deleted or banned accounts keep valid tokens until they expire
(about an hour for Firebase); sensitive routes should re-check account state (Firebase
`verifyIdToken(token, true)` checks revocation).

## Step 3. Never trust ids, roles or money from the request (IDOR)

```bash
grep -rnE '(body|json|data|input|params|searchParams|query)\??\.(uid|userId|user_id|owner|ownerId|role|isAdmin|plan|price|amount|credits|balance)' \
  --include='*.ts' --include='*.tsx' --include='*.js' . | grep -v node_modules
grep -rnE "params\.(id|slug)|searchParams\.get\(" --include='*.ts' --include='*.tsx' . | grep -v node_modules
```

Each hit in the first list is a fail unless the value is only used after an admin check.
The user id comes from the verified token. Prices are computed on the server from a
product id. Ids in the URL are the easiest attack (IDOR: change `/invoice/123` to
`/invoice/124`), so confirm every page and GET route that reads by id also filters by the
owner. Answer "missing" and "not yours" the same way (404). Random ids (UUIDs) make
guessing slow, but they are not access control. Signed file URLs should expire in
minutes, not years. Outside-in test: with user A's token, open user B's URL on each of
these routes; expect 404.

## Step 4. Input validation and injection

- Parse safely: `await req.json().catch(() => null)`, then reject null, arrays and non-objects.
- Check each field's type, length and range (zod or valibot keeps this short).
- Build database writes field by field (no mass assignment). Find spreads into writes:
  `grep -rnE '\.\.\.(body|data|input|payload)' --include='*.ts' . | grep -v node_modules`
- Check ids against a strict pattern before using them in a document path or key; a
  slash in an id can point to a different document.
- Cap body size by `content-length` before reading.

Injection (user text that becomes code):

```bash
grep -rnE "dangerouslySetInnerHTML|innerHTML *=|v-html|\{@html|eval\(|new Function\(" --include='*.ts' --include='*.tsx' --include='*.js' --include='*.jsx' --include='*.vue' --include='*.svelte' . | grep -v node_modules
grep -rnE 'queryRawUnsafe|executeRawUnsafe|sql\.raw|\.or\(`|\.filter\(`|query\(`' --include='*.ts' --include='*.js' . | grep -v node_modules
```

A hit that takes text from a user or a model fails unless that text is cleaned with
DOMPurify (for HTML, to stop XSS) or passed as a query parameter (for SQL). Supabase
`.or()` and `.filter()` strings built from user input can change what the filter matches,
so validate the value first.

## Step 5. File uploads

- Reject early by size (know your host's request body limit; check the current docs).
- Judge the type by the first bytes (magic numbers), not the file name or the browser's
  type, and accept only the types you need.
- If you accept images, decode with a pixel limit and re-encode before storing; refuse SVG
  unless sanitised.
- The server builds the storage key from the user id and a random id; never use a path or
  file name from the form.
- Test: a renamed file, a 0-byte file and an oversize file must all be refused.

## Step 6. Sign-in details

- Redirect targets after sign-in (`next`, `redirect`, `returnTo`, `callbackUrl`,
  `redirect_to`) must be a path on your own site: they start with `/` and not `//`.
  Otherwise phishing links can use your domain (open redirect).
- Session cookies: `HttpOnly`, `Secure`, `SameSite=Lax` or `Strict`. Check the
  `Set-Cookie` header after a real sign-in (browser dev tools, Network tab).
- Custom `route.ts` POST handlers that rely on cookies (not a bearer token) must check
  that the `Origin` header is your own site (CSRF). Next.js server actions already compare
  Origin and Host.
- Changing the email or password, deleting the account, or adding a payment method needs
  a recent sign-in.

## Step 7. CORS (which other websites may call your API)

```bash
grep -rnE "Access-Control-Allow-Origin|cors\(" --include='*.ts' --include='*.js' . | grep -v node_modules
S=https://your-staging-site.example
curl -sI -X OPTIONS "$S/api/<route>" -H 'Origin: https://evil.example' -H 'Access-Control-Request-Method: POST' | grep -i 'access-control'
```

Fail (Critical) if the reply echoes `https://evil.example` together with
`Access-Control-Allow-Credentials: true`: any site could act as your signed-in user.
`Access-Control-Allow-Origin: *` is fine only for public data that needs no login. For
routes that use cookies or tokens, list your own domains. AI-made Supabase Edge Functions
often copy `'Access-Control-Allow-Origin': '*'`; that does not check who is calling, so
the login check must still be there.

## Step 8. Outbound requests (SSRF)

```bash
grep -rn 'fetch(' --include='*.ts' --include='*.js' . | grep -v node_modules
```

Trace each URL. It must come from config or a fixed allow-list, never from request data,
a webhook body or a user-writable database field. Pass `redirect: 'error'` (or `'manual'`),
`cache: 'no-store'` and `signal: AbortSignal.timeout(10000)`. If user URLs are truly
needed, resolve DNS and block private ranges (127.0.0.0/8, 10.0.0.0/8, 172.16.0.0/12,
192.168.0.0/16, 169.254.0.0/16) and re-check after redirects.

## Step 9. Errors, logs and caching

- One error shape for every route, `Cache-Control: no-store`, and the right status
  (400, 401, 403, 404, 409, 413, 429 with `Retry-After`, 5xx). Never 200 on failure.
- No stack traces or raw provider text in responses:
  `grep -rn 'err.message\|error.message' --include='*.ts' . | grep -i 'json(' | grep -v node_modules`
- Logs hold an internal id only, never emails, tokens, passwords or full bodies:
  `grep -rnE 'console\.(log|error)\(.*(email|token|password|authorization|body)' --include='*.ts' . | grep -v node_modules`
- Per-user responses are `no-store` (and `dynamic = 'force-dynamic'` in Next.js) so a CDN
  never serves one person's data to another.
- A missing setting turns the feature off cleanly (503 with a clear code), and env values
  are `.trim()`-ed (pasted secrets often end with a newline).

## Step 10. Dependencies and a static analyser

```bash
npm audit --omit=dev --audit-level=high
```

Fix high and critical findings (`npm audit fix`, or bump the package; `npm ls <pkg>`
shows who pulls it in). Avoid `--force`. Read new package names for typosquats (fake
packages with names close to real ones). For high-risk apps, ask to run Semgrep (`brew
install semgrep` or `python3 -m pip install semgrep`), then
`semgrep scan --config p/default --metrics=off` (fixed public rules, nothing sent to
Semgrep), or CodeQL. Claude Code's `/security-review` covers branch changes only.

## Severity guide

| Finding | Severity |
|---------|----------|
| Route, server action or function reads or changes data with no auth check | Critical |
| Login checked only in middleware on an affected Next.js version | Critical |
| User id, role or price taken from the request | Critical |
| Changing an id in the URL shows another user's data | Critical |
| SQL built from user text | Critical |
| Reflected origin with credentials allowed (CORS) | Critical |
| User text rendered as raw HTML | High (Critical if admins view it) |
| Wildcard CORS on a route that uses cookies | High |
| Per-user response cacheable by a CDN | High |
| Admin route checked only in the browser | High |
| SSRF: server fetches a user-supplied URL | High |
| Upload trusts name or type, no size limit | High |
| High or critical `npm audit` finding in runtime deps | High |
| Open redirect after sign-in | Medium |
| Cookie POST route with no Origin check | Medium |
| Stack traces or provider errors in responses | Medium |
| Personal data or tokens in logs | Medium |

## What to report

- The list of entry points with pass, fail or unverified for each question, with `file:line`.
- The outside-in check (no token, wrong user, admin) for the riskiest routes:
  `curl -s -o /dev/null -w '%{http_code}\n' -X POST https://your-site.example/api/<route>`
  (expect 401 with no token, 403 for a normal user on admin routes, 404 for another
  user's id).
- `npm audit` summary and any static analyser results.
- Items for a second human reviewer if the app handles payments, sign-in or uploads.
