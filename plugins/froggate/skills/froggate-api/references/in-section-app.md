# Running inside a FROGGATE section (Internal application widget)

This applies when the app is being built **specifically to run inside a FROGGATE section** as the "Internal application" widget — i.e. the user is shipping the app into FROGGATE (typically via **Settings → Aplikace**, then placing it via the Internal application widget on a board), not deploying it on its own domain. Such apps usually don't make sense outside FROGGATE because their entire purpose is to read/write FROGGATE section data with the user's API key.

## How the key gets in

The app **does not embed any API key**. Instead, when the iframe loads, the parent section sends the key via `window.postMessage` to the iframe's window. The app installs a `message` event listener on `window` and reads `event.data`.

**Exact wire format** — the parent calls `iframe.contentWindow.postMessage(payload, appOrigin)` with one of these `payload`s:

| Field | Type | Value | Notes |
|---|---|---|---|
| `event.data.type` | string | `'fg-apikey'` or `'fg-apikey-error'` | **Discriminator.** Use it to filter — `window.message` events come from other sources too (browser extensions, dev tools, third-party SDKs). Ignore any other `type`. |
| `event.data.apikey` | string | e.g. `'fgs_temp_AB12CD34…'` | Only with `fg-apikey`. The API key value for the `Authorization` header. Always starts with the `fgs_temp_` prefix; current length is 49 chars (`fgs_temp_` + 40 random). Treat as opaque — do not parse. |
| `event.data.reason` | string | `'not_linked'`, `'ambiguous'` or `'not_enabled'` | Only with `fg-apikey-error` — no key could be issued for the person looking. See "App with sign-in" below. |

`appOrigin` is the app's own origin — the address it is published on (its main domain when it has one). The iframe is always loaded at that address, taken from FROGGATE rather than from the widget settings, and the message is posted only to it: if the iframe has navigated anywhere else (a link the user followed, a redirect off the app's domain), the browser does not deliver the key at all. `event.origin` reflects the parent FROGGATE origin (e.g. `https://client.froggate.cloud`); the app may check it as belt-and-braces defence, but an attacker cannot forge a valid `fgs_temp_…` value just by sending a `postMessage`.

**When the message arrives:**

- **Right after the iframe loads** — the parent fires it on the iframe's `load` event. The app must already have its listener attached when this happens (attach it before any other work in the entry point).
- **Again whenever the value changes** — if the admin assigns a different parent API key to the app, or the current temp key approaches its TTL window and gets rotated, the parent sends a fresh `postMessage` with the same `type` and a new `apikey`. The app should overwrite its stored value on every such event.

## What the app must do

1. **Wait for the message before making the first API request.** The app does not know the key on load — it has to listen first. Making API calls before the message arrives produces 401.
2. Store the latest `apikey` value and use it as the `Authorization: <key>` header on every API call.
3. Replace the stored value every time a fresh `fg-apikey` message arrives.

Minimal pattern:

```js
let apiKey = null;
let resolveApiKeyReady;
const apiKeyReady = new Promise((r) => { resolveApiKeyReady = r; });

window.addEventListener('message', (event) => {
  if (event.data && event.data.type === 'fg-apikey' && event.data.apikey) {
    apiKey = event.data.apikey;
    resolveApiKeyReady();   // first time only — subsequent updates just refresh `apiKey`
  }
  if (event.data && event.data.type === 'fg-apikey-error') {
    showOwnSignIn(event.data.reason);   // app with sign-in only — see "App with sign-in" below
  }
});

async function call(path, init = {}) {
  await apiKeyReady;
  return fetch(`https://api.froggate.cloud/v1/${companyId}${path}`, {
    ...init,
    headers: {
      'Content-Type': 'application/json',
      Authorization: apiKey,
      ...(init.headers ?? {}),
    },
  });
}
```

The rest of the integration (endpoints, request/response shapes, error envelope) is identical to the patterns in this skill — on the REST API the `fgs_temp_…` key behaves like any other API key (the MCP server refuses it). Authentication, allowed tables, allowed columns and allowed statuses come from the parent key the admin selected; the temp key carries that scope transparently. Everyone who may see the section receives such a key, so tell the user to link a key made for this app — not a personal key, not one with **Full access** — limited to what the screen reads and writes.

## App with sign-in (login key) inside FROGGATE

When the app's key is **restricted to users** (`fgs_login_…` outside FROGGATE) and has **Login with a FROGGATE account** switched on with a **FROGGATE user column** picked, the person looking at the app inside FROGGATE is signed in automatically: the `fgs_temp_…` key the iframe receives acts as the user-table row whose FROGGATE user column names them. It behaves exactly like a session after e-mail sign-in — only that user's rows, shared sections readable, link columns limited the same way — but needs **no session header**: send just `Authorization: <fgs_temp_…>`. It stops working the moment the person no longer maps to exactly that row, leaves the company or the sign-in is switched off; on a 401 the app waits for the next message, which is either a fresh `fg-apikey` or an `fg-apikey-error` saying why there is no key any more.

When no key can be issued, the iframe receives `fg-apikey-error` instead, and the app should offer its normal e-mail sign-in with the same login key:

- `not_linked` — the person is named in no row (in an allowed status) of the user table.
- `ambiguous` — they are named in more than one row; the admin has to fix the data.
- `not_enabled` — the key does not sign in with FROGGATE accounts.

One app therefore has two entrances: inside FROGGATE it is signed in automatically, outside it the user signs in with an e-mailed code or a password. A person with both lands on the same row.

A public key (`fgs_public_…`) cannot fill the FROGGATE user column: an insert carrying it is refused with 403, even when the key may write every other column of the user table. A public sign-up form leaves the column empty; rows are paired with FROGGATE users in FROGGATE or by a backend integration.

Recommended start-up flow for such an app:

1. **Not embedded** (`window.self === window.top`) — the app runs on its own address: show the normal sign-in right away (`fgs_login_…` + session token).
2. **Embedded** — attach the `message` listener first, then wait:
   - `fg-apikey` → the person is signed in; use the key and skip the sign-in screen entirely.
   - `fg-apikey-error` → show the normal sign-in, with a line that matches the reason: `not_linked` — "your FROGGATE account is not linked to an account in this application", `ambiguous` — "your FROGGATE account is linked to more than one account; ask the administrator", `not_enabled` — no extra line needed.
   - Nothing within a few seconds → the app is embedded somewhere other than FROGGATE; show the normal sign-in.
3. To show who is signed in, read the user's own row from the user table with `get-table-rows` — the key returns only that row.
4. Keep the signed-in key in memory only. A session from the normal sign-in inside the iframe is the app's own business; it does not mix with the FROGGATE key.

## What you don't have to handle

- **No login flow** for an app on a plain key — the viewer is already authenticated inside FROGGATE and the temp key represents that. An app on a login key needs its own sign-in only as the fallback above.
- **No key storage.** Never write the value to `localStorage` or a cookie — it expires within ~1 hour and is rotated automatically on the next data fetch from the section side.
- **No refresh logic.** When the key rotates, a new `postMessage` arrives; the app just overwrites its in-memory value.
- **No CORS configuration.** The temp key inherits the parent key's allowed origins; the section sends only valid keys.
