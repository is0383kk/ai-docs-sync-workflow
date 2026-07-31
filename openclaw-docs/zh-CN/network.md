---
read_when:
    - 你需要网络架构和安全概览
    - 你正在调试本地访问与 tailnet 访问或配对问题
    - 你想查看网络文档的权威列表
summary: 网络中枢：Gateway 网关接口、配对、设备发现和安全性
title: 网络
x-i18n:
    generated_at: "2026-07-26T06:51:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9751bb0fe71009455b243b109ef7ef4eda08d58f940f7dcef305800a5ed89586
    source_path: network.md
    workflow: 16
---

这个中心页面链接了有关 OpenClaw 如何在 localhost、LAN 和 tailnet 中连接、配对及保护设备的核心文档。

## 核心模型

大多数操作都通过 Gateway 网关（`openclaw gateway`）进行。它是一个长期运行的单一进程，负责管理渠道连接和 WebSocket 控制平面。

- **优先使用环回地址**：Gateway 网关 WS 默认使用 `ws://127.0.0.1:18789`。
  如果没有有效的 Gateway 网关身份验证路径，非环回地址绑定将拒绝启动：
  共享密钥令牌/密码身份验证，或正确配置的非环回地址
  `trusted-proxy` 部署。
- 建议**每台主机运行一个 Gateway 网关**。如需隔离，请使用相互独立的配置文件和端口运行多个 Gateway 网关（[多个 Gateway 网关](/zh-CN/gateway/multiple-gateways)）。
- **Canvas 主机**与 Gateway 网关使用同一端口（`/__openclaw__/canvas/`、`/__openclaw__/a2ui/`）提供服务；绑定到环回地址以外的地址时，受 Gateway 网关身份验证保护。
- **远程访问**通常使用 SSH 隧道或 Tailscale VPN（[远程访问](/zh-CN/gateway/remote)）。

关键参考资料：

- [Gateway 网关架构](/zh-CN/concepts/architecture)
- [Gateway 网关协议](/zh-CN/gateway/protocol)
- [Gateway 网关运行手册](/zh-CN/gateway)
- [Web 界面 + 绑定模式](/zh-CN/web)

## 配对 + 身份

- [配对概览（私信 + 节点）](/zh-CN/channels/pairing)
- [由 Gateway 网关管理的节点配对](/zh-CN/gateway/pairing)
- [设备 CLI（配对 + 令牌轮换）](/zh-CN/cli/devices)
- [配对 CLI（私信审批）](/zh-CN/cli/pairing)

本地信任：

- 直接通过 local loopback 连接时（无转发标头/代理标头），可自动批准配对，
  以确保同一主机上的用户体验顺畅。
- OpenClaw 还提供一个严格受限的后端/容器本地自连接路径，
  供受信任的共享密钥辅助程序流程使用。
- Tailnet 和 LAN 客户端（包括同一主机上的 tailnet 绑定）仍需
  明确批准配对。

## 设备发现 + 传输协议

- [设备发现和传输协议](/zh-CN/gateway/discovery)
- [Bonjour / mDNS](/zh-CN/gateway/bonjour)
- [远程访问（SSH）](/zh-CN/gateway/remote)
- [Tailscale](/zh-CN/gateway/tailscale)

## 节点 + 传输协议

- [节点概览](/zh-CN/nodes)
- [Bridge protocol（旧版节点，历史参考）](/zh-CN/gateway/bridge-protocol)
- [节点运行手册：iOS](/zh-CN/platforms/ios)
- [节点运行手册：Android](/zh-CN/platforms/android)

## 安全

- [安全概览](/zh-CN/gateway/security)
- [Gateway 网关配置参考](/zh-CN/gateway/configuration)
- [故障排查](/zh-CN/gateway/troubleshooting)
- [Doctor](/zh-CN/gateway/doctor)

## 相关内容

- [Gateway 网关运行手册](/zh-CN/gateway)
- [远程访问](/zh-CN/gateway/remote)
