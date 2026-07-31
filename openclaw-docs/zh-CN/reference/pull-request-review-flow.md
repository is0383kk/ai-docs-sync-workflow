---
read_when:
    - 跟进 Barnacle 或 ClawSweeper 的反馈
    - 请求 ClawSweeper 进行审查
    - 调试 Barnacle、ClawSweeper、过期标签或自动关闭问题
sidebarTitle: PR review flow
summary: Barnacle 和 ClawSweeper 的反馈如何帮助推动 OpenClaw 拉取请求通过审查。
title: PR 审查流程
x-i18n:
    generated_at: "2026-07-26T06:01:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e9bec4578d55d2279450e991480467946db7da5ca956f85c35b4221190b2babe
    source_path: reference/pull-request-review-flow.md
    workflow: 16
---

本页说明你在创建或更新 OpenClaw PR 后的审查流程：Barnacle 和 ClawSweeper 的作用、如何根据它们的反馈改进 PR，以及自动化长时间没有响应时应检查哪些事项。

Barnacle 和 ClawSweeper 帮助维护者维持审查队列的可用性，但不能取代维护者的判断。

## Barnacle

Barnacle 是采用确定性规则的 GitHub 分类工具。它会查找已知的队列管理情形，并通过添加标签、发表评论或关闭条目进行响应。

Barnacle 可能会在以下情况下采取行动：

- PR 正文几乎为空或缺少问题背景；
- PR 没有有效证据；
- 仅涉及文档、测试、重构、CI 或基础设施的更改缺少关联的维护者背景信息；
- 更改看起来应归入 ClawHub 或插件，而不是核心；
- 分支包含无关工作；
- 作者有超过 20 个未关闭的 PR。

Barnacle 通过受信任的仓库工作流代码运行。它不会检出或运行贡献者代码。

大多数路由标签都是供维护者或自动化使用的信号，因此贡献者无需自行添加标签。

## ClawSweeper

ClawSweeper 是用于 OpenClaw 仓库的 AI 辅助审查和维护机器人。它可以审查 PR、评估验证证据、留下持久的审查评论，并通过受保护的修复或自动合并流程协助维护者。

ClawSweeper 的正面结果属于支持性证据，并不代表维护者批准。维护者仍需决定 PR 是否以及何时可以合并。

ClawSweeper 采用队列机制。创建 PR、推送提交或添加审查请求后，不要期待它立即响应。ClawSweeper 运行后的标签更新也可能需要一些时间。

新 PR 会进入 ClawSweeper 审查队列。维护者也可以通过标签或命令将审查、修复或自动合并流程加入队列。对于贡献者的一般更新，仅在更新了分支、PR 描述、证据或代码后，才请求 ClawSweeper 再次审查。然后通过新的 PR 评论请求重新审查：

```text
@clawsweeper re-review
```

PR 作者也可以使用 `@clawsweeper re-run`；拥有仓库写入权限的用户可以对任何未关闭条目使用其中任一命令。普通的 `@clawsweeper review` 命令仅供维护者使用。请耐心等待：在完成所要求的更改之前再次发出请求只会增加队列噪声。

当 ClawSweeper 发起审查对话时，应将其视为普通审查反馈，并使用下方的后续检查清单。

如果已有贡献者或维护者接手 PR 并在积极处理，请勿同时召唤 ClawSweeper 或以其他方式处理该 PR。应先让人工审查或修复完成。如果活动停止，请检查是否已要求作者提供证据或进行其他更新。

## 在审查期间改进 PR

Barnacle、ClawSweeper 或维护者响应后，请将其反馈作为该 PR 后续步骤的检查清单。

1. 将 ClawSweeper 的 `Rank-up moves:` 和 `Proof guidance:` 视为该 PR 的行动清单。评级和标签是审查信号，并非固定的合并目标。
2. 推送所要求的代码或文档更改；如果问题、解决方案、用户影响或证据发生变化，还需更新 PR 描述。
3. 添加所要求的验证材料，使用与更改相匹配的证据。
4. 自行解决已处理的审查对话。仅在需要维护者或审查者判断时回复并保持对话未解决。
5. 仅在分支、PR 描述、证据和相关 CI 结果均为最新状态后请求重新审查。作者、维护者和 ClawSweeper 之间进行多轮更新和审查是正常现象。
6. 尽可能在 PR 中讨论。仅当 PR 需要维护者协调、自动化似乎受阻，或下一个决定难以通过 GitHub 评论达成一致时，才转到 Discord 上的 `#clawtributors`。请附上 PR 链接、当前状态以及具体问题或尚缺的证据。

保持 PR 正文为最新状态。评论有助于讨论，但 PR 描述才是维护者和自动化会再次查看的持久摘要。

`status: ⏳ waiting on author` 表示下一步应由 PR 作者处理：更新分支、PR 描述或证据，或者回复并补充缺失的背景信息，然后再请求重新审查。

有效证据包括针对性的测试输出、CI 结果、屏幕截图、录屏、终端输出、实时观察结果、经过脱敏的日志或工件链接。对于视觉更改，在可行时应提供更改前后的屏幕截图。对于证明文件，优先提供 CI 工件、上传到 GitHub 的屏幕截图或录屏链接，或简短的脱敏日志摘录。除非生成的证明文件属于实际文档、测试或产品更改的一部分，否则不要将其提交到仓库。

贡献者有责任对敏感数据进行脱敏。发布证据前，请移除密钥、令牌、私有 URL、用户数据和无关日志。

OpenClaw 还使用独立的陈旧条目自动化机制。未分配的议题和 PR 在连续 14 天无活动后可能被标记为陈旧，继续闲置 7 天后将被关闭。已分配的 PR 会在创建 27 天后被标记为陈旧，无论之后是否有更新；如果随后连续 7 天仍无活动，则会被关闭。如果已分配的 PR 仍在积极处理中，请与负责该 PR 的维护者协调。

## 自动化长时间没有响应时

当维护者已在处理条目、审查或修复请求仍在队列中、事件属于常规情况，或者 ClawSweeper 通道未配置所请求的操作时，自动化可能不会响应。

如果受信任的工作流需要运行不受信任的贡献者代码，它也可能不会采取行动。在这种情况下，维护者会改用常规审查或更安全的工作流。

## 故障排查

如果 ClawSweeper 没有立即响应，请先等待一段时间再重试。该服务采用队列机制，重复发表评论或更改标签只会使对话更难审查，并不会加快队列处理速度。

寻求帮助前，请检查：

- PR 描述为最新状态；
- 最新提交包含所要求的更改；
- CI 已完成，或者 PR 正文已说明为何任何仍存在的失败均与该 PR 无关；
- 最新审查请求已通过 PR 评论发出：
  `@clawsweeper re-review`；
- 没有维护者或贡献者正在积极处理该 PR；
- 最新请求已超过 ClawSweeper 的正常队列等待时间。

如果 PR 已更新到最新状态数小时后 ClawSweeper 仍未响应，或者 PR 看起来被自动化阻塞，请在 Discord 上的 `#clawtributors` 中寻求帮助。请附上 PR 链接、你的预期、发出请求的时间，以及自机器人上次评论以来发生的更改。

## 复用自动化

希望使用类似审查自动化的项目可以研究或复刻 ClawSweeper：

- [openclaw/clawsweeper](https://github.com/openclaw/clawsweeper)
- [ClawSweeper 文档](https://clawsweeper.bot/)

## 相关内容

- [贡献指南](https://github.com/openclaw/openclaw/blob/main/CONTRIBUTING.md)
- [CI 流水线](/zh-CN/ci)
