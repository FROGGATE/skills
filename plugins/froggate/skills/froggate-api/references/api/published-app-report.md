<!-- Generated from the FROGGATE OpenAPI specification (https://client.froggate.cloud/apidoc/api-bundled.yaml) — do not edit by hand. -->
# API reference: `/published-app-report/…`

Base URL: `https://api.froggate.cloud/v1/{companyId}` — replace `{companyId}` with the numeric company ID. Field names, types, allowed values and error codes below are exact.

Endpoints: `/published-app-report/send`

## POST /published-app-report/send

**Send a report to a chat channel**

Sends a short report — typically a JavaScript error a visitor saw — into the chat channel of a **report key** of a published app (Settings → Aplikace → (app) → Hlášení). The key also sets the API user the report is posted as.

**No API key and no headers.** The key's address carries its token in `t`; it ships in the site's code, so it is not a secret. If it gets abused, the customer generates a new address and the site must be rebuilt with it.

**Send the JSON body without an `application/json` Content-Type** — that header would trigger a CORS preflight. `fetch(address, {method: 'POST', body: JSON.stringify(report)})` and `navigator.sendBeacon(address, JSON.stringify(report))` both send `text/plain`; `sendBeacon` also works while the page is being closed. A body that is not a JSON object is taken as the message itself. Unknown top-level fields are ignored. The body may have at most 64 kB.

Limits per minute: 10 reports from one IP address for one key, 20 for the whole key, 60 from one IP address across all sites, and the same report (same message and details) only once. Dropped reports are summed up in one message in the channel after the minute.

Never retry a report automatically.

**Address:** `https://{host}` (not the API base URL)
- `{host}` — Host of the report key's address. Copy the whole address (it already carries `?t=`) from Settings → Aplikace → (app) → Hlášení. Default: `client.froggate.cloud`.

**Authentication:** none — no API key is sent.

### Parameters

- `t` (in query, string, required) — The report key's token — part of the address copied from the settings.

### Request body (`text/plain`)

- `message` (string, required) — The report text, shown in the channel as plain text — formatting, links and mentions are not interpreted. Cut at 4 000 characters. Example: `TypeError: x is undefined`.
- `details` (object) — Optional. Label → value, shown in the channel as bullet points under the message. Only the first 20 are shown; a label is cut at 100 characters, a value at 1 000, a `null` value is left out and a value that is not text is shown as JSON. Example: `{"Page":"/cart","Source":"index.js:12:40"}`.

### Responses

#### 202 — Accepted. An unknown or switched-off address answers the same, on purpose.

Content type `application/json`:

- `status` (string, required) — One of: `accepted`, `dropped`, `invalid`.
- `error` (string) — Why the report was dropped or refused. Missing for `accepted` and for a report dropped by a limit. One of: `tooLarge`, `messageRequired`, `invalidDetails`, `methodNotAllowed`.
- `retryAfter` (integer) — Only with a limit: seconds until the current minute ends. Do not retry the report itself.

#### 400 — The payload is wrong: `message` is missing, empty or not a string (`messageRequired`), or `details` is not an object (`invalidDetails`).

Content type `application/json`: the same body as 202.

#### 405 — Any method other than POST.

Content type `application/json`: the same body as 202.

#### 413 — The body is over 64 kB.

Content type `application/json`: the same body as 202.

#### 429 — Dropped by a limit. The `Retry-After` header carries the same number of seconds as `retryAfter`.

Content type `application/json`: the same body as 202.
