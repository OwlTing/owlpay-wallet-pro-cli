# Pay (x402)

Pay an x402-protected resource by signing an **EIP-3009 `TransferWithAuthorization`** locally. The client only signs; a facilitator broadcasts the on-chain `transferWithAuthorization` and pays gas. Two providers, auto-detected from the URL:

- **Harbor** (OwlPay checkout links, `*.owlpay.com`) — `requirements → enable (bind wallet + KYC) → challenge → settle`.
- **Standard** (any other x402 endpoint) — `GET → HTTP 402 (accepts) → settle` with the `X-PAYMENT` header.

The recipient, amount, asset, and network come from the server's payment challenge — you never pass `--to`/`--amount`/`--asset`. v1 supports the `exact` scheme on EVM with **USDC** only.

Requires a wallet (`owlp wallet create`/`import`); the EVM account (`m/44'/60'/0'/0/0`) is the payer. Does **not** require `owlp auth login` — x402 endpoints are anonymous. The private key never leaves the client.

## Operating modes

- **Agent mode** (`--json` / non-TTY): non-interactive. Without `--confirm` it stops after `pay.preview` (no signing). With `--confirm` it signs and settles.
- **TTY human mode**: renders the preview, asks `Pay <amount> <symbol> to <payTo>?`, then signs + settles on confirm. `--confirm` skips the prompt.

## Options

| Option | Description |
|--------|-------------|
| `<url>` (arg) | x402 resource / OwlPay checkout URL |
| `--chain <family>` | Payer chain family: `ethereum` (default), `avalanche`, `polygon`, `optimism`. Sets Harbor `source.chain`; the challenge network is authoritative for signing |
| `--provider <p>` | `auto` (default), `harbor`, or `x402` (standard) |
| `--kyc <json>` | `user_information` as an inline JSON string (Harbor enable) |
| `--kyc-file <path>` | JSON file with `user_information` |
| `--save-kyc` | Persist the provided KYC profile locally (mode 0600) for reuse |
| `--max-amount <usdc>` | Reject (even with `--confirm`) if the challenge amount exceeds this |
| `--inspect` | Read-only: show requirements + intent/challenge; no wallet/KYC/enable/sign |
| `--confirm` | Agent: sign + settle. TTY: skip the confirm prompt |
| `--idempotency-key <key>` | Replay protection: re-running with the same key + args returns the cached settlement (`idempotentReplay: true`) instead of paying again. Recommended whenever an agent might retry |

**Idempotency:** client-side, 24h window, env-scoped; the args hash covers `{url, from, chain}`. Same key + same args → cached settlement, no new signing/settling; the replayed stream contains only `meta.env` and `complete` with `"idempotentReplay": true`. Same key + different args → exit 3 `IDEMPOTENCY_KEY_REUSED`. Without a key nothing is recorded. Limits:

- A settle **rejected by the server** records nothing, so retrying with the same key is safe. A settle whose response was **lost** (timeout after dispatch) also records nothing — but the payment may have gone through; verify the payment intent status before retrying, and never auto-retry a dispatched settle.
- Use one **fresh key per intended payment**. Standard x402 resource URLs are stable, so paying the same URL twice on purpose with a reused key replays the old settlement (stale `transactionHash` reported as success, `--max-amount` never re-checked) instead of paying.
- Retry with the **same `--chain` value** you used originally — the requested chain family is part of the args hash, so switching it (e.g. after the "intent settles on X" warning) trips `IDEMPOTENCY_KEY_REUSED` instead of replaying. The warning is informational; the challenge network decides actual signing regardless of `--chain`.

**KYC precedence:** `--kyc` (inline) > `--kyc-file` > persisted profile. Both `--kyc` and `--kyc-file` accept either `{ "user_information": {...} }` or a bare `user_information` object. Required fields come from the runtime requirements schema (emitted on the `pay.requirements` event) — typically `applicant`, `country`, `email`, plus individual/company conditionals.

## Commands

```bash
owlp pay <url> --inspect --json                              # Read-only: see amount/payTo/status + KYC schema
owlp pay <url> --kyc '{...}' --json                          # Agent: enable + preview (no settle without --confirm)
owlp pay <url> --kyc '{...}' --confirm --json                    # Agent: sign + settle
owlp pay <url> --kyc-file ./kyc.json --save-kyc --confirm --json # Settle and persist KYC for reuse
owlp pay <url> --max-amount 100 --kyc-file ./kyc.json --confirm --json  # With an amount guardrail
```

**Agent execution** — capture output (SKILL.md § Output Discipline). Pay emits NDJSON; the last line is `complete`:

```bash
RESULT=$(owlp pay <url> --kyc-file ./kyc.json --confirm --json 2>/dev/null) && echo "$RESULT" | tail -1 | jq '.result | {transactionHash, amount, symbol, network}'
```

## Output Format — NDJSON Event Stream

Like `send`, `pay` uses the **event-stream** output (not the `{success, env, data}` envelope) — except `--inspect`, which uses the standard envelope. Parse line-by-line.

### Event sequence (Harbor, settle with `--confirm`)

```
{"type":"meta.env","env":"stage","endpoints":{...}}
{"type":"pay.probe","provider":"harbor","url":"https://checkout-stage.owlpay.com/pi_test_...","ts":...}
{"type":"pay.requirements","schema":{...X402_ENABLE JSON Schema...},"ts":...}
{"type":"pay.enable","transferUuid":"transfer_...","payerAddress":"0x...","expiresAt":"...","ts":...}
{"type":"pay.challenge","network":"eip155:11155111","chainId":11155111,"asset":"0x...","payTo":"0x...","amount":"105527638","maxTimeoutSeconds":60,"ts":...}
{"type":"pay.preview","payload":{"provider":"harbor","from":"0x...","payTo":"0x...","amount":"105527638","amountDisplay":"105.527638","asset":"0x...","symbol":"USDC","network":"eip155:11155111","chainId":11155111},"ts":...}
{"type":"pay.sign","chainId":11155111,"ts":...}
{"type":"pay.settle","provider":"harbor","ts":...}
{"type":"pay.done","transactionHash":"0x...","transferUuid":"transfer_...","network":"eip155:11155111","ts":...}
{"type":"complete","result":{"provider":"harbor","from":"0x...","payTo":"0x...","amount":"105527638","amountDisplay":"105.527638","asset":"0x...","symbol":"USDC","network":"eip155:11155111","chainId":11155111,"transactionHash":"0x...","transferUuid":"transfer_..."},"ts":...}
```

### Preview-only (no `--confirm`)

Stops after `pay.preview`; `complete.result` is the preview (no `transactionHash`). `pay.sign`/`pay.settle` are not emitted.

### Standard provider

No `pay.enable` (no KYC/enable step). Settlement uses the `X-PAYMENT` header; `complete.result` may include `X-PAYMENT-RESPONSE` data.

### Key events

| Event | Emitted when | Key fields |
|-------|-------------|-----------|
| `pay.probe` | Provider selected | `provider` (`harbor`/`standard`) |
| `pay.requirements` | Harbor requirements fetched | `schema` (KYC JSON Schema) |
| `pay.enable` | Harbor wallet+KYC bound | `transferUuid`, `payerAddress`, `expiresAt` |
| `pay.challenge` | Payment challenge parsed | `payTo`, `amount`, `asset`, `network`, `chainId`, `maxTimeoutSeconds` |
| `pay.preview` | Ready to sign | `payload` (full preview) |
| `pay.done` | Settlement confirmed | `transactionHash`, `transferUuid`, `network` |
| `warning` | Non-fatal (e.g. challenge family ≠ `--chain`) | `message` |
| `complete` | Stream finished | `result` |
| `error` | Command failed | `code`, `message`, `exitCode` |

## `--inspect` (read-only)

Standard `{success, env, data}` envelope (not NDJSON). Harbor returns `requirementsSchema` + an `intent` summary; standard returns a `challenge`. No wallet, KYC, enable, or signing.

**Note:** the Harbor `intent` summary is the checkout's *default* network/amount. The actual payable network and amount follow your `--chain` (Harbor recomputes the fee per chain) and only appear in the real `pay.challenge` after enable — e.g. an intent shown as `optimism / 100.50` becomes `ethereum(Sepolia) / 105.53` when enabled with `--chain ethereum`.

```bash
owlp pay <url> --inspect --json | jq '.data | {provider, intent}'
```

## Errors

| Code | Meaning |
|------|---------|
| `NO_WALLET` | No wallet configured — run `owlp wallet create`/`import` |
| `X402_KYC_REQUIRED` | Harbor enable needs `user_information` — pass `--kyc`/`--kyc-file` (schema is on `pay.requirements`) |
| `X402_ENABLE_FAILED` | Enable rejected — includes the server reason (e.g. intent already paid, KYC invalid) |
| `X402_NO_ACCEPTABLE_REQUIREMENT` | No `exact` EVM/USDC entry with an EIP-712 domain in the challenge |
| `X402_AMOUNT_EXCEEDS_MAX` | Challenge amount exceeds `--max-amount` |
| `X402_SETTLE_FAILED` | Settlement failed — server reason surfaced. **No auto-retry after dispatch** (avoids double-settlement) |
| `UNSUPPORTED_CHAIN` | `--chain` not one of the four EVM families |

## Notes

- **Facilitator pays gas.** The payer wallet needs only the USDC balance (≥ the challenge amount); no native token required.
- The settle payload is the full x402 v2 shape (`{x402Version, resource, accepted, payload, extensions}`) in the base64 `PAYMENT-SIGNATURE` header (Harbor) or `X-PAYMENT` (standard).
- The challenge's `network`/`asset`/`extra` are authoritative for the EIP-712 domain. On stage a family resolves to its testnet (e.g. `ethereum`→Sepolia); on prod, mainnet.
- A fresh challenge is fetched immediately before signing so the short validity window (`maxTimeoutSeconds`, ~60s) can't expire mid-flow.
- Settlement success is read from the settle `HTTP 200` body's `transactionHash`; you can also confirm via the payment intent's status (`created → checking_out → payment_processing → payment_completed`).

## Agent Response

**`--inspect`:** Summarize amount, payTo, network, status, and which KYC fields are required. Do not sign.

**Preview mode** (no `--confirm`): Summarize amount/asset/network/payTo, then ask whether to proceed with `--confirm`.

**Settle mode** (`--confirm`): Confirm payment sent; report `transactionHash` and network.

**Errors:** For `X402_KYC_REQUIRED`, read the `pay.requirements` schema and tell the user which fields to supply via `--kyc`/`--kyc-file`. For `X402_ENABLE_FAILED` / `X402_SETTLE_FAILED`, relay the server message verbatim. Never auto-retry a settle.
