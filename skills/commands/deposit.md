# Deposit (on-ramp)

Fund an OwlPay wallet from external sources. The first available method is **debit-card** (VDC = Virtual Debit Card on-ramp). The CLI is provider-shaped; future methods will appear under the same command tree.

The flow has four agent-facing stages:

1. `owlp deposit card add` — bind a card via browser handoff (one-time per card)
2. `owlp deposit quote` — fetch a USD→USDC quote
3. `owlp deposit submit` — create the deposit transaction (gated by `--confirm`)
4. `owlp deposit watch <tx-uuid>` — poll the transaction until it reaches a terminal state

`owlp deposit` is the provider-shaped human orchestrator for the same flow. `owlp deposit start` remains as a compatibility alias. In TTY human mode it is a step-by-step wizard. The first step lists local wallet chain destinations from the user's bound wallets, for example `main · Stellar · GABC...1234`; selecting one fills both `--chain` and `--destination-address`. Missing amount mode/value is collected before quote. For debit-card, the wizard selects an active usable card automatically only when exactly one exists; when multiple active cards exist, it asks the user to choose the funding card. The quote preview displays the selected funding card summary before asking `Submit this deposit?`; declining stops without submitting. In this milestone only `debit-card` is registered, so `--method` may be omitted and the CLI resolves to the sole provider. When multiple providers are registered in a future release, agent mode must pass `--method`; the CLI returns `INPUT_REQUIRED` instead of silently choosing one. If no active usable card exists, it exits with `INPUT_REQUIRED` and tells the user to run `owlp deposit card add`.

In this MVP, deposits are limited to user-owned wallet destinations. `--destination-address` is accepted only when it matches a chain address from the local CLI wallet list for the selected chain. Arbitrary external deposit addresses are intentionally not exposed yet.

Requires login. Submitting a deposit requires KYC verified — backend returns `KYC_REQUIRED` (HTTP 403) otherwise.

Environment handling follows the global CLI contract: `--stage`, `owlp env set stage`, or default `prod` select the active Wallet Pro API, cli-service URL, local auth, wallets, and idempotency namespace. Deposit examples are safe to validate on stage first, but command behavior must not rely on stage-only endpoints.

## Operating modes

- **Agent mode** — `--json`, non-TTY stdio, or `CLAUDECODE` / `CI` / `CURSOR_AGENT` env. NDJSON event streams; missing required flags throw `INPUT_REQUIRED` (exit 3).
- **TTY human mode** — same flag set, but `card add` opens the browser and prints a URL fallback to stderr.

## Currency contract (verified)

- Source currency is **always `USD`** (server validates `in:USD`).
- Destination currency is **always `USDC`** (server validates `in:USDC`).
- Amounts: pass `--source-amount <usd>` or `--destination-amount <usdc>`. They are mutually exclusive; one is required.

## Subcommand: `deposit card add`

| Option | Description |
|--------|-------------|
| `--success-url <url>` | Optional redirect on successful binding |
| `--failed-url <url>` | Optional redirect on binding failure |
| `--no-browser` | TTY only: skip auto-open |
| `--idempotency-key <key>` | Replay-safe key (default: random UUID; server has `idempotency.key` middleware) |
| `--timeout <ms>` | Polling timeout in ms (default: 1800000 = 30 min, matches `CARD_BINDING_LINK_TTL`) |

NDJSON sequence (TTY happy path):

```
{"type":"meta.env",...}
{"type":"card.binding_url","url":"https://...","card_id":"<uuid>","ts":...}
{"type":"card.binding_polling","attempt":1,"status":"pending","ts":...}
{"type":"card.binding_polling","attempt":2,"status":"pending","ts":...}
{"type":"card.binding_active","card_id":"<uuid>","last4_digits":"4242","card_company":"VISA","ts":...}
{"type":"complete","result":{"cardId":"<uuid>","status":"active","last4Digits":"4242","cardCompany":"VISA"},"ts":...}
```

Agent mode cannot perform the browser interaction. It emits the binding URL and exits:

```
{"type":"meta.env",...}
{"type":"card.binding_url","url":"https://...","card_id":"<uuid>","ts":...}
{"type":"card.browser_required","url":"https://...","ts":...}
{"type":"error","code":"CARD_BINDING_BROWSER_REQUIRED","message":"Card binding requires browser interaction. Visit the URL, complete card binding, then run `owlp deposit card list`.","exitCode":3,"ts":...}
```

Failure terminal: `expired | disabled | restricted` → `card.binding_failed` → exit 3 `CARD_BINDING_FAILED`.

When the CLI service is local (`localhost` / `127.0.0.1`), default success and failed redirect URLs include a browser-return session under `/app/deposit/card/{success,failed}`. The browser page records whether the bank page returned success or failed, and the terminal emits:

```
{"type":"card.browser_returned","result":"success","ts":...}
```

This event is diagnostic only. Wallet Pro `GET /v1/cards` remains the final source of truth; if the browser returns failed but Wallet Pro still reports `pending`, the CLI continues polling until an active or terminal card status appears, or until timeout.

## Subcommand: `deposit card list`

| Option | Description |
|--------|-------------|
| `--page <n>` | Page number (Laravel pagination) |
| `--per-page <n>` | Items per page |

Returns the standard `{success, env, data}` envelope. PII fields (`bin_number`, `first_name`, `last_name`) are stripped before output.

## Human orchestrator: `deposit`

`owlp deposit` is the preferred TTY entrypoint. `owlp deposit start` accepts the same options and is retained as an alias for scripts, docs, and users who already learned the explicit verb.

| Option | Required (agent) | Description |
|--------|------------------|-------------|
| `--method <method>` | No while only one provider exists | Provider key. Required once multiple providers are available. |
| `--chain <chain>` | Yes | Destination chain (e.g. `ethereum`, `stellar`, `solana`) |
| `--source-amount <usd>` | One of these | USD amount to spend |
| `--destination-amount <usdc>` | One of these | USDC amount to receive |
| `--destination-wallet <id>` | One of these | Wallet Pro internal wallet uuid only |
| `--destination-address <addr>` | One of these | Local CLI wallet chain address only |
| `--confirm` | No | Submit after quote preview; without this, TTY asks after showing the preview and agent mode returns a dry-run preview |

In human TTY mode, omitted destination fields are prompted from local wallet chain addresses. Selecting a local CLI wallet fills `--destination-address`, not `--destination-wallet`.

In agent mode, prefer `--destination-address <chain-address>` when depositing to a wallet created or imported by this CLI. The address must match `owlp wallet list` for the selected chain, otherwise the CLI returns `DESTINATION_ADDRESS_NOT_LOCAL` before quote/submit work starts. Do **not** pass a local wallet name, shortened label, or local wallet address alias to `--destination-wallet`; Wallet Pro treats that flag as an internal wallet uuid and may return `WALLET_FORBIDDEN` when the value is not a workspace wallet uuid. Use `--destination-wallet` only when you already have a Wallet Pro internal wallet uuid from the API.

The wizard always displays the quote preview before asking whether to submit, unless `--confirm` was explicitly supplied. In agent mode, omitted required fields return `INPUT_REQUIRED`; agent mode never prompts.

Agent-mode happy-path preview:

```
{"type":"meta.env",...}
{"type":"deposit.start.method_selected","method":"debit-card","ts":...}
{"type":"quote.start","method":"debit-card","chain":"stellar","ts":...}
{"type":"quote.done","quote_id":"q-...","source_amount":"500","destination_amount":"498.50","effective_rate":"0.997000","expires_at":"2026-05-05T12:00:00Z","ts":...}
{"type":"submit.preview","quote_id":"q-...","chain":"stellar","destination_type":"external_address","destination_address":"G...","card_id":"card-...","card_last4_digits":"9904","card_company":"VISA","ts":...}
{"type":"complete","result":{"txUuid":"","state":"init","preview":{...}},"ts":...}
```

`deposit` does not hide provider setup inside the generic command. For debit-card, card binding remains explicit through `deposit card add`; future methods should add their own provider setup instead of hardcoding card behavior in the generic orchestrator.

If multiple providers exist in a future release and no method is supplied in agent mode, return `INPUT_REQUIRED` rather than prompting or silently selecting one.

## Subcommand: `deposit quote`

| Option | Required (agent) | Description |
|--------|------------------|-------------|
| `--chain <chain>` | Yes | Destination chain (e.g. `ethereum`, `stellar`, `solana`) |
| `--source-amount <usd>` | One of these | USD amount to spend |
| `--destination-amount <usdc>` | One of these | USDC amount to receive |
| `--method <method>` | No while only one provider exists | Provider key. Required once multiple providers are available. |
| `--source-country <iso2>` | No | 2-letter country code |
| `--source-type <type>` | No | `individual` or `business` |

NDJSON sequence (happy path):

```
{"type":"meta.env",...}
{"type":"quote.start","method":"debit-card","chain":"stellar","ts":...}
{"type":"quote.done","quote_id":"q-...","source_amount":"500","destination_amount":"498.50","effective_rate":"0.997000","expires_at":"2026-05-05T12:00:00Z","ts":...}
{"type":"complete","result":{...},"ts":...}
```

When the upstream method is unavailable, server returns `success: true, data: {}`. The CLI maps that to:

```
{"type":"error","code":"DEPOSIT_METHOD_UNAVAILABLE","message":"Deposit method 'debit-card' is unavailable for this request.","exitCode":3,"ts":...}
```

When Bank Module rejects a quote with an amount-limit message but Wallet Pro does not return the numeric limit, the CLI maps it to a stable actionable error instead of surfacing the raw upstream string:

```
{"type":"error","code":"DEPOSIT_AMOUNT_BELOW_MINIMUM","message":"This amount is below the debit-card minimum for ethereum.","probableCause":"amount_below_minimum","limitKnown":false,"hintAction":"Enter a higher amount or try another chain.","exitCode":3,"ts":...}
```

The matching high-side error is `DEPOSIT_AMOUNT_ABOVE_MAXIMUM`. The CLI must not invent or hardcode debit-card limits. Agent-mode errors include `limitKnown:false` when no numeric limit is available.

In TTY human `owlp deposit` / `owlp deposit start`, these amount-limit quote errors are retryable: the wizard explains the issue and asks for a replacement amount, then re-runs quote with the same destination, method, funding-source, and preview/submit choice. Low-level `owlp deposit quote` remains fail-fast.

Server intentionally strips fee fields from the response (`DebitCardQuoteController.php:71–75`); the CLI shows source/destination amounts and a client-computed effective rate only.

## Subcommand: `deposit submit`

| Option | Required | Description |
|--------|----------|-------------|
| `--quote-id <id>` | Yes | Quote id from `deposit quote` |
| `--card-id <id>` | Yes | Card id from `deposit card add` / `card list` |
| `--chain <chain>` | Yes | Must match the quote (server overrides via `prepareForPipeline`, but validation requires it) |
| `--destination-wallet <id>` | One of these | Wallet Pro internal wallet uuid only (sets `destination_type=internal_address`) |
| `--destination-address <addr>` | One of these | Local CLI wallet chain address only (sets `destination_type=external_address`) |
| `--method <method>` | No while only one provider exists | Provider key. Required once multiple providers are available. |
| `--idempotency-key <key>` | No | Replay-safe key (default: random UUID; **client-side**, server has no dedup middleware on `POST /transaction`) |
| `--confirm` | No | Without this, returns dry-run preview (no API call) |

Without `--confirm`, NDJSON emits a `submit.preview` event and `complete.result.preview` describes the planned request. With `--confirm`:

```
{"type":"meta.env",...}
{"type":"submit.validating","ts":...}
{"type":"submit.done","tx_uuid":"...","state":"created","ts":...}
{"type":"complete","result":{"txUuid":"...","serialNumber":"SN-...","state":"created"},"ts":...}
```

If the same `--idempotency-key` is replayed with identical args within 24h, the cached envelope is returned with `idempotentReplay: true`. Same key + different args → exit 3 `IDEMPOTENCY_KEY_REUSED`.

Errors mapped from verified backend exceptions:

| CLI code | Backend trigger |
|---|---|
| `KYC_REQUIRED` | HTTP 403 (KYC guard) |
| `WALLET_FORBIDDEN` | HTTP 403 (wallet ownership) |
| `QUOTE_NOT_FOUND` | `1014400003` `QuoteNotFoundException` (cache miss / expired) |
| `CARD_INVALID` | HTTP 400 `InvalidLinkedCardException` |
| `DAILY_LIMIT_EXCEEDED` | `1003400005` (payload includes `total_value`, `daily_limit`) |
| `ADDITIONAL_DATA_REQUIRED` | `1014400001` |

## Subcommand: `deposit watch <tx-uuid>`

| Option | Description |
|--------|-------------|
| `--timeout <ms>` | Total timeout in ms (default: 3600000 = 60 min) |

Polls `GET /v1/transaction/<uuid>` every 3000 ms. Each poll yields a `watch.progress` event with the verified `state` (one of: `init, created, failed, in_review, chain_in_process, wait_for_funds_confirmation, wait_for_user_offline_action, fiat_in_process, completed, refunded, information_required`).

Terminal mapping:

| Final state | Exit code | CLI code |
|---|---|---|
| `completed` | 0 | — |
| `failed` | 3 | `DEPOSIT_FAILED` (message = `failure_reason` when present) |
| `refunded` | 3 | `DEPOSIT_REFUNDED` |
| `information_required` OR `is_rfi_required=true` | 3 | `RFI_REQUIRED` (preceded by `deposit.rfi_required` event) |

`awaiting_fi_approval` is bank-module side, not Wallet Pro — the CLI does **not** emit any event for it.

## Cancellation

`SIGINT` aborts polling cleanly and emits `{"type":"cancelled","reason":"user_interrupt"}` (exit 130).

## Agent Response Guidance

- Prefer running one direct `owlp ... --json` command and reading the NDJSON events from the command output. Do not wrap normal user-facing runs in multi-line shell snippets with `OUT=...`, `CODE=$?`, `printf`, or `jq`; those wrappers are noisy in visible agent terminals. Use `jq` only when the user explicitly asks for machine extraction or when a script needs it.
- For `deposit card add --json`, surface the `card.browser_required.url` to the user and explain that the command exits with `CARD_BINDING_BROWSER_REQUIRED` until a human completes the hosted browser flow.
- **`card add` complete**: Confirm binding succeeded, name the card (`last4_digits` + `card_company`), and suggest `owlp deposit quote` next.
- **`quote.done`**: Report `source_amount → destination_amount`, `effective_rate`, `expires_at`. Suggest `submit --quote-id <q> --confirm`.
- **`submit.done`** (with `--confirm`): Report `tx_uuid`, suggest `owlp deposit watch <tx-uuid>` to follow it.
- **`watch.done` completed**: Report `tx_hash` and that funds are confirmed.
- **`RFI_REQUIRED`**: Tell the user the deposit needs additional information; this milestone does not render the form natively.
- **`DEPOSIT_METHOD_UNAVAILABLE`**: Backend has no `DEBIT_CARD` quote available for the requested chain/amount/source — suggest checking workspace-level VDC enablement or trying a different chain.
- **`DEPOSIT_AMOUNT_BELOW_MINIMUM` / `DEPOSIT_AMOUNT_ABOVE_MAXIMUM`**: Explain that the requested amount is outside the debit-card limit for the selected chain. Suggest entering a higher/lower amount or trying another chain.

## Notes

- This command tree is provider-shaped; only `debit-card` is registered in this milestone.
- Card resource PII (`bin_number`, `first_name`, `last_name`) is stripped at the service layer.
- Server-side `POST /cards` has `idempotency.key` middleware; `POST /transaction` does not — the CLI maintains client-side dedup.
- Currency pair USD/USDC is hardcoded by the adapter (server enum-restricted).
