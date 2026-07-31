---
read_when:
    - 你希望在无需 API key 或模型服务器的情况下进行本地文本推理
    - 你希望使用本地 GGUF 模型生成记忆搜索嵌入向量
    - 你正在配置 memory.search.provider = "local"
    - 你需要拥有 node-llama-cpp 运行时的 OpenClaw 插件
sidebarTitle: llama.cpp Provider
summary: 使用 llama.cpp 在 OpenClaw 中运行本地 GGUF 文本推理和记忆嵌入
title: llama.cpp 提供商
x-i18n:
    generated_at: "2026-07-26T06:50:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 88e6d66943adcbc602421b8cc00359b3ed87357194c3ffaa845c1db7fbcd9c38
    source_path: plugins/llama-cpp.md
    workflow: 16
---

`llama-cpp` 是官方外部提供商插件，用于进程内本地 GGUF
文本推理和嵌入。它注册文本提供商 `llama-cpp`、
嵌入提供商 `local`，并拥有 `node-llama-cpp` 原生运行时。

使用本地推理或本地记忆嵌入之前，请先安装该插件：

```bash
openclaw plugins install @openclaw/llama-cpp-provider
```

主 `openclaw` npm 包不包含 `node-llama-cpp`。将原生依赖保留在此插件中，
可防止常规 OpenClaw npm 更新删除手动安装在 OpenClaw
包目录内的运行时。

## 本地文本推理

在交互式新手引导期间选择 **本地模型（llama.cpp）**。下载默认模型之前，
OpenClaw 会先询问：

`hf:bartowski/Qwen_Qwen3-4B-Instruct-2507-GGUF/Qwen_Qwen3-4B-Instruct-2507-Q4_K_M.gguf`

Qwen3 4B Instruct 2507 Q4_K_M 文件约为 2.5 GB。请为模型权重预留约 3 GB
内存，此外还需要上下文和 OpenClaw 运行时开销。默认上下文会自动调整大小，
上限为 8,192 个 token，因此在 8 GB 内存的计算机上仍可实际使用。仅当计算机
拥有足够内存时，才配置更大的上下文。

新手引导的发现检查是只读的。仅当默认或已配置的 GGUF 文件已存在于模型缓存中时，
它才会自动提供 llama.cpp 选项；发现期间绝不会下载。Ollama 和 LM Studio
仍是独立的本地服务选项，并保留各自的发现流程。手动选择 llama.cpp
才会提示下载默认模型。

该提供商使用 GGUF 模型内嵌的聊天模板和原生 node-llama-cpp
函数调用。文本逐 token 流式传输。工具调用会返回 OpenClaw 执行，
而不是在 node-llama-cpp 内部运行。

### 使用其他 GGUF 模型

将模型添加到 `models.providers.llama-cpp`。在 `params.modelPath` 中填写本地路径
或完整的 `hf:` 文件 URI：

```json5
{
  models: {
    mode: "merge",
    providers: {
      "llama-cpp": {
        baseUrl: "local://llama-cpp",
        api: "openai-completions",
        params: {
          modelCacheDir: "~/.node-llama-cpp/models",
        },
        models: [
          {
            id: "my-local-model",
            name: "My local GGUF",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 2048,
            params: {
              modelPath: "~/Models/my-model.Q4_K_M.gguf",
              contextSize: 8192,
            },
            compat: { supportsTools: true },
          },
        ],
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "llama-cpp/my-local-model" },
    },
  },
}
```

推理绝不会隐式下载缺失的模型。对于自定义 `hf:` URI，
请先将 GGUF 下载到 `modelCacheDir`。发现过程使用 node-llama-cpp
自身的只读缓存解析器，包括仓库、分支和拆分文件命名。

## 记忆嵌入配置

将 `memory.search.provider` 设置为 `local`：

```json5
{
  memory: {
    search: {
      provider: "local",
      local: {
        modelPath: "hf:ggml-org/embeddinggemma-300m-qat-q8_0-GGUF/embeddinggemma-300m-qat-Q8_0.gguf",
      },
    },
  },
}
```

`local.modelPath` 默认为上面所示的 `hf:` URI（`embeddinggemma-300m-qat-Q8_0.gguf`）。
如需使用其他模型，请将其指向不同的 `hf:` URI 或本地
`.gguf` 文件。`local.modelCacheDir` 会覆盖已下载模型的缓存位置
（默认值：`~/.node-llama-cpp/models`），而 `local.contextSize` 接受整数或
`"auto"`。

当 `local.contextSize` 为数值时，该提供商还会将此要求传递给
node-llama-cpp 的自动 GPU 层放置机制。这样，node-llama-cpp 可在保留其内存
安全检查的同时，共同容纳模型和嵌入上下文。使用 `"auto"` 时，
node-llama-cpp 会保留其常规自动放置行为。

## 原生运行时

使用 Node 24 可获得最顺畅的原生安装体验。使用 pnpm 的源码检出可能需要
批准并重新构建原生依赖：

```bash
pnpm approve-builds
pnpm rebuild node-llama-cpp
```

## 记忆运行时诊断

提供商加载后运行 `openclaw memory status --deep`，以检查所选后端和构建版本、
设备名称、GPU 卸载层数、请求的上下文大小，以及最近观测到的 VRAM
或统一内存快照。VRAM 值包含观测时间戳，因为被动状态读取不会重新加载模型
或轮询设备。

如果正在运行的 Gateway 网关已使用本地提供商，同样的最后已知信息也可能显示在
`openclaw doctor` 中。普通的状态或 Doctor 命令不会仅为了收集诊断信息
而加载模型。

## 故障排查

如果 `node-llama-cpp` 缺失或加载失败，OpenClaw 会报告失败并提供以下信息：

1. 安装插件：`openclaw plugins install @openclaw/llama-cpp-provider`。
2. 使用 Node 24 进行原生安装/更新。
3. 从 pnpm 源码检出中执行：`pnpm approve-builds`，然后执行 `pnpm rebuild node-llama-cpp`。

如果需要不依赖进程内原生依赖的本地推理，请改用 Ollama 或 LM Studio
提供商。如果需要更省事的本地嵌入，请将 `memory.search.provider` 改为远程
嵌入提供商，例如 `lmstudio`、`ollama`、
`openai` 或 `voyage`。
