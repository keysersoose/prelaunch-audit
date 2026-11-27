---
name: payments-and-abuse-review
description: Review the parts of a web app that strangers can trigger to move money or run up a bill - payment webhooks (signature, replay, duplicates), cron and internal endpoints, signing keys, rate limits, SMS and email OTP abuse (SMS pumping), cloud budgets that only alert, and AI model calls (per-user quotas, max tokens, prompt injection, model output). Use when someone says "check my webhook", "check my Stripe webhook", "rate limits", "can someone run up my bill", "SMS pumping", "check my AI feature", "prompt injection" or "cron security".
---

# Payments and abuse review

Goal: no stranger can grant themselves money or access, and no stranger can make the
owner pay a large bill. These routes are public URLs even when no user ever sees them.

Read-only. Write each answer as **pass**, **fail** or **unverified** with `file:line`.

## Step 1. Find the routes

```bash
grep -rlnE "webhook|stripe|signature|CRON_SECRET|crons" --include='*.ts' --include='*.js' --include='*.json' . | grep -v node_modules
cat vercel.json 2>/dev/null | grep -A5 '"crons"'
grep -rnE "openai|@anthropic-ai|@ai-sdk|@google/genai|generativelanguage|replicate|groq|max_tokens|maxTokens|maxOutputTokens" \
  --include='*.ts' --include='*.tsx' --include='*.js' . | grep -v node_modules
grep -rnE "signInWithPhoneNumber|verifications|sendOtp|signInWithOtp|resetPassword" --include='*.ts' --include='*.tsx' --include='*.js' . | grep -v node_modules
```

## Step 2. Webhooks

1. Read the raw body once (`await req.text()`), verify the provider's signature over the
   raw body with a constant-time compare (`timingSafeEqual`, built into Node), and only
   then `JSON.parse`. Parsing first breaks the signature.
2. Reject stale timestamps if the provider sends one (replay protection).
3. Cap the body size (a small limit that fits the provider's largest real event) and
   return 413 above it.
4. For payments, take only the payment id from the notice, re-read the payment from the
   provider's API, and check status, amount, currency and your own order id before
   granting anything.
5. Idempotency (doing it twice has the same effect as once): write a marker (event id,
   order status) in the same database transaction as the effect, and check it first. A
   replayed or duplicate event must do nothing.

## Step 3. Cron and internal jobs

1. Require a secret header (Vercel sends `Authorization: Bearer <CRON_SECRET>` when that
   variable is set). Return 401 if the env secret is missing or empty, so an empty header
   never matches. Compare in constant time.
2. Jobs must be safe to run twice or late.

Signing keys: use a separate key for each purpose (for example email links and callback
URLs), not one secret for everything. A value signed for one purpose must never verify
for another.

## Step 4. Rate limits on costly or abusable actions

List login, sign-up, contact forms, uploads and anything that calls a paid API. On
serverless, a counter in a JS `Map` resets on every cold start and is not shared, so it
only slows things down. Money and quotas need a counter in the database with a time
window, updated inside a transaction, keyed on account plus a hashed IP. Return 429.

- SMS and phone sign-in: attackers trigger thousands of OTP texts to premium numbers they
  own (SMS pumping), and you pay for every text. Limit OTP sends per phone number, per IP
  and per day. Allow only the countries you serve (Firebase: SMS region policy; Twilio
  Verify: Geo permissions and fraud guard; check the current docs). Put a CAPTCHA before
  any send.
- Email OTP, magic links and password reset: limit sends per address and per IP.
- Set billing alerts and hard caps at every outside API provider. Google Cloud and
  Firebase budgets only send alerts; they do not stop spending. For a hard stop, set API
  quotas or cap usage in your own code.

## Step 5. AI features (if the app calls a language model)

- The key and the call live on the server only. The route checks login first.
- Each call sets a maximum output length (`max_tokens` or the SDK's equivalent). Input
  length is capped. The model name is fixed on the server, never taken from the request.
- Each user has a daily limit kept in the database (Step 4), and the provider account has
  a monthly spend limit.
- Prompt injection (hidden instructions inside text the model reads): user text, uploaded
  files, web pages, emails and database rows can all carry them. If the model can use
  tools (send email, write to the database, fetch a URL, run code), each tool must check
  the user's permission again on the server, touch only that user's data, and ask the
  user before anything that sends, pays or deletes.
- Never put secrets, other users' data or admin-only facts in the system prompt. Assume
  users can make the model repeat it.
- Model output is untrusted. Never put it into `dangerouslySetInnerHTML`, SQL, a shell
  command, `eval`, or a server-side fetch without checks. Markdown images in model output
  can leak data through the image URL, so allow images only from your own domains.

## Step 6. Outside-in checks (staging, small bursts only)

```bash
S=https://your-staging-site.example
curl -s -o /dev/null -w '%{http_code}\n' -H 'authorization: Bearer wrong' "$S/api/<cron-route>"   # expect 401
curl -s -o /dev/null -w '%{http_code}\n' -X POST -d '{}' "$S/api/<webhook-route>"                # expect 400 or 401
for i in $(seq 1 30); do curl -s -o /dev/null -w '%{http_code} ' -X POST "$S/api/<costly-route>"; done; echo   # expect 401 or 429, never 30x 200
```

## Severity guide

| Finding | Severity |
|---------|----------|
| Webhook with no signature check, or access or balance granted from the webhook body alone | Critical |
| Non-idempotent money or balance change | Critical |
| AI route with no login or no per-user limit | High (Critical if the provider has no spend cap) |
| AI tool can read or change other users' data, or send/delete without asking | High |
| Model output rendered as raw HTML | High |
| Cron route with no secret, or open when the secret is unset | High |
| No database-backed limit on paid or abusable actions | High |
| SMS/OTP send with no per-number and per-country limits | High |
| No spend alert or cap at a paid provider | Medium |

## What to report

- Each webhook, cron and paid route with pass, fail or unverified per step, with `file:line`.
- The outside-in results (route, expected, actual, date).
- Which providers have spend caps, which have only alerts, and which have neither.
- Items for a second human reviewer: payment webhooks and any AI tool that can act.
