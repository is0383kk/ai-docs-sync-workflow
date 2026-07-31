---
read_when:
    - 配置 Skills 加载、安装或门控行为
    - 设置每个智能体的 Skills 可见性
    - 调整 Skill Workshop 限制或审批策略
sidebarTitle: Skills config
summary: Skills、智能体允许列表、workshop 设置和沙箱环境变量处理的完整配置架构参考。
title: Skills 配置
x-i18n:
    generated_at: "2026-07-26T06:28:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bc154bdf8a8537095a4d39bc6e86ebfd716e6beacd45def9c8a1c15fcdc93698
    source_path: tools/skills-config.md
    workflow: 16
---

大多数 Skills 配置位于
`~/.openclaw/openclaw.json` 中的 `skills` 下。Agent 专属的可见性配置位于
`agents.defaults.skills` 和 `agents.entries.*.skills` 下。

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
  对于内置图像生成，请使用 `agents.defaults.mediaModels.image`
  加上核心 `image_generate` 工具，而不是 `skills.entries`。Skill
  条目仅用于自定义或第三方 Skill 工作流。
</Note>

## 加载（`skills.load`）

<ParamField path="skills.load.extraDirs" type="string[]">
  要扫描的其他 Skill 目录，优先级最低（低于
  内置和插件 Skills）。路径展开支持 `~`。
</ParamField>

<ParamField path="skills.load.allowSymlinkTargets" type="string[]">
  受信任的真实目标目录；符号链接的 Skill 文件夹可以解析到这些目录，
  即使符号链接位于配置的根目录之外。将其用于有意采用的相邻仓库布局，例如
  `<workspace>/skills/manager -> ~/Projects/manager/skills`。此列表应保持
  严格范围——不要指向 `~` 或 `~/Projects` 等宽泛的根目录。
</ParamField>

<ParamField path="skills.load.watch" type="boolean" default="true">
  监视 Skill 文件夹，并在 `SKILL.md` 文件
  发生变化时刷新 Skills 快照。涵盖分组 Skill 根目录下的嵌套文件。
</ParamField>

## 安装（`skills.install`）

<ParamField path="skills.install.preferBrew" type="boolean" default="true">
  当 `brew` 可用时，优先使用 Homebrew 安装程序。
</ParamField>

<ParamField path="skills.install.nodeManager" type='"npm" | "pnpm" | "yarn" | "bun"' default='"npm"'>
  安装 Skill 时首选的 Node 包管理器。此设置仅影响 Skill
  安装——OpenClaw CLI 和 Gateway 网关运行时需要 Node，因为
  规范状态存储使用 `node:sqlite`。`openclaw setup --node-manager` 和
  `openclaw onboard --node-manager` 接受 `npm`、`pnpm` 或 `bun`；对于
  由 Yarn 支持的 Skill 安装，请直接在配置中设置 `"yarn"`。
</ParamField>

<ParamField path="skills.install.allowUploadedArchives" type="boolean" default="false">
  允许受信任的 `operator.admin` Gateway 网关客户端安装通过
  `skills.upload.*` 暂存的私有 zip 归档。常规 ClawHub 安装不需要
  此设置。
</ParamField>

## 操作员安装策略（`security.installPolicy`）

当操作员需要通过受信任的本地命令，依据主机特定策略批准或阻止 Skill 和插件安装时，
请使用 `security.installPolicy`。该策略在 OpenClaw 暂存源材料之后、
安装或更新继续之前运行。它适用于 ClawHub Skills、上传的 Skills、Git/本地
Skills、Skill 依赖项安装程序，以及插件安装/更新源。

```json5
{
  security: {
    installPolicy: {
      enabled: true,
      // 省略 targets 以涵盖所有支持的目标。
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
  启用由操作员管理的安装策略。如果启用后没有有效的 `exec`
  命令，安装将以关闭方式失败。
</ParamField>

<ParamField path="security.installPolicy.targets" type='("skill" | "plugin")[]'>
  可选的目标筛选器。省略时，策略适用于所有支持的
  目标，因此新的安装类型不会意外开放。
</ParamField>

<ParamField path="security.installPolicy.exec.command" type="string">
  受信任策略可执行文件的绝对路径。OpenClaw 不通过
  shell 运行该文件，并会在使用前验证路径。
</ParamField>

<ParamField path="security.installPolicy.exec.args" type="string[]">
  在 `command` 之后传递的静态参数。
</ParamField>

<ParamField path="security.installPolicy.exec.timeoutMs" type="number" default="10000">
  单次策略决策允许的最大实际运行时间。
</ParamField>

<ParamField path="security.installPolicy.exec.noOutputTimeoutMs" type="number" default="timeoutMs">
  在策略以关闭方式失败之前，stdout 或 stderr 无输出的最长
  时间。
</ParamField>

<ParamField path="security.installPolicy.exec.maxOutputBytes" type="number" default="1048576">
  从策略进程接受的 stdout 和 stderr 合计最大字节数。
</ParamField>

<ParamField path="security.installPolicy.exec.env" type="Record<string, string>">
  提供给策略进程的字面环境变量。
</ParamField>

<ParamField path="security.installPolicy.exec.passEnv" type="string[]">
  从 OpenClaw 进程复制到
  策略进程的环境变量名称。仅传递指定名称的变量。
</ParamField>

<ParamField path="security.installPolicy.exec.trustedDirs" type="string[]">
  可以包含策略可执行文件的目录可选允许列表。
</ParamField>

<ParamField path="security.installPolicy.exec.allowInsecurePath" type="boolean" default="false">
  绕过命令路径所有权和权限检查。仅当
  该路径受其他机制保护时使用。
</ParamField>

<ParamField path="security.installPolicy.exec.allowSymlinkCommand" type="boolean" default="false">
  允许配置的命令路径为符号链接。解析后的目标
  仍必须满足其他路径检查。解释器脚本参数必须
  是直接的常规文件，不能是符号链接。
</ParamField>

该策略通过 stdin 接收一个 JSON 对象，其中包含 `protocolVersion: 1`、
`openclawVersion`、`targetType`、`targetName`、`sourcePath`、`sourcePathKind`、
可选的结构化 `source`、结构化 `origin` 和 `request`。它必须
向 stdout 写入一个 JSON 对象：`{ "protocolVersion": 1, "decision": "allow" }`
或 `{ "protocolVersion": 1, "decision": "block", "reason": "..." }`。非零
退出、超时、JSON 格式错误、字段缺失或协议版本不受支持，
都将以关闭方式失败。

OpenClaw 在 Gateway 网关正常启动期间不会执行安装策略。
如果策略已启用但不可用，安装和更新将以关闭方式失败。
`openclaw doctor` 执行静态验证；`openclaw doctor --deep`
针对配置的命令执行合成安装探测。

批量更新会对每个目标应用策略：被阻止的 Skill 或插件更新会导致
该目标失败，但不会禁用策略，也不会跳过批次中的后续目标。

stdin 示例：

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

最小策略命令：

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
        reason: "此主机不允许使用本地插件路径",
      }),
    );
    return;
  }
  process.stdout.write(JSON.stringify({ protocolVersion: 1, decision: "allow" }));
});
```

## 内置 Skill 允许列表

<ParamField path="skills.allowBundled" type="string[]">
  仅适用于**内置** Skills 的可选允许列表。设置后，只有列表中的内置
  Skills 符合使用条件。托管级、Agent 级和工作区
  Skills 不受影响。
</ParamField>

## 按 Skill 配置的条目（`skills.entries`）

默认情况下，`entries` 下的键与 Skill 的 `name` 匹配。如果 Skill 定义了
`metadata.openclaw.skillKey`，则改用该键。包含连字符的名称需要加引号
（JSON5 允许带引号的键）。

<ParamField path="skills.entries.<key>.enabled" type="boolean">
  `false` 会禁用该 Skill，即使它是内置或已安装的。内置
  Skill `coding-agent` 需要主动启用——将其设置为 `true`，并确保
  `claude`、`codex`、`opencode` 或其他受支持的 CLI 之一已安装且
  已完成身份验证。
</ParamField>

<ParamField path="skills.entries.<key>.apiKey" type='string | { source, provider, id }'>
  为声明 `metadata.openclaw.primaryEnv` 的 Skills 提供的便捷字段。
  支持纯文本字符串或 SecretRef：`{ source: "env", provider: "default", id: "VAR_NAME" }`。
</ParamField>

<ParamField path="skills.entries.<key>.env" type="Record<string, string>">
  为 Agent 运行注入的环境变量。仅当
  进程中尚未设置该变量时才注入。
</ParamField>

<ParamField path="skills.entries.<key>.config" type="object">
  用于自定义按 Skill 配置字段的可选容器。
</ParamField>

## Agent 允许列表（`agents`）

如果希望使用同一台机器/工作区的 Skill 根目录，但为每个 Agent
设置不同的可见 Skill 集，请使用 Agent 配置。

```json5
{
  agents: {
    defaults: {
      skills: ["github", "weather"], // 共享基线
    },
    list: [
      { id: "writer" }, // 继承 github、weather
      { id: "docs", skills: ["docs-search"] }, // 完全替换默认值
      { id: "locked-down", skills: [] }, // 无 Skills
    ],
  },
}
```

<ParamField path="agents.defaults.skills" type="string[]">
  由省略 `agents.entries.*.skills` 的 Agents 继承的
  共享基线允许列表。完全省略可使 Skills 默认不受
  限制。
</ParamField>

<ParamField path="agents.entries.*.skills" type="string[]">
  该 Agent 的明确最终 Skill 集。明确指定的列表会**替换**
  继承的默认值，而不会合并。设置为 `[]` 可不向
  该 Agent 公开任何 Skills。
</ParamField>

<Warning>
  Agent Skill 允许列表是 OpenClaw
  Skill 发现、提示词、斜杠命令发现、沙箱同步和 Skill
  快照的可见性及加载筛选器。它们不是 shell 运行时的授权边界。如果 Agent
  可以运行主机上的 `exec`，该 shell 仍可运行外部客户端或读取
  执行用户可见的主机文件，包括 `~/.openclaw/skills/config/mcporter.json` 等 MCP 客户端
  注册表。对于
  按 Agent 隔离 MCP 的场景，请将 Skill 允许列表与沙箱/操作系统用户
  隔离结合使用，拒绝主机 Exec 或对其设置严格的允许列表，并优先在 MCP 服务器上使用按 Agent
  配置的凭据。
</Warning>

## Workshop（`skills.workshop`）

<ParamField path="skills.workshop.autonomous.enabled" type="boolean" default="false">
  当 `true` 时，OpenClaw 可以根据持久化更正创建待处理提案，
  并可在系统进入空闲状态后审查已成功完成且具有实质性的工作。
  这可能会在符合条件的轮次后增加一次后台模型运行。当此设置为 `false` 时，
  用户提示触发的技能创建和 `/learn` 仍可正常工作。
</ParamField>

有关资格条件、隐私、成本、仅提案权限和故障排除，请参阅[自我学习](/zh-CN/tools/self-learning)。

<ParamField path="skills.workshop.approvalPolicy" type='"pending" | "auto"' default='"auto"'>
  `auto` 允许智能体主动应用、拒绝或隔离提案，无需额外的审批提示。
  `pending` 则需要操作员审批。
</ParamField>

<ParamField path="skills.workshop.allowSymlinkTargetWrites" type="boolean" default="false">
  允许 Skill Workshop 在工作区技能符号链接的真实目标已受
  `skills.load.allowSymlinkTargets` 信任时，通过这些符号链接写入。除非应用生成的提案时
  应修改该共享技能根目录，否则请保持禁用此选项。
</ParamField>

<ParamField path="skills.workshop.maxPending" type="number" default="50">
  每个工作区保留的待处理和已隔离提案的最大数量（允许范围：1-200）。
</ParamField>

<ParamField path="skills.workshop.maxSkillBytes" type="number" default="40000">
  提案正文的最大字节数（允许范围：1024-200000）。提案描述另有
  160 字节的硬性上限，因为它们会出现在发现和列表输出中。
</ParamField>

有关此配置所控制的提案生命周期、CLI 命令、智能体工具参数和 Gateway 网关方法，
请参阅 [Skill Workshop](/zh-CN/tools/skill-workshop)。

## 使用符号链接的技能根目录

默认情况下，工作区、项目智能体、额外目录和内置技能根目录均为
包含范围边界。位于 `<workspace>/skills` 下、解析结果超出根目录的
符号链接技能文件夹将被跳过，并记录一条日志消息。

若要允许有意设置的符号链接布局，请声明受信任的目标：

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

使用此配置后，`<workspace>/skills/manager -> ~/Projects/manager/skills`
将在 realpath 解析后被接受。`extraDirs` 会直接扫描同级仓库；
`allowSymlinkTargets` 则为现有布局保留符号链接路径。

默认情况下，Skill Workshop 在应用提案时不会通过这些符号链接写入。
若要允许 Workshop 在应用提案时修改已受信任的符号链接目标下的技能，
请单独选择启用：

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

托管的 `~/.openclaw/skills` 和个人的 `~/.agents/skills` 目录
已无条件接受技能目录符号链接（仍会对每个技能应用
`SKILL.md` 包含范围检查）——只有工作区、额外目录和项目智能体
（`<workspace>/.agents/skills`）根目录才需要 `allowSymlinkTargets`。

## 沙箱隔离的技能和环境变量

<Warning>
  `skills.entries.<skill>.env` 和 `apiKey` 仅适用于**主机**运行。
  它们在沙箱内不起作用——依赖 `GEMINI_API_KEY` 的技能将因
  `apiKey not configured` 而失败，除非另行向沙箱提供该变量。
</Warning>

通过以下配置将密钥传入 Docker 沙箱：

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
  拥有 Docker 守护进程访问权限的用户可以通过 Docker 元数据检查
  `sandbox.docker.env` 的值。如果这种暴露不可接受，请使用挂载的密钥文件、
  自定义镜像或其他传递路径。
</Note>

## 加载顺序提醒

```text
workspace/skills      （最高）
workspace/.agents/skills
~/.agents/skills
~/.openclaw/skills
内置技能
skills.load.extraDirs （最低）
```

启用监视器后，对技能和配置的更改将在下一个新会话中生效；
当监视器检测到更改时，也可在智能体的下一轮中生效。

## 相关内容

<CardGroup cols={2}>
  <Card title="Skills 参考" href="/zh-CN/tools/skills" icon="puzzle-piece">
    技能的定义、加载顺序、门控机制以及 SKILL.md 格式。
  </Card>
  <Card title="创建技能" href="/zh-CN/tools/creating-skills" icon="hammer">
    编写自定义工作区技能。
  </Card>
  <Card title="Skill Workshop" href="/zh-CN/tools/skill-workshop" icon="flask">
    用于智能体起草技能的提案队列。
  </Card>
  <Card title="自我学习" href="/zh-CN/tools/self-learning" icon="brain">
    根据已完成工作生成的保守型选择启用提案。
  </Card>
  <Card title="斜杠命令" href="/zh-CN/tools/slash-commands" icon="terminal">
    原生斜杠命令目录和聊天指令。
  </Card>
</CardGroup>
