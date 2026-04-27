---
name: connect-cowork
description: Set up the Ninja Pricer MCP connection for a Claude Cowork user — walks them through adding the production deploy as a custom connector in Cowork's UI with their `np_live_...` token. Use whenever a Cowork user mentions setting up Ninja Pricer for the first time, rotating their token, or when the existing `ninja-pricer` skill can't reach the MCP server in a Cowork session (tools missing, "no such tool" errors, `-32001 Unauthorized`). For local Claude Code users use the `connect` skill instead — that one writes to `~/.zshenv`, which is meaningless in Cowork's ephemeral VM environment.
---

# Connect Ninja Pricer (Cowork)

Walks a Claude Cowork user through wiring up the `ninja-pricer` MCP server as a custom connector. Cowork doesn't run MCP servers locally — it reaches out from Anthropic's cloud — so the bundled `.mcp.json` env-var substitution path used by Claude Code (`${NINJA_PRICER_TOKEN}` from a shell file) doesn't apply. Cowork users add the server through the **Connectors** UI instead.

This is the Cowork-only path. If the user is on local Claude Code, point them at the `connect` skill. The two are mutually exclusive.

## What's needed

The Ninja Pricer MCP server is a public HTTP endpoint that authenticates with a static bearer token:

- **Server URL:** `https://ninjapricer-production.up.railway.app/api/mcp`
- **Auth header:** `Authorization: Bearer np_live_...` (the user's own token)

Cowork's custom-connector UI accepts a remote MCP URL plus a bearer token, which is exactly the shape this server expects. No OAuth required on the user's side.

## Decision flow

```
1. Does the user have a token already? → if not, send them to /settings/tokens.
2. Validate the token against the deploy.
3. Walk them through Cowork's Connectors UI step by step.
4. Verify the connector is healthy (tools/list shows ninja-pricer tools).
5. If the plugin's skill isn't loading, point them at Customize → Browse plugins or upload.
```

## Step 1 — confirm they have a token

Ask:

> Do you have your `np_live_...` Ninja Pricer token? You can grab one (or rotate the old one) at `https://ninjapricer-production.up.railway.app/settings/tokens` — click "New token" and copy the value. It's shown exactly once.

If they don't have one, wait until they do. Don't continue without it.

## Step 2 — validate the token before they paste it into Cowork

Cowork's connector UI doesn't surface a clear error if a token is bad — it'll just show "couldn't connect". Validate up front so we catch typos and revoked tokens before they go into Connectors.

If you have shell access in this environment (Cowork code-execution sandbox or local terminal), run:

```bash
curl -sS -o /dev/null -w "%{http_code}\n" \
  -X POST https://ninjapricer-production.up.railway.app/api/mcp \
  -H "Authorization: Bearer np_live_..." \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"connect-cowork","version":"1"}}}'
```

- **200** → token valid. Continue.
- **401** → revoked or typo. Send them back to `/settings/tokens` for a fresh one. Don't continue.
- **other** → deploy issue, not a token problem. Tell the user to ping their admin.

If you don't have shell access, skip validation and warn the user that if Cowork shows "couldn't connect" after the next steps, the token is the most likely culprit.

## Step 3 — walk them through the Connectors UI

Tell the user, in order:

1. In the Claude Desktop app, open the **Cowork** tab.
2. In the left sidebar, click **Connectors** (or **Customize → Connectors** depending on UI version).
3. Click **Add custom connector** (sometimes labeled "Add connector" or "+").
4. In the form:
   - **Name:** `Ninja Pricer` (or whatever they want — display only)
   - **Server URL:** `https://ninjapricer-production.up.railway.app/api/mcp`
   - **Authentication:** select bearer-token / authorization-token style. Paste their token (`np_live_...`). If the UI asks for a header name explicitly, it's `Authorization`, value is `Bearer np_live_...`.
5. Save.

If the UI insists on OAuth (Client ID/Secret) and won't accept a bare bearer token, the user is on a Cowork build where custom connectors are OAuth-only. In that case, fall back to the section "When custom connectors require OAuth" below.

## Step 4 — verify the connector is healthy

Once saved, the connector should report a "connected" or green-dot status. Have the user open a fresh Cowork session (the connector becomes available on next session), and ask Claude something like:

> What products do we price?

If the `ninja-pricer` skill is installed, Claude calls `list_products` and answers with the seeded products (Ninja Notes, Training & White-glove, Service). If you get tools-not-found errors, the connector isn't connected yet — refresh the Connectors panel or restart the session.

## Step 5 — if the skill isn't loading

The connector handles the *tools*. The plugin handles the *skill* that teaches Claude how to drive those tools. They're separate in Cowork.

If tools are present but Claude doesn't seem to know how to use them, the plugin isn't installed. Tell the user:

1. **Customize → Browse plugins** — if Ninja Pricer is listed, click Install.
2. **Customize → Upload custom plugin** — if they have a `ninja-pricer.plugin` file (you can fetch the latest from `https://github.com/NinjaBoldry/ninja-pricer-plugin/releases/latest`), drag it in.

After install, restart the Cowork session.

## Step 6 — token rotation

When a token is rotated:

1. Issue a new one at `/settings/tokens` (and revoke the old one).
2. In Cowork: **Connectors → Ninja Pricer → Edit** → paste the new token → save.
3. Restart the Cowork session so the connector picks up the new value.

No plugin reinstall needed — the plugin doesn't carry credentials.

## When custom connectors require OAuth

Some Cowork builds gate custom connectors behind OAuth Client ID/Secret only. The Ninja Pricer server doesn't currently expose OAuth endpoints (no `/.well-known/oauth-authorization-server`, no Dynamic Client Registration). If the user's Cowork build forces OAuth on custom connectors, this skill cannot complete the install.

Tell the user honestly:

> Your Cowork build appears to require OAuth on custom connectors, but the Ninja Pricer MCP server only supports static bearer tokens right now. Two options:
>
> 1. Use the local Claude Code path instead — install the plugin from `https://github.com/NinjaBoldry/ninja-pricer-plugin` and follow the `connect` skill there.
> 2. Wait for OAuth support on the server — that's a known gap and on the roadmap.

Don't try to fake an OAuth flow or paste the bearer into the Client Secret field — that won't work and may leave a half-broken connector entry.

## What this skill does NOT do

- **Doesn't manage the Cowork session itself.** Connector edits often require a session restart to take effect; the skill can't restart a Cowork session.
- **Doesn't issue tokens.** Token issuance is at `/settings/tokens` on the deploy.
- **Doesn't handle local Claude Code setup.** That's the `connect` skill's job — `~/.zshenv` based, totally different mechanism.
- **Doesn't bypass the env-var injection bug.** Cowork's plugin `.mcp.json` env-var substitution is unreliable as of April 2026 ([claude-code#39125](https://github.com/anthropics/claude-code/issues/39125)). That's exactly why we route credentials through Connectors instead of bundling them in the plugin.

## Common gotchas

| Symptom | Likely cause | Action |
|---|---|---|
| "Couldn't connect" right after saving connector | Token typo, or token revoked | Re-validate via curl (step 2). Re-paste. |
| Connector saved but tools don't show in session | Session needs restart | Quit and reopen the Cowork session |
| `-32001 Unauthorized` mid-session | Token revoked while user was active | Issue a new token, edit the connector, restart session |
| Tools work but Claude doesn't know how to use them | Skill not installed | Install the `ninja-pricer.plugin` file (step 5) |
| Cowork connector form only shows OAuth fields | Build forces OAuth on custom connectors | Fall back to Claude Code, or wait for server-side OAuth |
