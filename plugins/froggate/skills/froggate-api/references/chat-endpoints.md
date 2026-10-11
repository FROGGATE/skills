# Chat endpoints

A program can read and post messages in the company chat — a notifier posting "order shipped" into a channel, a bot answering in threads, an export of a channel. Only **channels** are reachable; direct messages and group conversations never are. The same four operations are available as tools of the MCP server for an AI client acting on the user's account.

**Setup — walk the user through it before writing code** (full click-path guide for the user: `https://client.froggate.cloud/documentation/konfigurace/chat-pres-api.md`):

1. **A key of an API user** (Settings → API → API users) for a program that should post under its own name, or the user's **personal key** for a tool acting as them. An API user's key must be linked to an application (the Application column) — without it every request is refused.
2. In the key's **Permissions**, switch on **Read chat** and/or **Write to chat** — full access alone does not open the chat.
3. **Channel membership** decides where the key reaches: a personal key reaches the user's channels; an API user must be **added to each channel** — in the chat, the channel settings → **Apps**, by someone who manages the channel and may post in it.
4. **The channel ID** — every endpoint takes the channel as `channelId`: 10 letters and digits (e.g. `K7Q2M9XA1B`). For a program that writes to one fixed channel, ask the user to copy it from the chat: the channel's **⋯** menu → **Info** → **Channel ID** at the bottom, with a copy button (every member has it). A program can also take `id` from `get-channels`. It is not the number in the chat's address; that one is refused.

Backend-only: public, login and temporary keys get `403 CHAT_NOT_ALLOWED`.

| Endpoint | Body | Returns |
|---|---|---|
| `POST /chat/get-channels` | — | `channels[]`: `id`, `name`, `isPrivate`, `isShared`, `isArchived`, `canPost`, `canReplyInThreads`, `requiresForm`, `lastMessageId`, `lastMessageAt` |
| `POST /chat/get-channel-members` | `channelId` | `members[]`: `id`, `name`, `isApp` |
| `POST /chat/get-messages` | `channelId`, optional `threadId`, `beforeMessageId` \| `afterMessageId`, `includeReplies`, `limit` (1–100, default 50) | `messages[]` oldest first, `hasOlder`, `hasNewer` |
| `POST /chat/send-message` | `channelId`, `text` \| `html` and/or `files`, optional `threadParentId`, `clientMessageId` | `message` |

- **Cursors, not offsets.** No cursor = newest page; `beforeMessageId` = the page before; `afterMessageId` = what came since (the messages right after it, so nothing is skipped). Message IDs only grow.
- **Threads.** A page is the main line; a message with replies carries `thread` (`replyCount`, `lastReplyId`, `lastReplyAt`) — read the replies with `threadId`. `includeReplies: true` returns everything in one stream.
- **Following new messages (there are no webhooks):** poll `get-channels`, and for each channel whose `lastMessageId` grew past your stored cursor call `get-messages` with `afterMessageId` and `includeReplies: true` until `hasNewer` is false. Edits and deletions of older messages do not move the cursor.
- A message with `type: "notice"` is not a message — `noticeKind` says what it reports: `rateLimited` (`noticeCount` messages from its author were dropped by the rate limit) or `fieldChanged` (a field value of the thread's main-line message was changed). `source` is `api` / `mcp` for messages sent through the API. Reading never marks anything as read.
- A ticket-style channel requires a title or fields on every main-line message; the API cannot fill them in, so sending a main-line message into a channel requiring a title or a required field without a default value answers `CHANNEL_REQUIRES_FORM` (a field with a default takes it; a thread reply still works). `get-channels` flags such a channel with `requiresForm: true`, so check it before sending instead of learning it from the refusal. Such messages carry `title` and `ticketNumber` (when the channel numbers them), and their `text` starts with the number and title; `isLocked` is on every message.
- **Mentions:** `<@userId>` in `text` or `html` (IDs from `get-channel-members`); it resolves only for people in the channel (a member hidden in this company by their chat availability setting only once they took part in the channel), anything else stays plain text.
- **Retries:** send a `clientMessageId` (UUID) and repeat it only for the same message — the repeat returns the already-posted message; the same id with other content or another channel is refused with `DUPLICATE_CLIENT_MESSAGE_ID`, so never derive it from a counter that restarts. Every message carries its `clientMessageId` back (in the send response and in `get-messages`), so a program recognises its own messages when it reads the channel. A person's message sent with their personal key shows as "automaticky" (automated) in the chat.
- **Files:** `files` is a list of up to 20 `{fileName, base64file}` (the name with its extension, the content base64-encoded without a `data:` prefix), sent in the same request as the text; a message may be files alone. The whole request may have at most 100 MB and base64 adds about a third, so the files together can have about 70 MB (a bigger request gets HTTP 413). Files count toward the company's daily upload limit — once it is spent, a message with files answers `UPLOAD_LIMIT_REACHED` and is not delivered. A retry with the same `clientMessageId` must carry the same files.
- **Limits:** 60 messages per minute per channel and 200 per minute per sender; over it the call answers `RATE_LIMITED` and the message is not delivered. Reading returns attachments as name, type and size only; downloading their content is not possible.

Error codes of the chat endpoints: `CHANNEL_NOT_FOUND`, `THREAD_NOT_FOUND`, `CHANNEL_READ_ONLY`, `CHANNEL_ARCHIVED`, `CHANNEL_REQUIRES_FORM`, `DUPLICATE_CLIENT_MESSAGE_ID`, `RATE_LIMITED`, `UPLOAD_LIMIT_REACHED`, `CHAT_NOT_ALLOWED` (HTTP 403 — the key lacks the chat permission, or is a public, login or temporary key), plus the standard `MISSING_PARAMETER` / `INVALID_PARAMETER`.
