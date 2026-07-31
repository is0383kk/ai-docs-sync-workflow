---
read_when:
    - ID 認識プロキシの背後で OpenClaw を実行する
    - OpenClaw の前段に OAuth を使用して Pomerium、Caddy、または nginx を設定する
    - リバースプロキシ構成で発生する WebSocket 1008 認証エラーの修正
    - HSTS およびその他の HTTP セキュリティ強化ヘッダーを設定する場所の決定
sidebarTitle: Trusted proxy auth
summary: Gateway 認証を信頼できるリバースプロキシ（Pomerium、Caddy、nginx + OAuth）に委任する
title: 信頼済みプロキシ認証
x-i18n:
    generated_at: "2026-07-26T09:03:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 39bf8f12b3ae95f53b21bfed12deb1c8ed8f767711955bbee52c74538052a89f
    source_path: gateway/trusted-proxy-auth.md
    workflow: 16
---

<Warning>
**セキュリティ上の注意が必要な機能です。** このモードでは、認証を完全にリバースプロキシに委任します。設定を誤ると、Gateway が不正アクセスにさらされる可能性があります。有効にする前に、このページをよくお読みください。
</Warning>

## 使用する場合

- OpenClaw を **ID 認識プロキシ**（Pomerium、Caddy + OAuth、nginx + oauth2-proxy、Traefik + forward auth）の背後で実行している場合。
- プロキシがすべての認証を処理し、ユーザー ID をヘッダー経由で渡す場合。
- プロキシが Gateway への唯一の経路となる Kubernetes またはコンテナ環境を使用している場合。
- ブラウザーが WS ペイロードでトークンを渡せないため、WebSocket `1008 unauthorized` エラーが発生している場合。

## 使用しない場合

- プロキシがユーザーを認証しない場合（単なる TLS ターミネーターまたはロードバランサーの場合）。
- プロキシを迂回して Gateway に到達できる経路がある場合（ファイアウォールの穴、内部ネットワークアクセスなど）。
- プロキシが転送ヘッダーを正しく削除または上書きするか不明な場合。
- 個人用の単一ユーザーアクセスだけが必要な場合（代わりに Tailscale Serve + loopback を検討してください）。

## 仕組み

<Steps>
  <Step title="プロキシがユーザーを認証する">
    リバースプロキシがユーザーを認証します（OAuth、OIDC、SAML など）。
  </Step>
  <Step title="プロキシが ID ヘッダーを追加する">
    プロキシが認証済みユーザー ID を含むヘッダーを追加します（例: `x-forwarded-user: nick@example.com`）。
  </Step>
  <Step title="Gateway が信頼できる送信元を検証する">
    OpenClaw は、リクエストが **信頼できるプロキシ IP**（`gateway.trustedProxies`）から届いており、Gateway 自身の loopback またはローカルインターフェースアドレスではないことを確認します。
  </Step>
  <Step title="Gateway が ID を抽出する">
    OpenClaw は必須ヘッダーを読み取り、設定されたヘッダーからユーザー ID を取得します。
  </Step>
  <Step title="認可する">
    すべての確認に成功し、ユーザーが `allowUsers`（設定されている場合）を満たすと、リクエストが認可されます。
  </Step>
</Steps>

## 設定

```json5
{
  gateway: {
    // 信頼できるプロキシによる認証では、デフォルトでプロキシの送信元 IP が loopback 以外であることが求められる
    bind: "lan",

    // 重要: ここにはプロキシの IP のみを追加する
    trustedProxies: ["10.0.0.1", "172.17.0.1"],

    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        // 認証済みユーザー ID を含むヘッダー（必須）
        userHeader: "x-forwarded-user",

        // 任意: 必ず存在しなければならないヘッダー（プロキシ検証）
        requiredHeaders: ["x-forwarded-proto", "x-forwarded-host"],

        // 任意: 特定のユーザーに制限する（空 = すべて許可）
        allowUsers: ["nick@example.com", "admin@company.org"],

        // 任意: 明示的にオプトインした場合、同一ホストの loopback プロキシを許可する
        allowLoopback: false,

        // 任意: 認証済みのプロキシユーザーが新しいブラウザーデバイスを登録できるようにする
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
**評価順に並べたランタイムルール**

1. リクエストの送信元 IP は `gateway.trustedProxies`（CIDR 対応）と一致する必要があります。一致しない場合は拒否されます（`trusted_proxy_untrusted_source`）。
2. loopback 送信元からのリクエスト（`127.0.0.1`、`::1`）は、`gateway.auth.trustedProxy.allowLoopback = true` が設定され、かつ loopback アドレスも `trustedProxies` に含まれている場合（`trusted_proxy_loopback_source`）を除き、拒否されます。この確認はヘッダーの確認より先に行われるため、必須ヘッダーも欠けている場合でも、loopback 送信元はこの理由で失敗します。
3. Gateway ホスト自身のローカルネットワークインターフェースアドレスのいずれかと一致する loopback 以外の送信元は、なりすまし防止のため拒否されます（`trusted_proxy_local_interface_source`）。インターフェースの検出自体に失敗した場合も、リクエストは拒否されます（`trusted_proxy_local_interface_check_failed`）。
4. `requiredHeaders` と `userHeader` は存在し、空白以外の値でなければなりません。
5. `allowUsers` が空でない場合、抽出されたユーザーを含んでいる必要があります。

**転送ヘッダーの存在は、ローカル直接フォールバックにおける loopback のローカル性より優先されます。** リクエストが loopback に到着していても、`Forwarded`、任意の `X-Forwarded-*`、または `X-Real-IP` ヘッダーを含む場合、その存在によってローカル直接パスワードフォールバックとデバイス ID ゲーティングの対象外になります。ただし、loopback であるため、信頼できるプロキシによる認証には引き続き失敗します。

`allowLoopback` は、Gateway ホスト上のローカルプロセスをリバースプロキシと同程度に信頼します。Gateway がリモートからの直接アクセスをファイアウォールで引き続き遮断されており、ローカルプロキシがクライアント指定の ID ヘッダーを削除または上書きする場合にのみ有効にしてください。

リバースプロキシを経由しない内部 Gateway クライアントは、信頼できるプロキシの ID ヘッダーではなく、`gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` を使用する必要があります。loopback 以外の Control UI デプロイメントでは、引き続き明示的な `gateway.controlUi.allowedOrigins` が必要です。
</Warning>

### 設定リファレンス

<ParamField path="gateway.trustedProxies" type="string[]" required>
  信頼するプロキシ IP アドレス（または CIDR）の配列。他の IP からのリクエストは拒否されます。
</ParamField>
<ParamField path="gateway.auth.mode" type="string" required>
  `"trusted-proxy"` でなければなりません。
</ParamField>
<ParamField path="gateway.auth.trustedProxy.userHeader" type="string" required>
  認証済みユーザー ID を含むヘッダー名。
</ParamField>
<ParamField path="gateway.auth.trustedProxy.requiredHeaders" type="string[]">
  リクエストを信頼するために存在しなければならない追加ヘッダー。
</ParamField>
<ParamField path="gateway.auth.trustedProxy.allowUsers" type="string[]">
  ユーザー ID の許可リスト。空の場合、認証済みのすべてのユーザーを許可します。
</ParamField>
<ParamField path="gateway.auth.trustedProxy.allowLoopback" type="boolean" default="false">
  同一ホストの loopback リバースプロキシをオプトインでサポートします。
</ParamField>
<ParamField path="gateway.auth.trustedProxy.deviceAutoApprove.enabled" type="boolean" default="false">
  信頼できるプロキシによる認証後、新しい Control UI および WebChat のデバイス ID を自動的に承認します。
</ParamField>
<ParamField path="gateway.auth.trustedProxy.deviceAutoApprove.scopes" type="string[]" default='["operator.read", "operator.write", "operator.approvals"]'>
  自動承認されたブラウザーデバイスに付与される最大スコープ。`operator.admin` を明示的に指定すると、プロキシで認証されたすべてのユーザーが完全な管理者権限のデバイス付与を自動的にリクエストでき、スコープなしのリクエストにも完全な管理者権限が自動的に付与されます。また、CRITICAL の `gateway.trusted_proxy_device_auto_approve_admin` セキュリティ監査検出事項と Gateway 起動時の警告が発生します。
</ParamField>

<Warning>
ローカルリバースプロキシを意図した信頼境界とする場合にのみ、`allowLoopback` を有効にしてください。Gateway に接続できるすべてのローカルプロセスは、プロキシの ID ヘッダーを送信しようとする可能性があります。そのため、Gateway への直接アクセスはホスト内に限定し、`x-forwarded-proto` などのプロキシ所有ヘッダー、またはプロキシが対応している場合は署名付きアサーションヘッダーを必須にしてください。
</Warning>

## デバイスの自動承認

信頼できるプロキシによる認証では、必要に応じてプロキシ ID を新しいブラウザーデバイスの承認境界として使用できます。

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

デフォルトは `enabled: false` です。有効にすると、次のすべてのルールが適用されます。

1. WebSocket は、空でないユーザー ID を使用して `trusted-proxy` メソッドによる認証を完了している必要があり、許可リストが設定されている場合は、その ID が `allowUsers` を満たしていなければなりません。トークン、パスワード、Tailscale、および未認証の接続には、このポリシーは適用されません。
2. 自動承認できるのは、新しい Control UI または WebChat のブラウザーデバイスだけです。スコープのアップグレードを含め、既存デバイスに対するリクエストはすべて、`openclaw devices approve <requestId>` による手動承認を待つ状態のままになります。
3. デバイスはロール `operator` で承認されます。接続リクエストにスコープが含まれる場合、付与されるスコープは、要求されたスコープと `deviceAutoApprove.scopes` の正確な共通部分です。リクエストでスコープが省略された場合は、設定されたリストが付与されます。そのリストも省略された場合、デフォルトで `operator.read`、`operator.write`、および `operator.approvals` になります。その後、接続に [`x-openclaw-scopes`](#control-ui-pairing-behavior) プロキシヘッダーが存在する場合、結果として得られる付与内容はさらにそのヘッダーによって制限されます。そのため、プロキシがユーザーのスコープを狭めると、セッションだけでなく **永続的な** デバイス付与も制限されます。ヘッダーが存在していても空の場合、スコープは付与されません。この制限は、クライアント自身のスコープリストが省略された場合にも適用されます。
4. `operator.admin` は、`deviceAutoApprove.scopes` に明示的に指定した場合にのみ許可されます。指定すると、プロキシで認証されたすべてのユーザーが、新しいブラウザーデバイスで完全な管理者権限を要求し、自動的に取得できます。スコープのないリクエストには、完全な管理者権限が自動的に付与されます。`openclaw security audit` は CRITICAL の `gateway.trusted_proxy_device_auto_approve_admin` 検出事項を報告し、Gateway は起動時に一度警告を記録します。ID ごとのロールが利用可能になるまでは、`openclaw devices approve` または `openclaw devices rotate` を使用した管理者権限の手動承認を推奨します。

<Warning>
このオプションを有効にすると、新しいブラウザーデバイスの登録が完全にリバースプロキシの ID に委任されます。侵害されたプロキシアカウントは、設定されているすべてのスコープを持つ永続的なデバイスを登録できます。`operator.admin` を指定すると、手動承認なしでそのデバイスが完全な管理者になります。Gateway にはプロキシ経由でのみ到達できるようにし、プロキシで強力な認証を必須にし、ID ヘッダーを上書きし、`allowUsers` リストを必要最小限に絞ってください。
</Warning>

## Control UI のペアリング動作

`gateway.auth.mode = "trusted-proxy"` が有効で、リクエストが信頼できるプロキシの確認に合格すると、Control UI の WebSocket セッションはデバイスペアリング ID なしで接続できます。

スコープへの影響:

- デバイスなしの Control UI WebSocket セッションは接続できますが、デフォルトではオペレータースコープを受け取りません。OpenClaw は要求されたスコープリストを `[]` にクリアするため、承認済みのペアリング済みデバイスまたはトークンに関連付けられていないセッションが権限を自己申告することはできません。
- WebSocket の接続に成功した後、メソッドが `missing scope` で失敗する場合は、ブラウザーがデバイス ID を生成してペアリングを完了できるように HTTPS を使用してください。[Control UI の安全でない HTTP](/ja-JP/web/control-ui#insecure-http) を参照してください。
- 廃止された
  `gateway.controlUi.dangerouslyDisableDeviceAuth=true` キーがまだ含まれている古い設定には、限定的な
  [Control UI アップグレード移行](/ja-JP/web/control-ui#device-pairing-first-connection)が適用されます。

リバースプロキシによるスコープ制限: Control UI の WebSocket アップグレードリクエストでプロキシが `x-openclaw-scopes` を送信すると、OpenClaw はセッションスコープを、要求されたスコープと宣言されたスコープの共通部分に制限します。このヘッダーはスコープを付与せず、セッションが保持できるスコープを狭めるだけです。`deviceAutoApprove.enabled` が true の場合、同じ制限が[デバイスの自動承認](#automatic-device-approval)によって書き込まれる永続的なデバイス付与にも適用されるため、自動承認されたデバイスがプロキシで宣言された範囲を超えるスコープを保持することはありません。

影響:

- ペアリングは、デバイスなしの Control UI アクセスに対する主要なゲートではなくなります。`deviceAutoApprove.enabled` が true の場合、プロキシ ID は新しいブラウザーデバイス登録の承認ゲートにもなります。
- リバースプロキシの認証ポリシーと `allowUsers` が、実質的なアクセス制御になります。
- Gateway の受信経路は、信頼できるプロキシ IP のみに制限してください（`gateway.trustedProxies` + ファイアウォール）。

カスタム WebSocket クライアントは Control UI セッションではありません。廃止された Control UI の
アップグレード入力によって、任意の
`client.mode: "backend"` または CLI 形式のクライアントに一時アクセスが付与されることはありません。カスタム自動化では、
デバイス ID とペアリング、予約済みのローカル直接 `client.id: "gateway-client"`
バックエンドヘルパーパス、または HTTP のリクエスト/レスポンス形式がより適している場合は [管理用 HTTP RPC Plugin](/ja-JP/plugins/admin-http-rpc)
を使用してください。

## オペレータースコープヘッダー

Trusted-proxy 認証は**アイデンティティを伴う** HTTP モードであるため、呼び出し元は HTTP API リクエストで `x-openclaw-scopes` を使用してオペレータースコープを任意で宣言できます。

注: WebSocket スコープは、Gateway プロトコルのハンドシェイクとデバイスアイデンティティのバインディングによって決定されます。Control UI の WebSocket アップグレードリクエストでは、`x-openclaw-scopes` はネゴシエートされたセッションスコープの上限にすぎず、スコープを付与するものではありません。[Control UI のペアリング動作](#control-ui-pairing-behavior)を参照してください。

例:

- `x-openclaw-scopes: operator.read`
- `x-openclaw-scopes: operator.read,operator.write`
- `x-openclaw-scopes: operator.admin,operator.write`

動作:

- ヘッダーが存在する場合、OpenClaw は宣言されたスコープセットを適用します。
- ヘッダーが存在していても空の場合、リクエストはオペレータースコープを**一切持たない**ことを宣言します。
- ヘッダーが存在しない場合、通常のアイデンティティを伴う HTTP API は、標準のオペレーター用デフォルトスコープセット（`operator.admin`、`operator.read`、`operator.write`、`operator.approvals`、`operator.pairing`、`operator.talk.secrets`）にフォールバックします。
- Gateway 認証を使用する **Plugin HTTP ルート**は、デフォルトではより制限されています。`x-openclaw-scopes` が存在しない場合、そのランタイムスコープは `operator.write` のみにフォールバックします。
- ブラウザーオリジンの HTTP リクエストは、trusted-proxy 認証に成功した後も、`gateway.controlUi.allowedOrigins`（または明示的に有効化した Host ヘッダーのフォールバックモード）を通過する必要があります。

実用上のルール: trusted-proxy リクエストをデフォルトより狭いスコープにする場合、または Gateway 認証を使用する Plugin ルートで書き込みスコープより強い権限が必要な場合は、`x-openclaw-scopes` を明示的に送信してください。

## TLS 終端と HSTS

TLS 終端ポイントは 1 つにし、そこで HSTS を適用します。

<Tabs>
  <Tab title="プロキシでの TLS 終端（推奨）">
    リバースプロキシが `https://control.example.com` の HTTPS を処理する場合は、そのドメインについてプロキシで `Strict-Transport-Security` を設定します。

    - インターネットに公開するデプロイに適しています。
    - 証明書と HTTP 強化ポリシーを一元管理できます。
    - OpenClaw はプロキシの背後でループバック HTTP のまま運用できます。

    ヘッダー値の例:

    ```text
    Strict-Transport-Security: max-age=31536000; includeSubDomains
    ```

  </Tab>
  <Tab title="Gateway での TLS 終端">
    OpenClaw 自体が HTTPS を直接提供する場合（TLS 終端プロキシを使用しない場合）は、次を設定します。

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

    `strictTransportSecurity` には文字列のヘッダー値を指定できます。また、明示的に無効化するには `false` を指定できます。

  </Tab>
</Tabs>

### ロールアウトのガイダンス

- トラフィックを検証している間は、まず短い最大有効期間（例: `max-age=300`）から始めます。
- 十分な確信が得られた後にのみ、長期間の値（例: `max-age=31536000`）へ増やします。
- すべてのサブドメインで HTTPS を利用できる場合にのみ、`includeSubDomains` を追加します。
- ドメインセット全体でプリロード要件を意図的に満たす場合にのみ、preload を使用します。
- ループバックのみを使用するローカル開発では、HSTS の利点はありません。

## プロキシ設定例

<AccordionGroup>
  <Accordion title="Pomerium">
    Pomerium は `x-pomerium-claim-email`（またはその他のクレームヘッダー）でアイデンティティを渡し、`x-pomerium-jwt-assertion` で JWT を渡します。

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // Pomerium の IP
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

    Pomerium の設定スニペット:

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
  <Accordion title="OAuth を使用する Caddy">
    `caddy-security` Plugin を使用する Caddy は、ユーザーを認証してアイデンティティヘッダーを渡すことができます。

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // Caddy/サイドカープロキシの IP
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-forwarded-user",
          },
        },
      },
    }
    ```

    Caddyfile のスニペット:

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
    oauth2-proxy はユーザーを認証し、`x-auth-request-email` でアイデンティティを渡します。

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // nginx/oauth2-proxy の IP
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-auth-request-email",
          },
        },
      },
    }
    ```

    nginx の設定スニペット:

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
  <Accordion title="フォワード認証を使用する Traefik">
    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["172.17.0.1"], // Traefik コンテナの IP
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

## トークンを併用した設定

共有トークン（`gateway.auth.token` または `OPENCLAW_GATEWAY_TOKEN`）も設定されている場合、Gateway の起動時に trusted-proxy 認証は拒否されます。この 2 つは相互排他的です。共有トークンを使用すると、同じホスト上の呼び出し元が、このモードで強制することを意図したプロキシ検証済みアイデンティティとはまったく異なる経路で認証できてしまうためです。

`gateway auth mode is trusted-proxy, but a shared token is also configured` のようなエラーで起動に失敗する場合:

- trusted-proxy モードを使用するときは共有トークンを削除するか、
- トークンベースの認証を使用する場合は、`gateway.auth.mode` を `"token"` に切り替えます。

ループバックの trusted-proxy アイデンティティヘッダーもフェイルクローズになります。同じホスト上の呼び出し元が、暗黙的にプロキシユーザーとして認証されることはありません。プロキシを経由しない OpenClaw の内部呼び出し元は、代わりに `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` で認証できます。trusted-proxy モードでは、トークンへのフォールバックは意図的にサポートされていません。

## セキュリティチェックリスト

trusted-proxy 認証を有効にする前に、以下を確認してください。

- [ ] **プロキシが唯一の経路である**: Gateway ポートへのアクセスを、使用するプロキシ以外のすべてからファイアウォールで遮断します。
- [ ] **trustedProxies が最小限である**: サブネット全体ではなく、実際に使用するプロキシの IP のみを指定します。
- [ ] **ループバックのプロキシソースを意図的に使用している**: 同じホスト上のプロキシ向けに `gateway.auth.trustedProxy.allowLoopback` を明示的に有効化しない限り、ループバックソースのリクエストに対する trusted-proxy 認証はフェイルクローズになります。
- [ ] **プロキシがヘッダーを除去する**: プロキシがクライアントからの `x-forwarded-*` ヘッダーを追記するのではなく、上書きするようにします。
- [ ] **TLS 終端**: プロキシが TLS を処理し、ユーザーが HTTPS 経由で接続するようにします。
- [ ] **allowedOrigins が明示されている**: local loopback 以外の Control UI では、`gateway.controlUi.allowedOrigins` を明示的に指定します。
- [ ] **allowUsers が設定されている**（推奨）: 認証されたすべてのユーザーを許可するのではなく、既知のユーザーに制限します。
- [ ] **トークンを併用していない**: `gateway.auth.token` と `gateway.auth.mode: "trusted-proxy"` の両方を設定しないでください。
- [ ] **ローカルパスワードへのフォールバックが非公開である**: 内部の直接呼び出し元向けに `gateway.auth.password` を設定する場合は、プロキシを経由しないリモートクライアントが Gateway ポートへ直接到達できないよう、ファイアウォールで遮断してください。
- [ ] **デバイスの自動承認を意図的に使用している**: `deviceAutoApprove.enabled` が true の場合は、リバースプロキシのアカウントセキュリティをデバイス登録の境界として扱い、付与するスコープ一覧を管理者権限なしの最小限に保ちます。

## セキュリティ監査

`openclaw security audit` は trusted-proxy 認証を**重大**な深刻度の検出結果として報告します。これは意図された動作であり、セキュリティをプロキシ設定へ委任していることを注意喚起するものです。

監査では以下を確認します。

- 基本の `gateway.trusted_proxy_auth` 警告/重大リマインダー。
- `trustedProxies` 設定の欠如。
- `userHeader` 設定の欠如。
- 空の `allowUsers`（認証されたすべてのユーザーを許可します）。
- 同じホスト上のプロキシソースに対して `allowLoopback` が有効になっていること。
- ブラウザーデバイスの自動承認が有効になっていること（新しいデバイスのペアリングをプロキシアイデンティティへ委任します）。

Control UI が公開されている場合は、trusted-proxy に固有ではない別の検出結果も適用されます。これには、ワイルドカードまたは未設定の `gateway.controlUi.allowedOrigins`、および Host ヘッダーによるオリジンのフォールバックが含まれます。

## トラブルシューティング

<AccordionGroup>
  <Accordion title="trusted_proxy_untrusted_source">
    リクエストの送信元が `gateway.trustedProxies` に含まれる IP ではありませんでした。以下を確認してください。

    - プロキシの IP は正しいですか？（Docker コンテナの IP は変わる可能性があります。）
    - プロキシの前段にロードバランサーがありますか？
    - 実際の IP を確認するには、`docker inspect` または `kubectl get pods -o wide` を使用します。

  </Accordion>
  <Accordion title="trusted_proxy_loopback_source">
    OpenClaw は、ループバックソースからの trusted-proxy リクエストを拒否しました。

    以下を確認してください。

    - プロキシは `127.0.0.1` / `::1` から接続していますか？
    - 同じホスト上のループバックリバースプロキシで trusted-proxy 認証を使用しようとしていますか？

    修正方法:

    - プロキシを経由しない同じホスト上の内部クライアントには、トークン/パスワード認証を使用することを推奨します。または、
    - local loopback 以外の信頼済みプロキシアドレスを経由させ、その IP を `gateway.trustedProxies` に保持します。または、
    - 意図的に同じホスト上のリバースプロキシを使用する場合は、`gateway.auth.trustedProxy.allowLoopback = true` を設定し、ループバックアドレスを `gateway.trustedProxies` に保持して、プロキシがアイデンティティヘッダーを除去または上書きすることを確認します。

  </Accordion>
  <Accordion title="trusted_proxy_local_interface_source / trusted_proxy_local_interface_check_failed">
    リクエストの送信元 IP が、（プロキシではなく）Gateway ホスト自体の local loopback 以外のネットワークインターフェースアドレスのいずれかと一致しました。これは、tailnet または Docker ブリッジネットワーク上で同じホストから送信元を偽装したトラフィックを防止するための保護機能です。`..._check_failed` は、インターフェースの検出自体でエラーが発生したことを示すため、OpenClaw はフェイルクローズになります。

    以下を確認してください。

    - Gateway ホスト自体のプロセスが、プロキシを迂回してアイデンティティヘッダーを直接送信していませんか？
    - プロキシが Gateway と同じネットワーク名前空間で実行されており、その IP がローカルインターフェースとしても表示されていませんか？

    修正方法: Gateway ホストにもローカルでバインドされているアドレスを経由しないようにプロキシトラフィックをルーティングするか、実際に同じホスト上でプロキシを構成する場合にのみ `allowLoopback` を使用します。

  </Accordion>
  <Accordion title="trusted_proxy_user_missing">
    ユーザーヘッダーが空であるか、存在しませんでした。以下を確認してください。

    - プロキシはアイデンティティヘッダーを渡すように設定されていますか？
    - ヘッダー名は正しいですか？（大文字と小文字は区別されませんが、スペルは正確である必要があります）
    - ユーザーは実際にプロキシで認証されていますか？

  </Accordion>
  <Accordion title="trusted_proxy_missing_header_*">
    必須ヘッダーが存在しませんでした。以下を確認してください。

    - 該当する各ヘッダーについてのプロキシ設定。
    - 経路の途中でヘッダーが除去されていないか。

  </Accordion>
  <Accordion title="trusted_proxy_user_not_allowed">
    ユーザーは認証されていますが、`allowUsers` に含まれていません。ユーザーを追加するか、許可リストを削除してください。
  </Accordion>
  <Accordion title="trusted_proxy_no_proxies_configured / trusted_proxy_config_missing">
    `gateway.auth.mode` は `"trusted-proxy"` ですが、`gateway.trustedProxies` が空であるか、`gateway.auth.trustedProxy` 自体がありません。両方を設定するまで、すべてのリクエストが拒否されます。
  </Accordion>
  <Accordion title="trusted_proxy_origin_not_allowed">
    信頼済みプロキシ認証には成功しましたが、ブラウザーの `Origin` ヘッダーが Control UI のオリジンチェックに合格しませんでした。

    次を確認してください。

    - `gateway.controlUi.allowedOrigins` にブラウザーの正確なオリジンが含まれている。
    - すべてを許可する動作を意図している場合を除き、ワイルドカードオリジンに依存していない。
    - Host ヘッダーのフォールバックモードを意図的に使用する場合、`gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true` が明示的に設定されている。

  </Accordion>
  <Accordion title="接続には成功するが、メソッドでスコープ不足が報告される">
    WebSocket は接続されますが、`chat.history`、`sessions.list`、または
    `models.list` が `missing scope: operator.read` で失敗します。

    一般的な原因:

    - デバイスなしの Control UI セッション: 信頼済みプロキシ認証では、デバイス ID がなくても WebSocket 接続を許可できますが、OpenClaw は設計上、デバイスなしのセッションのスコープを消去します。
    - カスタムバックエンドクライアント: 廃止された Control UI アップグレード入力では、任意のバックエンドクライアントや CLI 形式の WebSocket クライアントにアクセス権が付与されることはありません。
    - 過度に限定された `x-openclaw-scopes`: プロキシが Control UI の WebSocket アップグレードリクエストにこのヘッダーを挿入すると、セッションスコープはそのセットに制限されます。ヘッダー値が空の場合、スコープは付与されません。

    修正方法:

    - Control UI では、ブラウザーがデバイス ID を生成してペアリングを完了できるように HTTPS を使用します。
    - カスタム自動化では、デバイス ID とペアリング、予約済みの直接ローカル `gateway-client` バックエンドヘルパーパス、または[管理用 HTTP RPC](/ja-JP/plugins/admin-http-rpc)を使用します。
    - 現在の設定に、廃止された `gateway.controlUi.dangerouslyDisableDeviceAuth` キーを追加しないでください。古いインストールでは、1 回限りの自己ペアリング移行が自動的に使用されます。

  </Accordion>
  <Accordion title="WebSocket が引き続き失敗する">
    プロキシが次の条件を満たしていることを確認してください。

    - WebSocket アップグレード (`Upgrade: websocket`、`Connection: upgrade`) をサポートしている。
    - HTTP だけでなく、WebSocket アップグレードリクエストでも ID ヘッダーを渡している。
    - WebSocket 接続に別の認証パスを使用していない。

  </Accordion>
</AccordionGroup>

## トークン認証からの移行

<Steps>
  <Step title="プロキシを設定する">
    ユーザーを認証してヘッダーを渡すようにプロキシを設定します。
  </Step>
  <Step title="プロキシを個別にテストする">
    プロキシの設定を個別にテストします（ヘッダーを指定した curl）。
  </Step>
  <Step title="OpenClaw の設定を更新する">
    OpenClaw の設定を信頼済みプロキシ認証で更新します。
  </Step>
  <Step title="Gateway を再起動する">
    Gateway を再起動します。
  </Step>
  <Step title="WebSocket をテストする">
    Control UI から WebSocket 接続をテストします。
  </Step>
  <Step title="監査する">
    `openclaw security audit` を実行し、検出結果を確認します。
  </Step>
</Steps>

## 関連項目

- [設定](/ja-JP/gateway/configuration) — 設定リファレンス
- [オペレータースコープ](/ja-JP/gateway/operator-scopes) — ロール、スコープ、承認チェック
- [リモートアクセス](/ja-JP/gateway/remote) — その他のリモートアクセスパターン
- [セキュリティ](/ja-JP/gateway/security) — 完全なセキュリティガイド
- [Tailscale](/ja-JP/gateway/tailscale) — tailnet のみにアクセスする場合の、より簡単な代替手段
