# Transactions

List transaction history with optional filters and pagination, or view full details of a single transaction.

Requires login. `tx` is an alias for `transactions`.

The command has two operating modes:

- **Agent mode** — triggered by `--json`, non-TTY stdio, or `CLAUDECODE` / `CI` / `CURSOR_AGENT` env vars. `tx list` prints the flat paginated NDJSON / JSON envelope documented below. `tx detail` requires the positional `<id>` or throws `BusinessError('INPUT_REQUIRED')` (exit 3).
- **TTY human mode** — `tx list` opens an interactive paginated picker (10 per page, Next/Prev navigation). Selecting a row prints the detail and offers `[← Back to list / Exit]`. `tx detail` still requires an `<id>`; running it without one prints a friendly redirect to `owlp tx list`.

Agents should always invoke `owlp tx list --json` to guarantee the non-interactive path and the stable JSON shape below.

## Commands

```bash
owlp tx list --json                       # Paginated list (10 per page)
owlp tx list --type send --json           # Filter by type
owlp tx list --type receive --json
owlp tx list --page 2 --json              # Pagination
owlp tx list --wallet my-wallet --json    # Use a non-default wallet
owlp tx detail <id> --json                # Full details of one transaction (positional id required)
```

**Agent execution** — always capture output (SKILL.md § Output Discipline):

```bash
RESULT=$(owlp tx list --json 2>/dev/null) && echo "$RESULT" | jq -r '.data.transactions[] | "\(.created_at) \(.type) \(.from_amount) \(.from_currency)"'
```

`tx list` history types: `send`, `receive`, `withdraw`, `deposit`, `bridge` (the values the API accepts for `--type`).

`tx detail` envelope `data.type` values: `send`, `receive`, `withdraw`, `deposit`, `bridge` — the last three throw `UNSUPPORTED_TX_TYPE` (exit 3) and are reserved for follow-up CLI changes.

`tx list` paginates at 10 transactions per page. Navigate with `--page N` and inspect `meta.last_page` / `meta.total` in the response.

## JSON Responses

All responses are wrapped in `{success, env, data}`. Only the `data` payload is shown below.

### tx list

```json
{
  "transactions": [
    {
      "type": "send",
      "state": "completed",
      "source_chain": "stellar",
      "destination_chain": "stellar",
      "from_currency": "USDC",
      "from_amount": "10.00",
      "to_currency": "USDC",
      "to_amount": "10.00",
      "source_address": "G...",
      "destination_address": "G...",
      "gas_fee_display_currency": "XLM",
      "gas_fee_display_amount": "0.00002",
      "historable_order_serial": "TXN-20250101-001",
      "completed_at": "2025-01-01T00:00:00.000000Z",
      "created_at": "2025-01-01T00:00:00.000000Z"
    }
  ],
  "meta": {
    "current_page": 1,
    "last_page": 1,
    "total": 1
  }
}
```

Use `historable_order_serial` or the API id as the `<id>` argument for `tx detail`.

### tx detail — tagged envelope

`tx detail` returns a **tagged discriminated envelope** keyed on `data.type`. Branch on `data.type` and read the type-specific `data.detail` fields:

```json
{
  "type": "send" | "receive" | "deposit" | "withdraw" | "bridge",
  "detail": { /* per-type shape — see tables below */ }
}
```

Currently implemented: `send`, `receive`. The other four types throw `BusinessError('UNSUPPORTED_TX_TYPE')` with exit code 3 — they are reserved for future CLI changes.

#### Shared header (present on every `data.detail`)

| Field | Type | Description |
|---|---|---|
| `serial_number` | string | Transaction order serial (use as the `<id>` for `tx detail`) |
| `state` | string | Transaction state (e.g. `completed`, `confirmed`) |
| `completed_at` | string \| null | ISO timestamp when the transaction completed |
| `created_at` | string \| null | ISO timestamp when the transaction was created |
| `source_chain` | string \| null | Source blockchain name |
| `destination_chain` | string \| null | Destination blockchain name |
| `amount` | string | Transaction amount |
| `currency` | string | Currency symbol |
| `memo` | string \| null | Optional memo |
| `message` | string \| null | Optional message |

Missing values are `null`; keys are never omitted.

#### `data.type === "send"` body

`data.detail` extends the shared header with:

| Field | Type | Description |
|---|---|---|
| `source_address` | string | Sender address |
| `source_address_explorer` | string \| null | Sender explorer URL |
| `destination_address` | string | Recipient address |
| `destination_address_explorer_url` | string \| null | Recipient explorer URL |
| `destination_address_contact_name` | string \| null | Contact name if saved |
| `gas_fee_amount` | string \| null | Gas fee amount |
| `gas_fee_currency` | string \| null | Gas fee currency |
| `order` | object \| null | Wrapping object (see below) |
| `order.action` | string | e.g. `withdraw` |
| `order.state` | string | Order state |
| `order.fees` | string \| null | Order fees |
| `order.tx_hash_explorer_url` | string \| null | Explorer URL for the order's tx hash |
| `order.blockchain_transaction` | object \| null | Blockchain info (see below) |
| `order.blockchain_transaction.tx_hash` | string | On-chain transaction hash |
| `order.blockchain_transaction.state` | string | Chain-side state (e.g. `confirmed`) |
| `order.blockchain_transaction.block_height` | number \| null | Block height |
| `order.blockchain_transaction.confirm_blocks` | number \| null | Confirmations |
| `order.blockchain_transaction.transaction_time` | string \| null | ISO timestamp |
| `order.blockchain_transaction.explorer_url` | string | Explorer URL for the tx |

Explorer verification URLs for `send`: `data.detail.order.tx_hash_explorer_url` and `data.detail.order.blockchain_transaction.explorer_url`.

Example:

```json
{
  "type": "send",
  "detail": {
    "serial_number": "TXN-20250101-001",
    "state": "completed",
    "completed_at": "2025-01-01T00:00:00.000000Z",
    "created_at": "2025-01-01T00:00:00.000000Z",
    "source_chain": "ethereum",
    "destination_chain": "ethereum",
    "amount": "100",
    "currency": "USDC",
    "memo": null,
    "message": null,
    "source_address": "0xfrom",
    "source_address_explorer": "https://etherscan.io/address/0xfrom",
    "destination_address": "0xto",
    "destination_address_explorer_url": "https://etherscan.io/address/0xto",
    "destination_address_contact_name": null,
    "gas_fee_amount": "0.0042",
    "gas_fee_currency": "ETH",
    "order": {
      "action": "withdraw",
      "state": "completed",
      "fees": "0.5",
      "tx_hash_explorer_url": "https://etherscan.io/tx/0xhash",
      "blockchain_transaction": {
        "tx_hash": "0xhash",
        "state": "confirmed",
        "block_height": 12345,
        "confirm_blocks": 12,
        "transaction_time": "2025-01-01T00:00:00.000000Z",
        "explorer_url": "https://etherscan.io/tx/0xhash"
      }
    }
  }
}
```

#### `data.type === "receive"` body

`data.detail` extends the shared header with:

| Field | Type | Description |
|---|---|---|
| `from_address` | string | Sender address (NOT `source_address`) |
| `from_address_explorer_url` | string \| null | Sender explorer URL |
| `to_address` | string | Recipient address (NOT `destination_address`) |
| `to_address_explorer_url` | string \| null | Recipient explorer URL |
| `tx_hash_explorer_url` | string \| null | Explorer URL for the tx hash (top-level) |
| `gas_fee_amount` | string \| null | Gas fee amount |
| `gas_fee_currency` | string \| null | Gas fee currency |
| `blockchain_transaction` | object \| null | Top-level blockchain info (same fields as send's nested one) |

Explorer verification URLs for `receive`: `data.detail.tx_hash_explorer_url` and `data.detail.blockchain_transaction.explorer_url`.

Example:

```json
{
  "type": "receive",
  "detail": {
    "serial_number": "TXN-20250101-002",
    "state": "confirmed",
    "completed_at": "2025-01-01T00:05:00.000000Z",
    "created_at": "2025-01-01T00:04:00.000000Z",
    "source_chain": "stellar",
    "destination_chain": "stellar",
    "amount": "42",
    "currency": "USDC",
    "memo": null,
    "message": null,
    "from_address": "GSENDER",
    "from_address_explorer_url": "https://stellar.expert/sender",
    "to_address": "GRECEIVER",
    "to_address_explorer_url": "https://stellar.expert/receiver",
    "tx_hash_explorer_url": "https://stellar.expert/tx/abc",
    "gas_fee_amount": "0.00002",
    "gas_fee_currency": "XLM",
    "blockchain_transaction": {
      "tx_hash": "abc",
      "state": "confirmed",
      "block_height": 999,
      "confirm_blocks": 30,
      "transaction_time": "2025-01-01T00:05:00.000000Z",
      "explorer_url": "https://stellar.expert/tx/abc"
    }
  }
}
```

#### Unsupported types

When `type` resolves to `deposit`, `withdraw`, or `bridge`, `tx detail` throws `BusinessError('UNSUPPORTED_TX_TYPE')` (exit code 3) with a message naming the type. These will be added in follow-up CLI changes.

## Notes

- `meta.last_page > 1` means more pages are available — use `--page` to fetch them.
- Default page size is 10.
- Poll `tx detail <id>` after a `send --confirm` until `data.detail.state == "completed"` and `data.detail.order.blockchain_transaction.state == "confirmed"` to know the send has landed.
- `tx detail` requires the `<id>` positional argument in every mode. Agent mode throws `INPUT_REQUIRED` when omitted; TTY mode prints a friendly redirect to `owlp tx list`.
- `tx list` is interactive in TTY mode. Agents MUST pass `--json` (or rely on the auto-detected agent env / non-TTY triggers) to get the flat paginated output — otherwise the picker will block on input.

## Agent Response

### tx list

Present transactions as a list — each entry should include the date, type (send/receive), amount, counterparty address, and status. Mention current page and total pages. If more pages are available, offer to fetch the next page or view a specific transaction's detail.

### tx detail

Summarize the transaction in natural language. Highlight the key facts: type, amount, currency, counterparty, chain, status, and completion time. Include the explorer URL so the user can verify on-chain. Do not enumerate every field — focus on what matters.
