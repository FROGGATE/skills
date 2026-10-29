---
name: froggate-api
description: "Guide to building apps and websites on FROGGATE and integrating with its API. Use when the user wants to build something on FROGGATE or connect something to it, or asks what can be built — server-side backend integration, public-facing presentation website, per-user frontend app with login and registration, a shared-record link (access token), an app that runs inside a FROGGATE section, publishing a web app into FROGGATE with content from sections, automation and webhooks, or a program that reads and posts messages in the company chat. Covers the building blocks and how to set each up in FROGGATE, all API key types (backend, fgs_public_, fgs_login_ incl. the per-record access-token mode, the in-section temporary key), and connecting the FROGGATE MCP server, which building on FROGGATE also needs."
metadata:
  version: "36"
---

# FROGGATE API — Agent Skill

## Update check — once per conversation, before answering

This skill is published at https://github.com/FROGGATE/skills and updated there as the FROGGATE API evolves; the installed copy may be older.

1. Download `https://raw.githubusercontent.com/FROGGATE/skills/main/VERSION` (one integer, plain text) — e.g. `curl -fsSL <url>`.
2. Compare it with `metadata.version` in the frontmatter of this file.
3. **If the published number is higher**, update through the tool that installed the skill, picked by where this file was loaded from. If you can run shell commands, run the update yourself (the user approves it like any other command); otherwise give the user the command:
   - path under `.claude/plugins/synced/`, or you run in claude.ai / Cowork: the update arrives on its own from the user's claude.ai account — do nothing and say nothing
   - path under `.claude/plugins/` (Claude Code plugin): `claude plugin marketplace update froggate && claude plugin update froggate@froggate`
   - path under `.codex/plugins/` (Codex plugin): `codex plugin marketplace upgrade froggate && codex plugin add froggate@froggate`
   - installed with `npx skills`: `npx skills update`
   - anything else: reinstall from https://github.com/FROGGATE/skills

   Tell the user in one sentence that the FROGGATE skill was updated (or how to update it) and that the new version applies from the next session. Then continue with this copy. Never rewrite the skill files yourself.
4. If the numbers match or the download fails, continue without mentioning it.

---

This skill equips an AI agent to build apps and websites on FROGGATE. FROGGATE keeps a company's data and processes in sections and is at the same time the backend of the apps built on it — with no server of their own, hosted by FROGGATE or running inside its sections.

The person you work with is often not a developer — anyone from the business side who knows what they want to achieve. Offer them the product choices in plain words (what each option means for their customers or colleagues), and do the technical work yourself.

## You need two tools: this skill and the MCP server

Building on FROGGATE always takes both:

- **The FROGGATE MCP server** works inside the user's account: it creates and changes sections, columns, statuses and their settings, explains what each setting does (`get_settings_reference`), and reads and writes records while you build and test. It also serves the company's knowledge in FROGGATE (`get_knowledge`) — rules its people wrote for how their sections are read and filled. Read the knowledge of every section your code will write before designing what it writes: the code runs later without an AI, so those rules must be built into it.
- **This skill** covers everything around the code: which key, sign-in flows, the REST API, publishing the app.

**If the MCP server is not connected, set it up before you start building** (see "Connecting the MCP server" at the end of this file). Do not work around it with throwaway scripts that call the REST API to create test records or reshape data: the REST API cannot touch the structure at all, and for data you would lose what MCP adds — the settings reference and the instructions on what each value means. Neither tool creates API keys, publishes an app or adds section actions; for those, tell the user exactly what to set in FROGGATE.

## What you can build — building blocks

Apps on FROGGATE are put together from building blocks. Know which exist before you design anything: the user describes the product, not the blocks, and several blocks do without code what they would otherwise ask you to program. `references/building-blocks.md` says for each block how to set it up in FROGGATE (sections, columns, statuses, settings), which variants exist and which to recommend, what the app must handle and what it combines with.

| Block | What it gives the product |
|---|---|
| Data in sections | where every app keeps its data; statuses are the switch everything else listens to |
| User sign-in | each customer, member or employee signs in and sees only their own records |
| Registration | sign-up from the app — open, or with approval by the company |
| Shared lists for signed-in users | price lists, menus, documents every signed-in user reads |
| Shared-record link | one record (a quote, an order) opened from a link, no sign-in |
| Public data and forms | a public catalogue, a contact or order form, no sign-in |
| Web content filled in at publish time | texts, photos, listings and a page per record written into the site's HTML — seen by search engines |
| Standalone page | the app hosted by FROGGATE on its own address or the customer's domain |
| App inside a section | the app shown inside FROGGATE for the company's own people |
| Files | attachments on records — upload, download, photos |
| Automation | e-mails, status changes, webhooks and documents FROGGATE runs on its own when a record changes |
| Connecting another system | a sync with an e-shop, an ERP or any service — by code, or by webhooks without code |
| Chat | notifications into the company chat, a bot, error reports from a site |

Blocks combine freely — a standalone page with a public catalogue, sign-in and registration with approval is one ordinary product. When the request leaves a product choice open (open sign-up or approval? may customers see each other's records?), ask, and say what each option would give them.

## Where the details are

This file holds what every integration needs. Open a reference when the task reaches it:

| When you… | Read |
|---|---|
| map the user's wish to building blocks, or set a block up in FROGGATE (which sections, columns, statuses, settings) | `references/building-blocks.md` |
| set up or explain a key in detail — backend, public, login (sign-in methods, self-registration, e-mail existence), shared-record link, scope changes, per-table read/write, allowed origins | `references/access-model.md` |
| call `/auth/*` (login code, password, access token, registration) | `references/auth-endpoints.md` |
| read or write rows, statuses, files or selection options, or run a table action | `references/data-endpoints.md` |
| write a value into a long-text (`text`) column — it is HTML with an allow-list | `references/text-column-html-format.md` |
| read or post chat messages | `references/chat-endpoints.md` |
| write the client code — TypeScript types, one client class per scenario, examples | `references/client-code.md` |
| build a frontend app to publish in FROGGATE (Settings → Aplikace) — ZIP layout, build step, reporting errors to a chat channel | `references/publishing.md` |
| put texts and images the customer edits in sections into a published site's HTML (`data-fg-*`) | `references/web-content.md` |
| build an app that runs inside a FROGGATE section (Internal application widget) | `references/in-section-app.md` |
| need the exact request fields, response fields, allowed values or error codes of an endpoint | `references/api/<group>.md` — `auth.md`, `dynamic-tables.md`, `chat.md`, `published-app-report.md` (one per first segment of the endpoint path) |

The `references/api/` files are generated from the OpenAPI specification and are the wire-level truth; the other references explain when and how to use the endpoints. The same specification as one YAML file, e.g. for generating a typed client: `https://client.froggate.cloud/apidoc/api-bundled.yaml`.

## Agent Instructions

### Step 1 — Identify the integration scenario

First map the request to building blocks (above) and settle the open product choices with the user. Then, before you write any code, decide which of the five scenarios fits — the scenario is the choice of key and access, and one product can use several (a public catalogue next to a signed-in area):

1. **Backend integration** — server-side code (cron, webhook receiver, server endpoint, backend service) that talks to FROGGATE. Uses the **backend API key** (prefix `fgs_{companyId}_…`) shown once when generated in admin. It is a secret and stays on the server. If the key was created for an **API user** (a service identity for integrations, not a person), it must additionally be linked to an application in Settings → API (the Application column of the key) — until that is done, the API refuses its requests with an error saying exactly that.
2. **Public-facing presentation website** — a public site/landing page/catalogue/contact form. No login, anyone visiting sees the same data scoped by the key. Uses **`fgs_public_…`** directly in the browser, no backend needed.
3. **Per-user frontend app** — frontend application where each end user signs in with their e-mail (an e-mailed code, a password, or both) and only sees their own rows. Uses **`fgs_login_…`** plus a session token issued by the login flow. Lists every signed-in user may see (a menu, a price list, general texts) come from a **shared section** on the same key — never from a second, public key, whose data anyone can read without signing in. Lets the user build a full SaaS-style frontend with FROGGATE as the only backend, no own backend required.
4. **Shared-record link (no login)** — share a single record (a price quote, an invoice, an order status page) with someone via a link that carries a per-record token; the recipient sees only that one record and its connected records, no sign-in. Uses the **same `fgs_login_…`** key in **Access token** mode plus a session token issued by `/auth/verify-access-token`. See "Capability-link access (access token)" in `references/access-model.md`.
5. **In-section app (Internal application widget)** — a frontend app designed to run inside a FROGGATE section as the "Internal application" widget. It is typically only meaningful inside FROGGATE; it does not embed any key itself, the parent section delivers a short-lived API key to the iframe via `postMessage` at runtime. See `references/in-section-app.md`.

The choice drives where the key lives, which header carries it and which endpoints are reachable, so it must come first. If unsure, ask the user one or two clarifying questions ("does the site need users to log in?", "does the integration run in a browser or on a server?").

### Step 2 — Direct the user to the admin UI

Tell the user where to set things up in their FROGGATE client account:

- Open the client account (the URL contains `?company={companyId}` — the numeric ID is what you need).
- Go to **Settings → API**.
- For an existing key, click **Permissions** to open the scope modal; otherwise create a new key first and then click **Permissions**.

The Permissions modal is the single source of truth for everything that affects this skill:

- Allowed tables, with per-table allowed columns and allowed statuses.
- Per-table access mode: **read-only**, **write-only**, or **read+write**.
- Allowed origins (CORS whitelist).
- "Restrict to user" — turning this on converts the key into a login-flow key (`fgs_login_…`). Then pick the **Access mode**: **User login** (pick the user table + e-mail column — end users sign in with their address, using an e-mailed code, a password, or both; with **Login with a FROGGATE account** and a **FROGGATE user column**, the person opening the app inside FROGGATE in the Internal application widget is signed in automatically as the row naming them, see "App with sign-in" in `references/in-section-app.md`; with **Sign-in with multiple accounts** switched on, rows of a linked accounts table — e.g. contact persons pointing at the customer through a *Record from another table* column — sign in as that customer with their own addresses and passwords) or **Access token** (pick the shared table + the access-token column — recipients open a per-record link, no login). See "Per-user frontend" and "Capability-link access" in `references/access-model.md`.
- **Shared section, read-only** — for a login-flow key, in the detail of an allowed table that is not linked to the user table. Every signed-in user then reads all records of that section in the allowed statuses and columns; writing into it is refused. Only sections that neither hold nor link to records of individual users can be shared — for any other the switch is not offered. See "Per-user isolation" in `references/access-model.md`.
- Generate / regenerate the **public key** (`fgs_public_…`) — only available when the key is *not* restricted to user and *not* full-access.
- Generate / regenerate the **login session key** (`fgs_login_…`) — only available when "Restrict to user" is on.
- Session token TTL (hours) — applies to tokens issued through the login flow. Default 168 (7 days). The user can set this to whatever fits their app (short for sensitive data, long for low-friction).
- **Read chat** / **Write to chat** (both off by default; full access does not include them) — let the key use the chat endpoints. See `references/chat-endpoints.md`.
- **Allow user self-registration** (off by default) — opt-in toggle that enables `/auth/request-registration-code` and `/auth/register`. When you enable it, also pick **Status for new users** (the status assigned to the row created by the registration flow) and customise the registration e-mail template if the default Czech wording does not fit. If the user wants public sign-up on their app, point them here — without these toggles the registration endpoints reject every request.

Ask the user for:
- the **company ID** (from the URL),
- the **right type of key** for the chosen scenario.

Never ask the user to paste the key into chat. Use placeholders (e.g. `<paste-your-key-locally>`) and ask them to fill it in outside the conversation.

### Step 3 — Identify the table and operation

Which table, which columns, which operation. Tip: enable Developer Mode (user icon → Edit profile → Developer mode) to see numeric IDs next to UI elements.

If the user does not know the table ID, suggest calling `get-active-tables` first — it lists everything the key has access to.

### Step 4 — Implement using the right pattern

Read `references/client-code.md` before writing the client — there is one client class per scenario.

### Step 5 — Plan the deployment for frontend apps

Frontend apps using `fgs_public_…` or `fgs_login_…` are typically hosted directly inside FROGGATE (**Settings → Aplikace** — upload a ZIP, no own hosting needed). When you are about to help build such an app, read `references/publishing.md` **before** scaffolding the project — the ZIP layout and the `process.env.NODE_ENV` fix are easier to set up upfront than to retrofit.

---

## Choosing the right key

| Scenario | Key prefix | Where it lives | Backend needed? |
|---|---|---|---|
| Backend integration | `fgs_{companyId}_xxxxx-…` | server-side env | yes |
| Public website | `fgs_public_…` | directly in browser JS | no |
| Per-user frontend | `fgs_login_…` (+ session token after login) | directly in browser JS | no |
| Shared-record link | `fgs_login_…` in Access-token mode (+ session token) | inside a link / browser JS | no |
| In-section app | `fgs_temp_…` | **not embedded** — delivered to the iframe at runtime via `postMessage` | no |

What each key may do, how its scope is set and how the login, registration and access-token flows behave is in `references/access-model.md`.


## Storing the key safely

- **Backend API key** (`fgs_{companyId}_…`) — server-side env (`.env` / `.env.local` / hosting platform secret store). Never put into `NEXT_PUBLIC_*`, `VITE_*`, `REACT_APP_*` or any browser bundle. Add the env file to `.gitignore` before writing any code. Tell the user clearly: this key gives full configured access to the FROGGATE tables.
- **`fgs_public_…`** — designed to ship to the browser, fine in `NEXT_PUBLIC_*` style variables. Avoid committing it to a public repo when easily avoidable, but it is not a secret — its safety relies on the admin's scope, not on hiding the value.
- **`fgs_login_…`** — same as `fgs_public_…`. The session token returned by `/auth/verify-login-code` belongs in browser storage (`localStorage`, `sessionStorage` or a secure cookie, depending on the app's needs).

When you need to show the user a setup snippet, use placeholders and ask them to fill the real value in outside chat.

---

## Base URL & Authentication

```
Base URL: https://api.froggate.cloud/v1/{companyId}
```

All requests:
```
Content-Type: application/json
Authorization: <key>
```

The `Authorization` header carries the key value directly — no scheme prefix. A `Bearer ` prefix is tolerated, so existing code that sends it keeps working; write new code without it.

For login-flow data calls (after `/auth/verify-login-code` returned a token), add a second header:
```
Authorization: fgs_login_…
X-User-Session-Token: <token from verify-login-code>
```

### Response envelope

Every response has this shape:

```json
{
  "status": "ok",
  "statusInfo": null,
  "errorCode": null,
  "data": { ... }
}
```

On error:
```json
{
  "status": "error",
  "statusInfo": "User session token is required for this API key",
  "errorCode": "BAD_REQUEST",
  "data": null
}
```

Always check `response.status === "ok"` before processing `data`. One exception: on `run-action`, the `ACTION_PARTIALLY_FAILED` / `ACTION_FAILED` response of a per-row action keeps `data` populated with what already ran — see the run-action section of `references/data-endpoints.md` before discarding it.

### HTTP status / error reference

- `200` with `status: "error"` — a business error. Branch on `errorCode` (stable), never on `statusInfo` (human text, may change). Each endpoint's own codes are listed in `references/api/`; on any endpoint you can also get `MISSING_PARAMETER` / `INVALID_PARAMETER` (fix the call — `statusInfo` names the field and the allowed range) and `INTERNAL_SERVER_ERROR` (our side — `statusInfo` carries a reference to quote when reporting it). Handle every code the endpoint lists: you will rarely trigger them while developing, and an unhandled one reaches the end user as a generic error.
- `401 Authorization by API key is required` / `401 API key is invalid for this company` — the key is missing, or it is not a key of this company.
- `401 Auth endpoints require a login_session_key (fgs_login_…)…` — an `/auth/*` endpoint was called with a backend API key; use the `fgs_login_…` key.
- `401 User session token is required for this API key` — login-flow key called a data endpoint without `X-User-Session-Token`.
- `401 Invalid or expired user session token` — session token expired or rejected. Run the login flow (or `/auth/verify-access-token`) again.
- `error ACCESS_TOKEN_VERIFY_FAILED` — `/auth/verify-access-token` could not resolve the token. An unknown or empty token and a record outside the reachable statuses get one identical response (no information leak); a token shared by two rows or an incomplete setup on the key has its own message — a configuration error for the admin.
- `401 login_session_key invalid …` / `401 public_key only valid when …` — admin changed the scope flags, the key has been wiped. Regenerate in Permissions.
- `401 This temporary key is not signed in …` — an Internal application's `fgs_temp_…` key from before the admin restricted the app's key to users. Wait for the next `fg-apikey` / `fg-apikey-error` message (or reload the application).
- `403 Origin not allowed for this API key` — domain not in allowed origins.
- `403 This endpoint is not available for user-restricted API keys` — login-flow keys can only call the listed endpoints.
- `403 This endpoint is not available for a public API key` — a public key is append-only (read + insert only); `update-table-row` and deletes are refused. See "Public website" in `references/access-model.md`.
- `403 The updateOnDuplicateColumnId parameter is not available for this API key type` — upsert is blocked for public and per-user login-flow keys, because its row match can reach a row the caller doesn't own.
- `403` table/column scope errors — the call is outside what the admin allowed; widen the scope or change the call.
- `403` run-action scope refusals carry a specific `errorCode` (`ACTION_NOT_ALLOWED`, `TABLE_NOT_ALLOWED`, `ROW_STATUS_NOT_ALLOWED`, `ROW_NOT_ALLOWED`) — see the run-action section of `references/data-endpoints.md`.
- `403 CHAT_NOT_ALLOWED` — the key may not read / write the chat, or it is a public, login or temporary key (those never reach the chat). See `references/chat-endpoints.md`.

---


## Connecting the MCP server

Sections, columns, statuses and their settings are managed exclusively through the FROGGATE MCP server — the REST API has nothing to call for them. The server is purpose-built for AI work inside the account: it ships its own agent instructions, a reference of every setting, flat tool arguments and per-field validation. Connect it before you start building (see "You need two tools" at the top).

### How to set it up

1. Tell the user you are about to configure the FROGGATE MCP server in their AI client (Claude Desktop / Cursor / VS Code / Codex / similar). Wait for confirmation before touching any config files.
2. The MCP endpoint URL is:
   ```
   https://mcp.froggate.cloud/{companyId}
   ```
3. Authentication uses the same `Authorization: <key>` header as the data API (the key value directly, no scheme prefix). MCP takes only the user's own **personal key** (**Settings → API**) — the AI then acts as them, with their permissions and their name in the history — and the key stays out of any committed config file. An API user's key is for integrations and apps on the data API; MCP refuses it. To create sections, columns and statuses the key needs **Full access** in its Permissions; a key limited to chosen tables may not change the structure. MCP refuses with 403 a key with **Restrict to user** (`fgs_login_…`) and every temporary key an app receives inside FROGGATE (`fgs_temp_…`): apps and their signed-in users work through the REST API.
4. Add the server to the user's AI client — the way depends on where the client runs:
   - **A client on the user's machine** (Claude Code, Codex, Cursor, VS Code, …): an `mcpServers`-style entry pointing to the URL above with the key in the `Authorization` header. Once you have the user's confirmation, perform the edit yourself if the tooling allows it; otherwise hand the user an exact snippet for their client's config file.

     **Lock the key to the user's IP — say this whenever you ask for or read such a key.** It lives in the client's config file, which you (the agent) read in order to set the server up — so the plaintext key is necessarily exposed to the agent and to anything that can read that file or the user's machine. The mitigation is a network-level lock: in **Settings → API → Permissions**, enable **Restrict to IP addresses** and list the user's own IP address(es) or CIDR range(s), one per line (e.g. `203.0.113.7`, `203.0.113.0/24`, `2001:db8::/32`). A key locked this way is useless from any other address even if it leaks. **Allowed origins (CORS)** does nothing here — server-side callers send no `Origin` header. A user on changing networks (a laptop on the road) cannot use the lock; say so rather than lock them out.
   - **claude.ai, the Claude app or Cowork**: you cannot edit their configuration — tell the user to add it themselves under **Customize → Connectors → Add custom connector**: the URL above, authentication **No sign-in**, and under **Request headers** the header `authorization` with the key as its value. Claude offers *Request headers* only to some accounts; without it the server cannot be connected there — the user needs a client on their machine. Each person adds the connector themselves, with their own personal key. Never suggest adding it for a whole organization: the key in its header would be shared by everyone who uses it, so they would all act with one person's permissions without knowing which. These connectors call FROGGATE from Anthropic's servers, not the user's machine, so **do not recommend the IP lock here** — it would block the connector.

   The user's step-by-step guide in the FROGGATE manual: `https://client.froggate.cloud/documentation/konfigurace/vyvoj-s-ai.md`.
5. Once connected, structural work happens through the MCP server's own tools. It publishes them together with its agent instructions on first connect — read those and follow them; this skill does not duplicate the tool list.

The data endpoints in this skill are the **runtime data flow** of the app you build. Work in the account while you develop — the structure, test records, checking what a setting does — is what MCP is for.
