# Balance

Query wallet balances across all chains and tokens. Returns native coins and configured tokens (USDC, etc.) for each chain.

Requires login.

## Commands

```bash
owlp balance --json                                 # All chains, all tokens on the default wallet
owlp balance --chain ethereum --json                # Filter to one chain
owlp balance --chain stellar --address G... --json  # Specific address on a specific chain
owlp balance --wallet my-wallet --json              # Use a non-default wallet
```

**Agent execution** — always capture output (SKILL.md § Output Discipline):

```bash
RESULT=$(owlp balance --json 2>/dev/null) && echo "$RESULT" | jq -r '.data[] | "\(.chain) \(.symbol): \(.balance)"'
```

Supported chains: `ethereum`, `avalanche`, `polygon`, `optimism`, `arbitrum`, `stellar`, `solana`. All five EVM chains share the wallet's single EVM address. A chain the server reports no tokens for simply contributes no rows.

Partial failures degrade gracefully: if a chain/token query fails, the remaining rows are still returned (JSON `data` stays a flat array, exit 0). In `--json` mode the envelope carries a top-level `failures: [{chain, token, message}]` field **only when at least one query failed** — check `.failures` before treating the report as complete (`jq 'has("failures")'`). Each failure is also written as a `⚠` warning line on **stderr in both modes** (diagnostic mirror; parse the envelope, not stderr).

`--address` requires `--chain` (an address is only meaningful on one chain); without it the command exits 3 with `INPUT_REQUIRED`.

## JSON Response

Wrapped in the standard `{success, env, data}` envelope; `data` is an array:

```json
{
  "success": true,
  "env": "prod",
  "data": [
    {
      "address": "0x...",
      "balance": "1.00",
      "symbol": "ETH",
      "chain": "ethereum",
      "balanceInUsd": "2000.00"
    },
    {
      "address": "0x...",
      "balance": "100.00",
      "symbol": "USDC",
      "chain": "ethereum",
      "balanceInUsd": "100.00"
    }
  ]
}
```

Parse with `jq '.data[]'`.

## Agent Response

Introduce the wallet by name, then list each token's balance and chain. Include the USD equivalent when available. If a chain has zero balance, mention it briefly or omit — use your judgement based on whether the user cares about that chain. Example tone: "你的 my-wallet 錢包目前有 100 USDC on Ethereum（≈ $100）、50 XLM on Stellar（≈ $5）。"
