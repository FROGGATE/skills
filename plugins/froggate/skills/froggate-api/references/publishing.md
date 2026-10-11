# Publishing a frontend app to FROGGATE (Settings → Aplikace)

Frontend apps built against `fgs_public_…` or `fgs_login_…` are usually hosted directly inside FROGGATE — no own hosting needed. The user goes to **Settings → Aplikace**, creates an app, uploads a ZIP, activates a version, and FROGGATE serves it as static HTML/JS/CSS from a managed URL.

Mention this whenever the user is about to ship a frontend you helped build.

## The ZIP layout

- **`index.html` must be at the root of the archive** (no folder prefix). The archive root is served directly, so all assets must be at the same root-relative paths that `index.html` references.
- Max upload size is 50 MB.

## Automate the ZIP step in the build

If the user's bundler emits a `dist/` (or `build/`, `out/`, …), wire the ZIP step into the build script so `npm run build` produces the uploadable artifact every time. Don't make the user zip by hand — the two pitfalls below are easy to hit.

**Pitfall 1 — the ZIP must not contain itself, and must have no folder prefix.**

Wrong: `cd dist && zip -r dist.zip .` — the archive is being written into the directory it's archiving, so it can capture itself recursively (corrupt / oversized file).
Wrong: `zip -r dist.zip dist` from the parent — embeds a `dist/` prefix inside the archive, so there is no `index.html` at its root and the upload is refused.

Reliable pattern: build the ZIP to a temp path *outside* `dist/`, then move it into `dist/` after the archiver has finished:

```js
// post-build script — bundler-agnostic, adapt the dir name if needed
import { execSync, renameSync } from 'node:child_process';
import path from 'node:path';
const distDir = path.resolve('dist');
const tmpZip = path.resolve('.app-bundle.zip.tmp');
execSync(`zip -rq "${tmpZip}" .`, { cwd: distDir });
renameSync(tmpZip, path.join(distDir, 'app-bundle.zip'));
```

Hook it into the build script (e.g. `"build": "<bundler-build> && node scripts/zip-dist.mjs"`).

**Pitfall 2 — `process.env.NODE_ENV` must be inlined to `"production"` at build time.**

Bundlers in **app mode** typically inline this automatically; **library mode** does not. React 19, the React scheduler and several other libraries read `process.env.NODE_ENV` at runtime — the browser has no `process` global, so the page crashes immediately with:

```
Uncaught ReferenceError: process is not defined
```

Set it explicitly in the bundler config — the knob is bundler-specific:

- **Vite:** `define: { 'process.env.NODE_ENV': JSON.stringify('production') }` in `vite.config.ts`.
- **webpack:** `new webpack.DefinePlugin({ 'process.env.NODE_ENV': JSON.stringify('production') })`.
- **Rollup:** `@rollup/plugin-replace` with `'process.env.NODE_ENV': JSON.stringify('production')` and `preventAssignment: true`.
- **esbuild:** `--define:process.env.NODE_ENV=\"production\"` (or the equivalent `define` option).

Sanity check after fixing: the gzipped bundle typically drops ~30 % because dev-only React branches get tree-shaken. If the size did not drop, the replacement did not propagate — re-check the config.

## Server-rendered content from sections (`data-fg-*`)

A published app can have editorial content — headlines, descriptions, photos, contact
details — **written into its HTML on the server** from FROGGATE sections, instead of
fetched by JavaScript at runtime. You mark an element with a `data-fg-*` attribute and
FROGGATE fills it in when the content is published.

**Reach for this whenever you are building a presentation website whose texts and
images the customer will maintain in FROGGATE.** Content fetched by JavaScript is
frequently not indexed by search engines, which for a marketing site defeats the
purpose; server-rendered content also removes the API round-trip on page load. Keep
the API for per-visitor or interactive data (search, forms, logged-in areas).

**Read the full reference before writing any `data-fg-*` markup:** `web-content.md`
next to this file.

It covers the attribute list, how values are addressed, what the customer has to set
up in FROGGATE (and how to walk them through it), the fallback rule that keeps the
site working in local development, how photos are published, when the content
refreshes, and — importantly — what the mechanism deliberately cannot do, so you do
not design a layout around a capability that does not exist.

## Reporting errors (or anything) to a chat channel

A published app can send short reports into a channel of the company chat — typically
JavaScript errors that a visitor sees and nobody else would. The customer creates a
**report key** in **Settings → Aplikace → (app) → Hlášení**: a name, the channel, and the
API user the reports are posted as. Ask the user to copy the key's address from there — it
already carries `?t=<token>`, and the request needs **no API key and no headers**. Several
keys can point to different channels, so the site sorts reports by choosing the address —
there is no "type" field. Exact schemas: `api/published-app-report.md` next to this file.

```json
{"message": "TypeError: x is undefined", "details": {"Page": "/cart", "Source": "index.js:12:40"}}
```

- `message` is required. `details` is optional: label → value, shown in the chat as bullet
  points. Unknown top-level fields are ignored; a body that is not a JSON object is taken
  as the message itself.
- **No HTML.** Everything is shown as plain text — anyone who knows the address can post,
  and markup in an internal channel would look like the company's own message. Put
  structure into `details`.
- **Do not set an `application/json` Content-Type** — it would trigger a CORS preflight.
  `fetch(address, {method: 'POST', body: JSON.stringify(report)})` and
  `navigator.sendBeacon(address, JSON.stringify(report))` both send `text/plain`;
  `sendBeacon` also works while the page is being closed.
- The response is always JSON with `status`: `202 accepted` (an unknown or switched-off
  address answers the same, on purpose), `429 dropped` with `retryAfter`, `413 dropped`
  (body over 64 kB), `400 invalid` (fix the payload). Never retry a report automatically.
- Limits per minute: 10 reports from one IP address for one key, 20 for the whole key, 60
  from one IP address across all sites, and the same report only once. Dropped reports are
  summed up in one message in the channel after the minute.
- **The address is not a secret** — it ships in the site's code. If it gets abused, the
  customer generates a new address in the same tab and the site must be rebuilt with it.

Guard the reporter itself: one report per distinct message per page load, a cap per page
load, and never report a failed report.

```js
const REPORT_URL = '<key address from Settings → Aplikace → Hlášení>';
const sent = new Set();
window.addEventListener('error', (e) => {
  const report = {
    message: e.message,
    details: {
      Source: `${e.filename}:${e.lineno}:${e.colno}`,
      Page: location.href,
    },
  };
  const fingerprint = JSON.stringify(report);
  if (sent.has(fingerprint) || sent.size >= 10) {
    return;
  }
  sent.add(fingerprint);
  navigator.sendBeacon(REPORT_URL, fingerprint);
});
```
