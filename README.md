# Ninja Pricer plugin

Companion plugin for the [Ninja Pricer](https://ninjapricer-production.up.railway.app) pricing engine. Two distribution paths — pick the one that matches how you use Claude.

## Cowork (Claude Desktop / claude.ai)

1. **Add the MCP server as a custom connector.** In the Cowork tab → **Connectors → Add custom connector**:
   - **Server URL:** `https://ninjapricer-production.up.railway.app/api/mcp`
   - **Authorization header:** `Bearer np_live_...` (your personal token from `/settings/tokens` on the deploy)
2. **Install the plugin.** Download [`dist/ninja-pricer.plugin`](dist/ninja-pricer.plugin) (or the latest from [Releases](https://github.com/NinjaBoldry/ninja-pricer-plugin/releases)). In Cowork → **Customize → Upload custom plugin** → drop the file in.
3. **Restart the Cowork session.** Then say "what products do we price?" — Claude calls `list_products` and answers.

If anything goes sideways, in any Cowork session say "connect Ninja Pricer in Cowork" and the bundled `connect-cowork` skill walks you through it.

> **Why two steps?** Cowork doesn't reliably inject env vars into MCP server processes ([claude-code#39125](https://github.com/anthropics/claude-code/issues/39125)), so credentials live in the Connectors UI (where they actually persist) and the plugin ships only the skills.

## Local Claude Code (terminal)

```bash
claude plugin marketplace add https://github.com/NinjaBoldry/ninja-pricer-plugin
claude plugin install ninja-pricer
```

Restart Claude Code, then say "set up Ninja Pricer" — the `connect` skill writes your token to `~/.zshenv` and validates the connection. The `.mcp.json` bundled with the plugin handles the MCP server registration automatically.

## What's in the plugin

- **`ninja-pricer` skill** — teaches Claude the pricing tool surface (scenarios, bundles, quotes, catalog edits), error decoding, and the sales-vs-admin capability matrix.
- **`connect` skill** — local Claude Code setup/repair. Writes the token to `~/.zshenv`.
- **`connect-cowork` skill** — Cowork setup/repair. Walks the user through the Connectors UI.
- **`.mcp.json`** — MCP server config used by Claude Code only. Excluded from the Cowork `.plugin` build because Cowork wires up the server through Connectors.

## Updates

**Cowork:** download the latest `ninja-pricer.plugin` from this repo and re-upload via **Customize → Upload custom plugin**. Cowork replaces the previous version.

**Claude Code:**
```bash
claude plugin marketplace update ninja-pricer
claude plugin update ninja-pricer
```
Then quit and relaunch Claude Code.

## Building the Cowork `.plugin` artifact from source

```bash
cd plugins/ninja-pricer
zip -r ../../dist/ninja-pricer.plugin . -x ".mcp.json" -x "*.DS_Store"
```

`.mcp.json` is intentionally excluded — Cowork users bring their own credentials via the Connectors UI.
