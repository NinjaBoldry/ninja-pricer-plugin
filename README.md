# Ninja Pricer plugin

Companion plugin for the [Ninja Pricer](https://ninjapricer-production.up.railway.app) pricing engine. Distribution status by client:

| Client | Status |
|---|---|
| **Claude Code** (terminal) | ✅ Working |
| **Claude Cowork** (Desktop / claude.ai) | ⛔ Blocked on server-side OAuth — see below |

## Claude Code (recommended path today)

```bash
claude plugin marketplace add https://github.com/NinjaBoldry/ninja-pricer-plugin
claude plugin install ninja-pricer
```

Restart Claude Code, then say "set up Ninja Pricer" — the bundled `connect` skill writes your `np_live_...` token to `~/.zshenv` and validates the connection. The `.mcp.json` bundled with the plugin handles MCP server registration automatically.

## Claude Cowork

**Not currently working.** Cowork's "Add custom connector" form has only `Name`, `Remote MCP server URL`, and optional OAuth `Client ID` / `Client Secret` fields — there is no field for a static bearer token. The Ninja Pricer MCP server today only accepts `Authorization: Bearer np_live_...` and does not expose OAuth metadata or Dynamic Client Registration, so Cowork's flow has nowhere to plug in.

The unblock is server-side: ship OAuth 2.1 + DCR on `/api/mcp` per the [MCP authorization spec](https://modelcontextprotocol.io/specification/draft/basic/authorization). When that lands, Cowork users will be able to add this connector with URL alone — no paste, no token management — and the marketplace "Connect" tile will work too.

Until then, **use Claude Code** for any Claude-driven Ninja Pricer workflow. The web UI at `https://ninjapricer-production.up.railway.app` remains available for direct (non-Claude) use.

The bundled `connect-cowork` skill in the plugin exists only to confirm this status to a Cowork user who tries to set it up — it explicitly tells them not to paste anything into the OAuth fields and redirects them to the Claude Code path.

## What's in the plugin

- **`ninja-pricer` skill** — teaches Claude the pricing tool surface (scenarios, bundles, quotes, catalog edits), error decoding, and the sales-vs-admin capability matrix.
- **`connect` skill** — Claude Code setup/repair. Writes the token to `~/.zshenv`.
- **`connect-cowork` skill** — explains the current Cowork-blocked status and routes users to Claude Code. Will be rewritten with real install steps once OAuth ships on the server.
- **`.mcp.json`** — MCP server config consumed by Claude Code. Not used by Cowork.

A pre-built Cowork-shaped `.plugin` zip lives at [`dist/ninja-pricer.plugin`](dist/ninja-pricer.plugin). It's only useful as documentation today — without a working connector, the skill has no tools to call. Keep it around for the day OAuth ships.

## Updates

**Claude Code:**
```bash
claude plugin marketplace update ninja-pricer
claude plugin update ninja-pricer
```
Then quit and relaunch Claude Code.

**Cowork:** N/A until OAuth ships.

## Building the Cowork `.plugin` artifact from source

```bash
cd plugins/ninja-pricer
zip -r ../../dist/ninja-pricer.plugin . -x ".mcp.json" -x "*.DS_Store"
```

`.mcp.json` is excluded from the Cowork build — even when OAuth ships, Cowork will register the server through Connectors, not via plugin-bundled config.
