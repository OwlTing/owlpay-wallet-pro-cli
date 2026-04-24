# OwlPay Wallet Pro CLI

[English](README.md)

每個 Agent 都值得擁有一個錢包。給人類與 AI Agent 使用的 CLI 錢包 —— 私鑰在本地，資金你掌握。支援 EVM、Stellar、Solana。

> 本 Repository 是 `owlp` 的公開入口：Issue 追蹤器、Agent Skill 及文件。CLI 原始碼為閉源；二進位檔透過 npm 發佈。

## 開始使用

從零到錢包，三個步驟：

### 1. 安裝

需要 Node.js >= 20.12.0。

```bash
npm install -g @owlpay/owlp-cli
owlp -V   # 驗證安裝
```

### 2. 教會你的 Agent

```bash
npx skills add OwlTing/owlpay-wallet-pro-cli
```

支援 Claude Code、Cursor、Gemini CLI 及其他相容 skill 的 Agent。安裝後，Agent 即熟悉每個指令、flag 與 NDJSON 事件格式，無需翻閱文件。

### 3. 新手引導

```bash
owlp onboard
```

一次完成帳號建立（瀏覽器）、錢包建立（終端機）及 KYC 驗證（瀏覽器）。完成後即可收發加密貨幣。

> `owlp onboard` 為互動式指令，需要真實的終端機（TTY）。AI Agent 應改用 `owlp auth login` + `owlp wallet create --json` —— 詳見 [Agent Skill](skills/SKILL.md) 的 First-Run Checklist。

## 使用方式

```bash
owlp balance                # 查詢各鏈餘額
owlp send --to <addr> --amount 10 --token USDC --chain stellar
owlp tx list                # 交易紀錄
owlp status                 # 帳號、錢包、KYC 狀態
owlp --help                 # 所有指令
```

每個指令都支援 `--json` 取得機器可讀輸出。多步驟流程（`send`、`onboard`、`kyc submit`）以 NDJSON 事件串流。

## 主要特色

- **Agent 優先，不只是相容** —— 每個指令都有 `--json`，多步驟流程以 NDJSON 事件串流，自動偵測 Agent 模式
- **私鑰永不離開你的裝置** —— 助記詞在本地產生，交易在客戶端簽署
- **一組助記詞，三條鏈** —— 從單一種子衍生 EVM、Stellar、Solana 地址
- **只在必要時開瀏覽器** —— 登入與 KYC 走瀏覽器，其餘操作都在終端機完成

## Agent Skill

[`skills/`](skills/) 目錄包含完整的 Agent Skill —— 指令參考、JSON 回應格式、新手流程及常見工作流。Agent 透過閱讀這些檔案自主操作 `owlp`。

- [`skills/SKILL.md`](skills/SKILL.md) — 主 Skill：安裝、首次使用、環境、驗證、工作流
- [`skills/commands/*.md`](skills/commands/) — 各指令詳細參考（13 個檔案）

## Issues 與回饋

- **Bug 回報**：[開一個 Issue](../../issues/new?template=bug_report.yml)
- **功能建議**：[開一個 Issue](../../issues/new?template=feature_request.yml)
- **問題與討論**：[前往 Discussions](../../discussions)

請**絕對不要**在 Issue 或 Discussion 中張貼私鑰、助記詞或 Session Token。

## 貢獻

請參閱 [CONTRIBUTING.md](CONTRIBUTING.md) 了解回報 Bug、建議功能及貢獻 Skill 的指南。

## 授權

[Apache 2.0](LICENSE) — 涵蓋本 Repository 中的 skills 及文件。`owlp` CLI 二進位檔為專有軟體，透過 npm 另行發佈。
