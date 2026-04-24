# Chains and Tokens

Discover supported blockchain networks and tokens. These commands do **not** require authentication.

For address verification, see `verify.md`.

## Commands

```bash
owlp chains --json                        # List all supported blockchain networks
owlp tokens --json                        # List all supported tokens (all chains)
owlp tokens --chain ethereum --json       # Filter tokens by chain
owlp tokens --chain stellar --json
owlp tokens --chain solana --json
```

**Agent execution** — always capture output (SKILL.md § Output Discipline):

```bash
RESULT=$(owlp tokens --chain ethereum --json 2>/dev/null) && echo "$RESULT" | jq -r '.data[] | .symbol'
```

Supported `--chain` values: `ethereum`, `stellar`, `solana`.

## JSON Responses

All responses are wrapped in `{success, env, data}`. Only the `data` payload is shown below.

### chains

```json
[
  { "chain": "ethereum", "explorer_url": "https://etherscan.io" },
  { "chain": "stellar",  "explorer_url": "https://stellar.expert/explorer/public" },
  { "chain": "solana",   "explorer_url": "https://solscan.io" }
]
```

The server may return additional fields; only `chain` and `explorer_url` are used by the CLI's human-mode output.

### tokens

```json
[
  { "chain": "ethereum", "symbol": "ETH" },
  { "chain": "ethereum", "symbol": "USDC" },
  { "chain": "stellar",  "symbol": "XLM" },
  { "chain": "stellar",  "symbol": "USDC" },
  { "chain": "solana",   "symbol": "SOL" },
  { "chain": "solana",   "symbol": "USDC" }
]
```

## Notes

- Use `owlp tokens --chain <chain> --json` (captured, per Output Discipline) before `send` to confirm the `--token` symbol is supported.
- An unsupported `--chain` value throws `UNSUPPORTED_CHAIN` (exit code 3).
