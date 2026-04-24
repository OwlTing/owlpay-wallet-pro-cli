# Reset — wipe local state

Destructive commands to clear local state for the active env, or everything.

```bash
owlp reset [--force]         # Delete active env: auth.json + wallets.json for this env
owlp reset --all [--force]   # Delete ALL envs + config.json (full factory reset)
```

## Confirmation protocol

All destructive commands share a uniform confirmation:

- **TTY + human mode**: prints a warning to stderr, then prompts `Type "confirmed" to proceed:`. Anything other than the literal string `confirmed` cancels (exit 0, `Cancelled.`).
- **`--json` or non-TTY**: no prompt is possible. Must pass `--force`. Without `--force`, the command throws `DESTRUCTIVE_REQUIRES_CONFIRMATION` (exit 3).

## Scope

| Command | `~/.owlpay-wallet-pro/config.json` | Active env | Other env |
|---|---|---|---|
| `owlp reset` | keep | delete | keep |
| `owlp reset --all` | delete | delete | delete |

After `owlp reset --all`, running `owlp env get` returns the default env (`prod`) because the persisted config is gone.

## JSON Responses

### Success
```json
{ "success": true, "env": "prod", "data": { "message": "Reset complete.", "scope": "prod" } }
```
`scope` is either the active env name (for `owlp reset`) or `"all"` (for `owlp reset --all`).

### Refused (json / non-TTY without `--force`)
```json
{ "error": true, "code": "DESTRUCTIVE_REQUIRES_CONFIRMATION", "message": "This command is destructive. Re-run with --force after backing up required secrets (e.g. wallet mnemonics)." }
```
Exit code: 3.

## Examples

```bash
# Wipe just the stage env (prod untouched)
owlp --stage reset --force

# Full factory reset
owlp reset --all --force

# Interactive reset (prompts for "confirmed")
owlp reset
```

## Related

- `owlp wallet reset` — clear only wallets (keep auth session)
- `owlp auth logout` — clear only the auth session (keep wallets)
