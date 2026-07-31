---
read_when:
    - 在 EasyRunner 上部署 OpenClaw
    - 在 EasyRunner 的 Caddy 代理后运行 Gateway 网关
    - 为托管式 Gateway 网关选择持久卷和身份验证方式
summary: 使用 Podman 和 Caddy 在 EasyRunner 上运行 OpenClaw Gateway 网关
title: EasyRunner
x-i18n:
    generated_at: "2026-07-26T06:50:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80cbde016a8bf7662d4b4a056a3d122a423264179daf70b5705e8f10b0dad5cb
    source_path: platforms/easyrunner.md
    workflow: 16
---

EasyRunner 将 OpenClaw Gateway 网关作为一个小型容器化应用托管在其
Caddy 代理后。本指南假设 EasyRunner 主机运行兼容 Podman 的
Compose 应用，并通过 Caddy 终止 HTTPS。

## 开始之前

- 一台已将域名路由至其上的 EasyRunner 服务器。
- 官方 OpenClaw 镜像（`ghcr.io/openclaw/openclaw`）或你自行构建的镜像。
- 用于 `/home/node/.openclaw` 的持久化配置卷。
- 用于 `/home/node/.openclaw/workspace` 的持久化工作区卷。
- 高强度的 Gateway 网关令牌或密码。

尽可能保持设备身份验证处于启用状态。如果反向代理无法正确传递
设备身份，请先修复受信任代理设置（参见
[受信任代理身份验证](/zh-CN/gateway/trusted-proxy-auth)）；仅应在完全私有且由操作员控制的网络中
使用危险的身份验证绕过方式。

## Compose 应用

使用如下结构的 Compose 文件创建 EasyRunner 应用：

```yaml
services:
  openclaw:
    image: ghcr.io/openclaw/openclaw:latest
    restart: unless-stopped
    environment:
      OPENCLAW_GATEWAY_TOKEN: ${OPENCLAW_GATEWAY_TOKEN}
      OPENCLAW_HOME: /home/node
      OPENCLAW_STATE_DIR: /home/node/.openclaw
      OPENCLAW_CONFIG_PATH: /home/node/.openclaw/openclaw.json
      OPENCLAW_WORKSPACE_DIR: /home/node/.openclaw/workspace
    volumes:
      - openclaw-config:/home/node/.openclaw
      - openclaw-workspace:/home/node/.openclaw/workspace
    labels:
      caddy: openclaw.example.com
      caddy.reverse_proxy: "{{upstreams 1455}}"
    command: ["node", "openclaw.mjs", "gateway", "--bind", "lan", "--port", "1455"]

volumes:
  openclaw-config:
  openclaw-workspace:
```

将 `openclaw.example.com` 替换为你的 Gateway 网关主机名。请将
`OPENCLAW_GATEWAY_TOKEN` 存储在 EasyRunner 的密钥/环境管理器中，而不要
提交到应用定义中。镜像默认绑定到 local loopback，
因此 `command` 中显式指定的 `--bind lan --port 1455` 是 Caddy
访问该容器所必需的。

## 配置 OpenClaw

在持久化配置卷内，确保 Gateway 网关只能通过
代理访问并要求身份验证：

```json5
{
  gateway: {
    bind: "lan",
    port: 1455,
    auth: {
      token: "${OPENCLAW_GATEWAY_TOKEN}",
    },
  },
}
```

如果 Caddy 为 Gateway 网关终止 TLS，请针对
确切的代理路径配置受信任代理设置，而不要全局禁用身份验证检查。参见
[受信任代理身份验证](/zh-CN/gateway/trusted-proxy-auth)。

## 验证

在你的工作站上：

```bash
openclaw gateway probe --url https://openclaw.example.com --token <token>
openclaw gateway status --url https://openclaw.example.com --token <token>
```

在 EasyRunner 主机上，`GET /healthz`（存活性）和 `GET /readyz`
（就绪性）无需身份验证，并为镜像内置的容器健康
检查提供支持。还应检查应用日志，确认 Gateway 网关正在监听，且启动时
未出现 SecretRef、插件或渠道身份验证失败。

## 更新和备份

- 拉取或构建新的 OpenClaw 镜像，然后重新部署 EasyRunner 应用。
- 更新前备份 `openclaw-config` 卷。该卷包含
  `openclaw.json`、`agents/<agentId>/agent/auth-profiles.json` 和已安装的
  插件包状态。
- 如果智能体会在 `openclaw-workspace` 中写入持久项目数据，请备份该位置。
- 重大更新后运行 `openclaw doctor`，以发现配置迁移和
  服务警告。

## 故障排查

- `gateway probe` 无法连接：确认 Caddy 主机名指向该应用，
  并且容器正在 `0.0.0.0:1455` 上监听。
- 身份验证失败：同时轮换 EasyRunner 密钥中的令牌和本地客户端
  命令中的令牌。
- 恢复后文件归 root 所有：镜像以 `node`（uid 1000）身份运行；
  修复挂载卷的所有权和权限，使该用户能够写入
  `/home/node/.openclaw` 和 `/home/node/.openclaw/workspace`。
- 浏览器或渠道插件失败：检查所需的外部
  二进制文件、网络出口和挂载的凭据在容器内是否可用。
