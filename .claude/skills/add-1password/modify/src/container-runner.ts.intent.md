# Intent: src/container-runner.ts modifications

## What changed
Added `OP_SERVICE_ACCOUNT_TOKEN` to `readSecrets()` so the 1Password service account token is passed to the container via stdin (never mounted or written to disk).

## Key sections

### readSecrets()
- Added: `'OP_SERVICE_ACCOUNT_TOKEN'` to the `readEnvFile([...])` array alongside the existing API key entries:
  ```typescript
  function readSecrets(): Record<string, string> {
    return readEnvFile([
      'CLAUDE_CODE_OAUTH_TOKEN',
      'ANTHROPIC_API_KEY',
      'ANTHROPIC_BASE_URL',
      'ANTHROPIC_AUTH_TOKEN',
      'OP_SERVICE_ACCOUNT_TOKEN',
    ]);
  }
  ```

## Invariants
- No volume mounts added — credentials are env-var only
- All existing secrets entries are unchanged
- No other functions are modified
