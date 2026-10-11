<!-- Generated — the source is https://client.froggate.cloud/apidoc/text-column-html-format.md. -->
# Long-text (`text`) column HTML format

Every column reported as `type: text` (`valueFormat: string`) stores its value as
**HTML**, not plain text. When you write to such a column via `insert-table-row` /
`update-table-row` (REST API) or `insert_row` / `update_row` (MCP), the string you
send is interpreted as HTML and sanitized server-side. Plain text is accepted —
it is auto-wrapped in a paragraph on save — but if you want any kind of formatting
(headings, lists, bold, links, …) you must produce well-formed HTML using the tags
and attributes documented below. Anything outside this allow-list is stripped on
save.

When you read a long-text column back, the value you receive is the saved HTML —
prepare for it on the display side.

## Recommended baseline

Use a paragraph for ordinary text:

```html
<p>Some text.</p>
```

Empty paragraph (renders with the expected line height):

```html
<p><br></p>
```

Do **not** emit `<p></p>` for blank lines — an empty paragraph collapses to zero
height in the renderer.

## Allowed tags

**Block-level:** `p`, `div`, `h1`–`h6`, `blockquote`, `pre`, `code`, `hr`, `figure`,
`img`, `ul`, `ol`, `li`, `table`, `tbody`, `colgroup`, `col`, `tr`, `td`

**Inline:** `span`, `br`, `a`, `strong`, `b`, `em`, `i`, `u`, `s`, `sub`, `sup`

## Allowed attributes

`style`, `dir`, `class`, `href`, `target`, `rel`, `src`, `alt`, `width`, `height`,
`colspan`, `rowspan`, `start`, `type`, `disabled`, `checked`, `role`,
`aria-checked`

Other attributes are removed during sanitization.

## Checklists

Checklists are an unordered list where each `<li>` carries a `role="checkbox"`
attribute plus an `aria-checked` state, and contains a disabled native checkbox
input followed by the item text:

```html
<ul>
  <li role="checkbox" aria-checked="true">
    <input type="checkbox" disabled checked>
    Item text
  </li>
  <li role="checkbox" aria-checked="false">
    <input type="checkbox" disabled>
    Another item
  </li>
</ul>
```

Keep `role` / `aria-checked` on the `<li>` and `disabled` on the `<input>` in
sync — both are required for the renderer to display the item as a checklist
row and not as a plain bullet.

## Code blocks

A code block is a `<pre>` that wraps a `<code>`. Without the inner `<code>`, the
value is not saved as a code block.

```html
<pre><code>echo "hello"</code></pre>
```

Language hint (used for syntax highlighting on display) goes on the `<code>` via
either `class="language-XX"` or `data-language="XX"`:

```html
<pre><code class="language-javascript">console.log('hello');</code></pre>
```

Background colour, if you need one, goes on the outer `<pre>` via inline
`style="background-color: …"`. Other styles on `<pre>` / `<code>` are stripped.

## Images

Preferred form is a `<figure>` wrapping an `<img>`:

```html
<figure><img src="https://…" alt="" width="600" height="400"></figure>
```

Alignment is stored as `style="text-align: left|center|right"` on the outer
`<figure>`. A bare `<img>` is accepted but cannot carry alignment that survives
re-editing in the UI.

`width` and `height` are pixel integers (e.g. `600`), not CSS values (no `px`
suffix, no `%`).

## Things that are silently dropped

- Tags not in the allow-list (e.g. `<script>`, `<iframe>`, `<style>`, `<form>`,
  `<input>` outside the checklist pattern).
- Attributes not in the allow-list — including event handlers (`onclick`, …) and
  `javascript:` URLs in `href` / `src`.
- Inline `style` on `<pre>` / `<code>` other than `background-color`.
- Width / height expressed as CSS values rather than numbers.

If your output is missing after a save, the most likely cause is that it relied
on a tag or attribute outside this list.
