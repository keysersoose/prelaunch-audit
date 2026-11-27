# Words used in this plugin

Short, everyday meanings. Skills point here when they use a technical word.

- **RLS (row level security)**: a lock on each row of a table, so people see only their
  own rows. Like a mailbox per flat instead of one open pile of letters.
- **Policy / grant**: the rule for who may open which lock, and the basic permission to
  touch a table at all.
- **Anon / publishable key**: a public key that is safe in the browser. It is like the
  building's street address, not the key to a door.
- **Service / secret key**: the master key. It skips every lock, so it lives on the server only.
- **JWT / token**: a signed pass that says who you are. The server must check the
  signature, not just read the pass.
- **Webhook**: a message another service (for example a payment provider) sends to your
  site. Check its signature, like checking an ID before handing over a parcel.
- **Idempotent**: doing it twice has the same effect as doing it once, like pressing a
  lift button twice.
- **Rate limit**: a cap on how often someone can do something, like "three tries, then wait".
- **IDOR**: changing an id in the URL to see someone else's data, like changing the flat
  number on a parcel slip.
- **SSRF**: tricking your server into fetching a web address the attacker chose.
- **XSS**: getting your site to run an attacker's script in another user's browser.
- **SQL injection**: user text that becomes part of a database command.
- **Prompt injection**: hidden instructions inside text that an AI reads, which try to
  make it do something else.
- **CORS**: the rule for which other websites may call your API from a browser.
- **CSRF**: another website making a signed-in user's browser send a request to your site.
- **CSP / HSTS**: browser rules: which scripts may run, and "always use https".
- **Mass assignment**: saving every field the browser sent, including ones it should not
  set (like `role`).
- **Security definer**: a database function that runs with its owner's rights, not the caller's.
- **Typosquat**: a fake package with a name close to a real one.
- **SMS pumping**: bots triggering many text messages to numbers they profit from, on your bill.
- **Rotate a key**: make a new key, switch to it, then cancel the old one. Like changing
  the locks after losing a key.
- **Outside-in check**: testing the live site from the internet the way a stranger
  would, with `curl`.
- **Staging / preview**: a copy of the site for testing that real users do not use.
- **Rollback**: going back to the last version that worked.
- **PITR (point-in-time recovery)**: restoring the database to any minute in the past.
- **Expand / migrate / contract**: change data in three releases: add the new, move the
  data, remove the old later.
- **LCP (Largest Contentful Paint)**: how long the main content of a page takes to show.
