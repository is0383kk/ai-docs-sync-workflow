---
read_when:
    - 在身份感知代理后运行 OpenClaw
    - 在 OpenClaw 前端设置带 OAuth 的 Pomerium、Caddy 或 nginx
    - 修复反向代理设置中的 WebSocket 1008 未授权错误
    - 确定在何处设置 HSTS 和其他 HTTP 安全强化标头
sidebarTitle: Trusted proxy auth
summary: 将 Gateway 网关身份验证委托给受信任的反向代理（Pomerium、Caddy、nginx + OAuth）
title: 可信代理身份验证
x-i18n:
    generated_at: "2026-07-26T06:10:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 39bf8f12b3ae95f53b21bfed12deb1c8ed8f767711955bbee52c74538052a89f
    source_path: gateway/trusted-proxy-auth.md
    workflow: 16
---

<Warning>
**安全敏感功能。** 此模式将身份验证完全委托给反向代理。配置错误可能会使 Gateway 网关遭到未经授权的访问。启用前请仔细阅读本页。
</Warning>

## 何时使用

- 你在**身份感知代理**（Pomerium、Caddy + OAuth、nginx + oauth2-proxy、Traefik + forward auth）后运行 OpenClaw。
- 你的代理处理所有身份验证，并通过请求头传递用户身份。
- 你使用 Kubernetes 或容器环境，且代理是访问 Gateway 网关的唯一路径。
- 由于浏览器无法在 WS 载荷中传递令牌，你遇到了 WebSocket `1008 unauthorized` 错误。

## 不应使用的情况

- 你的代理不验证用户身份（仅作为 TLS 终止器或负载均衡器）。
- 存在任何绕过代理访问 Gateway 网关的路径（防火墙漏洞、内部网络访问）。
- 你不确定代理是否正确移除或覆盖转发请求头。
- 你只需要个人单用户访问（可考虑改用 Tailscale Serve + loopback）。

## 工作原理

<Steps>
  <Step title="代理验证用户身份">
    反向代理验证用户身份（OAuth、OIDC、SAML 等）。
  </Step>
  <Step title="代理添加身份请求头">
    代理添加一个包含已验证用户身份的请求头（例如 `x-forwarded-user: nick@example.com`）。
  </Step>
  <Step title="Gateway 网关验证可信来源">
    OpenClaw 检查请求是否来自**可信代理 IP**（`gateway.trustedProxies`），并确认其不是 Gateway 网关自身的 loopback 或本地接口地址。
  </Step>
  <Step title="Gateway 网关提取身份">
    OpenClaw 读取必需请求头，然后从配置的请求头中读取用户身份。
  </Step>
  <Step title="授权">
    如果所有检查均通过，且用户通过 `allowUsers` 检查（如已设置），则授权该请求。
  </Step>
</Steps>

## 配置

```json5
{
  gateway: {
    // 可信代理身份验证默认要求代理的源 IP 不是 loopback
    bind: "lan",

    // 关键：此处仅添加代理的 IP
    trustedProxies: ["10.0.0.1", "172.17.0.1"],

    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        // 包含已验证用户身份的请求头（必需）
        userHeader: "x-forwarded-user",

        // 可选：必须存在的请求头（代理验证）
        requiredHeaders: ["x-forwarded-proto", "x-forwarded-host"],

        // 可选：仅限特定用户（空 = 允许所有用户）
        allowUsers: ["nick@example.com", "admin@company.org"],

        // 可选：显式选择启用后，允许同一主机上的 loopback 代理
        allowLoopback: false,

        // 可选：允许通过代理验证身份的用户注册新的浏览器设备
        deviceAutoApprove: {
          enabled: false,
          scopes: ["operator.read", "operator.write", "operator.approvals"],
        },
      },
    },
  },
}
```

<Warning>
**运行时规则（按评估顺序）**

1. 请求的源 IP 必须匹配 `gateway.trustedProxies`（支持 CIDR），否则将被拒绝（`trusted_proxy_untrusted_source`）。
2. 除非启用了 `gateway.auth.trustedProxy.allowLoopback = true`，并且 loopback 地址也在 `trustedProxies` 中（`trusted_proxy_loopback_source`），否则会拒绝来自 loopback 的请求（`127.0.0.1`、`::1`）。此检查先于请求头检查执行，因此即使必需请求头也缺失，loopback 来源仍会以此方式失败。
3. 作为防欺骗措施，如果非 loopback 来源匹配 Gateway 网关主机自身的某个本地网络接口地址，则会拒绝该请求（`trusted_proxy_local_interface_source`）。如果接口发现过程本身失败，也会拒绝该请求（`trusted_proxy_local_interface_check_failed`）。
4. `requiredHeaders` 和 `userHeader` 必须存在且不能为空白。
5. 如果 `allowUsers` 非空，则其中必须包含提取出的用户。

**对于本地直连回退，转发请求头证据优先于 loopback 本地性。** 如果请求从 loopback 到达，但携带 `Forwarded`、任意 `X-Forwarded-*` 或 `X-Real-IP` 请求头，则这些证据会使其不符合本地直连密码回退和设备身份门控的条件，即使它仍会因为来自 loopback 而无法通过可信代理身份验证。

`allowLoopback` 对 Gateway 网关主机上的本地进程给予与反向代理相同程度的信任。仅当 Gateway 网关仍通过防火墙阻止远程直接访问，并且本地代理会移除或覆盖客户端提供的身份请求头时，才应启用此选项。

不经过反向代理的内部 Gateway 网关客户端应使用 `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`，而不是可信代理身份请求头。非 loopback 的 Control UI 部署仍需显式配置 `gateway.controlUi.allowedOrigins`。
</Warning>

### 配置参考

<ParamField path="gateway.trustedProxies" type="string[]" required>
  要信任的代理 IP 地址（或 CIDR）数组。来自其他 IP 的请求将被拒绝。
</ParamField>
<ParamField path="gateway.auth.mode" type="string" required>
  必须为 `"trusted-proxy"`。
</ParamField>
<ParamField path="gateway.auth.trustedProxy.userHeader" type="string" required>
  包含已验证用户身份的请求头名称。
</ParamField>
<ParamField path="gateway.auth.trustedProxy.requiredHeaders" type="string[]">
  要信任请求必须存在的其他请求头。
</ParamField>
<ParamField path="gateway.auth.trustedProxy.allowUsers" type="string[]">
  用户身份允许列表。为空表示允许所有已验证身份的用户。
</ParamField>
<ParamField path="gateway.auth.trustedProxy.allowLoopback" type="boolean" default="false">
  选择启用对同一主机上 loopback 反向代理的支持。
</ParamField>
<ParamField path="gateway.auth.trustedProxy.deviceAutoApprove.enabled" type="boolean" default="false">
  在可信代理身份验证后，自动批准新的 Control UI 和 WebChat 设备身份。
</ParamField>
<ParamField path="gateway.auth.trustedProxy.deviceAutoApprove.scopes" type="string[]" default='["operator.read", "operator.write", "operator.approvals"]'>
  授予自动批准的浏览器设备的最大权限范围。显式列出 `operator.admin` 后，每个通过代理验证身份的用户都可以请求自动授予设备完整管理员权限；未指定权限范围的请求会自动获得完整管理员权限；同时还会触发严重级别的 `gateway.trusted_proxy_device_auto_approve_admin` 安全审计发现和 Gateway 网关启动警告。
</ParamField>

<Warning>
仅当本地反向代理是预期的信任边界时，才启用 `allowLoopback`。任何能连接 Gateway 网关的本地进程都可以尝试发送代理身份请求头，因此请确保只能从主机内部直接访问 Gateway 网关，并要求使用由代理控制的请求头（例如 `x-forwarded-proto`），或在代理支持的情况下使用签名断言请求头。
</Warning>

## 自动批准设备

可信代理身份验证可以选择将代理身份用作新浏览器设备的批准边界：

```json5
{
  gateway: {
    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        userHeader: "x-forwarded-user",
        allowUsers: ["operator@example.com"],
        deviceAutoApprove: {
          enabled: true,
          scopes: ["operator.read", "operator.write", "operator.approvals"],
        },
      },
    },
  },
}
```

默认值为 `enabled: false`。启用后，以下所有规则均适用：

1. WebSocket 必须通过 `trusted-proxy` 方法完成身份验证，具有非空用户身份，并且在配置了允许列表时通过 `allowUsers` 检查。令牌、密码、Tailscale 和未经身份验证的连接绝不会使用此策略。
2. 只能自动批准新的 Control UI 或 WebChat 浏览器设备。对现有设备的任何请求（包括权限范围升级）仍会保持待处理状态，需使用 `openclaw devices approve <requestId>` 手动批准。
3. 设备以角色 `operator` 获得批准。如果连接请求包含权限范围，授予的权限将是所请求权限范围与 `deviceAutoApprove.scopes` 的精确交集。如果请求省略权限范围，则授予配置的列表；省略该列表时，默认为 `operator.read`、`operator.write` 和 `operator.approvals`。如果连接中存在 [`x-openclaw-scopes`](#control-ui-pairing-behavior) 代理请求头，最终授权还会受其进一步限制，因此代理缩小用户权限范围时，不仅会限制会话，还会限制**持久化**设备授权；如果请求头存在但值为空，则不会授予任何权限范围。即使客户端省略了自己的权限范围列表，此限制也仍然适用。
4. 仅当在 `deviceAutoApprove.scopes` 中显式列出时，才允许 `operator.admin`。列出后，每个通过代理验证身份的用户都可以请求并自动获得新浏览器设备的完整管理员权限；未指定权限范围的请求会自动获得完整管理员权限。`openclaw security audit` 会报告严重级别的 `gateway.trusted_proxy_device_auto_approve_admin` 发现，Gateway 网关还会在启动时记录一次警告。在按身份分配角色功能可用之前，建议通过 `openclaw devices approve` 或 `openclaw devices rotate` 手动批准管理员权限。

<Warning>
启用此选项会将新浏览器设备注册完全委托给反向代理身份。遭到入侵的代理账户可以注册一个具有所有已配置权限范围的持久化设备。列出 `operator.admin` 会使该设备无需手动批准即可成为完整管理员。确保只能通过代理访问 Gateway 网关，要求代理使用强身份验证，覆盖身份请求头，并使用范围较窄的 `allowUsers` 列表。
</Warning>

## Control UI 配对行为

当 `gateway.auth.mode = "trusted-proxy"` 处于启用状态且请求通过可信代理检查时，Control UI WebSocket 会话无需设备配对身份即可连接。

权限范围影响：

- 无设备身份的 Control UI WebSocket 会话可以连接，但默认不会获得任何操作员权限范围。OpenClaw 会将请求的权限范围列表清空为 `[]`，从而防止未绑定到已批准配对设备或令牌的会话自行声明权限。
- 如果 WebSocket 成功连接后，方法调用因 `missing scope` 而失败，请使用 HTTPS，以便浏览器生成设备身份并完成配对。请参阅 [Control UI 不安全的 HTTP](/zh-CN/web/control-ui#insecure-http)。
- 仍包含已停用的
  `gateway.controlUi.dangerouslyDisableDeviceAuth=true` 键的旧配置会使用受限的
  [Control UI 升级迁移](/zh-CN/web/control-ui#device-pairing-first-connection)。

反向代理权限范围上限：如果代理在 Control UI WebSocket 升级请求中发送 `x-openclaw-scopes`，OpenClaw 会将会话权限范围限制为请求的权限范围与声明的权限范围的交集。此请求头不会授予权限范围；它只会缩小会话可持有的权限范围。当 `deviceAutoApprove.enabled` 为 true 时，同一上限也适用于由[自动批准设备](#automatic-device-approval)写入的持久化设备授权，因此自动批准的设备绝不会持有超过代理声明范围的权限。

影响：

- 配对不再是无设备身份的 Control UI 访问的主要门控。当 `deviceAutoApprove.enabled` 为 true 时，代理身份也会成为新浏览器设备注册的批准门控。
- 你的反向代理身份验证策略和 `allowUsers` 将成为实际的访问控制机制。
- 确保 Gateway 网关入口仅限可信代理 IP（`gateway.trustedProxies` + 防火墙）。

自定义 WebSocket 客户端不是 Control UI 会话。已停用的 Control UI
升级输入不会向任意
`client.mode: "backend"` 或 CLI 形式的客户端授予临时访问权限。自定义自动化应使用
设备身份/配对、预留的本地直连 `client.id: "gateway-client"`
后端辅助路径，或在 HTTP 请求/响应接口更合适时使用 [admin HTTP RPC 插件](/zh-CN/plugins/admin-http-rpc)。

## 操作员权限范围请求头

可信代理身份验证是一种**携带身份信息**的 HTTP 模式，因此调用方可以选择在 HTTP API 请求中通过 `x-openclaw-scopes` 声明操作员权限范围。

注意：WebSocket 权限范围由 Gateway 网关协议握手和设备身份绑定决定。在 Control UI WebSocket 升级请求中，`x-openclaw-scopes` 只是协商所得会话权限范围的上限，并不会授予权限。请参阅 [Control UI 配对行为](#control-ui-pairing-behavior)。

示例：

- `x-openclaw-scopes: operator.read`
- `x-openclaw-scopes: operator.read,operator.write`
- `x-openclaw-scopes: operator.admin,operator.write`

行为：

- 存在该请求头时，OpenClaw 会采用所声明的权限范围集合。
- 存在该请求头但其值为空时，请求声明**不具有任何**操作员权限范围。
- 缺少该请求头时，常规的携带身份信息 HTTP API 会回退到标准操作员默认权限范围集合（`operator.admin`、`operator.read`、`operator.write`、`operator.approvals`、`operator.pairing`、`operator.talk.secrets`）。
- Gateway 网关身份验证的**插件 HTTP 路由**默认权限范围更窄：缺少 `x-openclaw-scopes` 时，其运行时权限范围仅回退到 `operator.write`。
- 即使可信代理身份验证成功，来自浏览器的 HTTP 请求仍必须通过 `gateway.controlUi.allowedOrigins`（或有意启用的 Host 请求头回退模式）。

实用规则：当你希望可信代理请求的权限范围比默认值更窄，或者 Gateway 网关身份验证插件路由需要比写入权限更强的权限时，请显式发送 `x-openclaw-scopes`。

## TLS 终止和 HSTS

仅使用一个 TLS 终止点，并在该处应用 HSTS。

<Tabs>
  <Tab title="代理 TLS 终止（推荐）">
    当反向代理为 `https://control.example.com` 处理 HTTPS 时，请在代理上为该域名设置 `Strict-Transport-Security`。

    - 非常适合面向互联网的部署。
    - 将证书和 HTTP 安全强化策略集中在一处。
    - OpenClaw 可以在代理后继续使用环回 HTTP。

    请求头值示例：

    ```text
    Strict-Transport-Security: max-age=31536000; includeSubDomains
    ```

  </Tab>
  <Tab title="Gateway 网关 TLS 终止">
    如果 OpenClaw 自身直接提供 HTTPS（没有执行 TLS 终止的代理），请设置：

    ```json5
    {
      gateway: {
        tls: { enabled: true },
        http: {
          securityHeaders: {
            strictTransportSecurity: "max-age=31536000; includeSubDomains",
          },
        },
      },
    }
    ```

    `strictTransportSecurity` 接受字符串形式的请求头值，也可设置为 `false` 以显式禁用。

  </Tab>
</Tabs>

### 推出指南

- 验证流量时，首先使用较短的最大有效期（例如 `max-age=300`）。
- 仅在信心充足后，才增加为长期有效值（例如 `max-age=31536000`）。
- 仅当所有子域名均已支持 HTTPS 时，才添加 `includeSubDomains`。
- 仅当你有意满足完整域名集合的预加载要求时，才使用预加载。
- 仅限环回的本地开发无法从 HSTS 中受益。

## 代理设置示例

<AccordionGroup>
  <Accordion title="Pomerium">
    Pomerium 通过 `x-pomerium-claim-email`（或其他声明请求头）传递身份，并通过 `x-pomerium-jwt-assertion` 传递 JWT。

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // Pomerium 的 IP
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-pomerium-claim-email",
            requiredHeaders: ["x-pomerium-jwt-assertion"],
          },
        },
      },
    }
    ```

    Pomerium 配置片段：

    ```yaml
    routes:
      - from: https://openclaw.example.com
        to: http://openclaw-gateway:18789
        policy:
          - allow:
              or:
                - email:
                    is: nick@example.com
        pass_identity_headers: true
    ```

  </Accordion>
  <Accordion title="使用 OAuth 的 Caddy">
    安装 `caddy-security` 插件的 Caddy 可以对用户进行身份验证并传递身份请求头。

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // Caddy/边车代理 IP
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-forwarded-user",
          },
        },
      },
    }
    ```

    Caddyfile 片段：

    ```caddy
    openclaw.example.com {
        authenticate with oauth2_provider
        authorize with policy1

        reverse_proxy openclaw:18789 {
            header_up X-Forwarded-User {http.auth.user.email}
        }
    }
    ```

  </Accordion>
  <Accordion title="nginx + oauth2-proxy">
    oauth2-proxy 对用户进行身份验证，并通过 `x-auth-request-email` 传递身份。

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // nginx/oauth2-proxy IP
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-auth-request-email",
          },
        },
      },
    }
    ```

    nginx 配置片段：

    ```nginx
    location / {
        auth_request /oauth2/auth;
        auth_request_set $user $upstream_http_x_auth_request_email;

        proxy_pass http://openclaw:18789;
        proxy_set_header X-Auth-Request-Email $user;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    ```

  </Accordion>
  <Accordion title="使用转发身份验证的 Traefik">
    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["172.17.0.1"], // Traefik 容器 IP
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-forwarded-user",
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## 混合令牌配置

如果同时配置了共享令牌（`gateway.auth.token` 或 `OPENCLAW_GATEWAY_TOKEN`），Gateway 网关启动时会拒绝可信代理身份验证。两者互斥，因为共享令牌会让同一主机上的调用方通过一条与此模式旨在强制执行的代理验证身份完全不同的路径进行身份验证。

如果启动失败并出现类似 `gateway auth mode is trusted-proxy, but a shared token is also configured` 的错误：

- 使用可信代理模式时移除共享令牌，或者
- 如果你打算使用基于令牌的身份验证，请将 `gateway.auth.mode` 切换为 `"token"`。

环回可信代理身份请求头仍会以失败关闭方式处理：同一主机上的调用方不会被静默认证为代理用户。绕过代理的 OpenClaw 内部调用方可以改用 `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` 进行身份验证。在可信代理模式下，令牌回退仍被有意设为不支持。

## 安全检查清单

启用可信代理身份验证之前，请验证：

- [ ] **代理是唯一路径**：通过防火墙阻止除代理以外的所有对象访问 Gateway 网关端口。
- [ ] **trustedProxies 保持最小范围**：仅包含实际代理 IP，而不是整个子网。
- [ ] **环回代理来源经过有意配置**：除非为同一主机上的代理显式启用 `gateway.auth.trustedProxy.allowLoopback`，否则来自环回源的请求会导致可信代理身份验证失败关闭。
- [ ] **代理会移除请求头**：代理会覆盖（而不是追加）客户端提供的 `x-forwarded-*` 请求头。
- [ ] **TLS 终止**：代理负责处理 TLS；用户通过 HTTPS 连接。
- [ ] **allowedOrigins 已显式设置**：非环回 Control UI 使用显式的 `gateway.controlUi.allowedOrigins`。
- [ ] **已设置 allowUsers**（推荐）：限制为已知用户，而不是允许任何已通过身份验证的用户。
- [ ] **没有混合令牌配置**：不要同时设置 `gateway.auth.token` 和 `gateway.auth.mode: "trusted-proxy"`。
- [ ] **本地密码回退保持私有**：如果为内部直接调用方配置 `gateway.auth.password`，请通过防火墙保护 Gateway 网关端口，确保非代理远程客户端无法直接访问。
- [ ] **设备自动批准经过有意配置**：如果 `deviceAutoApprove.enabled` 为 true，请将反向代理账户安全性视为设备注册边界，并确保授予的权限范围列表不包含管理员权限且保持最小范围。

## 安全审计

`openclaw security audit` 会以**严重**级别标记可信代理身份验证。这是有意设计的；它提醒你正在将安全性委托给代理设置。

审计会检查：

- 基础 `gateway.trusted_proxy_auth` 警告/严重提醒。
- 缺少 `trustedProxies` 配置。
- 缺少 `userHeader` 配置。
- `allowUsers` 为空（允许任何已通过身份验证的用户）。
- 为同一主机上的代理来源启用了 `allowLoopback`。
- 启用了浏览器设备自动批准（将新设备配对委托给代理身份）。

只要 Control UI 对外公开，其他与可信代理无关的独立发现也同样适用：`gateway.controlUi.allowedOrigins` 使用通配符或缺失，以及 Host 请求头来源回退。

## 故障排查

<AccordionGroup>
  <Accordion title="trusted_proxy_untrusted_source">
    请求并非来自 `gateway.trustedProxies` 中的 IP。请检查：

    - 代理 IP 是否正确？（Docker 容器 IP 可能会变化。）
    - 代理前方是否存在负载均衡器？
    - 使用 `docker inspect` 或 `kubectl get pods -o wide` 查找实际 IP。

  </Accordion>
  <Accordion title="trusted_proxy_loopback_source">
    OpenClaw 拒绝了来自环回源的可信代理请求。

    请检查：

    - 代理是否从 `127.0.0.1` / `::1` 连接？
    - 你是否尝试通过同一主机上的环回反向代理使用可信代理身份验证？

    修复方法：

    - 对于不经过代理的同一主机内部客户端，优先使用令牌/密码身份验证，或者
    - 通过非环回的可信代理地址进行路由，并将该 IP 保留在 `gateway.trustedProxies` 中，或者
    - 对于有意配置的同一主机反向代理，请设置 `gateway.auth.trustedProxy.allowLoopback = true`，将环回地址保留在 `gateway.trustedProxies` 中，并确保代理移除或覆盖身份请求头。

  </Accordion>
  <Accordion title="trusted_proxy_local_interface_source / trusted_proxy_local_interface_check_failed">
    请求的源 IP 与 Gateway 网关主机自身的某个非环回网络接口地址（而非代理）匹配；这是一项防护措施，用于阻止 tailnet 或 Docker 桥接网络中同一主机流量的身份伪造。`..._check_failed` 表示接口发现本身发生错误，因此 OpenClaw 会失败关闭。

    请检查：

    - Gateway 网关主机自身的某个进程是否绕过代理，直接发送身份请求头？
    - 代理是否与 Gateway 网关运行在同一网络命名空间中，且其 IP 也显示为本地接口？

    修复方法：通过未同时绑定到 Gateway 网关主机本地的地址路由代理流量；仅在确实为同一主机代理设置时使用 `allowLoopback`。

  </Accordion>
  <Accordion title="trusted_proxy_user_missing">
    用户请求头为空或缺失。请检查：

    - 代理是否已配置为传递身份请求头？
    - 请求头名称是否正确？（不区分大小写，但拼写必须正确）
    - 用户是否确实已在代理处通过身份验证？

  </Accordion>
  <Accordion title="trusted_proxy_missing_header_*">
    缺少必需的请求头。请检查：

    - 代理中针对这些特定请求头的配置。
    - 请求头是否在链路中的某处被移除。

  </Accordion>
  <Accordion title="trusted_proxy_user_not_allowed">
    用户已通过身份验证，但不在 `allowUsers` 中。请将其添加到允许列表，或移除该允许列表。
  </Accordion>
  <Accordion title="trusted_proxy_no_proxies_configured / trusted_proxy_config_missing">
    `gateway.auth.mode` 为 `"trusted-proxy"`，但 `gateway.trustedProxies` 为空，或 `gateway.auth.trustedProxy` 本身缺失。在两者均设置完成之前，所有请求都会被拒绝。
  </Accordion>
  <Accordion title="trusted_proxy_origin_not_allowed">
    可信代理身份验证成功，但浏览器的 `Origin` 标头未通过 Control UI 来源检查。

    请检查：

    - `gateway.controlUi.allowedOrigins` 包含准确的浏览器来源。
    - 除非有意允许所有来源，否则不要依赖通配符来源。
    - 如果有意使用 Host 标头回退模式，请确保已明确设置 `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true`。

  </Accordion>
  <Accordion title="连接成功，但方法报告缺少权限范围">
    WebSocket 已连接，但 `chat.history`、`sessions.list` 或
    `models.list` 因 `missing scope: operator.read` 而失败。

    常见原因：

    - 无设备身份的 Control UI 会话：可信代理身份验证可以在没有设备身份的情况下允许建立 WebSocket 连接，但 OpenClaw 按设计会清除无设备身份会话的权限范围。
    - 自定义后端客户端：已停用的 Control UI 升级输入绝不会向任意后端或 CLI 形式的 WebSocket 客户端授予访问权限。
    - `x-openclaw-scopes` 过于狭窄：如果代理在 Control UI WebSocket 升级请求中注入此标头，会话权限范围将限制为该标头指定的集合。标头值为空时不会获得任何权限范围。

    修复方法：

    - 对于 Control UI，请使用 HTTPS，以便浏览器生成设备身份并完成配对。
    - 对于自定义自动化，请使用设备身份/配对、预留的直接本地 `gateway-client` 后端辅助程序路径，或[管理员 HTTP RPC](/zh-CN/plugins/admin-http-rpc)。
    - 不要将已停用的 `gateway.controlUi.dangerouslyDisableDeviceAuth` 键添加到当前配置。旧版安装会自动使用一次性自配对迁移。

  </Accordion>
  <Accordion title="WebSocket 仍然失败">
    请确保代理：

    - 支持 WebSocket 升级（`Upgrade: websocket`、`Connection: upgrade`）。
    - 在 WebSocket 升级请求中传递身份标头（而不仅是 HTTP 请求）。
    - 未对 WebSocket 连接使用单独的身份验证路径。

  </Accordion>
</AccordionGroup>

## 从令牌身份验证迁移

<Steps>
  <Step title="配置代理">
    配置代理以验证用户身份并传递标头。
  </Step>
  <Step title="独立测试代理">
    独立测试代理设置（使用带标头的 curl）。
  </Step>
  <Step title="更新 OpenClaw 配置">
    更新 OpenClaw 配置以使用可信代理身份验证。
  </Step>
  <Step title="重启 Gateway 网关">
    重启 Gateway 网关。
  </Step>
  <Step title="测试 WebSocket">
    从 Control UI 测试 WebSocket 连接。
  </Step>
  <Step title="审计">
    运行 `openclaw security audit` 并审查发现的问题。
  </Step>
</Steps>

## 相关内容

- [配置](/zh-CN/gateway/configuration) — 配置参考
- [操作员权限范围](/zh-CN/gateway/operator-scopes) — 角色、权限范围和审批检查
- [远程访问](/zh-CN/gateway/remote) — 其他远程访问模式
- [安全性](/zh-CN/gateway/security) — 完整的安全指南
- [Tailscale](/zh-CN/gateway/tailscale) — 仅限 tailnet 访问的更简单替代方案
