---
name: connect-cowork
description: Set up the Ninja Pricer MCP connection in Claude Cowork (Claude Desktop / claude.ai). Walks the user through adding the production deploy as a custom connector — they paste a URL, leave the OAuth fields blank, and Cowork handles the rest via the standard OAuth 2.1 + Dynamic Client Registration flow against the deploy. The user signs in with their existing Microsoft account when prompted. Use whenever a Cowork user mentions setting up Ninja Pricer for the first time, can't see the pricing tools after a recent install, or is troubleshooting a "couldn't connect" error in Cowork.
---

# Connect Ninja Pricer (Cowork)

The Ninja Pricer MCP server speaks OAuth 2.1 + Dynamic Client Registration as of April 2026, so Cowork's "Add custom connector" flow handles everything automatically — the user pastes a URL, signs in with the Microsoft account they already use for Ninja Pricer, and Cowork stores the resulting token. No `np_live_...` token paste required.

## The actual install

Tell the user, in order:

1. In Claude Desktop, switch to the **Cowork** tab.
2. **Connectors → Add custom connector** (left sidebar, then the "+" or "Add" button).
3. Fill in the form:
   - **Name:** `Ninja Pricer`
   - **Remote MCP server URL:** `https://ninjapricer-production.up.railway.app/api/mcp`
   - **Advanced settings:** leave OAuth Client ID and Client Secret **blank**. Cowork registers itself as a client automatically via DCR.
4. Click **Add**. Cowork opens a browser tab to sign you in.
5. Sign in with the **same Microsoft account you use for Ninja Pricer**. Domain allowlist applies, so it must be a real org account.
6. The browser tab redirects back and Cowork shows the connector as connected.
7. Either start a new Cowork session or refresh the connector list. The `ninja-pricer` tools should appear.

Confirm by asking the user to say something like "what products do we price?" — Claude should call `list_products` and reply with the catalog.

## Skill (this plugin)

The connector handles the *tools*. This plugin's skill content (the `ninja-pricer` skill that teaches Claude how to drive the tools) is loaded when the user uploads the `.plugin` file via **Customize → Upload custom plugin**. If they haven't done that yet:

- Latest build: `https://github.com/NinjaBoldry/ninja-pricer-plugin/blob/main/dist/ninja-pricer.plugin`
- Cowork → **Customize → Upload custom plugin** → drop the file in.
- Restart the Cowork session.

The plugin upload and the connector setup are independent — connector gives Cowork the tools, plugin gives Claude the knowledge of how to use them. Both are needed.

## Role inheritance

The token Cowork receives from the OAuth flow inherits the role of the user who signed in:

- Microsoft user with **ADMIN** in NinjaPricer → all 63 tools (catalog edits, commissions, rails, etc.)
- Microsoft user with **SALES** → 16 tools (workups, scenarios, quotes; no catalog/admin writes)

If a user's role changes in the web UI, the change takes effect immediately on the next MCP request — no token rotation, no reconnect.

## Token rotation (automatic)

OAuth-issued access tokens expire every hour. Cowork automatically refreshes them in the background using a refresh token (good for 30 days). The user does not need to do anything. If they're idle for more than 30 days, they'll be prompted to sign in with Microsoft again on the next use — same as any other OAuth-backed connector.

## Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| "Couldn't connect" right after clicking Add | Likely a transient deploy/network blip | Try Add again. If persistent, check the deploy's status page. |
| Browser opens to Microsoft sign-in but bounces with "access denied" | Email domain not on the allowlist (`ALLOWED_EMAIL_DOMAIN`) | Confirm the user is signing in with their org account, not personal Microsoft |
| Sign-in succeeds but Cowork shows the connector errored | Race condition between callback and connector save — known intermittent | Remove the connector, add it again. Should work the second time. |
| Tools don't appear in a fresh Cowork session | Connector saved but session started before it finished registering | New Cowork session; the connector applies on next session |
| Tools appear but Claude doesn't seem to know how to use them | Plugin not installed (only the connector is) | Upload `ninja-pricer.plugin` via **Customize → Upload custom plugin** |
| Old `np_live_...` token also configured somewhere | Both auth paths active is fine — server accepts either | No action needed; OAuth is preferred but `np_live_*` tokens still work |

## Token revocation (if needed)

To invalidate an OAuth-issued connection (e.g., user leaves the company, suspected token leak):

- Delete the user's row in NinjaPricer (cascades to all their OAuth tokens), or
- The next time we add a `/revoke` endpoint or admin "kill OAuth tokens for user X" UI, do it through that.

For now, deletion via the admin UI is the immediate kill-switch.

## What this skill does NOT do

- **Doesn't handle local Claude Code setup.** That's the `connect` skill — different mechanism (`~/.zshenv` based, paste a static token).
- **Doesn't issue tokens.** OAuth tokens are issued automatically by the deploy on sign-in; static `np_live_...` tokens are issued at `/settings/tokens` for users who want them for direct API use.
- **Doesn't restart Cowork sessions.** When the connector or plugin changes, the user has to start a fresh session for changes to take effect.
- **Doesn't help non-employees.** The Microsoft sign-in is gated by the deploy's email-domain allowlist. External users can't connect even if Cowork lets them try.

## When OAuth is unavailable

If the deploy's OAuth endpoints are down (rare; would also affect the web UI), `np_live_...` static tokens still work as a fallback for direct API use. But Cowork's connector flow will show "couldn't connect" because it can't complete the OAuth dance. Users in that situation should fall back to the local Claude Code path (`connect` skill) until the deploy is healthy.
