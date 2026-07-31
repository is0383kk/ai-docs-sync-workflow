---
permalink: /security/formal-verification/
read_when:
    - 审查正式安全模型的保证或限制
    - 复现或更新 TLA+/TLC 安全模型检查
summary: 针对 OpenClaw 最高风险路径的机器验证安全模型。
title: 形式化验证（安全模型）
x-i18n:
    generated_at: "2026-07-26T06:23:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 185ee5c1cff7325f10827330c0c7e55ddc3ca40caf6088d4c930ae5e090d6b27
    source_path: security/formal-verification.md
    workflow: 16
---

OpenClaw 的形式化安全模型（目前使用 TLA+/TLC）通过机器检查论证了在明确陈述的假设下，特定的最高风险路径——授权、会话隔离、工具门控和错误配置安全——会执行其预期策略。

> 注意：某些较旧的链接可能会使用此前的项目名称。

## 这是什么

一套可执行、由攻击者驱动的安全回归测试套件：

- 每项声明都有一个可在有限状态空间中运行的模型检查。
- 许多声明还配有对应的负向模型，可针对现实中的缺陷类别生成反例轨迹。

这**并不**证明 OpenClaw 在所有方面都是安全的，也不会验证完整的 TypeScript 实现。

## 模型的位置

模型在单独的仓库中维护：[vignesh07/openclaw-formal-models](https://github.com/vignesh07/openclaw-formal-models)。

<Note>
该仓库目前无法访问（截至本文撰写时，GitHub 返回 “Repository not found”）。如果你仍无法访问，请先在 OpenClaw 维护者渠道中询问当前位置，不要直接认定这些模型已被移除。
</Note>

## 注意事项

- 这些是模型，而不是完整的 TypeScript 实现——模型与代码之间可能存在偏差。
- 结果受 TLC 探索的状态空间限制。检查通过并不意味着在建模的假设和边界之外也是安全的。
- 部分声明依赖明确的环境假设（例如正确部署以及正确的配置输入）。

## 复现结果

克隆模型仓库并运行 TLC：

```bash
git clone https://github.com/vignesh07/openclaw-formal-models
cd openclaw-formal-models

# 需要 Java 11+（TLC 在 JVM 上运行）。
# 该仓库内置固定版本的 tla2tools.jar，并提供 bin/tlc 和 Make 目标。

make <target>
```

目前尚未集成到此仓库的 CI 中；后续迭代可以添加由 CI 运行的模型并提供公开工件（反例轨迹、运行日志），或者为小型有界检查提供托管的“运行此模型”工作流。

## 声明和目标

### Gateway 网关暴露和开放式 Gateway 网关错误配置

**声明：**根据模型的假设，在没有身份验证的情况下绑定到 loopback 之外的地址可能导致远程入侵，并扩大暴露面；令牌或密码可以阻止未经身份验证的攻击者。

| 结果           | 目标                                                             |
| -------------- | ---------------------------------------------------------------- |
| 通过           | `make gateway-exposure-v2`, `make gateway-exposure-v2-protected` |
| 失败（预期）   | `make gateway-exposure-v2-negative`                              |

另请参阅模型仓库中的 `docs/gateway-exposure-matrix.md`。

### 节点 Exec 流水线（最高风险能力）

**声明：**在模型中，`exec host=node` 要求：(a) 节点命令允许列表以及已声明的命令；(b) 配置后需要实时审批；审批使用令牌化机制以防止重放。

| 结果           | 目标                                                            |
| -------------- | --------------------------------------------------------------- |
| 通过           | `make nodes-pipeline`, `make approvals-token`                   |
| 失败（预期）   | `make nodes-pipeline-negative`, `make approvals-token-negative` |

### 配对存储（私信门控）

**声明：**配对请求遵守 TTL 和待处理请求数量上限。

| 结果           | 目标                                                 |
| -------------- | ---------------------------------------------------- |
| 通过           | `make pairing`, `make pairing-cap`                   |
| 失败（预期）   | `make pairing-negative`, `make pairing-cap-negative` |

### 入口门控（提及和控制命令绕过）

**声明：**在要求提及的群组上下文中，未经授权的控制命令无法绕过提及门控。

| 结果           | 目标                           |
| -------------- | ------------------------------ |
| 通过           | `make ingress-gating`          |
| 失败（预期）   | `make ingress-gating-negative` |

### 路由和会话键隔离

**声明：**来自不同对端的私信不会合并到同一会话中，除非它们被明确关联或进行了相应配置。

| 结果           | 目标                              |
| -------------- | --------------------------------- |
| 通过           | `make routing-isolation`          |
| 失败（预期）   | `make routing-isolation-negative` |

## v1++ 模型：并发、重试和轨迹正确性

后续模型进一步提高了对现实故障模式的还原度：非原子更新、重试和消息扇出。

### 配对存储的并发与幂等性

**声明：**即使操作交错执行，配对存储也会强制执行 `MaxPending` 和幂等性——先检查再写入的过程必须是原子的或受锁保护，并且刷新不得产生重复项。具体而言：并发请求不能超过某个渠道的 `MaxPending`，对同一个 `(channel, sender)` 的重复请求或刷新不会创建重复的有效待处理行。

| 结果           | 目标                                                                                                                                                                        |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 通过           | `make pairing-race`（原子或加锁的上限检查）、`make pairing-idempotency`、`make pairing-refresh`、`make pairing-refresh-race`                                              |
| 失败（预期）   | `make pairing-race-negative`（非原子的开始/提交上限竞争）、`make pairing-idempotency-negative`、`make pairing-refresh-negative`、`make pairing-refresh-race-negative` |

### 入口轨迹关联和幂等性

**声明：**摄取过程会在扇出期间保留轨迹关联，并且在提供商重试时具有幂等性。当一个外部事件转换为多条内部消息时，每个部分都保留相同的轨迹/事件身份；重试不会导致重复处理；如果缺少提供商事件 ID，去重机制会回退到安全键（例如轨迹 ID），避免丢弃不同的事件。

| 结果           | 目标                                                                                                                                        |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 通过           | `make ingress-trace`, `make ingress-trace2`, `make ingress-idempotency`, `make ingress-dedupe-fallback`                                     |
| 失败（预期）   | `make ingress-trace-negative`, `make ingress-trace2-negative`, `make ingress-idempotency-negative`, `make ingress-dedupe-fallback-negative` |

### 路由 dmScope 优先级和 identityLinks

**声明：**`dmScope` 的优先级和身份关联以确定性方式工作：默认的 `main` 作用域会在单个所有者的私信之间共享一个滚动会话（个人智能体的默认设置），而任何已配置的隔离作用域（`per-peer`、`per-channel-peer`、`per-account-channel-peer`）都会严格分隔私信会话。渠道特定的 `dmScope` 覆盖项优先于全局默认值；`identityLinks` 仅在明确关联的组内合并会话，不会跨无关对端合并。多用户收件箱应选择隔离作用域（运行时安全审计检测到多用户私信流量时会提出此建议）。

| 结果           | 目标                                                                      |
| -------------- | ------------------------------------------------------------------------- |
| 通过           | `make routing-precedence`, `make routing-identitylinks`                   |
| 失败（预期）   | `make routing-precedence-negative`, `make routing-identitylinks-negative` |

## 相关内容

- [威胁模型](/zh-CN/security/THREAT-MODEL-ATLAS)
- [参与贡献威胁模型](/zh-CN/security/CONTRIBUTING-THREAT-MODEL)
- [事件响应](/zh-CN/security/incident-response)
