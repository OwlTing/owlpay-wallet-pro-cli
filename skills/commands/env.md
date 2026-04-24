# Environment

Manage the persistent environment (`prod` or `stage`) that the CLI uses when no per-command `--stage` flag is passed.

No authentication required. `env` commands **do not** emit the stage banner.

## Resolution Precedence

1. `--stage` flag on the current command (highest)
2. `env` field in `~/.owlpay-wallet-pro/config.json`
3. Default: `prod`

## Commands

```bash
owlp env              # Show current env + endpoints (same as `env get`)
owlp env get --json   # Show current env + endpoints (machine-readable)
owlp env set stage    # Persist env = stage to config.json
owlp env set prod     # Persist env = prod to config.json
```

**Agent execution** — always capture output (SKILL.md § Output Discipline):

```bash
RESULT=$(owlp env get --json 2>/dev/null) && echo "$RESULT" | jq '.data | {env, source}'
```

## JSON Responses

All responses are wrapped in `{success, env, data}`.

### env get

```json
{
  "success": true,
  "env": "stage",
  "data": {
    "env": "stage",
    "source": "config.json",
    "endpoints": {
      "api": "https://wallet-pro-stage.owlting.com/api",
      "sso": "https://auth-dev.owlting.com"
    }
  }
}
```

`data.source` is one of:

| Value | Meaning |
|-------|---------|
| `--stage` | Resolved from the `--stage` flag on this command |
| `config.json` | Resolved from the persisted config file |
| `default` | No flag, no config — falling back to `prod` |

### env set

```json
{
  "success": true,
  "env": "prod",
  "data": {
    "changed": true,
    "from": "prod",
    "to": "stage",
    "endpoints": {
      "api": "https://wallet-pro-stage.owlting.com/api",
      "sso": "https://auth-dev.owlting.com"
    }
  }
}
```

`data.changed` is `false` when the target env already matches the persisted value. Invalid values throw `INVALID_ENV`.

## Notes

- `config.json` lives at `~/.owlpay-wallet-pro/config.json` (root level, **not** env-scoped). The same file is consulted regardless of which env is active.
- Passing `--stage` while running `env set prod` logs a warning to stderr but still persists `prod`.
- Auth tokens and wallets remain env-scoped under `~/.owlpay-wallet-pro/{prod|stage}/` — switching env also switches which `auth.json` / `wallets.json` is loaded.
