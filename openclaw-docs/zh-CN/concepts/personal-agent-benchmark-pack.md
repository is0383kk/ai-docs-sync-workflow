---
read_when:
    - 运行本地个人智能体可靠性检查
    - 扩展由仓库支持的 QA 场景目录
    - 验证提醒、回复、记忆、脱敏、安全工具后续执行、任务状态、可安全共享的诊断信息、有证据支撑的完成声明和故障恢复
summary: 用于隐私保护型个人助理工作流检查的本地 qa-channel 场景。
title: 个人智能体基准测试包
x-i18n:
    generated_at: "2026-07-26T06:07:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 35da45e4b22b1044a777fa8d6bce87f9ace377950dd0af3f2419b40cfe4d9be6
    source_path: concepts/personal-agent-benchmark-pack.md
    workflow: 16
---

Personal Agent Benchmark Pack 是一个由小型仓库支持的 QA 场景包，用于
本地个人助理工作流。它不是通用模型基准测试，并且
不需要新的运行器：它复用私有 QA 栈（[QA overview](/zh-CN/concepts/qa-e2e-automation)）、
合成的 [QA channel](/zh-CN/channels/qa-channel) 以及现有的
`qa/scenarios` YAML 目录。

## 场景

共十个场景，定义于 `qa/scenarios/personal/*.yaml`：

| 场景 ID                                   | 检查项                                                                                     |
| ------------------------------------------ | ------------------------------------------------------------------------------------------ |
| `personal-reminder-roundtrip`              | 通过本地定时任务投递模拟个人提醒                                                         |
| `personal-channel-thread-reply`            | 通过 `qa-channel` 路由模拟私信和话题串回复                                         |
| `personal-memory-preference-recall`        | 从临时 QA 工作区的记忆文件中回忆模拟偏好                                                |
| `personal-redaction-no-secret-leak`        | 模拟密钥不回显检查                                                                       |
| `personal-tool-safety-followthrough`       | 在简短的审批式轮次后，安全地继续执行有读取结果支持的工具操作                             |
| `personal-approval-denial-stop`            | 对敏感本地读取请求拒绝审批后的停止行为                                                   |
| `personal-task-followthrough-status`       | 有证据支持的任务状态报告，明确区分待处理、已阻塞和已完成                                 |
| `personal-share-safe-diagnostics-artifact` | 可安全分享的诊断工件：保留有用状态，同时省略原始个人内容                                 |
| `personal-no-fake-progress`                | 有证据支持的完成声明，避免在本地证据存在前报告虚假进度                                   |
| `personal-failure-recovery`                | 故障恢复：报告部分状态，并清晰标明重试边界                                               |

机器可读的包元数据（ID 列表、标题、描述）位于
`extensions/qa-lab/src/scenario-packs.ts`，名称为 `QA_PERSONAL_AGENT_SCENARIO_IDS`。
使用 `--pack personal-agent` 运行该场景包：

```bash
OPENCLAW_ENABLE_PRIVATE_QA_CLI=1 pnpm openclaw qa suite \
  --provider-mode mock-openai \
  --pack personal-agent \
  --concurrency 1
```

`--pack` 可与重复的 `--scenario` 标志叠加使用。显式指定的场景会
先运行，然后场景包中的场景按照 `QA_PERSONAL_AGENT_SCENARIO_IDS` 顺序运行，
并移除重复项。

该场景包面向使用 `mock-openai` 或其他本地 QA 提供商
通道的 `qa-channel`。请勿将其指向实时聊天服务或真实个人账户。

## 隐私模型

场景仅使用模拟用户、模拟偏好、模拟密钥以及
测试套件创建的临时 QA Gateway 网关工作区。它们不得读取或
写入真实的 OpenClaw 用户记忆、会话、凭据、启动代理、全局
配置或实时 Gateway 网关状态。

工件保留在现有 QA 测试套件的工件目录下，并按
测试输出处理。脱敏检查使用模拟标记，因此可以安全地
检查失败并将其提交到议题中。

## 扩展场景包

在 `qa/scenarios/personal/` 下添加新的 `.yaml` 用例，然后将场景 ID
添加到 `QA_PERSONAL_AGENT_SCENARIO_IDS`。每个用例应保持小型、本地，并在
`mock-openai` 中具有确定性，且专注于一种个人助理行为。

适合作为后续工作的候选项：经过脱敏的轨迹导出检查、仅限本地的
插件工作流检查。

在场景目录拥有足够多的稳定用例、足以证明新增表面合理之前，
请避免添加新的运行器、插件、依赖项、实时传输或模型评审器。
