---
name: database-access-audit
description: Check that a web app's database and file storage cannot be read or changed by the wrong person - Supabase row level security, policies, grants, keys, storage policies, advisors and auth settings - and prove it with a logged-out read and write test and a two-account test. Firebase apps use the firebase-access-audit skill plus this skill's Section C. Use when someone says "check my RLS", "are my Supabase tables safe", "can users see each other's data", "is my database public" or "check storage buckets".
---

# Database access audit

Goal: prove that a visitor or a signed-in user can only reach the rows and files they
should. The Supabase anon (publishable) key is public by design, so anyone can call the
database API directly. Safety depends only on row level security (RLS: a lock on each
row so people see only their own), policies, grants and your server code. Reading the
rules is not enough; test them.

Safety: connect read-only, scoped to one project, preferably a staging copy or a
database branch. Never change rules, policies or grants during the audit. Never write
test rows to production. Write tests in Section C run on staging only.

**Firebase apps:** follow `${CLAUDE_PLUGIN_ROOT}/skills/firebase-access-audit/SKILL.md`,
then come back and do Section C here.

## Section A. Supabase

### A1. Connect read-only

Preferred: the Supabase MCP server in read-only mode, scoped to one project, added at
local scope (stored in your own `~/.claude.json`, so nothing is written into the repo):

```bash
claude mcp add --scope local --transport http supabase "https://mcp.supabase.com/mcp?project_ref=<ref>&read_only=true"
```

Then type `/mcp`, pick "supabase" and choose Authenticate. Keep manual approval on for
every tool call (text inside data could try to trick the agent). Remove it after the
audit: `claude mcp remove supabase --scope local`. Without MCP, use `psql` or the
dashboard SQL editor with a read-only role.

### A2. Row level security on every table

```sql
select schemaname, tablename, rowsecurity
from pg_tables
where schemaname not in ('pg_catalog','information_schema','auth','storage','realtime','extensions')
order by schemaname, tablename;
```

Every table in an exposed schema (`public` and any schema added to the API) needs
`rowsecurity = true`. A table without it can be read and written by anyone with a grant.
RLS on with no policies means "deny all" (often fine for server-only tables).

### A3. Read every policy

```sql
select schemaname, tablename, policyname, cmd, roles, qual as using_expr, with_check
from pg_policies where schemaname = 'public' order by tablename, cmd;
```

Flag:

- `using (true)` or `with check (true)` on anything that is not truly public data.
- Policies for `anon` (logged-out visitors) on private tables.
- Policies not tied to the caller, for example missing `(select auth.uid()) = user_id`.
- Permission checks based on `user_metadata` (users can edit it). Use `app_metadata`.
- UPDATE policies with no `with check`, so a user can move a row to another owner.
- Tables where a user can update their own `role`, `plan`, `credits` or `is_admin` column.
- `auth.uid()` called per row instead of `(select auth.uid())` (slow; the Performance
  Advisor flags it).

A good owner policy looks like this:

```sql
create policy own_rows on items for all to authenticated
  using ((select auth.uid()) = user_id)
  with check ((select auth.uid()) = user_id);
```

### A4. Grants

Policies do not remove grants. Check what `anon` and `authenticated` may do:

```sql
select table_schema, table_name, grantee, string_agg(privilege_type, ', ') as privileges
from information_schema.role_table_grants
where grantee in ('anon','authenticated') and table_schema = 'public'
group by 1,2,3 order by 2,3;
```

Danger pattern: a "permission denied" (error 42501) "fixed" by a blanket `grant all ... to anon`
or by switching the app to the secret key. That is Critical.

### A5. Views and functions

Flag: views without `security_invoker = true` over private tables (they skip RLS);
`security definer` functions (they run with the owner's rights) callable by `anon` or
`authenticated`; functions with a mutable `search_path` (fix: `set search_path = ''` and
fully qualified names); any public view over `auth.users`.

### A6. Keys

- The secret key (`sb_secret_...`) and the legacy `service_role` key skip RLS. Server and
  Edge Functions only, never the browser (the secret-scan skill checks the bundle).
- Legacy JWT keys (start with `eyJ`) stay valid until disabled, even after new keys exist.
- Server code that uses the secret key must add its own owner filter, for example
  `.eq('user_id', user.id)`, because RLS is skipped.
- Edge Functions must check the caller and never trust a user id sent in the request.

### A7. Storage

```sql
select id, name, public from storage.buckets;
select policyname, cmd, roles, qual, with_check from pg_policies where schemaname = 'storage';
```

- A public bucket with private files is High or Critical: public URLs skip policy checks.
- Private files: bucket `public = false`, object paths start with the user id, and a
  policy like
  `using (bucket_id = 'files' and (storage.foldername(name))[1] = (select auth.uid())::text)`.
- Upsert needs INSERT, SELECT and UPDATE policies. Backups do not include Storage files.

### A8. Advisors and auth settings

Run the Security Advisor and then the Performance Advisor: MCP `get_advisors` with type
`security`, then `performance`, or on a linked project `supabase db advisors --type security`
then `supabase db advisors --type performance`. The dashboard (Advisors) and MCP are the
final word. Some findings may be on purpose; check each against the intended access.

Supabase's built-in email sender is for testing only and sends very few emails per hour
(check the current limit). On launch day, new users stop getting confirmation and
password-reset emails. Set up custom SMTP with a real email provider before launch.

Also review (read-only, dashboard): email confirmation on, OTP and link expiry 3600 seconds
or less, test phone numbers removed, CAPTCHA on sign-up and password reset, auth rate
limits, allowed redirect URLs limited to your own domains, SSL enforcement, MFA for every
org member, more than one Owner, backups. Mark anything you cannot read as "not checked".

## Section C. Logged-out and per-user isolation tests (all stacks)

Setup: the anon (or publishable) key is in the Supabase dashboard, Project Settings, API.
It is public, so using it is fine. Make test accounts A and B by signing up twice in the
staging app with two email addresses you own.

1. Logged out, with only the public key, try to list each private table or collection.
   Supabase: `curl -s "$SUPABASE_URL/rest/v1/<table>?select=*&limit=1" -H "apikey: $ANON_KEY"`.
   Expect `[]` or an error, never real rows. (Firebase: see firebase-access-audit F5.)
2. On staging only, logged out, try to write:

   ```bash
   H=(-H "apikey: $ANON_KEY" -H 'Content-Type: application/json' -H 'Prefer: return=representation')
   curl -s -w ' %{http_code}\n' -X POST   "$SUPABASE_URL/rest/v1/<table>" "${H[@]}" -d '{}'
   curl -s -w ' %{http_code}\n' -X PATCH  "$SUPABASE_URL/rest/v1/<table>?id=eq.<id>" "${H[@]}" -d '{}'
   curl -s -w ' %{http_code}\n' -X DELETE "$SUPABASE_URL/rest/v1/<table>?id=eq.<id>" "${H[@]}"
   ```

   The insert must fail (401 or 403, error 42501). A blocked PATCH or DELETE can still
   return 200 or 204 with an empty `[]`. Pass means `[]` or an error, and the row is
   unchanged when you read it again. A table that is safe to read can still accept writes.
3. Sign in as A in the app. Copy one id that belongs to B (from B's session).
4. As A, with the client SDK or a direct API call using A's token, try to read, update and
   delete B's row or document. All must fail. Prefer a 404 over a 403 so you do not
   confirm the id exists.
5. As A, try to change your own `role`, `plan` or `credits` field directly. Must fail.
6. Repeat for files: A must not open B's file path or signed URL.

## Severity guide

| Finding | Severity |
|---------|----------|
| Table without RLS holding user data | Critical |
| `using (true)` on private data | Critical |
| Logged-out visitor can insert, update or delete rows | Critical |
| User A can read or change user B's data | Critical |
| User can raise their own role, plan or credits | Critical |
| Secret or service key used in the browser | Critical |
| Public bucket holding private files | High |
| `security definer` view or function open to anon | High |
| Sign-up needs email but uses Supabase's default sender | High |
| Slow policy (`auth.uid()` per row), missing index on policy column | Low |

## What to report

- How you connected (read-only, which project, staging or production).
- The RLS table list, the policy review, the grants and the bucket list, as short tables.
- The isolation and write test results as a table: actor, action, target, expected, actual.
- Anything "not checked" and why.
