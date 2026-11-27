---
name: firebase-access-audit
description: Check that a Firebase web app's data and files cannot be read or changed by the wrong person - Firestore and Realtime Database security rules (repo vs live), Admin SDK only on the server, emulator rules tests, Storage rules, App Check, and outside-in tests against Firestore, Realtime Database and storage buckets (also public blob and S3 buckets). Use when someone says "audit Firestore rules", "is my Firebase database open", "check my Realtime Database rules", "check storage rules" or "check my bucket".
---

# Firebase access audit

Goal: prove that a visitor or a signed-in user can only reach the documents and files
they should. The Firebase web config is public by design, so anyone can call the
database API directly. Safety depends only on security rules and your server code.

Read-only. Never change or deploy rules during the audit. After this skill, do Section C
(the two-account test) of `${CLAUDE_PLUGIN_ROOT}/skills/database-access-audit/SKILL.md`.

## F1. Read the rules in the repo and the live rules

Open `firestore.rules`, `database.rules.json` and `storage.rules`. Then compare with what
is deployed: the Firebase console (Rules tab of each product) or the Firebase MCP
`firebase_get_security_rules`. A console edit made while debugging can stay open for months.

Flag:

- `allow read, write: if true;` or `if request.auth != null;` on whole collections.
  Signed in is not the same as owner.
- Time-limited test mode rules: `request.time < timestamp.date(...)`.
- Rules that trust fields the client writes (`request.resource.data.role`).
- A broad `match /{document=**}` with an allow (paths with no match are denied, which is good).

Strongest pattern: browsers never touch the database. One catch-all rule
`match /{document=**} { allow read, write: if false; }` and all access through the server.
Check the client code: `grep -rnE "from ['\"]firebase/(firestore|storage|database)" src app components lib`.

## F2. Admin SDK only on the server

`grep -rn "firebase-admin" --include='*.ts' --include='*.tsx' --include='*.js' . | grep -v node_modules`.
Every hit must be in server-only code. The Admin SDK skips rules, so each server route
needs its own access check (the server-code-review skill covers this).

## F3. Test the rules in the emulator

Rules that look right can still leak. Ask before installing tools (the emulator needs Java).

```bash
npm i -D @firebase/rules-unit-testing
npx -y firebase-tools@<pinned-version> emulators:exec --only firestore --project demo-audit "npm run test:rules"
```

Tests should assert `assertFails` for: signed-out reads and lists, user B reading user A,
a user writing server-only fields, and a fake admin. And `assertSucceeds` for the owner.

## F4. Storage rules and App Check

Same questions for `storage.rules`: files under `users/{uid}/...` readable only by that uid.
Firebase Storage download URLs with a token work until the token is revoked. App Check
reduces abuse from scripts but does not replace rules.

## F5. Realtime Database and other file storage

If the app uses Realtime Database (`database.rules.json`, `firebase/database`, or a
`*.firebaseio.com` / `*.firebasedatabase.app` URL), read its rules. `".read": true` or
`".write": true` at the root is Critical.

Outside-in tests (signed out):

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
  "https://firestore.googleapis.com/v1/projects/$PROJECT_ID/databases/(default)/documents/users"   # expect 403
curl -s -o /dev/null -w '%{http_code}\n' "https://<db-name>.firebaseio.com/.json?shallow=true"     # expect 401
curl -s -o /dev/null -w '%{http_code}\n' "https://firebasestorage.googleapis.com/v0/b/<bucket>/o"  # expect 403, not a file list
curl -s "https://<bucket>.s3.amazonaws.com/?list-type=2" | head -c 200                              # must not list files
```

Try each top-level collection name you found in the code. Vercel Blob files uploaded with
public access can be opened by anyone who has the URL, so do not store private files that
way (check the current docs for private options).

## Severity guide

| Finding | Severity |
|---------|----------|
| Firestore open read/write holding user data | Critical |
| `if true` on private data | Critical |
| Realtime Database root open, or a bucket that lists files to anyone | Critical |
| Admin SDK used in browser code | Critical |
| Live rules differ from repo rules | High |
| Private files stored with public URLs | High |
| No rules tests | Medium |

## What to report

- Repo rules vs live rules for each product (same or different).
- Emulator test results, and the outside-in status codes with the date.
- Then the Section C table from database-access-audit.
- Anything "not checked" and why.
