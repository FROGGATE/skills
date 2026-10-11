# Keys and access in detail

Contents: Backend integration · Public website · Per-user frontend (sign-in, per-user isolation, shared section, read vs write columns, files on a record) · Capability-link access (access token) · Self-registration flow · E-mail existence disclosure · Auto-clearing of keys on scope change · Per-table read/write · Origins (CORS)

## Backend integration (backend API key)

The plaintext key (prefix `fgs_{companyId}_…`) returned by FROGGATE when an API key is created — copy it from the admin UI right after generation, it is not retrievable later. Gives whatever access the admin configures in Permissions, often `full_access=1`. This is the only secret-grade key in the system — it must stay server-side and must never be bundled into a browser app.

When the underlying key has `user_restricted=1`, the backend API key alone is **not accepted** for data calls — those keys must be used through the login flow (see below). That is enforced server-side: a user-restricted key used directly would defeat the per-user isolation, so the API rejects it.

## Public website (`fgs_public_…`)

A public-by-design key intended to be embedded directly in browser JavaScript. The backing API key must have **`full_access=0`** AND **`user_restricted=0`**, so the admin first defines a strict scope (allowed tables, columns, statuses, per-table read/write) and then generates the public key against that scope.

The public key gives identical access to whatever the underlying scope allows — no per-user filtering, no login. Anyone hitting the page can use it. Safety comes from the scope: if the only thing the key can do is read three columns from one table and submit a contact form, that is the worst case if the key leaks.

**A public key is append-only.** It can read (with read access) and insert new rows, but it can **never update or delete existing rows** — for a public key, write access means *insert only*. Because it carries no user identity, it cannot be scoped to "its own" rows the way a per-user login key can, so `update-table-row` and every structure/delete endpoint are refused for it. This is deliberate: it stops a leaked public key from overwriting arbitrary existing records by guessing their row ids. If your integration needs to update existing records, use a backend key or a per-user login key, not the public key. (`updateOnDuplicateColumnId` on insert is likewise rejected for a public key.)

**Grant read only on genuinely public data.** A public key applies **no per-user filtering** — a read returns *every* row's data (in the allowed columns and statuses) to anyone who holds the key. So only give a public key read access to a table, and only allow the statuses and columns, whose data is meant to be public for **all** rows. If some rows or columns must stay private, do not expose them through a public key at all — use a per-user login key, which filters to the caller's own rows.

**Regeneration:** the admin can regenerate the public key in **Settings → API → Permissions** at any time. Doing so immediately invalidates the previous value — this is the right reaction to a suspected leak. The deployed app will stop working until the new key is rolled out; that is the intended trade-off.

## Per-user frontend (`fgs_login_…`)

Used when the frontend needs a login screen and per-user data. The login key by itself is also a public-by-design value (lives in the browser bundle). It only opens the auth endpoints; the actual data calls additionally require a session token obtained via the login flow.

**Two credentials, independently switchable.** A key in this mode can accept a one-time code e-mailed to the user, a password, or both — they are separate settings in **Settings → API → Permissions** (*Login with an e-mailed code* / *Login with a password*), and either may be off. **Find out which are enabled before designing the sign-in screen**; neither is safe to assume, and a key with both off signs nobody in. In both cases the user is identified by the address in the key's **e-mail column** — there is no separate login identifier.

**Linked accounts.** When the key has **Sign-in with multiple accounts** on (same settings screen), the address may instead belong to a row of the key's **accounts table** (its *Account e-mail column*) — typically a contact person linked to a customer. That row signs in *as the customer it is linked to*: the session is the customer's, every data call is scoped to the customer exactly as if the customer had signed in, and each account has its own password (*Account password column*). Nothing changes in the endpoints or the request bodies; find out from the admin whether this is on, because "which address do I sign in with" is then a different question for the end user, and an address found on more than one row — across the user table and the accounts table — is refused as ambiguous. Through the data endpoints the accounts table is read-only in the ways that matter: rows cannot be inserted, their status cannot be changed, and the e-mail/password columns cannot be written.

Code flow — two steps:

1. The frontend calls `/auth/request-login-code` with the user's e-mail using `Authorization: fgs_login_…`. FROGGATE looks up the e-mail in the configured user table (and, with multiple accounts on, in the accounts table); if it matches, a 6-digit code is sent to that e-mail. Its validity is configured per API key in **Settings → API → Permissions** (*E-mail code validity*, 30 minutes by default).
2. The user types the code, the frontend calls `/auth/verify-login-code` with `{email, code}`. On success FROGGATE returns `{token, expires_in}`. The token is stored client-side and sent in `X-User-Session-Token` on every subsequent data call.

**Never hardcode how long a code is valid — read it from the response.** All three code-request endpoints (`request-login-code`, `request-registration-code`, `request-password-reset-code`) return `data.valid_minutes` in the success envelope, and that is the number to put on screen: "the code is valid for 30 minutes", a countdown, the moment you offer "send a new one". *E-mail code validity* is a per-key setting the admin can change at any time, and the same value fills `{VALID_MINUTES}` in the e-mail — taking it from the response is what keeps the page and the e-mail from contradicting each other. It is returned even when the address turns out not to be registered, so you can read it without knowing whether a code was actually sent.

Password flow — one step: `/auth/user-login` with `{email, password}`, returning the same `{token, expires_in}` envelope. Bind it to the same post-auth handler as the code flow.

**A password is only ever set by the user, never issued to them.** There is no admin-side "create a password" step and no way to write the password column through the data API, so the app must ship the set-password flow to be usable at all:

1. `/auth/request-password-reset-code` with `{email}` → a code is e-mailed, using the password e-mail template configured on the key (its own template, separate from the login and registration ones, and the same configurable validity, returned as `data.valid_minutes`).
2. `/auth/set-password` with `{email, code, password}` → the password is stored hashed and a session token is returned in the same envelope, so the user lands signed in rather than at a login screen. Minimum 8 characters, maximum 72 **bytes** (accented characters take more than one byte each — validate this client-side too, it surprises people); a refused password does **not** spend the code.

**Build step 2 as a link, not a code-entry field — this is the recommended implementation and the default template already assumes it.** The password code is a 32-character string precisely because nobody should retype it. The e-mail template supports four placeholders — `{CODE}`, `{EMAIL}`, `{EMAIL_URL}` (the address percent-encoded, for a query string) and `{VALID_MINUTES}` — so the body carries a link like:

```
https://your-app.example/set-password?email={EMAIL_URL}&code={CODE}
```

The user clicks it, your page reads `email` and `code` out of the query string and shows **one field: the new password**. Then it posts all three to `/auth/set-password` and stores the returned session token. Carrying the address in the link is what makes this work at all — the recipient often opens the mail on another device, so the page cannot assume it knows who they are. When you set this up for someone, tell them to replace the placeholder address in the template (Settings → API → Permissions) with their application's own; until they do, the link in the e-mail goes nowhere.

The same placeholders exist on the login and registration code e-mails, so the same click-through pattern works there too — but those codes stay 6 digits, because they are also meant to be read aloud and typed.

The same pair serves both "I am new here" and "I forgot my password" — the endpoints do not distinguish, so build one screen and reach it from both places. **The first invitation usually does not come from your app at all**: the FROGGATE admin can send it from the section's *Odeslat e-mail* action, which puts the very same link into an e-mail of their own wording, valid for days rather than minutes. Your set-password page is the landing point either way, so build it to work from a link alone — address and code read out of the query string, nothing carried over from a previous screen. Setting a password **revokes that user's existing sessions** (that is what makes it useful after a compromise); other users of the key are unaffected. A user who has never set a password cannot sign in with `/auth/user-login` at all — the refusal is deliberately identical to a wrong password, so route people to the set-password flow by offering it, not by detecting it.

**Sliding TTL:** the session token has a sliding expiry. Every successful API call refreshes it server-side, so an active user is not logged out mid-use. Inactive users eventually expire. The TTL in hours is set on the API key in **Settings → API → Permissions** (default 168 = 7 days, the user can shorten/extend it). When a data call returns 401, run the login flow again.

**Per-user isolation:** data access for login-flow keys is automatically restricted to the logged-in user's rows via each allowed table's **user-link column** — i.e. the `from_other_table` (link) column pointing to the user table. Cross-user reads/writes are impossible:
- `get-table-rows` returns only the user's own rows; the password column on the user table is always stripped from responses.
- `insert-table-row` auto-fills the user-link column with the user's row id; any value the client sends for that column is ignored.
- `update-table-row` rejects with 403 if the row's user-link column does not match the logged-in user.
- A link column (`from_other_table`) set on insert or update may point only at a record the user may see: their own user row, a record of a section linked to the user table that is theirs or has no user filled in (e.g. the shared rows of a price list with per-client prices; not a deleted one, and not in a table anyone signs in from), or a record of a section that holds no per-user data (a menu, a price list). Any other id is refused exactly like an id that does not exist.
- The logged-in user can also **update their own row** in the user table itself (e.g. edit their profile / name) — subject to the same write scope, but never the e-mail / password / access-token columns, and never the row's status (those are the login anchor and stay locked).
- A table **without** a user-link column is refused, unless it is set as a **shared section** on the allowed table. A shared section is read in full by every signed-in user (only rows in the allowed statuses, only the allowed columns, files only with file read) and is never writable — `insert-table-row`, `update-table-row` and `run-action` on it are refused with a message saying it is read-only. A request without a valid session gets nothing from it, which is what sets it apart from a public key. Sections holding records of individual users cannot be shared, so isolation still holds.

**Read vs write columns.** Each allowed column carries two independent switches: **read** (its value is returned in responses) and **write** (the key may set it on insert/update). They are separate on purpose — a value can be readable without being writable. A computed column (e.g. a formula-filled price) is typically **read-only**: you expose it so the frontend can display it, but it can never be written back, even if write is toggled on. So put display-only fields on **read**, and only the fields the app actually edits on **write** too.

**Files on a record.** Files attached to a record are two **separate per-table capabilities, both off by default**, enabled on the allowed table in Permissions. **File read** governs whether `get-table-rows` returns the record's `files` list at all — without it the list comes back empty even for records the key may read. **File upload** governs attaching a file (insert `files` / update `insertFiles`) and refiling one into another category (update `updateFiles`), and applies to a public key on insert as well. They are independent of each other and of the allowed **columns**: an attachment belongs to the record, not to a column, so narrowing the readable columns never narrows the files. Both are ignored for a full-access key.

A file can also carry a **file category** — a per-section list the section defines. Set it with `fileCategoryId` when the file is attached, or move an already-stored file with `updateFiles`; a category of another section is refused. Where the section restricts access per category, the category is what a file is protected by, so refiling one changes who can open it. The ids are listed in the app under Settings → API → Element IDs.

**Tip when recommending the column scope.** When you walk the user through which columns to put on the allowed-columns list of an `fgs_login_…` key, **don't mention the user-link column** (the `from_other_table` column pointing to the user table). The API key doesn't need access to it — per-user filtering is enforced server-side via that column regardless of whether it's in scope, and the column is auto-filled on insert (never user-writable). There's simply no reason to list it among the columns the user has to allow.

**Allowed endpoints** for login-flow keys: `/auth/request-login-code`, `/auth/verify-login-code`, `/auth/user-login`, `/auth/request-password-reset-code`, `/auth/set-password`, `/auth/verify-access-token`, `/auth/request-registration-code`, `/auth/register`, `get-table-rows`, `insert-table-row`, `update-table-row`, `run-action` (only on the logged-in user's own rows, and only actions granted to the key). Anything else (`get-active-tables`, `insert-column-selection-value`, …) returns 403 — so a login-flow / access-token frontend works against table and column IDs you already know, not ones it discovers at runtime.

## Capability-link access (access token)

A variant of the per-user frontend that needs **no login at all**. Instead of an end user signing in with their e-mail, you share a single record with someone by sending them a link that carries a secret per-record token. Whoever holds the link sees exactly that one record and its connected records — nothing else. Ideal for "send this one thing to this one person": a price quote, an order-status page, an invoice, a single customer-portal record.

It runs on the **same `fgs_login_…` key** as the login flow — the key is still generated by turning **Restrict to user** on. The only difference is one setting in Permissions: **Access mode = Access token** (instead of User login). In that mode you pick:

- **Shared table** — the table whose rows are shared.
- **Access-token column** — a text column on that table holding each row's secret token. You fill it with a long, unguessable random string (≥ 20 characters) per row — this *is* the capability, so treat it like a password scoped to that one record.
- **Statuses in which a row is reachable** — the token only resolves while the record is in one of these statuses (see "kill switch" below).

Single-step flow — there is no e-mail round-trip and no request-code stage:

1. You put the record's token in a link you send to the recipient (e.g. `https://your-app/quote?t=<token>`).
2. The frontend calls `/auth/verify-access-token` with `{ token }` using `Authorization: fgs_login_…`. On success FROGGATE returns `{ token, expires_in }` — the **same session-token shape as `verify-login-code`**. Store it and send it in `X-User-Session-Token` on every subsequent data call, exactly like the login flow.

From there everything is identical to the login flow's data calls — same headers, same sliding TTL, same response envelope.

**Single-record isolation.** Once the session is bound, every data call is scoped to the one record the token resolved to:
- `get-table-rows` on the shared table returns only that record; on a connected table (one linked to the shared table via a `from_other_table` column) it returns only the rows linked to that record. Any id / link-column filter the client sends is overridden server-side — the scope cannot be widened.
- The **access-token column is never returned** in any data response — not as a value, and not as a foreign-key display label. The token never leaves the system once issued.
- The shared record itself can be **updated** in this mode — but only the columns you put on its **write** scope (see "Read vs write columns"), and never the access-token column or the record's status (both stay locked: they are the capability and the kill switch). So "let the recipient add a note / accept the quote / attach a signed PDF" can be done **directly on the shared record** — write-scope those columns (and enable file upload for the PDF). A connected table still works too for one-to-many additions, but you no longer have to route a simple field edit through one.

**Kill switch / expiry.** The link stops working the moment the record leaves the reachable statuses — move a quote to an "Expired" or "Archived" status that isn't on the reachable list and the token no longer resolves, and any active session for it is rejected on the next call. This is on top of the sliding session TTL. There is no separate "revoke" button — status is the control; to rotate a single link, change that row's token value.

**Security model — it is a bearer link.** Anyone who has the link can open it; there is no second factor. That is the point (frictionless sharing), but it means a forwarded e-mail or a leaked URL grants access. Keep the reachable-status window tight for sensitive data, and prefer the e-mail login flow when you need to prove *who* is looking, not merely *that they hold the link*.

## Self-registration flow (`/auth/request-registration-code` + `/auth/register`)

Optional companion to the login flow that lets end users sign themselves up from the frontend without any backend round-trip on the integrator's side. Disabled by default — the admin must enable **Allow user self-registration** on the API key in Settings → API → Permissions and pick a **Status for new users** before the endpoints respond. If the toggle is off (or the status is missing), every call returns `errorCode: REGISTRATION_NOT_ALLOWED` (for the missing status: `REGISTRATION_CODE_REQUEST_FAILED` on the request step, `REGISTRATION_FAILED` on `register`). Tell the user where to enable it whenever they ask for sign-up; do not silently work around the disabled state.

Two-step flow, mirrors the login flow's mechanics (6-digit e-mail code, the same configurable TTL, the same per-address cooldown):

1. The frontend calls `/auth/request-registration-code` with the user's e-mail. FROGGATE checks the e-mail is not already registered (any non-deleted status counts; soft-deleted rows status=0 are ignored — same e-mail is registerable again). If it's free, a code is e-mailed.
2. The user types the code, the frontend calls `/auth/register` with `{email, code}`. On success FROGGATE inserts a new row in the user table with the configured status and answers in the **same response shape as `verify-login-code`**. When the status for new users is also one of the key's login statuses, the envelope carries a session token and the user is now logged in — bind both flows to the same post-auth handler, with the same `X-User-Session-Token` header on subsequent data calls. When it is not, the user is **pending approval**: `token` and `expires_in` are `null`, nothing is stored, and the app shows "your account is waiting for approval" until an administrator moves the row into a login status.

**What you can build — pick the setup with the user.** Whether a new user is signed in right away is decided by one choice on the API key: is the **Status for new users** also one of the key's login statuses? Ask the user which product they want before configuring it, and say what the other option would give them.

- **Open sign-up** — the status for new users is a login status (e.g. both are "Active"). `register` returns a token and the user is in the app at once. Suits a consumer app where anyone may join.
- **Sign-up with approval** — the status for new users is outside the login statuses (e.g. new users land in "Waiting for approval", login is allowed only from "Active"). `register` creates the row and returns no token; the app shows "your account is waiting for approval". Someone in the company approves by moving the row to "Active" — by hand in FROGGATE, or with an action on the section. To tell the user they were approved, add a scenario action on the section that sends an e-mail when the row enters "Active". Rejecting = moving the row to a status that is not a login status (or deleting it). Suits B2B portals, member areas, anything where the company decides who gets in.

**Trap for the approval setup — build the UI for it.** A user waiting for approval is not in a login status, so the login flow does not find them: `request-login-code` answers `EMAIL_NOT_REGISTERED` (or the neutral "code sent" with disclosure off), while `request-registration-code` with the same address answers `EMAIL_ALREADY_REGISTERED`. A one-field "type your e-mail" entry would bounce such a user between "not registered" and "already registered". When registration says the address is taken right after login said it is unknown, show "your account exists but cannot sign in yet — it is waiting for approval or has been deactivated" instead. The same applies to a user whose row was moved out of the login statuses later.

**Existence is intentionally revealed** on `request-registration-code`: when the e-mail is already taken the response is `errorCode: EMAIL_ALREADY_REGISTERED`. This is by design — the frontend can detect this code and route the user into the login flow. Build the UI so a user that mistypes "register" instead of "login" gets a friendly redirect, not a generic error. The login flow has a mirror-image code (`EMAIL_NOT_REGISTERED`), but there it is configurable — see "E-mail existence disclosure on `request-login-code`" below.

**Race-condition guard.** Between request and verify (for as long as the code stays valid) the e-mail can become taken — admin adds the user manually, or another self-registration races in. The verify step re-checks and refuses with `EMAIL_ALREADY_REGISTERED` rather than producing a duplicate row. Surface the same "looks like you already have an account, log in" UX as for the request-time variant.

**Brute-force defence.** A wrong code does NOT burn the original — the user can retry a few times. Once the allowed wrong attempts are used up, the code is purged and `register` returns `REGISTRATION_TOO_MANY_ATTEMPTS`; the frontend should prompt the user to request a new code. Do not count attempts in the app — the number is a server-side security setting and may change; the error code is the signal.

**UX recommendation — keep the user logged in after sign-up.** A token returned by `/auth/register` is interchangeable with one returned by `/auth/verify-login-code`. The post-registration UX should drop the user straight into the app, not bounce them through a "now please log in" screen. Concretely: persist the token in `localStorage` immediately, set the in-memory state, navigate to the dashboard. Re-using the same `verifyLoginCode`-style handler is the simplest way to do it.

**Default e-mail wording is Czech.** The default subject (`Registrační kód`) and body (`Váš registrační kód je: {CODE} … Kód je platný {VALID_MINUTES} minut.`) are configured per API key in **Settings → API → Permissions → Self-registration**. `{EMAIL}` and `{EMAIL_URL}` are available here too if the template is turned into a click-through link. If the app is not Czech-language, instruct the admin to override the template there. The placeholders `{CODE}` and `{VALID_MINUTES}` are replaced at send time. The template is independent from the login template — register and login e-mails are configured separately so the wording can match the user's mental state in each flow.

**Rate limits.** `request-login-code`, `request-registration-code` and `request-password-reset-code` are rate-limited; every cap returns `errorCode: TOO_MANY_REQUESTS`. The numbers are a server-side security setting and may change, so never hard-code them — design for the three kinds of cap:

- per-(key, e-mail), all flows — a short cooldown that covers the "user clicked Send code twice" case. Show "Please wait a moment before requesting another code."
- per-(key, IP), on registration and password reset, not on login — sized for one person, so a single attacker on one IP is capped without affecting other users. Call these endpoints from the visitor's browser, never through your own server: behind a server every visitor shares its IP and the cap. A legitimate user who hits it is probably on a heavily shared connection and needs to wait.
- per-(key, day), all flows — backstop against distributed abuse, namespaced per flow so they cannot drain each other. A real app trips this only under attack; the admin should investigate when it fires (suspicious traffic, leaked key, etc.).

Build the UI with this in mind: any `TOO_MANY_REQUESTS` response is a "wait, then retry" — not a permanent failure and not a code/data error.

**E-mail validation.** Both flows reject syntactically invalid addresses and any value containing CR/LF before doing anything else. The server also canonicalises the value (trim + lowercase) — that canonical form is what gets compared, rate-limited and (on registration) stored. Practical UX implication: do a basic HTML5 `type="email"` / client-side validity check before submitting, so the user gets immediate feedback instead of a round-trip error. Do not try to "preserve case" on the registered row — the server overwrites that anyway, and the canonical form is what makes subsequent logins find the same row regardless of how the user types it.

## E-mail existence disclosure on `request-login-code`

Whether `/auth/request-login-code` admits that an e-mail has no account is a **per-key setting** — **On login, reveal that the e-mail is not registered** in Settings → API → Permissions. It is **on by default**. Find out which mode the key is in before designing the login screen; do not assume either behaviour.

**On (default) — disclosure.** An e-mail with no matching user returns `errorCode: EMAIL_NOT_REGISTERED`. This exists so the app can offer a single entry field for both cases: the user types an e-mail and is either sent a login code or dropped into sign-up, with no "do you already have an account?" question up front. Handle the code by calling `/auth/request-registration-code` with the same address (self-registration must be enabled on the key too), not by showing an error.

**Off — anti-enumeration.** The response is identical whether or not the e-mail exists — the generic "code sent" success, with nothing actually sent for an unknown address — so the endpoint cannot be used to discover which addresses have an account. Keep the UI neutral to match: show "If the e-mail is registered, a code has been sent." Anything more specific (e.g. "Account not found") is a guess that leaks exactly what the setting was turned off to protect.

Which mode to recommend depends on what the user table holds. Advise turning it **off** when merely knowing that an address has an account is itself sensitive — a B2B supplier's customer list, a client or patient roster, anything where "is X your customer?" is a question outsiders should not be able to answer. Leaving it **on** suits a general consumer sign-up where accounts are not secret and the smoother one-field entry is worth more.

Two things behave the same in both modes: a malformed e-mail (bad format, contains a line break) is rejected with `errorCode: LOGIN_CODE_REQUEST_FAILED` — "you sent garbage" is not an existence channel — and rate-limit budget is consumed for unknown e-mails too, so throttling never becomes a side channel either.

## Auto-clearing of public/login keys on scope change

The "Full access" and "Restrict to user" switches in Permissions own the lifecycle of `fgs_public_…` and `fgs_login_…`:

- Turning **Full access** or **Restrict to user** **on** → the public key is wiped immediately (a public key on an unrestricted-access key would be a privilege escalation, so it cannot coexist).
- Turning **Restrict to user** **off** → the login key is wiped (the login flow no longer applies).

When this happens the deployed frontend starts getting 401s. If the user reports "it suddenly stopped working", check whether they changed scope flags in the admin — they need to regenerate the relevant key in **Settings → API → Permissions** and roll the new value out.

## Per-table read/write

Each allowed table can be set to read-only, write-only or read+write in Permissions. Reading a write-only table or writing to a read-only table returns 403. Worth knowing when you propose a scope to the user. Two caveats for a **public key**: write means *insert only* (it can never update or delete existing rows, whatever this setting says — see Public website), and read returns every row to anyone, so only enable it on data that is public for all rows.

## Origins (CORS)

Every API key has an allowed-origin list managed in Permissions. For browser-embedded keys (`fgs_public_…`, `fgs_login_…`) the user should list the actual production domain(s) of the site (and `http://localhost:…` for development). For a backend API key `*` is acceptable (it never reaches the browser anyway). If a call fails with `403 Origin not allowed`, point the user at the **Allowed origins** tab in the Permissions modal.
