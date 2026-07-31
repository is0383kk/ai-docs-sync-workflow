---
read_when:
    - 你想在 OpenClaw 中使用 OpenAI 模型
    - 你希望使用 Codex 订阅身份验证，而不是 API key
    - 你需要更严格的 GPT-5 智能体执行行为
summary: 在 OpenClaw 中通过 API 密钥或 Codex 订阅使用 OpenAI
title: OpenAI
x-i18n:
    generated_at: "2026-07-26T06:57:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 612a36760899e01126364ddca523f0a6340036253cf349ae2755ba15c6451ba6
    source_path: providers/openai.md
    workflow: 16
---

OpenClaw 对直接 API 密钥身份验证和
ChatGPT/Codex 订阅身份验证使用同一个提供商 ID：`openai`。`openai/*` 是规范模型路由。
对于未设置运行时策略或策略为 `auto` 的嵌入式智能体轮次，OpenAI 的路由
信息决定 OpenClaw 是否可以隐式选择内置的 Codex app-server 运行时。
仅有 `openai/*` 前缀不会选择运行时。

- **智能体模型** - 通过显式 `agentRuntime` 配置或 OpenAI 的隐式路由策略所选的运行时使用 `openai/*`。
  若要使用 ChatGPT/Codex 订阅，请通过 Codex
  身份验证登录；若要按密钥计费，请配置 API 密钥身份验证
  配置文件。
- **非智能体 OpenAI API** - 通过 `OPENAI_API_KEY` 或 `openai` API 密钥身份验证配置文件
  直接访问 OpenAI Platform，并按使用量计费。
- **旧版配置** - `openclaw doctor --fix` 会将 `codex/*` 和 `openai-codex/*` 引用修复为
  `openai/*`，并添加模型范围的 `agentRuntime.id: "codex"`。

OpenAI 明确支持在 OpenClaw 等外部工具和
工作流中使用订阅 OAuth。

## 使用量和成本跟踪

OpenClaw 将订阅配额与 Platform API 计费分开处理：

- ChatGPT/Codex OAuth 显示订阅方案、配额周期和点数余额。
- `OPENAI_ADMIN_KEY` 在 Control UI **使用量**中显示提供商报告的 30 天组织成本和补全使用量，包括每日支出、请求/令牌总量、热门模型和成本类别。
- `OPENAI_PROJECT_ID` 可选择将 Admin API 历史记录限定到一个项目。
- OpenClaw 绝不会将 `OPENAI_API_KEY` 或 `openai` 推理配置文件发送到组织 API；这些凭据可能属于自定义、Azure 或智能体本地端点。

显式 Admin 密钥的优先级高于 OAuth。提供商报告的历史记录不会与 OpenClaw 根据会话估算的成本合并；其中可能包括其他客户端产生的 API 活动以及提供商侧的计费调整。

OpenAI 的 [API 使用量仪表板](https://help.openai.com/en/articles/10478918)文档介绍了获取使用量数据所需的组织所有者权限和显式 Usage Dashboard 权限。

提供商、模型、运行时和渠道是相互独立的层。如果这些标签
混淆在一起，请先阅读 [Agent Runtimes](/zh-CN/concepts/agent-runtimes)，再
更改配置。

## 快速选择

| 目标                                              | 使用                                                                | 说明                                                               |
| ------------------------------------------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------- |
| ChatGPT/Codex 订阅、原生 Codex 运行时  | `openai/gpt-5.6-sol`                                               | 全新订阅设置；使用 Codex 身份验证登录。                  |
| 智能体轮次的直接 API 密钥计费            | `openai/gpt-5.6` 加上有序的 API 密钥身份验证配置文件              | 全新 API 密钥设置；不带限定的直接 API ID 会解析为 Sol。        |
| 选择确切的 GPT-5.6 层级                      | `openai/gpt-5.6-sol`、`-terra` 或 `-luna`                         | 查看 `models list` 以了解此账户可用的层级。        |
| 无 GPT-5.6 访问权限的账户                    | `openai/gpt-5.5`                                                   | 显式恢复选项；OpenClaw 不会静默降级。     |
| 直接 API 密钥计费、显式 OpenClaw 运行时 | `openai/gpt-5.6` 加上提供商/模型 `agentRuntime.id: "openclaw"` | 选择普通的 `openai` API 密钥配置文件。                           |
| 最新 ChatGPT Instant 模型别名                | `openai/chat-latest`                                               | 仅限直接 API 密钥；这是动态别名，不是稳定默认值。          |
| 图像生成或编辑                       | `openai/gpt-image-2`                                               | 可与 `OPENAI_API_KEY` 或 Codex OAuth 配合使用。                         |
| 透明背景图像                     | `openai/gpt-image-1.5`                                             | 将 `outputFormat` 设为 `png` 或 `webp`，并设置 `background=transparent`。 |

## 名称映射

| 显示的名称                            | 层             | 含义                                                                                  |
| --------------------------------------- | ----------------- | ---------------------------------------------------------------------------------------- |
| `openai`                                | 提供商前缀   | 规范 OpenAI 模型路由；路由信息决定隐式运行时。                |
| `codex` 插件                          | 插件            | 提供原生 Codex app-server 运行时和 `/codex` 聊天控制的内置插件。 |
| 提供商/模型 `agentRuntime.id: codex` | Agent runtime     | 强制匹配的嵌入式轮次使用原生 Codex app-server harness。                   |
| `/codex ...`                            | 聊天命令集  | 从对话中绑定/控制 Codex app-server 线程。                               |
| `runtime: "acp", agentId: "codex"`      | ACP 会话路由 | 通过 ACP/acpx 运行 Codex 的显式后备路径。                                 |

## 隐式智能体运行时

当未设置提供商/模型 `agentRuntime` 策略或策略为 `auto` 时，OpenAI
由提供商所有的路由策略会根据有效
端点和适配器选择隐式运行时：

| 有效路由信息                                                                                                                                                  | 隐式运行时      |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------- |
| 使用 `openai-responses` 的确切官方 Platform HTTPS 端点，或使用 `openai-chatgpt-responses` 的确切官方 ChatGPT HTTPS 端点；没有主动设置的请求覆盖 | 可以选择 Codex |
| 主动设置的 `openai-completions` 适配器                                                                                                                                  | OpenClaw              |
| 自定义端点                                                                                                                                                        | OpenClaw              |
| 显式使用 HTTP 的确切官方端点                                                                                                                            | 拒绝              |
| 带有主动设置的提供商/模型请求覆盖的路由                                                                                                                 | OpenClaw              |

显式的非默认提供商/模型 `agentRuntime.id` 仍具有最高决定权。
例如，`agentRuntime.id: "openclaw"` 会使原本符合 Codex 条件的
路由继续使用 OpenClaw，而 `agentRuntime.id: "codex"` 则要求使用 Codex；当有效路由未声明为兼容 Codex 时，
它会以失败关闭方式处理。
运行时选择不会更改凭据类型或计费方式：Platform API 密钥
身份验证与 ChatGPT/Codex 订阅身份验证仍然彼此独立。

`openclaw doctor --fix` 会将旧版 `codex/*` 和 `openai-codex/*` 模型
引用、旧版 Codex 身份验证配置文件 ID，以及旧版 Codex 身份验证顺序条目迁移到
规范 `openai` 路由。迁移后的模型引用会获得模型范围的
`agentRuntime.id: "codex"`；新的身份验证顺序配置请使用 `auth.order.openai`。

<Note>
仅当未配置主模型时，全新的 OpenAI 设置才会应用 GPT-5.6 主模型。
添加或刷新 OpenAI 身份验证会保留现有的显式
选择（包括 `openai/gpt-5.5`），除非你显式使用
`models auth login --set-default` 或 `models set`。仅当你希望智能体模型
使用 API 密钥身份验证时，才使用 API 密钥身份验证配置文件。
</Note>

## GPT-5.6 限量预览

OpenClaw 可识别确切的 `openai/gpt-5.6-sol`、
`openai/gpt-5.6-terra` 和 `openai/gpt-5.6-luna` 模型 ID。在当前目录中，这三个模型均提供
`xhigh` 和 `max` 推理能力。OpenAI 将 Sol 描述为
旗舰层级，将 Terra 描述为均衡层级，将 Luna 描述为快速、
低成本层级。请参阅
[GPT-5.6 发布公告](https://openai.com/index/previewing-gpt-5-6-sol/)
和[访问指南](https://help.openai.com/en/articles/20001325-a-preview-of-gpt-5-6-sol-terra-and-luna)。

使用直接 OpenAI API 密钥身份验证时，不带限定的 `openai/gpt-5.6` ID 是 Sol 的别名，
也是全新设置的默认值。原生 Codex 目录不会在客户端应用
该直接 API 别名；根据工作空间的访问权限，它可以显示
确切的 Sol、Terra 和 Luna ID。因此，全新的 ChatGPT/Codex OAuth 设置
使用 `openai/gpt-5.6-sol`。使用以下命令检查当前账户：

```bash
openclaw models list --provider openai
```

API 组织与 Codex 工作空间的访问权限可能不同。如果 GPT-5.6
不可用，请显式选择 GPT-5.5：

```bash
openclaw models set openai/gpt-5.5
```

OpenClaw 会显示上游访问错误，不会静默地将
GPT-5.6 选择替换为 GPT-5.5。

<Note>
当未设置运行时策略或策略为 `auto` 时，符合条件的确切官方 HTTPS 路由可以选择内置的 Codex app-server
插件；主动设置的 Completions 路由、
自定义端点和请求传输覆盖仍使用 OpenClaw。明文
官方 HTTP 端点会被拒绝。显式提供商/模型运行时配置仍
具有最高决定权。运行 `openclaw doctor --fix` 可修复过时的旧版 Codex 模型
引用、`codex-cli/*` 引用，或并非由
显式运行时配置设置的旧运行时会话固定项。
</Note>

## OpenClaw 功能覆盖范围

| OpenAI 能力               | OpenClaw 界面                                                                               | 状态                                                            |
| ------------------------- | --------------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| 聊天 / Responses          | `openai/<model>` 模型提供商                                                               | 是                                                              |
| Codex 订阅模型            | `openai/<model>`，使用 OpenAI OAuth                                                        | 是                                                              |
| 旧版 Codex 模型引用       | 旧版 Codex 模型引用，`codex-cli/<model>`                                                     | 由 Doctor 修复为 `openai/<model>`                             |
| Codex app-server harness  | 兼容 Codex 的 HTTPS 路由，运行时未设置或为 `auto`，或者显式设为 `agentRuntime.id: codex` | 是                                                              |
| 服务端 Web 搜索           | 原生 OpenAI Responses 工具                                                                  | 是，前提是已启用 Web 搜索且未固定其他提供商                     |
| 图像                      | `image_generate`                                                                          | 是                                                              |
| 视频                      | `video_generate`                                                                          | 是                                                              |
| 文本转语音                | `tts.provider: "openai"` / `tts`                                                     | 是                                                              |
| 批量语音转文本            | `tools.media.audio` / 媒体理解                                                               | 是                                                              |
| 流式语音转文本            | 语音通话 `streaming.provider: "openai"`                                                                 | 是                                                              |
| 实时语音                  | 语音通话 `realtime.provider: "openai"` / Control UI Talk `talk.realtime.provider: "openai"`                            | 是（OpenAI Platform API key）                                   |
| 嵌入                      | 记忆嵌入提供商                                                                              | 是                                                              |

<Note>
OpenAI 实时语音通过公共 **OpenAI Platform Realtime
API**，并且需要 Platform API key。Codex OAuth 令牌用于对
ChatGPT Codex 后端进行身份验证；它们不能与公共 Realtime 端点所需的 Platform API
key 互换使用。

如果 API key 身份验证报告缺少计费，请在
[platform.openai.com/account/billing](https://platform.openai.com/account/billing)
为实时凭据所属的组织充值 Platform 额度。使用 API key
身份验证时，实时语音接受由
`openclaw onboard --auth-choice openai-api-key` 创建的 `openai` API key 身份验证配置文件、通过
`talk.realtime.providers.openai.apiKey` 为 Control UI Talk 设置的 Platform API key、通过
`plugins.entries.voice-call.config.realtime.providers.openai.apiKey` 为语音通话设置的 Platform API key，或
`OPENAI_API_KEY` 环境变量。

在 Control UI Video Talk 中，OpenAI WebRTC 会按需接收摄像头上下文：
当模型调用 `describe_view` 时，浏览器会通过实时数据通道发送一张大小受限的 JPEG。
OpenClaw 不会将连续摄像头轨道附加到 OpenAI 会话。
</Note>

## 记忆嵌入

OpenClaw 可以使用 OpenAI 或兼容 OpenAI 的嵌入端点，为
`memory_search` 索引和查询生成嵌入：

```json5
{
  memory: {
    search: {
      provider: "openai",
      model: "text-embedding-3-small",
    },
  },
}
```

对于需要非对称嵌入标签的兼容 OpenAI 端点，请在 `memory.search` 下设置
`queryInputType` 和 `documentInputType`。OpenClaw
会将它们作为提供商特定的 `input_type` 请求字段转发：查询嵌入使用
`queryInputType`；已索引的记忆分块和批量索引使用
`documentInputType`。完整示例请参阅
[记忆配置参考](/zh-CN/reference/memory-config#provider-specific-config)。

## 入门指南

<Tabs>
  <Tab title="API key（OpenAI Platform）">
    **最适合：**直接访问 API 和按使用量计费。

    <Steps>
      <Step title="获取 API key">
        从 [OpenAI Platform 控制面板](https://platform.openai.com/api-keys)创建或复制 API key。
      </Step>
      <Step title="运行新手引导">
        ```bash
        openclaw onboard --auth-choice openai-api-key
        ```

        或者直接传入 key：

        ```bash
        openclaw onboard --openai-api-key "$OPENAI_API_KEY"
        ```
      </Step>
      <Step title="验证模型是否可用">
        ```bash
        openclaw models list --provider openai
        ```
      </Step>
    </Steps>

    ### 路由摘要

    | 模型引用         | 运行时策略或路由事实                                          | 路由                      | 身份验证                          |
    | ---------------- | ------------------------------------------------------------- | ------------------------- | --------------------------------- |
    | `openai/gpt-5.6` | 未设置或为 `auto`、精确匹配官方 HTTPS 原生路由、无请求级覆盖 | 可选择 Codex              | 按顺序选择的 API key 身份验证配置文件 |
    | `openai/gpt-5.6` | 提供商/模型 `agentRuntime.id: "openclaw"`                               | OpenClaw 嵌入式运行时     | 选定的 `openai` API key 配置文件 |
    | `openai/gpt-5.5` | 显式提供商/模型 `agentRuntime.id`                           | 选定的 Agent Runtimes     | 选定的 OpenAI API key 配置文件   |
    | `openai/*` | 自行指定的 Completions、自定义端点或请求级覆盖                | OpenClaw 嵌入式运行时     | 凭据类型保持不变                  |
    | `openai/*` | 明文官方 HTTP 端点                                            | 拒绝                      | 不发送凭据                        |

    <Note>
    当运行时未设置或为 `auto` 时，只有符合条件且精确匹配的官方 HTTPS 原生
    路由才能隐式选择 Codex app-server harness。要为智能体模型使用 API key 身份验证，
    请创建 `openai` API key 身份验证配置文件，并使用
    `auth.order.openai` 对其排序；`OPENAI_API_KEY` 仍是非智能体 OpenAI API
    界面的直接回退。运行 `openclaw doctor --fix` 可迁移较旧的
    旧版 Codex 身份验证顺序条目。
    </Note>

    ### 配置示例

    ```json5
    {
      env: { OPENAI_API_KEY: "example-openai-key-not-real" },
      agents: { defaults: { model: { primary: "openai/gpt-5.6" } } },
    }
    ```

    纯直接 API 的 `gpt-5.6` ID 会解析为 Sol 层级。如果此 API
    组织未开放 GPT-5.6，请将主模型显式设置为
    `openai/gpt-5.5`。

    要通过 OpenAI API 试用 ChatGPT 当前的 Instant 模型，请将模型
    设置为 `openai/chat-latest`：

    ```json5
    {
      env: { OPENAI_API_KEY: "example-openai-key-not-real" },
      agents: { defaults: { model: { primary: "openai/chat-latest" } } },
    }
    ```

    `chat-latest` 是一个动态别名。新的 OpenAI API key 设置改用
    `openai/gpt-5.6`，其纯直接 API ID 会解析为 Sol。现有的
    显式主模型（包括 `openai/gpt-5.5`）保持不变。
    `chat-latest` 别名仅接受 `medium` 文本详略程度；对于此模型，
    OpenClaw 会将其他任何请求的详略程度强制设为 `medium`。

    <Warning>
    OpenClaw **不会**在直接 OpenAI
    API key 路由上开放 `gpt-5.3-codex-spark`。只有当已登录账户开放该模型时，
    才能通过 Codex 订阅目录条目使用它。
    </Warning>

  </Tab>

  <Tab title="Codex 订阅">
    **最适合：**使用 ChatGPT/Codex 订阅和原生 Codex
    app-server 执行，而不是单独使用 API key。Codex 云端功能需要
    登录 ChatGPT。

    <Steps>
      <Step title="运行 Codex OAuth">
        ```bash
        openclaw onboard --auth-choice openai
        ```

        或直接运行 OAuth：

        ```bash
        openclaw models auth login --provider openai
        ```

        对于无头环境或无法接收回调的设置，请添加 `--device-code`，改用
        ChatGPT 设备代码流程登录，而不是使用 localhost 浏览器
        回调：

        ```bash
        openclaw models auth login --provider openai --device-code
        ```
      </Step>
      <Step title="使用规范的 OpenAI 模型路由">
        ```bash
        openclaw config set agents.defaults.model.primary openai/gpt-5.6-sol
        ```

        对于此精确匹配的官方 HTTPS 原生路由，无需配置运行时。
        它可以自动选择 Codex app-server 运行时；选择该运行时后，
        OpenClaw 会安装或修复内置 Codex 插件。
      </Step>
      <Step title="验证 Codex 身份验证是否可用">
        ```bash
        openclaw models list --provider openai
        ```

        Gateway 网关运行后，在聊天中发送 `/codex status` 或 `/codex models`
        以验证原生 app-server 运行时。
      </Step>
    </Steps>

    ### 路由摘要

    | 模型引用                 | 运行时策略或路由事实                                          | 路由                                                     | 身份验证                                           |
    | ------------------------ | ------------------------------------------------------------- | -------------------------------------------------------- | -------------------------------------------------- |
    | `openai/gpt-5.6-sol`       | 未设置或为 `auto`、精确匹配官方 HTTPS 原生路由、无请求级覆盖 | 可选择 Codex                                             | Codex 登录，或按顺序选择的 `openai` 身份验证配置文件 |
    | `openai/gpt-5.6-terra`       | 未设置或为 `auto`、精确匹配官方 HTTPS 原生路由、无请求级覆盖 | 可选择 Codex                                             | 当目录开放 Terra 时使用 Codex 登录                 |
    | `openai/gpt-5.6-luna`       | 未设置或为 `auto`、精确匹配官方 HTTPS 原生路由、无请求级覆盖 | 可选择 Codex                                             | 当目录开放 Luna 时使用 Codex 登录                  |
    | `openai/gpt-5.6-sol`       | 提供商/模型 `agentRuntime.id: "openclaw"`                               | OpenClaw 嵌入式运行时、内部 Codex 身份验证传输           | 选定的 `openai` OAuth 配置文件           |
    | `openai/gpt-5.5`       | 显式提供商/模型 `agentRuntime.id`                           | 选定的 Agent Runtimes                                    | 选定的 OpenAI 身份验证配置文件                     |
    | `openai/*`       | 自行指定的 Completions、自定义端点或请求级覆盖                | OpenClaw 嵌入式运行时                                    | 凭据要求仍由具体路由决定                           |
    | `openai/*`       | 明文官方 HTTP 端点                                            | 拒绝                                                     | 不发送凭据                                         |
    | 旧版 Codex GPT-5.5 引用  | 由 Doctor 修复                                                | 重写为 `openai/gpt-5.5`                                | 已迁移的 OpenAI OAuth 配置文件                     |
    | `codex-cli/gpt-5.5`       | 由 Doctor 修复                                                | 重写为 `openai/gpt-5.5`                                | Codex app-server 身份验证                          |

    <Warning>
    全新订阅支持的设置使用精确的 `openai/gpt-5.6-sol`；原生 Codex 目录也可能公开精确的 Terra 或 Luna 引用。如果账户未公开 GPT-5.6，请明确选择 `openai/gpt-5.5`。较旧的 Codex GPT 引用是旧版 OpenClaw 路由，而不是原生 Codex 运行时路径；运行 `openclaw doctor --fix` 可迁移这些引用，同时不会升级现有的显式 GPT-5.5 选择。`gpt-5.3-codex-spark` 仍仅限 Codex 订阅目录中提供该模型的账户使用；其直接 OpenAI API key 和 Azure 引用仍会被隐藏。
    </Warning>

    <Note>
    新配置应将 OpenAI 智能体身份验证顺序放在 `auth.order.openai` 下；Doctor 会迁移较旧的旧版 Codex 身份验证顺序条目。
    </Note>

    ### 配置示例

    ```json5
    {
      plugins: { entries: { codex: { enabled: true } } },
      agents: {
        defaults: {
          model: { primary: "openai/gpt-5.6-sol" },
        },
      },
    }
    ```

    使用 API key 作为备用方式时，请将所选模型保留在 `openai/*` 下，并将身份验证顺序放在 `openai` 下。OpenClaw 会先尝试订阅，再尝试 API key，同时始终使用 Codex harness：

    ```json5
    {
      plugins: { entries: { codex: { enabled: true } } },
      agents: {
        defaults: {
          model: { primary: "openai/gpt-5.6-sol" },
        },
      },
      auth: {
        order: {
          openai: [
            "openai:user@example.com",
            "openai:api-key-backup",
          ],
        },
      },
    }
    ```

    <Note>
    新手引导不再从 `~/.codex` 导入 OAuth 材料。请使用浏览器 OAuth（默认）或上述设备代码流程登录；OpenClaw 会在自己的智能体身份验证存储中管理生成的凭据。
    </Note>

    ### 检查并恢复 Codex OAuth 路由

    ```bash
    openclaw models status
    openclaw models auth list --provider openai
    openclaw config get agents.defaults.model --json
    openclaw config get models.providers.openai.agentRuntime --json
    ```

    对于特定智能体，请添加 `--agent <id>`：

    ```bash
    openclaw models status --agent <id>
    openclaw models auth list --agent <id> --provider openai
    ```

    如果较旧的配置仍包含旧版 Codex GPT 引用，或者存在没有显式运行时配置的过期 OpenAI 运行时会话固定项，请修复它：

    ```bash
    openclaw doctor --fix
    openclaw config validate
    ```

    如果 `models auth list --provider openai` 显示没有可用的配置文件，请重新登录：

    ```bash
    openclaw models auth login --provider openai
    openclaw models status --probe --probe-provider openai
    ```

    对同一智能体中的多个 Codex OAuth 登录使用 `--profile-id`，然后通过身份验证顺序或 `/model ...@<profileId>` 控制它们：

    ```bash
    openclaw models auth login --provider openai --profile-id openai:ritsuko
    openclaw models auth login --provider openai --profile-id openai:lain
    ```

    在依赖配置文件顺序之前，运行 `openclaw doctor --fix` 以迁移较旧的旧版 OpenAI Codex 前缀配置文件 ID 和顺序条目。

    ### 状态指示器

    聊天中的 `/status` 会显示当前会话正在使用哪个模型运行时。当符合条件的隐式路由或显式提供商/模型运行时策略选中内置 Codex app-server harness 时，它会显示为 `Runtime: OpenAI Codex`。

    ### Doctor 警告

    如果配置或会话状态中仍有旧版 Codex 模型引用或过期的 OpenAI 运行时固定项，除非 OpenClaw 已进行显式配置，否则 `openclaw doctor --fix` 会使用 Codex 运行时将它们重写为 `openai/*`。

    ### 上下文窗口默认值和长上下文选择启用

    OpenClaw 将原生模型容量和有效运行时预算视为两个独立的值：

    - `contextWindow` 声明提供商的模型总窗口。
    - `contextTokens` 限制 OpenClaw 将其中多少用于有效输入。

    ChatGPT/Codex OAuth 遵循实时 Codex 账户目录。当前目录通常为 GPT-5.6 提供 `272000` token 的有效窗口。直接使用 API key 的 GPT-5.5 和 GPT-5.6 模型也默认使用 `272000` `contextTokens`，即使 Platform API 提供了更大的原生窗口也是如此。这样可以在各种身份验证模式之间保持一致的常规延迟、质量和成本特征。配置的 `agents.defaults.contextTokens` 值可以进一步降低该预算，但无法将模型提高到其配置的 `contextTokens` 上限之上。

    对于直接使用 API key 的 GPT-5.5 和 GPT-5.6，OpenAI 文档说明提供商窗口为 `1050000` token，最大输出 token 数为 `128000`。预留完整输出额度后，可为输入保留 `922000` token。这是推导出的运行预算，并非提供商单独发布的输入限制。请参阅官方[模型比较](https://developers.openai.com/api/docs/models/compare)和 [GPT-5.5 模型页面](https://developers.openai.com/api/docs/models/gpt-5.5)。以下示例让一个 Terra 模型选择使用该额度，并要求 OpenAI 在有效 token 数达到 `700000` 时进行压缩：

    ```json5
    {
      models: {
        providers: {
          openai: {
            models: [
              {
                id: "gpt-5.6-terra",
                name: "GPT-5.6 Terra",
                contextWindow: 1050000,
                contextTokens: 922000,
                maxTokens: 128000,
              },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: { primary: "openai/gpt-5.6-terra" },
          models: {
            "openai/gpt-5.6-terra": {
              agentRuntime: { id: "openclaw" },
              params: {
                responsesServerCompaction: true,
                responsesCompactThreshold: 700000,
              },
            },
          },
        },
      },
    }
    ```

    此示例特意使用 `agentRuntime.id: "openclaw"`。这证明嵌入式 OpenClaw Responses 路径正在使用上述模型元数据和服务端压缩设置。原生 Codex harness 线程则在 Codex 配置中管理自己的上下文预算；请参阅 [Codex harness 长上下文](/zh-CN/plugins/codex-harness#direct-api-long-context)。

    <Warning>
    当 GPT-5.5 或 GPT-5.6 请求超过 `272000` 个输入 token 时，OpenAI 会采用更高的长上下文定价：整个符合条件的请求按 2× 输入费率和 1.5× 输出费率计费。大型提示词会在多轮对话中重新发送或压缩，因此即使可见回复很短，选择启用的会话成本也可能远高于默认值。请参阅 [OpenAI API 定价](https://developers.openai.com/api/docs/pricing)。账户访问权限、实际限制和计费以 API 为准。
    </Warning>

    ### 目录恢复

    当存在 `gpt-5.5` 的上游 Codex 目录元数据时，OpenClaw 会使用这些元数据。如果账户已通过身份验证，但实时 Codex 发现中缺少 `gpt-5.5` 行，OpenClaw 会合成该 OAuth 模型行，避免定时任务、子智能体和已配置默认模型的运行因 `Unknown model` 而失败。

  </Tab>
</Tabs>

## 原生 Codex app-server 身份验证

当符合条件的精确官方 HTTPS 路由隐式选择原生 Codex app-server harness，或者提供商/模型 `agentRuntime.id: "codex"` 显式选择它时，该 harness 使用 `openai/*` 模型引用。其身份验证仍以账户为基础。OpenClaw 按以下顺序选择身份验证方式：

1. 智能体的有序 OpenAI 身份验证配置文件，最好位于 `auth.order.openai` 下。运行 `openclaw doctor --fix` 可迁移较旧的旧版 Codex 身份验证配置文件 ID 和身份验证顺序。
2. app-server 的现有账户，例如本地 Codex CLI ChatGPT 登录。对于默认的隔离智能体主目录，OpenClaw 会通过登录 RPC 将该原生 CLI 账户桥接到 app-server；它不会共享 CLI 的配置、插件或线程存储。
3. 仅适用于本地 stdio app-server 启动，并且仅在 app-server 报告没有账户时：先使用 `CODEX_API_KEY`，再使用 `OPENAI_API_KEY`。

即使 Gateway 网关进程还为直接 OpenAI 模型或嵌入设置了 `OPENAI_API_KEY`，也不会因此替换本地 ChatGPT/Codex 订阅登录。环境变量 API key 回退仅适用于本地 stdio 无账户路径；它绝不会通过 WebSocket app-server 连接发送。选择订阅类型的 Codex 配置文件时，OpenClaw 还会从生成的 stdio app-server 子进程中排除 `CODEX_API_KEY` 和 `OPENAI_API_KEY`，改为通过 app-server 登录 RPC 发送所选凭据。

当该订阅配置文件受 Codex 使用限制阻止时，OpenClaw 会将此配置文件标记为已阻止，直至 Codex 提供的重置时间，并允许身份验证顺序轮换到下一个 `openai:*` 配置文件，而不会更改所选模型或退出 Codex harness。重置时间过后，该订阅配置文件将再次可用。

## 图像生成

内置 `openai` 插件通过 `image_generate` 工具注册图像生成功能。它通过同一个 `openai/gpt-image-2` 模型引用同时支持 OpenAI API key 和 Codex OAuth 图像生成。

| 能力                      | OpenAI API key                     | Codex OAuth                          |
| ------------------------- | ---------------------------------- | ------------------------------------ |
| 模型引用                  | `openai/gpt-image-2`               | `openai/gpt-image-2`                 |
| 身份验证                  | `OPENAI_API_KEY`                   | OpenAI Codex OAuth 登录              |
| 传输方式                  | OpenAI Images API                  | Codex Responses 后端                 |
| 每次请求的最大图像数      | 4                                  | 4                                    |
| 编辑模式                  | 已启用（最多 5 张参考图像）        | 已启用（最多 5 张参考图像）          |
| 尺寸覆盖                  | 支持，包括 2K/4K 尺寸              | 支持，包括 2K/4K 尺寸                |
| 宽高比/分辨率             | 不转发到 OpenAI Images API         | 在安全时映射到受支持的尺寸           |

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: { primary: "openai/gpt-image-2" },
    },
  },
}
```

<Note>
有关共享工具参数、提供商选择和故障转移行为，请参阅[图像生成](/zh-CN/tools/image-generation)。
</Note>

`gpt-image-2` 是 OpenAI 文生图和图像编辑的默认值。`gpt-image-1.5`、`gpt-image-1` 和 `gpt-image-1-mini` 仍可用作显式模型覆盖。使用 `openai/gpt-image-1.5` 可获得透明背景的 PNG/WebP 输出；当前 `gpt-image-2` API 会拒绝 `background: "transparent"`。

对于透明背景请求，请使用 `model: "openai/gpt-image-1.5"`、`outputFormat: "png"` 或 `"webp"`，以及 `background: "transparent"` 调用 `image_generate`；较旧的 `openai.background` 提供商选项仍可接受。OpenClaw 还会通过将默认的 `openai/gpt-image-2` 透明请求重写为 `gpt-image-1.5`，保护公共 OpenAI 和 OpenAI Codex OAuth 路由；Azure 和自定义 OpenAI 兼容端点会保留其配置的部署/模型名称。

同一设置也适用于无头 CLI 运行：

```bash
openclaw infer image generate \
  --model openai/gpt-image-1.5 \
  --output-format png \
  --background transparent \
  --prompt "透明背景上的简洁红色圆形贴纸" \
  --json
```

从输入文件开始时，请对 `openclaw infer image edit` 使用相同的 `--output-format` 和 `--background` 标志。`--openai-background` 仍可用作 OpenAI 专用别名。使用 `--quality low|medium|high|auto` 控制 OpenAI Images 的质量和成本。使用 `--openai-moderation low|auto` 从 `image generate` 或 `image edit` 传递 OpenAI 的审核提示。

对于 ChatGPT/Codex OAuth 安装，请保持使用同一个 `openai/gpt-image-2` 引用。配置
`openai` OAuth 配置文件后，OpenClaw 会解析其中存储的 OAuth
访问令牌，并通过 Codex Responses 后端发送图像请求；它不会先尝试
`OPENAI_API_KEY`，也不会静默回退到 API key。
如果要改用直接调用 OpenAI Images API 的路径，请通过 API key、自定义基础
URL 或 Azure 端点显式配置 `models.providers.openai`。如果该自定义图像端点位于受信任的局域网/私有地址，
还需设置 `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork: true`；除非明确选择启用此项，否则 OpenClaw
会阻止访问私有/内部的 OpenAI 兼容图像端点。

生成：

```
/tool image_generate model=openai/gpt-image-2 prompt="为 macOS 上的 OpenClaw 制作一张精美的发布海报" size=3840x2160 count=1
```

生成透明 PNG：

```
/tool image_generate model=openai/gpt-image-1.5 prompt="透明背景上的简洁红色圆形贴纸" outputFormat=png background=transparent
```

编辑：

```
/tool image_generate model=openai/gpt-image-2 prompt="保留物体形状，将材质改为半透明玻璃" image=/path/to/reference.png size=1024x1536
```

## 视频生成

内置的 `openai` 插件通过
`video_generate` 工具注册视频生成功能。

| 能力             | 值                                                                                 |
| ---------------- | ---------------------------------------------------------------------------------- |
| 默认模型         | `openai/sora-2`                                                                 |
| 模式             | 文本生成视频、图像生成视频、单视频编辑                                             |
| 参考输入         | 1 张图像或 1 个视频                                                                |
| 尺寸覆盖         | 文本生成视频和图像生成视频均支持                                                   |
| 宽高比           | 转换为最接近的受支持尺寸，而不是原样转发                                           |
| 其他覆盖         | 不支持 `resolution`、`audio`、`watermark`，将予以丢弃并发出工具警告 |

OpenAI 图像生成视频请求使用 `POST /v1/videos`，并提供图像
`input_reference`。单视频编辑使用 `POST /v1/videos/edits`，上传的视频放在
`video` 字段中。

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "openai/sora-2" },
    },
  },
}
```

<Note>
有关共享工具参数、提供商选择和故障转移行为，请参阅[视频生成](/zh-CN/tools/video-generation)。

OpenAI provider 声明了 `supportsSize`，但未声明 `supportsAspectRatio` 或
`supportsResolution`。OpenClaw 的共享规范化层会在请求到达提供商之前，将所请求的
`aspectRatio` 转换为最接近的 OpenAI `size`，因此宽高比请求通常仍然有效。
`resolution` 没有尺寸回退，将被丢弃，并以
`Ignored unsupported overrides for openai/<model>: resolution=<value>` 的形式呈现给调用方。
</Note>

## GPT-5 提示词贡献

OpenClaw 会为 `openai` 提供商上的 GPT-5 系列模型添加共享的 GPT-5
提示词贡献（包括规范化为 `openai/*` 的修复前旧版 Codex 引用）。
其他同样提供 GPT-5 系列模型 ID 的提供商（例如 OpenRouter 或 opencode 路由）
不会收到此覆盖；它根据提供商 ID `openai` 启用，而不只根据模型 ID。
较旧的 GPT-4.x 模型绝不会收到此覆盖。

原生 Codex app-server harness 不会通过开发者指令接收角色设定/工具纪律行为契约
或友好交互风格覆盖；原生 Codex 保留由 Codex 所有的基础行为、模型行为和项目文档行为，
而且 OpenClaw 会为原生线程禁用 Codex 的内置个性，使 Agent 工作区中的个性文件保持权威。
OpenClaw 只向原生 Codex 线程贡献运行时上下文：渠道投递、OpenClaw 动态工具、
ACP 委派、工作区上下文和 OpenClaw Skills。来自同一贡献的 Heartbeat 指导文本是唯一的例外：
原生 Codex Heartbeat 轮次确实会收到该文本，它会作为专用协作指令注入，而不是通过共享的
提示词贡献钩子注入。

对于匹配的、由 OpenClaw 组装的提示词，GPT-5 贡献会添加带标签的行为契约，
涵盖角色设定持久性、执行安全、工具纪律、输出形式、完成情况检查和验证。
特定于渠道的回复与静默消息行为仍保留在共享 OpenClaw 系统提示词和出站投递策略中。
友好交互风格层相互独立且可配置。

| 值                     | 效果                     |
| ---------------------- | ------------------------ |
| `"friendly"`（默认） | 启用友好交互风格层       |
| `"on"`     | `"friendly"` 的别名 |
| `"off"`     | 仅禁用友好风格层         |

<Tabs>
  <Tab title="配置">
    ```json5
    {
      agents: {
        defaults: {
          promptOverlays: {
            gpt5: { personality: "friendly" },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="CLI">
    ```bash
    openclaw config set agents.defaults.promptOverlays.gpt5.personality off
    ```
  </Tab>
</Tabs>

<Tip>
运行时值不区分大小写，因此 `"Off"` 和 `"off"` 都会禁用
友好风格层。
</Tip>

<Note>
当共享的 `agents.defaults.promptOverlays.gpt5.personality` 设置未配置时，仍会读取旧版
`plugins.entries.openai.config.personality` 作为兼容性回退。
</Note>

## 语音与语音处理

<AccordionGroup>
  <Accordion title="语音合成（TTS）">
    内置的 `openai` 插件为
    `tts` 接口注册语音合成功能。

    | 设置         | 配置路径                                                  | 默认值                              |
    | ------------ | --------------------------------------------------------- | ----------------------------------- |
    | 模型         | `tts.providers.openai.model`                                        | `gpt-4o-mini-tts`                  |
    | 语音         | `tts.providers.openai.speakerVoice`                                        | `coral`                  |
    | 速度         | `tts.providers.openai.speed`                                        | （未设置）                          |
    | 指令         | `tts.providers.openai.instructions`                                        | （未设置，仅限 `gpt-4o-mini-tts`） |
    | 格式         | `tts.providers.openai.responseFormat`                                        | 语音消息使用 `opus`，文件使用 `mp3` |
    | API key      | `tts.providers.openai.apiKey`                                        | 回退到 `OPENAI_API_KEY`           |
    | 基础 URL     | `tts.providers.openai.baseUrl`                                        | `https://api.openai.com/v1`                  |
    | 附加请求体   | `tts.providers.openai.extraBody` / `extra_body`                   | （未设置）                          |

    可用模型：`gpt-4o-mini-tts`、`tts-1`、`tts-1-hd`。可用语音：
    `alloy`、`ash`、`ballad`、`cedar`、`coral`、`echo`、`fable`、`juniper`、
    `marin`、`onyx`、`nova`、`sage`、`shimmer`、`verse`。

    `extraBody` 会在 OpenClaw 生成的字段之后合并到 `/audio/speech`
    请求 JSON 中，因此可将其用于需要额外键（例如 `lang`）的
    OpenAI 兼容端点。原型键将被忽略。

    ```json5
    {
      tts: {
        providers: {
          openai: { model: "gpt-4o-mini-tts", speakerVoice: "coral" },
        },
      },
    }
    ```

    <Note>
    设置 `OPENAI_TTS_BASE_URL` 可覆盖 TTS 基础 URL，而不影响聊天 API 端点。
    OpenAI TTS 和实时语音都通过 OpenAI Platform API key 配置；仅使用 OAuth 的安装
    仍可使用 Codex 支持的聊天模型，但无法使用 OpenAI 实时语音回传。
    </Note>

  </Accordion>

  <Accordion title="语音转文本">
    内置的 `openai` 插件通过 OpenClaw 的媒体理解转录接口注册
    批量语音转文本功能。

    - 默认模型：`gpt-4o-transcribe`
    - 端点：OpenAI REST `/v1/audio/transcriptions`
    - 输入路径：multipart 音频文件上传
    - 用于所有读取 `tools.media.audio` 的入站音频转录位置，
      包括 Discord 语音频道片段和渠道音频附件

    要强制使用 OpenAI 进行入站音频转录：

    ```json5
    {
      tools: {
        media: {
          audio: {
            models: [
              {
                type: "provider",
                provider: "openai",
                model: "gpt-4o-transcribe",
              },
            ],
          },
        },
      },
    }
    ```

    如果共享音频媒体配置或每次调用的转录请求提供了语言和提示词提示，
    则会将它们转发给 OpenAI。

  </Accordion>

  <Accordion title="实时转录">
    内置的 `openai` 插件为语音通话插件注册实时转录功能。

    | 设置             | 配置路径                                                          | 默认值 |
    | ---------------- | ----------------------------------------------------------------- | ------ |
    | 模型             | `plugins.entries.voice-call.config.streaming.providers.openai.model`                                                | `gpt-4o-transcribe` |
    | 语言             | `...openai.language`                                                | （未设置） |
    | 提示词           | `...openai.prompt`                                                | （未设置） |
    | 静音时长         | `...openai.silenceDurationMs`                                                | `800` |
    | VAD 阈值         | `...openai.vadThreshold`                                                | `0.5` |
    | 身份验证         | `...openai.apiKey`、`OPENAI_API_KEY` 或 `openai` API-key 配置文件 | 需要 Platform API key |

    <Note>
    使用 WebSocket 连接到 `wss://api.openai.com/v1/realtime`，音频采用 G.711 u-law
    （`g711_ulaw` / `audio/pcmu`）。对于 `openai`
    API-key 配置文件，Gateway 网关会在打开 WebSocket 前签发一个临时的实时转录客户端密钥。
    此流式提供商用于语音通话插件的实时转录路径；Discord 语音目前会录制短片段，
    并改用批量 `tools.media.audio` 转录路径。
    </Note>

  </Accordion>

  <Accordion title="实时语音">
    内置的 `openai` 插件为语音通话插件注册实时语音功能。

    | 设置                               | 配置路径                                                              | 默认值             |
    | --------------------------------------- | ---------------------------------------------------------------------------- | ---------------------- |
    | 模型                                  | `plugins.entries.voice-call.config.realtime.providers.openai.model`     | `gpt-realtime-2.1`  |
    | 语音                                  | `...openai.voice`                                                       | `alloy`             |
    | 温度（Azure 部署桥接）  | `...openai.temperature`                                                 | `0.8`               |
    | VAD 阈值                          | `...openai.vadThreshold`                                                | `0.5`                |
    | 静音持续时间                       | `...openai.silenceDurationMs`                                           | `500`                |
    | 前缀填充                         | `...openai.prefixPaddingMs`                                             | `300`                |
    | 推理强度                       | `...openai.reasoningEffort`                                             | （未设置）              |
    | 身份验证                                   | `openai` API 密钥配置文件、`...openai.apiKey` 或 `OPENAI_API_KEY` | 需要 OpenAI Platform API 密钥 |

    `gpt-realtime-2.1` 可用的内置 Realtime 语音：`alloy`、`ash`、
    `ballad`、`coral`、`echo`、`sage`、`shimmer`、`verse`、`marin`、`cedar`。
    OpenAI 推荐使用 `marin` 和 `cedar`，以获得最佳 Realtime 质量。这
    与上面的文本转语音语音是不同的集合；仅限 TTS 的语音
    （例如 `fable`、`nova` 或 `onyx`）不能用于 Realtime 会话。
    如果更倾向于使用
    更小、成本更低的 Realtime 2.1 变体，请将模型显式设置为 `gpt-realtime-2.1-mini`。

    <Note>
    **GPT-Live（即将推出）。** OpenAI 的全双工 `gpt-live-1` 和
    `gpt-live-1-mini` 模型已于 2026 年 7 月取代 ChatGPT 语音模式；
    开发者 API 正在向抢先体验组织逐步推出。OpenClaw
    能识别该模型系列，但尚不能运行它：GPT-Live 会话
    仅支持 WebRTC、自行管理轮次交接（无 VAD），并通过 OpenClaw 的 Realtime 传输
    尚未实现的交接事件协议委派
    智能体工作。配置 `gpt-live-*` 模型时会以失败关闭，并
    提供有关 WebSocket 桥接和 Talk 浏览器会话的指导，而不是
    在智能体无法访问的情况下静默连接音频。抢先体验期间，API 访问权限也
    按 OpenAI 组织设置门槛。在 GPT-Live 支持上线前，请继续使用 `gpt-realtime-2.1`（
    默认值）。
    </Note>

    <Note>
    后端 OpenAI Realtime 桥接使用正式发布的 Realtime WebSocket 会话
    结构，该结构不接受 `session.temperature`。Azure OpenAI
    部署仍可通过 `azureEndpoint` 和 `azureDeployment` 使用，并
    保留与部署兼容的会话结构（包括 `temperature`）。
    支持双向工具调用和 G.711 μ-law 音频。
    </Note>

    <Note>
    Realtime 语音在创建会话时选定。OpenAI 允许稍后更改大多数
    会话字段，但模型在该会话中发出音频后，便无法更改
    语音。OpenClaw 目前以字符串形式公开
    内置 Realtime 语音 ID。
    </Note>

    <Note>
    Control UI Talk 使用 OpenAI 浏览器 Realtime 会话，其中包含由 Gateway 网关
    签发的临时客户端密钥，并由浏览器直接通过 WebRTC SDP 与
    OpenAI Realtime API 交换。Gateway 网关使用
    选定的 `openai` 凭据签发该客户端密钥。已配置的密钥、API 密钥配置文件和
    `OPENAI_API_KEY` 优先；`openai` OAuth 配置文件或外部
    Codex 登录作为后备方案。Gateway 网关中继和语音通话后端 Realtime
    WebSocket 桥接对原生 OpenAI 端点使用相同的凭据顺序。
    维护者可通过
    `OPENAI_API_KEY=... GEMINI_API_KEY=... node --import tsx scripts/dev/realtime-talk-live-smoke.ts`
    进行实时验证；OpenAI 环节会同时验证后端 WebSocket 桥接和浏览器
    WebRTC SDP 交换，且不会记录密钥。
    传入 `--openai-only` 可在没有 Google 凭据的情况下运行这两个环节。
    </Note>

  </Accordion>
</AccordionGroup>

## Azure OpenAI 端点

内置的 `openai` 提供商可以通过覆盖基础 URL，将 Azure OpenAI 资源用于图像
生成。在图像生成路径上，OpenClaw
会检测 `models.providers.openai.baseUrl` 上的 Azure 主机名，并自动切换到
Azure 的请求结构。

<Note>
Realtime 语音使用单独的配置路径
（`plugins.entries.voice-call.config.realtime.providers.openai.azureEndpoint`），
不受 `models.providers.openai.baseUrl` 影响。有关其 Azure 设置，请参阅[语音和语音功能](#voice-and-speech)下的 **Realtime
语音**折叠面板。
</Note>

以下情况适合使用 Azure OpenAI：

- 你已有 Azure OpenAI 订阅、配额或企业
  协议
- 你需要 Azure 提供的区域数据驻留或合规控制
- 你希望将流量保留在现有的 Azure 租户内

### 配置

要通过内置的 `openai` 提供商使用 Azure 生成图像，请将
`models.providers.openai.baseUrl` 指向你的 Azure 资源，并将 `apiKey` 设置为
Azure OpenAI 密钥（而非 OpenAI Platform 密钥）：

```json5
{
  models: {
    providers: {
      openai: {
        baseUrl: "https://<your-resource>.openai.azure.com",
        apiKey: "<azure-openai-api-key>",
      },
    },
  },
}
```

OpenClaw 可识别以下 Azure 主机后缀，并将其用于 Azure 图像生成
路由：

- `*.openai.azure.com`
- `*.services.ai.azure.com`
- `*.cognitiveservices.azure.com`

对于已识别 Azure 主机上的图像生成请求，OpenClaw：

- 发送 `api-key` 标头，而不是 `Authorization: Bearer`
- 使用部署范围路径（`/openai/deployments/{deployment}/...`）
- 将 `?api-version=...` 附加到每个请求
- 对 Azure 图像生成调用使用默认的 600 秒请求超时。
  每次调用的 `timeoutMs` 值仍会覆盖此默认值。

其他基础 URL（公共 OpenAI、OpenAI 兼容代理）继续使用标准的
OpenAI 图像请求结构。

<Note>
`openai` 提供商的图像生成路径要使用 Azure 路由，需要
OpenClaw 2026.4.22 或更高版本。更早的版本会将任何自定义
`openai.baseUrl` 视为公共 OpenAI 端点，因而无法用于 Azure 图像
部署。
</Note>

### API 版本

设置 `AZURE_OPENAI_API_VERSION`，为 Azure 图像生成路径固定特定的 Azure 预览版或正式发布
版本：

```bash
export AZURE_OPENAI_API_VERSION="2024-12-01-preview"
```

未设置该变量时，默认值为 `2024-12-01-preview`。

### 模型名称就是部署名称

Azure OpenAI 将模型绑定到部署。对于通过内置 `openai` 提供商
路由的 Azure 图像生成请求，OpenClaw 中的 `model` 字段
必须是你在 Azure 门户中配置的 **Azure 部署名称**，而不是
公共 OpenAI 模型 ID。

如果你创建了一个名为 `gpt-image-2-prod`、用于提供 `gpt-image-2` 的部署：

```
/tool image_generate model=openai/gpt-image-2-prod prompt="简洁的海报" size=1024x1024 count=1
```

相同的部署名称规则适用于通过内置 `openai` 提供商
路由的任何图像生成调用。

### 区域可用性

Azure 图像生成功能目前仅在部分区域可用
（例如 `eastus2`、`swedencentral`、`polandcentral`、`westus3`、
`uaenorth`）。创建部署前，请查看 Microsoft 当前的区域列表，
并确认你的区域提供所需的具体模型。

### 参数差异

Azure OpenAI 和公共 OpenAI 接受的图像参数并不总是相同。
Azure 可能会拒绝公共 OpenAI 允许的选项（例如 `gpt-image-2` 上的某些
`background` 值），或仅在特定模型
版本上提供这些选项。这些差异源于 Azure 和底层模型，而非
OpenClaw。如果 Azure 请求因验证错误而失败，请在
Azure 门户中查看你的特定部署和 API 版本所支持的
参数集。

<Note>
Azure OpenAI 使用原生传输和兼容行为，但不会接收
OpenClaw 的隐藏归因标头——请参阅[高级配置](#advanced-configuration)下的 **原生路由与 OpenAI 兼容
路由**折叠面板。

对于 Azure 上的聊天或 Responses 流量（图像生成除外），请使用
新手引导流程或专用 Azure 提供商配置；仅设置 `openai.baseUrl`
不会采用 Azure API/身份验证结构。另有一个
`azure-openai-responses/*` 提供商；请参阅下方的服务端压缩
折叠面板。
</Note>

## 高级配置

下面的各模型 `params` 示例会调整 OpenClaw 的嵌入式提供商
请求。配置这些参数属于明确编写的请求行为，因此原本符合条件的
`auto` 路由会继续使用 OpenClaw，而不会隐式选择 Codex。原生
Codex 应用服务器 harness 拥有自己的传输和请求设置；当有效路由未声明
与 Codex 兼容时，显式设置 `agentRuntime.id: "codex"` 会以失败关闭。

<AccordionGroup>
  <Accordion title="传输（WebSocket 与 SSE）">
    OpenClaw 对 `openai/*` 优先使用 WebSocket，并以 SSE 作为后备（`"auto"`）。

    在 `"auto"` 模式下，OpenClaw：
    - 在回退到 SSE 前重试一次早期 WebSocket 故障
    - 故障后将 WebSocket 标记为降级 60 秒，并在
      冷却期间使用 SSE
    - 附加稳定的会话和轮次身份标头，用于重试和
      重新连接
    - 在不同传输变体间规范化用量计数器（`input_tokens` / `prompt_tokens`）

    | 值                | 行为                          |
    | ---------------------- | ------------------------------------ |
    | `"auto"`（默认）   | 优先使用 WebSocket，以 SSE 作为后备     |
    | `"sse"`              | 仅强制使用 SSE                    |
    | `"websocket"`        | 仅强制使用 WebSocket              |

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": {
              params: { transport: "auto" },
            },
          },
        },
      },
    }
    ```

    相关 OpenAI 文档：
    - [通过 WebSocket 使用 Realtime API](https://platform.openai.com/docs/guides/realtime-websocket)
    - [流式 API 响应（SSE）](https://platform.openai.com/docs/guides/streaming-responses)

  </Accordion>

  <Accordion title="快速模式">
    OpenClaw 为 `openai/*` 提供共享的快速模式开关：

    - **聊天/UI：** `/fast status|auto|on|off`
    - **配置：** `agents.defaults.models["<provider>/<model>"].params.fastMode`

    启用后，OpenClaw 会将快速模式映射到 OpenAI 优先处理
    （`service_tier = "priority"`）。现有的 `service_tier` 值会
    保留，快速模式不会重写 `reasoning` 或
    `text.verbosity`。`fastMode: "auto"` 会让新的模型调用以快速模式启动，直到达到
    自动截止时间；此后启动的重试、后备、工具结果或
    继续调用将不使用快速模式。截止时间默认为 60 秒；
    设置活跃模型上的 `params.fastAutoOnSeconds` 可更改该时间。

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": { params: { fastMode: "auto", fastAutoOnSeconds: 30 } },
          },
        },
      },
    }
    ```

    <Note>
    会话覆盖值优先于配置。在会话 UI 中清除会话覆盖值后，
    会话将恢复为已配置的默认值。
    </Note>

  </Accordion>

  <Accordion title="优先处理（service_tier）">
    OpenAI 的 API 通过 `service_tier` 提供优先处理。在 OpenClaw 中为每个
    模型进行设置：

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": { params: { serviceTier: "priority" } },
          },
        },
      },
    }
    ```

    支持的值：`auto`、`default`、`flex`、`priority`。

    <Warning>
    `serviceTier` 仅转发到原生 OpenAI 端点
    （`api.openai.com`）和原生 Codex 端点（`chatgpt.com/backend-api`）。
    如果通过代理路由任一提供商，OpenClaw 将保持
    `service_tier` 不变。
    </Warning>

  </Accordion>

  <Accordion title="服务器端压缩（Responses API）">
    对于直接使用 OpenAI Responses 的模型（`api.openai.com` 上的 `openai/*`），
    OpenAI 插件的 OpenClaw 流包装器会自动启用服务器端
    压缩：

    - 强制启用 `store: true`（除非模型兼容性设置了 `supportsStore: false`）
    - 注入 `context_management: [{ type: "compaction", compact_threshold: ... }]`
    - 默认 `compact_threshold`：`contextWindow` 的 70%（不可用时则使用
      `80000`）

    这适用于 OpenClaw 内置运行时路径，以及嵌入式运行所使用的 OpenAI provider
    钩子。原生 Codex app-server harness 通过 Codex 管理自己的上下文，
    不受此设置影响。

    <Tabs>
      <Tab title="显式启用">
        适用于 Azure OpenAI Responses 等兼容端点：

        ```json5
        {
          agents: {
            defaults: {
              models: {
                "azure-openai-responses/gpt-5.5": {
                  params: { responsesServerCompaction: true },
                },
              },
            },
          },
        }
        ```
      </Tab>
      <Tab title="自定义阈值">
        ```json5
        {
          agents: {
            defaults: {
              models: {
                "openai/gpt-5.5": {
                  params: {
                    responsesServerCompaction: true,
                    responsesCompactThreshold: 120000,
                  },
                },
              },
            },
          },
        }
        ```
      </Tab>
      <Tab title="禁用">
        ```json5
        {
          agents: {
            defaults: {
              models: {
                "openai/gpt-5.5": {
                  params: { responsesServerCompaction: false },
                },
              },
            },
          },
        }
        ```
      </Tab>
    </Tabs>

    <Note>
    `responsesServerCompaction` 仅控制 `context_management` 的注入。
    直接使用 OpenAI Responses 的模型仍会强制启用 `store: true`，除非兼容性设置了
    `supportsStore: false`。
    </Note>

  </Accordion>

  <Accordion title="严格智能体式 GPT 模式">
    对于通过 OpenClaw 嵌入式运行时运行的 `openai` 提供商 GPT-5 系列模型，
    OpenClaw 已默认采用一种名为
    `strict-agentic` 的更严格执行契约。只要解析后的提供商为
    `openai` 且模型 ID 与 GPT-5 系列匹配，它就会自动激活，除非配置
    显式选择退出：

    ```json5
    {
      agents: {
        defaults: {
          embeddedAgent: { executionContract: "default" },
        },
      },
    }
    ```

    显式设置 `"strict-agentic"` 在受支持的路径中不会产生任何效果（它
    已经是默认值），而在不受支持的提供商/模型组合中也不起作用。

    启用 `strict-agentic` 后，OpenClaw：
    - 针对实质性工作自动启用 `update_plan`
    - 遇到结构为空或仅含推理的轮次时，通过续写可见答案进行重试
    - 当所选 harness 提供显式计划事件时使用这些事件

    OpenClaw 不会通过对助手文本进行分类来判断某个轮次是
    计划、进度更新还是最终答案。

    <Note>
    此契约完全位于 OpenClaw 的嵌入式智能体运行器中。它不
    适用于原生 Codex app-server harness；后者会管理自己的
    轮次和计划行为。对于原生 Codex 运行而言，harness 的选择比
    执行契约设置更重要。
    </Note>

  </Accordion>

  <Accordion title="原生路由与 OpenAI 兼容路由">
    OpenClaw 对直接 OpenAI、Codex 和 Azure OpenAI 端点的处理
    不同于通用 OpenAI 兼容的 `/v1` 代理：

    **原生路由**（`openai/*`、Azure OpenAI）：
    - 仅为支持 OpenAI `none` 工作量设置的模型保留 `reasoning: { effort: "none" }`
    - 对于拒绝 `reasoning.effort: "none"` 的模型或代理，省略已禁用的推理设置
    - 工具 schema 默认使用严格模式
    - 仅在经过验证的原生主机上附加隐藏的归属标头（Azure
      OpenAI 不会获得这些标头，即使它属于原生路由）
    - 保留仅适用于 OpenAI 的请求调整（`service_tier`、`store`、
      推理兼容性、提示缓存提示）

    **代理/兼容路由：**
    - 使用更宽松的兼容行为
    - 从非原生 `openai-completions` 载荷中移除 Completions `store`
    - 接受面向 OpenAI 兼容 Completions 代理的高级 `params.extra_body`/`params.extraBody` 透传 JSON
    - 接受面向 vLLM 等 OpenAI 兼容 Completions
      代理的 `params.chat_template_kwargs`
    - 不强制使用严格工具 schema 或仅限原生路由的标头

  </Accordion>
</AccordionGroup>

## 相关内容

<CardGroup cols={2}>
  <Card title="模型选择" href="/zh-CN/concepts/model-providers" icon="layers">
    选择提供商、模型引用和故障转移行为。
  </Card>
  <Card title="图像生成" href="/zh-CN/tools/image-generation" icon="image">
    共用的图像工具参数和提供商选择。
  </Card>
  <Card title="视频生成" href="/zh-CN/tools/video-generation" icon="video">
    共用的视频工具参数和提供商选择。
  </Card>
  <Card title="OAuth 和身份验证" href="/zh-CN/gateway/authentication" icon="key">
    身份验证详情和凭据复用规则。
  </Card>
</CardGroup>
