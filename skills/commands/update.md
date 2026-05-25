# Update

Update owlp to the latest version. Spawns `npm install -g @owlpay/owlp-cli@latest` under the hood.

No authentication required.

## Commands

```bash
owlp update          # Human: spinner + result message
owlp update --json   # Agent: JSON envelope with update result
```

**Agent execution** — always capture output (SKILL.md § Output Discipline):

```bash
RESULT=$(owlp update --json 2>/dev/null) && echo "$RESULT" | jq '.data | {previous, current, updated}'
```

## JSON Responses

All responses are wrapped in `{success, env, data}`.

### Updated successfully

```json
{
  "success": true,
  "env": "prod",
  "data": {
    "previous": "0.5.14",
    "current": "0.5.15",
    "updated": true
  }
}
```

### Already up to date

```json
{
  "success": true,
  "env": "prod",
  "data": {
    "previous": "0.5.15",
    "current": "0.5.15",
    "updated": false
  }
}
```

### Update failed

```json
{
  "error": true,
  "code": "UPDATE_FAILED",
  "message": "<npm error output>"
}
```

Exit code: 1.

## Notes

- Does not require authentication — works even when not logged in.
- Requires network access to npm registry (`registry.npmjs.org`), but does **not** call the OwlPay API.
- Suppresses the post-command update notice (since the command itself handles updates).
- In human mode, displays a spinner during the npm install and prints `✓ Updated to <version>` or `✓ Already up to date (<version>)`.

## Agent Response

**Updated**: Report the version change (e.g. "owlp 已從 0.5.14 更新至 0.5.15"). Suggest the user verify with `owlp -V`.

**Already up to date**: Report that the current version is already the latest.

**Failed**: Report the error message and suggest the user check npm permissions (e.g. may need `sudo` or a Node version manager).
