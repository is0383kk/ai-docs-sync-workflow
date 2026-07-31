---
read_when:
    - 你正在建立新的自訂 skill
    - 你需要一套適用於以 SKILL.md 為基礎的 Skills 快速入門工作流程
    - 你想要使用 Skill Workshop 提交一項 Skill 供代理程式審查
sidebarTitle: Creating skills
summary: 為你的 OpenClaw 代理程式建置、測試及發布自訂的 SKILL.md 工作區 Skills。
title: 建立 Skills
x-i18n:
    generated_at: "2026-07-26T08:15:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cba2aa863ebd083d4592e8a764dbdc2c30a0dd8aff49d273927e82df0069bc81
    source_path: tools/creating-skills.md
    workflow: 16
---

Skills 會教導代理程式如何以及何時使用工具。每個 Skill 都是一個目錄，
其中包含具有 YAML frontmatter 和 Markdown 指示的 `SKILL.md` 檔案。
OpenClaw 會依照定義的[優先順序](/zh-TW/tools/skills#loading-order)，從多個根目錄載入 Skills。

## 建立你的第一個 Skill

<Steps>
  <Step title="建立 Skill 目錄">
    Skills 位於你工作區的 `skills/` 資料夾中：

    ```bash
    mkdir -p ~/.openclaw/workspace/skills/hello-world
    ```

    你可以將 Skills 分組放在子資料夾中，以便整理；Skill 仍由
    `SKILL.md` frontmatter 命名，而不是由資料夾路徑命名：

    ```bash
    mkdir -p ~/.openclaw/workspace/skills/personal/hello-world
    # Skill 名稱仍為 "hello-world"，使用 /hello-world 呼叫
    ```

  </Step>

  <Step title="撰寫 SKILL.md">
    frontmatter 定義中繼資料；本文則提供代理程式指示。

    ```markdown
    ---
    name: hello-world
    description: 一個會輸出問候語的簡單 Skill。
    ---

    # Hello World

    當使用者要求問候語時，使用 `exec` 工具執行：

    ```bash
    echo "來自你的自訂 Skill 的問候！"
    ```
    ```

    命名規則：
    - `name` 僅可使用小寫字母、數字和連字號。
    - 目錄名稱與 frontmatter 的 `name` 應保持一致。
    - `description` 會顯示給代理程式，也會出現在斜線命令探索結果中；
      請保持為單行且少於 160 個字元。

  </Step>

  <Step title="確認 Skill 已載入">
    ```bash
    openclaw skills list
    ```

    OpenClaw 預設會監看 Skills 根目錄下的 `SKILL.md` 檔案。如果
    監看器已停用，或你正在繼續現有工作階段，請啟動新的工作階段，
    讓代理程式收到更新後的清單：

    ```bash
    # 從聊天中封存目前工作階段並重新開始
    /new

    # 或重新啟動閘道
    openclaw gateway restart
    ```

  </Step>

  <Step title="進行測試">
    ```bash
    openclaw agent --message "給我一句問候語"
    ```

    或開啟聊天，直接向代理程式提出要求。使用 `/skill hello-world`
    依名稱明確呼叫它。

  </Step>
</Steps>

## SKILL.md 參考資料

### 必填欄位

| 欄位         | 說明                                                     |
| ------------- | --------------------------------------------------------------- |
| `name`        | 使用小寫字母、數字和連字號的唯一 slug        |
| `description` | 顯示給代理程式並出現在探索輸出中的單行說明 |

### 選用 frontmatter 鍵

| 欄位                      | 預設值 | 說明                                                                      |
| -------------------------- | ------- | -------------------------------------------------------------------------------- |
| `user-invocable`           | `true`  | 將 Skill 公開為使用者斜線命令                                         |
| `disable-model-invocation` | `false` | 不將 Skill 放入代理程式的系統提示詞中（仍可透過 `/skill` 執行）        |
| `command-dispatch`         | —       | 設為 `tool`，將斜線命令直接路由至工具並略過模型 |
| `command-tool`             | —       | 設定 `command-dispatch: tool` 時要呼叫的工具名稱                         |
| `command-arg-mode`         | `raw`   | 進行工具分派時，將原始引數字串轉送給工具                      |
| `homepage`                 | —       | 在 macOS Skills 使用者介面中顯示為「Website」的 URL                                    |

如需閘控欄位（`requires.bins`、`requires.env` 等）的資訊，請參閱
[Skills — 閘控](/zh-TW/tools/skills#gating)。

### 使用 `{baseDir}`

在 Skill 目錄中參照檔案，而不必寫死路徑；代理程式會以 Skill 自身的目錄
解析 `{baseDir}`：

```markdown
執行位於 `{baseDir}/scripts/run.sh` 的輔助指令碼。
```

## 新增條件式啟用

為你的 Skill 設定閘控，使其僅在相依項目可用時載入：

```markdown
---
name: gemini-search
description: 使用 Gemini 命令列介面搜尋。
metadata: { "openclaw": { "requires": { "bins": ["gemini"] }, "primaryEnv": "GEMINI_API_KEY" } }
---
```

<AccordionGroup>
  <Accordion title="閘控選項">
    | 鍵 | 說明 |
    | --- | --- |
    | `requires.bins` | 所有二進位檔都必須存在於 `PATH` 上 |
    | `requires.anyBins` | 至少一個二進位檔必須存在於 `PATH` 上 |
    | `requires.env` | 每個環境變數都必須存在於程序或設定中 |
    | `requires.config` | 每個 `openclaw.json` 路徑都必須為真值 |
    | `os` | 平台篩選器：`["darwin"]`、`["linux"]`、`["win32"]` |
    | `always` | 設定 `true` 以略過所有閘控，並一律納入該 Skill |

    完整參考資料：[Skills — 閘控](/zh-TW/tools/skills#gating)。

  </Accordion>
  <Accordion title="環境與 API 金鑰">
    在 `openclaw.json` 中將 API 金鑰連接至 Skill 項目：

    ```json5
    {
      skills: {
        entries: {
          "gemini-search": {
            enabled: true,
            apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" },
          },
        },
      },
    }
    ```

    該金鑰只會在那次代理程式回合中注入主機程序。
    它不會進入沙箱；請參閱
    [沙箱化的環境變數](/zh-TW/tools/skills-config#sandboxed-skills-and-env-vars)。

  </Accordion>
</AccordionGroup>

## 透過 Skill Workshop 提案

若是由代理程式草擬的 Skills，或你希望 Skill 上線前先由操作者審查，
請使用 [Skill Workshop](/zh-TW/tools/skill-workshop) 提案，而不要直接寫入
`SKILL.md`。

```bash
# 提案建立全新的 Skill
openclaw skills workshop propose-create \
  --name "hello-world" \
  --description "一個會輸出問候語的簡單 Skill。" \
  --proposal ./PROPOSAL.md

# 提案更新現有 Skill
openclaw skills workshop propose-update hello-world \
  --proposal ./PROPOSAL.md \
  --description "已更新的問候 Skill"
```

當提案包含支援檔案時，請使用 `--proposal-dir`：

```bash
openclaw skills workshop propose-create \
  --name "hello-world" \
  --description "一個會輸出問候語的簡單 Skill。" \
  --proposal-dir ./hello-world-proposal/
```

該目錄的根目錄中必須包含 `PROPOSAL.md`。支援檔案應放在
`assets/`、`examples/`、`references/`、`scripts/` 或 `templates/` 下。

審查後：

```bash
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop apply <proposal-id>
```

完整的提案生命週期請參閱 [Skill Workshop](/zh-TW/tools/skill-workshop)。

## 發布至 ClawHub

<Steps>
  <Step title="確認你的 SKILL.md 完整無缺">
    請確認已設定 `name`、`description` 及所有 `metadata.openclaw` 閘控欄位。
    如果你有專案頁面，請新增 `homepage` URL。
  </Step>
  <Step title="安裝獨立版 ClawHub 命令列介面並登入">
    ```bash
    npm i -g clawhub
    clawhub login
    ```
  </Step>
  <Step title="發布">
    ```bash
    clawhub skill publish ./path/to/hello-world
    ```

    新增 `--version <version>` 或 `--owner <owner>`，即可覆寫推斷的
    版本或以特定擁有者身分發布。如需完整流程、擁有者範圍及其他
    維護命令（`clawhub sync`、`clawhub skill rename` 等），請參閱
    [ClawHub — 發布](/zh-TW/clawhub/publishing)和
    [ClawHub 命令列介面](/zh-TW/clawhub/cli)。

  </Step>
</Steps>

## 最佳實務

<Tip>
  - **保持精簡** — 指示模型要做*什麼*，而不是如何成為 AI。
  - **安全優先** — 如果你的 Skill 使用 `exec`，請確保提示詞不會允許
    不受信任的輸入任意注入命令。
  - **在本機測試** — 分享前使用 `openclaw agent --message "..."`。
  - **使用 ClawHub** — 從頭建置前，先到 [clawhub.ai](https://clawhub.ai)
    瀏覽社群 Skills。
</Tip>

## 相關資源

<CardGroup cols={2}>
  <Card title="Skills 參考資料" href="/zh-TW/tools/skills" icon="puzzle-piece">
    載入順序、閘控、允許清單及 SKILL.md 格式。
  </Card>
  <Card title="Skill Workshop" href="/zh-TW/tools/skill-workshop" icon="flask">
    由代理程式草擬之 Skills 的提案佇列。
  </Card>
  <Card title="Skills 設定" href="/zh-TW/tools/skills-config" icon="gear">
    完整的 `skills.*` 設定結構描述。
  </Card>
  <Card title="ClawHub" href="/zh-TW/clawhub" icon="cloud">
    在公開登錄檔中瀏覽及發布 Skills。
  </Card>
  <Card title="建置外掛" href="/zh-TW/plugins/building-plugins" icon="plug">
    外掛可將 Skills 與其所記載的工具一同提供。
  </Card>
</CardGroup>
