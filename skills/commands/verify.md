# Verify Address

Validate an address and auto-detect its chain. Sends the address to the server's validation endpoint and also infers the chain locally from the address format.

No authentication required.

## Commands

```bash
owlp verify <address> --json
```

**Agent execution** — always capture output (SKILL.md § Output Discipline):

```bash
RESULT=$(owlp verify <address> --json 2>/dev/null) && echo "$RESULT" | jq '.data | {address, chain, valid}'
```

## JSON Responses

All responses are wrapped in `{success, env, data}`. Only the `data` payload is shown below.

### Valid address

```json
{
  "address": "GABC...",
  "chain": "stellar",
  "valid": true
}
```

Additional fields from the server's validation response (e.g. normalized address) are passed through.

### Invalid address

```json
{
  "address": "not-an-address",
  "chain": null,
  "valid": false
}
```

When the server returns HTTP 422, the command still exits 0 and emits `valid: false`. Any other error (network, 5xx) is thrown.

## Chain Auto-Detection

The `chain` field is inferred **locally** from the address format:

| Format | `chain` value |
|--------|--------------|
| `0x` + 40 hex chars | `evm` |
| `G` + 55 Base32 chars | `stellar` |
| 32–44 Base58 chars | `solana` |
| `T` + 33 Base58 chars | `tron` |
| other | `null` |

> **Note:** the inferred value is `evm` (not `ethereum`). When passing to `owlp send --chain`, use the CLI flag value `ethereum` instead.

Only `ethereum`, `stellar`, and `solana` are supported by `send`. `tron` is recognized by `verify` but not by other commands.

## Notes

- Always run `verify` before `send` to confirm the destination address and infer the correct `--chain` value.
- `verify` does not check whether the address exists on-chain or has a balance — only that the format is valid.
