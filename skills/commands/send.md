# Send

Send tokens to an external address. The command has two distinct operating modes:

- **Agent mode** — triggered by `--json`, non-TTY stdio, or `CLAUDECODE` / `CI` / `CURSOR_AGENT` env vars. Non-interactive; requires every flag below or returns `INPUT_REQUIRED` (exit 3). `--confirm` is the switch between preview-only and actual send.
- **TTY human mode** — a clack wizard prompts for any missing fields (address → chain → wallet → token → amount → memo), then renders the preview and asks `Proceed with this transfer?` before signing. `--confirm` in TTY mode *skips* that prompt instead of gating the send.

Agents should always invoke this command with `--json` plus every required flag; that guarantees the non-interactive path and the stable NDJSON event stream documented below.

Requires login. Always run preview first to inspect fees before submitting.

## Options

| Option | Required (agent) | Description |
|--------|------------------|-------------|
| `--to <address>` | Yes | Destination address |
| `--amount <number>` | Yes | Amount to send |
| `--token <token>` | Yes | Token symbol (`USDC`, `ETH`, `XLM`, `SOL`, or `native`) |
| `--chain <chain>` | Yes | `ethereum`, `stellar`, or `solana` |
| `--memo <text>` | No | Memo / tag (Stellar and Solana) |
| `--confirm` | No | In agent mode: sign and submit (omit for preview only). In TTY mode: skip the `Proceed?` prompt. |
| `--idempotency-key <key>` | No | Replay protection: re-running with the same key + args returns the cached result (`idempotentReplay: true`) instead of sending again. Recommended whenever an agent might retry. |

Native chain coins may be passed either as `native` or by chain symbol:
`ETH` on `ethereum`, `XLM` on `stellar`, and `SOL` on `solana`.
The CLI normalizes these aliases to the backend native-transfer contract.

In agent mode, missing required flags throw `BusinessError('INPUT_REQUIRED')` with a single message listing **every** missing flag and a machine-readable `missing` array on the JSON error envelope (e.g. `"missing":["--to","--chain"]`) — retry once with all of them. In TTY mode, missing flags are collected interactively instead.

### Idempotency (`--idempotency-key`)

Client-side, 24h window, env-scoped; the key is also forwarded as the server `Idempotency-Key` header. Same key + same args → cached `SubmitResult` with `"idempotentReplay": true` and **no** new signing or broadcast. Same key + different args → exit 3 `IDEMPOTENCY_KEY_REUSED`. Without a key, nothing is recorded (unlike `deposit submit`, which generates a key by default). Limits:

- Only a **received success** is recorded. An on-ledger failure (Stellar `successful: false`) is not cached — retrying it re-sends, as intended. A submit whose response was **lost** (timeout after dispatch) is also not cached, but the transfer may have landed: on an unknown outcome, check `owlp tx list --json` before retrying.
- Whitespace and chain casing are normalized before hashing; other formatting differences (`10` vs `10.0`) count as different args. Retry with byte-identical values.
- No lock: don't run concurrent invocations with the same key.
- If the cache write fails after a successful broadcast, a `warning` event is emitted stating the transfer WAS submitted — gate any retry on it.
- After a replay, verify on-chain state with `owlp tx list --json` if freshness matters.

## Commands

```bash
owlp send --to <address> --amount 10 --token USDC --chain stellar --json              # Preview (no --confirm)
owlp send --to <address> --amount 10 --token USDC --chain stellar --confirm --json     # Execute
owlp send --to G... --amount 10 --token USDC --chain stellar --memo "invoice-123" --confirm --json   # Stellar with memo
```

**Agent execution** — always capture output (SKILL.md § Output Discipline). Send emits NDJSON; the last line is the `complete` event:

```bash
RESULT=$(owlp send --to <address> --amount 10 --token USDC --chain stellar --confirm --json 2>/dev/null) && echo "$RESULT" | tail -1 | jq '.result | {txHash, chain, amount, symbol}'
```

## Output Format — NDJSON Event Stream

`send` uses the **event-stream** output, not the `{success, env, data}` envelope. In `--json` mode it emits NDJSON: one JSON object per line.

Parse line-by-line, e.g.:

```bash
owlp send ... --json | while IFS= read -r line; do
  echo "$line" | jq -c 'select(.type == "complete" or .type == "submit.done")'
done
```

### Event Sequence (submit mode with `--confirm`)

```
{"type":"meta.env","env":"prod","endpoints":{...}}
{"type":"check.start","chain":"stellar","from":"G...","to":"G...","amount":"10","token":"USDC","ts":...}
{"type":"check.fees","estimatedFee":"0.00001","feeUnit":"XLM","feeSponsored":true,"ts":...}
{"type":"check.balance","nativeBalance":"1.5","nativeSymbol":"XLM","insufficientGas":false,"feeSponsored":true,"ts":...}
{"type":"check.validate","chain":"stellar","unit":"stroops","ts":...}
{"type":"check.done","payload":{"chain":"stellar","from":"G...","to":"G...","amount":"10","tokenSymbol":"USDC","estimatedFee":"0.00001","feeUnit":"XLM","nativeBalance":"1.5","nativeSymbol":"XLM","insufficientGas":false,"feeSponsored":true,"unit":"stroops"},"ts":...}
{"type":"sign.start","chain":"stellar","ts":...}
{"type":"sign.done","chain":"stellar","ts":...}
{"type":"submit.start","chain":"stellar","ts":...}
{"type":"submit.done","chain":"stellar","txHash":"abc123...","ts":...}
{"type":"complete","result":{"chain":"stellar","txHash":"abc123...","from":"G...","to":"G...","amount":"10","symbol":"USDC","fee_charged":"100"},"ts":...}
```

### Preview-only sequence (no `--confirm`)

Preview mode stops after `check.done`; the final `complete` event carries a `CheckResult` (no `txHash`):

```
{"type":"meta.env",...}
{"type":"check.start",...}
{"type":"check.fees",...}
{"type":"check.balance",...}
{"type":"check.validate",...}
{"type":"check.done",...}
{"type":"complete","result":{"chain":"stellar","rawTransaction":"...","from":"G...","to":"G...","amount":"10","unit":"stroops"},"ts":...}
```

### Key Events

| Event | Emitted when | Key fields |
|-------|-------------|-----------|
| `check.fees` | Fee estimate fetched | `estimatedFee`, `feeUnit`, `feeSponsored` |
| `check.balance` | Native balance fetched | `nativeBalance`, `nativeSymbol`, `insufficientGas`, `feeSponsored` |
| `check.done` | Server returned an unsigned raw transaction | `payload` (full transfer preview) |
| `sign.done` | Local signing succeeded (submit mode only) | `chain` |
| `submit.done` | Broadcast succeeded (submit mode only) | `txHash` |
| `complete` | Stream finished successfully | `result` — see shapes below |
| `warning` | Non-fatal problem (e.g. idempotency cache write failed after a successful broadcast) | `message` — **read it before retrying**: it states whether the transfer WAS submitted |
| `error` | Command failed mid-stream | `code`, `message`, `exitCode`, optional `hintAction` |
| `cancelled` | User interrupted (SIGINT) | `reason: "user_interrupt"` |

Missing-flag validation runs **before** the stream starts, so `INPUT_REQUIRED` arrives as a JSON error envelope — `{"error":true,"code":"INPUT_REQUIRED","message":"Missing required flags: ...","missing":["--to","--chain"],...}` — not as a stream `error` event.

On an idempotent replay (same `--idempotency-key` + args), the stream contains only `meta.env` and `complete`; the result carries `"idempotentReplay": true`.

### `complete.result` Shapes

Submit mode (`SubmitResult`):

```json
{
  "chain": "stellar",
  "txHash": "abc123...",
  "from": "G...",
  "to": "G...",
  "amount": "10",
  "symbol": "USDC",
  "status": "submitted",
  "gasUsed": 21000,
  "gasFee": 42000000000000,
  "blockNumber": 12345678,
  "ledger": 45678901,
  "successful": true,
  "fee_charged": "100"
}
```

Chain-specific fields: `gasUsed`/`gasFee`/`blockNumber` are EVM; `ledger`/`successful`/`fee_charged` are Stellar. When the result was answered from the idempotency cache it additionally carries `"idempotentReplay": true`.

Preview mode (`CheckResult` — the `check.done` payload plus the unsigned transaction):

```json
{
  "chain": "stellar",
  "rawTransaction": "AAAAAgAAAAB...",
  "message": "abc123...",
  "from": "G...",
  "to": "G...",
  "amount": "10",
  "tokenSymbol": "USDC",
  "estimatedFee": "0.00001",
  "feeUnit": "XLM",
  "nativeBalance": "1.5",
  "nativeSymbol": "XLM",
  "insufficientGas": false,
  "feeSponsored": true,
  "unit": "stroops"
}
```

## Insufficient Gas

If native balance is below the estimated fee (and fees are not sponsored), the stream **fails fast**: `check.balance` carries `"insufficientGas": true`, then the command exits 3 with `INSUFFICIENT_GAS` — `check.done`/`complete` are never reached. Top up the native token, then retry. When fees are sponsored, `insufficientGas` is always `false` on emitted events.

## Fee Sponsorship

Stellar transfers have their fees sponsored by OwlPay. Check for Stellar omits the estimate-fee API call; the `check.fees` and `check.balance` events carry `"feeSponsored": true`. The human-mode output renders `Fee    : Sponsored (no fee required)` in place of the fee line.

## Notes

- Private keys never leave the client — signing happens locally via `lib/signer.ts`. The API only receives signed transactions.
- KYC is **not** required for crypto-to-crypto transfers (only for fiat off-ramp).
- Always run `owlp verify <address> --json` (captured, per Output Discipline) to confirm the destination is valid.
- Always run `owlp tokens --chain <chain> --json` (captured, per Output Discipline) to confirm the token symbol is supported.
- On SIGINT, a `{"type":"cancelled","reason":"user_interrupt"}` event is emitted and the process exits 130.

## Agent Response

**Preview mode** (no `--confirm`): Summarize the transfer details — from, to, amount, token, chain, and fee information. If fees are sponsored, mention it. Then ask the user whether to proceed with `--confirm`.

**Submit mode** (with `--confirm`): Confirm the transaction was sent. Mention the amount, token, destination address, and txHash. Suggest using `owlp tx detail <id>` to track confirmation status.

**Errors:**
- `INSUFFICIENT_GAS`: Explain the native token balance is too low to cover gas fees, and suggest topping up the native token on that chain.
- `CHECK_VALIDATE_FAILED`: Report the server validation error message. If `probableCause` or `hintAction` is present in the event, relay it.
