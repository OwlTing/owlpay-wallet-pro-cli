# OwlPay Wallet Pro CLI

[繁體中文](README.zh-TW.md)

Every agent deserves a wallet. A CLI wallet for humans and AI agents — keys stay local, funds stay yours. EVM, Stellar, Solana.

> This repository is the public home for `owlp`: issue tracker, agent skill, and documentation. The CLI source is closed-source; the binary is distributed via npm.

## Get Started

Three steps from zero to wallet:

### 1. Install

Requires Node.js >= 20.12.0.

```bash
npm install -g @owlpay/owlp-cli
owlp -V   # verify installation
```

### 2. Teach your agent

```bash
npx skills add OwlTing/owlpay-wallet-pro-cli
```

Works with Claude Code, Cursor, Gemini CLI, and other skill-compatible agents. Once installed, your agent knows every command, flag, and NDJSON event shape — no README deep-dive needed.

### 3. Onboard

```bash
owlp onboard
```

Creates your account (browser), wallet (terminal), and KYC record (browser) in one guided flow. After onboarding, you're ready to send and receive crypto.

> `owlp onboard` is interactive and requires a real terminal (TTY). AI agents should use `owlp auth login` + `owlp wallet create --json` instead — see the [agent skill](skills/SKILL.md) for the full first-run checklist.

## Usage

```bash
owlp balance                # Balances across all chains
owlp send --to <addr> --amount 10 --token USDC --chain stellar
owlp tx list                # Transaction history
owlp status                 # Account, wallet, and KYC readiness
owlp --help                 # All commands
```

Every command accepts `--json` for machine-readable output. Multi-step flows (`send`, `onboard`, `kyc submit`) stream NDJSON events.

## Key Features

- **Agent-first, not agent-tolerant** — `--json` on every command, NDJSON event streams for multi-step flows, auto-detected agent mode
- **Keys never leave your machine** — mnemonic generated locally, transactions signed client-side
- **One mnemonic, three chains** — EVM, Stellar, Solana addresses derived from a single seed
- **Browser only when it has to** — login and KYC open the browser; everything else stays in terminal

## Agent Skill

The [`skills/`](skills/) directory contains the full agent skill — command reference, JSON response shapes, onboarding flow, and common workflows. Agents read this to operate `owlp` autonomously.

- [`skills/SKILL.md`](skills/SKILL.md) — Main skill: installation, first-run checklist, environment, authentication, workflows
- [`skills/commands/*.md`](skills/commands/) — Per-command reference (13 files)

## Issues & Feedback

- **Bug reports**: [Open an issue](../../issues/new?template=bug_report.yml)
- **Feature requests**: [Open an issue](../../issues/new?template=feature_request.yml)
- **Questions & help**: [Start a discussion](../../discussions)

Please **never** include private keys, mnemonics, or session tokens in issues or discussions.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on reporting bugs, requesting features, and contributing skills.

## License

[Apache 2.0](LICENSE) — covers the skills and documentation in this repository. The `owlp` CLI binary is proprietary and distributed separately via npm.
