---
name: secret-scan
description: Find leaked API keys and passwords in a web app's git history, working tree and built browser bundle (including AI, email and SMS provider keys), and explain what to do when one has leaked (rotate first, then clean up). Use when someone says "scan for secrets", "did I leak a key", "check for API keys in my repo", "is my .env committed", "NEXT_PUBLIC secret", "VITE_ key exposed", "OpenAI key in the browser" or "gitleaks".
---

# Secret scan

Goal: prove that no secret value lives in git history, in files on disk, or in the
JavaScript sent to visitors. AI coding tools often paste keys into code, notes and
example configs, and a key that was committed once stays in history, clones and forks.

Rule for this whole skill: **print key NAMES and file locations, never values.**
Always pass `--redact` to scanners. Do not `cat` env files.

## Step 1. Get the real gitleaks (not an npm lookalike)

1. Check if it is installed: `command -v gitleaks && gitleaks version`.
2. If not, ask the owner before installing. Download from the official releases page:
   https://github.com/gitleaks/gitleaks/releases (or `brew install gitleaks` on macOS).
3. Verify the download against the checksums file from the same release:
   `sha256sum -c gitleaks_<version>_checksums.txt --ignore-missing`
   (macOS: `shasum -a 256 -c gitleaks_<version>_checksums.txt --ignore-missing`;
   Windows: `certutil -hashfile <file> SHA256` and compare the result by eye).
4. Keep the binary outside the repo.
5. **Do not run `npx gitleaks`.** It is not the official project. Running an unverified
   package for a security tool can itself be an attack.

## Step 2. Scan git history and the working tree

Run both modes. They find different things.

```bash
# All commits on all branches
gitleaks git --redact -v --log-opts="--all" .
# Files on disk right now, including untracked notes and docs
gitleaks dir --redact -v .
```

Older gitleaks versions (before 8.19) use `gitleaks detect --source . --log-opts="--all" --redact`
for history and `gitleaks protect --staged --redact` for staged changes (check the current docs).

Each hit is a finding. A hit in history is still a leak even if the file is gone today.

Optional second opinion (ask first): TruffleHog, from https://github.com/trufflesecurity/trufflehog.
`trufflehog git file://. --results=verified,unknown`. Its "verify" step tests found keys
against the provider's API, so use it only on code the owner owns.

## Step 3. Check that env files are not tracked

```bash
git ls-files | grep -iE '(^|/)\.env' || echo "no env files tracked"
git check-ignore -v .env .env.local .env.production 2>/dev/null
git ls-files | grep -iE '\.(pem|key|p12|pfx)$|service-account|credentials.*\.json' || echo "no key files tracked"
```

- Only an example file (like `.env.example`) with empty or fake values may be tracked.
  Open it and confirm no real values (read the names, do not print it into the report).
- `.gitignore` must cover `.env*` (with an exception for the example file if needed).

## Step 4. Check public-prefix variables

Variables with these prefixes are copied into the browser bundle: `NEXT_PUBLIC_` (Next.js),
`VITE_` (Vite), `REACT_APP_` (Create React App), `PUBLIC_` (SvelteKit, Astro),
`EXPO_PUBLIC_` (Expo).

```bash
grep -rhoE '(NEXT_PUBLIC|VITE|REACT_APP|PUBLIC|EXPO_PUBLIC)_[A-Z0-9_]+' . \
  --include='*.ts' --include='*.tsx' --include='*.js' --include='*.jsx' --include='.env*' \
  --exclude-dir=node_modules --exclude-dir=.next --exclude-dir=dist | sort -u
# -o prints only the matched NAME, never the value after the '='
grep -rnoE '(NEXT_PUBLIC|VITE|REACT_APP|PUBLIC|EXPO_PUBLIC)_[A-Z0-9_]*(SECRET|SERVICE|PRIVATE|ADMIN|PASSWORD|TOKEN|API_KEY|OPENAI|ANTHROPIC|CLAUDE|GEMINI|GROQ|MISTRAL|REPLICATE|OPENROUTER|ELEVENLABS|SENDGRID|TWILIO|STRIPE_SECRET)[A-Z0-9_]*' . \
  --include='*.ts' --include='*.tsx' --include='*.js' --include='*.jsx' --include='.env*' \
  --exclude-dir=node_modules --exclude-dir=.next --exclude-dir=dist
```

Read the list and ask of each name: "is this meant to be public?" Fine in the browser:
the Firebase web config, the Supabase URL and anon or publishable key, a Stripe
publishable key (`pk_`). Never in the browser: Supabase `service_role` or `sb_secret_` keys,
Firebase Admin service accounts, Stripe secret keys (`sk_`), any provider API secret, any
AI model key (OpenAI, Anthropic, Gemini with its own key, Groq, Replicate, OpenRouter,
ElevenLabs), and email or SMS provider keys. Anyone who finds such a key can run up your
bill. In a Vite or Lovable app with no server, the AI call must move into a server function
(for example a Supabase Edge Function) that holds the key.

The Firebase web key is public by design. But if paid Google APIs (for example the Gemini
API) are turned on in the same Google Cloud project, an unrestricted key may work on them
too. In Google Cloud Console (APIs & Services, Credentials), restrict the browser key to
your domains and to only the APIs the web app needs (check the current docs).

## Step 5. Grep the built output (the build cannot lie)

Source review can miss a server-only import pulled into a client component. The built
files show what visitors really download.

```bash
npm run build
OUT=".next/static"   # Vite: dist   Next static export: out
# 1) Names of your server secrets (fill in the real NAMES from your env list)
grep -rlE "SERVER_SECRET_NAME_1|SERVER_SECRET_NAME_2" $OUT && echo "FOUND" || echo "clean"
# 2) Common key shapes
KEYS='BEGIN [A-Z ]*PRIVATE KEY|"private_key"|service_role|sb_secret_|sk_live_[A-Za-z0-9]{10,}|sk_test_[A-Za-z0-9]{10,}|rk_live_[A-Za-z0-9]{10,}|AKIA[0-9A-Z]{16}|ghp_[A-Za-z0-9]{30,}|sk-proj-[A-Za-z0-9_-]{20,}|sk-ant-[A-Za-z0-9_-]{20,}|sk-or-v1-[A-Za-z0-9]{20,}|gsk_[A-Za-z0-9]{20,}|r8_[A-Za-z0-9]{20,}|SG\.[A-Za-z0-9_-]{20,}|xox[bp]-[A-Za-z0-9-]{10,}'
grep -rlE "$KEYS" $OUT && echo "FOUND" || echo "clean"
```

Only file names are printed (`-l`), not the matching text. A hit means a secret or
server code reached the browser. Expect a few false hits; open each file name and judge it.

Legacy Supabase keys are JWTs (long strings starting `eyJ`), so the word `service_role`
is hidden inside them and grep will not see it. If the bundle contains `eyJ` strings, decode
only the middle part locally and read the `role` field (`anon` is fine, `service_role` is
Critical). Never paste a key into an online decoder.

Also check the live site, since the deployed build can differ (Next.js serves chunks from
`/_next/static`, Vite and Lovable from `/assets`):

```bash
S=https://your-site.example
curl -s "$S/" | grep -oE '(/_next/static|/assets)/[^"]+\.js' | sort -u | head -30 > /tmp/chunks.txt
while read -r p; do curl -s "$S$p" | grep -qE "$KEYS" && echo "HIT $p"; done < /tmp/chunks.txt
```

This only covers chunks linked from the home page. Repeat for `/login` and the main app
page, or rely on the local build grep above for full coverage.

Public source maps (`.map` files) also expose code. In Next.js keep
`productionBrowserSourceMaps` off unless you want the source public.

## Step 6. Check the allowlist

If the repo has `.gitleaksignore` or a gitleaks config with an allowlist:

- Each entry must be a single fingerprint (`file:rule-id:line` from a dir scan, or
  `commit:file:rule-id:line` from a git scan), never a whole path or a whole rule.
- Each entry needs a comment saying why it is not secret (for example "fake value used
  only in a unit test"). Comment support: check the current docs.
- Broad allowlists quietly turn the scanner off. Flag them as High.

## Step 7. If a secret leaked: rotate first, then clean up

Deleting the file or the commit does not help. The value is already in clones, forks,
CI caches and logs. Bots scrape public repos within minutes. Order of work:

1. Create a new key at the provider.
2. Put the new key in the host's environment variables and redeploy.
3. Revoke the old key at the provider.
4. Only then remove it from the repo and history (`git filter-repo`, or GitHub's guide
   on removing sensitive data), and re-run `gitleaks git --redact -v .` to confirm.
5. Check the provider's logs for use of the old key between leak and revocation.

These steps change production, so they need the owner's approval. Report them as the fix.

## Step 8. Keep it from coming back

Suggest (do not install without approval): a pre-commit hook with
`gitleaks git --pre-commit --staged --redact` (check the current docs for your version), the
official gitleaks GitHub Action in CI, and GitHub secret scanning with push protection
turned on in the repo settings.

## Severity guide

| Finding | Severity |
|---------|----------|
| Live secret key in the browser bundle or live site | Critical |
| AI, email or SMS provider key in the browser bundle | Critical |
| Secret in git history of a public repo, not rotated | Critical |
| Secret in git history of a private repo, not rotated | Critical (any unrotated leaked key is NO-GO) |
| Real `.env` or service-account file tracked in git | Critical |
| Secret name with a public prefix (`NEXT_PUBLIC_...SECRET`) | Critical |
| Unrestricted Firebase/Google browser key with paid APIs on in the same project | High |
| Broad path-wide or rule-wide allowlist | High |
| Leaked key already rotated, history not yet cleaned | Medium |
| No pre-commit or CI secret scan | Medium |
| Public source maps not intended | Low |

## What to report

- gitleaks version and the exact commands run, with the count of findings per mode.
- For each hit: rule id, file, line or commit, and whether it is still live (ask the owner).
  Never the value.
- The public-prefix variable list with a yes or no for "meant to be public".
- Bundle grep result (clean or the file names that hit) for the local build and the live site.
- Rotation steps needed, in order, for each live leak.
