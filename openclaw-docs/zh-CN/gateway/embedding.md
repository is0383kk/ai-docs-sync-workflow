---
read_when:
    - 在桌面或服务器应用中嵌入 OpenClaw
    - 将 Gateway 网关作为子进程进行监管
    - 无需抓取日志即可处理 Gateway 网关就绪、重启、关闭或配置无效的情况
summary: 从 Electron 或其他宿主应用中将 OpenClaw Gateway 网关作为子进程进行监管
title: 嵌入 OpenClaw
x-i18n:
    generated_at: "2026-07-26T05:48:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ca67e03994f21446bfeca58c95c2cb624dde767b9983a89982627145f80dfb90
    source_path: gateway/embedding.md
    workflow: 16
---

嵌入宿主应监管已安装的 `openclaw` 可执行文件，使用
Gateway 网关 WebSocket 协议作为其控制平面，并将子进程视为
可替换的运行时。这样可明确进程所有权、就绪状态、故障恢复
和升级，而无须依赖 OpenClaw 的私有状态布局。

有关客户端身份验证和重连状态，请阅读
[构建 Gateway 客户端](https://docs.openclaw.ai/gateway/clients)。

## 使用嵌入预设启动子进程

使用真实的 `node_modules` 安装并生成软件包可执行文件。对于负责
设备发现、重启和渠道生命周期的宿主，一个实用的基准配置是：

```ts
import { spawn } from "node:child_process";
import { dirname, resolve } from "node:path";
import { fileURLToPath } from "node:url";

// 提供由宿主应用管理的真实 Node 运行时的绝对路径。
declare const hostNodeExecutable: string;

const packageEntry = fileURLToPath(import.meta.resolve("openclaw"));
const openclawEntry = resolve(dirname(packageEntry), "..", "openclaw.mjs");
const gateway = spawn(hostNodeExecutable, [openclawEntry, "gateway", "--allow-unconfigured"], {
  env: {
    ...process.env,
    OPENCLAW_DISABLE_BONJOUR: "1",
    OPENCLAW_EXEC_SHELL_SNAPSHOT: "0",
    OPENCLAW_NO_RESPAWN: "1",
    OPENCLAW_SKIP_CHANNELS: "1",
  },
  stdio: ["ignore", "inherit", "inherit"],
});
```

如上所示，通过已安装的软件包解析 OpenClaw；不要假定宿主进程的
`PATH` 中存在项目本地的 `openclaw` 二进制文件。该示例
继承输出，因此子进程不会因 stdout 或 stderr 管道已满而阻塞。如果宿主
改为捕获这些流，请在生成子进程后立即附加使用方。

| 设置                          | 嵌入效果                                                                                                                                                                           |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `OPENCLAW_DISABLE_BONJOUR=1`     | 当宿主负责设备发现时，禁用由 Gateway 网关负责的局域网多播通告。                                                                                                             |
| `OPENCLAW_NO_RESPAWN=1`          | 在非托管嵌入子进程中，阻止 OpenClaw 将更新重启交给分离的子进程。常规重启仍在进程内进行，因此宿主继续拥有被跟踪 PID 的所有权。 |
| `OPENCLAW_EXEC_SHELL_SNAPSHOT=0` | 禁用宿主 Exec 命令的登录 shell 快照捕获。                                                                                                                              |
| `OPENCLAW_SKIP_CHANNELS=1`       | 跳过渠道启动和重新加载。仅当嵌入应用需要纯控制平面或仅限 WebChat 的 Gateway 网关时才设置此项。                                                                        |

`--allow-unconfigured` 仅绕过 `gateway.mode=local` 启动保护。
它不会写入配置或修复无效文件。当嵌入应用通过新手引导、配置 CLI
或 Gateway RPC 提供常规本地配置时，请省略此项。

### Electron shell 快照警告

Shell 快照捕获会从登录 shell 运行 `process.execPath -e <script>`。在
普通 Node 进程中，`process.execPath` 是 Node 可执行文件。在 Electron 下，
它是 Electron 二进制文件，可能会将调用解释为应用启动，
并显示“Unable to find Electron app”弹窗。请在 Gateway 网关子进程的
环境中设置 `OPENCLAW_EXEC_SHELL_SNAPSHOT=0`，而不只是在渲染器进程中设置。
出于同样的原因，`hostNodeExecutable` 必须指向真实的 Node 运行时，
而不是 Electron 的 `process.execPath`。

## 按退出代码处理无效配置

对于配置类启动失败（包括无效配置），Gateway 网关启动使用退出代码
`78`（`EX_CONFIG`）。应根据退出代码进行分支处理，
而不是抓取供人阅读的 stderr：

1. 针对与 Gateway 网关子进程相同的配置和
   状态环境运行 `openclaw doctor --fix --yes --non-interactive`。
2. Doctor 成功退出后，重试一次 Gateway 网关启动。
3. 如果子进程再次以 `78` 退出，请停止修复循环，并向用户显示配置
   失败。

保留 stderr 以供诊断，但不要根据其中的措辞做出生命周期决策。

成功启动后，无效的实时配置编辑造成的破坏较小。配置监视器会记录
已跳过重新加载，并继续使用最后一次接受的内存中配置提供服务。修复
文件后，让监视器接受下一个有效快照。

## 等待协议就绪

使用 WebSocket 信号，而不是日志子字符串：

1. 打开 Gateway 网关 WebSocket。
2. 等待 `connect.challenge` 事件。它证明监听器已接受
   WebSocket，并且可以开始质询握手。
3. 发送带有质询绑定设备签名的 `connect`。
4. 将 `hello-ok` 视为已通过身份验证的 RPC 的应用就绪信号。

质询特意早于完整初始化。如果启动
边车仍在等待，`connect` 会返回可重试的 `UNAVAILABLE` 错误，其中包含
`details.reason: "startup-sidecars"`、有界的 `retryAfterMs`，然后使用代码
`1013` 和原因 `gateway starting` 关闭。
使用来自 `@openclaw/gateway-protocol/startup-unavailable` 的
`resolveGatewayStartupRetryAfterMs` 或参考客户端的内置
策略，然后重新连接。

## 解释重启和关闭

在有序关闭之前，Gateway 网关会广播一个包含 `reason`
和 `restartExpectedMs` 的 `shutdown` 事件。非空的 `restartExpectedMs`
表示预期进行进程内或受监管的重启；`null` 表示最终关闭。

这两种情况下，后续 WebSocket 关闭代码都是 `1012`。
普通客户端的关闭原因在两种情况下也都是 `service restart`，因此关闭代码和
原因都无法区分重启与关闭。当先前的 `shutdown` 载荷到达时，请将其保留，
并结合宿主自身的停止意图和子进程退出状态进行判断。如果连接在没有该事件的情况下
消失，请使用常规的有界重连和子进程监管策略。

## 使用 RPC，而不是状态文件

让 Gateway 网关成为 OpenClaw 状态的唯一所有者。常见的嵌入操作
已有相应的 RPC 方法：

| 任务                          | RPC 方法                                          |
| ----------------------------- | ---------------------------------------------------- |
| 会话目录和生命周期 | `sessions.list`、`sessions.patch`、`sessions.delete` |
| 对话记录显示            | `chat.history`                                       |
| 成本和使用量报告        | `usage.cost`、`sessions.usage`                       |
| 模型凭据状态       | `models.authStatus`                                  |
| 配置                 | `config.get`、`config.patch`                         |

`config.get` 会在返回快照前隐去敏感值和 SecretRef 标识符。
写入方法也会返回已隐去敏感信息的配置。客户端必须将隐去标记视为不透明值，
并使用文档记录的配置写入契约；绝不能期望 Gateway 网关返回明文密钥。

不要通过读取或修改 `~/.openclaw` 下的文件、SQLite 表、对话记录文件
或缓存目录来实现应用功能。这些布局是私有运行时实现细节，
可以在不保持协议兼容性的情况下移动或更改。

## 安装；不要扁平化

根 `openclaw` 软件包不是单文件内嵌目标。`dist/extensions`
下的内置运行时文件保留 `openclaw/plugin-sdk/*` 等裸自引用导入，
而 npm 软件包会有意排除每个扩展的 `node_modules` 目录树。

通过 npm、pnpm 或其他常规 Node 软件包安装方式安装 OpenClaw，以便
Node 能解析软件包导出和根依赖树。生成已安装的 `openclaw`
可执行文件。不要只复制 `dist`，不要将软件包扁平化到应用
捆绑包中，也不要内嵌选定的扩展文件。

## 相关内容

- [构建 Gateway 客户端](https://docs.openclaw.ai/gateway/clients)
- [Gateway 网关协议](https://docs.openclaw.ai/gateway/protocol)
- [Gateway CLI](https://docs.openclaw.ai/cli/gateway)
- [外部应用的 Gateway 网关集成](https://docs.openclaw.ai/gateway/external-apps)
