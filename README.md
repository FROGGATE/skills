# FROGGATE for AI agents

Connect your AI agent to [FROGGATE](https://froggate.cloud) and let it work for you:

- **In your account** — it finds and updates records, sets up sections, columns and statuses, and uses your chat when you allow it. It acts as you, with your permissions.
- **On top of it** — it builds apps and websites on FROGGATE. You don't have to be a developer, you say what you want to achieve. The agent knows the building blocks (sign-in and registration, shared links, public forms, website content from your sections, apps inside FROGGATE, automation, chat), sets each one up, chooses the right API key, writes the code and prepares the app for publishing in FROGGATE — what only you can set there (the app's API key, publishing, section actions), it tells you step by step.

Install it as a plugin — it brings the connection to your account along and keeps itself up to date.

## Install

### Claude — claude.ai, the desktop app, Cowork and Claude Code

In claude.ai or the Claude desktop app open **Customize → Plugins → Add → Add marketplace**, enter `FROGGATE/skills`, add the **froggate** plugin and turn on **Sync automatically**. The plugin is saved to your account, so it is available in chat, in Cowork and in Claude Code signed in to the same account, and new versions arrive on their own.

### Claude Code only

```
claude plugin install froggate --marketplace FROGGATE/skills
```

Then turn on automatic updates: `/plugin` → Marketplaces → froggate → Enable auto-update.

### Codex

```
codex plugin marketplace add FROGGATE/skills
codex plugin add froggate@froggate
```

Updates: `codex plugin marketplace upgrade froggate && codex plugin add froggate@froggate` — the skill runs it for you when it sees a newer version.

### Other agents (Cursor, Gemini CLI, OpenCode, …)

```
npx skills add FROGGATE/skills -g
```

Updates: `npx skills update` — the skill runs it for you when it sees a newer version.

## For an AI agent asked to install this

Install the plugin for the tool you are running in, from a shell:

- **Claude Code:** `claude plugin install froggate --marketplace FROGGATE/skills`, then turn on automatic updates by adding `"autoUpdate": true` to the `froggate` entry under `extraKnownMarketplaces` in `~/.claude/settings.json` (the install has just written that entry).
- **Codex:** `codex plugin marketplace add FROGGATE/skills` and `codex plugin add froggate@froggate`.
- **Anything else:** `npx skills add FROGGATE/skills -g`.

Then tell the user the skill loads from the next session and that the FROGGATE MCP server it brings waits for their sign-in: in Claude Code `/mcp` → `froggate` → Authenticate, in Codex `codex mcp login froggate` (see "Sign in to your FROGGATE account" below). If the user works in claude.ai, the desktop app or Cowork, you cannot install it there — give them the steps from the Claude section above.

## Sign in to your FROGGATE account

The agent works inside your account through the FROGGATE MCP server, which the plugin brings along — its address is `https://mcp.froggate.cloud`. For everyday work it is all you need. Building takes it too: the agent creates the sections, columns and statuses the app needs and reads and writes records while it builds — without it the agent can't set anything up. You connect it by signing in with your FROGGATE account; on the sign-in page you choose the company, how long the connection lasts and whether it may use your chat. The agent then works as you, with your permissions:

- **claude.ai, the Claude app and Cowork** — in the plugin's **Connectors** tab choose Add and Connect. Without the plugin, add a custom connector with that address; the sign-in follows on first use.
- **Claude Code** — run `/mcp`, pick `froggate` and Authenticate.
- **ChatGPT and other tools with custom MCP servers** — add the address as a custom MCP server; the sign-in is the same.
- **Codex** — the plugin asks you to sign in when you install it; later `codex mcp login froggate`.
- **Other tools on your computer** — ask the agent to connect it; the skill tells it how.

One connection works in one company. For another company, add the server again with the company's ID at the end of the address. Connecting needs the permission **Connecting AI tools** in FROGGATE; the sign-in page tells you whom to ask when you don't have it.

Step-by-step guide (in Czech): https://client.froggate.cloud/documentation/pouzivani/ai-nastroje.md

## Versions

`VERSION` holds the current version. The skill checks it once per conversation and, when a newer one exists, updates itself through the tool that installed it (or tells you how).
