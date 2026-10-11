# Data endpoints

## POST /dynamic-tables/get-active-tables

Returns all tables visible to the current key, with their column structure and statuses. All body fields are optional — send `{}` to fetch every accessible table.

```json
{
  "tableId": 25,
  "nameFilter": "zamě",
  "verbose": false
}
```

- `tableId` — return only this table.
- `nameFilter` — case-insensitive substring filter on the section name (the table's menu item name, returned as `mainMenuName`); a match on the name of a menu group returns every section in that group. `%` and `_` are matched literally. Useful when the company has many tables.
- `verbose` — default `true`. When `false`, the response omits `statuses`, `columns` and `actions` and returns only the table identifiers and menu names. Much cheaper when you only need to locate a section by name; once you have the right `tableId`, call again with the default `verbose` to get the full structure.

```json
{
  "tables": [
    {
      "tableId": 10,
      "menuPath": "Menu 1 - Menu item",
      "mainMenuName": "Menu item",
      "submenuName": "Menu 1",
      "statuses": [{ "id": 1, "name": "New", "tabView": "table" }],
      "columns": [
        { "id": 110, "name": "Name", "type": "text", "valueFormat": "string", "editable": true, "possibleValues": null },
        { "id": 111, "name": "Category", "type": "selection", "valueFormat": "integer", "editable": true, "possibleValues": { "1": "Option A" } },
        { "id": 112, "name": "Price", "type": "float", "valueFormat": "number", "editable": true, "possibleValues": null },
        { "id": 113, "name": "Due", "type": "date", "valueFormat": "date", "editable": true, "possibleValues": null },
        { "id": 114, "name": "Department", "type": "from_other_table", "valueFormat": "integer", "editable": true, "possibleValues": null, "linkedTableId": 22, "linkedColumnId": 305 }
      ]
    }
  ]
}
```

Each status is one tab of the section, and `tabView` says how that tab displays its data: `table` = a plain table, `kanban` = a board whose columns are statuses. Treat `tabView` as an open vocabulary — further display types may be added. A status carrying `kanbanParentStatusId` instead has **no tab of its own**: it is a column of the board given by that ID, and the label the user sees on screen is its `kanbanColumnName`, not `name` (the board's own status is also the board's first column and carries `kanbanColumnName` too). So when matching a user's wording like "move it to Done" against a section that has a board, match against `kanbanColumnName` — and remember that moving a card between board columns simply changes the record's status.

Each column carries both `type` (high-level kind) and `valueFormat` (the JSON shape to send in `insert-table-row` / `update-table-row`):

| `type` (column kind) | `valueFormat` (how to send the value) | Example value in request |
|---|---|---|
| `varchar` | `string` (short plain text, at most 200 characters — a longer value is cut off) | `"Prague"` |
| `text` | `string` (interpreted as **HTML** — see below) | `"<p>Some text</p>"` |
| `float` | `number` (JSON number, **no quotes**) | `42.5` |
| `date` | `date` (JSON string `YYYY-MM-DD`) | `"2024-01-15"` |
| `datetime` | `datetime` (JSON string `YYYY-MM-DD HH:MM:SS`) | `"2024-01-15 09:30:00"` |
| `selection` | `integer` (option id from `possibleValues`) | `1` |
| `user` | `integer` (FROGGATE user id, a JSON integer — a string of digits is refused) | `65` |
| `from_other_table` | `integer` (row id from the linked section) | `88` |
| `int` | `integer` (the record's own ID number) | `7` |
| `image` | `object` — read-only, see below | — |

System columns use the same vocabulary: "who added" and cumulative columns report as `text` / `string`, "time added" and "time the status was set" as `datetime`. Whether any column is writable is decided by its `editable` flag, not by its type.

For dropdowns (`type: selection`), the integer is the option id from the `possibleValues` map, not the option label.

A link column (`type: from_other_table`) is **not** a dropdown: it has no `possibleValues` and no options to add. It carries two extra fields instead — `linkedTableId` (the section it points at) and `linkedColumnId` (which column of that section is shown) — and the value you send is a **row id from `linkedTableId`**, which you get with `get-table-rows` on that section. Send it as a whole number (`88`) or as a string of digits only (`"88"`, surrounding spaces are ignored); anything else — a decimal (`88.5`, `88.0`), other text, a list or `true`/`false` — is refused with an error quoting the value. `null` clears the link. Either field is `null` while the link is not fully configured.

> **Changed:** link columns used to be reported as `type: "selection"`. If you branch on `type`, add the `from_other_table` case; `valueFormat` was and stays `integer`, so nothing about sending values changes.

Image columns (`type: image`) are **read-only** — the file is uploaded in the app, and `insert-table-row` / `update-table-row` reject them. Reading a row returns an object describing the stored file, or `null` when the cell is empty:

```json
{ "fileName": "photo.jpg", "size": 48213, "width": 300, "height": 180 }
```

**Long-text (`text`) columns store HTML.** The string you send is interpreted as HTML and sanitized server-side. Plain text is accepted (auto-wrapped in a paragraph on save), but any markup outside the allow-list is stripped. Before writing a non-trivial value, see the full tag/attribute allow-list and required patterns (paragraphs, checklists, code blocks, images) in [`text-column-html-format.md`](text-column-html-format.md) next to this file.

For a scope-restricted key the response is **already filtered** — only allowed tables, allowed columns and allowed statuses appear. Treat what you get back as the full view available to this key.

Each table also carries an `actions` array — the table actions this key may execute via `run-action` (see below), with their IDs, limits and allowed statuses. Empty for most keys.

Each column also carries an `editable: boolean` flag. Columns where `editable` is `false` are read-only over the API — typically auto-filled values (formulas, sums, paired non-editable columns, link-driven date fills) or columns the current key has no write permission on. `insert-table-row` and `update-table-row` reject any `columnsData` entry pointing at such a column.

A non-editable column also carries `notEditableReason`, and the reason decides what you can do about it. `filledByFormula`, `filledByAutomaticSeries`, `composedFromOtherColumns`, `filledBySumFromLinkedSection`, `valueTakenFromLinkedRecord`, `filledByDateDifference` and `filledByDateFromLinkedRows` mean the section **computes** the value, so no key can ever write it — to change it, write the columns it is derived from. `columnTypeIsNeverWritableByApi` (image columns) and `writingThroughApiDisabledForThisColumn` are properties of the column itself. Only `callerHasNoEditPermissionForThisColumn` and `columnIsOutsideTheApiKeyWriteScope` are about **your key**, so the same column may be writable for another key. Never present a computed column to the user as a broken configuration.

A column that takes its value from a linked record and is set to let users edit that value (column setting `pairedToColumnEditable`) keeps a value you write into it — a value, `0` or `null` — and later changes of the linked record no longer overwrite it. Writing back the value the column already holds changes nothing: it stays linked, so reading a row and sending it back whole does not freeze its linked columns. Sent together with a different linked record, the value is kept even when it equals the stored one. A user-type column taken from a linked record is `editable` even without that setting; there a later change of the linked record overwrites what you wrote, and the rest of this paragraph does not apply. The one exception: `null` sent in the same call as the link column (the "record from another table" column it is taken through) means "take the value from the linked record", on insert and update alike, so sending the same upsert twice gives the same result. To put the linked value back on an existing row, send `null` together with the link column; leaving the column out of an update keeps what is stored. A column set to update only in some statuses takes nothing over for a row in another status — such a call leaves its value unchanged.

> **New:** `notEditableReason` is sent alongside `editable: false` (absent when the column is writable).

## POST /dynamic-tables/insert-table-row

```json
{
  "tableId": 25,
  "statusId": 1,
  "columnsData": [
    { "columnId": 110, "value": "Some text" },
    { "columnId": 112, "value": 42.5 },
    { "columnId": 113, "value": "2024-01-15" },
    { "columnId": 111, "value": 1 }
  ],
  "updateOnDuplicateColumnId": 110,
  "files": [
    { "fileName": "document.pdf", "base64file": "JVBERi0x...", "note": "Optional note", "fileCategoryId": 4 }
  ]
}
```

- `statusId` is required; use the status ID from `get-active-tables`.
- `value`'s JSON type depends on the column's `valueFormat` (see the table in `get-active-tables` above): string for text/date/datetime, JSON number for float, JSON integer for selection/link/user. Sending a JSON string for a number column is rejected with "Value of column ID … must be float or int".
- `updateOnDuplicateColumnId` makes it an upsert — if a row already has this column value, that row is updated instead of inserting a new one. The column whose ID is referenced here must also be present in `columnsData`. Because an upsert reaches into an existing row, it is refused (403) for public keys and per-user login-flow keys (see "Public website" in `access-model.md`); use a backend or full-access key for upserts.
- `files` is optional; base64-encoded file content. `fileCategoryId` is optional too and files the attachment into one of the section's file categories; omit it to leave the file uncategorised.
- For login-flow keys: the table's user-link column (the `from_other_table` column pointing to the user table) is auto-filled with the logged-in user's row id; any value the client sends for that column is ignored. A shared section is read-only, so inserting into it is refused.

Response `data`:
```json
{
  "rowId": 123,
  "files": [{ "fileName": "document.pdf", "fileId": 10, "fileCategoryId": 4 }],
  "updatedColumns": { "110": { "originalValue": "old", "newValue": "new" } }
}
```

`updatedColumns` stays empty (`{}`) unless an upsert matched a row and changed it.

## POST /dynamic-tables/update-table-row

```json
{
  "tableId": 25,
  "rowId": 123,
  "update": {
    "statusId": 2,
    "columnsData": [{ "columnId": 110, "value": "Updated value" }],
    "insertFiles": [
      { "fileName": "new.pdf", "base64file": "JVBERi0x...", "note": "Optional note", "fileCategoryId": 4 }
    ],
    "updateFiles": [
      { "fileId": 10, "fileCategoryId": 4 }
    ]
  }
}
```

- `rowId` is required.
- All fields inside `update` are optional; send only what changes.
- **Deleting a row:** there is no separate delete endpoint. To delete, call this endpoint with `update.statusId = 0` — that is the system-wide soft-delete convention.
- `updateFiles` refiles files already attached to this row into another file category — send `fileCategoryId: null` to clear it. The file content is never touched, and a file of another row is refused. Use `get-table-rows` for the file ids.
- For login-flow keys: the server checks the row's user-link column matches the logged-in user; otherwise 403. Cross-user updates are impossible. A shared section is read-only, so updating it is refused.

Response `data`:
```json
{
  "files": [{ "fileName": "new.pdf", "fileId": 11, "fileCategoryId": 4 }],
  "updatedFiles": [{ "fileId": 10, "fileCategoryId": 4 }],
  "updatedColumns": { "110": { "originalValue": "old", "newValue": "updated" } }
}
```

`updatedColumns` lists, of the columns you sent (and the status), those whose stored value the call changed — a value sent unchanged is not listed, and a column the section recomputes on its own is never listed. Both values are the stored ones, in the column's own type (string, number or null; dates and date-times in ISO 8601). A date-time is stored to the second; one sent without seconds is stored with `:00`.

## POST /dynamic-tables/get-table-rows

```json
{
  "tableId": 25,
  "parameters": {
    "limit": 100,
    "page": 1,
    "order": "DESC",
    "showFiles": false,
    "returnColumns": [110, 111]
  },
  "filter": {
    "status": 1,
    "id": 26,
    "columnsValues": { "110": "Prague", "111": null },
    "columnsNotNull": [112]
  }
}
```

- `parameters.limit` — records per page (default 100).
- `parameters.page` — page number (default 1).
- `parameters.order` — `"ASC"` or `"DESC"` (default `"DESC"`).
- `parameters.showFiles` — if true, returns base64 file content in `base64file`; if false, `base64file` is null. `publicUrl` is always returned when available, regardless of this setting. Either way the `files` list is only populated when the key has **file read** enabled on that table (see "Files on a record" in `access-model.md`); otherwise it is empty and `showFiles` has no effect.
- `parameters.returnColumns` — optional array of column IDs. If set, each row's `columns` map contains only these columns; if omitted, all columns the key can see are returned. Inaccessible / unknown IDs are ignored.
- `filter` — optional; all conditions combined with AND. Keys in `columnsValues` are column IDs as strings (surrounding spaces are ignored; a key made of anything else than digits is refused); a string/number value matches that exact value, a `null` value matches empty cells (IS NULL). `columnsNotNull` is a list of column IDs that must be non-empty (IS NOT NULL). A filter may use only columns the key can read — a column outside the key's read scope (or one the key's user may not see) is refused with an error naming it, never silently ignored. A column of a section this table links to (through a `from_other_table` column) filters through that link, and only linked records the key can read count: those in the statuses the key allows for that section, never deleted ones. Such a column is usable when it is the link's display value (the `valueDisplay` a read returns for that link) and the key reads the link column, or otherwise when the key may read the linked section — for a signed-in user only a shared section. `filter` must be an object; an empty list (`[]`) counts as no filter.
- For login-flow keys: only the user's own rows are returned regardless of filter; the password column on the user table is stripped. A shared section returns all its rows in the allowed statuses to every signed-in user.
- For all keys with a scope-restricted backing key: only allowed columns appear in each row, and `files` is empty unless file read is enabled on the table.

Response `data`:
```json
{
  "parameters": { "pages": 5, "rows": 437 },
  "rows": [
    {
      "id": 1,
      "status": 1,
      "columns": {
        "110": { "columnId": 110, "value": "Some text", "valueId": null, "valueDisplay": null },
        "111": { "columnId": 111, "value": "Option A", "valueId": 1, "valueDisplay": null }
      },
      "files": [
        {
          "fileName": "document.pdf",
          "fileId": 10,
          "type": "standard",
          "fileCategory": { "id": 4, "name": "Contract" },
          "base64file": null,
          "publicUrl": "https://cdn.froggate.cloud/..."
        }
      ]
    }
  ]
}
```

`parameters.rows` is the total number of rows matching the filter (across all pages), `parameters.pages` the number of pages at the requested `limit`.

`valueId` is set for `selection` columns (the option's ID, `value` holds its name) and `user` columns (the user's ID, `value` holds their name). `valueDisplay` is set for `from_other_table` columns: `value` holds the linked record's ID and `valueDisplay` that record's displayed column — the same value a filter through the link matches. In every other case both are `null`.

**File display tip:** when rendering an image or video on a frontend page, use `publicUrl` directly in `<img src=…>` / `<video src=…>`. Do **not** fetch `base64file` for display — that downloads the entire file through the API and burns bandwidth and quota.

## POST /dynamic-tables/insert-column-selection-value

Adds a new option to a `selection` column. Idempotent on the value name. Rejected on any other column type — a `from_other_table` column has no options; to widen what it accepts, insert a row into its `linkedTableId` section instead.

```json
{ "tableId": 25, "columnId": 111, "value": "New option" }
```

Response `data`:
```json
{ "valueId": 42 }
```

**Backend-only.** Public and login-flow keys get 403.

## POST /dynamic-tables/run-action

Executes a configured table action (e.g. generating PDF labels) on one or more rows — the same operation a user triggers from the table, with the same configuration. The caller only picks the rows; what the action does is defined entirely by its configuration in the section settings.

```json
{ "actionId": 123, "rowIds": [45, 46], "count": 4 }
```

- `actionId` — the action to run. The actions your key may execute are listed per table in the `actions` field of `get-active-tables`. Each entry carries: `actionId`, `name`, `resultType` (`file` / `message`), `fileType` (file extension, `null` for `message` actions), `acceptsCount`, `maxRows` (most row ids in one call) and `allowedStatusIds` (statuses the action runs in; `null` = every status the key sees) — plus, **only when `acceptsCount` is `true`**, `maxCount` (highest allowed `count`) and `maxTotalItems` (cap on rows × count in one call).
- `rowIds` — rows to process. Every row must belong to the action's table, must not be deleted, and must currently be in a status the action is enabled for.
- `count` — optional, only for actions with `acceptsCount: true` (e.g. number of labels per row). Sending it to an action that does not accept it is an error, not ignored.

Response for a `file` action (one file covering all requested rows):

```json
{ "resultType": "file", "fileName": "labels.pdf", "fileType": "pdf", "contentBase64": "JVBERi0…", "processedRowIds": [45, 46] }
```

Response for a `message` action carries `message` (or a per-row `results` list, each `{ "rowId": …, "message": … }`) instead of the file fields.

**Partial failure (per-row actions only):** an action processing rows one by one may fail on some rows and succeed on others. The response then has `status: "error"` (HTTP 200) with `errorCode: "ACTION_PARTIALLY_FAILED"` — or `"ACTION_FAILED"` when every row failed — and, as the one exception to the envelope rule, `data` stays populated: `data.processedRowIds` lists the rows that ran (executed rows are never rolled back), `data.results` their results and `data.errors` the failures as `{ "rowId": …, "error": "…" }` with a human-readable reason per row. After a partial failure, re-send only the rows missing from `processedRowIds` — never the whole batch. A `file` action never fails partially: it either returns the complete file covering all requested rows or fails as a whole (plain `ACTION_FAILED` with `data: null`), so a partial-failure response never carries `contentBase64`.

**Scope refusals** answer HTTP 403 with a machine-readable `errorCode`: `ACTION_NOT_ALLOWED` (the action is not granted to your key), `TABLE_NOT_ALLOWED` (no read access to the action's table, or the table cannot be used with this key type), `ROW_STATUS_NOT_ALLOWED` (a row's current status is outside the key's table-level or per-action status list), `ROW_NOT_ALLOWED` (a row does not belong to the logged-in user).

**Which keys (run-action):** available to backend keys and login-flow keys (a login-flow key only on the logged-in user's own rows, never in a shared section). A **public key gets 403** — actions are never executable with an anonymous credential. A non-full-access key needs read access to the action's table and the action explicitly granted in **Settings → API → key detail → the allowed table's detail → Allowed actions**; a full-access key may run every API-enabled action. Only action types enabled for API execution by FROGGATE can be called at all — if an action the user needs is missing from `get-active-tables` `actions` even though it is granted, its type is not API-enabled; there is no way to enable it from the integrator side.
