<!-- Generated from the FROGGATE OpenAPI specification (https://client.froggate.cloud/apidoc/api-bundled.yaml) — do not edit by hand. -->
# API reference: `/auth/…`

Base URL: `https://api.froggate.cloud/v1/{companyId}` — replace `{companyId}` with the numeric company ID. Field names, types, allowed values and error codes below are exact.

Endpoints: `/auth/request-login-code` · `/auth/verify-login-code` · `/auth/verify-access-token` · `/auth/user-login` · `/auth/request-password-reset-code` · `/auth/set-password` · `/auth/request-registration-code` · `/auth/register`

## POST /auth/request-login-code

**Request login code**

Send a 6-digit login code to the user's e-mail. The code is valid for the period configured per API key in **Settings → API → Permissions** (*E-mail code validity*, 30 minutes by default). The response carries that number as `data.valid_minutes`, so a frontend can tell the user how long the code lasts without hardcoding it.

Send the login-session key directly in the header, with no scheme prefix: `Authorization: <fgs_login_*>`. The login-session key is generated in the admin under Settings → API → Permissions when "Restrict to user" is enabled.

**Linked accounts.** When the key has **Sign-in with multiple accounts** enabled (Settings → API → Permissions), the address may also belong to a row of the configured accounts table — e.g. a contact person linked to a customer. Such an account signs in *as the customer it is linked to*: the session it receives is the customer's, so every data call is scoped exactly as if the customer had signed in with their own address. An account row must be in one of the **Allowed account statuses** and its customer in one of the login statuses. An address found on more than one row — across the user table and the accounts table — is refused as ambiguous.

**Existence disclosure is configurable** via the **On login, reveal that the e-mail is not registered** setting (Settings → API → Permissions). When ON (default), an e-mail with no matching user returns `errorCode: EMAIL_NOT_REGISTERED` so the frontend can redirect the user into the self-registration flow. When OFF, the response is intentionally generic regardless of whether the e-mail exists, to prevent user enumeration — a registered and an unregistered e-mail are indistinguishable. A malformed e-mail (invalid format, contains CR/LF) is rejected with `errorCode: LOGIN_CODE_REQUEST_FAILED` and message "Invalid e-mail address" — this is deliberately a separate error from the success envelope, since "you sent garbage" is not an enumeration channel.

Two-tier rate limit (the exact numbers are a server-side security setting and may change — react to the error code, never hard-code them):
  - **per-(API key, e-mail)** — a short cooldown before the same address can get another code; covers the "user clicked Send twice" case. Budget is consumed for unknown e-mails too so the rate-limit behaviour doesn't leak existence.
  - **per-(API key, day)** — a daily cap per key. Backstop against using the public `fgs_login_*` key to spam pre-existing users (the login flow only sends to e-mails that exist in the user table, but that is still a spam vector that can blacklist the customer's sending domain). Healthy apps stay well under it; once it fires it is a "key is being abused" signal.

Both return the same `errorCode: TOO_MANY_REQUESTS`. The login and registration flows are tracked independently, so saturating one does not affect the other.

### Request body (`application/json`)

- `email` (string, format email, required) — E-mail address of the end user. Must be in a syntactically valid e-mail format; CR/LF characters are rejected as a header-injection defence. The server canonicalises the value (trim + lowercase) before lookup, so case and surrounding whitespace do not matter. Must match a row in the user table configured for this API key. Example: `user@example.com`.

### Responses

#### 200 — OK response

Content type `application/json`:

- `status` (string) — Example: `ok`.
- `statusInfo` (string) — Example: `If the email matches a registered user, a login code has been sent`.
- `errorCode` (string) — On error, one of: - `EMAIL_NOT_REGISTERED` — no user matches this e-mail. Only returned when the **On login, reveal that the e-mail is not registered** setting is on (default); the frontend should redirect the user into the self-registration flow. With the setting off, an unregistered e-mail returns the generic `status: ok` "code sent" envelope instead (existence is hidden). - `LOGIN_CODE_REQUEST_FAILED` — malformed e-mail, or multiple users share this e-mail (ambiguous match). Operational; surface to the developer. - `TOO_MANY_REQUESTS` — rate limit hit. Tell the user to wait and try again.
- `data` (object)
  - `valid_minutes` (integer) — How long the code stays valid, in minutes, as configured for this API key (*E-mail code validity*). Render the code's lifetime from this value instead of hardcoding a number — the admin can change it at any time. Returned on every `status: ok` response, including the generic envelope sent when the address is not registered, so it never reveals whether a code was actually dispatched. Example: `30`.

#### 400 — Error response

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `API key is invalid for this company`.
- `errorCode` (string) — One of: `BAD_REQUEST`.
- `data` (null)


## POST /auth/verify-login-code

**Verify login code**

Verify the 6-digit login code and obtain a session token.

Send the login-session key the same way as in `request-login-code`: `Authorization: <fgs_login_*>`.

The address may be a customer's own or one of their linked accounts (see `request-login-code`); the session is the customer's either way.

**The returned `token` does NOT replace the login-session key in `Authorization`.** On every subsequent data call (`get-table-rows`, `insert-table-row`, `update-table-row`) you must send BOTH headers:
  - `Authorization: <fgs_login_*>` — unchanged, the same login-session key
  - `X-User-Session-Token: <token>` — the session token returned here

The session TTL is sliding — every successful API call refreshes it server-side, so an active user is not logged out mid-use.

### Request body (`application/json`)

- `email` (string, format email, required) — Same e-mail used in request-login-code. Example: `user@example.com`.
- `code` (string, required) — 6-digit code received by e-mail. Example: `482157`.

### Responses

#### 200 — OK response

Content type `application/json`:

- `status` (string) — Example: `ok`.
- `statusInfo` (string)
- `errorCode` (string) — On error, one of: - `LOGIN_CODE_VERIFY_FAILED` — wrong code (attempts remain), invalid e-mail, an e-mail that is not registered, or the key is not set up for e-mail-code login (`statusInfo` names a configuration problem). Show "code invalid, retry". - `LOGIN_CODE_TOO_MANY_ATTEMPTS` — too many wrong codes; this code is purged. Prompt the user to request a new code, which starts over. Do not count attempts in the client — the error code is the signal.
- `data` (object)
  - `token` (string) — Session token. Send in `X-User-Session-Token` header on subsequent data calls.
  - `expires_in` (integer) — Token TTL in seconds. The TTL is sliding — every successful API call refreshes it server-side. Example: `604800`.

#### 400 — Error response

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `API key is invalid for this company`.
- `errorCode` (string) — One of: `BAD_REQUEST`.
- `data` (null)


## POST /auth/verify-access-token

**Verify access token**

Exchange a per-record access token for a session token.

Use this when a record is shared via a capability link instead of an end-user login: the token travels in the link you send to the recipient, and whoever holds it gets read access (and any configured write access) to exactly that one record and its connected records — no e-mail login required.

Send the login-session key the same way as in `verify-login-code`: `Authorization: <fgs_login_*>`.

**The returned `token` does NOT replace the login-session key in `Authorization`.** On every subsequent data call (`get-table-rows`, `insert-table-row`, `update-table-row`) you must send BOTH headers:
  - `Authorization: <fgs_login_*>` — unchanged, the same login-session key
  - `X-User-Session-Token: <token>` — the session token returned here

The access token only resolves while its record is in one of the statuses configured as reachable for this key; once the record leaves that set the link stops working. The returned session TTL is sliding — every successful API call refreshes it server-side.

### Request body (`application/json`)

- `token` (string, required) — The access token carried in the shared link. It grants access to the single record it is stored on. Example: `a1b2c3d4e5f6g7h8i9j0`.

### Responses

#### 200 — OK response

Content type `application/json`:

- `status` (string) — Example: `ok`.
- `statusInfo` (string)
- `errorCode` (string) — On error: `ACCESS_TOKEN_VERIFY_FAILED`. An unknown or empty token and a record outside the reachable statuses get the same response, so the link reveals nothing — show "this link is no longer valid". A token shared by two records or an incomplete setup on the key has its own `statusInfo`; that is a configuration error for the administrator.
- `data` (object)
  - `token` (string) — Session token. Send in `X-User-Session-Token` header on subsequent data calls.
  - `expires_in` (integer) — Token TTL in seconds. The TTL is sliding — every successful API call refreshes it server-side. Example: `604800`.

#### 400 — Error response

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `API key is invalid for this company`.
- `errorCode` (string) — One of: `BAD_REQUEST`.
- `data` (null)


## POST /auth/user-login

**Log in with a password**

Exchange an e-mail and password for a session token.

Available only when the API key has **Login with a password** enabled (Settings → API → Permissions) and a password column configured. A key may accept the e-mailed code, a password, or both — they are independent settings, so this endpoint working says nothing about whether `request-login-code` does.

The user is identified by the address in the key's e-mail column; there is no separate login identifier. Only rows in one of the key's login statuses are considered.

**Linked accounts.** With **Sign-in with multiple accounts** enabled on the key, the address may also be a row of the accounts table (see `request-login-code`). Each linked account has its own password, stored in the **Account password column** of the accounts table, and signs in as the customer it is linked to. An address found on more than one row across the two tables is refused as ambiguous.

Send the login-session key directly in the header, with no scheme prefix: `Authorization: <fgs_login_*>`, and use the returned token exactly as the one from `verify-login-code`.

Refusals are deliberately indistinguishable from one another (`USER_LOGIN_FAILED`, "Invalid e-mail or password") — an unknown address, a user who has never set a password, and a wrong password all answer the same way. Rate limited per (API key, e-mail): a few failed attempts within a short window, then `TOO_MANY_REQUESTS` until the window passes (the exact numbers are a server-side security setting and may change — react to the error code). A successful sign-in clears the window, so only failed attempts accumulate. Once the window is spent it holds for the correct password too.

A user who has no password yet cannot sign in here at all; send them through `request-password-reset-code` + `set-password` first.

### Request body (`application/json`)

- `email` (string, format email, required) — E-mail address of the end user — the same address the e-mail code and the password code are sent to. There is no separate login identifier. Example: `user@example.com`.
- `password` (string, format password, required) — The user's password in plain text, sent over TLS and verified against the stored hash. Example: `correct-horse-battery`.

### Responses

#### 200 — OK response

Content type `application/json`:

- `status` (string) — Example: `ok`.
- `statusInfo` (string)
- `errorCode` (string) — On error, one of: - `USER_LOGIN_FAILED` — invalid credentials, password login not enabled for this key, incomplete configuration, or multiple users share this e-mail. - `TOO_MANY_REQUESTS` — rate limit hit. Tell the user to wait and try again.
- `data` (object)
  - `token` (string) — Session token. Send in `X-User-Session-Token` header on subsequent data calls.
  - `expires_in` (integer) — Token TTL in seconds. The TTL is sliding — every successful API call refreshes it server-side. Example: `604800`.

#### 400 — Error response

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `API key is invalid for this company`.
- `errorCode` (string) — One of: `BAD_REQUEST`.
- `data` (null)


## POST /auth/request-password-reset-code

**Request password code**

Send a code that authorises `set-password`. This is how a password is created in the first place and how a forgotten one is replaced — there is no other way to write a password, and no administrator ever issues one.

Available only when the API key has **Login with a password** enabled (Settings → API → Permissions) and a password column configured. Otherwise the call is refused.

Send the login-session key directly in the header, with no scheme prefix: `Authorization: <fgs_login_*>`.

The code is valid for the period configured per API key (*E-mail code validity*, 30 minutes by default) and is delivered to the address in the key's e-mail column, using the subject and body configured for the password e-mail. The response carries that number as `data.valid_minutes`, so a frontend can tell the user how long the link lasts without hardcoding it.

**The code is a 32-character string, not a short number, because it is not meant to be typed.** Build the flow as a link: the e-mail template supports the placeholders `{CODE}`, `{EMAIL}`, `{EMAIL_URL}` (the address percent-encoded for a query string) and `{VALID_MINUTES}`, so the configured body carries something like `https://your-app.example/set-password?email={EMAIL_URL}&code={CODE}`. The recipient clicks, your page reads both values out of the query string, and the only field they fill in is the new password. The default template is already shaped this way and needs its address replaced with the application's own.

The same placeholders exist on the login and registration code e-mails, so the same click-through pattern works there — but those codes stay 6 digits, since they are also meant to be readable and typed.

**Existence disclosure follows the same key setting as `request-login-code`** (*On login, reveal that the e-mail is not registered*). When ON, an unknown address returns `errorCode: EMAIL_NOT_REGISTERED`. When OFF, the response is the generic success envelope whether or not the address exists.

Only users whose row is in one of the key's login statuses can receive a code. A row moved out of those statuses can neither sign in nor give itself a new password.

A linked account (**Sign-in with multiple accounts**, see `request-login-code`) receives its code at the address in the accounts table and sets its own password there; the customer's password and those of the other accounts are separate. A linked account can only receive a code while the key has an **Account password column** configured.

Three-tier rate limit, all returning `errorCode: TOO_MANY_REQUESTS` (the exact numbers are a server-side security setting and may change — react to the error code, never hard-code them):
  - **per-(API key, e-mail)** — a short cooldown before the same address can get another code; the budget is consumed for unknown addresses too, so rate-limit behaviour does not leak existence.
  - **per-(API key, IP)** — an hourly cap per visitor's IP address, sized for one person. Call this endpoint from the visitor's browser, not through your own server — behind a server every visitor shares its IP and the cap.
  - **per-(API key, day)** — a daily cap per key.

This flow has its own budgets, so exhausting it does not affect sign-in or self-registration.

### Request body (`application/json`)

- `email` (string, format email, required) — E-mail address of the end user. Must be in a syntactically valid e-mail format; CR/LF characters are rejected as a header-injection defence. The server canonicalises the value (trim + lowercase) before lookup, so case and surrounding whitespace do not matter. Must match a row in the user table configured for this API key. Example: `user@example.com`.

### Responses

#### 200 — OK response

Content type `application/json`:

- `status` (string) — Example: `ok`.
- `statusInfo` (string) — Example: `If the email matches a registered user, a password code has been sent`.
- `errorCode` (string) — On error, one of: - `EMAIL_NOT_REGISTERED` — no user matches this e-mail. Only returned when the **On login, reveal that the e-mail is not registered** setting is on; with it off, an unregistered e-mail returns the generic `status: ok` envelope instead. - `PASSWORD_CODE_REQUEST_FAILED` — password login is not enabled for this key, its configuration is incomplete, the e-mail is malformed, or multiple users share this e-mail (ambiguous match). Operational; surface to the developer. - `TOO_MANY_REQUESTS` — rate limit hit. Tell the user to wait and try again.
- `data` (object)
  - `valid_minutes` (integer) — How long the code stays valid, in minutes, as configured for this API key (*E-mail code validity*). Render the code's lifetime from this value instead of hardcoding a number — the admin can change it at any time. Returned on every `status: ok` response, including the generic envelope sent when the address is not registered, so it never reveals whether a code was actually dispatched. Example: `30`.

#### 400 — Error response

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `API key is invalid for this company`.
- `errorCode` (string) — One of: `BAD_REQUEST`.
- `data` (null)


## POST /auth/set-password

**Set password**

Exchange the code from `request-password-reset-code` for a written password, and receive a session token in the same call.

The password arrives in plain text over TLS and is hashed (bcrypt) before storage; the stored value is never returned by any endpoint and cannot be written through `update-table-row`. The request body of every `/auth/*` endpoint is excluded from the request log, so the submitted password is not recorded anywhere.

A session token is issued immediately on success — proving control of the e-mail address is a stronger check than the password just chosen, so the user does not need to sign in again afterwards. Use it exactly as the token from `verify-login-code`: `Authorization: <fgs_login_*>` plus `X-User-Session-Token: <token>` on subsequent data calls.

**Any session this user already had is revoked**, so changing a password ends sessions held by anyone else. Other users of the same API key are unaffected. For a linked account (**Sign-in with multiple accounts**) the password is written into the accounts table and only that account's own sessions are revoked — the customer's sessions and those of their other accounts stay signed in.

**A wrong code never destroys the one that was sent.** Guessing a 32-character code is not a real attack, whereas a code destroyable by wrong guesses would let anyone who knows the address kill a pending link. Repeated wrong codes for the same address are throttled — past roughly ten in an hour the answer becomes `SET_PASSWORD_TOO_MANY_ATTEMPTS` — but the correct code still works even then. The code is spent only once the password has actually been stored: a password rejected for its length or characters, and a failure to write the value, both leave it usable.

### Request body (`application/json`)

- `email` (string, format email, required) — E-mail address the code was sent to. Canonicalised (trim + lowercase) before lookup. Example: `user@example.com`.
- `code` (string, required) — The code from the password e-mail — a 32-character URL-safe string, not a short numeric code, because it is meant to travel inside a link rather than be typed. Read it out of the query string of the link the user clicked. Valid only for this flow: a code obtained from `request-login-code` or `request-registration-code` is not accepted here. Example: `7Kq2xR9mVb4TzLpA1sWc6EnHy3Ud8Jf0`.
- `password` (string, format password, required) — The new password in plain text, sent over TLS and hashed server-side (bcrypt) before storage. Minimum 8 characters and at most 72 bytes — bcrypt reads only the first 72 bytes, so a longer value is refused rather than silently truncated (note that accented or non-Latin characters take more than one byte each). A null character is refused too. None of these refusals consume the code, so the user can correct the value without requesting a new one. Example: `correct-horse-battery`.

### Responses

#### 200 — OK response

Content type `application/json`:

- `status` (string) — Example: `ok`.
- `statusInfo` (string)
- `errorCode` (string) — On error, one of: - `PASSWORD_TOO_WEAK` — the password is shorter than 8 characters. The code is still valid; let the user try again. - `PASSWORD_TOO_LONG` — the password exceeds 72 bytes, the most bcrypt reads. The code is still valid. - `PASSWORD_INVALID` — the password contains a null character. The code is still valid. - `SET_PASSWORD_FAILED` — wrong or expired code, password login not enabled for this key, incomplete configuration, multiple users share this e-mail, or the value could not be written to the column. `statusInfo` names which; the code survives a write failure, so the same one can be submitted again once the configuration is corrected. - `SET_PASSWORD_TOO_MANY_ATTEMPTS` — too many wrong codes for this address in the last hour. The code itself is untouched; a correct one still works.
- `data` (object)
  - `token` (string) — Session token. Send in `X-User-Session-Token` header on subsequent data calls.
  - `expires_in` (integer) — Token TTL in seconds. The TTL is sliding — every successful API call refreshes it server-side. Example: `604800`.

#### 400 — Error response

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `API key is invalid for this company`.
- `errorCode` (string) — One of: `BAD_REQUEST`.
- `data` (null)


## POST /auth/request-registration-code

**Request registration code**

Send a 6-digit registration code to the user's e-mail as the first step of the public self-registration flow. Pair with `register` to create a new user row and obtain a session token.

**Disabled by default.** This endpoint is only available when the admin enables **Allow user self-registration** in **Settings → API → Permissions** AND configures a **Status for new users** there. Without these the call returns `errorCode: REGISTRATION_NOT_ALLOWED` (or `REGISTRATION_CODE_REQUEST_FAILED` when the status is missing).

Send the login-session key directly in the header, with no scheme prefix: `Authorization: <fgs_login_*>`. The backend API key and `fgs_public_*` are rejected — same rule as the rest of `/auth/*`.

**Existence is intentionally revealed.** Unlike `request-login-code` (which hides whether the e-mail exists to prevent enumeration), this endpoint returns the dedicated `errorCode: EMAIL_ALREADY_REGISTERED` when an e-mail is already taken. The frontend can detect this and route the user into the login flow instead of a registration retry. Soft-deleted rows (status=0) do NOT count as existing — that e-mail is free to register again.

Three-tier rate limit (the exact numbers are a server-side security setting and may change — react to the error code, never hard-code them):
  - **per-(API key, e-mail)** — a short cooldown before the same address can get another code. Stops trivial "spam the same address" attempts. Budget is consumed even when the e-mail is already registered, so timing/rate-limit observation can't be used to scrape the user table at high volume.
  - **per-(API key, IP)** — an hourly cap per visitor's IP address, sized for one person signing up. Primary defence against the SMTP-relay abuse vector: the `fgs_login_*` key is public by design (lives in the browser bundle), so without this cap an attacker could rotate the e-mail in each request and use the endpoint to mail any address in the world. Consequence for the design: call this endpoint from the visitor's browser, not through your own server — behind a server every visitor shares its IP and the cap.
  - **per-(API key, day)** — a daily cap per key. Distributed-attack backstop, fires when the per-IP cap is bypassed by a botnet. Healthy apps stay well under it on legitimate peaks; once it fires, the admin should treat it as a "key is being abused" signal.

All three return the same `errorCode: TOO_MANY_REQUESTS`. The login and registration flows are tracked independently, so saturating one does not affect the other.

A malformed e-mail (invalid format, contains CR/LF) is rejected with `errorCode: REGISTRATION_CODE_REQUEST_FAILED` and message "Invalid e-mail address" before any rate-limit budget is consumed.

The e-mail subject and body sent to the user are configured per API key in **Settings → API → Permissions** (the Self-registration section, separate from the login template). Both support the placeholders `{CODE}`, `{VALID_MINUTES}`, `{EMAIL}` and `{EMAIL_URL}` (the address percent-encoded, for a link's query string).

The code is valid for the period configured per API key in **Settings → API → Permissions** (*E-mail code validity*, 30 minutes by default; the same setting governs the login code), and the response carries that number as `data.valid_minutes` so a frontend can display it without hardcoding it. Calling this endpoint again before that overwrites the previous code (subject to the per-address cooldown).

### Request body (`application/json`)

- `email` (string, format email, required) — E-mail address the new user wants to register with. Must be in a syntactically valid e-mail format; CR/LF characters are rejected as a header-injection defence. The server canonicalises the value (trim + lowercase) before lookup AND storage, so the registered row always carries the canonical form. Must NOT match an existing non-deleted row in the user table — soft-deleted rows (status=0) are ignored, so an e-mail freed by an admin delete is registerable again. Example: `newuser@example.com`.

### Responses

#### 200 — OK response

Content type `application/json`:

- `status` (string) — Example: `ok`.
- `statusInfo` (string) — Example: `A registration code has been sent`.
- `errorCode` (string) — On error, one of: - `EMAIL_ALREADY_REGISTERED` — a non-deleted row with this e-mail already exists. Frontend should route the user into the login flow. - `REGISTRATION_NOT_ALLOWED` — `allow_self_registration` is off on this API key. - `REGISTRATION_CODE_REQUEST_FAILED` — wrong key type / `login_type` mismatch / user-table or e-mail-column not configured / registration status not configured. Operational misconfiguration; surface to the developer, not the end user. - `TOO_MANY_REQUESTS` — rate limit hit. Tell the user to wait and try again.
- `data` (object)
  - `valid_minutes` (integer) — How long the code stays valid, in minutes, as configured for this API key (*E-mail code validity*). Render the code's lifetime from this value instead of hardcoding a number — the admin can change it at any time. Example: `30`.

#### 400 — Error response

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `API key is invalid for this company`.
- `errorCode` (string) — One of: `BAD_REQUEST`.
- `data` (null)


## POST /auth/register

**Register new user (verify code + create row)**

Verify the 6-digit code from `request-registration-code` and complete the self-registration in a single call. Atomically:
  1. validates the code (one-time, TTL configured on the API key — 30 minutes by default),
  2. re-checks the e-mail is still free (race-condition guard — see below),
  3. inserts a new row in the user table with the e-mail filled in and the `Status for new users` configured on the API key,
  4. issues a session token for the brand-new row **only when the new user is immediately active** (see below).

**Auto-login depends on the new-user status.** If the `Status for new users` configured on the API key is also one of the `Statuses allowed for login lookup`, the new user is active immediately: `data.token` is returned and the response shape matches `verify-login-code` exactly, so the frontend can transition straight from the registration form into the logged-in state with no extra round-trip — bind the same post-login handler to both flows. If the new-user status is NOT a login-lookup status, the row is still created but the user is **pending approval**: the call still succeeds with `status: ok`, but `data.token` and `data.expires_in` are `null` and the user cannot sign in until an administrator moves the row into one of the allowed statuses. Inspect `data.token` to tell the two cases apart.

When a token is returned it does NOT replace the login-session key in `Authorization`; on every subsequent data call send BOTH headers:
  - `Authorization: <fgs_login_*>` — unchanged, the same login-session key
  - `X-User-Session-Token: <token>` — the session token returned here

Send the login-session key the same way as in `request-registration-code`: `Authorization: <fgs_login_*>`. The backend API key and `fgs_public_*` are rejected.

**Race-condition guard.** Between `request-registration-code` and this call (for as long as the code stays valid) a row with the same e-mail may have been created by an admin or by another self-registration. Even if the code is valid, the registration is refused with `errorCode: EMAIL_ALREADY_REGISTERED` rather than silently producing a duplicate row. Soft-deleted rows (status=0) do not count — same rule as the request step. An address already used by a linked account (**Sign-in with multiple accounts**) counts as registered too — registration always creates a customer row, never an account row.

**Brute-force defence.** A wrong code does NOT burn the original — the user can retry a few times. Once the allowed wrong attempts are used up, the code is purged and the call returns `REGISTRATION_TOO_MANY_ATTEMPTS`; the frontend should prompt the user to request a new code, which starts over. Do not count attempts in the client — the number is a server-side security setting and may change; the error code is the signal. A successful register wipes the code (one-time use, no replay).

The session TTL is sliding — every successful API call refreshes it server-side, so the just-registered user is not logged out mid-use.

The e-mail must be in a syntactically valid format; CR/LF characters are rejected. The server canonicalises (trim + lowercase) the value, so the same row can be referenced regardless of case differences between the request and verify steps.

### Request body (`application/json`)

- `email` (string, format email, required) — Same e-mail used in `request-registration-code`. Example: `newuser@example.com`.
- `code` (string, required) — 6-digit code received by e-mail. Example: `482157`.

### Responses

#### 200 — OK response

Content type `application/json`:

- `status` (string) — Example: `ok`.
- `statusInfo` (string)
- `errorCode` (string) — On error, one of: - `REGISTRATION_FAILED` — wrong code (attempts remain), invalid e-mail, key-type / configuration mismatch, or row insert failed. The frontend treats it as "code invalid, retry". An insert that fails leaves the code usable and names the reason in `statusInfo`. - `REGISTRATION_TOO_MANY_ATTEMPTS` — too many wrong codes; this code is purged. Prompt the user to request a new code. - `EMAIL_ALREADY_REGISTERED` — race-condition guard fired (see description). Route the user into the login flow. - `REGISTRATION_NOT_ALLOWED` — `allow_self_registration` was toggled off between the request and verify steps; in-flight codes are dead. Operational, surface to the developer.
- `data` (object)
  - `token` (string, nullable) — Session token for the newly registered user, or `null` when the new-user status is not a login-lookup status (pending approval — see description). When present, send it in the `X-User-Session-Token` header on subsequent data calls.
  - `expires_in` (integer, nullable) — Token TTL in seconds, or `null` when no token was issued (pending approval). The TTL is sliding — every successful API call refreshes it server-side. Example: `604800`.

#### 400 — Error response

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `API key is invalid for this company`.
- `errorCode` (string) — One of: `BAD_REQUEST`.
- `data` (null)
