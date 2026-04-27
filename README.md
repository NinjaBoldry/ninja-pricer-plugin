# Ninja Pricer plugin

Public marketplace for the [Ninja Pricer](https://ninjapricer-production.up.railway.app) Claude Code plugin. The plugin itself lives in [`plugins/ninja-pricer/`](plugins/ninja-pricer) — see the [plugin README](plugins/ninja-pricer/README.md) for setup, token rotation, and troubleshooting.

## Quick install

```bash
claude plugin marketplace add https://github.com/NinjaBoldry/ninja-pricer-plugin
claude plugin install ninja-pricer
```

Restart Claude Code, then in any session say "set up Ninja Pricer" — the bundled `connect` skill walks you through getting your `np_live_...` token in the right place.

## What's in the plugin

- **MCP server config** — connects to `https://ninjapricer-production.up.railway.app/api/mcp` with your personal token.
- **`ninja-pricer` skill** — teaches Claude the pricing tool surface (scenarios, bundles, quotes, catalog edits), error decoding, and the sales-vs-admin capability matrix.
- **`connect` skill** — guided setup/repair flow. Diagnoses your shell, validates your token against the deploy, installs it in the right shell file (`~/.zshenv`, not `~/.zshrc`).

## Updates

When new versions ship:

```bash
claude plugin marketplace update ninja-pricer
claude plugin update ninja-pricer
```

Then quit and relaunch Claude Code.
