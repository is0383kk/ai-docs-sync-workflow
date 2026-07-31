---
read_when:
    - 你正在构建一个新的消息渠道插件
    - 你想将 OpenClaw 连接到消息平台
    - 你需要了解 `ChannelPlugin` 适配器接口面
sidebarTitle: Channel Plugins
summary: 为 OpenClaw 构建消息渠道插件的分步指南
title: 构建渠道插件
x-i18n:
    generated_at: "2026-07-26T07:38:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ff8ad04346babf3eece7e10bd38946ee290947b2e504b6b5ca438865531bf38
    source_path: plugins/sdk-channel-plugins.md
    workflow: 16
---

本指南将构建一个把 OpenClaw 连接到消息平台的渠道插件：私信安全、配对、回复线程和出站消息。

<Info>
  初次接触 OpenClaw 插件？请先阅读[入门指南](/zh-CN/plugins/building-plugins)，
  了解包结构和清单设置。
</Info>

## 插件负责的内容

渠道插件不实现发送、编辑或回应工具；核心提供一个
共享的 `message` 工具。你的插件负责：

- **配置** - 账号解析和设置向导
- **安全** - 私信策略和允许列表
- **配对** - 私信审批流程
- **会话语法** - 提供商特定的会话 ID 如何映射到基础
  聊天、线程 ID 和父级回退
- **出站** - 向平台发送文本、媒体和投票
- **线程化** - 回复如何组织到线程中
- **Heartbeat 输入状态** - 用于 Heartbeat 投递
  目标的可选输入中/忙碌信号

核心负责共享消息工具、提示词接线、外层会话键结构、
通用 `:thread:` 记录管理以及分派。

## 消息适配器

通过 `openclaw/plugin-sdk/channel-outbound` 中的 `defineChannelMessageAdapter` 公开一个
`message` 适配器。仅声明原生传输实际支持的持久最终发送
能力，并通过契约测试证明原生副作用和返回的回执。文本/媒体发送应
指向旧版 `outbound` 适配器所使用的相同传输函数。有关完整
API 契约、能力矩阵、回执规则、实时预览最终化、接收确认策略、测试
和迁移表，请参阅[渠道出站 API](/zh-CN/plugins/sdk-channel-outbound)。

如果现有的 `outbound` 适配器已经具备正确的发送方法和
能力元数据，请使用 `createChannelMessageAdapterFromOutbound(...)` 派生
`message` 适配器，而不是手写另一个
桥接层。适配器发送会返回 `MessageReceipt` 值。对于旧版 ID，请使用
`listMessageReceiptPlatformIds(...)` 或
`resolveMessageReceiptPrimaryId(...)` 派生，而不是保留并行的 `messageIds`
字段。

请准确声明实时能力和最终化器能力——核心据此决定渠道可以执行
哪些操作，而声明与实际行为之间的偏差会导致契约测试失败：

| 表面                                  | 值                                                                                               |
| ------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `message.live.capabilities`           | `draftPreview`、`previewFinalization`、`progressUpdates`、`nativeStreaming`、`quietFinalization` |
| `message.live.finalizer.capabilities` | `finalEdit`、`normalFallback`、`discardPending`、`previewReceipt`、`retainOnAmbiguousFailure`    |

需要原地最终化草稿预览的渠道，应通过
`defineFinalizableLivePreviewAdapter(...)` 和
`deliverWithFinalizableLivePreviewAdapter(...)` 路由运行时逻辑，并使用
`verifyChannelMessageLiveCapabilityAdapterProofs(...)` 和
`verifyChannelMessageLiveFinalizerProofs(...)` 测试为声明的能力提供保障，避免原生预览、
进度、编辑、回退/保留、清理和回执行为发生无声偏差。

延迟平台确认的入站接收器应声明
`message.receive.defaultAckPolicy` 和 `supportedAckPolicies`，而不是把
确认时机隐藏在监视器本地状态中。使用
`verifyChannelMessageReceiveAckPolicyAdapterProofs(...)` 覆盖每项声明的策略。

`dispatchInboundReplyWithBase` 和
`recordInboundSessionAndDispatchReply` 等旧版回复辅助函数仍可供兼容性
分派器使用。请勿在新的渠道代码中使用它们；应改用 `message`
适配器、回执以及 `openclaw/plugin-sdk/channel-outbound` 上的接收/发送生命周期辅助函数。

### 入站入口（实验性）

正在迁移入站授权的渠道可以从运行时接收路径使用实验性的
`openclaw/plugin-sdk/channel-ingress-runtime` 子路径。它接收平台事实、原始允许列表、路由描述符、命令
事实和访问组配置，然后返回发送者、路由、命令和激活投影以及有序的入口图，
而平台查询和副作用仍留在插件中。请在传给解析器的描述符中保留插件身份
规范化；不要序列化已解析状态或决策中的原始匹配值。有关 API 设计、
所有权边界和测试预期，请参阅
[频道入口 API](/zh-CN/plugins/sdk-channel-ingress)。

### 持久入口和重放去重

采用持久入口的渠道应使用 `openclaw/plugin-sdk/channel-outbound` 中的
`createChannelIngressMonitor`，除非它们需要实质上不同的准入或泵送契约。
在单一接收瓶颈处将原始传输信封加入队列（接收时不进行规范化）；
对于 webhook 传输，仅在持久追加完成后才允许传输确认；为每个会话
派生一条串行通道；并在分派采用事件时将其标记为完成。队列的主键是
`(queue_name, event_id)`，完成时为行添加墓碑而不是删除它，因此平台稍后再次
投递相同的 `event_id` 时，会在墓碑保留窗口内被持久拒绝。
有关监视器 API 和关闭契约，请参阅
[渠道出站 API](/zh-CN/plugins/sdk-channel-outbound#durable-ingress-monitors)。

该墓碑构成重放防护
（`openclaw/plugin-sdk/persistent-dedupe`）的分层规则：采用排空机制的渠道仅在防护的
身份或保留时间超出队列范围时，才保留单独的重放防护——例如，与传输投递
ID 不同的逻辑消息键（Telegram 会对 `chat_id:message_id` 去重，因为防抖
合并可能让消息以新的 `update_id` 再次出现），或比渠道墓碑
保留期更长的窗口。如果防护键与排空机制的 `event_id` 相同，
则在采用排空机制时删除该防护，并调整
`completedTtlMs`/`completedMaxEntries` 的大小以覆盖旧的防护窗口。
年龄限制等非去重保护与此规则无关。稳定的出站消息 ID 应使用
`openclaw/plugin-sdk/channel-outbound` 中的共享出站回显注册表，而不是渠道本地的 TTL 缓存。

#### 传输类别和保留策略

根据接收边界的恢复保证对传输进行分类：

- **确认门控的 webhook 或事件投递：**仅在持久追加后确认或返回成功。
  追加失败时，必须让投递仍可重试，或使接收边界失败。此类别包括 Slack、SMS、Zalo、
  Microsoft Teams、Google Chat、LINE 和 Synology Chat。
- **等待式轮询或流式投递：**仅在追加后推进远程游标或发送
  传输确认。如果没有显式游标，请保持接收回调串行并等待其完成，避免追加
  失败后接收循环仍继续向前运行。Telegram 轮询、Signal 和 Tlon 使用此类别；
  Telegram webhook 投递遵循上述确认门控规则。
- **不可重放的套接字：**IRC、Mattermost、Twitch 和 Zalo Personal 无法请求
  平台重新投递已接受的事件。它们的持久队列可防范进程崩溃窗口并支持本地
  重启恢复；完成墓碑对于平台重放几乎不起作用。

将 30 天作为整个服务集群的墓碑 TTL 约定，而非 SDK 默认值。高流量的
重新投递窗口通常使用 20,000 条已完成记录的上限；低流量的等待式传输和
不可重放传输通常使用 1,000-2,000 条。当前例外包括 LINE 的 4,096 条上限、
SMS 的 24 小时完成 TTL，以及 Tlon 仅设置上限的完成保留策略。失败行的上限
也可能低于已完成行的上限。TTL 和上限都会清理行，因此达到任一边界时，
实际保留期即告结束。仅可因有文档记录的平台重试范围、需要保留的已发布
重放防护窗口、预期流量或磁盘预算，或不可重放传输而偏离此规则，并使用
测试覆盖保留契约。

#### 至少一次副作用

排空分派会先运行命令副作用，然后入口行才会生成完成墓碑。如果进程在这两步
之间崩溃，该行会被重放，并可能再次执行副作用。这一“至少一次”的崩溃窗口
是默认契约。对于配置写入、存储清除或回复通道之外的可见确认等非幂等工作，
请使用 `openclaw/plugin-sdk/ingress-effect-once` 中的
`createIngressEffectOnce(...)`。为每次调用提供稳定的入口
`eventId` 和一个效果名称。为每个入口队列/账号创建一个辅助实例，并为该
作用域使用稳定且唯一的 `namespacePrefix`，因为传输事件 ID 可能仅在队列内唯一。
辅助函数仅在效果成功后提交其持久声明；抛出异常的效果会释放声明，以便排空
重试能够再次执行，而并发调用者会等待活动声明。持久状态发生错误时，如果
提供了 `onDiskError`，则调用它并拒绝操作，而不是回退到进程内存。

将辅助函数的 `ttlMs` 至少设置为渠道入口墓碑保留期，加上效果提交与
行完成之间的最大延迟，包括有界停机时间和排空重试。效果记录的 TTL 从提交时
开始，而墓碑保留期从之后的完成时开始；如果待处理行的生命周期无界，则任何
有限 TTL 都无法覆盖任意长度的停机时间。墓碑不再能够重放该行后，更旧的效果
记录便成为无用负担。根据该保留窗口中可能存在的每个不同事件/效果键设置
`stateMaxEntries` 的大小，同时考虑队列的已完成记录上限和每个事件的最大效果数。
更低的上限会在记录的 TTL 到期之前淘汰最旧记录，从而允许该效果再次执行。
如果进程在效果成功后、声明提交前终止，或持久化失败，或记录在其入口行仍处于
待处理状态时到期，仍会留下“至少一次”窗口。

#### 账号作用域的重启契约

默认情况下，渠道配置更改会重启整个渠道。多账号渠道仅可在配置解析只读取
渠道范围的共享字段和所选账号、绝不读取同级账号，并且 Gateway 网关能够停止
和启动单个 `(channel, accountId)` 运行时而无需替换同级运行时时，设置
`reload.accountScopedRestart: true`。

作用域路径仅适用于
`channels.<channel>.accounts.<non-default-id>.*` 下的更改。对共享渠道
字段、`accounts.default`、已删除或无法解析的账号所做的更改，以及可能影响
继承的混合更改，都会升级为整个渠道重启。未选择启用此功能的插件始终使用
整个渠道路径。

对于使用持久入口排空机制的渠道，账号监视器的停止路径必须先处理完所有已接受
的传输准入，然后释放并等待其排空机制完成。启动账号时会打开同一个以账号为键的
队列，其初始排空过程会恢复尚未分派的持久行。请勿添加第二个专用于重新加载的
重放过程；队列恢复是规范的重启路径。

请将此标志视为能力声明，而非性能偏好。契约测试应证明：添加和编辑一个具名账号
不会更改同级账号的已解析配置；停止一个账号仅会处理该账号的监视器和排空机制；
新的监视器会恰好恢复一次该账号的行。如果无法证明其中任一保证，请省略此标志。

### 输入状态指示器

如果渠道支持在入站回复之外显示输入状态指示器，请在渠道插件上公开
`heartbeat.sendTyping(...)`。核心会在 Heartbeat 模型运行开始前，使用已解析的
Heartbeat 投递目标调用它，并使用共享的输入状态保活/清理生命周期。如果平台
需要显式停止信号，请添加 `heartbeat.clearTyping(...)`。

### 媒体来源参数

如果渠道添加了携带媒体来源的消息工具参数，请通过
`plugin.actions.describeMessageTool(...).mediaSourceParams` 公开这些参数名称。
核心会使用此显式列表进行沙箱路径规范化和出站媒体访问策略处理，因此插件无需
针对提供商特定的头像、附件或封面图像参数在共享核心中添加特殊情况。

优先使用按操作键控的映射，例如 `{ "set-profile": ["avatarUrl", "avatarPath"] }`，
这样不相关的操作就不会继承其他操作的媒体参数。对于有意在所有公开操作之间共享的参数，
扁平数组仍然适用。

必须公开临时公共 URL 供平台侧获取媒体的渠道可以将
`openclaw/plugin-sdk/outbound-media` 中的 `createHostedOutboundMediaStore(...)`
与插件状态存储结合使用。平台路由解析和令牌强制验证应保留在渠道插件中；共享辅助函数
仅负责媒体加载、过期元数据、分块行和清理。

入站附件使用有序事实，而不是并行的 `Media*` 字段。使用
`openclaw/plugin-sdk/channel-inbound` 中的 `toInboundMediaFacts(...)`
规范化渠道记录，并在构建入站上下文时将其作为 `media` 传递。
当插件必须授权本地媒体读取时，从专用的
`openclaw/plugin-sdk/media-local-roots` 子路径导入
`getAgentScopedMediaLocalRoots(...)` 或
`getAgentScopedMediaLocalRootsForSources(...)`。旧的
`agent-media-payload` 构建器/根门面是已弃用的兼容层。

### 原生载荷塑形

如果你的渠道需要为 `message(action="send")` 进行提供商特定的塑形，
优先使用 `actions.prepareSendPayload(...)`。将原生卡片、块、嵌入内容或
其他持久数据放在 `payload.channelData.<channel>` 下，并让核心通过
出站/消息适配器发送。仅当载荷无法序列化和重试时，才使用
`actions.handleAction(...)` 发送，作为兼容性回退。

### 会话对话语法

如果你的平台在对话 ID 中存储额外的作用域，请使用
`messaging.resolveSessionConversation(...)` 将解析逻辑保留在插件中。这是将
`rawId` 映射到基础对话 ID、可选线程 ID、显式
`baseConversationId` 和任何
`parentConversationCandidates` 的规范钩子。返回 `parentConversationCandidates` 时，
请按从最窄父级到最宽泛/基础对话的顺序排列。

`messaging.resolveParentConversationCandidates(...)` 是已弃用的
兼容性回退，适用于只需在通用/原始 ID 之上添加父级回退的插件。
如果两个钩子都存在，核心会优先使用
`resolveSessionConversation(...).parentConversationCandidates`，并且仅在规范钩子省略相关内容时
回退到 `resolveParentConversationCandidates(...)`。

需要在渠道注册表启动前执行相同解析的内置插件，可以公开一个顶层
`session-key-api.ts` 文件，其中包含匹配的
`resolveSessionConversation(...)` 导出（参见 Feishu 和 Telegram
插件）。仅当运行时插件注册表尚不可用时，核心才会使用这个可安全引导的接口。

当插件代码需要规范化类似路由的字段、比较子线程与其父路由，或根据
`{ channel, to, accountId, threadId }` 构建稳定的去重键时，请使用
`openclaw/plugin-sdk/channel-route`。该辅助函数以与核心相同的方式
规范化数字线程 ID，因此应优先使用它，而不是临时的
`String(threadId)` 比较。具有提供商特定目标语法的插件
应公开 `messaging.resolveOutboundSessionRoute(...)`，使核心无需解析器兼容层即可获得
提供商原生的会话和线程标识。

### 账号范围的对话绑定支持

当渠道支持通用的当前对话绑定时，设置
`conversationBindings.supportsCurrentConversationBinding`。`createChatChannelPlugin(...)`
默认将此静态能力设置为 `true`。

如果不同已配置账号的支持情况不同，还应实现
`conversationBindings.isCurrentConversationBindingSupported({ accountId })`。
核心仅在静态能力启用后才会评估这个同步钩子。返回
`false` 会使该账号无法使用通用的当前对话能力、
绑定、查找、列出、触碰和解绑操作。省略该钩子会将静态能力应用于所有账号。

请从已加载的账号配置或运行时状态中解析答案。此钩子仅控制通用的当前对话绑定；
它不会替代已配置的绑定规则或插件自有的会话路由。契约测试应通过
`openclaw/plugin-sdk/channel-core` 导出的
`ChannelPlugin["conversationBindings"]` 契约，至少覆盖一个受支持账号和一个不受支持账号。

## 审批和渠道能力

大多数渠道插件不需要审批专用代码。核心负责同聊天
`/approve`、共享审批按钮载荷和通用回退投递。
`ChannelPlugin.approvals` 已被移除；应改为将审批投递/原生/渲染/身份验证
事实放在同一个 `approvalCapability` 对象上。`plugin.auth` 仅用于登录/登出，
核心不再从该对象读取审批身份验证钩子。

仅将 `approvalCapability.delivery` 用于原生审批路由或回退抑制，
并且仅在渠道确实需要自定义审批载荷而非共享渲染器时，才使用
`approvalCapability.render`。

### 审批身份验证

- `approvalCapability.authorizeActorAction` 和
  `approvalCapability.getActionAvailabilityState` 是规范的
  审批身份验证接口。
- 使用 `getActionAvailabilityState` 表示同聊天审批身份验证的可用性。
  即使原生投递已禁用，也应让已配置的审批人可用于 `/approve`；
  投递/设置指导应改用原生发起界面状态。
- 如果你的渠道公开原生 Exec 审批，请使用
  `approvalCapability.getExecInitiatingSurfaceState` 表示
  发起界面/原生客户端状态（当它与同聊天审批身份验证不同时）。
  核心使用这个 Exec 专用钩子区分 `enabled` 与
  `disabled`，判断发起渠道是否支持原生 Exec 审批，
  并在原生客户端回退指导中包含该渠道。
  `createApproverRestrictedNativeApprovalCapability(...)` 可处理
  常见情况。
- 如果渠道可以根据现有配置推断出稳定、类似所有者的私信身份，
  请使用 `openclaw/plugin-sdk/approval-runtime` 中的
  `createResolvedApproverActionAuthAdapter` 限制同聊天 `/approve`，
  而无需添加审批专用核心逻辑。
- 如果自定义审批身份验证有意仅允许同聊天回退，请从
  `openclaw/plugin-sdk/approval-auth-runtime` 返回
  `markImplicitSameChatApprovalAuthorization({ authorized: true })`；否则核心会将结果视为显式审批人授权。
- 如果渠道自有的原生回调直接解析审批，请在解析前使用
  `isImplicitSameChatApprovalAuthorization(...)`，以确保隐式回退仍经过渠道的常规参与者授权。

### 载荷生命周期和设置指导

- 使用 `outbound.shouldSuppressLocalPayloadPrompt` 或
  `outbound.beforeDeliverPayload` 实现渠道特定的载荷生命周期行为，
  例如隐藏重复的本地审批提示，或在投递前发送正在输入指示。
- 当渠道希望在禁用路径的回复中说明启用原生 Exec 审批
  所需的确切配置项时，请使用 `approvalCapability.describeExecApprovalSetup`。
  该钩子接收 `{ channel, channelLabel, accountId }`；
  具名账号渠道应渲染账号范围的路径，例如
  `channels.<channel>.accounts.<id>.execApprovals.*`，而不是顶层默认值。
- 当插件审批失败指导可以安全地用于插件审批无路由和超时失败时，
  请使用 `approvalCapability.describePluginApprovalSetup`。`createApproverRestrictedNativeApprovalCapability(...)`
  不会根据 `describeExecApprovalSetup` 推断这一点；仅当插件审批和 Exec 审批
  确实使用相同的原生设置时，才显式传递同一个辅助函数。

### 原生审批投递

如果渠道需要原生审批投递，请将渠道代码聚焦于目标规范化以及传输/呈现事实。
使用 `openclaw/plugin-sdk/approval-runtime` 中的
`createChannelExecApprovalProfile`、`createChannelNativeOriginTargetResolver`、
`createChannelApproverDmTargetResolver` 和
`createApproverRestrictedNativeApprovalCapability`。将渠道特定事实放在
`approvalCapability.nativeRuntime` 后面，最好通过
`createChannelApprovalNativeRuntimeAdapter(...)` 或
`createLazyChannelApprovalNativeRuntimeAdapter(...)` 实现，以便核心可以组装处理程序，并负责请求过滤、
路由、去重、过期、Gateway 网关订阅和已路由到其他位置的通知。

`nativeRuntime` 被拆分为几个更小的接口：

- `availability` - 账号是否已配置，以及是否应处理请求
- `presentation` - 将共享审批视图模型映射为
  待处理/已解析/已过期的原生载荷或最终操作
- `transport` - 准备目标，并发送/更新/删除原生审批消息
- `interactions` - 用于原生按钮或表情回应的可选绑定/解绑/清除操作钩子，
  以及可选的 `cancelDelivered` 钩子。当 `deliverPending`
  注册进程内或持久状态（例如表情回应目标存储）时，请实现
  `cancelDelivered`，这样，如果处理程序停止导致投递在
  `bindPending` 运行前被取消，或
  `bindPending` 未返回句柄，就可以释放该状态
- `observe` - 可选的投递诊断钩子

其他审批辅助函数：

- 当渠道同时支持会话来源的原生投递和显式审批转发目标时，
  请使用 `openclaw/plugin-sdk/approval-native-runtime` 中的
  `createNativeApprovalChannelRouteGates`。该辅助函数集中处理审批配置选择、
  `mode`、智能体/会话过滤器、账号绑定、会话目标匹配和目标列表匹配；
  调用方仍负责渠道 ID、默认转发模式、账号查找、传输启用检查、目标规范化和轮次来源
  目标解析。不要用它创建由核心拥有的渠道策略默认值；
  请显式传递该渠道已记录的默认模式。
- `createChannelNativeOriginTargetResolver` 默认对 `{ to, accountId, threadId }` 目标
  使用共享渠道路由匹配器。仅当渠道具有提供商特定的等价规则时，
  才传递 `targetsMatch`，例如 Slack 时间戳前缀匹配。
  当渠道需要在默认路由匹配器或自定义 `targetsMatch` 回调运行前
  规范化提供商 ID，同时保留原始投递目标时，请传递 `normalizeTargetForMatch`。
  仅当解析出的投递目标本身也应规范化时，才使用 `normalizeTarget`。
- 如果渠道需要运行时自有对象，例如客户端、令牌、Bolt
  应用或 webhook 接收器，请通过
  `openclaw/plugin-sdk/channel-runtime-context` 注册它们。通用运行时上下文
  注册表让核心能够从渠道启动状态引导由能力驱动的处理程序，
  而无需添加审批专用的包装粘合代码。
- 仅当能力驱动的接口表达能力尚且不足时，
  才使用较低层级的 `createChannelApprovalHandler` 或
  `createChannelNativeApprovalRuntime`。
- 原生审批渠道必须通过这些辅助函数路由
  `accountId` 和 `approvalKind`。
  `accountId` 将多账号审批策略限定在正确的机器人账号范围内，
  而 `approvalKind` 则让渠道能够使用 Exec 与插件审批行为，
  无需在核心中硬编码分支。
- 核心也负责审批重新路由通知。渠道插件不应从
  `createChannelNativeApprovalRuntime` 发送自己的“审批已转到私信/其他渠道”后续消息；
  应通过共享审批能力辅助函数公开准确的来源 + 审批人私信路由，
  并让核心汇总实际投递情况后，再向发起聊天发布任何通知。
- 应端到端保留已投递审批 ID 的类型。原生客户端不应
  根据渠道本地状态猜测或改写 Exec 与插件审批路由。
- 将显式的 `approvalKind` 传递给 `resolveApprovalOverGateway`。
  这会使用规范的 `approval.resolve` 服务，并在其他界面先响应时
  返回已记录的胜出者。较旧的显式 `resolveMethod` 输入
  仍保留给命令支持的控件；新的原生操作不得使用它，也不得根据 ID 推断类型。
- 不同审批类型可以有意公开不同的原生界面。
  当前内置示例：Matrix 对 Exec 和插件审批保留相同的原生私信/渠道路由和
  表情回应用户体验，同时仍允许身份验证因审批类型而异；Slack 则为 Exec 和
  插件 ID 都保留原生审批路由。
- `createApproverRestrictedNativeApprovalAdapter` 仍作为
  兼容性包装器存在，但新代码应优先使用能力构建器，并在插件上公开
  `approvalCapability`。

### 更窄的审批运行时子路径

对于高频渠道入口点，如果只需要该系列的一部分，
请优先使用以下更窄的子路径，而不是更宽泛的
`approval-runtime` 桶形导出：

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-gateway-runtime`
- `openclaw/plugin-sdk/approval-reference-runtime`
- `openclaw/plugin-sdk/approval-handler-adapter-runtime`
- `openclaw/plugin-sdk/approval-handler-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`
- `openclaw/plugin-sdk/channel-runtime-context`

同样，如果并非全部需要，请优先使用 `openclaw/plugin-sdk/reply-runtime`、
`openclaw/plugin-sdk/reply-dispatch-runtime`、
`openclaw/plugin-sdk/reply-reference` 和
`openclaw/plugin-sdk/reply-chunking`，而不是范围更广的总括接口。

### 设置子路径

- `openclaw/plugin-sdk/setup-runtime` 涵盖可安全用于运行时的设置辅助工具：
  `createSetupTranslator`、可安全导入的设置补丁适配器
  （`createPatchedAccountSetupAdapter`、`createEnvPatchedAccountSetupAdapter`、
  `createSetupInputPresenceValidator`）、查找说明输出、
  `promptResolvedAllowFrom`、`splitSetupEntries`，以及委托式
  设置代理构建器。
- `openclaw/plugin-sdk/channel-setup` 涵盖可选安装设置
  构建器以及一些可安全用于设置的基础组件：`createOptionalChannelSetupSurface`、
  `createOptionalChannelSetupAdapter`、`createOptionalChannelSetupWizard`、
  `DEFAULT_ACCOUNT_ID`、`createTopLevelChannelDmPolicy`、
  `setSetupChannelEnabled` 和 `splitSetupEntries`。
- 仅当还需要更重型的共享设置/配置辅助工具（例如
  `moveSingleAccountChannelSectionToDefaultAccount(...)`）时，才使用范围更广的
  `openclaw/plugin-sdk/setup` 接口。

如果你的渠道只需在设置界面中提示“请先安装此插件”，
请优先使用 `createOptionalChannelSetupSurface(...)`。生成的
适配器/向导会在配置写入和最终确定时采用失败关闭策略，并在验证、
最终确定和文档链接文案中复用同一条要求安装的消息。

如果你的渠道支持通过环境变量进行设置或身份验证，请通过
渠道配置 schema 和设置描述符公开这些功能。渠道运行时中的 `envVars` 或
本地常量仅用于面向操作员的文案。

如果你的渠道可能在插件运行时启动前出现在 `status`、`channels list`、`channels status` 或
SecretRef 扫描中，请在
`package.json` 中添加 `openclaw.setupEntry`。该入口点应能在只读命令
路径中安全导入，并应返回这些摘要所需的渠道元数据、可安全用于设置的配置适配器、
状态适配器和渠道密钥目标元数据。不要从
设置入口启动客户端、监听器或传输运行时。

同时保持主渠道入口的导入路径精简。设备发现可以对
入口和渠道插件模块求值以注册能力，而无需
激活渠道。诸如 `channel-plugin-api.ts` 的文件应导出
渠道插件对象，但不应导入设置向导、传输
客户端、套接字监听器、子进程启动器或服务启动模块。
请将这些运行时组件放入从 `registerFull(...)`、运行时
setter 或惰性能力适配器加载的模块中。

### 其他精简的渠道子路径

对于其他渠道热路径，请优先使用精简的辅助工具，而不是范围更广的旧版
接口：

- `openclaw/plugin-sdk/account-core`、`openclaw/plugin-sdk/account-id`、
  `openclaw/plugin-sdk/account-resolution` 和
  `openclaw/plugin-sdk/account-helpers`，用于多账户配置和
  默认账户回退
- `openclaw/plugin-sdk/inbound-envelope` 和
  `openclaw/plugin-sdk/channel-inbound`，用于入站路由/信封及
  记录并分派的连线
- `openclaw/plugin-sdk/channel-targets`，用于目标解析辅助工具
- `openclaw/plugin-sdk/channel-outbound`，用于出站身份/发送委托
  和类型化负载规划
- 当出站路由应保留显式
  `replyToId`/`threadId`，或在基础会话键仍然匹配后恢复当前 `:thread:`
  会话时，使用 `openclaw/plugin-sdk/channel-core` 中的
  `buildThreadAwareOutboundSessionRoute(...)`。当提供商的平台具有原生线程投递语义时，
  提供商插件可以覆盖优先级、后缀行为和线程 ID 规范化。
- `openclaw/plugin-sdk/thread-bindings-runtime`，用于线程绑定生命周期
  和适配器注册

仅支持身份验证的渠道通常可止步于默认路径：核心处理
审批，而插件只公开出站/身份验证能力。Matrix、Slack、Telegram 等原生
审批渠道以及自定义聊天传输应使用共享原生辅助工具，而不是自行实现审批
生命周期。

## 入站提及策略

将入站提及处理分为两层：

- 插件自行收集证据
- 共享策略评估

使用 `openclaw/plugin-sdk/channel-mention-gating` 做出提及策略决策。
仅当需要范围更广的
入站辅助工具桶文件时，才使用 `openclaw/plugin-sdk/channel-inbound`。

适合放在插件本地逻辑中的内容：

- 检测是否回复机器人
- 检测是否引用机器人
- 检查线程参与情况
- 排除服务/系统消息
- 证明机器人参与情况所需的平台原生缓存

适合共享辅助工具的内容：

- `requireMention`
- 显式提及结果
- 隐式提及允许列表
- 命令绕过
- 最终跳过决策

推荐流程：

1. 计算本地提及事实。
2. 将这些事实传入 `resolveInboundMentionDecision({ facts, policy })`。
3. 在入站门控中使用 `decision.effectiveWasMentioned`、`decision.shouldBypassMention` 和
   `decision.shouldSkip`。

```typescript
import {
  implicitMentionKindWhen,
  matchesMentionWithExplicit,
  resolveInboundMentionDecision,
} from "openclaw/plugin-sdk/channel-inbound";
import { resolveChannelImplicitMentions } from "openclaw/plugin-sdk/channel-ingress-runtime";

const wasMentioned = matchesMentionWithExplicit({
  text,
  mentionRegexes,
  explicit: {
    hasAnyMention,
    isExplicitlyMentioned,
    canResolveExplicit,
  },
});

const facts = {
  canDetectMention: true,
  wasMentioned,
  hasAnyMention,
  implicitMentionKinds: [
    ...implicitMentionKindWhen("reply_to_bot", isReplyToBot),
    ...implicitMentionKindWhen("quoted_bot", isQuoteOfBot),
  ],
};

const implicitMentions = resolveChannelImplicitMentions({
  cfg,
  channel: channelId,
  accountId,
});

const decision = resolveInboundMentionDecision({
  facts,
  policy: {
    isGroup,
    requireMention,
    implicitMentions,
    allowTextCommands,
    hasControlCommand,
    commandAuthorized,
  },
});

if (decision.shouldSkip) return;
```

`matchesMentionWithExplicit(...)` 返回布尔值。`hasAnyMention`、
`isExplicitlyMentioned` 和 `canResolveExplicit` 来自渠道自身的
原生提及元数据（消息实体、回复机器人标志及类似信息）；
当你的平台无法检测这些值时，请提供 `false`/`undefined` 值。

`api.runtime.channel.mentions` 为已经依赖运行时注入的
内置渠道插件公开相同的共享提及辅助工具：
`buildMentionRegexes`、`matchesMentionPatterns`、`matchesMentionWithExplicit`、
`implicitMentionKindWhen`、`resolveInboundMentionDecision`。

如果只需要 `implicitMentionKindWhen` 和 `resolveInboundMentionDecision`，
请从 `openclaw/plugin-sdk/channel-mention-gating` 导入，以避免加载
无关的入站运行时辅助工具。

## 演练

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="包和清单">
    创建标准插件文件。`openclaw.plugin.json` 中的
    `channels` 字段（而非 `kind` 字段）用于标记清单
    拥有某个渠道。有关完整的包元数据接口，请参阅
    [插件设置和配置](/zh-CN/plugins/sdk-setup#openclaw-channel)：

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-chat",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "acme-chat",
          "label": "Acme Chat",
          "blurb": "将 OpenClaw 连接到 Acme Chat。"
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "acme-chat",
      "channels": ["acme-chat"],
      "name": "Acme Chat",
      "description": "Acme Chat 渠道插件",
      "configSchema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {}
      },
      "channelConfigs": {
        "acme-chat": {
          "schema": {
            "type": "object",
            "additionalProperties": false,
            "properties": {
              "token": { "type": "string" },
              "allowFrom": {
                "type": "array",
                "items": { "type": "string" }
              }
            }
          },
          "uiHints": {
            "token": {
              "label": "机器人令牌",
              "sensitive": true
            }
          }
        }
      }
    }
    ```
    </CodeGroup>

    `configSchema` 验证 `plugins.entries.acme-chat.config`。将它用于
    不属于渠道账户配置的插件自有设置。
    `channelConfigs.acme-chat.schema` 验证 `channels.acme-chat`，并且是
    插件运行时加载前供配置 schema、设置和 UI 界面使用的冷路径
    数据源。有关完整的顶层字段参考，请参阅[插件清单](/zh-CN/plugins/manifest)。

  </Step>

  <Step title="构建渠道插件对象">
    `ChannelPlugin` 接口包含许多可选适配器接口。请从
    最小集合——`id`、`config` 和 `setup`——开始，并根据
    需要添加适配器。

    创建 `src/channel.ts`：

    ```typescript src/channel.ts
    import {
      createChatChannelPlugin,
      createChannelPluginBase,
    } from "openclaw/plugin-sdk/channel-core";
    import type { OpenClawConfig } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatApi } from "./client.js"; // your platform API client

    type ResolvedAccount = {
      accountId: string | null;
      token: string;
      allowFrom: string[];
      dmPolicy: string | undefined;
    };

    function resolveAccount(
      cfg: OpenClawConfig,
      accountId?: string | null,
    ): ResolvedAccount {
      const section = (cfg.channels as Record<string, any>)?.["acme-chat"];
      const token = section?.token;
      if (!token) throw new Error("acme-chat: token is required");
      return {
        accountId: accountId ?? null,
        token,
        allowFrom: section?.allowFrom ?? [],
        dmPolicy: section?.dmSecurity,
      };
    }

    export const acmeChatPlugin = createChatChannelPlugin<ResolvedAccount>({
      base: createChannelPluginBase({
        id: "acme-chat",
        // Account resolution/inspection belongs on `config`, not `setup`.
        // `setup` covers onboarding writes (applyAccountConfig, validateInput).
        config: {
          listAccountIds: () => ["default"],
          resolveAccount,
          inspectAccount(cfg, accountId) {
            const section =
              (cfg.channels as Record<string, any>)?.["acme-chat"];
            return {
              enabled: Boolean(section?.token),
              configured: Boolean(section?.token),
              tokenStatus: section?.token ? "available" : "missing",
            };
          },
        },
        setup: {
          applyAccountConfig: ({ cfg, input }) => ({
            ...cfg,
            channels: {
              ...cfg.channels,
              "acme-chat": { ...(cfg.channels as any)?.["acme-chat"], ...input },
            },
          }),
        },
      }),

      // DM security: who can message the bot
      security: {
        dm: {
          channelKey: "acme-chat",
          resolvePolicy: (account) => account.dmPolicy,
          resolveAllowFrom: (account) => account.allowFrom,
          defaultPolicy: "allowlist",
        },
      },

      // Pairing: approval flow for new DM contacts
      pairing: {
        text: {
          idLabel: "Acme Chat username",
          message: "Send this code to verify your identity:",
          notify: async ({ target, code }) => {
            await acmeChatApi.sendDm(target, `Pairing code: ${code}`);
          },
        },
      },

      // Threading: how replies are delivered
      threading: { topLevelReplyToMode: "reply" },

      // Outbound: send messages to the platform
      outbound: {
        attachedResults: {
          channel: "acme-chat",
          sendText: async (params) => {
            const result = await acmeChatApi.sendMessage(
              params.to,
              params.text,
            );
            return { messageId: result.id };
          },
        },
        base: {
          sendMedia: async (params) => {
            await acmeChatApi.sendFile(params.to, params.filePath);
          },
        },
      },
    });
    ```

    对于同时接受规范顶层私信键和旧版嵌套键的渠道，请使用 `plugin-sdk/channel-config-helpers` 中的辅助函数：`resolveChannelDmAccess`、`resolveChannelDmPolicy`、`resolveChannelDmAllowFrom` 和 `normalizeChannelDmPolicy`，以确保账户本地值优先于继承的根值。将同一个解析器与通过 `normalizeLegacyDmAliases` 执行的 Doctor 修复配合使用，使运行时和迁移读取同一契约。

    <Accordion title="createChatChannelPlugin 为你完成的工作">
      无需手动实现底层适配器接口，只需传入声明式选项，构建器便会组合它们：

      | 选项 | 接入的功能 |
      | --- | --- |
      | `security.dm` | 从配置字段解析作用域内的私信安全策略 |
      | `pairing.text` | 通过代码交换实现基于文本的私信配对流程 |
      | `threading` | 回复模式解析器（固定、账户作用域或自定义） |
      | `outbound.attachedResults` | 返回结果元数据（消息 ID）的发送函数；需要同级的 `channel` ID，以便核心为返回的投递结果添加标记 |

      如果需要完全控制，也可以传入原始适配器对象，而不是声明式选项。

      原始出站适配器可以定义 `chunker(text, limit, ctx)` 函数。
      可选的 `ctx.formatting` 携带投递时的格式化决策，
      例如 `maxLinesPerMessage`；请在发送前应用它，以便共享出站投递只需解析一次
      回复线程和分块边界。
      当原生回复目标已解析时，发送上下文还会包含 `replyToIdSource`
      （`implicit` 或 `explicit`），使有效负载辅助函数能够保留
      显式回复标签，而不会占用隐式的一次性回复槽位。
    </Accordion>

    ### 群组工具策略适配器

    实现 `group.resolveToolPolicy` 并支持
    `toolsBySender` 的渠道必须将完整的 `ChannelGroupContext` 转发给其
    共享策略解析器。尤其要遵循 `senderPolicyMode: "never"`：
    在匹配群组和通配符作用域中都跳过特定于发送者的覆盖层，
    同时仍应用基础 `tools` 策略。

    OpenClaw 仅为可信的非入口执行设置此模式，此类执行的发送者权限
    已捕获在服务器所有的信封中，例如显式限制能力的定时运行。
    插件不得从入站元数据推导此模式、将其持久化为渠道状态，
    或将其公开为配置。请添加一个适配器测试，证明此模式会跳过通配符
    `toolsBySender` 条目，同时不会丢弃匹配的基础
    `tools` 限制。

  </Step>

  <Step title="接入入口点">
    创建 `index.ts`：

    ```typescript index.ts
    import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineChannelPluginEntry({
      id: "acme-chat",
      name: "Acme Chat",
      description: "Acme Chat channel plugin",
      plugin: acmeChatPlugin,
      registerCliMetadata(api) {
        api.registerCli(
          ({ program }) => {
            program
              .command("acme-chat")
              .description("Acme Chat management");
          },
          {
            descriptors: [
              {
                name: "acme-chat",
                description: "Acme Chat management",
                hasSubcommands: false,
              },
            ],
          },
        );
      },
      registerFull(api) {
        api.registerGatewayMethod(/* ... */);
      },
    });
    ```

    将渠道所有的 CLI 描述符放入 `registerCliMetadata(...)`，这样 OpenClaw
    无需激活完整渠道运行时即可在根帮助中显示它们，
    而常规完整加载仍会获取相同描述符以实际注册命令。
    将 `registerFull(...)` 保留给仅运行时工作。
    `defineChannelPluginEntry` 会自动处理注册模式的拆分。
    如果 `registerFull(...)` 注册 Gateway 网关 RPC 方法，请使用
    插件专用前缀。核心管理命名空间（`config.*`、
    `exec.approvals.*`、`wizard.*`、`update.*`）保持保留，并且始终
    解析为 `operator.admin`。有关所有选项，请参阅
    [入口点](/zh-CN/plugins/sdk-entrypoints#definechannelpluginentry)。

  </Step>

  <Step title="添加设置入口">
    创建 `setup-entry.ts`，用于新手引导期间的轻量加载：

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    当渠道被禁用或未配置时，OpenClaw 会加载此入口，而不是完整入口。
    这样可以避免在设置流程中载入繁重的运行时代码。
    详情请参阅[设置和配置](/zh-CN/plugins/sdk-setup#setup-entry)。

    将设置安全导出拆分到附属模块中的内置工作区渠道，
    如果还需要显式的设置时运行时设值函数，
    可以使用 `openclaw/plugin-sdk/channel-entry-contract` 中的
    `defineBundledChannelSetupEntry(...)`。

  </Step>

  <Step title="处理入站消息">
    你的插件需要从平台接收消息并将其转发给 OpenClaw。
    典型模式是使用 Webhook 验证请求，然后通过渠道的入站处理程序分发请求：

    ```typescript
    registerFull(api) {
      api.registerHttpRoute({
        path: "/acme-chat/webhook",
        auth: "plugin", // plugin-managed auth (verify signatures yourself)
        handler: async (req, res) => {
          const event = parseWebhookPayload(req);

          // Your inbound handler dispatches the message to OpenClaw.
          // The exact wiring depends on your platform SDK -
          // see a real example in the bundled Microsoft Teams or Google Chat plugin package.
          await handleAcmeChatInbound(api, event);

          res.statusCode = 200;
          res.end("ok");
          return true;
        },
      });
    }
    ```

    <Note>
      入站消息处理因渠道而异。每个渠道插件都拥有自己的入站管道。
      请参考内置渠道插件（例如 Microsoft Teams 或 Google Chat 插件包）
      中的实际模式。
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="测试">
在 `src/channel.test.ts` 中编写同目录测试：

    ```typescript src/channel.test.ts
    import { describe, it, expect } from "vitest";
    import { acmeChatPlugin } from "./channel.js";

    describe("acme-chat plugin", () => {
      it("resolves account from config", () => {
        const cfg = {
          channels: {
            "acme-chat": { token: "test-token", allowFrom: ["user1"] },
          },
        } as any;
        const account = acmeChatPlugin.config.resolveAccount(cfg, undefined);
        expect(account.token).toBe("test-token");
      });

      it("inspects account without materializing secrets", () => {
        const cfg = {
          channels: { "acme-chat": { token: "test-token" } },
        } as any;
        const result = acmeChatPlugin.config.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(true);
        expect(result.tokenStatus).toBe("available");
      });

      it("reports missing config", () => {
        const cfg = { channels: {} } as any;
        const result = acmeChatPlugin.config.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(false);
      });
    });
    ```

    ```bash
    pnpm test <bundled-plugin-root>/acme-chat/
    ```

    有关共享测试辅助函数，请参阅[测试](/zh-CN/plugins/sdk-testing)。

</Step>
</Steps>

## 文件结构

```text
<bundled-plugin-root>/acme-chat/
├── package.json              # openclaw.channel metadata
├── openclaw.plugin.json      # Manifest with config schema
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # Public exports (optional)
├── runtime-api.ts            # Internal runtime exports (optional)
└── src/
    ├── channel.ts            # ChannelPlugin via createChatChannelPlugin
    ├── channel.test.ts       # Tests
    ├── client.ts             # Platform API client
    └── runtime.ts            # Runtime store (if needed)
```

## 高级主题

<CardGroup cols={2}>
  <Card title="线程选项" icon="git-branch" href="/zh-CN/plugins/sdk-entrypoints#registration-mode">
    固定、账户范围或自定义回复模式
  </Card>
  <Card title="消息工具集成" icon="puzzle" href="/zh-CN/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool 和操作发现
  </Card>
  <Card title="目标解析" icon="crosshair" href="/zh-CN/plugins/architecture-internals#channel-target-resolution">
    inferTargetChatType、looksLikeId、reservedLiterals、resolveTarget
  </Card>
  <Card title="运行时辅助工具" icon="settings" href="/zh-CN/plugins/sdk-runtime">
    通过 api.runtime 使用 TTS、STT、媒体和子智能体
  </Card>
  <Card title="频道入口 API" icon="bolt" href="/zh-CN/plugins/sdk-channel-inbound">
    共享入口事件生命周期：接收、解析、记录、分派、完成
  </Card>
</CardGroup>

<Note>
为维护内置插件及实现兼容性，目前仍保留了一些内置辅助接口。
不建议新渠道插件采用这些模式；除非你直接维护对应的内置插件系列，
否则应优先使用公共 SDK 接口中的通用 channel/setup/reply/runtime 子路径。
</Note>

## 后续步骤

- [提供商插件](/zh-CN/plugins/sdk-provider-plugins) - 如果你的插件还提供模型
- [SDK 概览](/zh-CN/plugins/sdk-overview) - 完整的子路径导入参考
- [SDK 测试](/zh-CN/plugins/sdk-testing) - 测试工具和契约测试
- [插件清单](/zh-CN/plugins/manifest) - 完整的清单架构

## 相关内容

- [插件 SDK 设置](/zh-CN/plugins/sdk-setup)
- [构建插件](/zh-CN/plugins/building-plugins)
- [Agent harness plugins](/zh-CN/plugins/sdk-agent-harness)
