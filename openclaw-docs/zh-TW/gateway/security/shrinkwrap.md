---
read_when:
    - 你想了解 npm shrinkwrap 在 OpenClaw 版本發布中的含義
    - 你正在審查套件鎖定檔、相依性變更或供應鏈風險
    - 你正在驗證根 npm 套件或外掛 npm 套件，準備發布
summary: OpenClaw 發行版本中 npm shrinkwrap 的白話與技術說明
title: npm shrinkwrap
x-i18n:
    generated_at: "2026-07-26T08:19:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d1e6c0d4541da9220d50cde0b9db064e5a91b81d6562cb16ac697de7d4017098
    source_path: gateway/security/shrinkwrap.md
    workflow: 16
---

OpenClaw 原始碼簽出使用 `pnpm-lock.yaml`。已發布的 OpenClaw npm 套件使用 `npm-shrinkwrap.json`，也就是 npm 可發布的相依性鎖定檔，因此套件安裝會使用發布期間已審查的相依性圖。

## 為何重要

Shrinkwrap 是隨 npm 套件發布之相依性樹的明細：它會告訴 npm 應安裝哪些確切的遞移相依性版本。

| 檔案                  | 適用情境         | 意義                     |
| --------------------- | ------------------------ | --------------------------------- |
| `pnpm-lock.yaml`      | OpenClaw 原始碼簽出 | 維護者相依性圖       |
| `npm-shrinkwrap.json` | 已發布的 npm 套件    | 使用者的 npm 安裝圖       |
| `package-lock.json`   | 本機 npm 應用程式           | 並非 OpenClaw 的發布契約 |

對 OpenClaw 發布而言，這表示：

- 已發布的套件不會要求 npm 在安裝時臨時產生新的相依性圖；
- 相依性變更可供審查，因為它們會呈現在鎖定檔差異中；
- 發布驗證所測試的相依性圖，與使用者將安裝的相同；
- 套件大小或原生相依性的意外情況會在發布前浮現。

Shrinkwrap 並非沙箱。它本身不會讓相依性變得安全，也無法取代主機隔離、`openclaw security audit`、套件來源證明或安裝煙霧測試。

OpenClaw 是閘道、外掛主機、模型路由器與代理程式執行階段，因此預設安裝會影響啟動時間、磁碟使用量、原生套件下載及供應鏈暴露風險。Shrinkwrap 為發布審查提供穩定的邊界：審查者可查看遞移相依性的變動，驗證工具會拒絕非預期的鎖定檔漂移，而外掛套件會攜帶各自鎖定的相依性圖，不必依賴根套件。

## 產生與檢查

根 `openclaw` npm 套件、OpenClaw 擁有的 npm 外掛套件（例如 `@openclaw/discord`），以及 [`@openclaw/ai`](/zh-TW/reference/openclaw-ai) 等可發布的工作區套件，會在發布時包含 `npm-shrinkwrap.json`。工作區相依性不會納入根 shrinkwrap，因為它們會與根套件一同發布；每個可發布的工作區套件則會分別鎖定自己的遞移相依性樹。適合的外掛套件也可使用明確的 `bundledDependencies` 進行發布，將其執行階段相依性檔案納入外掛 tarball，而非僅依賴安裝時解析。

```bash
# 所有由 shrinkwrap 管理的套件（根套件 + 可發布的外掛）
pnpm deps:shrinkwrap:generate
pnpm deps:shrinkwrap:check

# 僅根套件
pnpm deps:shrinkwrap:root:generate
pnpm deps:shrinkwrap:root:check

# 僅受目前變更集影響的套件
pnpm deps:shrinkwrap:changed:generate
pnpm deps:shrinkwrap:changed:check
```

產生器會解析 npm 的可發布鎖定格式，但會拒絕產生任何尚未存在於 `pnpm-lock.yaml` 中的套件版本。這能維持 pnpm 相依性版本新舊程度、覆寫與修補程式的審查邊界。

請將以下項目視為安全性敏感項目進行審查：

- `pnpm-lock.yaml`
- `npm-shrinkwrap.json`
- 隨附外掛的相依性內容
- 任何 `package-lock.json` 差異

OpenClaw 套件驗證工具要求新的根套件 tarball 包含 shrinkwrap，並拒絕已發布套件中的 `package-lock.json`。外掛的 npm 發布流程會檢查外掛本身的 shrinkwrap、安裝套件本身隨附的相依性，然後進行封裝或發布。

## 檢查已發布的套件

根套件：

```bash
npm pack openclaw@<version> --json --pack-destination /tmp/openclaw-pack
tar -tf /tmp/openclaw-pack/openclaw-<version>.tgz | grep '^package/npm-shrinkwrap.json$'
```

外掛套件：

```bash
npm pack @openclaw/discord@<version> --json --pack-destination /tmp/openclaw-plugin-pack
tar -tf /tmp/openclaw-plugin-pack/openclaw-discord-<version>.tgz | grep '^package/npm-shrinkwrap.json$'
tar -tf /tmp/openclaw-plugin-pack/openclaw-discord-<version>.tgz | grep '^package/node_modules/'
```

背景資訊：[npm-shrinkwrap.json](https://docs.npmjs.com/cli/v11/configuring-npm/npm-shrinkwrap-json)。
