---
name: connect
description: Set up or repair the Ninja Pricer MCP connection for a Claude Code user — collect their `np_live_...` token, validate it against the deploy, write it to `~/.zshenv` so non-interactive shells inherit it, and migrate it out of `~/.zshrc` if that's where it ended up. Use whenever the user mentions setting up Ninja Pricer for the first time, rotating their token, or when the existing `ninja-pricer` skill can't reach the MCP server (tools missing, "Failed to connect" in `claude mcp list`, `-32001 Unauthorized` errors). NOT for fixing the claude.ai "Connect" button — that needs server-side OAuth, not a token paste.
---

# Connect Ninja Pricer

Walks a Claude Code user through getting the `ninja-pricer` MCP plugin connected. The plugin's `.mcp.json` references `${NINJA_PRICER_TOKEN}`, and the most common failure mode is that the env var isn't reaching Claude Code's MCP subprocess.

This skill is for the **Claude Code** install path. It does not help the claude.ai / Claude Desktop "Connect" button — that flow needs OAuth 2.1 + DCR on the server, which a skill can't add. If the user is asking about the marketplace Connect button, tell them to use the Claude Code plugin instead, or to add a custom connector with the token pasted manually.

## Decision flow

```
1. Already connected? → confirm and stop.
2. Token in env but invalid? → rotate.
3. Token missing or in wrong file? → collect, validate, install.
4. Restart instruction → always end here. Skills can't restart Claude Code.
```

## Step 1 — diagnose current state

Before asking the user for anything, find out where they are. Run these in one Bash call:

```bash
echo "current shell: $SHELL"
echo "interactive token len: ${#NINJA_PRICER_TOKEN}"
echo "non-interactive token len:"
zsh -c 'echo ${#NINJA_PRICER_TOKEN}' 2>/dev/null
echo "in zshenv:"
grep -c '^export NINJA_PRICER_TOKEN=' ~/.zshenv 2>/dev/null || echo 0
echo "in zshrc:"
grep -c '^export NINJA_PRICER_TOKEN=' ~/.zshrc 2>/dev/null || echo 0
echo "in bash_profile:"
grep -c '^export NINJA_PRICER_TOKEN=' ~/.bash_profile 2>/dev/null || echo 0
echo "claude mcp status:"
claude mcp list 2>&1 | grep -i ninja-pricer || echo "(not registered)"
```

Interpret:

- **non-interactive token len > 0 AND `claude mcp list` shows connected** → already good. Confirm with the user, offer to test (`list_products` via the existing `ninja-pricer` skill), and stop.
- **non-interactive token len = 0 BUT zshrc has the export** → token is in the wrong file. Skip to "Migrate from zshrc" without asking again.
- **non-interactive token len = 0 AND nothing anywhere** → first-time setup. Ask the user for a token.
- **non-interactive token len > 0 BUT `claude mcp list` says "Failed to connect"** → token is probably revoked/expired. Validate, then rotate.

## Step 2 — collect the token (only if needed)

Ask the user:

> Paste your `np_live_...` token. If you don't have one, sign in at `https://ninjapricer-production.up.railway.app/settings/tokens`, click "New token", and copy the value (it's shown exactly once).

Validate format before saving — must match `^np_live_[A-Za-z0-9]{20,}$`. If it doesn't, tell the user what you got and ask them to paste again. Don't save a malformed token.

## Step 3 — validate against the deploy

Always test the token before persisting it. This catches typos, revoked tokens, and wrong environments early.

```bash
TOKEN='np_live_...'    # the exact string the user pasted, single-quoted to avoid shell expansion
curl -sS -o /dev/null -w "%{http_code}\n" \
  -X POST https://ninjapricer-production.up.railway.app/api/mcp \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"connect-skill","version":"1"}}}'
```

- **200** → token is good. Continue.
- **401** → token is invalid or revoked. Tell the user, ask them to issue a new one, restart at step 2.
- **other** → something is wrong with the deploy. Don't persist. Tell the user to check the URL or contact their admin.

## Step 4 — install the export

Append to `~/.zshenv` (zsh) and `~/.bash_profile` (bash). Both are cheap; covers the user regardless of which shell they ended up in. Use a sentinel comment so you can find and replace cleanly on rotations.

**For first-time install** (no existing line):

```bash
TOKEN='np_live_...'    # validated in step 3
URL="https://ninjapricer-production.up.railway.app"

for f in ~/.zshenv ~/.bash_profile; do
  touch "$f"
  if ! grep -q '^export NINJA_PRICER_TOKEN=' "$f"; then
    {
      echo ""
      echo "# Ninja Pricer — required by Claude Code MCP plugin (must be in zshenv/bash_profile, not zshrc, to load for non-interactive shells)"
      echo "export PRICER_APP_URL=\"$URL\""
      echo "export NINJA_PRICER_TOKEN=$TOKEN"
    } >> "$f"
    echo "appended to $f"
  fi
done
```

**For rotation** (line already exists): replace in place. Do NOT use `sed -i ''` (BSD vs GNU portability). Use a small awk/python rewrite or `sed -i.bak` and clean up the backup.

```bash
TOKEN='np_live_...'
for f in ~/.zshenv ~/.bash_profile; do
  [ -f "$f" ] || continue
  if grep -q '^export NINJA_PRICER_TOKEN=' "$f"; then
    /usr/bin/sed -i.bak "s|^export NINJA_PRICER_TOKEN=.*|export NINJA_PRICER_TOKEN=$TOKEN|" "$f"
    rm -f "$f.bak"
    echo "rotated in $f"
  fi
done
```

## Step 5 — migrate from zshrc (if found there)

If step 1 found the export in `~/.zshrc`, the user's interactive shell sees the token but Claude Code's MCP subprocess does not. Move it.

```bash
# Show the user the line you're about to remove first
grep -n 'NINJA_PRICER_TOKEN\|PRICER_APP_URL' ~/.zshrc
```

Tell the user: "I found these lines in `~/.zshrc`. Claude Code's MCP subprocess can't read them there. I'll move them to `~/.zshenv`. OK?"

After they confirm:

```bash
/usr/bin/sed -i.bak '/^export NINJA_PRICER_TOKEN=/d; /^export PRICER_APP_URL=/d; /^# Ninja Pricer/d' ~/.zshrc
rm -f ~/.zshrc.bak
```

Then run step 4 to install in `~/.zshenv` if not already there.

## Step 6 — verify the env var is now visible non-interactively

The whole point of this skill. Confirm before declaring victory:

```bash
zsh -c 'echo "non-interactive token len: ${#NINJA_PRICER_TOKEN}"'
```

If length is 0 here, something is wrong — `~/.zshenv` exists but isn't being read. Check for a `.zprofile` or shell config that's overriding the var, and surface that to the user. Don't tell them to restart Claude Code if you can't prove the env var would be inherited.

## Step 7 — tell the user to restart Claude Code

Skills can't do this for the user. Be explicit:

> Done. The token is in `~/.zshenv` and validated against the deploy. Last step is on you:
>
> 1. Quit Claude Code completely (⌘Q on macOS — closing the window isn't enough; the MCP subprocess only re-reads env on full relaunch).
> 2. Reopen Claude Code.
> 3. In the new session, ask me "what products do we price?" — I should call `list_products` and answer directly. If `claude mcp list` still shows "Failed to connect" after restart, come back and we'll dig in.

## What this skill does NOT do

- **Doesn't make the claude.ai marketplace "Connect" button work.** That requires OAuth 2.1 + Dynamic Client Registration on `/api/mcp`. If the user wants claude.ai/Desktop access, point them at: (a) installing Claude Code with this plugin, or (b) using claude.ai's "Add custom connector" flow with the URL `https://ninjapricer-production.up.railway.app/api/mcp` and `Authorization: Bearer <their-token>` pasted manually.
- **Doesn't manage the plugin install itself.** If `claude mcp list` doesn't show `ninja-pricer` at all, the plugin isn't installed. Run `claude plugin marketplace add https://github.com/NinjaBoldry/NinjaPricer` then `claude plugin install ninja-pricer` — or point the user at the plugin README.
- **Doesn't issue tokens.** Token issuance happens at `/settings/tokens` on the deploy. The skill can only persist a token the user already has.

## Errors you might hit

| Symptom | Likely cause | Action |
|---|---|---|
| `curl: command not found` in shell | Sandboxed PATH | Prefix `PATH=/usr/bin:/bin` or call `/usr/bin/curl` directly |
| `sed: -i: requires an argument` | GNU sed was called with BSD syntax (or vice versa) | Use `/usr/bin/sed -i.bak` then `rm -f *.bak` — works on both |
| Validation returns 200 but `claude mcp list` still fails after restart | Plugin not installed, or `.mcp.json` missing | Check `claude plugin list` includes `ninja-pricer@ninja-pricer`. Re-run `claude plugin install ninja-pricer` if not. |
| Token validation returns 403 | Token is sales-role and the user expected admin | The token inherits the role of the issuing user. They need an admin token from an admin account in `/settings/tokens`. |
