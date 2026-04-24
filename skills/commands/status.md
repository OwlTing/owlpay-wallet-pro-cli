# Status

Unified dashboard showing account, KYC, and wallet readiness in a single call. Use this as the **first check** in any agent workflow to determine what steps are needed.

Unlike other authenticated commands, `status` is **non-throwing** — when the user is not logged in, it returns `ready: false` sections instead of an `AUTH_REQUIRED` error.

## Commands

```bash
owlp status --json   # Full status dashboard
```

**Agent execution** — always capture output (SKILL.md § Output Discipline):

```bash
RESULT=$(owlp status --json 2>/dev/null) && echo "$RESULT" | jq '.data | {account: .account.ready, kyc: .kyc.ready, wallets: .wallets.ready, email: .account.email, default_wallet: .wallets.default}'
```

## JSON Response

Wrapped in `{success, env, data}`:

```json
{
  "success": true,
  "env": "prod",
  "data": {
    "account": {
      "ready": true,
      "name": "Alice Wang",
      "email": "alice@example.com",
      "id": "uuid-xxx",
      "workspace": { "id": "ws-xxx", "name": "my-workspace", "type": "personal" }
    },
    "kyc": {
      "ready": true,
      "status": "verified",
      "label": "Verified"
    },
    "wallets": {
      "ready": true,
      "count": 2,
      "default": "my-wallet",
      "list": ["my-wallet", "savings"]
    }
  }
}
```

When not logged in, all sections return `{ "ready": false }`:

```json
{
  "success": true,
  "env": "prod",
  "data": {
    "account": { "ready": false },
    "kyc": { "ready": false },
    "wallets": { "ready": false }
  }
}
```

## `ready` Field

Each section has a `ready` boolean:

| Section | `ready: true` when |
|---------|-------------------|
| `account` | User is logged in with a valid session |
| `kyc` | KYC status is `verified` |
| `wallets` | At least one wallet exists |

## KYC Status Values

| Status | `ready` | Description |
|--------|---------|-------------|
| `verified` | `true` | Approved — off-ramp available |
| `unverified` | `false` | Not yet submitted |
| `unfinished` | `false` | Started but not completed |
| `finished` | `false` | Submitted, pending review |
| `rejected` | `false` | Rejected — can resubmit |
| `declined` | `false` | Declined |
| `revoked` | `false` | Previously verified, now revoked |
| `banned` | `false` | Account banned |

## Notes

- **Non-throwing**: always exits 0, even when not logged in. Errors are represented as `ready: false` in the relevant section rather than thrown.
- Use this as the **first check** in agent workflows — a single call replaces separate `whoami`, `kyc status`, and `wallet list` calls.
- KYC readiness is only relevant for off-ramp (fiat withdrawal) flows, not crypto-to-crypto transfers.
- For actionable KYC states (`unverified`, `rejected`, `revoked`), the human-mode output hints `→ Run owlp kyc submit` (not `owlp onboard`).

## Agent Response

Report account, KYC, and wallet readiness in a single natural-language response. When everything is ready, a short confirmation mentioning the user's email and default wallet is enough. When any section is not ready, name it and suggest the specific action: `owlp auth login` for account, `owlp kyc submit` for KYC, `owlp wallet create` for wallets. Note that KYC is only required for fiat off-ramp — mention this if KYC is the only unready section.
