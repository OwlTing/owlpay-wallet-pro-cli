# Auth — login, logout, status

Manage the local OwlPay authentication session. Session tokens are stored in `~/.owlpay-wallet-pro/{prod|stage}/auth.json`.

**Important**: Local wallets (`wallets.json`) are **not** deleted by `auth logout`. Use `owlp wallet reset` to remove wallets, or `owlp reset` to wipe the active env's state entirely.

## Commands

```bash
owlp auth login              # Opens browser for authentication (humans only)
owlp auth logout             # Clears local auth.json (keeps wallets)
owlp auth status             # Shows current logged-in user
```

Bare `owlp auth` prints the subcommand help (no default subcommand). Call `owlp auth status` explicitly to check login state.

`auth login` is interactive — it opens the default browser to the `owlp-cli-service` auth page and waits for a callback. Not agent-friendly; for agent-driven account setup prefer the two-step `owlp onboard --json` + `--auth-code` flow (see `skills/commands/onboard.md`). Calling `auth login` when already logged in short-circuits with `Already logged in.` (JSON: `{message, workspace}`) and does not re-open the browser.

`auth logout` is **non-destructive** — no prompt, no `--force`, idempotent. It clears `auth.json` only. Use `owlp reset` if you want to clear wallets too.

## JSON Responses

### auth login
```json
{ "success": true, "env": "prod", "data": { "message": "Login successful.", "email": "user@example.com" } }
```
In `--json` mode, `auth login` also emits NDJSON events (`auth.browser_open`, `auth.callback_received`, `auth.done`) before the final `complete` line.

### auth logout
```json
{ "success": true, "env": "prod", "data": { "message": "Logged out." } }
```

### auth status — logged in
```json
{
  "success": true,
  "env": "prod",
  "data": {
    "loggedIn": true,
    "id": "...",
    "name": "...",
    "email": "...",
    "workspace": { "id": "...", "name": "...", "type": "personal" }
  }
}
```

### auth status — not logged in (exit 0)
```json
{ "success": true, "env": "prod", "data": { "loggedIn": false } }
```

## Multi-account warning

After `auth login` succeeds, the CLI checks whether the local `wallets.json` (if any) was created under a different account. If so, it emits a warning pointing at `owlp wallet reset`:

- Human mode: warning written to **stderr** after login success
- JSON mode: an NDJSON `{"type":"warning","code":"OWNER_UUID_MISMATCH","message":"...","storedUuid":"...","newUuid":"..."}` event before the final `complete` line

The login itself still succeeds — the warning is advisory.

## Notes

- `auth login` is a two-step flow internally: SSO login → token exchange via `owlp-cli-service`. Both sets of tokens are persisted to `auth.json`.
- 401 on subsequent API calls triggers automatic refresh via the `ApiClient` interceptor.
- To create a new account, use `owlp onboard` (interactive wizard).
- For a unified readiness view, use `owlp status` (separate, top-level command that composes account/KYC/wallet).
