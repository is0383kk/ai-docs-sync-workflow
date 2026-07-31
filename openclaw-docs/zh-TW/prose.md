---
read_when:
    - 你想要執行或撰寫 .prose 工作流程檔案
    - 你想要啟用 OpenProse 外掛
    - 你需要瞭解 OpenProse 如何對應至 OpenClaw 的基本元件
sidebarTitle: OpenProse
summary: OpenProse 是一種以 Markdown 為優先的工作流程格式，適用於多代理 AI 工作階段。在 OpenClaw 中，它以外掛形式提供，並附有 `/prose` 斜線指令和 Skills 套件。
title: OpenProse
x-i18n:
    generated_at: "2026-07-26T07:52:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8b04eb23bf827fbec6db11c1e95993e7f6c617451c5f4fda771ad078674c12bc
    source_path: prose.md
    workflow: 16
---

OpenProse 是一種可攜式、以 Markdown 為優先的工作流程格式，用於協調 AI
工作階段。在 OpenClaw 中，它以外掛形式提供，會安裝 OpenProse skill
套件與 `/prose` 斜線命令。程式位於 `.prose` 檔案中，並可
透過明確的控制流程產生多個子代理。

<CardGroup cols={3}>
  <Card title="安裝" icon="download" href="#install">
    啟用 OpenProse 外掛並重新啟動閘道。
  </Card>
  <Card title="執行程式" icon="play" href="#slash-command">
    使用 `/prose run` 執行 `.prose` 檔案或遠端程式。
  </Card>
  <Card title="撰寫程式" icon="pencil" href="#example-parallel-research-and-synthesis">
    使用平行與循序步驟編寫多代理工作流程。
  </Card>
</CardGroup>

## 安裝

<Steps>
  <Step title="啟用外掛">
    OpenProse 已隨附但預設停用。請啟用它：

    ```bash
    openclaw plugins enable open-prose
    ```

  </Step>
  <Step title="重新啟動閘道">
    ```bash
    openclaw gateway restart
    ```
  </Step>
  <Step title="驗證">
    ```bash
    openclaw plugins list | grep prose
    ```

    應會看到 `open-prose` 已啟用。現在可以在聊天中使用
    `/prose` skill 命令。

  </Step>
</Steps>

從儲存庫簽出版本可直接安裝外掛：
`openclaw plugins install ./extensions/open-prose`

## 斜線命令

OpenProse 將 `/prose` 註冊為使用者可叫用的 skill 命令：

```text
/prose help
/prose run <file.prose>
/prose run <handle/slug>
/prose run <https://example.com/file.prose>
/prose compile <file.prose>
/prose examples
/prose update
```

`/prose run <handle/slug>` 會解析為 `https://p.prose.md/<handle>/<slug>`。
直接 URL 會使用 `web_fetch` 工具依原樣擷取。

頂層遠端執行必須明確指定。`.prose` 程式內的遠端匯入是
遞移程式碼相依項目：在 OpenProse 擷取任何遠端 `use` 目標前，
會顯示解析後的匯入清單，並要求操作者針對該次執行準確回覆
`approve remote prose imports`。

## 功能

- 具備明確平行處理的多代理研究與統整。
- 可重複且安全經核准的工作流程（程式碼審查、事件分級處理、內容流水線）。
- 可在支援的代理執行環境中執行的可重用 `.prose` 程式。

## 範例：平行研究與統整

```prose
# 由兩個代理平行執行研究與統整。

input topic: "我們應該研究什麼？"

agent researcher:
  model: sonnet
  prompt: "你要進行深入研究並引用來源。"

agent writer:
  model: opus
  prompt: "你要撰寫簡潔摘要。"

parallel:
  findings = session: researcher
    prompt: "研究 {topic}。"
  draft = session: writer
    prompt: "摘要 {topic}。"

session "將研究結果與草稿合併為最終答案。"
  context: { findings, draft }
```

## OpenClaw 執行環境對應

OpenProse 程式會對應至 OpenClaw 基本元件：

| OpenProse 概念            | OpenClaw 工具                                    |
| ------------------------- | ----------------------------------------------- |
| 產生工作階段／Task 工具   | `sessions_spawn`                                |
| 檔案讀取／寫入            | `read` / `write`                                |
| 網頁擷取                  | `web_fetch`（需要 POST 時使用 `exec` + curl） |

<Warning>
  如果你的工具允許清單封鎖 `sessions_spawn`、`read`、`write` 或
  `web_fetch`，OpenProse 程式將會失敗。請檢查你的
  [工具允許清單設定](/zh-TW/gateway/config-tools)。
</Warning>

## 檔案位置

OpenProse 將狀態保存在工作區的 `.prose/` 下：

```text
.prose/
├── .env                      # 設定（key=value），例如 OPENPROSE_POSTGRES_URL
├── runs/
│   └── {YYYYMMDD}-{HHMMSS}-{random}/
│       ├── program.prose     # 執行中程式的副本
│       ├── state.md          # 執行狀態
│       ├── bindings/
│       ├── imports/          # 巢狀遠端程式執行
│       └── agents/
└── agents/                   # 專案範圍的持久代理
```

跨專案共用的使用者層級持久代理位於：

```text
~/.prose/agents/
```

## 狀態後端

<AccordionGroup>
  <Accordion title="檔案系統（預設）">
    狀態會寫入工作區中的 `.prose/runs/...`。不需要額外
    相依套件。
  </Accordion>
  <Accordion title="內容內">
    暫時狀態保存在上下文視窗中；使用 `--in-context` 選取。
    適合小型、短期執行的程式。
  </Accordion>
  <Accordion title="sqlite（實驗性）">
    使用 `--state=sqlite` 選取。需要 `PATH` 上的 `sqlite3` 二進位檔
    （缺少時會退回檔案系統）；狀態會儲存在
    `.prose/runs/{id}/state.db`。
  </Accordion>
  <Accordion title="postgres（實驗性）">
    使用 `--state=postgres` 選取。需要 `psql`，並須在
    `OPENPROSE_POSTGRES_URL` 中提供連線字串（請在 `.prose/.env` 中設定）。

    <Warning>
      Postgres 認證資訊會流入子代理記錄。請使用專用且具最低權限的資料庫。
    </Warning>

  </Accordion>
</AccordionGroup>

## 安全性

將 `.prose` 檔案視同程式碼。在執行前先審查，包括遠端
`use` 匯入。頂層 `/prose run https://...` 請求必須明確指定，但
遞移遠端匯入在擷取或執行前，每次執行都需要核准。使用 OpenClaw 工具允許清單與核准閘門來控制
副作用。若需要具確定性且須經核准的工作流程，可與
[Lobster](/zh-TW/tools/lobster) 比較。

## 相關內容

<CardGroup cols={2}>
  <Card title="Skills 參考資料" href="/zh-TW/tools/skills" icon="puzzle-piece">
    OpenProse 的 skill 套件如何載入，以及會套用哪些閘門。
  </Card>
  <Card title="子代理" href="/zh-TW/tools/subagents" icon="users">
    OpenClaw 原生的多代理協調層。
  </Card>
  <Card title="文字轉語音" href="/zh-TW/tools/tts" icon="volume-high">
    為你的工作流程新增音訊輸出。
  </Card>
  <Card title="斜線命令" href="/zh-TW/tools/slash-commands" icon="terminal">
    所有可用的聊天命令，包括 /prose。
  </Card>
</CardGroup>

官方網站：[https://www.prose.md](https://www.prose.md)
