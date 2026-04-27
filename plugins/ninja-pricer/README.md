# Ninja Pricer plugin

Claude Code plugin that bundles:

- **MCP server config** — points at the live Ninja Pricer deploy (`https://ninjapricer-production.up.railway.app/api/mcp`) with your personal API token.
- **`ninja-pricer` skill** — teaches Claude how to use the tool surface, when to prefer `compute_quote` vs `generate_quote`, how role-gating works, and how to translate MCP error codes.
- **`connect` skill** — guided setup/repair flow. If your token isn't reaching the MCP subprocess, ask Claude to "connect Ninja Pricer" and it'll diagnose, validate the token against the deploy, and install it in the right shell file.

## Install (first time)

1. **Get a token.** Sign in to `https://ninjapricer-production.up.railway.app`, visit `/settings/tokens`, click "New token". Copy the raw `np_live_...` value — it's shown exactly once.

2. **Export the token where Claude Code can see it.** Add to `~/.zshenv` (zsh) or `~/.bash_profile` (bash):

   ```bash
   export NINJA_PRICER_TOKEN=np_live_YourTokenHere
   ```

   > **Why `.zshenv` and not `.zshrc`?** Claude Code launches the MCP server in a non-interactive subshell. `.zshrc` only loads for interactive shells, so a token there is invisible to the plugin and you'll see "Failed to connect" in `claude mcp list`. `.zshenv` loads for every zsh invocation, interactive or not.

   Open a fresh terminal (or `source ~/.zshenv`) and verify:

   ```bash
   echo "${NINJA_PRICER_TOKEN:0:9}…"   # should print "np_live_…"
   ```

3. **Add the marketplace.** One-time per machine:

   ```bash
   claude plugin marketplace add https://github.com/NinjaBoldry/ninja-pricer-plugin
   ```

4. **Install the plugin.**

   ```bash
   claude plugin install ninja-pricer
   ```

5. **Restart Claude Code.** The MCP server connects on the next session; the skill loads automatically when your conversation touches pricing.

> **Stuck on any of steps 2–5?** In a Claude Code session, just say "connect Ninja Pricer" — the bundled `connect` skill will diagnose your shell, validate your token, write it to the right file, and tell you exactly what to restart.

## Verify

In a fresh Claude Code session:

```
> What products do we price?
```

Claude should invoke the skill, call `list_products`, and report back with the seeded products (Ninja Notes, Training & White-glove, Service).

If nothing happens:

- `claude mcp list` — confirm `ninja-pricer` is connected (not "Failed to connect"). If it fails to connect, the token isn't reaching the MCP subprocess.
- `echo $NINJA_PRICER_TOKEN` in a *non-interactive* shell — `zsh -c 'echo $NINJA_PRICER_TOKEN'` should print the token. If it's empty, the export is in `.zshrc` instead of `.zshenv`. Move it.
- Restart Claude Code after changing env vars — the MCP subprocess only reads the env at launch.
- Hit `/settings/tokens` on the deploy — confirm your token isn't revoked and you're still an active user.

## Token rotation

Revoke + reissue any time via `/settings/tokens`. Update `NINJA_PRICER_TOKEN` in `~/.zshenv`, open a fresh terminal so the new value is exported, then restart Claude Code. No plugin reinstall needed.

## Role inheritance

Your token inherits the role of the user who issued it. Admin in the web UI = admin via MCP (all 63 tools). Sales = sales (16 tools). If your role changes, the next MCP request reflects the new role — no token rotation required.
