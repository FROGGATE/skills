# Building blocks — what you can build and how to set it up

Every app on FROGGATE is put together from the blocks below. Each says how to set it up in
FROGGATE, which variants exist and which one to recommend, what the app must handle and what it
combines with. Map the user's wish to blocks, settle the open product choices with them, then build.

**Who sets what.** Sections, columns, statuses and most of their settings you create yourself
through the FROGGATE MCP server — read its `get_settings_reference` before each column or setting;
it says which type fits which need and what MCP cannot set yet. API keys and their permissions,
publishing an app, section actions and add-ons the user sets in FROGGATE: give them the exact list
of what to set, never a vague "configure the permissions".

Contents: Data in sections · User sign-in · Registration · Shared lists for signed-in users ·
Shared-record link · Public data and forms · Web content filled in at publish time · Standalone
page · App inside a section · Files · Automation · Connecting another system · Chat

## Data in sections

The foundation of every app. The data lives in sections, and the company's people work in
FROGGATE with the same records the app shows.

Set up (MCP):

- **One section per kind of thing** — customers, orders, products — never one per screen of the
  app. Records of different kinds are connected by a **link column** (*Record from another table*,
  `from_other_table`) on the "many" side: an order links to its customer, an order item to its order.
- **A short fixed list is a selection column; a list whose items have their own details is a
  section.** Categories with their own page, photo or description on a website are a section the
  products link to; "size S / M / L" is a selection column.
- **Statuses are the switch the rest of the system listens to.** Every record is in exactly one
  status, and keys (which statuses they read and write), sign-in (who may sign in), shared links
  (whether a link works) and the website (what is published) all decide by status. Model the
  record's life as statuses — *New → Confirmed → Shipped* — and add a status for "out" wherever
  something must stop working: *Archived*, *Expired*, *Blocked*.
- **Let FROGGATE compute what it can** — a formula, a sum from a linked section, a value taken from
  the linked record. The app reads such a column and never writes it, and the company sees the same
  number the app shows.

The app must handle: each value in its column's `valueFormat`; computed columns are read-only
(`notEditableReason` says why); deleting a record is moving it to status `0`. A column's type cannot
be changed once it exists, so settle it with the user before creating it.

Details: `references/data-endpoints.md`.

## User sign-in

Each customer, member or employee signs in to the app and sees only their own records.

Set up:

- MCP: a **users section** — one record per user, a short-text **e-mail column**, and a
  **password column** if they sign in with a password. Statuses: at least one they may sign in from
  (*Active*); a status outside the sign-in statuses (*Blocked*) is how the company locks someone out.
- MCP: every section holding a user's own records (orders, requests, bookings) gets a **link column
  straight to the users section**. Isolation works only through that direct link: order items linked
  just to their order are not readable by the signed-in user, so give them their own link to users
  too — the API fills it on insert by itself.
- User: in **Settings → API**, a key with **Restrict to user** and *Access mode* **User login** —
  the users section, its e-mail column, the statuses that may sign in and the sign-in methods; then
  the allowed sections with their columns and read/write. Generate the login key (`fgs_login_…`).

Variants:

- **E-mailed code** — nothing to remember or reset. Recommend it unless users sign in every day.
- **Password** — quicker for daily use, but the app must ship the set-password flow: a password is
  only ever set by the user, through a link the app or the company sends.
- **Both** — the code doubles as the way in when the password is forgotten.
- **Several people per customer** (*Sign-in with multiple accounts*) — contact persons in their own
  section, linked to the customer; each signs in with their own address and works with the
  customer's data.
- **Signed in by their FROGGATE account** — the company's own people using the app inside FROGGATE
  (see *App inside a section*).

The app must handle: which methods the key has on (never assume); whether sign-in admits that an
address is unknown (a per-key setting); `TOO_MANY_REQUESTS` as "wait and retry"; `401` on an expired
session → sign in again; the code's validity from `valid_minutes`.

Combines with: Registration, Shared lists, Files, Automation (an action run on the user's own
records), App inside a section.

Details: `references/access-model.md` (Per-user frontend), `references/auth-endpoints.md`.

## Registration

New users sign themselves up in the app instead of the company adding them.

Set up — on top of *User sign-in*:

- **Open sign-up** — one status: *Active* is both the status for new users and a sign-in status.
  The new user is in the app right after typing the code.
- **Sign-up with approval** — two statuses: *Waiting for approval* (status for new users, **not** a
  sign-in status) and *Active* (sign-in status); add *Rejected* if the company keeps refused requests.
  Approving is moving the record to *Active* — by hand, or with a button action. To let the user
  know, add an **Automatic e-mail** action (*Automatický e-mail*) sent when a record enters *Active*,
  to the address in the e-mail column.
- User: on the key, **Allow user self-registration**, **Status for new users**, and the wording of
  the registration e-mail (the default is Czech).

Recommend approval whenever the company decides who gets in — a B2B portal, a member area, anything
with prices or data not meant for everyone; open sign-up for a consumer app anyone may join. Ask
the user which one they want.

The app must handle:

- **Registration stores only the e-mail address.** With open sign-up the user is signed in right
  away, so the app asks for name or company next and saves them into the user's own record. With
  approval there is no session until someone approves, so the approver sees just the address — if
  they need more to decide, the app asks for it after the first sign-in, or the company collects it
  another way (e.g. a FROGGATE *Public form*).
- A user waiting for approval is "not registered" to sign-in and "already registered" to
  registration — show "your account is waiting for approval" instead of bouncing them.
- `EMAIL_ALREADY_REGISTERED` on registration → offer sign-in.

Details: `references/access-model.md` (Self-registration flow).

## Shared lists for signed-in users

Data every signed-in user reads and nobody owns — a price list, a menu, a catalogue, general
documents.

Set up:

- MCP: an ordinary section **without** a link to the users section.
- User: on the login key, the section among the allowed tables, and in its detail **Shared
  section, read-only**.

Every signed-in user reads all its records in the allowed statuses and columns; nobody writes to it
through the app, and without a session nothing comes back. Use it instead of a second, public key,
whose data anyone can read without signing in. A section that holds or links to users' records
cannot be shared — if everyone should see their own records there, give it a link column to users.

Details: `references/access-model.md` (Per-user isolation).

## Shared-record link

One record — a price quote, an order, an invoice, a booking — opened from a link, with no sign-in.
The recipient sees that record and the records linked to it (the items of the quote), nothing else.

Set up:

- MCP: in the shared section a short-text **token column**, filled per record with a long random
  string. The **Generate value** action (*Vygenerovat hodnotu*, 20+ letters and digits) fills it
  without code — run it from a **Scenario** (*Scénář*) when a record is added. Statuses: one in
  which the link works (*Sent*) and at least one in which it stops (*Expired*, *Withdrawn*) — moving
  the record there is how the company switches the link off.
- MCP: what the recipient may change goes into ordinary columns with write access on the key — a
  selection column *Response* (Accepted / Declined), a note, a signed PDF as a file. The recipient
  cannot change the status; to move the record when they answer, add a **Scenario** that runs when
  *Response* changes and sets the status.
- User: a key with **Restrict to user** and *Access mode* **Access token** — the shared section,
  the token column and the statuses in which the link works.
- The link goes out by e-mail: a **Send e-mail** or **Automatic e-mail** action with the token
  column's code in the address (`https://your-app/quote?t=<token column code>`).

The app must handle: `ACCESS_TOKEN_VERIFY_FAILED` → "this link is no longer valid" (expired,
withdrawn or mistyped — deliberately not told apart). It is a bearer link: whoever has it gets in.
When it matters *who* is looking, use sign-in instead.

Details: `references/access-model.md` (Capability-link access).

## Public data and forms

A public catalogue, availability, a contact or order form — anyone can use it without signing in.

Set up:

- MCP: the sections to read and the section the form writes into (*Inquiries*, with a status new
  records land in, *New*).
- User: a key **without** full access and without *Restrict to user* — only the sections, columns
  and statuses that are public for **every** record; read for the catalogue, write for the form;
  allowed origins = the site's domain. Generate the public key (`fgs_public_…`).

Variants:

- **Content that is the same for every visitor** (texts, a product list, photos) — prefer *Web
  content filled in at publish time*: search engines see it and nothing loads at runtime. Read
  through the public key only what changes while the page is open or depends on what the visitor
  does (search, live availability).
- **A form that is part of the site's own design** — insert through the public key. **A standalone
  form is enough** — a FROGGATE **Public form** action (*Veřejný formulář*) needs no code at all;
  offer it before building one.
- To tell the company about a new submission: a **Scenario** that runs when a record is added and
  sends an e-mail or a **Chat message** (*Zpráva do chatu*).

The app must handle: the public key only reads and inserts, never updates or deletes — anything the
visitor edits later needs sign-in or a shared-record link.

Details: `references/access-model.md` (Public website).

## Web content filled in at publish time

Texts, photos, listings and a page per record written into the site's HTML by FROGGATE — what a
presentation website, a catalogue or a blog needs. Search engines see it and nothing is fetched
when the page loads.

Set up:

- MCP: a section per kind of content — *Pages* (a record per block of text), *Products*,
  *Categories*, *References* — with a column per text field (a column per language on a multilingual
  site), a link column from a product to its category, and a short-text column with a code that
  never changes, for readable page addresses.
- **What goes on the web — by status or by a selection column.** Statuses *Draft* and *Published*,
  with the site's key allowed only *Published*, is the simplest: the company publishes a record by
  moving it. A selection column (*On the web*: yes / no) fits when publishing must not change the
  record's workflow status or differs per language; the site then filters on it.
- Photos: **Nastavení sekce → Soubory → Automatický veřejný odkaz** (*Automatic public link*,
  optionally for one file type such as *Web photo*), so every uploaded photo is usable on the site.
  Tell the company to keep photos at 300 kB or less.
- User: in **Settings → Aplikace** the app, a key that reads those sections (**API klíč** tab),
  then on **Obsah ze sekcí** switch on **Automatická aktualizace**.

Variants:

- **Fixed texts** — one value per marked element (texts, long texts, SEO tags).
- **Listings** — markup repeated per record, narrowed by equality filters.
- **A page per record** — a product, an article, a reference. **Category pages**: generate a page
  per category and list on it the products linked to that category; a category the company adds
  gets its page with its products automatically, with no change to the site.
- **Multilingual** — a template per language, a column per language.

The app must handle: only `.html` files are processed (no client-rendered pages); every marked
element keeps fallback content; an edit reaches the site within minutes; nothing differs per
visitor — per-visitor or signed-in content goes through the API.

Combines with: Public data and forms (the runtime part of the same site), User sign-in (a signed-in
area next to the public pages), Standalone page.

Details: `references/web-content.md` — read it before writing any markup.

## Standalone page

The app hosted by FROGGATE on its own address — no hosting and no server of your own.

Set up (user): **Settings → Aplikace** — create the app, upload the ZIP (`index.html` at its root),
activate the version. It runs at `https://<address>-<companyId>.client-app.logea.com/`, and on the
customer's own domains through a `CNAME` record and FROGGATE support. A report key (**Hlášení**)
sends the site's JavaScript errors into a chat channel.

Any frontend works as long as it builds to static files — set up the ZIP step in the build from the
start. The page reaches data through a public key, a login key or a shared-record link, and its
content can be filled in at publish time.

Details: `references/publishing.md`, `references/web-content.md` (the site's domain).

## App inside a section

The app shown inside FROGGATE, in a section, for the company's own people — a screen next to the
data it works with: a technician's checklist over a job, a planning view, a calculator over the
section's records.

Set up (user): publish the app (*Standalone page*), link a key to it on its **API klíč** tab, then
in the section's settings → **Doplňky** (add-ons) add **Internal application** (*Interní aplikace*)
with that app and switch it on. A section of the *Dashboard* type can hold it as its whole content.

The linked key decides what every viewer can do: each colleague who may see the section receives
a temporary key with that key's scope, and can use it on the REST API outside the app too. So link
a key made for this app — never someone's personal key and never one with **Full access** —
limited to the sections, columns and statuses the screen works with, read-only where it only reads.

Variants:

- **Plain key** — every colleague who may see the section works with that key's scope and in its
  user's name: the same data for all of them, and changes show the key's user as their author. Use
  a key of an **API user** (a service account), so the history says the app made the change. Suits
  a team tool over shared data.
- **Signed in by their FROGGATE account** — a login key with *Login with a FROGGATE account* and a
  *FROGGATE user* column in the users section: each colleague is the record naming them and sees
  only their records. The same app works outside FROGGATE with the normal sign-in.

The app must handle: no key in the code — wait for the key's `postMessage`, keep it in memory only.

Details: `references/in-section-app.md`.

## Files

Attachments on records — contracts, photos, signed PDFs.

Set up:

- Files belong to a record; no column is needed. **File types** (*Typy souborů*) in the section's
  settings sort them (*Contract*, *Photo*) and, where the section restricts files per type, decide
  who may open them.
- User: on the key, per allowed section, **File read** and **File upload** — both off by default and
  independent of the columns.
- Photos for a website need a permanent public link — see *Web content filled in at publish time*.

The app must handle: show images by `publicUrl`, never by downloading their content through the API;
upload as base64 in the insert or update call, the type by `fileCategoryId`.

Details: `references/access-model.md` (Files on a record), `references/data-endpoints.md`.

## Automation

What FROGGATE does on its own when a record changes — an e-mail to the customer, a status change, a
webhook to another system, a document, a chat message. These are **actions** the user adds in the
section's settings; MCP shows how an existing action is configured (`get_action_settings`) but
cannot create one.

**Put reactions to a change into actions, not into the app.** An action runs whoever changed the
record — the app, a colleague in FROGGATE, an import — while code in the app runs only when the
change came from the app. "E-mail the customer when the order ships" is an action on the *Shipped*
status.

- **Automatic e-mail** (*Automatický e-mail*) — sent when a record enters a status, or on the day in
  a date column.
- **Scenario** (*Scénář*) — runs several actions in a row, by itself when a record matches its
  filters (statuses, column values, the kind of change) or at an hour, and can set a status or
  column values at the end.
- **Send webhook** (*Odeslat webhook*) — posts the record to another system's address.
- **Chat message** (*Zpráva do chatu*), **Dynamic PDF** (*Dynamické PDF*) and the rest — the full
  list: `https://client.froggate.cloud/documentation/konfigurace/akce-typy-prehled.md`.

From the app, `run-action` runs an action granted to the key on chosen records (e.g. "download the
PDF") — only the action types FROGGATE enables for the API.

Details: `references/data-endpoints.md` (run-action).

## Connecting another system

An e-shop, an ERP, accounting or a booking system exchanging records with FROGGATE.

Variants:

- **Without code, when the other system speaks webhooks**: **Receive webhook** (*Přijmout webhook*)
  gives an address the other system posts JSON to, and FROGGATE creates or updates records from it;
  **Send webhook** sends changes the other way. Check this first — it may make a sync program
  unnecessary.
- **A backend integration** (a server, a scheduled job) with a key of an **API user** — a service
  account that keeps working whoever leaves the company and shows as the author in the change
  history; its key must be linked to an application (Settings → API, the *Application* column).
  Upsert by a unique column (`updateOnDuplicateColumnId`) keeps a repeated sync from creating
  duplicates. To learn about changes made in FROGGATE, point a *Send webhook* action at your server,
  or read with `get-table-rows`.

The app must handle: the backend key is a secret — it stays on the server.

Details: `references/access-model.md` (Backend integration), `references/client-code.md`.

## Chat

Messages in the company chat — a notifier ("order shipped" into the warehouse channel), a bot
answering in threads, error reports from a published site.

Set up (user): a key of an API user with **Read chat** / **Write to chat**, and the API user added
to each channel. For a site's error reports, a report key in **Settings → Aplikace → Hlášení** — no
API key needed.

Variants: from code through the chat endpoints; from FROGGATE without code through the *Chat
message* action; from an AI client through the MCP chat tools.

Details: `references/chat-endpoints.md`, `references/publishing.md` (reporting errors).
