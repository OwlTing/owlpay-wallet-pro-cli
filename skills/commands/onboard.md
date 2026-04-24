# Onboard

Guided setup that walks a new user through three steps:

1. **Account** — creates an auth session on `owlp-cli-service`, opens the browser to the login/sign-up page, and waits for authentication. In TTY mode, polls automatically; in agent mode (`--json`), pauses after emitting a `loginUrl` and resumes when `--auth-code` is provided. Skipped if already logged in.
2. **Wallet** — an in-terminal clack prompt that asks whether to **create** a new BIP39 wallet or **import** from a 12-word mnemonic. If wallets already exist, asks whether to add another one or keep the existing default.
3. **KYC** (optional) — a confirm prompt asks whether to complete KYC now or skip. If the user declines (or `--skip-kyc` is set), Step 3 is skipped and the flow finishes successfully. If accepted, opens the browser to `/kyc` for identity verification, then polls `integrated-status` until a terminal state. Skipped entirely when KYC is already `verified`.

`onboard` supports both TTY mode (interactive) and agent mode (`--json`). In agent mode, `owlp onboard` only handles account auth (Step 1); wallet and KYC are orchestrated by the agent via separate commands — see **Agent Mode Flow** below for the end-to-end workflow.

## Commands

```bash
owlp onboard                               # TTY: guided interactive flow
owlp onboard --skip-kyc                    # TTY: skip KYC step (finish after wallet setup)
owlp onboard --json                        # Agent: start session, emit loginUrl, exit 3
owlp onboard --json --auth-code WOLF-3842  # Agent: claim passcode, complete auth
```

| Flag | Description |
|------|-------------|
| `--skip-kyc` | Skip the KYC verification step entirely. The flow completes after wallet setup (exit 0). Useful for users who only need crypto-to-crypto features. |
| `--auth-code <CODE>` | Passcode obtained from the browser after login (agent mode only). Format: `WORD-NNNN`. |

The `cliServiceUrl` is picked up from the active env's endpoints (resolved via `--stage` / `config.json` / default).

## TTY Mode Flow

In TTY mode, Step 1 works as follows:
1. Calls `POST /api/auth/sessions` to create a session
2. Opens the browser to the returned `loginUrl`
3. Polls `GET /api/auth/sessions/:id` every 3 seconds for up to 10 minutes
4. When tokens arrive (`200 OK`), writes `auth.json` and continues to wallet setup

## Agent Mode Flow

**When the user asks an agent to onboard, the agent should drive the full end-to-end flow below.** The CLI's `owlp onboard --json` only handles Step 1 (auth); the agent continues with wallet and KYC via separate commands. The `--skip-kyc` flag only applies to TTY mode.

### Step 1 — Account Auth (two CLI invocations)

**First invocation** — `owlp onboard --json`:
- Emits `{"type":"auth.code_required","loginUrl":"<url>","sessionId":"<id>"}`
- Exits with code 3 (`AUTH_CODE_REQUIRED`)
- Display the `loginUrl` and ask the user to open it, complete login, and read the passcode shown in the browser (format: `WORD-NNNN`, e.g. `WOLF-3842`)

**Second invocation** — `owlp onboard --json --auth-code WOLF-3842`:
- Claims the passcode, writes tokens to `auth.json`
- Emits `{"type":"auth.done","email":"<email>"}` then `{"type":"complete","result":{"email":"<email>"}}`
- Exits 0 — auth is done. **Continue to Step 2.**

**Invalid or expired passcode** — exits code 3 with:
```json
{"type":"error","code":"INVALID_AUTH_CODE","message":"..."}
```

**Already authenticated** — skips auth and exits 0 immediately. Proceed to Step 2.

### Step 2 — Wallet Setup

**Immediately after auth completes**, check whether a wallet already exists and set one up if not.

Display the security warning from § Wallet Step Security Warning below. Then:

```bash
# Check existing wallets
RESULT=$(owlp wallet list --json 2>/dev/null) && echo "$RESULT" | jq '.data.wallets | length'

# If no wallets exist, create one (agent-safe — mnemonic shown locally, not returned):
RESULT=$(owlp wallet create --json 2>/dev/null) && echo "$RESULT" | jq '.data | {name, chains}'
```

If the user prefers to import, strongly recommend they run `owlp wallet import` themselves in a TTY (mnemonic stays off the network). Only run `owlp wallet import --json --mnemonic "<12 words>"` if the user explicitly accepts the risk.

See `skills/commands/wallet.md` for full JSON responses and the Agent Response guidance.

**After wallet setup completes, continue to Step 3.**

### Step 3 — KYC (ask user, then two CLI invocations)

**Ask the user:** "Do you want to complete KYC now? It's only needed for fiat deposit/withdrawal — crypto transfers work without it."

If the user declines, report onboarding as complete and mention they can run `owlp kyc submit` later.

If the user agrees:

**3a.** Run `owlp kyc submit --json` — emits the KYC browser URL then exits:
```
{"type":"kyc.browser_required","url":"https://<cliServiceUrl>/app/kyc?access_token=...","ts":...}
# exits 3 — code: KYC_BROWSER_REQUIRED
```
Display the `url` and ask the user to open it and complete identity verification.

**3b.** When the user confirms they have finished in the browser, run `owlp kyc wait --json`:
```
{"type":"kyc_poll.progress","attempt":1,"status":"finished","ts":...}
...
{"type":"kyc_poll.done","finalStatus":"verified","ts":...}
{"type":"complete","result":{"status":"verified","verified":true},"ts":...}
```

**Agent execution** — capture the final result:
```bash
RESULT=$(owlp kyc wait --json 2>/dev/null) && echo "$RESULT" | tail -1 | jq '.result | {status, verified}'
```

Terminal states that stop polling: `verified`, `rejected`, `banned`, `declined`, `revoked`.

### Summary

After all steps complete, report onboarding status:
- Account: logged in as (email)
- Wallet: (name) with EVM/Stellar/Solana addresses
- KYC: verified / skipped (mention `owlp kyc submit` if skipped)

## Wallet Step Security Warning (Agent Mode)

**Before proceeding to Step 2 (wallet create / import), you MUST display this warning and obtain explicit user confirmation.**

> ⚠️ **Security warning — wallet setup**
>
> The next step creates or imports a wallet. This involves a BIP39 mnemonic (seed phrase) that controls real funds.
>
> **Strongly recommended:** exit agent mode and run this step yourself in your own terminal:
> ```bash
> owlp wallet create       # generates a new wallet, shows mnemonic for you to back up
> owlp wallet import       # lets you type the mnemonic with a masked prompt
> ```
> Running these commands yourself keeps the mnemonic off the network and out of AI context.
>
> If you continue via agent, `wallet create` is safer (the agent never receives the mnemonic — only addresses). `wallet import` requires the agent to receive your mnemonic directly, which is a significant security risk.
>
> **Do you want to proceed with agent-driven wallet setup, or will you run `owlp wallet create` / `owlp wallet import` yourself?**

Only proceed to Step 2 if the user explicitly confirms they want the agent to handle it. If they choose to do it themselves, pause and wait for them to confirm they have completed wallet setup before continuing.

## Output Format

**TTY mode:** Each step is a clack `note` card (`Step 1/3 · Account`, `Step 2/3 · Wallet`, `Step 3/3 · KYC Verification`), with a spinner whose label updates from stream events (`Opening browser…`, `Polling for sign-in…`, `Reviewing (attempt N)…`, etc.). When a browser is opened, the URL is printed to stderr so the user can copy/paste if the auto-open failed. The flow ends with a `Setup Complete` note summarizing account, wallet, and KYC status.

**Agent mode** (`owlp onboard --json`): Emits NDJSON (one JSON object per line). Only auth events are emitted — wallet and KYC are separate commands:
- `{"type":"meta.env","env":"prod","endpoints":{...}}` — prepended automatically
- First invocation: `{"type":"auth.code_required","loginUrl":"<url>","sessionId":"<id>"}` then exits 3
- Second invocation (after `--auth-code`): `{"type":"auth.done","email":"<email>"}` then `{"type":"complete","result":{"email":"<email>"}}` then exits 0

## When to Use

- **First-time users** on a fresh machine. One command replaces: `auth login` → `wallet create` → (optionally) `kyc submit`.
- **Crypto-only users** — run `owlp onboard --skip-kyc` to skip identity verification and get started with crypto transfers immediately.
- **Agent workflows** — use `owlp onboard --json` (two-invocation flow; agent displays the `loginUrl`, user opens browser and reads the passcode, agent calls `--auth-code <CODE>` to complete).
- **Tutorials or screencasts** where the interactive prompts are the demo.

## When NOT to Use

- **Re-running setup** — when already set up, onboard still prompts for KYC unless KYC is already `verified` (or `--skip-kyc` is passed). Use individual commands instead.
- **KYC-only setup** — if account and wallet already exist but KYC is needed, use `owlp kyc submit` directly.

## Notes

- SIGINT at any step maps to `BusinessError('USER_CANCELLED')` (exit 3).
- Passcode format: `WORD-NNNN` (4-letter uppercase word + hyphen + 4 digits, e.g. `WOLF-3842`). Passcodes are one-time use with a 10-minute TTL.
- Polling timeout in TTY mode: 10 minutes without a `200` response → timeout error.
- If a multi-account situation is detected during Step 1 (local `wallets.json` was created under a different `uuid`), a `warning` event with code `OWNER_UUID_MISMATCH` is yielded. Login still succeeds.
- **TTY mode only:** For Step 2 **create**, the user must type `confirmed` in a follow-up prompt to acknowledge the mnemonic has been backed up before the flow proceeds.
- **TTY mode only:** Step 3 is preceded by a confirm prompt: "KYC is only needed for fiat deposit/withdrawal. Complete now?" If the user declines or `--skip-kyc` is set, the flow skips to the summary with KYC shown as incomplete and a hint to run `owlp kyc submit` later. When Step 3 proceeds, polling happens automatically.
- **Agent mode:** Steps 2 and 3 are not part of `owlp onboard --json`. The agent orchestrates them via `owlp wallet create --json`, `owlp kyc submit --json`, and `owlp kyc wait --json` — see **Agent Mode Flow** above.
- `owlp-cli-service` must be running at the configured `cliServiceUrl` for the browser flows in Steps 1 and 3 to work.
