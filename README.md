# Ninja Pricer plugin

Companion plugin for the [Ninja Pricer](https://ninjapricer-production.up.railway.app) pricing engine. Works with Claude Code and Claude Cowork (Desktop / claude.ai).

| Client | Status |
|---|---|
| **Claude Cowork** (Desktop / claude.ai) | ✅ Working — OAuth, no token paste |
| **Claude Code** (terminal) | ✅ Working — static token via `~/.zshenv` |

## Claude Cowork

1. **Add the MCP server as a custom connector.** Cowork → **Connectors → Add custom connector**:
   - **Name:** `Ninja Pricer`
   - **Remote MCP server URL:** `https://ninjapricer-production.up.railway.app/api/mcp`
   - **Advanced settings:** leave OAuth Client ID and Secret blank — Cowork auto-registers via DCR.
   - Click **Add**.
2. **Sign in with Microsoft** when the browser opens. Use the same org account you use for Ninja Pricer.
3. **Upload the plugin** so Claude knows how to drive the tools. Download [`dist/ninja-pricer.plugin`](dist/ninja-pricer.plugin), then in Cowork → **Customize → Upload custom plugin** → drop the file in.
4. **Restart the Cowork session.** Then ask "what products do we price?" — Claude calls `list_products` and answers.

If anything's off, in any Cowork session say "connect Ninja Pricer in Cowork" — the bundled `connect-cowork` skill walks you through it.

> **No token to manage.** Cowork stores the OAuth token; refreshes happen automatically in the background. Role (admin/sales) is inherited from your Microsoft user every request — change a user's role in the web UI and MCP picks it up immediately.

## Claude Code (terminal)

```bash
claude plugin marketplace add https://github.com/NinjaBoldry/ninja-pricer-plugin
claude plugin install ninja-pricer
```

Restart Claude Code, then say "set up Ninja Pricer" — the bundled `connect` skill writes your `np_live_...` token to `~/.zshenv` and validates the connection. The `.mcp.json` bundled with the plugin handles MCP server registration automatically.

If you'd rather use the OAuth path in Claude Code instead of a static token, you can — just point your `.mcp.json` at `https://ninjapricer-production.up.railway.app/api/mcp` without an `Authorization` header. Claude Code will discover the OAuth metadata and walk you through sign-in, same as Cowork. Most local users prefer the static-token path because it's simpler.

## What's in the plugin

- **`ninja-pricer` skill** — teaches Claude the pricing tool surface (scenarios, bundles, quotes, catalog edits), error decoding, and the sales-vs-admin capability matrix.
- **`connect` skill** — Claude Code setup/repair. Writes a `np_live_...` token to `~/.zshenv`.
- **`connect-cowork` skill** — Cowork setup/repair. Walks the user through Custom Connector + Microsoft sign-in.
- **`.mcp.json`** — MCP server config consumed by Claude Code with the static-token path. Excluded from the Cowork build (Cowork registers the server through Connectors instead).

A pre-built Cowork-shaped `.plugin` zip lives at [`dist/ninja-pricer.plugin`](dist/ninja-pricer.plugin).

## Updates

**Cowork:** download the latest `ninja-pricer.plugin` and re-upload via **Customize → Upload custom plugin**. Cowork replaces the previous version.

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

`.mcp.json` is excluded from the Cowork build because Cowork registers the server through the Connectors UI, not via plugin-bundled config.
