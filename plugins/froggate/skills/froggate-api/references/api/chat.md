<!-- Generated from the FROGGATE OpenAPI specification (https://client.froggate.cloud/apidoc/api-bundled.yaml) — do not edit by hand. -->
# API reference: `/chat/…`

Base URL: `https://api.froggate.cloud/v1/{companyId}` — replace `{companyId}` with the numeric company ID. Field names, types, allowed values and error codes below are exact.

Endpoints: `/chat/get-channels` · `/chat/get-channel-members` · `/chat/get-messages` · `/chat/send-message`

## POST /chat/get-channels

**List chat channels**

The chat channels this API key reaches — the channels its user is a member of. Direct messages and group conversations are never reachable through the API.

**Access:** the key needs "Read chat" (Settings → API → the key's Permissions); full access does not include the chat. Only the API key itself reaches the chat; public, login and temporary keys derived from it never do. A key of an API user reaches the channels the app was added to — someone who manages the channel adds it in the chat, in the channel settings under Apps.

**Channel ID:** every chat endpoint identifies the channel by its channel ID — 10 letters and digits, shown to every member in the chat under the channel's ⋯ menu → Info. A program that always writes to the same channel can take the ID from there and skip this call.

**Following the chat:** `lastMessageId` is the newest message of the channel, thread replies included. Keep the last ID you read per channel and call get-messages only for channels whose `lastMessageId` is higher.

### Request body (`application/json`)

- object — No parameters.

### Responses

#### 200 — OK response

Content type `application/json`:

- `status` (string) — Example: `ok`.
- `statusInfo` (string, nullable) — Example: `null`.
- `errorCode` (string, nullable) — Null on success. `CHAT_NOT_ALLOWED` (HTTP 403) — the key lacks "Read chat", or it is a public, login or temporary key, which never reach the chat. Example: `null`.
- `data` (object)
  - `channels` (array of object)
    - `id` (string) — Channel ID — the same ID the chat shows under the channel's ⋯ menu → Info. Pass it as channelId to the other chat endpoints. Example: `K7Q2M9XA1B`.
    - `name` (string) — Example: `sklad`.
    - `isPrivate` (boolean)
    - `isShared` (boolean) — Shared with another company.
    - `isArchived` (boolean) — An archived channel can be read, not written to.
    - `canPost` (boolean) — May this key post a main-line message here right now (the key's write access, the member's right and the archive combined).
    - `canReplyInThreads` (boolean) — May this key reply in threads here right now.
    - `requiresForm` (boolean) — True for a ticket-style channel that requires a title or a required field without a default value on every main-line message — the API cannot fill those in, so a main-line send-message there is refused with CHANNEL_REQUIRES_FORM; a thread reply still works.
    - `lastMessageId` (integer, nullable) — ID of the newest message including thread replies; null for an empty channel.
    - `lastMessageAt` (string, nullable) — Example: `2026-09-23 15:30:00`.

#### 400 — Error response

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `API key is invalid for this company`.
- `errorCode` (string) — One of: `BAD_REQUEST`.
- `data` (null)

#### 403 — Refusal — the API key may not use the chat this way

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `This API key may not read the chat. Allow "Read chat" for th…`.
- `errorCode` (string) — `CHAT_NOT_ALLOWED` — the key lacks "Read chat" / "Write to chat", or it is a key derived from the API key (public, login or temporary), which never reaches the chat. `BAD_REQUEST` — other refusals (origin or IP restrictions, a key of an API user not linked to an application, an endpoint not available for this key type). One of: `CHAT_NOT_ALLOWED`, `BAD_REQUEST`.
- `data` (null)


## POST /chat/get-channel-members

**List members of a chat channel**

Who is in a channel — use it to get the user ID for a mention in send-message. In a channel shared with another company, that company's members appear only once they have written or reacted in the channel.

### Request body (`application/json`)

- `channelId` (string, required) — Channel ID (10 letters and digits) — `id` from get-channels, or the Channel ID shown in the chat under the channel's ⋯ menu → Info. Example: `K7Q2M9XA1B`.

### Responses

#### 200 — OK response

Content type `application/json`:

- `status` (string) — Example: `ok`.
- `statusInfo` (string, nullable) — Example: `null`.
- `errorCode` (string, nullable) — Null on success. `CHANNEL_NOT_FOUND` — the channel is not among the channels this key reaches. `CHAT_NOT_ALLOWED` (HTTP 403) — the key lacks "Read chat", or it is a public, login or temporary key, which never reach the chat. Example: `null`.
- `data` (object)
  - `channelId` (string) — Example: `K7Q2M9XA1B`.
  - `members` (array of object)
    - `id` (integer) — Example: `65`.
    - `name` (string) — Example: `Jana Nováková`.
    - `isApp` (boolean) — True for an app (API user). An app cannot be mentioned.

#### 400 — Error response

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `API key is invalid for this company`.
- `errorCode` (string) — One of: `BAD_REQUEST`.
- `data` (null)

#### 403 — Refusal — the API key may not use the chat this way

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `This API key may not read the chat. Allow "Read chat" for th…`.
- `errorCode` (string) — `CHAT_NOT_ALLOWED` — the key lacks "Read chat" / "Write to chat", or it is a key derived from the API key (public, login or temporary), which never reaches the chat. `BAD_REQUEST` — other refusals (origin or IP restrictions, a key of an API user not linked to an application, an endpoint not available for this key type). One of: `CHAT_NOT_ALLOWED`, `BAD_REQUEST`.
- `data` (null)


## POST /chat/get-messages

**Read chat messages**

One page of a channel's messages, oldest first.

**Paging:** by message ID, never by offset. Without a cursor you get the newest page. `beforeMessageId` (the first ID you have) returns the older page; `afterMessageId` (the last ID you have) returns what came since — when more arrived than fit, the messages right after the cursor, so nothing is skipped. `hasOlder` / `hasNewer` say whether more exists.

**Threads:** the page is the channel's main line. A message with replies carries `thread` (count, last reply ID and time) — read the replies with `threadId` set to its ID, with the same cursors. `includeReplies: true` returns every message of the channel, replies included, in one stream; with `afterMessageId` it is the way to follow everything new.

**Edits and deletions** of older messages do not move the cursor; a message carries `editedAt` and `isDeleted`.

Reading never marks anything as read.

### Request body (`application/json`)

- `channelId` (string, required) — Channel ID (10 letters and digits) — `id` from get-channels, or the Channel ID shown in the chat under the channel's ⋯ menu → Info. Example: `K7Q2M9XA1B`.
- `threadId` (integer) — Optional. ID of a main-line message — returns the replies of its thread instead of the main line. Example: `5821`.
- `beforeMessageId` (integer) — Optional. Return the messages older than this ID. Cannot be combined with afterMessageId.
- `afterMessageId` (integer) — Optional. Return the messages newer than this ID. Cannot be combined with beforeMessageId.
- `includeReplies` (boolean) — Optional. Include thread replies — every message of the channel in one stream. Cannot be combined with threadId. Default: `false`.
- `limit` (integer) — Optional. How many messages to return. Minimum `1`, maximum `100`. Default: `50`.

### Responses

#### 200 — OK response

Content type `application/json`:

- `status` (string) — Example: `ok`.
- `statusInfo` (string, nullable) — Example: `null`.
- `errorCode` (string, nullable) — Null on success. `CHANNEL_NOT_FOUND` — the channel is not among the channels this key reaches. `THREAD_NOT_FOUND` — `threadId` is not a main-line message of the channel. `CHAT_NOT_ALLOWED` (HTTP 403) — the key lacks "Read chat", or it is a public, login or temporary key, which never reach the chat. `MISSING_PARAMETER` / `INVALID_PARAMETER` — see statusInfo. Example: `null`.
- `data` (object)
  - `channelId` (string) — Example: `K7Q2M9XA1B`.
  - `threadId` (integer, nullable)
  - `messages` (array of object)
    - `id` (integer) — Message ID. IDs only grow, so they serve as the cursor of get-messages. Example: `5821`.
    - `threadParentId` (integer, nullable) — ID of the main-line message whose thread this reply belongs to; null for a main-line message. Example: `null`.
    - `type` (string) — `notice` is not something the author wrote — see `noticeKind`. Do not treat it as a message. One of: `message`, `notice`.
    - `noticeKind` (string, nullable) — For a notice: `rateLimited` — `noticeCount` messages from that author were not delivered because they exceeded the rate limit; `fieldChanged` — a field value of the thread's main-line message was changed (its `text` says which). Null for a message. One of: `rateLimited`, `fieldChanged`, `null`.
    - `noticeCount` (integer, nullable) — For a rateLimited notice, how many messages were not delivered; null otherwise. Example: `null`.
    - `title` (string, nullable) — In a channel that requires a title on main-line messages (a ticket-style channel): the message title. `text` starts with the number and title too. Null otherwise. Example: `null`.
    - `ticketNumber` (integer, nullable) — In a channel with message numbering: the message's number within the channel (shown as #number). Null otherwise. Example: `null`.
    - `isLocked` (boolean) — True when the message is locked: its author can no longer edit or delete it. Example: `false`.
    - `author` (object)
      - `id` (integer) — Example: `65`.
      - `name` (string) — Example: `Jana Nováková`.
      - `isApp` (boolean) — True for an app (API user), false for a person. Example: `false`.
    - `createdAt` (string) — Example: `2026-09-23 14:05:12`.
    - `editedAt` (string, nullable) — Time of the last edit; null when never edited. Example: `null`.
    - `isDeleted` (boolean) — The author deleted the message but a thread still hangs under it. It keeps its place, with empty text, no reactions and no attachments. Example: `false`.
    - `text` (string) — Plain text of the message. Example: `Objednávka 1024 je odeslaná`.
    - `html` (string) — The message as sanitized HTML, the way the chat shows it. Example: `<p>Objednávka <b>1024</b> je odeslaná</p>`.
    - `source` (string, nullable) — How the message was sent when it was not typed in the chat: `api` through this API, `mcp` through the MCP server. Null when typed in the chat or sent by a table action. One of: `api`, `mcp`, `null`.
    - `clientMessageId` (string, nullable) — The clientMessageId the message was sent with, so a program recognises its own messages when it reads the channel back. The chat sets its own ids too; null for a message sent without one. Example: `3f2b1c9e-0c1d-4b8a-9d2e-6f1a2b3c4d5e`.
    - `mentionedUserIds` (array of integer)
    - `reactions` (array of object)
      - `emoji` (string) — Example: `👍`.
      - `count` (integer) — Example: `2`.
    - `attachments` (array of object) — Files attached to the message — name, type and size only; the content is not available through the API.
      - `name` (string) — Example: `faktura.pdf`.
      - `mimeType` (string) — Example: `application/pdf`.
      - `size` (integer) — Size in bytes. Example: `48213`.
    - `thread` (object, nullable) — Present on a main-line message that has replies; null otherwise. Read the replies with get-messages and `threadId` set to this message's ID.
      - `replyCount` (integer) — Example: `3`.
      - `lastReplyId` (integer) — Example: `5840`.
      - `lastReplyAt` (string) — Example: `2026-09-23 15:30:00`.
  - `hasOlder` (boolean)
  - `hasNewer` (boolean)

#### 400 — Error response

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `API key is invalid for this company`.
- `errorCode` (string) — One of: `BAD_REQUEST`.
- `data` (null)

#### 403 — Refusal — the API key may not use the chat this way

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `This API key may not read the chat. Allow "Read chat" for th…`.
- `errorCode` (string) — `CHAT_NOT_ALLOWED` — the key lacks "Read chat" / "Write to chat", or it is a key derived from the API key (public, login or temporary), which never reaches the chat. `BAD_REQUEST` — other refusals (origin or IP restrictions, a key of an API user not linked to an application, an endpoint not available for this key type). One of: `CHAT_NOT_ALLOWED`, `BAD_REQUEST`.
- `data` (null)


## POST /chat/send-message

**Send a chat message**

Posts a message into a channel as the key's user. Members see it immediately and are notified, like with any message.

**Access:** the key needs "Write to chat" (Settings → API → the key's Permissions; full access does not include the chat), and its user must be allowed to post in the channel — check `canPost` / `canReplyInThreads` from get-channels. A message a person sends through their own key is marked as automated in the chat.

**Content:** `text` (plain text, each line becomes a line) or `html` (formatting: paragraphs, bold, italic, lists, links — sanitized like a message typed in the chat, so anything else does not survive as markup). Mention a channel member with `<@userId>` (IDs from get-channel-members); a mention of anyone else stays plain text, so a message never brings a new person into the channel. A member who has hidden themselves in this company (their chat availability setting) can be mentioned only once they have written or reacted in the channel — the chat itself does not offer them either.

**Files:** attach up to 20 files in `files`, each with its name and base64-encoded content; a message may be files alone, without text. Members download them from the message like any file attached in the chat. The whole request may have at most 100 MB — base64 makes a file about a third larger, so the files together can have about 70 MB; a bigger request is refused with HTTP 413 before it is processed. Files count toward the company's daily upload limit: once it is spent, a message with files is not delivered (`UPLOAD_LIMIT_REACHED`, statusInfo says the limit) and a message without files still goes through.

**Retries:** send your own `clientMessageId` (e.g. a UUID) and repeat it only when retrying the same message — a repeat returns the message already sent instead of posting it twice, and does not store its files again. Using it for a different message (other text, channel, thread or files) is refused with `DUPLICATE_CLIENT_MESSAGE_ID`.

**Rate limit:** one sender may post at most 60 messages per minute into one channel and 200 per minute overall. A message over the limit is not delivered (`RATE_LIMITED`) and the channel shows a notice saying how many messages were dropped.

**Ticket-style channels:** a channel can require a title or filled-in fields on every main-line message. The API cannot supply them, so a send into a channel requiring a title or a required field without a default value is refused with `CHANNEL_REQUIRES_FORM`; a field with a default value takes it. A reply in a thread (`threadParentId`) is still possible.

### Request body (`application/json`)

- `channelId` (string, required) — Channel ID (10 letters and digits) — `id` from get-channels, or the Channel ID shown in the chat under the channel's ⋯ menu → Info. A program writing to one fixed channel can take it from there and needs no lookup call. Example: `K7Q2M9XA1B`.
- `text` (string) — The message as plain text. Send this or html, not both. May be left out when files are attached. Example: `Objednávka 1024 je odeslaná, díky <@65>`.
- `html` (string) — The message as HTML. Send this or text, not both. May be left out when files are attached. Example: `<p>Objednávka <b>1024</b> je odeslaná</p>`.
- `files` (array of object) — Optional. Files attached to the message, at most 20. Channel members download them from the message like any attachment. The whole request may have at most 100 MB, and base64 makes a file about a third larger, so the files together can have about 70 MB. Max items `20`.
  - `fileName` (string, required) — The name the file shows under in the chat, with its extension. Max length `255`. Example: `faktura-1024.pdf`.
  - `base64file` (string, format byte, required) — The file content, base64-encoded, without a "data:…;base64," prefix. An empty file is refused. Example: `JVBERi0xLjQK`.
- `threadParentId` (integer) — Optional. Reply in the thread of this main-line message.
- `clientMessageId` (string) — Optional. Your ID of this message: 1–36 characters from A–Z, a–z, 0–9 and . _ : - (a UUID fits). Repeat it only when retrying the same message, with the same files. Max length `36`. Example: `3f2b1c9e-0c1d-4b8a-9d2e-6f1a2b3c4d5e`.

### Responses

#### 200 — OK response

Content type `application/json`:

- `status` (string) — Example: `ok`.
- `statusInfo` (string, nullable) — Example: `null`.
- `errorCode` (string, nullable) — Null on success. `CHANNEL_NOT_FOUND` — the channel is not among the channels this key reaches. `CHANNEL_ARCHIVED` — nobody can write to an archived channel. `CHANNEL_READ_ONLY` — the key's user may not post there (or may not reply in threads). `THREAD_NOT_FOUND` — `threadParentId` is not a live main-line message of the channel. `DUPLICATE_CLIENT_MESSAGE_ID` — the clientMessageId was used for another message. `RATE_LIMITED` — over the rate limit, not delivered. `UPLOAD_LIMIT_REACHED` — the company's daily upload limit is spent, a message with files is not delivered. `CHANNEL_REQUIRES_FORM` — the channel requires a title or a required field without a default value on main-line messages, which the API cannot fill in. `CHAT_NOT_ALLOWED` (HTTP 403) — the key lacks "Write to chat", or it is a public, login or temporary key, which never reach the chat. `MISSING_PARAMETER` / `INVALID_PARAMETER` — see statusInfo. Example: `null`.
- `data` (object)
  - `message` (object) — One chat message. Dates are `YYYY-MM-DD HH:MM:SS` in the server's local time.
    - `id` (integer) — Message ID. IDs only grow, so they serve as the cursor of get-messages. Example: `5821`.
    - `threadParentId` (integer, nullable) — ID of the main-line message whose thread this reply belongs to; null for a main-line message. Example: `null`.
    - `type` (string) — `notice` is not something the author wrote — see `noticeKind`. Do not treat it as a message. One of: `message`, `notice`.
    - `noticeKind` (string, nullable) — For a notice: `rateLimited` — `noticeCount` messages from that author were not delivered because they exceeded the rate limit; `fieldChanged` — a field value of the thread's main-line message was changed (its `text` says which). Null for a message. One of: `rateLimited`, `fieldChanged`, `null`.
    - `noticeCount` (integer, nullable) — For a rateLimited notice, how many messages were not delivered; null otherwise. Example: `null`.
    - `title` (string, nullable) — In a channel that requires a title on main-line messages (a ticket-style channel): the message title. `text` starts with the number and title too. Null otherwise. Example: `null`.
    - `ticketNumber` (integer, nullable) — In a channel with message numbering: the message's number within the channel (shown as #number). Null otherwise. Example: `null`.
    - `isLocked` (boolean) — True when the message is locked: its author can no longer edit or delete it. Example: `false`.
    - `author` (object)
      - `id` (integer) — Example: `65`.
      - `name` (string) — Example: `Jana Nováková`.
      - `isApp` (boolean) — True for an app (API user), false for a person. Example: `false`.
    - `createdAt` (string) — Example: `2026-09-23 14:05:12`.
    - `editedAt` (string, nullable) — Time of the last edit; null when never edited. Example: `null`.
    - `isDeleted` (boolean) — The author deleted the message but a thread still hangs under it. It keeps its place, with empty text, no reactions and no attachments. Example: `false`.
    - `text` (string) — Plain text of the message. Example: `Objednávka 1024 je odeslaná`.
    - `html` (string) — The message as sanitized HTML, the way the chat shows it. Example: `<p>Objednávka <b>1024</b> je odeslaná</p>`.
    - `source` (string, nullable) — How the message was sent when it was not typed in the chat: `api` through this API, `mcp` through the MCP server. Null when typed in the chat or sent by a table action. One of: `api`, `mcp`, `null`.
    - `clientMessageId` (string, nullable) — The clientMessageId the message was sent with, so a program recognises its own messages when it reads the channel back. The chat sets its own ids too; null for a message sent without one. Example: `3f2b1c9e-0c1d-4b8a-9d2e-6f1a2b3c4d5e`.
    - `mentionedUserIds` (array of integer)
    - `reactions` (array of object)
      - `emoji` (string) — Example: `👍`.
      - `count` (integer) — Example: `2`.
    - `attachments` (array of object) — Files attached to the message — name, type and size only; the content is not available through the API.
      - `name` (string) — Example: `faktura.pdf`.
      - `mimeType` (string) — Example: `application/pdf`.
      - `size` (integer) — Size in bytes. Example: `48213`.
    - `thread` (object, nullable) — Present on a main-line message that has replies; null otherwise. Read the replies with get-messages and `threadId` set to this message's ID.
      - `replyCount` (integer) — Example: `3`.
      - `lastReplyId` (integer) — Example: `5840`.
      - `lastReplyAt` (string) — Example: `2026-09-23 15:30:00`.

#### 400 — Error response

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `API key is invalid for this company`.
- `errorCode` (string) — One of: `BAD_REQUEST`.
- `data` (null)

#### 403 — Refusal — the API key may not use the chat this way

Content type `application/json`:

- `status` (string) — Example: `error`.
- `statusInfo` (string) — Example: `This API key may not read the chat. Allow "Read chat" for th…`.
- `errorCode` (string) — `CHAT_NOT_ALLOWED` — the key lacks "Read chat" / "Write to chat", or it is a key derived from the API key (public, login or temporary), which never reaches the chat. `BAD_REQUEST` — other refusals (origin or IP restrictions, a key of an API user not linked to an application, an endpoint not available for this key type). One of: `CHAT_NOT_ALLOWED`, `BAD_REQUEST`.
- `data` (null)
