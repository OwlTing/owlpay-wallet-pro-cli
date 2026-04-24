# Wallet Management

Requires login first. Each wallet derives three addresses from a single BIP39 mnemonic:
- Ethereum (EVM): `0x...`
- Stellar: `G...`
- Solana: Base58

## Commands

```bash
owlp wallet create [--name <name>] --json                       # Generate new wallet
owlp wallet import [--name <name>] [--mnemonic "<12 words>"]    # Import from 12-word mnemonic (alias: restore)
owlp wallet list --json                                         # List all wallets
owlp wallet rename <current> <new-name> --json                  # Rename a wallet
owlp wallet switch <name-or-address> --json                     # Change default wallet
owlp wallet export-key --chain <chain> [--yes] --json           # Export private key
owlp wallet reset [--force]                                     # Remove ALL local wallets
```

**Agent execution** — always capture output (SKILL.md § Output Discipline):

```bash
RESULT=$(owlp wallet list --json 2>/dev/null) && echo "$RESULT" | jq -r '.data.wallets[] | "\(.name) — \(.chains.ethereum)"'
```

> **Security note**: `export-key` prompts for confirmation unless `--yes` is passed. In **human mode**, `wallet import` reads the mnemonic interactively with a masked prompt (never pass it on the CLI — it would land in shell history). In **agent mode** (`--json` / non-TTY) there is no prompt — pass `--mnemonic "<12 words>"` or the command throws `MNEMONIC_REQUIRED` (exit 3). Never log or share the returned private key or mnemonic.
>
> **⚠️ Agent onboarding — wallet step requires explicit user consent.** Before running `wallet create` or `wallet import` as part of onboarding, display a security warning and confirm the user wants agent-driven setup. Recommend they run `owlp wallet create` / `owlp wallet import` themselves in a TTY instead — this keeps the mnemonic off the network and out of AI context. `wallet create` is safer in agent mode (only addresses are returned; mnemonic is shown locally). Running `wallet import --mnemonic ...` as an agent exposes the mnemonic in AI context — only do this if the user explicitly accepts that risk.

## JSON Responses

All responses are wrapped in `{success, env, data}`. Only the `data` payload is shown below.

### wallet create / wallet import

```json
{
  "name": "wallet-abc123",
  "chains": {
    "ethereum": "0x...",
    "stellar": "G...",
    "solana": "Base58..."
  },
  "mnemonic": "word1 word2 ... word12"
}
```

### wallet list

```json
{
  "default": "0x...",
  "wallets": [
    {
      "name": "my-wallet",
      "address": "0x...",
      "mnemonic": "word1 word2 ...",
      "chains": {
        "ethereum": "0x...",
        "stellar": "G...",
        "solana": "Base58..."
      },
      "created_at": "2025-01-01T00:00:00.000Z"
    }
  ]
}
```

`default` is the EVM address of the current default wallet. Compare against each `wallets[].address` to find it.

### wallet rename

```json
{ "renamed": "old-name", "to": "new-name" }
```

### wallet switch

```json
{ "default": "my-wallet", "address": "0x..." }
```

### wallet export-key

```json
{ "privateKey": "0x...", "chain": "ethereum" }
```

Supported `--chain` values: `ethereum`, `stellar`, `solana`.

### wallet reset

```json
{ "message": "Wallets reset." }
```

**Destructive**: removes `wallets.json` from the current env. Requires typing `confirmed` in TTY + human mode, or `--force` in `--json` / non-TTY mode. Without either, the command exits with `DESTRUCTIVE_REQUIRES_CONFIRMATION` (exit 3). Mnemonics cannot be recovered after this — back up first.

## Agent Response

### wallet create

**Before executing:** Warn the user that creating a wallet generates a 12-word mnemonic — the only way to recover this wallet. If lost, funds are permanently inaccessible. Ask the user to confirm they are in a safe environment and ready to record the mnemonic before proceeding.

**After executing:** Display the wallet name, the mnemonic, and all three chain addresses. Remind the user to back up the mnemonic immediately — it will not be shown again.

### wallet import

**Before executing:** Warn the user that entering a mnemonic in this environment may leave it in logs or conversation history. Ask the user to confirm they understand the risk before proceeding.

**After executing:** Confirm the wallet was imported, show the wallet name and chain addresses. Do not echo the mnemonic back.

### wallet export-key

**Before executing:** Warn the user that exporting a private key is a high-risk operation — if the key is leaked, all assets on that chain can be stolen with no recourse. Ask the user to confirm they understand the risk.

**After executing:** Display the private key and remind the user not to share it with anyone.

### wallet list

Introduce how many wallets exist and which one is the default, then list them by name. Include creation dates if useful for context.
