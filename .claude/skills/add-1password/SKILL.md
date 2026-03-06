---
name: add-1password
description: Add 1Password vault access to NanoClaw using a service account token. The agent gets read access to vault items via the 1Password MCP server. No channel mode — tool-only.
---

# Add 1Password Integration

This skill gives the container agent access to 1Password vault items via the `@takescake/1password-mcp` MCP server. Authentication uses a service account token (no OAuth, no browser flow).

## Phase 1: Pre-flight

### Check if already applied

Read `.nanoclaw/state.yaml`. If `1password` is in `applied_skills`, skip to Phase 3 (Setup). The code changes are already in place.

## Phase 2: Apply Code Changes

### Initialize skills system (if needed)

If `.nanoclaw/` directory doesn't exist yet:

```bash
npx tsx scripts/apply-skill.ts --init
```

### 1. Add token to readSecrets()

Apply the changes described in `modify/src/container-runner.ts.intent.md` to `src/container-runner.ts`: add `'OP_SERVICE_ACCOUNT_TOKEN'` to the `readEnvFile([...])` array inside `readSecrets()`.

### 2. Add 1Password MCP server to agent runner

Apply the changes described in `modify/container/agent-runner/src/index.ts.intent.md` to `container/agent-runner/src/index.ts`: add the `1password` MCP server entry to `mcpServers` and add `'mcp__1password__*'` to `allowedTools`.

### 3. Record in state

Add `1password` to `.nanoclaw/state.yaml` under `applied_skills` with `mode: tool-only`.

### 4. Validate

```bash
npm run build
```

Build must be clean before proceeding.

## Phase 3: Setup

### Check for existing token

```bash
grep -q OP_SERVICE_ACCOUNT_TOKEN .env 2>/dev/null && echo "Token already set" || echo "Token not set"
```

If the token is already in `.env`, skip to "Build and restart" below.

### Create a 1Password service account

Tell the user:

> I need a 1Password service account token with read access to the vaults you want the agent to use:
>
> 1. Open **1Password** and go to **Developer Tools > Service Accounts**
>    (or visit your 1Password account at `<your-team>.1password.com/developer-tools/service-accounts`)
> 2. Click **New Service Account**, give it a name (e.g. "NanoClaw")
> 3. Grant it **read access** to the vaults the agent should be able to query
> 4. Copy the generated token (starts with `ops_...`)
>
> Paste the token here and I'll add it to your `.env`.

When the user provides the token, append it to `.env`:

```bash
echo "OP_SERVICE_ACCOUNT_TOKEN=ops_..." >> .env
```

(Use the actual token value the user provided.)

### Build and restart

Clear stale per-group agent-runner copies so existing groups pick up the new MCP server:

```bash
rm -r data/sessions/*/agent-runner-src 2>/dev/null || true
```

Rebuild the container (agent-runner changed):

```bash
cd container && ./build.sh
```

Then compile and restart:

```bash
npm run build
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # macOS
# Linux: systemctl --user restart nanoclaw
```

## Phase 4: Verify

### Test vault access

Tell the user:

> 1Password is connected. Send this in your main channel:
>
> `list my 1Password vaults` or `find the login for GitHub in 1Password`

### Check logs if needed

```bash
tail -f logs/nanoclaw.log | grep -iE "(1password|op_)"
```

## Troubleshooting

### "OP_SERVICE_ACCOUNT_TOKEN not set" or MCP server fails to start

- Verify the token is in `.env`: `grep OP_SERVICE_ACCOUNT_TOKEN .env`
- Verify the token starts with `ops_`
- Restart after editing `.env`: `launchctl kickstart -k gui/$(id -u)/com.nanoclaw` (macOS) or `systemctl --user restart nanoclaw` (Linux)

### Agent can't find vault items

- Confirm the service account has read access to the target vault in 1Password settings
- Try a broader query: "list all vaults I have access to in 1Password"

### Container can't reach the MCP server

- Check container logs: `cat groups/main/logs/container-*.log | tail -50`
- Verify the agent-runner was rebuilt: stale copies won't have the `1password` MCP entry

### npx takes too long on first use

The first invocation downloads `@takescake/1password-mcp`. Subsequent calls use the npx cache. This is expected.

## Removal

1. Remove `'OP_SERVICE_ACCOUNT_TOKEN'` from `readSecrets()` in `src/container-runner.ts`
2. Remove the `1password` MCP server and `mcp__1password__*` from `container/agent-runner/src/index.ts`
3. Remove `OP_SERVICE_ACCOUNT_TOKEN` from `.env`
4. Remove `1password` from `.nanoclaw/state.yaml`
5. Clear stale agent-runner copies: `rm -r data/sessions/*/agent-runner-src 2>/dev/null || true`
6. Rebuild: `cd container && ./build.sh && cd .. && npm run build && launchctl kickstart -k gui/$(id -u)/com.nanoclaw` (macOS) or `systemctl --user restart nanoclaw` (Linux)
