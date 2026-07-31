---
read_when:
    - 設定 Skills 載入、安裝或門控行為
    - 設定各代理程式的 Skills 可見性
    - 調整 Skill Workshop 限制或核准政策
sidebarTitle: Skills config
summary: skills.* 設定結構描述、代理程式允許清單、工作坊設定，以及沙箱環境變數處理的完整參考資料。
title: Skills 設定
x-i18n:
    generated_at: "2026-07-26T08:02:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bc154bdf8a8537095a4d39bc6e86ebfd716e6beacd45def9c8a1c15fcdc93698
    source_path: tools/skills-config.md
    workflow: 16
---

大多數 Skills 設定都位於 `skills` 之下，並存放在
`~/.openclaw/openclaw.json`。代理程式特定的可見性則位於
`agents.defaults.skills` 和 `agents.entries.*.skills` 之下。

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
      watch: true,
    },
    install: {
      preferBrew: true,
      nodeManager: "npm",
      allowUploadedArchives: false,
    },
    workshop: {
      autonomous: { enabled: false },
      allowSymlinkTargetWrites: false,
      approvalPolicy: "auto",
      maxPending: 50,
      maxSkillBytes: 40000,
    },
    entries: {
      "image-lab": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" },
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

<Note>
  若要使用內建的影像生成功能，請使用 `agents.defaults.mediaModels.image`
  搭配核心 `image_generate` 工具，而不是 `skills.entries`。Skill
  項目僅適用於自訂或第三方 Skill 工作流程。
</Note>

## 載入（`skills.load`）

<ParamField path="skills.load.extraDirs" type="string[]">
  要掃描的其他 Skill 目錄，其優先順序最低（低於
  隨附及外掛的 Skills）。路徑展開支援 `~`。
</ParamField>

<ParamField path="skills.load.allowSymlinkTargets" type="string[]">
  符號連結 Skill 資料夾可解析至的受信任實際目標目錄，即使符號連結位於
  設定的根目錄之外亦然。此設定適用於刻意採用的同層儲存庫配置，例如
  `<workspace>/skills/manager -> ~/Projects/manager/skills`。此清單應保持精簡，
  請勿指向 `~` 或 `~/Projects` 等範圍廣泛的根目錄。
</ParamField>

<ParamField path="skills.load.watch" type="boolean" default="true">
  監看 Skill 資料夾，並在 `SKILL.md` 檔案變更時
  重新整理 Skills 快照。涵蓋分組 Skill 根目錄下的巢狀檔案。
</ParamField>

## 安裝（`skills.install`）

<ParamField path="skills.install.preferBrew" type="boolean" default="true">
  當 `brew` 可用時，優先使用 Homebrew 安裝程式。
</ParamField>

<ParamField path="skills.install.nodeManager" type='"npm" | "pnpm" | "yarn" | "bun"' default='"npm"'>
  安裝 Skill 時偏好的 Node 套件管理工具。這只會影響 Skill
  安裝；OpenClaw 命令列介面與閘道執行階段需要 Node，因為
  標準狀態儲存區使用 `node:sqlite`。`openclaw setup --node-manager` 和
  `openclaw onboard --node-manager` 接受 `npm`、`pnpm` 或 `bun`；
  若要使用由 Yarn 支援的 Skill 安裝，請直接在設定中設置
  `"yarn"`。
</ParamField>

<ParamField path="skills.install.allowUploadedArchives" type="boolean" default="false">
  允許受信任的 `operator.admin` 閘道用戶端安裝透過
  `skills.upload.*` 暫存的私有 zip 封存檔。一般的 ClawHub 安裝
  不需要此設定。
</ParamField>

## 操作者安裝原則（`security.installPolicy`）

當操作者需要使用受信任的本機命令，依據主機特定原則核准或封鎖
Skill 與外掛安裝時，請使用 `security.installPolicy`。此原則會在
OpenClaw 暫存來源素材之後、安裝或更新繼續之前執行。它適用於
ClawHub Skills、上傳的 Skills、Git／本機 Skills、Skill 相依項目安裝程式，
以及外掛安裝／更新來源。

```json5
{
  security: {
    installPolicy: {
      enabled: true,
      // 省略 targets 以涵蓋所有支援的目標。
      targets: ["skill", "plugin"],
      exec: {
        source: "exec",
        command: "/usr/local/bin/openclaw-install-policy",
        args: ["--json"],
        timeoutMs: 10000,
        noOutputTimeoutMs: 10000,
        maxOutputBytes: 1048576,
        passEnv: ["OPENCLAW_STATE_DIR", "PATH"],
        env: { POLICY_MODE: "strict" },
        trustedDirs: ["/usr/local/bin"],
      },
    },
  },
}
```

<ParamField path="security.installPolicy.enabled" type="boolean" default="false">
  啟用由操作者擁有的安裝原則。啟用後若沒有有效的 `exec`
  命令，安裝將採取封閉式失敗。
</ParamField>

<ParamField path="security.installPolicy.targets" type='("skill" | "plugin")[]'>
  選用的目標篩選器。若省略，原則會套用至所有支援的
  目標，避免新的安裝意外採取開放式失敗。
</ParamField>

<ParamField path="security.installPolicy.exec.command" type="string">
  受信任原則可執行檔的絕對路徑。OpenClaw 執行時不會使用
  Shell，並會在使用前驗證路徑。
</ParamField>

<ParamField path="security.installPolicy.exec.args" type="string[]">
  接在 `command` 後傳入的靜態引數。
</ParamField>

<ParamField path="security.installPolicy.exec.timeoutMs" type="number" default="10000">
  單次原則決策允許的最長實際經過時間。
</ParamField>

<ParamField path="security.installPolicy.exec.noOutputTimeoutMs" type="number" default="timeoutMs">
  在原則採取封閉式失敗之前，stdout 或 stderr 最長可無輸出的
  時間。
</ParamField>

<ParamField path="security.installPolicy.exec.maxOutputBytes" type="number" default="1048576">
  原則程序可接受的 stdout 與 stderr 位元組合計上限。
</ParamField>

<ParamField path="security.installPolicy.exec.env" type="Record<string, string>">
  提供給原則程序的常值環境變數。
</ParamField>

<ParamField path="security.installPolicy.exec.passEnv" type="string[]">
  從 OpenClaw 程序複製到原則程序的環境變數名稱。只會傳遞
  指定名稱的變數。
</ParamField>

<ParamField path="security.installPolicy.exec.trustedDirs" type="string[]">
  可包含原則可執行檔之目錄的選用允許清單。
</ParamField>

<ParamField path="security.installPolicy.exec.allowInsecurePath" type="boolean" default="false">
  略過命令路徑擁有權與權限檢查。僅在路徑受到其他機制
  保護時使用。
</ParamField>

<ParamField path="security.installPolicy.exec.allowSymlinkCommand" type="boolean" default="false">
  允許設定的命令路徑為符號連結。解析後的目標仍必須符合
  其他路徑檢查。直譯器指令碼引數必須是直接的一般檔案，
  不得為符號連結。
</ParamField>

原則會透過 stdin 接收一個 JSON 物件，其中包含 `protocolVersion: 1`、
`openclawVersion`、`targetType`、`targetName`、`sourcePath`、`sourcePathKind`、
選用的結構化 `source`、結構化 `origin`，以及 `request`。它必須
向 stdout 寫入一個 JSON 物件：`{ "protocolVersion": 1, "decision": "allow" }`
或 `{ "protocolVersion": 1, "decision": "block", "reason": "..." }`。非零結束狀態、
逾時、格式錯誤的 JSON、缺少欄位或不支援的通訊協定版本，
都會採取封閉式失敗。

OpenClaw 在閘道正常啟動期間不會執行安裝原則。
若原則已啟用但無法使用，安裝與更新會採取封閉式失敗。
`openclaw doctor` 會執行靜態驗證；`openclaw doctor --deep`
則會針對設定的命令執行模擬安裝探測。

大量更新會針對每個目標套用原則：遭封鎖的 Skill 或外掛更新會讓
該目標失敗，但不會停用原則，也不會略過批次中後續的目標。

stdin 範例：

```json
{
  "protocolVersion": 1,
  "openclawVersion": "2026.6.1",
  "targetType": "skill",
  "targetName": "weather",
  "sourcePath": "/var/folders/.../openclaw-skill-clawhub/root",
  "sourcePathKind": "directory",
  "source": {
    "kind": "clawhub",
    "authority": "openclaw",
    "mutable": false,
    "network": true
  },
  "origin": {
    "type": "clawhub",
    "registry": "https://clawhub.openclaw.ai",
    "slug": "weather",
    "version": "1.0.0"
  },
  "request": {
    "kind": "skill-install",
    "mode": "install",
    "requestedSpecifier": "clawhub:weather@1.0.0"
  },
  "skill": {
    "installId": "clawhub"
  }
}
```

最小原則命令：

```js
#!/usr/bin/env node

let input = "";
process.stdin.setEncoding("utf8");
process.stdin.on("data", (chunk) => {
  input += chunk;
});
process.stdin.on("end", () => {
  const request = JSON.parse(input);
  if (request.targetType === "plugin" && request.source?.kind === "local-path") {
    process.stdout.write(
      JSON.stringify({
        protocolVersion: 1,
        decision: "block",
        reason: "此主機不允許本機外掛路徑",
      }),
    );
    return;
  }
  process.stdout.write(JSON.stringify({ protocolVersion: 1, decision: "allow" }));
});
```

## 隨附 Skill 允許清單

<ParamField path="skills.allowBundled" type="string[]">
  僅適用於**隨附** Skills 的選用允許清單。設定後，只有清單中的
  隨附 Skills 才符合使用資格。受管理、代理程式層級與工作區
  Skills 不受影響。
</ParamField>

## 各 Skill 項目（`skills.entries`）

`entries` 下的鍵預設會比對 Skill 的 `name`。若 Skill 定義了
`metadata.openclaw.skillKey`，請改用該鍵。含連字號的名稱需加上引號
（JSON5 允許使用引號括住鍵）。

<ParamField path="skills.entries.<key>.enabled" type="boolean">
  即使 Skill 是隨附或已安裝，`false` 也會停用它。
  隨附的 `coding-agent` Skill 須選擇加入——請將其設為 `true`，並確保
  已安裝且驗證 `claude`、`codex`、`opencode` 或其他受支援的命令列介面
  之一。
</ParamField>

<ParamField path="skills.entries.<key>.apiKey" type='string | { source, provider, id }'>
  適用於宣告 `metadata.openclaw.primaryEnv` 之 Skills 的便利欄位。
  支援純文字字串或 SecretRef：`{ source: "env", provider: "default", id: "VAR_NAME" }`。
</ParamField>

<ParamField path="skills.entries.<key>.env" type="Record<string, string>">
  為代理程式執行注入的環境變數。僅在程序中尚未設定該
  變數時注入。
</ParamField>

<ParamField path="skills.entries.<key>.config" type="object">
  自訂各 Skill 設定欄位的選用集合。
</ParamField>

## 代理程式允許清單（`agents`）

若希望使用相同的機器／工作區 Skill 根目錄，但讓每個代理程式看見
不同的 Skill 集合，請使用代理程式設定。

```json5
{
  agents: {
    defaults: {
      skills: ["github", "weather"], // 共用基準
    },
    list: [
      { id: "writer" }, // 繼承 github、weather
      { id: "docs", skills: ["docs-search"] }, // 完全取代預設值
      { id: "locked-down", skills: [] }, // 沒有 Skills
    ],
  },
}
```

<ParamField path="agents.defaults.skills" type="string[]">
  省略 `agents.entries.*.skills` 的代理程式會繼承的共用基準
  允許清單。若要讓 Skills 預設不受限制，請完全省略。
</ParamField>

<ParamField path="agents.entries.*.skills" type="string[]">
  該代理程式的明確最終 Skill 集合。明確清單會**取代**
  繼承的預設值，而不是合併。設為 `[]` 可讓該代理程式
  不顯示任何 Skills。
</ParamField>

<Warning>
  代理程式 Skill 允許清單是 OpenClaw Skill 探索、提示詞、
  斜線命令探索、沙箱同步與 Skill 快照的可見性及載入篩選器。
  它們不是 Shell 執行階段的授權邊界。若代理程式可以執行主機
  `exec`，該 Shell 仍可執行外部用戶端，或讀取執行使用者可見的
  主機檔案，包括 `~/.openclaw/skills/config/mcporter.json` 等 MCP 用戶端
  登錄。若要實現每個代理程式的 MCP 隔離，請將 Skill 允許清單與沙箱／作業系統使用者
  隔離搭配使用，拒絕主機 exec 或採用嚴格的允許清單，並優先在 MCP 伺服器
  使用每個代理程式各自的認證資訊。
</Warning>

## Workshop（`skills.workshop`）

<ParamField path="skills.workshop.autonomous.enabled" type="boolean" default="false">
  當 `true` 時，OpenClaw 可從持久保留的修正建立待處理提案，
  並可在系統進入閒置後，審查已成功完成且具實質內容的工作。
  這可能會在符合條件的輪次後新增一次背景模型執行。由使用者提示的
  Skill 建立和 `/learn` 在此設定為 `false` 時仍可運作。
</ParamField>

請參閱[自我學習](/zh-TW/tools/self-learning)，瞭解資格條件、隱私權、成本、
僅限提案的權限和疑難排解。

<ParamField path="skills.workshop.approvalPolicy" type='"pending" | "auto"' default='"auto"'>
  `auto` 允許代理程式自行發起套用、拒絕或隔離，而不需
  額外的核准提示。`pending` 則需要操作人員核准。
</ParamField>

<ParamField path="skills.workshop.allowSymlinkTargetWrites" type="boolean" default="false">
  允許 Skill Workshop 在工作區 Skill 符號連結的實際目標已受
  `skills.load.allowSymlinkTargets` 信任時，透過該連結寫入。除非套用產生的提案
  應修改該共用 Skill 根目錄，否則請保持停用。
</ParamField>

<ParamField path="skills.workshop.maxPending" type="number" default="50">
  每個工作區保留的待處理與已隔離提案數量上限（允許範圍：
  1-200）。
</ParamField>

<ParamField path="skills.workshop.maxSkillBytes" type="number" default="40000">
  提案本文大小上限（位元組，允許範圍：1024-200000）。提案說明
  另有 160 位元組的硬性上限，因為它們會出現在探索與清單輸出中。
</ParamField>

請參閱 [Skill Workshop](/zh-TW/tools/skill-workshop)，瞭解此設定所控制的提案生命週期、命令列介面
命令、代理程式工具參數和閘道方法。

## 使用符號連結的 Skill 根目錄

依預設，工作區、專案代理程式、額外目錄和內建 Skill 根目錄都是
範圍限制邊界。位於 `<workspace>/skills` 下、解析後指向根目錄外部的
符號連結 Skill 資料夾會被略過，並記錄一則日誌訊息。

若要允許刻意安排的符號連結配置，請宣告受信任的目標：

```json5
{
  skills: {
    load: {
      extraDirs: ["~/Projects/manager/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
  },
}
```

使用此設定後，`<workspace>/skills/manager -> ~/Projects/manager/skills`
會在實際路徑解析後獲准。`extraDirs` 會直接掃描同層級的儲存庫；
`allowSymlinkTargets` 則會保留現有配置所使用的符號連結路徑。

Skill Workshop 預設不會透過這些符號連結寫入。若要讓 Workshop
套用作業修改已受信任符號連結目標下的 Skill，請另行選擇啟用：

```json5
{
  skills: {
    load: {
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
    workshop: {
      allowSymlinkTargetWrites: true,
    },
  },
}
```

受管理的 `~/.openclaw/skills` 和個人的 `~/.agents/skills` 目錄
已無條件允許 Skill 目錄符號連結（各 Skill 的
`SKILL.md` 範圍限制仍適用）— `allowSymlinkTargets` 僅適用於
工作區、額外目錄和專案代理程式（`<workspace>/.agents/skills`）
根目錄。

## 沙箱化 Skill 與環境變數

<Warning>
  `skills.entries.<skill>.env` 和 `apiKey` 僅適用於**主機**執行。
  在沙箱內不會產生任何效果 — 依賴 `GEMINI_API_KEY` 的 Skill
  會因 `apiKey not configured` 而失敗，除非另行將該變數提供給沙箱。
</Warning>

使用以下設定將祕密傳入 Docker 沙箱：

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          env: { GEMINI_API_KEY: "your-key-here" },
        },
      },
    },
  },
}
```

<Note>
  擁有 Docker 常駐程式存取權的使用者可透過 Docker 中繼資料檢查
  `sandbox.docker.env` 值。若無法接受此種暴露，請使用掛載的祕密檔案、
  自訂映像檔或其他傳遞途徑。
</Note>

## 載入順序提醒

```text
workspace/skills      （最高）
workspace/.agents/skills
~/.agents/skills
~/.openclaw/skills
內建 Skill
skills.load.extraDirs （最低）
```

啟用監看器時，Skill 與設定的變更會在下一個新工作階段生效；若監看器
偵測到變更，則會在代理程式的下一個輪次生效。

## 相關內容

<CardGroup cols={2}>
  <Card title="Skill 參考資料" href="/zh-TW/tools/skills" icon="puzzle-piece">
    Skill 的定義、載入順序、閘控機制和 SKILL.md 格式。
  </Card>
  <Card title="建立 Skill" href="/zh-TW/tools/creating-skills" icon="hammer">
    編寫自訂工作區 Skill。
  </Card>
  <Card title="Skill Workshop" href="/zh-TW/tools/skill-workshop" icon="flask">
    代理程式草擬 Skill 的提案佇列。
  </Card>
  <Card title="自我學習" href="/zh-TW/tools/self-learning" icon="brain">
    根據已完成工作產生的保守選擇啟用提案。
  </Card>
  <Card title="斜線命令" href="/zh-TW/tools/slash-commands" icon="terminal">
    原生斜線命令目錄和聊天指令。
  </Card>
</CardGroup>
