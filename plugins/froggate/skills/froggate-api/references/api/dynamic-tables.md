<!-- Generated from the FROGGATE OpenAPI specification (https://client.froggate.cloud/apidoc/api-bundled.yaml) — do not edit by hand. -->
# API reference: `/dynamic-tables/…`

Base URL: `https://api.froggate.cloud/v1/{companyId}` — replace `{companyId}` with the numeric company ID. Field names, types, allowed values and error codes below are exact.

Endpoints: `/dynamic-tables/get-active-tables` · `/dynamic-tables/insert-table-row` · `/dynamic-tables/update-table-row` · `/dynamic-tables/get-table-rows` · `/dynamic-tables/aggregate-table-rows` · `/dynamic-tables/insert-column-selection-value` · `/dynamic-tables/run-action`

## POST /dynamic-tables/get-active-tables

**Get active tables**

Get all tables and table data structure, allowed for your client account. All body fields are optional — send an empty `{}` to fetch every accessible table with full structure.

### Request body (`application/json`)

- `tableId` (integer) — If set, only the data of this table ID is returned. Example: `25`.
- `nameFilter` (string) — Case-insensitive substring filter on the section name (the table's menu item name, returned as `mainMenuName`). Wildcard characters `%` and `_` are treated literally. Useful when the company has many tables. Example: `zamě`.
- `verbose` (boolean) — If false, the response omits `statuses` and `columns` and returns only the table identifiers (`tableId`, `menuId`, `parentMenuId`, `timeCreated`) and menu names (`menuPath`, `mainMenuName`, `submenuName`). Much cheaper when you only need to locate a section by name. Default: `true`.

### Responses

#### 200 — OK response

Content type `application/json`:

- `status` (string) — Example: `ok`.
- `statusInfo` (string) — Example: `null`.
- `errorCode` (string) — Example: `null`.
- `data` (object)
  - `tables` (array of object)
    - `tableId` (integer) — Unique table ID Example: `10`.
    - `menuId` (integer) — Menu ID of this section itself — it identifies the section in the sidebar, and is NOT the id of the group it sits in; that one is parentMenuId below. Example: `42`.
    - `parentMenuId` (integer, nullable) — Menu ID of the group this section is filed under, or null when the section sits at the top level. Example: `12`.
    - `timeCreated` (string, format date-time) — Time when this table was created
    - `menuPath` (string) — Full menu path of this table Example: `Menu 1 - Menu item`.
    - `mainMenuName` (string) — Section name Example: `Menu item`.
    - `submenuName` (string, nullable) — Name of submenu is section belongs to submenu Example: `Menu 1`.
    - `statuses` (array of object) — Available statuses for this table. Omitted when `verbose=false`. Each status is one tab of the section, except statuses that are columns of a kanban board — see `tabView` / `kanbanParentStatusId` below.
      - `id` (integer) — Unique status ID Example: `1`.
      - `name` (string) — Status name. For a status that belongs to a kanban board this is NOT what the user sees on screen — use `kanbanColumnName`. Example: `Status 1`.
      - `tabView` (string) — How this status's tab displays its data: `table` = a plain table, `kanban` = a board whose columns are statuses (this status is the board's first column, followed by every status whose `kanbanParentStatusId` is this status ID). Absent on statuses that have no tab of their own — i.e. exactly those carrying `kanbanParentStatusId`. Treat the value as an open vocabulary: further display types may be added. Example: `kanban`.
      - `kanbanParentStatusId` (integer) — Only on a status that is a column of another status's kanban board (absent otherwise). Such a status has NO tab of its own — the user reaches it only as a column of the board given by this ID, and moving a card between columns changes the record's status. Example: `42`.
      - `kanbanColumnName` (string) — Only on statuses that belong to a kanban board — either the board itself or one of its columns (absent otherwise). The label the user sees on that board column; falls back to `name` when no separate column label is configured. Example: `Done`.
    - `columns` (array of object) — Column structure for this table. Omitted when `verbose=false`.
      - `id` (integer) — Unique column ID Example: `1`.
      - `name` (string) — Column name Example: `Column 1`.
      - `hint` (string) — Present only when the column has a hint (absent otherwise). The help text people see on the column header — what the column means and what belongs in it, written by whoever set up the section. Example: `Kolik kusů dílu jde do jednoho obalu.`.
      - `possibleValues` (object, nullable) — Only on selection columns: the allowed options as a map of option id → option name; the value written to the column is the id. Null while a selection column has no options yet, and always null for other types, including `from_other_table` — a link column has no option list; its permitted values are row ids of the section given by `linkedTableId`. Example: `{"125":"Option A","126":"Option B"}`.
        - each value (string)
      - `linkedTableId` (integer, nullable) — Only on `from_other_table` columns (absent on all other types). ID of the section this column links to — accepted values are row ids from that section. Null while the link has not been configured yet. Example: `1494`.
      - `linkedColumnId` (integer, nullable) — Only on `from_other_table` columns (absent on all other types). ID of the column from the linked section that is displayed as the value. Null while it has not been configured yet. Example: `7974`.
      - `editable` (boolean) — Whether values for this column can be written via insert-table-row / update-table-row. False for auto-filled columns (formulas, sums, paired non-editable, link-driven date fills, …) and for columns the current key has no write permission on. Example: `true`.
      - `notEditableReason` (string) — Present only when `editable` is false (absent otherwise). Why the column cannot be written. The first values describe the COLUMN — the section computes or reserves the value, so no key can ever write it and the value must be produced by changing the columns it is derived from. `columnTypeIsNeverWritableByApi` = the system fills the column itself (who created the record, when, the status change time, the record ID) or it is an image uploaded in the app; no setting changes that. The last two describe THE CALLER — `callerHasNoEditPermissionForThisColumn` and `columnIsOutsideTheApiKeyWriteScope` — so the same column may be writable for a different user or key. One of: `valueTakenFromLinkedRecord`, `filledByAutomaticSeries`, `composedFromOtherColumns`, `filledBySumFromLinkedSection`, `filledByFormula`, `filledByDateDifference`, `filledByDateFromLinkedRows`, `columnTypeIsNeverWritableByApi`, `writingThroughApiDisabledForThisColumn`, `callerHasNoEditPermissionForThisColumn`, `columnIsOutsideTheApiKeyWriteScope`. Example: `filledByFormula`.
      - `type` (string) — Type of this column. `text` columns store HTML — see https://client.froggate.cloud/apidoc/text-column-html-format.md for the allowed tag/attribute list when writing values. `from_other_table` is a link to another section — see `linkedTableId`. Auto-filled columns are reported by the kind of value they hold: who created the record = `user` (read like any user column), creation and status-change time = `datetime`. One of: `varchar`, `text`, `float`, `date`, `datetime`, `selection`, `user`, `from_other_table`, `int`, `image`.
      - `valueFormat` (string) — JSON value format expected by `insert-table-row` / `update-table-row` for this column. `string` = JSON string. `number` = JSON number (int or float, no quotes). `integer` = JSON integer (used for selection option ids, user ids, link-target row ids). `date` = JSON string in YYYY-MM-DD format. `datetime` = JSON string in YYYY-MM-DD HH:MM:SS format. `object` = image columns only: read-only, a read returns `{fileName, size, width, height}` or null. One of: `string`, `number`, `integer`, `date`, `datetime`, `object`.
    - `actions` (array of object) — Actions of this table that your API key may execute via `run-action`. Empty for most keys — an action appears here only when it is API-enabled and granted to your key. Omitted when `verbose=false`.
      - `actionId` (integer) — Action ID to pass to `run-action`. Example: `123`.
      - `name` (string) — Action name as configured in the section. Example: `Print labels`.
      - `resultType` (string) — What `run-action` returns for this action: `file` = a base64-encoded file, `message` = a text result. One of: `file`, `message`.
      - `fileType` (string, nullable) — File extension of the generated file. Null for `message` actions. Example: `pdf`.
      - `acceptsCount` (boolean) — Whether the action accepts the `count` parameter (e.g. number of labels per row).
      - `maxRows` (integer) — Maximum number of row ids in one `run-action` call for this action. Example: `500`.
      - `maxCount` (integer) — Maximum value of `count`. Present only when `acceptsCount` is true. Example: `500`.
      - `maxTotalItems` (integer) — Maximum of rows × count in one call. Present only when `acceptsCount` is true. Example: `2000`.
      - `allowedStatusIds` (array of integer, nullable) — Statuses in which rows may be processed by this action. Null = every status your key sees on this table.

#### 400 — Error response

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `API key is invalid for this company`.
- `errorCode` (string) — One of: `BAD_REQUEST`.
- `data` (null)


## POST /dynamic-tables/insert-table-row

**Insert table row**

Add new row to selected table.

### Request body (`application/json`)

- `tableId` (integer, format int64, required) — Example: `25`.
- `statusId` (integer, required) — Example: `15`.
- `columnsData` (array of object, required) — If the same columnId is listed more than once, only its last occurrence is used — for the stored value as well as for updateOnDuplicateColumnId.
  - `columnId` (integer) — Example: `110`.
  - `value` (one of the variants below) — Value to write. JSON type depends on the column's `valueFormat` from get-active-tables: string for text/date (YYYY-MM-DD)/datetime (YYYY-MM-DD HH:MM:SS), number for float (no quotes), integer for selection (option id) / user (user id) / link columns (target row id; a string of digits is accepted too, surrounding spaces ignored, any other value is refused). Length limits for string columns: `varchar` up to 200 characters (longer values are silently truncated), `text` up to ~16 000 characters. Values written to `text` columns are interpreted as HTML and sanitized server-side — see https://client.froggate.cloud/apidoc/text-column-html-format.md for the allowed tag/attribute list and required patterns.
    - variant 1: string — Example: `test 123`.
    - variant 2: number — Example: `42.5`.
    - variant 3: integer — Example: `7`.
    - variant 4: null
- `updateOnDuplicateColumnId` (integer) — In this parameter you can set one of the columnId from columnsData. If it is set and columnId value is found in table, all other columns are updated. If it is not found, the new row is inserted. This is useful if you don't want to have duplicate data in this table column, but want the other table columns to be updated. The id must be a column of the table given in tableId, otherwise the request is rejected. When the column is missing from columnsData or its value is empty, the parameter is ignored and a new row is inserted — an empty value never matches an existing row. Example: `110`.
- `files` (array of object)
  - `fileName` (string) — Example: `test.jpg`.
  - `base64file` (string, format byte) — Example: `JVBERi0xLjQKJdPr6eEKMSAwIG9iago8PC9UaXRsZSAodGVzdCkKL1Byb2R1…`.
  - `note` (string) — Example: `test`.
  - `fileCategoryId` (integer, nullable) — Optional. Files the attachment into one of the section's file categories. The category must belong to the same section, otherwise the request is refused. Omit it or send null to leave the file uncategorised. The ids are listed in the app under Settings -> API -> Element IDs. Example: `4`.

### Responses

#### 200 — OK response

Content type `application/json`:

- `status` (string) — Example: `ok`.
- `statusInfo` (string)
- `errorCode` (string) — On error: `NOT_FOUND` — the table, status, column, file category or `updateOnDuplicateColumnId` does not exist in this table; `statusInfo` names which. Read the current IDs from `get-active-tables`. Scope refusals (table or column outside the key's access) come back as HTTP 403 — see Errors in the introduction.
- `data` (object)
  - `rowId` (integer) — Unique row ID Example: `1`.
  - `files` (array of object) — Uploaded files
    - `fileName` (string) — Example: `test.jpg`.
    - `fileId` (integer) — Unique file ID Example: `10`.
    - `fileCategoryId` (integer, nullable) — File category the file is filed under after the call; null when it has none. Re-sending a file that is already attached to the record returns that file, so this reports its resulting category rather than what was requested. Example: `4`.
  - `updatedColumns` (object) — Empty unless updateOnDuplicateColumnId matched a row and changed it: then, of the columns sent (and the status), those whose stored value the call changed. A column sent with the value it already holds is not listed; a column the section recomputes on its own is never listed. Both values are the stored ones, in the column's own type (string, number or null) — date and date-time values in ISO 8601 (e.g. 2026-09-30T17:00:09+02:00).
    - `123` (object) — ID of affected column
      - `originalValue` (any) — Stored value before update — a string, number or null, by the column's type Example: `123`.
      - `newValue` (any) — Stored value after update — a string, number or null, by the column's type; may differ in form from the value sent (e.g. a date-time sent without seconds is stored with :00) Example: `456`.

#### 400 — Error response

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `API key is invalid for this company`.
- `errorCode` (string) — One of: `BAD_REQUEST`.
- `data` (null)


## POST /dynamic-tables/update-table-row

**Update table row**

Update row in selected table. To delete a row, set `statusId` to `0` — there is no separate delete endpoint, this is the system-wide soft-delete convention.

### Request body (`application/json`)

- `tableId` (integer, format int64, required) — Example: `25`.
- `rowId` (integer, required) — Example: `1`.
- `update` (object, required)
  - `columnsData` (array of object)
    - `columnId` (integer) — Example: `110`.
    - `value` (one of the variants below) — Value to write. JSON type depends on the column's `valueFormat` from get-active-tables: string for text/date (YYYY-MM-DD)/datetime (YYYY-MM-DD HH:MM:SS), number for float (no quotes), integer for selection (option id) / user (user id) / link columns (target row id; a string of digits is accepted too, surrounding spaces ignored, any other value is refused). Length limits for string columns: `varchar` up to 200 characters (longer values are silently truncated), `text` up to ~16 000 characters. Values written to `text` columns are interpreted as HTML and sanitized server-side — see https://client.froggate.cloud/apidoc/text-column-html-format.md for the allowed tag/attribute list and required patterns.
      - variant 1: string — Example: `test 123`.
      - variant 2: number — Example: `42.5`.
      - variant 3: integer — Example: `7`.
      - variant 4: null
  - `statusId` (integer) — Example: `10`.
  - `insertFiles` (array of object)
    - `fileName` (string) — Example: `test.jpg`.
    - `base64file` (string, format byte) — Example: `JVBERi0xLjQKJdPr6eEKMSAwIG9iago8PC9UaXRsZSAodGVzdCkKL1Byb2R1…`.
    - `note` (string) — Example: `test`.
    - `fileCategoryId` (integer, nullable) — Optional. Files the attachment into one of the section's file categories. The category must belong to the same section, otherwise the request is refused. Omit it or send null to leave the file uncategorised. The ids are listed in the app under Settings -> API -> Element IDs. Example: `4`.
  - `updateFiles` (array of object) — Refiles files already attached to this row into a different file category. The file content is not touched.
    - `fileId` (integer, required) — Id of a file already attached to this row. A file of another row is refused. Example: `10`.
    - `fileCategoryId` (integer, nullable, required) — The section's file category to file it under, or null to leave it uncategorised. The category must belong to the same section. The ids are listed in the app under Settings -> API -> Element IDs. Example: `4`.

### Responses

#### 200 — OK response

Content type `application/json`:

- `status` (string) — Example: `ok`.
- `statusInfo` (string)
- `errorCode` (string) — On error: `NOT_FOUND` — the table, row, status, column, file category or file does not exist; `statusInfo` names which. Read the current IDs from `get-active-tables`. Scope refusals (table, column, status or row outside the key's access) come back as HTTP 403 — see Errors in the introduction.
- `data` (object)
  - `files` (array of object) — Uploaded files
    - `fileName` (string) — Example: `test.jpg`.
    - `fileId` (integer) — Unique file ID Example: `10`.
    - `fileCategoryId` (integer, nullable) — File category the file is filed under after the call; null when it has none. Re-sending a file that is already attached to the record returns that file, so this reports its resulting category rather than what was requested. Example: `4`.
  - `updatedFiles` (array of object) — Files refiled by updateFiles
    - `fileId` (integer) — Unique file ID Example: `10`.
    - `fileCategoryId` (integer, nullable) — File category the file is now filed under Example: `4`.
  - `updatedColumns` (object) — Of the columns sent (and the status), those whose stored value this call changed. A column sent with the value it already holds is not listed; a column the section recomputes on its own is never listed. Both values are the stored ones, in the column's own type (string, number or null) — date and date-time values in ISO 8601 (e.g. 2026-09-30T17:00:09+02:00).
    - `123` (object) — ID of affected column
      - `originalValue` (any) — Stored value before update — a string, number or null, by the column's type Example: `123`.
      - `newValue` (any) — Stored value after update — a string, number or null, by the column's type; may differ in form from the value sent (e.g. a date-time sent without seconds is stored with :00) Example: `456`.

#### 400 — Error response

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `API key is invalid for this company`.
- `errorCode` (string) — One of: `BAD_REQUEST`.
- `data` (null)


## POST /dynamic-tables/get-table-rows

**Get table rows**

Get data of selected table rows

### Request body (`application/json`)

- `tableId` (integer, format int64, required) — Example: `25`.
- `parameters` (object) — Main parameters for request
  - `limit` (integer) — Number of records per page. A larger value is refused with `INVALID_PARAMETER`; read more rows page by page. Minimum `1`, maximum `1000`. Default: `100`. Example: `10`.
  - `page` (integer) — Page number, starting at 1. The response's `parameters.pages` says how many pages there are at the requested `limit`. Minimum `1`. Default: `1`. Example: `1`.
  - `order` (string) — Order of records. Records are ordered by the time they were added (`ASC` = oldest first). They follow their manual order (records dragged into place) instead when `filter.status` asks for exactly one status whose tab is sorted manually — by its own setting, by the section's, or because it is shown as a kanban board (in the order of its cards) — or, without a filter for exactly one status, when the whole section is sorted manually. `ASC` then lists them top to bottom as the table shows them, `DESC` bottom to top. One of: `ASC`, `DESC`. Default: `DESC`. Example: `ASC`.
  - `showFiles` (boolean) — If true, the base64 content of every file attached to a row is sent in the response. Otherwise only the file metadata is sent. Either way the files list is returned only when the API key is allowed to read files on this table. Default: `false`. Example: `true`.
  - `returnColumns` (array of integer) — If set, each returned row's columns map contains only these column IDs. If omitted, every column the key can see is returned. Column IDs that are inaccessible or do not exist on the table are ignored.
- `filter` (object) — If this parameters is set, records are filtered by selected filters. All filters are applied. If you want to filter for example just by status - add only status parameter. Only columns the API key can read may be used in columnsValues, columnsNotNull and conditions; any other column is refused with an error naming it. A column of a linked section is usable when it is the display value (valueDisplay) of a link column the key reads, or when the key may read that section (for a signed-in user only a shared section). An empty list ([]) counts as no filter.
  - `status` (integer) — ID of status Example: `15`.
  - `id` (integer) — ID of row Example: `26`.
  - `columnsValues` (object) — If this parameters is set, rows are filtered by an exact match on column values. Each key is a column ID (surrounding spaces are ignored; a key made of anything else than digits is refused). A string/number value matches rows where the column equals that value; a null value matches rows where the column is empty (IS NULL). All conditions are combined with AND.
    - `123` (string|int|null) — '123' is just example - it is ID of column. The row's value in this column must equal the given value, or be empty when the value is null. If this table is connected to another table, you can also use the column ID from the target table; only connected rows the API key can read count (rows in the statuses the key allows for that table, never deleted rows). Example: `test`.
    - `456` (string|int|null) — Example: `test`.
  - `columnsNotNull` (array of integer) — Column IDs that must be non-empty (IS NOT NULL). To match empty cells (IS NULL) instead, pass the column with a null value in columnsValues.
  - `conditions` (array of object) — Conditions with an operator — ranges (dates, numbers), lists, text search, empty / filled. Use them where an exact match in `columnsValues` is not enough, for example rows of one month: `{"columnId": 123, "operator": "between", "value": ["2026-09-01", "2026-09-30"]}`. Max items `50`.
    - `columnId` (integer, required) — Column ID — of this table, or of a table this one links to (joined the same way as in `columnsValues`). Only a column the API key can read; any other is refused with an error naming it. Example: `123`.
    - `operator` (string, required) — - `eq` / `neq` — equal / not equal. `neq` also matches empty cells. - `gt`, `gte`, `lt`, `lte` — greater than, greater or equal, less than, less or equal. - `between` — `value` is `[from, to]`, both included. On a number or a date `from` after `to` is refused with `INVALID_PARAMETER`, not answered with an empty result. - `in` / `notIn` — `value` is a list (at most 1000 values). `notIn` also matches empty cells. - `contains` — the text contains `value` (case-insensitive). - `empty` / `notEmpty` — the cell is empty / filled; no `value`. On a text column an empty text counts as empty. Which operators a column takes depends on its type: a number, a date and a text take all of them except that `contains` is for texts only; a date and time takes `eq`, `neq`, the ranges and `between`, `empty` and `notEmpty`; a selection, a user, a who-added and a link column take `eq`, `neq`, `in`, `notIn`, `empty` and `notEmpty`; any other column only `empty` and `notEmpty`. One of: `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `between`, `in`, `notIn`, `contains`, `empty`, `notEmpty`. Example: `between`.
    - `value` (one of the variants below) — The value to compare with — omitted for `empty` / `notEmpty`, a list of two for `between`, a list for `in` / `notIn`. A number column takes numbers; a date column a date `YYYY-MM-DD`; a date-and-time column `YYYY-MM-DD HH:MM:SS` or a bare date `YYYY-MM-DD`, which stands for the whole day (`lte 2026-09-30` includes that evening, `eq 2026-09-30` matches any time on that day); a selection column option IDs (as in `get-active-tables` → `possibleValues`; a number or the same digits as text) or option names, and a value that is neither an option ID nor an option name of the column is refused; a user or who-added column user IDs; a link column IDs of rows in the linked table; a text column a text. Example: `["2026-07-01","2026-09-30"]`.
      - variant 1: string
      - variant 2: number
      - variant 3: array of one of the variants below
        - variant 1: string
        - variant 2: number

### Responses

#### 200 — OK response

Content type `application/json`:

- `status` (string) — Example: `ok`.
- `statusInfo` (string)
- `errorCode` (string) — On error: `NOT_FOUND` — no active table with this `tableId`. `NOT_ALLOWED` — the API key's user has no access to the data of this table (the section's permissions in FROGGATE). Scope refusals (filter on a column the key may not read) come back as HTTP 403 — see Errors in the introduction.
- `data` (object)
  - `parameters` (object)
    - `pages` (integer) — Total pages count Example: `1`.
    - `rows` (integer) — Total number of rows matching the filter, across all pages Example: `437`.
  - `rows` (array of object)
    - `id` (integer) — Unique row ID Example: `1`.
    - `status` (integer) — Status ID Example: `10`.
    - `columns` (object)
      - `123` (object)
        - `columnId` (integer) — Example: `123`.
        - `value` (mixed) — Example: `test`.
        - `valueId` (null|integer) — The option ID on selection columns and the user ID on user columns, including the auto-filled column of who created the record (`value` then holds the option's or the user's name; both null on a record the system created); null on every other column Example: `123`.
        - `valueDisplay` (mixed) — Display value from target table Example: `Text`.
      - `456` (object)
        - `columnId` (integer) — Example: `456`.
        - `value` (mixed) — Example: `test`.
        - `valueId` (null|integer) — The option ID on selection columns and the user ID on user columns, including the auto-filled column of who created the record (`value` then holds the option's or the user's name; both null on a record the system created); null on every other column Example: `123`.
        - `valueDisplay` (mixed) — Display value from target table Example: `Text`.
    - `files` (array of object) — Files attached to the record. Returned only when the API key is allowed to read files on this table (see the File read switch in the key's permissions for that table); otherwise the list is empty. Full-access keys always receive it. The switch is independent of the allowed read columns — attachments belong to the record, not to a column.
      - `fileName` (string) — File name Example: `test.jpg`.
      - `fileId` (integer) — Unique file ID Example: `10`.
      - `type` (string) — File type Example: `standard`.
      - `fileCategory` (object, nullable) — File category the file is filed under; null when it has none.
        - `id` (integer) — Example: `4`.
        - `name` (string) — Example: `Smlouva`.
      - `base64file` (string, nullable) — Base64 encoded file content; null if showFiles is false
      - `publicUrl` (string, nullable) — Public URL of the file if available and not expired; null otherwise. Always returned regardless of showFiles. Example: `https://cdn.froggate.cloud/...`.

#### 400 — Error response

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `API key is invalid for this company`.
- `errorCode` (string) — One of: `BAD_REQUEST`.
- `data` (null)


## POST /dynamic-tables/aggregate-table-rows

**Aggregate table rows**

Computes figures over the rows of a table on the server — per column the sum, average, minimum, maximum and the number of filled cells, plus row counts — optionally grouped by a column, by a date period or by status. Use it for totals, counts, averages, shares and charts instead of reading every row with `get-table-rows` and adding them up in your code: it answers in one call whatever the size of the table.

**Which keys:** a backend key, a login-flow (signed-in user) key and the temporary key an application receives inside a FROGGATE section (`fgs_temp_…`, with the scope of the key it was issued from). A public key gets HTTP 403 — it may only read and insert rows.

**Which rows:** exactly those `get-table-rows` returns with the same `filter` — the key's allowed statuses and columns, and for a signed-in user only their own rows (or a shared table).

**Result:** `rowCount` and `columns` are the totals over all matching rows. With `groupBy` come also `groups` (at most `limit` of them, ordered by their values unless `orderBy` says otherwise), `groupCount` (all groups; absent for a top list over more than 10000 groups) and `truncated`. A figure a column's type does not have is left out, never `null`; `sum`, `avg`, `min` and `max` are `null` when no cell in the group is filled.

**Limits:** one aggregation runs for a limited time — a heavier one is stopped with `AGGREGATION_TIMEOUT`; split it into smaller parts (one call per year or quarter with a date range) and combine the results — sums, counts, minimum and maximum combine exactly, an average is the combined sum divided by the combined `filled` — or group more coarsely. Never present a figure for a narrower part as the answer to the whole question. A caller may run only a few aggregations at once; more come back as `TOO_MANY_REQUESTS` — retry when the earlier ones finish.

### Request body (`application/json`)

- `tableId` (integer, format int64, required) — Example: `25`.
- `columns` (array of integer) — Columns of this table to compute figures for. Each comes back with every figure its type has — a number: `sum`, `avg`, `min`, `max`, `filled`; a date or date and time: `min`, `max`, `filled`; any other column: `filled` (the number of filled cells). List the columns that make sense to add up: a number column that holds an identifier (a registration number, a postcode) is summed too if it is listed. Without `columns` only row counts come back. Only columns the API key can read; any other is refused with HTTP 403 naming it. Max items `20`.
- `groupBy` (array of object) — Splits the rows into groups, one per distinct value; two items group by both (for example month × status). Without it the response carries only the totals. A long-text or image column cannot be grouped by. Max items `2`.
  - `columnId` (integer) — Group by this column of the table. Either `columnId` or `status`, not both. Example: `124`.
  - `dateBucket` (string) — Only for a date or date-and-time column: the period one group covers. Group values are `2026-09-30` (day), `2026-W39` (ISO week), `2026-09` (month), `2026-Q3` (quarter), `2026` (year). One of: `day`, `week`, `month`, `quarter`, `year`. Default: `day`.
  - `status` (boolean) — `true` groups by status. Example: `true`.
- `orderBy` (object) — Orders the groups by a figure instead of by their values (the default, ascending). Needs `groupBy`. With `limit` it gives a top list, for example the ten customers with the highest revenue — also when there are more than 10000 groups (then the response carries no `groupCount`); without `orderBy` that many groups are refused with `TOO_MANY_GROUPS`.
  - `function` (string, required) — `count` orders by the number of rows in a group; the others by that figure of the column in `columnId`. One of: `count`, `sum`, `avg`, `min`, `max`, `filled`. Example: `sum`.
  - `columnId` (integer) — A column listed in `columns`, required for every function except `count`. `sum` and `avg` apply to number columns, `min` and `max` to numbers and dates. Example: `123`.
  - `direction` (string) — One of: `ASC`, `DESC`. Default: `DESC`.
- `limit` (integer) — How many groups to return. `groupCount` and `truncated` in the response say whether there are more; the totals always cover all groups. Minimum `1`, maximum `1000`. Default: `100`.
- `filter` (object) — The same filter as in `get-table-rows` (`status`, `id`, `columnsValues`, `columnsNotNull`, `conditions`) — the figures are computed over exactly the rows `get-table-rows` would return with it.
  - `status` (integer) — ID of status Example: `15`.
  - `columnsValues` (object) — Exact match on column values, as in `get-table-rows`.
  - `columnsNotNull` (array of integer) — Column IDs that must be non-empty.
  - `conditions` (array of object) — Max items `50`.
    - `columnId` (integer, required) — Column ID — of this table, or of a table this one links to (joined the same way as in `columnsValues`). Only a column the API key can read; any other is refused with an error naming it. Example: `123`.
    - `operator` (string, required) — - `eq` / `neq` — equal / not equal. `neq` also matches empty cells. - `gt`, `gte`, `lt`, `lte` — greater than, greater or equal, less than, less or equal. - `between` — `value` is `[from, to]`, both included. On a number or a date `from` after `to` is refused with `INVALID_PARAMETER`, not answered with an empty result. - `in` / `notIn` — `value` is a list (at most 1000 values). `notIn` also matches empty cells. - `contains` — the text contains `value` (case-insensitive). - `empty` / `notEmpty` — the cell is empty / filled; no `value`. On a text column an empty text counts as empty. Which operators a column takes depends on its type: a number, a date and a text take all of them except that `contains` is for texts only; a date and time takes `eq`, `neq`, the ranges and `between`, `empty` and `notEmpty`; a selection, a user, a who-added and a link column take `eq`, `neq`, `in`, `notIn`, `empty` and `notEmpty`; any other column only `empty` and `notEmpty`. One of: `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `between`, `in`, `notIn`, `contains`, `empty`, `notEmpty`. Example: `between`.
    - `value` (one of the variants below) — The value to compare with — omitted for `empty` / `notEmpty`, a list of two for `between`, a list for `in` / `notIn`. A number column takes numbers; a date column a date `YYYY-MM-DD`; a date-and-time column `YYYY-MM-DD HH:MM:SS` or a bare date `YYYY-MM-DD`, which stands for the whole day (`lte 2026-09-30` includes that evening, `eq 2026-09-30` matches any time on that day); a selection column option IDs (as in `get-active-tables` → `possibleValues`; a number or the same digits as text) or option names, and a value that is neither an option ID nor an option name of the column is refused; a user or who-added column user IDs; a link column IDs of rows in the linked table; a text column a text. Example: `["2026-07-01","2026-09-30"]`.
      - variant 1: string
      - variant 2: number
      - variant 3: array of one of the variants below
        - variant 1: string
        - variant 2: number

### Responses

#### 200 — OK response

Content type `application/json`:

- `status` (string) — Example: `ok`.
- `statusInfo` (string)
- `errorCode` (string) — On error: `NOT_FOUND` — no active table with this `tableId`. `NOT_ALLOWED` — the API key's user has no access to the data of this table. `AGGREGATION_TIMEOUT` — the aggregation took too long and was stopped; split it into smaller parts (e.g. by year with a date range) and combine the results, or group more coarsely. `TOO_MANY_GROUPS` — the grouping produces more than 10000 groups and no `orderBy` was given; for a top list add `orderBy` and `limit`, otherwise group by a column with fewer distinct values or use a coarser `dateBucket`. `TOO_MANY_REQUESTS` — other aggregations of the same caller are still running; retry when they finish. A column the key may not read (in `columns`, `groupBy` or the filter) comes back as HTTP 403 — see Errors in the introduction.
- `data` (object)
  - `tableId` (integer) — Example: `25`.
  - `rowCount` (integer) — Number of rows matching the filter. Example: `1250`.
  - `columns` (array of object) — Totals over all matching rows, one item per column in `columns`, in the same order.
    - `columnId` (integer) — Example: `123`.
    - `sum` (number, nullable) — Number columns only. Example: `184250.5`.
    - `avg` (number, nullable) — Number columns only — the sum divided by the number of filled cells. Example: `1771.64`.
    - `min` (any, nullable) — Number, date and date-and-time columns only (a date as `YYYY-MM-DD`, a date and time as `YYYY-MM-DD HH:MM:SS`). Example: `12`.
    - `max` (any, nullable) — Number, date and date-and-time columns only. Example: `9800`.
    - `filled` (integer) — Number of filled cells (an empty text counts as empty). Example: `104`.
  - `groupBy` (array of object) — The grouping as applied (with the default `dateBucket` filled in). Present only with `groupBy`.
  - `groupCount` (integer) — Number of all groups. Present only with `groupBy`, and absent for a top list (`orderBy` + `limit`) over more than 10000 groups — then only `truncated` says there are more. Example: `12`.
  - `truncated` (boolean) — True when more groups exist than `limit` returned. Present only with `groupBy`.
  - `groups` (array of object) — Present only with `groupBy`.
    - `keys` (array of object) — One item per `groupBy` item, in its order. `value` is the group's value: a period for a date bucket, the option name for a selection, the user's name for a user or who-added column, the linked row's display value for a link column, the status name for the status; `null` for empty cells. A selection, user, who-added, link and status key also carries `id` (option, user, linked row, status).
      - `value` (mixed) — Example: `2026-03`.
      - `id` (integer, nullable)
    - `rowCount` (integer) — Example: `104`.
    - `columns` (array of object)
      - `columnId` (integer) — Example: `123`.
      - `sum` (number, nullable) — Number columns only. Example: `184250.5`.
      - `avg` (number, nullable) — Number columns only — the sum divided by the number of filled cells. Example: `1771.64`.
      - `min` (any, nullable) — Number, date and date-and-time columns only (a date as `YYYY-MM-DD`, a date and time as `YYYY-MM-DD HH:MM:SS`). Example: `12`.
      - `max` (any, nullable) — Number, date and date-and-time columns only. Example: `9800`.
      - `filled` (integer) — Number of filled cells (an empty text counts as empty). Example: `104`.

#### 400 — Error response

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `API key is invalid for this company`.
- `errorCode` (string) — One of: `BAD_REQUEST`.
- `data` (null)


## POST /dynamic-tables/insert-column-selection-value

**Insert column selection value**

Add new selection value to column.

### Request body (`application/json`)

- `tableId` (integer, format int64, required) — Example: `25`.
- `columnId` (integer, required) — Example: `15`.
- `value` (string, required) — Value of selection. If value with same name is already added, new value is not added Example: `test`.

### Responses

#### 200 — OK response

Content type `application/json`:

- `status` (string) — Example: `ok`.
- `statusInfo` (string)
- `errorCode` (string) — On error: `NOT_FOUND` — no column with this ID in the table. Read the current IDs from `get-active-tables`.
- `data` (object)
  - `valueId` (integer) — Unique selection value ID Example: `1`.

#### 400 — Error response

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `API key is invalid for this company`.
- `errorCode` (string) — One of: `BAD_REQUEST`.
- `data` (null)


## POST /dynamic-tables/run-action

**Run action**

Execute a configured table action (e.g. generating PDF labels) on one or more rows — the same operation a user triggers from the table, with the same configuration. The caller only picks the rows (and optionally a count); what the action does is fully defined by its configuration in the section settings.

**Which actions can be called:** only action types enabled for API execution by FROGGATE, and only actions granted to your API key (Settings → API → key detail → the allowed table's detail → Allowed actions; a full-access key may run every API-enabled action). The executable actions, their IDs and limits are listed per table in the `actions` field of `get-active-tables`.

**Row conditions:** every row must belong to the action's table, must not be deleted, and its current status must be allowed — both by the key's table status scope and by the per-action status list. A key restricted to a specific user may only process that user's rows.

**Result:** depends on the action — `resultType: "file"` returns the generated file base64-encoded in `data.contentBase64` (one file covering all requested rows), `resultType: "message"` returns a text result (for actions processing rows one by one, a per-row `results` list). A file result is all-or-nothing: it either covers every requested row or the call fails as a whole with `data: null`.

**Partial failure:** an action processing rows one by one may fail on some rows and succeed on others. The response then has `status: "error"` (HTTP 200) with `errorCode: "ACTION_PARTIALLY_FAILED"` (`"ACTION_FAILED"` when every row failed) and — unlike every other error — `data` stays populated: `processedRowIds` lists the rows that ran (executed rows are never rolled back), `errors` the failed rows with a per-row reason. After a partial failure re-send only the rows missing from `processedRowIds`, never the whole batch. A partial-failure response never contains `contentBase64`.

### Request body (`application/json`)

- `actionId` (integer, required) — ID of the action to execute. Your key's executable actions (with their IDs and limits) are listed per table in the `actions` field of get-active-tables. Example: `123`.
- `rowIds` (array of integer, required) — IDs of the rows to process. All rows must belong to the action's table, must not be deleted, and must be in a status the action is enabled for. The per-call maximum is the action's `maxRows` from get-active-tables.
- `count` (integer) — Optional. How many items to produce per row (e.g. number of labels). Accepted only by actions with `acceptsCount = true` — sending it to other actions is an error, not ignored. Range 1–`maxCount`; rows × count must not exceed `maxTotalItems`. Example: `4`.

### Responses

#### 200 — OK response

Content type `application/json`:

- `status` (string) — Example: `ok`.
- `statusInfo` (string) — Example: `Action executed`.
- `errorCode` (string, nullable) — Null on success. Error codes: `ACTION_FAILED` (the action rejected the request or failed as a whole), `ACTION_PARTIALLY_FAILED` (a per-row action failed on some rows — see `data.errors`; processed rows are NOT rolled back), `ACTION_NOT_CONFIGURED` (the action's configuration in the section is incomplete), `LIMIT_EXCEEDED` (too many rows or rows × count over the limit), `NOT_FOUND` (unknown action / row, deleted row, or an action not executable through the API). With `ACTION_PARTIALLY_FAILED` — and with `ACTION_FAILED` from a per-row action — `data` stays populated (`processedRowIds`, `results`, `errors`); every other error has `data: null`. Example: `null`.
- `data` (object)
  - `resultType` (string) — One of: `file`, `message`.
  - `fileName` (string) — Suggested file name including extension. Only for `file` results. Example: `labels.pdf`.
  - `fileType` (string) — File extension. Only for `file` results. Example: `pdf`.
  - `contentBase64` (string) — The generated file, base64-encoded. Only for `file` results.
  - `message` (string, nullable) — Text result of the action. Only for `message` results of actions processing all rows in one run.
  - `results` (array of object) — Per-row results. Only for `message` results of actions processing rows one by one.
    - `rowId` (integer)
    - `message` (string, nullable)
  - `errors` (array of object) — Rows the action failed on. Present only with `ACTION_PARTIALLY_FAILED` / `ACTION_FAILED` of per-row actions.
    - `rowId` (integer)
    - `error` (string)
  - `processedRowIds` (array of integer) — Rows the action actually processed.

#### 400 — Error response

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `API key is invalid for this company`.
- `errorCode` (string) — One of: `BAD_REQUEST`.
- `data` (null)

#### 403 — Scope refusal — the call is outside what the API key is allowed to do

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `This action is not in the allowed actions list of this API key`.
- `errorCode` (string) — `ACTION_NOT_ALLOWED` — the action is not granted to this API key. `TABLE_NOT_ALLOWED` — no read access to the action's table, or the table cannot be used with this key type. `ROW_STATUS_NOT_ALLOWED` — a row's current status is outside the key's table-level or per-action status list. `ROW_NOT_ALLOWED` — a row does not belong to the logged-in user. `BAD_REQUEST` — other refusals (origin/IP restrictions, endpoint not available for this key type). One of: `ACTION_NOT_ALLOWED`, `TABLE_NOT_ALLOWED`, `ROW_STATUS_NOT_ALLOWED`, `ROW_NOT_ALLOWED`, `BAD_REQUEST`.
- `data` (null)
