---
read_when:
    - 你想使用容器化的 Gateway 网关，而不是在本地安装
    - 你正在验证 Docker 流程
summary: OpenClaw 的可选 Docker 设置和新手引导
title: Docker
x-i18n:
    generated_at: "2026-07-26T05:50:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c1784bd49f6847db75633840a4d5a8e49205200728bd2e9d59b646a446e508d6
    source_path: install/docker.md
    workflow: 16
---

Docker 是**可选的**。可将其用于隔离的、用完即弃的 Gateway 网关环境，或用于未进行本地安装的主机。如果你已在自己的计算机上进行开发，请改用常规安装流程。

启用 `agents.defaults.sandbox` 后，默认沙箱后端会使用 Docker，但沙箱隔离默认关闭，且不要求 Gateway 网关本身在 Docker 中运行。还可以使用 SSH 和 OpenShell 沙箱后端；请参阅[沙箱隔离](/zh-CN/gateway/sandboxing)。

要托管多个用户？有关每租户一个隔离单元的模型，请参阅[多租户托管](/zh-CN/gateway/multi-tenant-hosting)。

## 前置条件

- Docker Desktop（或 Docker Engine）+ Docker Compose v2
- 镜像构建至少需要 2 GB RAM（在 1 GB 主机上，`pnpm install` 可能因内存不足而被终止，退出码为 137）
- 有足够空间存储镜像和日志
- 在 VPS/公共主机上，请查看[网络暴露安全加固](/zh-CN/gateway/security)，尤其是 Docker `DOCKER-USER` 防火墙链

## 容器化 Gateway 网关

<Steps>
  <Step title="构建镜像">
    在仓库根目录中运行：

    ```bash
    ./scripts/docker/setup.sh
    ```

    这会在本地将 Gateway 网关镜像构建为 `openclaw:local`。若要改用预构建镜像：

    ```bash
    export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
    ./scripts/docker/setup.sh
    ```

    预构建镜像会优先发布到 [GitHub Container Registry](https://github.com/openclaw/openclaw/pkgs/container/openclaw)。GHCR 是发布自动化、固定版本部署和来源检查的主要注册表。同一版本还会在 Docker Hub 发布镜像副本 `openclaw/openclaw`：

    ```bash
    export OPENCLAW_IMAGE="openclaw/openclaw:latest"
    ./scripts/docker/setup.sh
    ```

    请使用 `ghcr.io/openclaw/openclaw` 或 `openclaw/openclaw`，并避免使用非官方镜像源，因为它们与 OpenClaw 的发布时间和保留策略并不一致。特定版本标签包括 `2026.2.26` 等正式版本和 `2026.2.26-beta.1` 等预发布版本。稳定版本会更新 `latest` 和 `main`；月末 Gateway 网关版本仅更新 `extended-stable`。变体包括 `slim`、`main-slim`、`extended-stable-slim`、`latest-browser`、`main-browser` 和 `extended-stable-browser`。默认镜像内置 `codex` 和 `diagnostics-otel` 插件。另有内置 Chromium 的 `-browser` 变体，便于使用[沙箱隔离浏览器](/zh-CN/gateway/sandboxing#sandboxed-browser)工具，无需在首次运行时安装 Playwright。

  </Step>

  <Step title="在隔离网络环境中重新运行">
    在离线主机上，请先传输并加载镜像：

    ```bash
    docker load -i openclaw-image.tar
    export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
    ./scripts/docker/setup.sh --offline
    ```

    `--offline` 会验证 `OPENCLAW_IMAGE` 已存在于本地，禁用 Compose 的隐式拉取和构建，然后运行常规流程：`.env` 同步、权限修复、新手引导、Gateway 网关配置同步和 Compose 启动。

    如果 `OPENCLAW_SANDBOX=1`，离线设置还会在 `OPENCLAW_DOCKER_SOCKET` 所指向的守护进程上检查已配置的默认沙箱镜像和各 Agent 沙箱镜像，包括基于 Docker 的浏览器镜像上的浏览器契约标签。如果所需镜像缺失或已过期，设置流程会退出且不更改沙箱配置，而不会错误地报告成功。

  </Step>

  <Step title="完成新手引导">
    设置脚本会自动运行新手引导：

    - 提示输入提供商 API 密钥
    - 生成 Gateway 网关令牌并将其写入 `.env`
    - 创建身份验证配置文件的密钥目录
    - 通过 Docker Compose 启动 Gateway 网关

    启动前的新手引导和配置写入会直接通过 `openclaw-gateway` 运行（使用 `--no-deps --entrypoint node`），因为 `openclaw-cli` 与 Gateway 网关共享网络命名空间，只有 Gateway 网关容器存在后才能工作。

  </Step>

  <Step title="打开 Control UI">
    打开 `http://127.0.0.1:18789/`，然后将写入 `.env` 的令牌粘贴到 Settings 中。如果已将容器切换为密码身份验证，请改用该密码。

    需要再次获取 URL？

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    ```

  </Step>

  <Step title="配置渠道（可选）">
    ```bash
    # WhatsApp（二维码）
    docker compose run --rm openclaw-cli channels login

    # Telegram
    docker compose run --rm openclaw-cli channels add --channel telegram --token "<token>"

    # Discord
    docker compose run --rm openclaw-cli channels add --channel discord --token "<token>"
    ```

    文档：[WhatsApp](/zh-CN/channels/whatsapp)、[Telegram](/zh-CN/channels/telegram)、[Discord](/zh-CN/channels/discord)

  </Step>
</Steps>

### 手动流程

```bash
BUILD_GIT_COMMIT="$(git rev-parse HEAD)"
BUILD_TIMESTAMP="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
docker build \
  --build-arg "GIT_COMMIT=${BUILD_GIT_COMMIT}" \
  --build-arg "OPENCLAW_BUILD_TIMESTAMP=${BUILD_TIMESTAMP}" \
  -t openclaw:local -f Dockerfile .
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js onboard --mode local --no-install-daemon
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js config set --batch-json '[{"path":"gateway.mode","value":"local"},{"path":"gateway.bind","value":"lan"},{"path":"gateway.controlUi.allowedOrigins","value":["http://localhost:18789","http://127.0.0.1:18789"]}]'
docker compose up -d openclaw-gateway
```

Docker 上下文会排除 `.git`。请按上述方式将源代码身份信息作为构建参数传入，以便镜像的“关于”页面显示当前检出提交和一个构建时间戳。`scripts/docker/setup.sh` 会自动解析并传递这两个值。

<Note>
请从仓库根目录运行 `docker compose`。如果启用了 `OPENCLAW_EXTRA_MOUNTS` 或 `OPENCLAW_HOME_VOLUME`，设置脚本会写入 `docker-compose.extra.yml`；请将其放在你自行维护的任何 `docker-compose.override.yml` 之后，例如 `-f docker-compose.yml -f docker-compose.override.yml -f docker-compose.extra.yml`。
</Note>

### 升级容器镜像

当你替换 OpenClaw 镜像但保留相同的挂载状态和配置时，新 Gateway 网关会在就绪前运行启动安全的升级迁移和插件收敛。常规镜像升级不应需要单独执行 `openclaw doctor --fix`。

如果启动过程无法安全完成这些修复，Gateway 网关会退出，而不会报告健康状态。如果配置了重启策略，Docker、Podman 或 Kubernetes 可能会显示 Gateway 网关容器正在反复重启。请保留已挂载的状态卷，然后使用 Gateway 网关所用的相同状态/配置挂载，将 `openclaw doctor --fix` 设为容器命令，用同一镜像运行一次：

```bash
docker run --rm -v <openclaw-state>:/home/node/.openclaw <image> openclaw doctor --fix
podman run --rm -v <openclaw-state>:/home/node/.openclaw <image> openclaw doctor --fix
```

Doctor 完成后，使用默认命令重新启动 Gateway 网关容器。在 Kubernetes 中，请在挂载到同一 PVC 的一次性 Job 或调试 Pod 中运行相同命令，然后重新启动 Deployment 或 StatefulSet。

### 环境变量

`scripts/docker/setup.sh` 接受的可选变量（Gateway 网关容器也可以直接通过 `docker-compose.yml` 接受）：

| 变量                                            | 用途                                                                                                              |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_IMAGE`                              | 使用远程镜像，而不是在本地构建                                                                                    |
| `OPENCLAW_IMAGE_APT_PACKAGES`                              | 在构建期间安装额外的 apt 软件包（以空格分隔）。旧版别名：`OPENCLAW_DOCKER_APT_PACKAGES`                                      |
| `OPENCLAW_IMAGE_PIP_PACKAGES`                              | 在构建期间安装额外的 Python 软件包（以空格分隔）                                                                  |
| `OPENCLAW_EXTENSIONS`                              | 编译/打包所选的受支持插件并安装其运行时依赖项（ID 以逗号或空格分隔）                                              |
| `OPENCLAW_DOCKER_BUILD_NODE_OPTIONS`                              | 覆盖本地源代码构建的 Node 选项（默认为 `--max-old-space-size=8192`）                                                       |
| `OPENCLAW_DOCKER_BUILD_TSDOWN_MAX_OLD_SPACE_MB`                              | 覆盖本地源代码构建中 tsdown 的堆大小（单位为 MB）                                                                 |
| `OPENCLAW_DOCKER_BUILD_SKIP_DTS`                              | 在仅运行时的本地镜像构建期间跳过声明输出（默认为 `1`）                                             |
| `OPENCLAW_INSTALL_BROWSER`                              | 在构建时将 Chromium + Xvfb 内置到镜像中                                                                           |
| `OPENCLAW_EXTRA_MOUNTS`                              | 额外的主机绑定挂载（以逗号分隔的 `source:target[:opts]`）                                                             |
| `OPENCLAW_HOME_VOLUME`                              | 将 `/home/node` 持久保存在具名 Docker 卷中                                                                  |
| `OPENCLAW_SANDBOX`                              | 选择启用沙箱引导（`1`、`true`、`yes`、`on`）                |
| `OPENCLAW_SKIP_ONBOARDING`                              | 跳过交互式新手引导步骤（`1`、`true`、`yes`、`on`）          |
| `OPENCLAW_DOCKER_SOCKET`                              | 覆盖 Docker 套接字路径                                                                                            |
| `OPENCLAW_DISABLE_BONJOUR`                              | 强制开启（`0`）或关闭（`1`）Bonjour/mDNS 广播；请参阅 [Bonjour / mDNS](#bonjour--mdns) |
| `OPENCLAW_DISABLE_BUNDLED_SOURCE_OVERLAYS`                              | 禁用内置插件源代码绑定挂载覆盖层                                                                                  |
| `OTEL_EXPORTER_OTLP_ENDPOINT`                              | 用于 OpenTelemetry 导出的共享 OTLP/HTTP 收集器端点                                                                |
| `OTEL_EXPORTER_OTLP_*_ENDPOINT`                              | 用于跟踪、指标或日志的信号专用 OTLP 端点                                                                          |
| `OTEL_EXPORTER_OTLP_PROTOCOL`                              | 覆盖 OTLP 协议。目前仅支持 `http/protobuf`                                                                     |
| `OTEL_SERVICE_NAME`                              | 用于 OpenTelemetry 资源的服务名称                                                                                 |
| `OTEL_SEMCONV_STABILITY_OPT_IN`                              | 选择启用最新的实验性 GenAI 语义属性                                                                                |
| `OPENCLAW_OTEL_PRELOADED`                              | 预加载 OpenTelemetry SDK 时，跳过启动第二个 OpenTelemetry SDK                                                     |

官方镜像不附带 Homebrew。在新手引导期间，如果 Linux 容器中没有 `brew`，OpenClaw 会隐藏仅限 brew 的 Skills 依赖项安装程序；请通过自定义镜像提供这些依赖项，或手动安装。对于 Debian 软件包形式的依赖项，请使用 `OPENCLAW_IMAGE_APT_PACKAGES`；对于 Python 依赖项，请使用 `OPENCLAW_IMAGE_PIP_PACKAGES`（它会在构建时运行 `python3 -m pip install --break-system-packages`，因此请固定版本，并且仅使用你信任的软件包索引）。

如果 Docker 报告 `ResourceExhausted`、`cannot allocate memory`，或在 `tsdown` 期间中止，请提高 Docker 构建器的内存限制，或使用更小的显式堆大小重试：

```bash
OPENCLAW_DOCKER_BUILD_NODE_OPTIONS=--max-old-space-size=4096 OPENCLAW_DOCKER_BUILD_TSDOWN_MAX_OLD_SPACE_MB=4096
```

### 包含所选插件的源代码构建镜像

`OPENCLAW_EXTENSIONS` 从源代码检出中选择插件清单 ID；
当现有源目录名称不同时，也接受这些名称。Docker
构建会将所选项一次性解析为源目录，安装生产
依赖，并且当所选插件通过
`openclaw.build.bundledDist: false` 单独发布时，将其运行时编译到根级内置
dist 中。这种仅用于 Docker 的打包方式不会改变插件的 npm 或 ClawHub
工件契约。未知、无效或有歧义的 ID 会导致镜像构建失败。
已知的仅依赖项/源代码 ID 会保留其现有的源代码和依赖项
暂存方式，而不会新增已编译的根级 dist 条目。包含
统一构建入口的所选插件必须成功编译；未选择的外部插件
源代码和运行时输出会被移除。

例如，以下命令会为 ClickClack、Slack 和 Microsoft Teams 构建独立的、多架构
FakeCo Gateway 网关镜像。ClawRouter
已经是根级 OpenClaw 运行时的一部分，因此 ClickClack 镜像仅选择
`clickclack`。显式传入空的浏览器参数可使默认镜像不包含
Chromium：

```bash
SOURCE_SHA="$(git rev-parse HEAD)"
BUILD_TIMESTAMP="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
REGISTRY="registry.example.com/fakeco"

build_gateway_image() {
  gateway="$1"
  selected_plugin="$2"
  docker buildx build \
    --platform linux/amd64,linux/arm64 \
    --build-arg "GIT_COMMIT=${SOURCE_SHA}" \
    --build-arg "OPENCLAW_BUILD_TIMESTAMP=${BUILD_TIMESTAMP}" \
    --build-arg "OPENCLAW_EXTENSIONS=${selected_plugin}" \
    --build-arg OPENCLAW_INSTALL_BROWSER= \
    --provenance=mode=max \
    --sbom=true \
    --tag "${REGISTRY}/openclaw-${gateway}:${SOURCE_SHA}" \
    --push \
    .
}

build_gateway_image clickclack clickclack
build_gateway_image slack slack
build_gateway_image teams msteams
```

对于单个原生本地构建，请使用 `--platform linux/arm64 --load` 或 `--platform linux/amd64 --load`。
多平台输出以及附加的 SBOM/来源证明
需要使用注册表或其他能够保留证明材料的 Buildx 输出。推送后，
检查清单并部署不可变摘要，而不是
可变的源 SHA 标签：

```bash
docker buildx imagetools inspect \
  "${REGISTRY}/openclaw-clickclack:${SOURCE_SHA}"
# 部署：registry.example.com/fakeco/openclaw-clickclack@sha256:<manifest-digest>
```

这些镜像适用于独立的、基于 OCI 的 Gateway 网关和通用 Docker 用户。
由 Crabhelm 管理的 Gateway 网关不会使用它们：该交付路径会构建一个
单独的 x86_64 设备归档，其中包含 OpenClaw npm tarball，并固定
Node、归档和清单摘要。请基于同一份已合入的 OpenClaw 源代码独立
构建该设备。

若要针对已打包镜像测试内置插件源代码，请将一个插件源目录挂载到其已打包的源路径上，例如 `OPENCLAW_EXTRA_MOUNTS=/path/to/fork/extensions/synology-chat:/app/extensions/synology-chat:ro`。这会覆盖同一插件 ID 对应的已编译 `/app/dist/extensions/synology-chat` 包。

### 可观测性

OpenTelemetry 导出从 Gateway 网关容器向外发送到你的 OTLP 收集器；无需发布 Docker 端口。若要在本地构建的镜像中包含内置导出器：

```bash
export OPENCLAW_EXTENSIONS="diagnostics-otel"
export OTEL_EXPORTER_OTLP_ENDPOINT="http://otel-collector:4318"
export OTEL_SERVICE_NAME="openclaw-gateway"
./scripts/docker/setup.sh
```

官方预构建镜像已内置 `diagnostics-otel`；仅当你已将其移除时，才需要自行安装 `clawhub:@openclaw/diagnostics-otel`。若要启用导出，请在配置中允许并启用 `diagnostics-otel` 插件，然后设置 `diagnostics.otel.enabled=true`（完整示例请参阅 [OpenTelemetry 导出](/zh-CN/gateway/opentelemetry)）。收集器身份验证标头通过 `diagnostics.otel.headers` 传递，而不是通过 Docker 环境变量。

Prometheus 指标复用已发布的 Gateway 网关端口。安装 `clawhub:@openclaw/diagnostics-prometheus`，启用 `diagnostics-prometheus` 插件，然后抓取：

```text
http://<gateway-host>:18789/api/diagnostics/prometheus
```

该路由受 Gateway 网关身份验证保护；不要暴露单独的公共 `/metrics` 端口或未经身份验证的反向代理路径。请参阅 [Prometheus 指标](/zh-CN/gateway/prometheus)。

### 健康检查

容器探针端点（无需身份验证）：

```bash
curl -fsS http://127.0.0.1:18789/healthz   # 存活状态
curl -fsS http://127.0.0.1:18789/readyz     # 就绪状态
```

镜像内置的 `HEALTHCHECK` 会探测 `/healthz`；反复失败会将容器标记为 `unhealthy`，以便编排器重启或替换它。

经过身份验证的深度健康快照：

```bash
docker compose exec openclaw-gateway node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"
```

### 局域网与 local loopback

`scripts/docker/setup.sh` 默认使用 `OPENCLAW_GATEWAY_BIND=lan`，因此主机上的 `http://127.0.0.1:18789` 可与 Docker 端口发布配合使用。

- `lan`（默认）：主机浏览器和主机 CLI 可以访问已发布的 Gateway 网关端口。
- `loopback`：只有容器网络命名空间内的进程才能直接访问 Gateway 网关。

<Note>
请使用 `gateway.bind` 中的绑定模式值（`lan` / `loopback` / `custom` / `tailnet` / `auto`），不要使用 `0.0.0.0` 或 `127.0.0.1` 等主机别名。
</Note>

### 主机本地提供商

在容器内，`127.0.0.1` 指向容器本身，而不是主机。对于在主机上运行的提供商，请使用 `host.docker.internal`：

| 提供商  | 主机默认 URL         | Docker 设置 URL                    |
| --------- | ------------------------ | ----------------------------------- |
| LM Studio | `http://127.0.0.1:1234`  | `http://host.docker.internal:1234`  |
| Ollama    | `http://127.0.0.1:11434` | `http://host.docker.internal:11434` |

内置设置会将这些 URL 用作 LM Studio/Ollama 新手引导的默认值，而 `docker-compose.yml` 会在 Linux Docker Engine 上将 `host.docker.internal` 映射到主机 Gateway 网关（Docker Desktop 在 macOS/Windows 上提供相同别名）。主机服务必须监听 Docker 可以访问的地址：

```bash
lms server start --port 1234 --bind 0.0.0.0
OLLAMA_HOST=0.0.0.0:11434 ollama serve
```

使用你自己的 Compose 文件或 `docker run`？请自行添加相同的映射，例如 `--add-host=host.docker.internal:host-gateway`。

### Docker 中的 Claude CLI 后端

官方镜像未预安装 Claude Code。请在容器的 `node` 用户环境中安装并登录，然后持久化该容器主目录，以免镜像升级清除二进制文件或身份验证状态。

对于全新安装，请在运行设置之前启用持久化 `/home/node` 卷：

```bash
export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
export OPENCLAW_HOME_VOLUME="openclaw_home"
./scripts/docker/setup.sh
```

对于现有安装，请先停止堆栈并重新加载当前的 `.env` 值——设置脚本始终根据当前 shell 和默认值重写 `.env`，它不会自行读取该文件：

```bash
set -a
. ./.env
set +a
export OPENCLAW_HOME_VOLUME="${OPENCLAW_HOME_VOLUME:-openclaw_home}"
./scripts/docker/setup.sh
```

如果 `.env` 包含 shell 无法加载的值，请先手动重新导出你依赖的内容（`OPENCLAW_IMAGE`、端口、绑定模式、自定义路径、`OPENCLAW_EXTRA_MOUNTS`、沙箱、跳过新手引导）。生成的叠加配置会为 `openclaw-gateway` 和 `openclaw-cli` 挂载主目录卷；请使用该叠加配置运行其余命令（如果你使用 `docker-compose.override.yml`，请先包含它）：

```bash
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  --entrypoint sh openclaw-cli -lc \
  'curl -fsSL https://claude.ai/install.sh | bash'
```

原生安装程序会将 `claude` 写入 `/home/node/.local/bin/claude`。
OpenClaw 镜像在 `PATH` 中包含 `/home/node/.local/bin`，因此内置的
Anthropic 插件无需覆盖适配器配置即可解析它。

从同一个持久化主目录登录并验证：

```bash
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  --entrypoint /home/node/.local/bin/claude openclaw-cli auth login
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  --entrypoint /home/node/.local/bin/claude openclaw-cli auth status --text
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  openclaw-cli models auth login \
  --provider anthropic --method cli --set-default
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  openclaw-cli models list --provider anthropic
```

然后使用内置的 `claude-cli` 后端：

```bash
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  openclaw-cli agent \
  --agent main \
  --model claude-cli/claude-sonnet-4-6 \
  --message "从 Docker Claude CLI 问好"
```

`OPENCLAW_HOME_VOLUME` 会持久化 `/home/node/.local/bin` 和 `/home/node/.local/share/claude` 下的原生安装，以及 `/home/node/.claude` 和 `/home/node/.claude.json` 下的 Claude Code 设置/身份验证。仅持久化 `/home/node/.openclaw` 并不足够；如果使用 `OPENCLAW_EXTRA_MOUNTS` 而不是主目录卷，请将所有这些 Claude 路径挂载到两个服务中。

<Note>
对于共享的生产自动化或可预测的 Anthropic 计费，请优先使用 Anthropic API 密钥路径。Claude CLI 复用会遵循 Claude Code 的已安装版本、账户登录、计费和更新行为。
</Note>

### Bonjour / mDNS

Docker 桥接网络通常无法可靠转发 Bonjour/mDNS 多播（`224.0.0.251:5353`）。当未设置 `OPENCLAW_DISABLE_BONJOUR` 时，内置 Bonjour 插件一旦检测到自己正在容器中运行，就会自动禁用局域网广播，因此不会因反复尝试桥接网络丢弃的多播而陷入崩溃循环。设置 `OPENCLAW_DISABLE_BONJOUR=1` 可无论检测结果如何都强制禁用它，设置 `0` 可强制启用它（仅适用于主机网络、macvlan 或其他已知支持 mDNS 多播的网络）。

否则，请为 Docker 主机使用已发布的 Gateway 网关 URL、Tailscale 或广域 DNS-SD。有关注意事项和故障排除，请参阅 [Bonjour 设备发现](/zh-CN/gateway/bonjour)。

### 存储和持久化

Docker Compose 将 `OPENCLAW_CONFIG_DIR` 绑定挂载到 `/home/node/.openclaw`，将 `OPENCLAW_WORKSPACE_DIR` 绑定挂载到 `/home/node/.openclaw/workspace`，并将 `OPENCLAW_AUTH_PROFILE_SECRET_DIR` 绑定挂载到 `/home/node/.config/openclaw`，因此这些路径可在容器替换后保留。当变量未设置时，`docker-compose.yml` 会回退到 `${HOME}` 下；如果连 `HOME` 本身也不存在，则回退到 `/tmp`，因此 `docker compose up` 在基础环境中绝不会生成源路径为空的卷规范。

该挂载的配置目录包含：

- `openclaw.json`：行为配置
- `agents/<agentId>/agent/auth-profiles.json`：存储的提供商 OAuth/API 密钥身份验证
- `.env`：由环境变量提供的运行时密钥，例如 `OPENCLAW_GATEWAY_TOKEN`

身份验证配置文件密钥目录存储用于 OAuth 身份验证配置文件令牌材料的本地加密密钥。请将它与 Docker 主机状态一起保留，但要与 `OPENCLAW_CONFIG_DIR` 分开。

已安装的可下载插件会将软件包状态存储在已挂载的 OpenClaw 主目录下，因此安装记录和软件包根目录可在容器替换后保留；Gateway 网关启动时不会重新生成内置插件的依赖项树。

有关完整的虚拟机持久化详情，请参阅 [Docker 虚拟机运行时——各项内容的持久化位置](/zh-CN/install/docker-vm-runtime#what-persists-where)。

**磁盘增长热点：** `media/`、每个智能体的 SQLite 数据库、旧版会话 JSONL 记录、共享 SQLite 状态数据库、已安装的插件软件包根目录，以及 `/tmp/openclaw/` 下的滚动文件日志。

### Shell 辅助工具（可选）

若要简化日常命令，请安装 [ClawDock](/zh-CN/install/clawdock)：

```bash
mkdir -p ~/.clawdock && curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/clawdock/clawdock-helpers.sh -o ~/.clawdock/clawdock-helpers.sh
echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc
```

如果你从旧的 `scripts/shell-helpers/clawdock-helpers.sh` 路径安装，请重新运行上述命令，以便本地辅助工具跟踪当前位置。然后使用 `clawdock-start`、`clawdock-stop`、`clawdock-dashboard` 等（运行 `clawdock-help` 可查看完整列表）。

<AccordionGroup>
  <Accordion title="为 Docker Gateway 网关启用智能体沙箱">
    ```bash
    export OPENCLAW_SANDBOX=1
    ./scripts/docker/setup.sh
    ```

    自定义套接字路径（例如无根 Docker）：

    ```bash
    export OPENCLAW_SANDBOX=1
    export OPENCLAW_DOCKER_SOCKET=/run/user/1000/docker.sock
    ./scripts/docker/setup.sh
    ```

    该脚本仅在沙箱先决条件检查通过后挂载 `docker.sock`。如果无法完成沙箱设置，它会将 `agents.defaults.sandbox.mode` 重置为 `off`。当 OpenClaw 沙箱处于活动状态时，对应轮次将禁用 Codex 代码模式（参见[沙箱隔离 § Docker 后端](/zh-CN/gateway/sandboxing#docker-backend)）；切勿将主机 Docker 套接字挂载到智能体沙箱容器中。

  </Accordion>

  <Accordion title="自动化 / CI（非交互式）">
    使用 `-T` 禁用 Compose 伪 TTY 分配：

    ```bash
    docker compose run -T --rm openclaw-cli gateway probe
    docker compose run -T --rm openclaw-cli devices list --json
    ```

  </Accordion>

  <Accordion title="共享网络安全注意事项">
    `openclaw-cli` 使用 `network_mode: "service:openclaw-gateway"`，使 CLI 命令可以通过 `127.0.0.1` 访问 Gateway 网关。请将其视为共享信任边界。Compose 配置会在 `openclaw-gateway` 和 `openclaw-cli` 上移除 `NET_RAW`/`NET_ADMIN`，并启用 `no-new-privileges`。
  </Accordion>

  <Accordion title="openclaw-cli 中的 Docker Desktop DNS 故障">
    某些 Docker Desktop 设置在移除 `NET_RAW` 后，共享网络中的 `openclaw-cli` 边车会出现 DNS 查询失败；在执行 `openclaw plugins install` 等由 npm 支持的命令时，这会表现为 `EAI_AGAIN`。正常运行时请保留默认的强化版 Compose 文件。以下覆盖配置仅为 `openclaw-cli` 容器恢复默认能力——请仅将其用于需要访问注册表的一次性命令，不要将其作为默认调用方式：

    ```bash
    printf '%s\n' \
      'services:' \
      '  openclaw-cli:' \
      '    cap_drop: !reset []' \
      > docker-compose.cli-no-dropped-caps.local.yml

    docker compose -f docker-compose.yml -f docker-compose.cli-no-dropped-caps.local.yml run --rm openclaw-cli plugins install <package>
    ```

    如果已经创建了长期运行的 `openclaw-cli` 容器，请使用相同的覆盖配置重新创建它——`docker compose exec`/`docker exec` 无法更改已创建容器的 Linux 能力。

  </Accordion>

  <Accordion title="权限和 EACCES">
    该镜像以 `node`（uid 1000）运行。如果在 `/home/node/.openclaw` 上看到权限错误，请确保主机绑定挂载归 uid 1000 所有：

    ```bash
    sudo chown -R 1000:1000 /path/to/openclaw-config /path/to/openclaw-workspace
    ```

    同样的不匹配也可能表现为先出现 `blocked plugin candidate: suspicious ownership (... uid=1000, expected uid=0 or root)`，随后出现 `plugin present but blocked`——进程 uid 与所挂载插件目录的所有者不一致。建议使用默认 uid 1000 运行，并修复绑定挂载的所有权。仅当你有意长期以 root 身份运行 OpenClaw 时，才将 `/path/to/openclaw-config/npm` 的所有者更改为 `root:root`。

  </Accordion>

  <Accordion title="加快重新构建">
    调整 Dockerfile 的顺序以缓存依赖层，从而避免在锁文件未更改时重新运行 `pnpm install`：

    ```dockerfile
    FROM node:24-bookworm
    RUN curl -fsSL https://bun.sh/install | bash
    ENV PATH="/root/.bun/bin:${PATH}"
    RUN corepack enable
    WORKDIR /app
    COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
    COPY ui/package.json ./ui/package.json
    COPY scripts ./scripts
    RUN pnpm install --frozen-lockfile
    COPY . .
    RUN pnpm build
    RUN pnpm ui:install
    RUN pnpm ui:build
    ENV NODE_ENV=production
    CMD ["node","dist/index.js"]
    ```

  </Accordion>

  <Accordion title="高级用户容器选项">
    默认镜像以安全性为优先，并以非 root 用户 `node` 运行。要获得功能更完整的容器：

    1. **持久化 `/home/node`**：`export OPENCLAW_HOME_VOLUME="openclaw_home"`
    2. **将系统依赖烘焙到镜像中**：`export OPENCLAW_IMAGE_APT_PACKAGES="git curl jq"`
    3. **将 Python 依赖烘焙到镜像中**：`export OPENCLAW_IMAGE_PIP_PACKAGES="requests==2.32.5 humanize==4.14.0"`
    4. **将 Playwright Chromium 烘焙到镜像中**：`export OPENCLAW_INSTALL_BROWSER=1`，或使用官方 `-browser` 镜像标签
    5. **或将 Playwright 浏览器安装到持久化卷中**：
       ```bash
       docker compose run --rm openclaw-cli \
         node /app/node_modules/playwright-core/cli.js install chromium
       ```
    6. **持久化浏览器下载内容**：使用 `OPENCLAW_HOME_VOLUME` 或 `OPENCLAW_EXTRA_MOUNTS`。OpenClaw 会在 Linux 上自动检测镜像中由 Playwright 管理的 Chromium。

  </Accordion>

  <Accordion title="OpenAI Codex OAuth（无头 Docker）">
    如果在向导中选择 OpenAI Codex OAuth，它会打开一个浏览器 URL。在 Docker 或无头设置中，复制最终跳转到的完整重定向 URL，并将其粘贴回向导以完成身份验证。
  </Accordion>

  <Accordion title="基础镜像元数据">
    运行时镜像使用 `node:24-bookworm-slim`，并以 PID 1 运行 `tini`，从而在长期运行的容器中回收僵尸进程并正确处理信号。它会发布 OCI 基础镜像注解，包括 `org.opencontainers.image.base.name` 和 `org.opencontainers.image.source`。Dependabot 会刷新固定的 Node 基础镜像摘要；发布构建不会运行单独的发行版升级层。参见 [OCI 镜像注解](https://github.com/opencontainers/image-spec/blob/main/annotations.md)。
  </Accordion>
</AccordionGroup>

### 在 VPS 上运行？

有关共享 VM 部署步骤（包括二进制文件烘焙、持久化和更新），请参见 [Hetzner（Docker VPS）](/zh-CN/install/hetzner)和 [Docker VM 运行时](/zh-CN/install/docker-vm-runtime)。

## 智能体沙箱

使用 Docker 后端启用 `agents.defaults.sandbox` 后，Gateway 网关会在隔离的 Docker 容器中运行智能体工具（shell、文件读写等），而 Gateway 网关本身仍留在主机上——无需将整个 Gateway 网关容器化，即可为不受信任或多租户智能体会话建立一道坚固的隔离墙。

沙箱范围可以是按智能体（默认）、按会话或共享；每个范围都有自己的工作区，并挂载到 `/workspace`。还可以配置工具允许/拒绝策略、网络隔离、资源限制和浏览器容器。

有关完整配置、镜像、安全注意事项和多智能体配置文件，请参阅：

- [沙箱隔离](/zh-CN/gateway/sandboxing) -- 完整的沙箱参考
- [OpenShell](/zh-CN/gateway/openshell) -- 以交互方式访问沙箱容器的 shell
- [多 Agent 沙盒和工具](/zh-CN/tools/multi-agent-sandbox-tools) -- 按智能体覆盖

### 快速启用

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // 关闭 | 非主会话 | 全部
        scope: "agent", // 会话 | 智能体 | 共享
      },
    },
  },
}
```

构建默认沙箱镜像（从源代码检出目录）：

```bash
scripts/sandbox-setup.sh
```

对于没有源代码检出目录的 npm 安装，请参见[沙箱隔离 § 镜像和设置](/zh-CN/gateway/sandboxing#images-and-setup)，了解内联 `docker build` 命令。

## 故障排查

<AccordionGroup>
  <Accordion title="镜像缺失或沙箱容器无法启动">
    使用 [`scripts/sandbox-setup.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/sandbox-setup.sh)（源代码检出目录）或[沙箱隔离 § 镜像和设置](/zh-CN/gateway/sandboxing#images-and-setup)中的内联 `docker build` 命令（npm 安装）构建沙箱镜像，或者将 `agents.defaults.sandbox.docker.image` 设置为你的自定义镜像。系统会根据需要为每个会话自动创建容器。
  </Accordion>

  <Accordion title="沙箱中的权限错误">
    将 `docker.user` 设置为与所挂载工作区所有权匹配的 UID:GID，或者更改工作区文件夹的所有者。
  </Accordion>

  <Accordion title="在沙箱中找不到自定义工具">
    OpenClaw 使用 `sh -lc`（登录 shell）运行命令，它会加载 `/etc/profile`，并可能重置 PATH。设置 `docker.env.PATH` 以将自定义工具路径添加到 PATH 开头，或者在 Dockerfile 的 `/etc/profile.d/` 下添加脚本。
  </Accordion>

  <Accordion title="镜像构建期间因 OOM 被终止（退出代码 137）">
    VM 至少需要 2 GB RAM。请使用规格更高的机器并重试。
  </Accordion>

  <Accordion title="Control UI 中显示未授权或需要配对">
    获取新的仪表板链接并批准浏览器设备：

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    docker compose run --rm openclaw-cli devices list
    docker compose run --rm openclaw-cli devices approve <requestId>
    ```

    详情参见：[仪表板](/zh-CN/web/dashboard)、[设备](/zh-CN/cli/devices)。

  </Accordion>

  <Accordion title="Gateway 网关目标显示 ws://172.x.x.x，或 Docker CLI 出现配对错误">
    重置 Gateway 网关模式和绑定：

    ```bash
    docker compose run --rm openclaw-cli config set --batch-json '[{"path":"gateway.mode","value":"local"},{"path":"gateway.bind","value":"lan"}]'
    docker compose run --rm openclaw-cli devices list --url ws://127.0.0.1:18789
    ```

  </Accordion>
</AccordionGroup>

## 相关内容

- [安装概览](/zh-CN/install) — 所有安装方法
- [Podman](/zh-CN/install/podman) — Docker 的 Podman 替代方案
- [ClawDock](/zh-CN/install/clawdock) — Docker Compose 社区设置
- [更新](/zh-CN/install/updating) — 让 OpenClaw 保持最新
- [配置](/zh-CN/gateway/configuration) — 安装后的 Gateway 网关配置
