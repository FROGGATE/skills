# Server-rendered content in a published app (`data-fg-*`)

A web app published through FROGGATE (**Settings → Aplikace**) can have parts of its
HTML filled in **on the server, at publish time**, from data the customer keeps in
FROGGATE sections. You mark an element with a `data-fg-*` attribute, and when the
content is updated FROGGATE writes the current value straight into the served HTML.

Contents: When to use this · Setting it up in FROGGATE (incl. the site's domain) · Addressing
a value · Attributes (fallback, SEO tags, static text) · Files (image size, non-image files) ·
Repeating markup per record · Generated pages (link columns, multilingual sites, sitemap) ·
How and when the content is updated · Not supported · Constraints worth knowing

## When to use this

**The trigger is simple: the moment you are about to call the API to get a text or
an image URL that you will then put on the page, use `data-fg-*` instead.**

Typical cases, all of which belong here rather than in an API call:

- A headline, perex, description or price the customer edits in a section.
- Contact details, opening hours, an address in the footer.
- A product photo, a hero image, a gallery of photos of one record.
- The page `<title>` and `<meta name="description">`.

Two reasons this matters, both of which are lost the moment you fetch the same
value with JavaScript:

- **Search engines see the content.** Text fetched after load is frequently not
  indexed; text present in the HTML always is. For a presentation website that is
  usually the entire point of building it.
- **No API request on page load.** The page ships complete — no request budget to
  think about, no flash of empty layout, nothing to break if the API is slow.

**Keep the API for what genuinely happens at runtime:** search, filtering, forms and
anything that writes, per-visitor or logged-in content, and any listing you cannot
express with the attributes below (see *Not supported*). The dividing line is not
"how important is this data" but **"does it differ per visitor or change while the
page is open"** — if not, it belongs in the HTML.

**Only `.html` and `.htm` files are processed** — anywhere in the bundle, including
subdirectories. This decides the architecture, so settle it before you build: the
attributes must sit in markup you actually ship as HTML. Put them in a JavaScript
component, a template string, or anything the browser assembles at runtime and they
are never filled in. A site whose pages are rendered client-side therefore cannot use
this at all; ship server-ready HTML instead (a static generator, or plain pages).

## Setting it up in FROGGATE

Only the first two steps are prerequisites for the markup to do anything; walk the
customer through them, they cannot be done from code.

1. **Publish the app** in **Settings → Aplikace** (create the app, upload the ZIP,
   activate the version — see the ZIP layout section of the FROGGATE API skill).
2. **Link an API key with read access.** On the app's **API klíč** tab the customer
   picks an active key; in **Settings → API → Oprávnění** that key needs *read*
   access to every section the site pulls from. This is the hard boundary — a
   reference to a section, column or status the key may not read is never filled in,
   no matter how the markup looks.
3. **Get the IDs** from the app's **Obsah ze sekcí** tab: it lists every section,
   column and status the linked key can read, with their IDs. Ask the customer to
   read them off that overview — do not guess IDs, and do not assume the numbering.
4. **For photos, have the customer publish the files** — see *Files* below.
5. **Publish the content.** On the same tab, **Aktualizovat obsah** rebuilds the site
   from current data right away, and the **Automatická aktualizace** switch keeps it
   in sync from then on. Recommend switching it on for any site whose content the
   customer will keep editing; without it every edit needs that button.

   The people who edit the content do not need this tab at all: once the site pulls
   from a section, that section shows a bar above its table naming the site, saying
   whether edits publish automatically and when it was last published, plus buttons
   **Otevřít web**, **Zveřejnit na webu** and — when automatic updating is off —
   **Náhled úprav**. Tell the customer to use that bar; the settings tab is only for
   setting the site up, and it keeps the technical wording (**Aktualizovat obsah**).

### The site's domain

A published app is served at `https://<address>-<companyId>.client-app.logea.com/`.
It can also run on the customer's **own domains**: the customer creates a `CNAME`
record for each pointing at `client-apps.logea.com` and asks FROGGATE support to enable
them. One domain is the **primary** one — the one the site declares as `canonical`; the
system address permanently redirects to it, and it is what FROGGATE puts into the
`sitemap.xml` it generates for generated pages (at the next content update). The other
domains serve the same files directly. **Build the site for the final domain from the
start** — set the generator's site URL, `canonical`, `hreflang` and every absolute link
to the primary domain, never to the system address; FROGGATE does not rewrite them.

Two per-domain options cover a multilingual site with one bundle:

- **Default folder** — the domain's `/` is redirected to `/<folder>/` (e.g. the `.sk`
  domain lands on `/sk/`); everything else (assets, the other languages) stays shared.
  Use this when one build serves all languages under language folders.
- **Root folder** — the domain serves only that folder of the bundle as if it were the
  whole site. Use this when a domain needs its own complete site (own assets, own
  `canonical` pointing at that domain): ship it as a self-contained folder inside the
  same ZIP (e.g. `_domains/sk/`), so it stays one app in FROGGATE. Root-relative links
  and assets inside such a folder must resolve within it.

Tell the customer which option each domain needs when they ask support to enable it.

After a rebuild, the same tab shows what was filled in and what could not be
resolved. **When something does not appear on the site, that report is the first
place to look.** Each row is one thing to fix. A data problem (missing value,
unpublished file, missing permission) names the section and the record and links
straight into it — including records rendered inside a listing or on their own
generated page, where the HTML itself only says `*` or `page`. A problem in the
site's HTML (wrong reference, an attribute on an element that cannot take it, an
unclosed tag) names the file and the attribute instead, because that is the only
place it can be fixed. A reference to a record number the section does not contain
counts as the second kind: the row says which section and number it is, with no link,
next to the file and attribute. That number is either a typo in your markup or a record
the customer has since deleted — check the markup first, then ask. Ask the customer to read the report to you rather than
guessing from the rendered page.

## Addressing a value

References are numeric IDs joined by colons:

```
section:record:column      e.g. 123:45:67
section:record             (for the file attributes)
```

Two tokens replace the `section:record` part where the record is implied by context:
inside a repeated list item, `*` is the current record (`data-fg-text="*:67"`,
`data-fg-img-file="*"`); on a generated page, `page` is the record the page was
generated for (`data-fg-text="page:67"`). Anywhere else the tokens are an error and
are reported.

## Attributes

| Attribute | Effect |
|---|---|
| `data-fg-text="123:45:67"` | Replaces the element's content with the cell value as plain text. |
| `data-fg-html="123:45:67"` | Replaces the element's content with the cell value as HTML (for long-text columns, which store HTML). Sanitized on output. The markup is what the FROGGATE text editor produces — paragraphs, headings, lists, tables, links, an image inserted into the text wrapped in `<figure>` with `<img>` inside, and file cards as `<figure>` with a link; the site's own CSS styles it (browsers give `<figure>` a default margin). |
| `data-fg-attr-<name>="123:45:67"` | Writes the cell value into the attribute `<name>` — e.g. `data-fg-attr-alt`, `data-fg-attr-content`, `data-fg-attr-href`. |
| `data-fg-text-prefix="…"` / `data-fg-text-suffix="…"` | Static text written before / after the value of `data-fg-text` on the same element (see *Static text around a value*). |
| `data-fg-attr-<name>-prefix="…"` / `-suffix="…"` | The same around the value of `data-fg-attr-<name>`. |
| `data-fg-img="123:45:67"` | Sets the element's `src` to the URL stored in the cell. |
| `data-fg-img-file="123:45"` | Sets `src` to the newest published **image** attached to that record (see *Files* below). |
| `data-fg-file-url="123:45"` | Puts the address of the newest published file — of any kind — into the element's own link attribute: `href` on `<a>`, `data` on `<object>`, `src` everywhere else. |
| `data-fg-list-files="123:45"` | On a container: repeats the child marked `data-fg-item` once per published file of that record. Inside the repeated child, `data-fg-img-file="*"` and `data-fg-file-url="*"` refer to the current file. |
| `data-fg-file-category="12"` | Narrows `data-fg-img-file` / `data-fg-list-files` to one file type of the section. |
| `data-fg-file-url-category="12"` | Narrows `data-fg-file-url` to one file type of the section. |
| `data-fg-list="123"` | On a container: repeats the child marked `data-fg-item` once per record of the section (see *Repeating markup per record*). |
| `data-fg-filter="67:hodnota"` | On the `data-fg-list` element or on a page template: only records whose column `67` holds that value. For a selection column write the **option ID**, for a link column the record number — `45`, or `*` / `page` for the record in context. Several conditions separated by `;` must all hold. |
| `data-fg-page="123"` | On the `<html>` element: the whole file is a template — one page is generated per record of the section (see *Generated pages*). |
| `data-fg-page-dir="produkty"` | Required with `data-fg-page`: the folder the generated pages are written into. |
| `data-fg-page-slug="61"` | Optional with `data-fg-page`: column whose value becomes the readable part of the page address. |
| `data-fg-filter="67:hodnota"` | Optional with `data-fg-page`: only the matching records get a page. |
| `data-fg-href-page="123:45"` | Writes the address of the record's generated page into the element's link attribute (`href` on `<a>`/`<area>`/`<link>`). `*` inside a list, `page` on a generated page. In place of the record you may name a **link column** — `page:64`, `*:64`, `123:45:64` — and the link then leads to the page of the record that column points at (see *Linking through a link column*). |
| `data-fg-href-lang="en"` | With `data-fg-href-page`: link to that **language version** of the page. Without it the link stays in the language of the page it sits on. |

All `data-fg-*` attributes are removed from the published HTML.

### The element's own content is the fallback

Whatever you put inside the marked element stays there if the reference cannot be
resolved — the record was deleted, the key lost access, the ID is wrong. So write
the markup with real placeholder content:

```html
<h1 data-fg-text="123:45:67">Fallback headline</h1>
<img data-fg-img="123:45:80" src="images/placeholder.jpg" alt="">
```

This is also what makes the site work in local development, where nothing fills the
attributes in: you see your placeholders instead of an empty page. **Never ship an
empty marked element** — a failed reference then renders as a blank gap.

An empty cell is *not* a failure: the customer cleared the field, so the element is
filled with nothing. Only unresolvable references fall back.

### SEO tags

The attributes work anywhere in the document, including the head:

```html
<title data-fg-text="123:45:61">Fallback title</title>
<meta name="description" data-fg-attr-content="123:45:62" content="Fallback description">
```

### Static text around a value

`data-fg-text` replaces the **whole** content of the element it sits on, so a heading that
mixes a value with fixed words is written by marking only the dynamic part:

```html
<h1><span data-fg-text="page:61">Barbecue</span> — Fulgor Milano</h1>
```

**That is the normal way to compose, and it needs nothing extra.** The two attributes below
exist only for the places where you cannot wrap: inside `<title>`, where a nested element is
rendered as literal text instead of markup, and inside an attribute value, which has nothing
to nest in.

```html
<title data-fg-text="page:61" data-fg-text-suffix=" — Fulgor Milano">Barbecue — Fulgor Milano</title>

<meta name="description" data-fg-attr-content="page:62"
      data-fg-attr-content-prefix="Discover " data-fg-attr-content-suffix=" at Fulgor Milano"
      content="Discover the barbecue at Fulgor Milano">
```

- **Write the fallback as the finished whole**, exactly as in those examples. An unresolved
  reference leaves the element's own content alone and the static text is never emitted by
  itself, so a title cannot degrade into a lone `— Fulgor Milano`.
- **The static text is yours, so it belongs in the language of the file it sits in.** A
  multilingual site already has one template per language (see *Multilingual sites*), which
  is where the translated wording goes — FROGGATE never translates it for you.
- **An empty cell still gets the static text.** The customer cleared the field, which is a
  value and not a failure. If a value can legitimately be empty, do not surround it with
  words that read as broken without it.
- The attribute does something only next to its own directive on the same element.
  `data-fg-text-suffix` with no `data-fg-text`, or either of them on `data-fg-html`, is
  reported as a bad reference and nothing is written.

## Files: only permanent public links are used

`data-fg-img-file` and `data-fg-list-files` use only the record's files that have a
**public link with unlimited validity**. A file with a time-limited link, or with no
link at all, is skipped as if it were not attached. On top of that, `data-fg-img-file`
takes the newest **image** — a PDF or a video is never put into an `img` element, even
when it is the newest file with a public link.

There are two ways the customer gets those links. Per file: open the record, in its
**Soubory** panel use **Vytvořit veřejný odkaz** and set **Platnost odkazu** to
**Neomezeně**. Or for the whole section at once: **Nastavení sekce → Soubory →
Automatický veřejný odkaz**, which publishes every file uploaded to a record from then
on, optionally only files of one chosen type. A link with any other validity is
deliberately ignored so the site cannot break when it expires.

If images do not appear, this is the first thing to check with them.

A gallery or carousel is built from the repeated markup; the rotation itself is your
code (any slider library works — it just finds the images already in the DOM):

```html
<div class="carousel" data-fg-list-files="123:45" data-fg-file-category="12">
  <img data-fg-item data-fg-img-file="*" src="images/placeholder.jpg" alt="">
</div>
```

The repeated element does not have to be the image itself — mark any wrapper with
`data-fg-item` and put the `data-fg-img-file="*"` image inside it.

An item that can only be filled with an image is not repeated for a file that is not one,
so a PDF attached to the record never adds a placeholder tile to the gallery.

### Image size is never reduced

The file goes out exactly as the customer uploaded it — there is no resizing, no format
conversion and no `srcset` generation. A photo straight from a phone therefore ships at
several MB and every visitor downloads all of it.

**The size to ask the customer for is 300 kB or less per photo.** That is the number to say
out loud; FROGGATE only *reports* from **1 MB** up, so that it stays quiet about photos
somebody already resized sensibly. A file between the two is still worth shrinking.

FROGGATE flags large images instead of fixing them, in five places:

- **on upload** in the record's file panel — but only when the image will really go out:
  the section has automatic public links on (and the image lands in a published type) and
  the site actually fills in files of that section and type;
- **on the file itself** — a permanent **Velký pro web** chip next to its size in the file
  panel, under the same conditions, so an image uploaded long ago is flagged too;
- **when the file's type changes** into one that publishes it, in the confirmation bubble;
- **on the bar above the section** — a chip with the count and total size; clicking it
  opens the list: the record each image belongs to, the file, its size and an **Otevřít
  záznam** button. Only records the user is allowed to see are counted and listed, and at
  most the 30 largest. The chip counts what the live site serves, so an image uploaded
  after the last content update shows up only once the content is published again;
- **in a Velké obrázky na webu card** in the app's **Obsah ze sekcí** tab, listing the
  file, its size and a link to the record.

Images pasted into a long-text cell count too — `data-fg-html` writes their `<img src>` into
the page, so visitors download them from FROGGATE exactly like a record attachment. Nothing
is ever blocked; the fix is to upload a smaller file to the record.

Tell the customer the 300 kB figure up front when the site shows photos they maintain — it
is the cheapest page-speed win on such a site, and they cannot discover it from the site
itself.

### Files that are not images

`data-fg-file-url` is the same idea for everything else — a document to download, a PDF
shown in the page, a video. It does not filter by kind, so pick what lands there with
`data-fg-file-url-category` rather than hoping the right file happens to be newest:

```html
<a data-fg-file-url="123:45" data-fg-file-url-category="12" href="files/fallback.pdf">Ceník</a>
<iframe data-fg-file-url="123:45" data-fg-file-url-category="12" src="files/fallback.pdf"></iframe>
```

The two category attributes are separate on purpose: an image directive and a file
directive can be narrowed to different types. Keep them on **different** elements unless
the file directive's target is `href`/`data` — on anything else both write `src`, and the
file address wins. With no published file the attribute keeps whatever your bundle has in
it, exactly like every other directive.

No directive that carries an address is filled on a `<script>` element — the file would be
executed as code on the site. Such a placement is reported as an error and the element is
left as your bundle has it.

A list of documents to download is the same repeated markup as a gallery, with the link
as the repeated item:

```html
<ul data-fg-list-files="123:45" data-fg-file-category="12">
  <li data-fg-item><a data-fg-file-url="*" href="files/fallback.pdf">Dokument</a></li>
</ul>
```

## Repeating markup per record

A product listing, a category menu, a run of text blocks — anything that repeats once
per record of a section — is built exactly like a file gallery: a container with
`data-fg-list`, one child marked `data-fg-item` as the item template, repeated per
record the linked key can read, in the order the section shows its rows:

```html
<ul class="products" data-fg-list="123">
  <li data-fg-item>
    <a data-fg-href-page="*" href="#">
      <img data-fg-img-file="*" src="images/placeholder.jpg" alt="">
      <span data-fg-text="*:61">Product name</span>
    </a>
  </li>
</ul>
```

Inside the item, `*` addresses the current record: `data-fg-text="*:67"` /
`data-fg-html` / `data-fg-attr-*` for its cells, `data-fg-img-file="*"` for its newest
published image, `data-fg-list-files="*"` for a gallery of its files,
`data-fg-href-page="*"` for the link to its generated page.

Narrow the records with `data-fg-filter="67:45"` — only records whose column `67` holds
that value. With a link column pointing at another section the value is the record
number, which is how a category page lists only its own products; with any other column
it is the value itself (`data-fg-filter="67:Praha"`).

**Selection columns take the option ID, not its text** (`data-fg-filter="67:891"` — the
IDs are in the section overview on the *Content from sections* tab). The text is a name
the customer can rename at any moment, and the filter would silently stop matching.

**Several conditions are separated by `;` and all of them must hold.** That is the normal
shape of a listing: one condition picks the section it belongs to, another keeps
unpublished records out —

```html
<ul data-fg-list="123" data-fg-filter="67:page;89:891">
```

means "products of this page's category **and** whose *Published* option is `891`". Each
condition is an equality on one column, and a column may appear only once; anything
beyond that (search, ranges, either/or) is runtime data and belongs in an API call.

A filter that cannot be used at all (unknown column, unknown option ID, the same column
twice) leaves the list on your fallback content and lands in the report — it never
silently falls back to "all records".

Lists nest: an item can contain another list, and the inner list's filter can point at
the outer record with `data-fg-filter="67:*"`.

Worth knowing:

- **An empty result is a valid page state** — the list renders with no items, so style
  the surrounding markup to survive being empty. If the *section* cannot be read at
  all (missing permission, wrong ID), the single template item stays with your
  fallback content and the report says why.
- A listing is cut at 10 000 records (reported as a warning). That is a safety stop,
  not a layout tool — a page that big needs a different design.

## Generated pages — one page per record

For a detail page per record (a product page), mark a whole HTML file as a template by
putting these attributes on its `<html>` element:

```html
<html data-fg-page="123" data-fg-page-dir="produkty" data-fg-page-slug="61">
```

- One page is generated per record of section `123`, written into the
  `data-fg-page-dir` folder and served at `/produkty/<slug>-<id>/` — `<slug>` is the
  value of the `data-fg-page-slug` column (lowercased, diacritics stripped), `<id>` is
  the record number. Without `data-fg-page-slug` the address is plain
  `/produkty/<id>/`.
- Inside the template, address the record with `page`: `data-fg-text="page:67"`,
  `data-fg-img-file="page"`, `data-fg-list-files="page"` — and a list on the page can
  be filtered by the page's record with `data-fg-filter="<column>:page"`. That is how
  a category page template lists the products of *that* category.
- **`data-fg-filter` works on the template too**, and then decides which records get a
  page at all: `<html data-fg-page="123" data-fg-page-dir="produkty" data-fg-filter="67:891">`
  publishes only the records where column `67` holds that value. Records left out get
  no page, and `data-fg-href-page` pointing at one keeps its fallback instead of
  linking to an address that does not exist. Use it whenever only part of a section
  belongs on the public site (a "publish" selection column the customer sets).
- **The template file itself is never published** — never link to it directly; every
  link to a record's page goes through `data-fg-href-page`, which always matches the
  address the page was actually written to.
- One template per section **and language** (see *Multilingual sites*); a second one for
  the same pair is ignored and reported. `data-fg-page-dir` is required, and an existing
  file of your bundle is never overwritten — a collision skips the page and lands in the
  report.
- **Choose the slug column deliberately.** The record id alone keeps the address
  stable forever. A slug column adds readability, but when the customer edits that
  value the page MOVES — the old address dies, and there are no redirects. For stable
  readable addresses have the customer fill a column they never change (a product
  code); the id-only form is the safe default.

### Linking through a link column

A product page needs a way back to its category — the breadcrumb. Instead of a record,
give `data-fg-href-page` the cell of the **link column** that holds the category:

```html
<a data-fg-href-page="page:64" data-fg-text="page:64">All products</a>
```

`page:64` reads as *the record that column `64` of this page's record points at*. The
same works with `*:64` inside a list and `123:45:64` for a fixed record. Nothing else is
needed for the label: `data-fg-text` on a link column already renders the name of the
record it points at.

- **You name only the column** — which section it leads to is taken from that column's
  configuration in FROGGATE, never from your HTML, so the same attribute cannot be aimed
  at another section. It also means the link follows that configuration: point the column
  at a different section and the links move with it.
- The target section needs its own page template (`data-fg-page`), like any other
  `data-fg-href-page` target.
- **An empty cell keeps the fallback — but only in the link.** A product with no category
  leaves whatever is in the element (typically the link to the full listing), and the
  report names the record that is missing it, because that is where it gets fixed. A label
  written with `data-fg-text` on the same cell behaves like any other value instead: an
  empty cell renders empty. So either put the fallback wording in an element the empty
  cell cannot blank, or accept that such a product shows an unlabelled breadcrumb.
- **One hop only.** The record you arrive at cannot be followed further through another
  link column.

### Multilingual sites

Put each language in its own path — `/cs/produkty/…`, `/en/products/…` — the way search
engines expect. One record then has one page per language, and each language version
takes its own columns and its own address:

```html
<html lang="cs" data-fg-page="123" data-fg-page-dir="cs/produkty" data-fg-page-slug="61">
<html lang="en" data-fg-page="123" data-fg-page-dir="en/products"  data-fg-page-slug="62">
```

- **The language comes from `<html lang>`** — the attribute a multilingual site has
  anyway. FROGGATE never invents the path: the folder is always yours, in
  `data-fg-page-dir`. Which column holds which language stays entirely inside the
  template; the system only tells the templates apart.
- **A link stays in the language of the page it sits on.** `data-fg-href-page="123:45"`
  on a `lang="en"` page leads to the English page of record 45 — nothing extra to write,
  so copying a template to another language cannot silently leave links pointing at the
  original language.
- **`data-fg-href-lang` links across languages** — the language switcher and `hreflang`:

  ```html
  <a data-fg-href-page="page" data-fg-href-lang="en">English</a>
  <link rel="alternate" hreflang="en" data-fg-href-page="page" data-fg-href-lang="en">
  ```

  An explicitly requested language is never substituted by another one: if that language
  version does not exist, the link keeps its fallback and the report says so. The same
  applies when the page's own language has no template for the target section.
- **The lang values must match** — `lang="cs"` and `lang="cs-CZ"` are two different
  variants. A section with a **single** template is used from pages in any language, so
  a site that does not deal with languages behaves exactly as before.
- **Give each language its own `data-fg-filter`** when only part of the section is
  translated — otherwise a record with no English text gets an English page with empty
  headings, which is worse for search engines than not existing.

### Sitemap

When the site generates pages, a `sitemap.xml` is generated next to them — listing
every HTML page of the published site including all generated ones. Search engines
have no other way to learn the generated addresses, and after a slug change it is what
gets them to the new ones. A `sitemap.xml` you ship in the bundle always wins — the
generated one never overwrites it.

## How and when the content is updated

Nothing you write in the markup schedules an update — the customer controls it on the
**Obsah ze sekcí** tab (step 5 above): the **Aktualizovat obsah** button rebuilds the
site immediately, the **Automatická aktualizace** switch keeps it in sync from then
on, and activating a version of the app rebuilds it too — the new version goes live
only once its content is filled in, and if it cannot be filled in (no usable API key, another
update running, the build failing) the activation is refused and the previous version keeps serving. The bar above every section
the site reads from carries the same action as **Zveřejnit na webu**, so whoever edits
the record can publish it without opening the settings.

A few consequences worth telling the customer about:

- **An edit is not live instantly.** With the switch on it takes a few minutes; with
  it off, until someone presses the button. Design nothing around a value being
  fresh to the second — if it must be, it is runtime data and belongs in an API call.
- **A bad content update is recoverable.** Every rebuild is kept for a while and can
  be restored from that tab, so the site can be rolled back to earlier content
  without re-uploading the app. While an older build is restored, automatic updating
  pauses so it will not be overwritten.
- **Anything can be checked before it goes live.** The **Náhled** button — on a
  version in the Verze tab, or on a rebuild in the list on the Obsah ze sekcí tab —
  opens that content on a separate temporary address without touching the running
  site. On a version the current section content is filled in first, so a freshly
  uploaded app can be judged before it is published; clicking again re-reads the
  content and keeps the same address. Note this is *current* content, not what the
  live site serves — if the customer has restored an older rebuild, the version
  preview shows newer content than the site does. Each preview address expires after a day;
  several can be open at once, so two of them can be compared side by side.

## Not supported

Do not design around these — they do not exist:

- **Anything beyond equality conditions on a list** — no sorting override (the order is
  the section's order), no ranges, no full-text search, no "either/or". Several
  conditions can be combined, but only as "all of them must hold". Interactive narrowing
  is runtime behaviour: fill in the full listing and filter it client-side, or call the API.
- **Pagination of a filled-in listing** — a list renders whole. If that is too much,
  restructure the data (e.g. one list per category page) or fetch at runtime.
- **Per-visitor content** — everything filled in at publish time is identical for every visitor.

## Constraints worth knowing

- The marked element must have a normal closing tag; the content of self-closing
  elements cannot be replaced (use `data-fg-attr-*` or `data-fg-img` on those).
- `data-fg-img` accepts a URL from the cell — an absolute `https://…`, a
  protocol-relative `//…`, or a path relative to the site. Anything else is refused.
- Everything the attributes do not touch is served byte-for-byte as you uploaded it,
  so hashed bundle filenames and inline scripts are safe.
