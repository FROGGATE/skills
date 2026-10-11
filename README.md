# FROGGATE skills for AI agents

Teach your AI agent to build apps and websites on [FROGGATE](https://froggate.cloud) — you don't have to be a developer, you say what you want to achieve. The agent knows the building blocks (sign-in and registration, shared links, public forms, website content from your sections, apps inside FROGGATE, automation, chat), sets each one up, chooses the right API key, writes the code and publishes the app straight into FROGGATE.

Install it as a plugin — that is what keeps it up to date.

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

Then tell the user the skill loads from the next session. If the user works in claude.ai, the desktop app or Cowork, you cannot install it there — give them the steps from the Claude section above.

## Connect the FROGGATE MCP server too

Building on FROGGATE takes two things: this plugin, which guides the code, and the FROGGATE MCP server, through which the agent works inside your account — it creates the sections, columns and statuses the app needs and reads and writes records while it builds. Without it the agent can't set anything up. You connect it with an API key from **Settings → API** in FROGGATE:

- **claude.ai, the Claude app and Cowork** — add it yourself as a custom connector with the key in a request header. Where Claude doesn't offer request headers for your account, it can't be connected there; use a tool on your computer instead.
- **Claude Code, Codex and other tools on your computer** — ask the agent to connect it; the skill tells it how, and you add the key to its configuration yourself, outside the conversation.

Step-by-step guide (in Czech): https://client.froggate.cloud/documentation/konfigurace/vyvoj-s-ai.md

## Versions

`VERSION` holds the current version. The skill checks it once per conversation and, when a newer one exists, updates itself through the tool that installed it (or tells you how).
