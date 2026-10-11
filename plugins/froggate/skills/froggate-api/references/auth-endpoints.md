# Auth endpoints (login flow)

These endpoints exist only on `fgs_login_…` keys. They issue and verify a per-user session token.

## POST /auth/request-login-code

```
Authorization: fgs_login_…
```
```json
{ "email": "user@example.com" }
```

On success `data.valid_minutes` says how long the code stays valid — display that, do not hardcode it.

An unknown e-mail returns `errorCode: EMAIL_NOT_REGISTERED` when the key has existence disclosure on (the default), or the very same success envelope as a known e-mail when it is off — see "E-mail existence disclosure on `request-login-code`" in `access-model.md`. Other failure codes: `LOGIN_CODE_REQUEST_FAILED` (malformed e-mail, or the address is found on more than one row — across the user table and the accounts table), `TOO_MANY_REQUESTS`.

## POST /auth/verify-login-code

```
Authorization: fgs_login_…
```
```json
{ "email": "user@example.com", "code": "123456" }
```

On success:
```json
{
  "status": "ok",
  "data": {
    "token": "…40 chars…",
    "expires_in": 604800
  }
}
```

The code is one-time — a successful verify spends it. A wrong code answers `errorCode: LOGIN_CODE_VERIFY_FAILED` and may be retried a few times; once the allowed attempts are used up the code is deleted and the answer is `LOGIN_CODE_TOO_MANY_ATTEMPTS` — offer "send a new code" then. Do not count attempts in the app; the number is a server-side security setting and may change. `expires_in` is the session token TTL in seconds (matches the hours configured on the API key).

After this point, every data call must include `X-User-Session-Token: <token>` alongside the same `fgs_login_…` value in `Authorization`. The token slides forward by the configured TTL on every successful call.

## POST /auth/verify-access-token

Only on `fgs_login_…` keys whose **Access mode** is **Access token**. Exchanges a per-record token (the one carried in the shared link) for a session token — the single-step equivalent of `verify-login-code`, with no request-code stage.

```
Authorization: fgs_login_…
```
```json
{ "token": "a1b2c3d4e5f6g7h8i9j0" }
```

On success returns the **same shape as `verify-login-code`** (`{ token, expires_in }`); use the returned token in `X-User-Session-Token` on subsequent data calls. The token resolves only while its record is in one of the statuses configured as reachable for the key.

Failures return `errorCode: ACCESS_TOKEN_VERIFY_FAILED`. An unknown, empty or whitespace token and a record outside the reachable statuses get the very same response, so a guessed token leaks nothing. A token shared by two rows and an incomplete access-token setup on the key carry the same code with their own message — both are configuration errors for the admin, not something the visitor can fix.

## POST /auth/request-registration-code

```
Authorization: fgs_login_…
```
```json
{ "email": "newuser@example.com" }
```

On success `data.valid_minutes` says how long the code stays valid — display that, do not hardcode it.

Disabled unless the API key has **Allow user self-registration** enabled with a **Status for new users**. Returns `errorCode: EMAIL_ALREADY_REGISTERED` when the e-mail is already taken (intentional — see "Self-registration flow" in `access-model.md`). Other failure codes: `REGISTRATION_NOT_ALLOWED`, `REGISTRATION_CODE_REQUEST_FAILED`, `TOO_MANY_REQUESTS`.

## POST /auth/register

```
Authorization: fgs_login_…
```
```json
{ "email": "newuser@example.com", "code": "123456" }
```

On success returns the **same shape as `verify-login-code`**:
```json
{
  "status": "ok",
  "data": {
    "token": "…40 chars…",
    "expires_in": 604800
  }
}
```

The new user row is created in the user table with the e-mail filled in and the `Status for new users` set on the API key. When that status is also one of the key's login statuses, the session is already valid — start using the token in `X-User-Session-Token` immediately. Otherwise the user is **pending approval**: still `status: ok`, but `token` and `expires_in` are `null` and the user cannot sign in until an administrator moves the row into a login status. Check `data.token` before storing it and show a "waiting for approval" screen when it is `null`. Wrong-code failures: `REGISTRATION_FAILED` (with retries left), `REGISTRATION_TOO_MANY_ATTEMPTS` (wrong attempts used up; request a new code), `EMAIL_ALREADY_REGISTERED` (race lost between request and verify).

## Password sign-in: /auth/user-login, /auth/request-password-reset-code, /auth/set-password

`/auth/user-login` signs in with `{ email, password }` and returns the **same shape as `verify-login-code`**. A password is only ever set by the user, so an app offering password sign-in must also ship the set-password flow (`request-password-reset-code` → `set-password`), which also signs the user in. When each step is used and how the set-password e-mail link is built is under "Per-user frontend" in `access-model.md`; request fields and error codes are in `api/auth.md`.
