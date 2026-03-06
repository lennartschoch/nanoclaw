# Intent: container/agent-runner/src/index.ts modifications

## What changed
Added the 1Password MCP server to the agent's available tools so it can read items from a 1Password vault using a service account token.

## Key sections

### mcpServers (inside runQuery → query() call)
- Added: `1password` MCP server alongside existing servers:
  ```typescript
  '1password': {
    command: 'npx',
    args: ['-y', '@1password/mcp'],
    env: {
      OP_SERVICE_ACCOUNT_TOKEN: sdkEnv['OP_SERVICE_ACCOUNT_TOKEN'] || '',
    },
  },
  ```

### allowedTools (inside runQuery → query() call)
- Added: `'mcp__1password__*'` to allow all 1Password MCP tools

## Invariants
- The `nanoclaw` MCP server configuration is unchanged
- All existing allowed tools are preserved
- The query loop, IPC handling, MessageStream, and all other logic is untouched
- Hooks (PreCompact, sanitize Bash) are unchanged
- Output protocol (markers) is unchanged

## Must-keep
- The `nanoclaw` MCP server with its environment variables
- All existing allowedTools entries
- The hook system (PreCompact, PreToolUse sanitize)
- The IPC input/close sentinel handling
- The MessageStream class and query loop
