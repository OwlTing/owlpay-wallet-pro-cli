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

In agent mode, a missing required flag throws `BusinessError('INPUT_REQUIRED')` with a message naming the flag. In TTY mode, missing flags are collected interactively instead.

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
{"type":"preview.start","chain":"stellar","from":"G...","to":"G...","amount":"10","ts":...}
{"type":"preview.done","chain":"stellar","from":"G...","to":"G...","amount":"10","tokenSymbol":"USDC","estimatedFee":"0.00001","feeUnit":"XLM","nativeBalance":"1.5","nativeSymbol":"XLM","insufficientGas":false,"feeSponsored":true,"ts":...}
{"type":"make.start","chain":"stellar","from":"G...","to":"G...","amount":"10","token":"USDC","ts":...}
{"type":"make.done","chain":"stellar","unit":"stroops","ts":...}
{"type":"sign.start","chain":"stellar","ts":...}
{"type":"sign.done","chain":"stellar","ts":...}
{"type":"submit.start","chain":"stellar","ts":...}
{"type":"submit.done","chain":"stellar","txHash":"abc123...","ts":...}
{"type":"complete","result":{"chain":"stellar","txHash":"abc123...","from":"G...","to":"G...","amount":"10","symbol":"USDC","fee_charged":"100"},"ts":...}
```

### Preview-only sequence (no `--confirm`)

Preview mode stops after `make.done`; the final `complete` event carries a `MakeResult` (no `txHash`):

```
{"type":"meta.env",...}
{"type":"preview.start",...}
{"type":"preview.done",...}
{"type":"make.start",...}
{"type":"make.done",...}
{"type":"complete","result":{"chain":"stellar","rawTransaction":"...","from":"G...","to":"G...","amount":"10","unit":"stroops"},"ts":...}
```

### Key Events

| Event | Emitted when | Key fields |
|-------|-------------|-----------|
| `preview.done` | Fee estimate + native balance fetched | `estimatedFee`, `feeUnit`, `nativeBalance`, `nativeSymbol`, `insufficientGas`, `feeSponsored` |
| `make.done` | Server returned an unsigned raw transaction | `chain`, `unit` |
| `sign.done` | Local signing succeeded (submit mode only) | `chain` |
| `submit.done` | Broadcast succeeded (submit mode only) | `txHash` |
| `complete` | Stream finished successfully | `result` — see shapes below |
| `error` | Command failed | `code`, `message`, `exitCode` |
| `cancelled` | User interrupted (SIGINT) | `reason: "user_interrupt"` |

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

Chain-specific fields: `gasUsed`/`gasFee`/`blockNumber` are EVM; `ledger`/`successful`/`fee_charged` are Stellar.

Preview mode (`MakeResult`):

```json
{
  "chain": "stellar",
  "rawTransaction": "AAAAAgAAAAB...",
  "message": "abc123...",
  "from": "G...",
  "to": "G...",
  "amount": "10",
  "unit": "stroops"
}
```

## Insufficient-Gas Warning

If native balance is below the estimated fee, `preview.done` includes `"insufficientGas": true`. Inspect this before continuing with `--confirm`. When fees are sponsored by OwlPay, `preview.done` emits `"feeSponsored": true` and the `insufficientGas` flag is informational — it has no effect on the user's ability to send (Stellar is the main case today).

## Fee Sponsorship

Stellar transfers have their fees sponsored by OwlPay. Preview for Stellar omits the estimate-fee API call; the `preview.done` event always carries `"feeSponsored": true`. The human-mode output renders `Fee    : Sponsored (no fee required)` in place of the fee line.

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
