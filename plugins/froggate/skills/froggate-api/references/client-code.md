# Client code: types, patterns, examples

Contents: TypeScript types · Implementation patterns — Backend (server-side, backend API key) · Public website (browser, `fgs_public_`) · Per-user frontend (browser, `fgs_login_` + session token) · Examples

## TypeScript types

```typescript
interface FgApiResponse<T> {
  status: "ok" | "error";
  statusInfo: string | null;
  errorCode: string | null;
  data: T | null;
}

interface FgColumnData { columnId: number; value: string | number | null; }
interface FgFile { fileName: string; base64file: string; note?: string; fileCategoryId?: number | null; }
interface FgFileUpdate { fileId: number; fileCategoryId: number | null; }

interface FgTableColumn {
  id: number;
  name: string;
  type: "varchar" | "text" | "float" | "date" | "datetime" | "selection" | "user" | "from_other_table" | "int" | "image";
  valueFormat: "string" | "number" | "integer" | "date" | "datetime" | "object";
  editable: boolean;
  notEditableReason?: string;
  possibleValues: Record<string, string> | null;
  linkedTableId?: number | null;
  linkedColumnId?: number | null;
}

interface FgTableStatus {
  id: number;
  name: string;
  tabView?: string;              // "table" | "kanban" | … — absent on a kanban column status
  kanbanParentStatusId?: number; // set when this status is a column of another status's board
  kanbanColumnName?: string;     // the label shown on the board column
}

// `statuses`, `columns` and `actions` are absent when called with verbose=false.
interface FgTable {
  tableId: number;
  menuId: number;
  parentMenuId: number | null;
  timeCreated: string;
  menuPath: string;
  mainMenuName: string;
  submenuName: string | null;
  statuses?: FgTableStatus[];
  columns?: FgTableColumn[];
  actions?: Array<Record<string, unknown>>; // run-action entries, see data-endpoints.md
}

interface FgImageValue {
  fileName: string;
  size: number;
  width: number;
  height: number;
}

interface FgRow {
  id: number;
  status: number;
  columns: Record<string, {
    columnId: number;
    // A string for text and selection, a number for float and link columns, an object for images.
    value: string | number | FgImageValue | null;
    valueId: number | null;
    valueDisplay: string | null;
  }>;
  files: Array<{
    fileName: string;
    fileId: number;
    type: string;
    fileCategory: { id: number; name: string } | null;
    base64file: string | null;
    publicUrl: string | null;
  }>;
}

interface FgVerifyLoginResponse {
  token: string;
  expires_in: number;
}

// /auth/register: both null while the new account waits for approval.
interface FgRegisterResponse {
  token: string | null;
  expires_in: number | null;
}

interface FgCodeRequestResponse {
  valid_minutes: number;
}
```

---

## Implementation patterns

Pick the client class that matches the scenario from "Choosing the right key".

### Backend (server-side, backend API key)

```typescript
class FroggateBackendApi {
  private baseUrl: string;
  private headers: Record<string, string>;

  constructor(companyId: string | number, apiKey: string) {
    this.baseUrl = `https://api.froggate.cloud/v1/${companyId}`;
    this.headers = {
      "Content-Type": "application/json",
      "Authorization": apiKey,
    };
  }

  async getTableStructure(tableId?: number): Promise<FgTable[]> {
    const res = await fetch(`${this.baseUrl}/dynamic-tables/get-active-tables`, {
      method: "POST",
      headers: this.headers,
      body: JSON.stringify(tableId !== undefined ? { tableId } : {}),
    });
    const json: FgApiResponse<{ tables: FgTable[] }> = await res.json();
    if (json.status !== "ok") throw new Error(json.statusInfo ?? "API error");
    return json.data!.tables;
  }

  async getRows(tableId: number, options?: {
    limit?: number;
    page?: number;
    order?: "ASC" | "DESC";
    showFiles?: boolean;
    returnColumns?: number[];
    filter?: { status?: number; id?: number; columnsValues?: Record<string, string | number | null>; columnsNotNull?: number[] };
  }): Promise<{ pages: number; rows: FgRow[] }> {
    const res = await fetch(`${this.baseUrl}/dynamic-tables/get-table-rows`, {
      method: "POST",
      headers: this.headers,
      body: JSON.stringify({
        tableId,
        parameters: {
          limit: options?.limit ?? 100,
          page: options?.page ?? 1,
          order: options?.order ?? "DESC",
          showFiles: options?.showFiles ?? false,
          ...(options?.returnColumns && { returnColumns: options.returnColumns }),
        },
        ...(options?.filter && { filter: options.filter }),
      }),
    });
    const json: FgApiResponse<{ parameters: { pages: number }; rows: FgRow[] }> = await res.json();
    if (json.status !== "ok") throw new Error(json.statusInfo ?? "API error");
    return { pages: json.data!.parameters.pages, rows: json.data!.rows };
  }

  async insertRow(
    tableId: number,
    statusId: number,
    columnsData: FgColumnData[],
    options?: { updateOnDuplicateColumnId?: number; files?: FgFile[] },
  ): Promise<{ rowId: number }> {
    const res = await fetch(`${this.baseUrl}/dynamic-tables/insert-table-row`, {
      method: "POST",
      headers: this.headers,
      body: JSON.stringify({
        tableId,
        statusId,
        columnsData,
        ...(options?.updateOnDuplicateColumnId !== undefined && {
          updateOnDuplicateColumnId: options.updateOnDuplicateColumnId,
        }),
        ...(options?.files && { files: options.files }),
      }),
    });
    const json: FgApiResponse<{ rowId: number }> = await res.json();
    if (json.status !== "ok") throw new Error(json.statusInfo ?? "API error");
    return { rowId: json.data!.rowId };
  }

  async updateRow(
    tableId: number,
    rowId: number,
    update: { statusId?: number; columnsData?: FgColumnData[]; insertFiles?: FgFile[]; updateFiles?: FgFileUpdate[] },
  ): Promise<void> {
    const res = await fetch(`${this.baseUrl}/dynamic-tables/update-table-row`, {
      method: "POST",
      headers: this.headers,
      body: JSON.stringify({ tableId, rowId, update }),
    });
    const json: FgApiResponse<unknown> = await res.json();
    if (json.status !== "ok") throw new Error(json.statusInfo ?? "API error");
  }

  async addSelectionValue(tableId: number, columnId: number, value: string): Promise<number> {
    const res = await fetch(`${this.baseUrl}/dynamic-tables/insert-column-selection-value`, {
      method: "POST",
      headers: this.headers,
      body: JSON.stringify({ tableId, columnId, value }),
    });
    const json: FgApiResponse<{ valueId: number }> = await res.json();
    if (json.status !== "ok") throw new Error(json.statusInfo ?? "API error");
    return json.data!.valueId;
  }
}

// Server-side env file (add to .gitignore before writing any code):
// FROGGATE_COMPANY_ID=30916
// FROGGATE_API_KEY=<paste-your-key-locally>
const api = new FroggateBackendApi(
  process.env.FROGGATE_COMPANY_ID!,
  process.env.FROGGATE_API_KEY!,
);
```

### Public website (browser, fgs_public_)

```typescript
class FroggatePublicApi {
  private baseUrl: string;
  private headers: Record<string, string>;

  constructor(companyId: string | number, publicKey: string) {
    if (!publicKey.startsWith("fgs_public_")) {
      throw new Error("FroggatePublicApi requires an fgs_public_… key");
    }
    this.baseUrl = `https://api.froggate.cloud/v1/${companyId}`;
    this.headers = {
      "Content-Type": "application/json",
      "Authorization": publicKey,
    };
  }

  async getTableStructure(tableId?: number): Promise<FgTable[]> {
    const res = await fetch(`${this.baseUrl}/dynamic-tables/get-active-tables`, {
      method: "POST",
      headers: this.headers,
      body: JSON.stringify(tableId !== undefined ? { tableId } : {}),
    });
    const json: FgApiResponse<{ tables: FgTable[] }> = await res.json();
    if (json.status !== "ok") throw new Error(json.statusInfo ?? "API error");
    return json.data!.tables;
  }

  async getRows(tableId: number, options?: {
    limit?: number;
    page?: number;
    order?: "ASC" | "DESC";
    returnColumns?: number[];
    filter?: { status?: number; id?: number; columnsValues?: Record<string, string | number | null>; columnsNotNull?: number[] };
  }): Promise<{ pages: number; rows: FgRow[] }> {
    const res = await fetch(`${this.baseUrl}/dynamic-tables/get-table-rows`, {
      method: "POST",
      headers: this.headers,
      body: JSON.stringify({
        tableId,
        parameters: {
          limit: options?.limit ?? 100,
          page: options?.page ?? 1,
          order: options?.order ?? "DESC",
          showFiles: false,
          ...(options?.returnColumns && { returnColumns: options.returnColumns }),
        },
        ...(options?.filter && { filter: options.filter }),
      }),
    });
    const json: FgApiResponse<{ parameters: { pages: number }; rows: FgRow[] }> = await res.json();
    if (json.status !== "ok") throw new Error(json.statusInfo ?? "API error");
    return { pages: json.data!.parameters.pages, rows: json.data!.rows };
  }

  async insertRow(tableId: number, statusId: number, columnsData: FgColumnData[]): Promise<number> {
    const res = await fetch(`${this.baseUrl}/dynamic-tables/insert-table-row`, {
      method: "POST",
      headers: this.headers,
      body: JSON.stringify({ tableId, statusId, columnsData }),
    });
    const json: FgApiResponse<{ rowId: number }> = await res.json();
    if (json.status !== "ok") throw new Error(json.statusInfo ?? "API error");
    return json.data!.rowId;
  }
}
```

### Per-user frontend (browser, fgs_login_ + session token)

```typescript
const SESSION_KEY = "froggate_session_token";

class FroggateLoginApi {
  private baseUrl: string;
  private loginKey: string;
  private sessionToken: string | null;

  constructor(companyId: string | number, loginKey: string) {
    if (!loginKey.startsWith("fgs_login_")) {
      throw new Error("FroggateLoginApi requires an fgs_login_… key");
    }
    this.baseUrl = `https://api.froggate.cloud/v1/${companyId}`;
    this.loginKey = loginKey;
    this.sessionToken = localStorage.getItem(SESSION_KEY);
  }

  isLoggedIn(): boolean {
    return this.sessionToken !== null;
  }

  // Returns how long the code is valid, so the code-entry screen can say it instead of
  // hardcoding a number the admin is free to change.
  async requestLoginCode(email: string): Promise<number> {
    const res = await fetch(`${this.baseUrl}/auth/request-login-code`, {
      method: "POST",
      headers: this.authHeaders(),
      body: JSON.stringify({ email }),
    });
    const json: FgApiResponse<FgCodeRequestResponse> = await res.json();
    if (json.status !== "ok" || !json.data) {
      // EMAIL_NOT_REGISTERED — only sent when the key discloses existence — is intentional;
      // route the user into sign-up. Re-throw with the errorCode so the UI can switch on it.
      const error = new Error(json.statusInfo ?? "API error");
      (error as Error & { code?: string }).code = json.errorCode ?? undefined;
      throw error;
    }
    return json.data.valid_minutes;
  }

  async verifyLoginCode(email: string, code: string): Promise<void> {
    const res = await fetch(`${this.baseUrl}/auth/verify-login-code`, {
      method: "POST",
      headers: this.authHeaders(),
      body: JSON.stringify({ email, code }),
    });
    const json: FgApiResponse<FgVerifyLoginResponse> = await res.json();
    if (json.status !== "ok" || !json.data) throw new Error(json.statusInfo ?? "Invalid code");
    this.sessionToken = json.data.token;
    localStorage.setItem(SESSION_KEY, this.sessionToken);
  }

  // Access-token mode: exchange the per-record token from the shared link for a session.
  // Same key, same session machinery as verifyLoginCode — just no e-mail/code step.
  async verifyAccessToken(token: string): Promise<void> {
    const res = await fetch(`${this.baseUrl}/auth/verify-access-token`, {
      method: "POST",
      headers: this.authHeaders(),
      body: JSON.stringify({ token }),
    });
    const json: FgApiResponse<FgVerifyLoginResponse> = await res.json();
    if (json.status !== "ok" || !json.data) throw new Error(json.statusInfo ?? "Invalid or expired access token");
    this.sessionToken = json.data.token;
    localStorage.setItem(SESSION_KEY, this.sessionToken);
  }

  async requestRegistrationCode(email: string): Promise<number> {
    const res = await fetch(`${this.baseUrl}/auth/request-registration-code`, {
      method: "POST",
      headers: this.authHeaders(),
      body: JSON.stringify({ email }),
    });
    const json: FgApiResponse<FgCodeRequestResponse> = await res.json();
    if (json.status !== "ok" || !json.data) {
      // EMAIL_ALREADY_REGISTERED is intentional — route the user into the login flow.
      // Re-throw with the errorCode so the UI layer can switch on it.
      const error = new Error(json.statusInfo ?? "Registration failed");
      (error as Error & { code?: string }).code = json.errorCode ?? undefined;
      throw error;
    }
    return json.data.valid_minutes;
  }

  // Returns true when the user is signed in, false when the account waits for approval.
  async register(email: string, code: string): Promise<boolean> {
    const res = await fetch(`${this.baseUrl}/auth/register`, {
      method: "POST",
      headers: this.authHeaders(),
      body: JSON.stringify({ email, code }),
    });
    const json: FgApiResponse<FgRegisterResponse> = await res.json();
    if (json.status !== "ok" || !json.data) {
      const error = new Error(json.statusInfo ?? "Registration failed");
      (error as Error & { code?: string }).code = json.errorCode ?? undefined;
      throw error;
    }
    // No token = the status for new users is not a login status: the row exists, but an
    // administrator has to approve it first. Storing null would read as "signed in".
    if (json.data.token === null) {
      return false;
    }
    // Same envelope as verifyLoginCode — store the token immediately so the UI can transition
    // straight into the logged-in state without an extra "please log in" round-trip.
    this.sessionToken = json.data.token;
    localStorage.setItem(SESSION_KEY, this.sessionToken);
    return true;
  }

  logout(): void {
    this.sessionToken = null;
    localStorage.removeItem(SESSION_KEY);
  }

  async getRows(tableId: number, options?: {
    limit?: number;
    page?: number;
    order?: "ASC" | "DESC";
    returnColumns?: number[];
    filter?: { columnsValues?: Record<string, string | number | null>; columnsNotNull?: number[] };
  }): Promise<{ pages: number; rows: FgRow[] }> {
    const data = await this.dataCall("/dynamic-tables/get-table-rows", {
      tableId,
      parameters: {
        limit: options?.limit ?? 100,
        page: options?.page ?? 1,
        order: options?.order ?? "DESC",
        showFiles: false,
        ...(options?.returnColumns && { returnColumns: options.returnColumns }),
      },
      ...(options?.filter && { filter: options.filter }),
    }) as { parameters: { pages: number }; rows: FgRow[] };
    return { pages: data.parameters.pages, rows: data.rows };
  }

  async insertRow(tableId: number, statusId: number, columnsData: FgColumnData[]): Promise<number> {
    const data = await this.dataCall("/dynamic-tables/insert-table-row", {
      tableId,
      statusId,
      columnsData,
    }) as { rowId: number };
    return data.rowId;
  }

  async updateRow(
    tableId: number,
    rowId: number,
    update: { statusId?: number; columnsData?: FgColumnData[] },
  ): Promise<void> {
    await this.dataCall("/dynamic-tables/update-table-row", { tableId, rowId, update });
  }

  private async dataCall(path: string, body: unknown): Promise<unknown> {
    if (!this.sessionToken) throw new Error("Not logged in");
    const res = await fetch(`${this.baseUrl}${path}`, {
      method: "POST",
      headers: { ...this.authHeaders(), "X-User-Session-Token": this.sessionToken },
      body: JSON.stringify(body),
    });
    if (res.status === 401) {
      this.logout();
      throw new Error("Session expired — please log in again");
    }
    const json: FgApiResponse<unknown> = await res.json();
    if (json.status !== "ok") throw new Error(json.statusInfo ?? "API error");
    return json.data;
  }

  private authHeaders(): Record<string, string> {
    return {
      "Content-Type": "application/json",
      "Authorization": this.loginKey,
    };
  }
}
```

### Examples

Submit a contact form from a public site:
```typescript
// Public key, browser. Table 25, columns: 110=name, 111=email, 112=message, status 1="New".
await api.insertRow(25, 1, [
  { columnId: 110, value: form.name },
  { columnId: 111, value: form.email },
  { columnId: 112, value: form.message },
]);
```

Login screen (e-mail → code → app):
```typescript
// Step 1 — with existence disclosure on (the default) an unknown e-mail means "offer sign-up",
// so one field can serve both cases. With it off this never fires and the screen instead shows
// a neutral "If the e-mail is registered, a code has been sent."
let validMinutes: number;
try {
  validMinutes = await api.requestLoginCode(email);
} catch (e) {
  if ((e as { code?: string }).code === "EMAIL_NOT_REGISTERED") {
    return startRegistration(email);
  }
  throw e;
}

// Step 2 — the code field says how long the user has, taken from the response rather than
// hardcoded, so it always agrees with the e-mail and survives the admin changing the setting.
showCodeField({ hint: `The code is valid for ${validMinutes} minutes.` });

// Step 3 — after the user types the code from the e-mail
await api.verifyLoginCode(email, code);

// From now on data calls just work — only the user's own rows.
const { rows } = await api.getRows(25);
```

Sign-up screen (e-mail → code → app, user is logged in immediately):
```typescript
// Step 1 — handle EMAIL_ALREADY_REGISTERED by redirecting to login instead of showing an error
let validMinutes: number;
try {
  validMinutes = await api.requestRegistrationCode(email);
} catch (e) {
  const errorCode = (e as { code?: string }).code;
  if (errorCode === "EMAIL_ALREADY_REGISTERED") {
    // Friendly UX: jump to the login screen with the e-mail pre-filled
    return showLoginScreen({ prefillEmail: email });
  }
  throw e;
}

// Step 2 — same rule as on the login screen: the validity comes from the response
showCodeField({ hint: `The code is valid for ${validMinutes} minutes.` });

// Step 3 — after the user types the code from the e-mail
const signedIn = await api.register(email, code);
if (!signedIn) {
  // The account exists but waits for an administrator's approval — no session yet.
  return showMessage("Thank you — your account is waiting for approval.");
}

// register() already stored the session token — go straight to the dashboard, do NOT
// route through the login screen.
navigateTo("/dashboard");
```

Shared-record link (e.g. a price quote opened from an e-mailed link — no login):
```typescript
// fgs_login_… key in Access-token mode. The token comes from the URL the recipient opened.
const token = new URLSearchParams(location.search).get("t");
if (!token) {
  return showError("This link is missing its access token.");
}
try {
  await api.verifyAccessToken(token);
} catch {
  // Single failure mode — link is wrong, expired, or the record left its reachable status.
  return showError("This link is no longer valid.");
}
// From now on data calls return only the shared record and its connected rows.
const { rows: quoteItems } = await api.getRows(quoteItemsTableId);
```

Upsert from a server-side webhook:
```typescript
// Backend, backend API key. If a row with the same e-mail already exists, update it.
await api.insertRow(25, 1, [
  { columnId: 110, value: payload.name },
  { columnId: 111, value: payload.email },
], { updateOnDuplicateColumnId: 111 });
```
