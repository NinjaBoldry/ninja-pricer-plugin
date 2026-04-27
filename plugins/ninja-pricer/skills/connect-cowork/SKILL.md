---
name: connect-cowork
description: Set expectations for a Claude Cowork user trying to connect to the Ninja Pricer MCP server, and redirect them to a path that actually works today. Use whenever a Cowork user mentions setting up Ninja Pricer, hits "couldn't connect" on the custom-connector flow, or asks why the marketplace "Connect" button errors out. As of April 2026 Cowork's custom-connector form has no field for static bearer tokens — only Name, URL, and optional OAuth Client ID/Secret — and the Ninja Pricer MCP server doesn't yet implement OAuth, so there's no working Cowork install path right now. Recommend Claude Code instead and flag OAuth support as the unblock.
---

# Connect Ninja Pricer (Cowork) — current status

**As of April 2026, there is no working Cowork install for Ninja Pricer.** This skill exists to be honest about that, save users the dead-end clicking, and route them to Claude Code, which does work today.

## Why it doesn't work in Cowork yet

Cowork's **Add custom connector** form has three fields:

- Name
- Remote MCP server URL
- Advanced settings → OAuth Client ID (optional) + OAuth Client Secret (optional)

There is no field to paste a static bearer token. The connector flow expects the MCP server to either:

- Be reachable anonymously, or
- Speak OAuth 2.1 with Dynamic Client Registration per the MCP authorization spec — Cowork performs DCR and walks the user through an in-browser sign-in.

The Ninja Pricer MCP server (`https://ninjapricer-production.up.railway.app/api/mcp`) currently only accepts `Authorization: Bearer np_live_...` static tokens. It does not expose OAuth metadata or DCR endpoints. So:

- Pasting just the URL → Cowork hits the server anonymously → 401 → Cowork tries OAuth discovery → no endpoints → "couldn't connect".
- Pasting OAuth Client ID/Secret → there's no IdP issuing those for this server → still fails.

Putting the bearer in a hidden field would help, but no such field exists.

## What to tell the user

Be direct:

> Cowork's custom-connector UI doesn't have a bearer-token field, and the Ninja Pricer MCP server doesn't speak OAuth yet — so there isn't a working Cowork install today. OAuth on the server is on the roadmap; once that's shipped, the Cowork "Add custom connector" flow will Just Work with this URL.
>
> In the meantime: **install via Claude Code instead.** That path is fully working — `claude plugin marketplace add https://github.com/NinjaBoldry/ninja-pricer-plugin` then `claude plugin install ninja-pricer`, and once you're in a Claude Code session say "set up Ninja Pricer" and the bundled `connect` skill walks you through the rest.

Don't have them paste anything into the Cowork form. Don't have them try the OAuth fields. Don't tell them to wait — give them the working alternative.

## What if they really need Cowork specifically

Some users genuinely can't or won't use Claude Code (no terminal access, IT policy, work primarily in Claude.ai web). For those users:

- The plugin's **skill content** can still be uploaded to Cowork as a `.plugin` file (`Customize → Upload custom plugin`, [latest build here](https://github.com/NinjaBoldry/ninja-pricer-plugin/blob/main/dist/ninja-pricer.plugin)). Claude will know *how* to use Ninja Pricer tools — but without a working connector, there are no tools to call. The skill becomes documentation Claude can read, not a working integration.
- They can still use the Ninja Pricer **web UI** (`https://ninjapricer-production.up.railway.app`) directly — the MCP server is just a Claude-driven facade over the same data. For workups and quotes, the web UI is the canonical path.

Tell them honestly that the integrated Cowork experience is blocked on OAuth shipping.

## How they'll know when this changes

When OAuth ships on the server, this skill should be rewritten with the actual install steps, and the README's Cowork section will be updated. Watch the repo:

`https://github.com/NinjaBoldry/ninja-pricer-plugin`

Or ping the admin who owns the deploy.

## Don't do these things

- **Don't tell the user to paste their `np_live_...` token into the OAuth Client Secret field.** It won't work, and OAuth secrets get logged differently than headers — could leak.
- **Don't fabricate a "hidden" bearer-token path that doesn't exist in the UI.** If the user pushes back, show them the screenshot of the form fields and confirm.
- **Don't suggest editing Cowork config files manually.** Cowork's connector store isn't a user-facing config file; sessions are ephemeral and cloud-managed.

## When OAuth ships (future state — do not act on this yet)

When the deploy adds the four endpoints — `/.well-known/oauth-protected-resource`, `/.well-known/oauth-authorization-server`, `/register` (DCR), `/authorize`, `/token` — and starts emitting `WWW-Authenticate: Bearer resource_metadata="..."` on 401s, Cowork's existing flow will handle everything automatically. The user pastes:

- **Name:** Ninja Pricer
- **Server URL:** `https://ninjapricer-production.up.railway.app/api/mcp`
- (OAuth fields blank — DCR populates them)

Cowork registers itself as a client, opens a browser tab to the deploy's `/authorize` endpoint, the user signs in with their normal NinjaPricer credentials, consents, and Cowork stores the access token. No paste, no env var, no shell config. That's the target state.

This skill should be rewritten once the server side is live.
