---
read_when:
    - 你想在 OpenClaw 中使用 Amazon Bedrock 模型
    - 你需要设置 AWS 凭证和区域才能调用模型
summary: 通过 OpenClaw 使用 Amazon Bedrock（Converse API）模型
title: Amazon Bedrock
x-i18n:
    generated_at: "2026-07-26T06:19:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9cbc9534c0d06e0d5642b8d167c633c16880908812b97adbbf9c6bd6c5511603
    source_path: providers/bedrock.md
    workflow: 16
---

OpenClaw 可通过其 **Bedrock Converse** 流式提供商使用 **Amazon Bedrock** 模型。Bedrock 身份验证使用 **AWS SDK 默认凭证链**，而非 API key。

| 属性 | 值                                                          |
| -------- | ----------------------------------------------------------- |
| 提供商 | `amazon-bedrock`                                            |
| API      | `bedrock-converse-stream`                                   |
| 身份验证 | AWS 凭证（环境变量、共享配置或实例角色） |
| 区域   | `AWS_REGION` 或 `AWS_DEFAULT_REGION`（默认值：`us-east-1`） |

## 入门指南

选择首选的身份验证方法，然后按照设置步骤操作。

<Tabs>
  <Tab title="访问密钥 / 环境变量">
    **最适合：** 开发人员计算机、CI，或由你直接管理 AWS 凭证的主机。

    <Steps>
      <Step title="在 Gateway 网关主机上设置 AWS 凭证">
        ```bash
        export AWS_ACCESS_KEY_ID="EXAMPLE_AWS_ACCESS_KEY_ID"
        export AWS_SECRET_ACCESS_KEY="..."
        export AWS_REGION="us-east-1"
        # 可选：
        export AWS_SESSION_TOKEN="..."
        export AWS_PROFILE="your-profile"
        # 可选（Bedrock API key/持有者令牌）：
        export AWS_BEARER_TOKEN_BEDROCK="..."
        ```
      </Step>
      <Step title="将 Bedrock 提供商和模型添加到配置中">
        不需要 `apiKey`。使用 `auth: "aws-sdk"` 配置提供商：

        ```json5
        {
          models: {
            providers: {
              "amazon-bedrock": {
                baseUrl: "https://bedrock-runtime.us-east-1.amazonaws.com",
                api: "bedrock-converse-stream",
                auth: "aws-sdk",
                models: [
                  {
                    id: "us.anthropic.claude-opus-4-6-v1",
                    name: "Claude Opus 4.6 (Bedrock)",
                    reasoning: true,
                    input: ["text", "image"],
                    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                    contextWindow: 200000,
                    maxTokens: 8192,
                  },
                ],
              },
            },
          },
          agents: {
            defaults: {
              model: { primary: "amazon-bedrock/us.anthropic.claude-opus-4-6-v1" },
            },
          },
        }
        ```
      </Step>
      <Step title="验证模型是否可用">
        ```bash
        openclaw models list
        ```
      </Step>
    </Steps>

    <Tip>
    使用环境标记身份验证（`AWS_ACCESS_KEY_ID`、`AWS_PROFILE` 或 `AWS_BEARER_TOKEN_BEDROCK`）时，OpenClaw 会自动启用隐式 Bedrock 提供商以发现模型，无需额外配置。
    </Tip>

  </Tab>

  <Tab title="EC2 实例角色（IMDS）">
    **最适合：** 附加了 IAM 角色，并使用实例元数据服务进行身份验证的 EC2 实例。

    <Steps>
      <Step title="显式启用发现">
        使用 IMDS 时，OpenClaw 无法仅通过环境标记检测 AWS 身份验证，因此必须主动启用：

        ```bash
        openclaw config set plugins.entries.amazon-bedrock.config.discovery.enabled true
        openclaw config set plugins.entries.amazon-bedrock.config.discovery.region us-east-1
        ```
      </Step>
      <Step title="可选：为自动模式添加环境标记">
        如果还希望环境标记自动检测路径生效（例如用于 `openclaw status` 界面）：

        ```bash
        export AWS_PROFILE=default
        export AWS_REGION=us-east-1
        ```

        **不**需要虚假的 API key。
      </Step>
      <Step title="验证是否发现模型">
        ```bash
        openclaw models list
        ```
      </Step>
    </Steps>

    <Warning>
    附加到 EC2 实例的 IAM 角色必须具有以下权限：

    - `bedrock:InvokeModel`
    - `bedrock:InvokeModelWithResponseStream`
    - `bedrock:ListFoundationModels`（用于自动发现）
    - `bedrock:ListInferenceProfiles`（用于推理配置文件发现）

    或附加托管策略 `AmazonBedrockFullAccess`。
    </Warning>

    <Note>
    只有在明确需要用于自动模式或状态界面的环境标记时，才需要 `AWS_PROFILE=default`。实际的 Bedrock 运行时身份验证路径使用 AWS SDK 默认链，因此即使没有环境标记，IMDS 实例角色身份验证也能正常工作。
    </Note>

  </Tab>
</Tabs>

## 自动模型发现

OpenClaw 可以自动发现支持**流式传输**和**文本输出**的 Bedrock 模型。发现过程使用 `bedrock:ListFoundationModels` 和
`bedrock:ListInferenceProfiles`，结果会被缓存（默认值：1 小时）。

隐式提供商的启用方式：

- 如果 `plugins.entries.amazon-bedrock.config.discovery.enabled` 为 `true`，
  即使不存在 AWS 环境标记，OpenClaw 也会尝试执行发现。
- 如果未设置 `plugins.entries.amazon-bedrock.config.discovery.enabled`，
  OpenClaw 仅在检测到以下任一 AWS 身份验证标记时自动添加
  隐式 Bedrock 提供商：
  `AWS_BEARER_TOKEN_BEDROCK`、`AWS_ACCESS_KEY_ID` +
  `AWS_SECRET_ACCESS_KEY`，或 `AWS_PROFILE`。
- 实际的 Bedrock 运行时身份验证路径仍使用 AWS SDK 默认链，因此即使发现过程需要通过 `enabled: true` 主动启用，共享配置、SSO 和 IMDS 实例角色身份验证仍可正常工作。

<Note>
对于显式的 `models.providers["amazon-bedrock"]` 条目，OpenClaw 仍可提前通过 `AWS_BEARER_TOKEN_BEDROCK` 等 AWS 环境标记解析 Bedrock 环境标记身份验证，而无需强制加载完整的运行时身份验证。实际的模型调用身份验证路径仍使用 AWS SDK 默认链。
</Note>

<AccordionGroup>
  <Accordion title="发现配置选项">
    配置选项位于 `plugins.entries.amazon-bedrock.config.discovery` 下：

    ```json5
    {
      plugins: {
        entries: {
          "amazon-bedrock": {
            config: {
              discovery: {
                enabled: true,
                region: "us-east-1",
                providerFilter: ["anthropic", "amazon"],
                refreshInterval: 3600,
                defaultContextWindow: 32000,
                defaultMaxTokens: 4096,
              },
            },
          },
        },
      },
    }
    ```

    | 选项 | 默认值 | 描述 |
    | ------ | ------- | ----------- |
    | `enabled` | 自动 | 在自动模式下，OpenClaw 仅在检测到受支持的 AWS 环境标记时启用隐式 Bedrock 提供商。设置 `true` 可强制执行发现。 |
    | `region` | `AWS_REGION` / `AWS_DEFAULT_REGION` / `us-east-1` | 用于发现 API 调用的 AWS 区域。 |
    | `providerFilter` |（全部）| 匹配 Bedrock 提供商名称（例如 `anthropic`、`amazon`）。 |
    | `refreshInterval` | `3600` | 缓存持续时间，以秒为单位。设置为 `0` 可禁用缓存。 |
    | `defaultContextWindow` | `32000` | 用于没有已知令牌限制的已发现模型的上下文窗口（如果知道模型限制，请覆盖此值）。 |
    | `defaultMaxTokens` | `4096` | 用于没有已知令牌限制的已发现模型的最大输出令牌数（如果知道模型限制，请覆盖此值）。 |

  </Accordion>

  <Accordion title="上下文窗口和最大令牌限制">
    Bedrock 的 `ListFoundationModels` 和 `GetFoundationModel` API 不返回
    令牌限制元数据，只返回模型 ID、名称、模态和生命周期
    状态。OpenClaw 内置了一张查询表，其中包含常用 Bedrock 模型
    （Claude、Nova、Llama、Mistral、DeepSeek 等）的已知上下文窗口和输出
    限制，以便这些模型的会话管理、压缩阈值和
    上下文溢出检测能够正常工作。

    表中未包含的已发现模型会回退到 `defaultContextWindow`
    和 `defaultMaxTokens`。如果你使用的模型缺少准确的限制，
    请使用显式的
    `models.providers["amazon-bedrock"].models` 条目进行覆盖。

  </Accordion>
</AccordionGroup>

## 快速设置（AWS 路径）

本演练将创建 IAM 角色、附加 Bedrock 权限、关联
实例配置文件，并在 EC2 主机上启用 OpenClaw 发现。

```bash
# 1. 创建 IAM 角色和实例配置文件
aws iam create-role --role-name EC2-Bedrock-Access \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy --role-name EC2-Bedrock-Access \
  --policy-arn arn:aws:iam::aws:policy/AmazonBedrockFullAccess

aws iam create-instance-profile --instance-profile-name EC2-Bedrock-Access
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-Bedrock-Access \
  --role-name EC2-Bedrock-Access

# 2. 附加到 EC2 实例
aws ec2 associate-iam-instance-profile \
  --instance-id i-xxxxx \
  --iam-instance-profile Name=EC2-Bedrock-Access

# 3. 在 EC2 实例上显式启用发现
openclaw config set plugins.entries.amazon-bedrock.config.discovery.enabled true
openclaw config set plugins.entries.amazon-bedrock.config.discovery.region us-east-1

# 4. 可选：如果希望在不显式启用的情况下使用自动模式，请添加环境标记
echo 'export AWS_PROFILE=default' >> ~/.bashrc
echo 'export AWS_REGION=us-east-1' >> ~/.bashrc
source ~/.bashrc

# 5. 验证是否发现模型
openclaw models list
```

## 高级配置

<AccordionGroup>
  <Accordion title="推理配置文件">
    OpenClaw 会在基础模型之外发现**区域和全局推理配置文件**。当配置文件映射到已知基础模型时，该配置文件会继承该模型的能力（上下文窗口、最大令牌数、推理、视觉），并自动注入正确的 Bedrock 请求区域。这意味着跨区域 Claude 配置文件无需手动覆盖提供商即可工作。全局跨区域配置文件（`global.*`）会在 `openclaw models list` 中优先列出，因为它们通常能提供更好的容量和自动故障转移。

    推理配置文件 ID 的形式类似 `us.anthropic.claude-opus-4-6-v1`（区域）
    或 `anthropic.claude-opus-4-6-v1`（全局）。如果后端模型已存在于
    发现结果中，配置文件会继承其完整能力集；
    否则将应用安全默认值。

    无需额外配置。只要启用了发现，并且 IAM
    主体具有 `bedrock:ListInferenceProfiles`，配置文件就会与
    基础模型一起出现在 `openclaw models list` 中。

  </Accordion>

  <Accordion title="服务层级">
    某些 Bedrock 模型支持使用 `service_tier` 参数优化成本
    或延迟。可使用以下层级：

    | 层级 | 描述 |
    |------|-------------|
    | `default` | 标准 Bedrock 层级 |
    | `flex` | 为能够容忍更长延迟的工作负载提供折扣处理 |
    | `priority` | 为延迟敏感型工作负载提供优先处理 |
    | `reserved` | 为稳态工作负载提供预留容量 |

    对于 Bedrock 模型请求，可通过 `agents.defaults.params` 设置 `serviceTier`（或 `service_tier`），也可以在
    `agents.defaults.models["<model-key>"].params` 中按模型设置：

    ```json5
    {
      agents: {
        defaults: {
          params: {
            serviceTier: "flex", // 应用于所有模型
          },
          models: {
            "amazon-bedrock/mistral.mistral-large-3-675b-instruct": {
              params: {
                serviceTier: "priority", // 按模型覆盖
              },
            },
          },
        },
      },
    }
    ```

    有效值为 `default`、`flex`、`priority` 和 `reserved`。Claude
    Fable 5、Opus 5 和 Sonnet 5 仅支持 `default` 层级；如果为这些模型请求
    `flex`、`priority` 或 `reserved`，OpenClaw 会发出警告并忽略该请求。对于
    其他模型，并非每个模型都支持所有层级——不受支持的层级会
    返回 Bedrock 验证错误，而且错误消息可能具有
    误导性（例如显示“The provided model identifier is invalid”，
    而不是指出问题出在层级上）。如果看到此错误，请检查
    该模型是否支持请求的层级。

  </Accordion>

  <Accordion title="Claude Opus 5、4.8 和 4.7 的温度">
    Bedrock 会拒绝 Claude Opus 5、Opus 4.8 和 Opus 4.7 的
    `temperature` 参数。对于任何匹配的 Bedrock 引用，OpenClaw 都会自动省略
    `temperature`，包括基础模型 ID、命名推理配置文件、底层模型通过
    `bedrock:GetInferenceProfile` 解析为 Opus 5/4.8/4.7 的应用程序推理配置文件，
    以及带可选区域前缀的点分 `opus-4.7`/`opus-4.8` 变体
    （`us.`、`eu.`、`ap.`、`apac.`、`au.`、`jp.`、
    `global.`）。无需配置选项，并且该省略同时适用于
    请求选项对象和 `inferenceConfig` 载荷字段。
  </Accordion>

  <Accordion title="Claude Opus 5">
    在 Messages API 的 Bedrock 端点上使用 `amazon-bedrock/anthropic.claude-opus-5`，
    或者使用 Bedrock 设备发现中出现的区域/全局推理配置文件，例如
    `global.anthropic.claude-opus-5`。
    OpenClaw 会应用 1,000,000 token 上下文窗口、128,000 token 输出
    限制、图像输入、提示词缓存、拒绝安全流式传输，以及原生
    `xhigh`/`max` 工作量级别。

    自适应思考默认为 `high`。`/think off` 会禁用思考，而
    `/think xhigh|max` 会保持启用自适应思考。OpenClaw 会省略自定义
    采样参数和不受支持的非默认服务层级。

  </Accordion>

  <Accordion title="Claude Fable 5">
    在 `us-east-1` 中使用 `amazon-bedrock/anthropic.claude-fable-5`，或使用
    区域推理 ID，例如 `us.anthropic.claude-fable-5`。
    OpenClaw 会应用 Fable 的 1M 上下文窗口、128K 输出限制、始终启用的
    自适应思考和受支持的工作量映射。`/think off` 和
    `/think minimal` 会映射到 `low`；温度和强制工具选择控制项
    会被省略，与 Opus 4.7/4.8 路由一致。流式输出会暂缓，
    直到 Bedrock 返回终止状态，以免流式传输中途的拒绝
    暴露部分文本。

    AWS 要求明确选择加入 `provider_data_share` 数据保留后，
    才能使用 Fable。提示词和补全内容会与 Anthropic 共享，
    并出于信任与安全目的最多保留 30 天。启用该模型前，请查看并配置
    [Bedrock 数据保留](https://docs.aws.amazon.com/bedrock/latest/userguide/data-retention.html)。

  </Accordion>

  <Accordion title="Claude Mythos 5">
    Claude Mythos 5 仅向获得所需有限访问权限批准的账户
    通过 Bedrock 提供。OpenClaw 可识别基础模型
    `anthropic.claude-mythos-5`，以及区域或全局推理配置文件，例如
    `us.anthropic.claude-mythos-5`。

    OpenClaw 会应用 1,000,000 token 上下文窗口、128,000 token 输出
    限制、图像输入、提示词缓存、拒绝安全流式传输和原生
    工作量级别。自适应思考始终启用：`/think off` 和
    `/think minimal` 会映射到 `low`，而 `xhigh` 和 `max` 仍然可用。
    自定义采样值和强制工具选择值会被省略。

  </Accordion>

  <Accordion title="Claude Sonnet 5">
    AWS 记录了 Sonnet 5 对
    [`bedrock-runtime` 和 `bedrock-mantle` 端点](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5.html)
    的支持。OpenClaw 可识别 Bedrock 基础模型
    `anthropic.claude-sonnet-5`，以及区域或全局推理配置文件，例如
    `us.anthropic.claude-sonnet-5`。它会应用 1,000,000 token 上下文
    窗口、128,000 token 输出限制、图像输入、原生工作量级别、
    提示词缓存和拒绝安全流式传输。

    Bedrock 会为 Sonnet 5 保持启用自适应思考。OpenClaw 默认为
    `high`；`/think off` 和 `/think minimal` 会映射到 `low`，因为此路由
    无法禁用思考。启用自适应思考时，会省略自定义温度值和强制工具选择值。

  </Accordion>

  <Accordion title="防护机制">
    通过向 `amazon-bedrock` 插件配置添加 `guardrail` 对象，
    可以将 [Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)
    应用于所有 Bedrock 模型调用。防护机制可用于强制实施内容过滤、
    主题拒绝、词语过滤、敏感信息过滤和上下文依据检查。

    ```json5
    {
      plugins: {
        entries: {
          "amazon-bedrock": {
            config: {
              guardrail: {
                guardrailIdentifier: "abc123", // 防护机制 ID 或完整 ARN
                guardrailVersion: "1", // 版本号或 "DRAFT"
                streamProcessingMode: "sync", // 可选："sync" 或 "async"
                trace: "enabled", // 可选："enabled"、"disabled" 或 "enabled_full"
              },
            },
          },
        },
      },
    }
    ```

    `guardrailIdentifier` 和 `guardrailVersion` 为必填项。

    | 选项 | 说明 |
    | ------ | ----------- |
    | `guardrailIdentifier` | 防护机制 ID（例如 `abc123`）或完整 ARN（例如 `arn:aws:bedrock:us-east-1:123456789012:guardrail/abc123`）。 |
    | `guardrailVersion` | 已发布的版本号，或表示工作草稿的 `"DRAFT"`。 |
    | `streamProcessingMode` | 在流式传输期间进行防护机制评估的 `"sync"` 或 `"async"`。如果省略，Bedrock 将使用其默认值。 |
    | `trace` | 用于调试的 `"enabled"` 或 `"enabled_full"`；在生产环境中省略或设为 `"disabled"`。 |

    <Warning>
    Gateway 网关使用的 IAM 主体除标准调用权限外，还必须拥有 `bedrock:ApplyGuardrail` 权限。
    </Warning>

  </Accordion>

  <Accordion title="用于记忆搜索的嵌入">
    Bedrock 也可作为[记忆搜索](/zh-CN/concepts/memory-search)的嵌入提供商。
    此项配置与推理提供商分开——将 `memory.search.provider` 设为
    `"bedrock"`：

    ```json5
    {
      memory: {
        search: {
          provider: "bedrock",
          model: "amazon.titan-embed-text-v2:0", // 默认值
        },
      },
    }
    ```

    Bedrock 嵌入使用与推理相同的 AWS SDK 凭证链（实例
    角色、SSO、访问密钥、共享配置和 Web 身份）。无需
    API 密钥。

    支持的嵌入模型包括 Amazon Titan Embed（v1、v2）、Amazon Nova
    Embed、Cohere Embed（v3、v4）和 TwelveLabs Marengo。有关
    完整模型列表和维度选项，请参阅
    [记忆配置参考——Bedrock](/zh-CN/reference/memory-config#bedrock-embedding-config)。

  </Accordion>

  <Accordion title="说明和注意事项">
    - Bedrock 要求在你的 AWS 账户/区域中启用**模型访问权限**。
    - 自动设备发现需要 `bedrock:ListFoundationModels` 和
      `bedrock:ListInferenceProfiles` 权限。
    - 如果依赖自动模式，请在 Gateway 网关主机上设置一个受支持的 AWS 身份验证环境变量标记。
      如果希望使用不带环境变量标记的 IMDS/共享配置身份验证，请设置
      `plugins.entries.amazon-bedrock.config.discovery.enabled: true`。
    - OpenClaw 按以下顺序显示凭证来源：`AWS_BEARER_TOKEN_BEDROCK`，
      然后是 `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`，再后是 `AWS_PROFILE`，最后是
      默认 AWS SDK 链。
    - 推理支持取决于模型；请查看 Bedrock 模型卡以了解
      当前能力。
    - 如果更倾向于托管密钥流程，也可以在 Bedrock 前面放置一个与 OpenAI 兼容的
      代理，并改为将其配置为 OpenAI provider。
  </Accordion>
</AccordionGroup>

## 相关内容

<CardGroup cols={2}>
  <Card title="模型选择" href="/zh-CN/concepts/model-providers" icon="layers">
    选择提供商、模型引用和故障转移行为。
  </Card>
  <Card title="记忆搜索" href="/zh-CN/concepts/memory-search" icon="magnifying-glass">
    用于记忆搜索配置的 Bedrock 嵌入。
  </Card>
  <Card title="记忆配置参考" href="/zh-CN/reference/memory-config#bedrock-embedding-config" icon="database">
    完整的 Bedrock 嵌入模型列表和维度选项。
  </Card>
  <Card title="故障排查" href="/zh-CN/help/troubleshooting" icon="wrench">
    常规故障排查和常见问题。
  </Card>
</CardGroup>
