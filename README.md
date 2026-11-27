# prelaunch-audit

A Claude Code plugin that checks a web app before launch.

It is made for freelancers and founders with an app built fast, often with AI tools
(Lovable, Bolt, Cursor, Claude Code). Typical stack: Next.js or Vite/React, Supabase or
Firebase, hosted on Vercel or Netlify.

It looks for the problems that hurt most after launch: leaked keys, a database anyone can
read, API routes with no login check, fake payment events, a surprise bill from bots or AI
calls, a broken deploy setup, and pages that break on phones. It ends with one table of
findings and a clear **GO / FIX FIRST / NO-GO** verdict, plus a short summary you can send
to a non-technical client.

## Install (30 seconds)

In your terminal:

```bash
claude plugin marketplace add keysersoose/prelaunch-audit
claude plugin install prelaunch-audit@prelaunch-audit
```

Then start `claude` in your app folder (or run `/reload-plugins` in an open session).
Newer Claude Code versions (v2.1.292+) can do both in one step:
`claude plugin install prelaunch-audit --marketplace keysersoose/prelaunch-audit`.

Or inside a Claude Code session:

```text
/plugin marketplace add keysersoose/prelaunch-audit
/plugin install prelaunch-audit@prelaunch-audit
/reload-plugins
```

`/plugin install` opens a panel: choose "Install for you".
Check it is there: `claude plugin list`.

Update later (auto-update is off by default for this kind of marketplace):

```bash
claude plugin marketplace update prelaunch-audit && claude plugin update prelaunch-audit@prelaunch-audit
```

## How to run

Open Claude Code in your app's folder and type:

```text
/prelaunch-audit:prelaunch-audit
```

That runs everything in order. You can also run one part on its own:

| Command | What it checks |
|---------|----------------|
| `/prelaunch-audit:secret-scan` | Keys in git history, files on disk and the built browser code (AI, email and SMS keys too); `.env` files in git; `NEXT_PUBLIC_` / `VITE_` misuse; what to do if a key leaked |
| `/prelaunch-audit:database-access-audit` | Supabase row level security, policies, grants, keys, storage, email sender; logged-out write test; a two-account test that user A cannot see user B |
| `/prelaunch-audit:firebase-access-audit` | Firestore and Realtime Database rules, Admin SDK use, emulator tests, storage rules, open buckets |
| `/prelaunch-audit:server-code-review` | Login check on every route, server action and function (not only middleware), ids in URLs, injection, uploads, sign-in redirects and cookies, CORS, outbound requests, error messages |
| `/prelaunch-audit:payments-and-abuse-review` | Webhooks (signature, replay, duplicates), cron routes, rate limits, SMS abuse, budgets, AI features and prompt injection |
| `/prelaunch-audit:deploy-checklist` | Env vars per environment, live payment keys, preview protection, tests before build, headers, leftover admin and debug routes, uptime alerts, plan limits, backups, rollback, email DNS, privacy policy |
| `/prelaunch-audit:ui-launch-check` | Many screen sizes, overflow, tap size, axe-core accessibility, keyboard focus, titles, search engine settings, broken links, speed |

Tip: give Claude your live or staging URL at the start. Without it, the live checks are
marked "not checked".

The commands are written for a bash shell. On Windows, use Git Bash or WSL. If a tool is
missing, use these instead: `nslookup -type=TXT` for `dig`, `netstat -ano` for `lsof`,
and `certutil -hashfile <file> SHA256` (Windows) or `shasum -a 256 -c` (macOS) for
`sha256sum -c`.

New to some of the words? See [GLOSSARY.md](GLOSSARY.md).

## Safety rules the plugin follows

- **Read-only first.** It reports problems and changes nothing until you say yes.
- **Fixes go on a branch**, never straight to your main branch.
- **No secret values are printed.** Only key names. Scanners run with `--redact`.
- **Databases are read-only**, scoped to one project, and a staging copy is preferred.
- **Live tests only on your own site.** Small request bursts, no attacks.
- **It asks before installing any tool**, and uses official downloads only
  (for example the real gitleaks binary, not an npm lookalike).
- **The report is not committed.** A findings list is a map for attackers.

Some checks need tools: gitleaks, Playwright, optional Semgrep or TruffleHog. Each skill
says how to install them.

## What it does NOT do

- It is **not a penetration test**. It does not attack your app.
- It is **not a guarantee**. A clean report means "these checks passed today", not "secure".
- It does not replace a human review of payment and sign-in code.
- It does not check legal compliance for your country.

## Contributing

Issues and pull requests are welcome.

- Keep each skill under about 220 lines, in plain simple English.
- Explain each technical word in brackets the first time it is used in a skill.
- Every command must be real. If you could not confirm a flag, write "(check the current docs)".
- Add a public source URL for every new fact to `SOURCES.md`.
- Bump the version in `.claude-plugin/plugin.json` on every release.
- Validate before you open a pull request: `claude plugin validate .`

## Licence

MIT. See [LICENSE](LICENSE).
