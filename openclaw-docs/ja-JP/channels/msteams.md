---
read_when:
    - Microsoft Teams チャネル機能の開発
summary: Microsoft Teams ボットのサポート状況、機能、設定
title: Microsoft Teams
x-i18n:
    generated_at: "2026-07-26T10:05:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5a4cf686da27e28b58f7afaad8cc837dbddb93219cde0c37285f9f6895f6fb8c
    source_path: channels/msteams.md
    workflow: 16
---

ステータス: テキストおよび DM の添付ファイルに対応しています。チャネル/グループでのファイル送信には `sharePointSiteId` と Graph 権限が必要です（[グループチャットでのファイル送信](#sending-files-in-group-chats)を参照）。投票は Adaptive Cards 経由で送信されます。メッセージアクションでは、ファイル優先の送信に明示的な `upload-file` が公開されます。

## バンドル済み Plugin

Microsoft Teams は、現在の OpenClaw リリースではバンドル済み Plugin として提供されます。通常のパッケージビルドでは、個別のインストールは必要ありません。

古いビルド、またはバンドル済み Teams を除外したカスタムインストールでは、npm パッケージを直接インストールします。

```bash
openclaw plugins install @openclaw/msteams
```

現在の公式リリースタグに追従するには、バージョン指定なしのパッケージを使用します。再現可能なインストールが必要な場合にのみ、正確なバージョンを固定してください。

ローカルチェックアウト（git リポジトリから実行）:

```bash
openclaw plugins install ./path/to/local/msteams-plugin
```

詳細: [Plugin](/ja-JP/tools/plugin)

## クイックセットアップ

[`@microsoft/teams.cli`](https://www.npmjs.com/package/@microsoft/teams.cli) は、1 つのコマンドでボット登録、マニフェスト作成、認証情報生成を処理します。

**1. インストールしてログインする**

```bash
npm install -g @microsoft/teams.cli@preview
teams login
teams status   # ログイン済みであることを確認し、テナント情報を表示
```

<Note>
Teams CLI は現在プレビュー版です。コマンドとフラグはリリース間で変更される可能性があります。
</Note>

**2. トンネルを開始する**（Teams は localhost に到達できません）

必要に応じて devtunnel CLI をインストールして認証します（[はじめにガイド](https://learn.microsoft.com/en-us/azure/developer/dev-tunnels/get-started)）。

```bash
# 初回のみのセットアップ（セッション間で永続する URL）:
devtunnel create my-openclaw-bot --allow-anonymous
devtunnel port create my-openclaw-bot -p 3978 --protocol auto

# 開発セッションごと:
devtunnel host my-openclaw-bot
# エンドポイント: https://<tunnel-id>.devtunnels.ms/api/messages
```

<Note>
Teams は devtunnels で認証できないため、`--allow-anonymous` が必要です。受信する各ボットリクエストは、引き続き Teams SDK によって検証されます。
</Note>

代替手段: `ngrok http 3978` または `tailscale funnel 3978`（URL はセッションごとに変更される場合があります）。

**3. アプリを作成する**

```bash
teams app create \
  --name "OpenClaw" \
  --endpoint "https://<your-tunnel-url>/api/messages"
```

これにより、Entra ID（Azure AD）アプリケーションの作成、クライアントシークレットの生成、Teams アプリマニフェスト（アイコンを含む）のビルドとアップロード、および Teams 管理ボットの登録が行われます（Azure サブスクリプションは不要）。出力には `CLIENT_ID`、`CLIENT_SECRET`、`TENANT_ID`、および **Teams App ID** が含まれ、アプリを Teams に直接インストールするオプションも提示されます。

**4. OpenClaw を設定する**。出力された認証情報を使用します。

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<CLIENT_ID>",
      appPassword: "<CLIENT_SECRET>",
      tenantId: "<TENANT_ID>",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

または、環境変数 `MSTEAMS_APP_ID`、`MSTEAMS_APP_PASSWORD`、`MSTEAMS_TENANT_ID` を直接使用します。

**5. アプリを Teams にインストールする**

`teams app create` によりアプリのインストールを求められます。「Install in Teams」を選択します。後でインストールリンクを取得するには、次を実行します。

```bash
teams app get <teamsAppId> --install-link
```

**6. すべてが動作することを確認する**

```bash
teams app doctor <teamsAppId>
```

ボット登録、AAD アプリ設定、マニフェストの有効性、SSO 設定にわたる診断を実行します。

本番環境では、クライアントシークレットの代わりに[フェデレーション認証](#federated-authentication-certificate-plus-managed-identity)（証明書またはマネージド ID）の使用を検討してください。

<Note>
グループチャットはデフォルトでブロックされます（`channels.msteams.groupPolicy: "allowlist"`）。グループへの返信を許可するには `channels.msteams.groupAllowFrom` を設定するか、`groupPolicy: "open"` を使用して任意のメンバーを許可します（メンション必須）。
</Note>

## 目標

- Teams の DM、グループチャット、またはチャネルを介して OpenClaw と対話します。
- ルーティングを決定論的に保ちます。返信は常に、受信したチャネルへ返されます。
- 安全なチャネル動作をデフォルトにします（別途設定しない限りメンションが必要です）。

## 設定の書き込み

デフォルトでは、Microsoft Teams は `/config set|unset` によってトリガーされる設定更新を書き込めます（`commands.config: true` が必要）。

無効にするには、次のように設定します。

```json5
{
  channels: { msteams: { configWrites: false } },
}
```

## アクセス制御（DM + グループ）

**DM アクセス**

- デフォルト: `channels.msteams.dmPolicy = "pairing"`。不明な送信者は、承認されるまで無視されます。
- `channels.msteams.allowFrom` には、安定した AAD オブジェクト ID、または `accessGroup:core-team` のような静的送信者アクセスグループを使用してください。
- 許可リストでは UPN/表示名の照合に依存しないでください。これらは変更される可能性があります。OpenClaw は直接的な名前照合をデフォルトで無効にしています。有効にするには `channels.msteams.dangerouslyAllowNameMatching: true` を使用します。
- 認証情報で許可されている場合、ウィザードは Microsoft Graph を介して名前を ID に解決できます。

**グループアクセス**

- デフォルト: `channels.msteams.groupPolicy = "allowlist"`（`groupAllowFrom` を追加しない限りブロック）。`channels.msteams.groupPolicy` が未設定の場合、`channels.defaults.groupPolicy` で共有デフォルトを上書きできます。
- `channels.msteams.groupAllowFrom` は、グループチャット/チャネルでトリガーできる送信者または静的送信者アクセスグループを制御します（`channels.msteams.allowFrom` にフォールバック）。
- 任意のメンバーを許可するには `groupPolicy: "open"` を設定します（デフォルトでは引き続きメンション必須）。
- **すべて**のチャネルをブロックするには、`channels.msteams.groupPolicy: "disabled"` を設定します。

例:

```json5
{
  channels: {
    msteams: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["00000000-0000-0000-0000-000000000000", "accessGroup:core-team"],
    },
  },
}
```

**チーム + チャネル許可リスト**

- `channels.msteams.teams` の下にチームとチャネルを列挙して、グループ/チャネルへの返信範囲を限定します。
- キーには、変更可能な表示名ではなく、Teams リンクから取得した安定した Teams 会話 ID を使用します（[チーム ID とチャネル ID](#team-and-channel-ids-common-gotcha)を参照）。
- `groupPolicy="allowlist"` でチーム許可リストが存在する場合、列挙されたチーム/チャネルのみが受け入れられます（メンション必須）。
- 設定ウィザードは `Team/Channel` エントリを受け付け、それらを保存します。
- 起動時に OpenClaw は、Graph 権限で許可されている場合、チーム/チャネルおよびユーザー許可リストの名前を ID に解決し、その対応関係をログに記録します。解決できない名前は入力どおりに保持されますが、`channels.msteams.dangerouslyAllowNameMatching: true` が設定されていない限り、ルーティングでは無視されます。

例:

```json5
{
  channels: {
    msteams: {
      groupPolicy: "allowlist",
      teams: {
        "My Team": {
          channels: {
            General: { requireMention: true },
          },
        },
      },
    },
  },
}
```

<details>
<summary><strong>手動セットアップ（Teams CLI を使用しない場合）</strong></summary>

### 動作の仕組み

1. Microsoft Teams Plugin が利用可能であることを確認します（現在のリリースではバンドル済み）。
2. **Azure Bot**（App ID + シークレット + テナント ID）を作成します。
3. 以下の RSC 権限を含め、ボットを参照する **Teams アプリパッケージ**をビルドします。
4. Teams アプリをチームにアップロード/インストールします（DM の場合は個人スコープ）。
5. `~/.openclaw/openclaw.json`（または環境変数）で `msteams` を設定し、Gateway を起動します。
6. Gateway はデフォルトで `/api/messages` 上の Bot Framework Webhook トラフィックを待ち受けます。

### ステップ 1: Azure Bot を作成する

1. [Create Azure Bot](https://portal.azure.com/#create/Microsoft.AzureBot) に移動します。
2. **Basics** タブに入力します。

   | フィールド         | 値                                                       |
   | ------------------ | -------------------------------------------------------- |
   | **Bot handle**     | ボット名（例: `openclaw-msteams`、一意である必要があります） |
   | **Subscription**   | Azure サブスクリプションを選択                           |
   | **Resource group** | 新規作成するか既存のものを使用                           |
   | **Pricing tier**   | 開発/テストでは **Free**                                 |
   | **Type of App**    | **Single Tenant**（推奨。以下の注記を参照）              |
   | **Creation type**  | **Create new Microsoft App ID**                          |

<Warning>
新しいマルチテナントボットの作成は 2025-07-31 以降、非推奨となりました。新しいボットには **Single Tenant** を使用してください。
</Warning>

3. **Review + create**、続いて **Create** をクリックします（約 1～2 分）。

### ステップ 2: 認証情報を取得する

1. Azure Bot リソース → **Configuration** → **Microsoft App ID**（`appId`）をコピーします。
2. **Manage Password** → App Registration → **Certificates & secrets** → **New client secret** → **Value**（`appPassword`）をコピーします。
3. **Overview** → **Directory (tenant) ID**（`tenantId`）をコピーします。

### ステップ 3: メッセージングエンドポイントを設定する

1. Azure Bot → **Configuration**。
2. **Messaging endpoint** を設定します。
   - 本番環境: `https://your-domain.com/api/messages`
   - ローカル開発: トンネルを使用します（[ローカル開発](#local-development-tunneling)を参照）

### ステップ 4: Teams チャネルを有効にする

1. Azure Bot → **Channels**。
2. **Microsoft Teams** → Configure → Save の順にクリックします。
3. 利用規約に同意します。

### ステップ 5: Teams アプリマニフェストをビルドする

- `botId = <App ID>` を持つ `bot` エントリを含めます。
- スコープ: `personal`、`team`、`groupChat`。
- `supportsFiles: true`（個人スコープでのファイル処理に必要）。
- RSC 権限を追加します（[RSC 権限](#current-teams-rsc-permissions-manifest)を参照）。
- アイコンを作成します: `outline.png`（32x32）と `color.png`（192x192）。
- `manifest.json`、`outline.png`、`color.png` をまとめて zip 圧縮します。

### ステップ 6: OpenClaw を設定する

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<APP_ID>",
      appPassword: "<APP_PASSWORD>",
      tenantId: "<TENANT_ID>",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

環境変数: `MSTEAMS_APP_ID`、`MSTEAMS_APP_PASSWORD`、`MSTEAMS_TENANT_ID`。

### ステップ 7: Gateway を実行する

Plugin が利用可能で、`msteams` 設定に認証情報が含まれている場合、Teams チャネルは自動的に起動します。

</details>

## フェデレーション認証（証明書とマネージド ID）

本番環境向けに、OpenClaw は `channels.msteams.authType: "federated"` を介して、クライアントシークレットの代替となる**フェデレーション認証**をサポートしています。方法は 2 つあります。

### オプション A: 証明書ベースの認証

Entra ID アプリ登録に登録した PEM 証明書を使用します。

**セットアップ:**

1. 証明書（秘密鍵を含む PEM 形式）を生成または取得します。
2. Entra ID → App Registration → **Certificates & secrets** → **Certificates** → 公開証明書をアップロードします。

**設定:**

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<APP_ID>",
      tenantId: "<TENANT_ID>",
      authType: "federated",
      certificatePath: "/path/to/cert.pem",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

**環境変数:**

- `MSTEAMS_AUTH_TYPE=federated`
- `MSTEAMS_CERTIFICATE_PATH=/path/to/cert.pem`

### オプション B: Azure Managed Identity

Azure インフラストラクチャ（AKS、App Service、Azure VM）上で、パスワードレス認証に Azure Managed Identity を使用します。

**動作の仕組み:**

1. ボットの Pod/VM にマネージド ID（システム割り当てまたはユーザー割り当て）があります。
2. フェデレーション ID 認証情報によって、マネージド ID と Entra ID アプリ登録がリンクされます。
3. 実行時に、OpenClaw は `@azure/identity` を使用して Azure IMDS エンドポイントからトークンを取得します。
4. トークンはボット認証のために Teams SDK に渡されます。

**前提条件:**

- マネージド ID が有効な Azure インフラストラクチャ（AKS ワークロード ID、App Service、VM）。
- Entra ID アプリ登録にフェデレーション ID 資格情報が作成されていること。
- ポッド/VM から IMDS（`169.254.169.254:80`）へのネットワークアクセス。

**設定（システム割り当てマネージド ID）：**

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<APP_ID>",
      tenantId: "<TENANT_ID>",
      authType: "federated",
      useManagedIdentity: true,
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

**設定（ユーザー割り当てマネージド ID）：** 上記のブロックに `managedIdentityClientId: "<MI_CLIENT_ID>"` を追加します。

**環境変数：**

- `MSTEAMS_AUTH_TYPE=federated`
- `MSTEAMS_USE_MANAGED_IDENTITY=true`
- `MSTEAMS_MANAGED_IDENTITY_CLIENT_ID=<client-id>`（ユーザー割り当ての場合のみ）

### AKS ワークロード ID の設定

ワークロード ID を使用する AKS デプロイの場合：

1. AKS クラスターで**ワークロード ID を有効化**します。
2. Entra ID アプリ登録に**フェデレーション ID 資格情報を作成**します：

   ```bash
   az ad app federated-credential create --id <APP_OBJECT_ID> --parameters '{
     "name": "my-bot-workload-identity",
     "issuer": "<AKS_OIDC_ISSUER_URL>",
     "subject": "system:serviceaccount:<NAMESPACE>:<SERVICE_ACCOUNT>",
     "audiences": ["api://AzureADTokenExchange"]
   }'
   ```

3. アプリのクライアント ID を使用して **Kubernetes サービスアカウントに注釈を付けます**：

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: my-bot-sa
     annotations:
       azure.workload.identity/client-id: "<APP_CLIENT_ID>"
   ```

4. ワークロード ID を注入するために**ポッドにラベルを付けます**：

   ```yaml
   metadata:
     labels:
       azure.workload.identity/use: "true"
   ```

5. IMDS（`169.254.169.254`）への**ネットワークアクセスを許可**します。NetworkPolicy を使用している場合は、ポート 80 の `169.254.169.254/32` に対するエグレスルールを追加します。

### 認証タイプの比較

| 方式                     | 設定                                           | 長所                                     | 短所                                             |
| ------------------------ | ---------------------------------------------- | ---------------------------------------- | ------------------------------------------------ |
| **クライアントシークレット** | `appPassword`                                  | 設定が簡単                               | シークレットのローテーションが必要で、安全性が低い |
| **証明書**               | `authType: "federated"` + `certificatePath`    | ネットワーク経由で共有シークレットを使わない | 証明書管理の負担                                 |
| **マネージド ID**        | `authType: "federated"` + `useManagedIdentity` | パスワード不要で、管理するシークレットがない | Azure インフラストラクチャが必要                 |

`certificateThumbprint` は `certificatePath` と同時に設定できますが、現在の認証パスでは読み取られません。前方互換性のためだけに受け付けられます。

**デフォルト：** `authType` が未設定の場合、OpenClaw はクライアントシークレット認証（`appPassword`）を使用します。既存の設定は変更なしで引き続き機能します。

## ローカル開発（トンネリング）

Teams は `localhost` に到達できません。セッション間で URL を安定させるため、永続的な開発トンネルを使用します：

```bash
# 初回のみの設定：
devtunnel create my-openclaw-bot --allow-anonymous
devtunnel port create my-openclaw-bot -p 3978 --protocol auto

# 開発セッションごと：
devtunnel host my-openclaw-bot
```

代替手段：`ngrok http 3978` または `tailscale funnel 3978`（セッションごとに URL が変わる場合があります）。

トンネル URL が変わった場合は、エンドポイントを更新します：

```bash
teams app update <teamsAppId> --endpoint "https://<new-url>/api/messages"
```

## ボットのテスト

**診断を実行：**

```bash
teams app doctor <teamsAppId>
```

ボット登録、AAD アプリ、マニフェスト、SSO 設定を一度に確認します。

**テストメッセージを送信：**

1. Teams アプリをインストールします（`teams app get <id> --install-link` のインストールリンクを使用）。
2. Teams でボットを見つけ、DM を送信します。
3. 着信アクティビティがないか Gateway ログを確認します。

## 環境変数

これらの認証関連の設定キーは、`openclaw.json` の代わりに環境変数で設定できます（`groupPolicy` や `historyLimit` など、その他の設定キーは設定ファイルでのみ指定できます）：

| 環境変数                             | 設定キー                  | 備考                                      |
| ------------------------------------ | ------------------------- | ----------------------------------------- |
| `MSTEAMS_APP_ID`                     | `appId`                   |                                           |
| `MSTEAMS_APP_PASSWORD`               | `appPassword`             |                                           |
| `MSTEAMS_TENANT_ID`                  | `tenantId`                |                                           |
| `MSTEAMS_AUTH_TYPE`                  | `authType`                | `"secret"` または `"federated"`         |
| `MSTEAMS_CERTIFICATE_PATH`           | `certificatePath`         | フェデレーション + 証明書                 |
| `MSTEAMS_CERTIFICATE_THUMBPRINT`     | `certificateThumbprint`   | 受け付けられるが、認証には不要            |
| `MSTEAMS_USE_MANAGED_IDENTITY`       | `useManagedIdentity`      | フェデレーション + マネージド ID          |
| `MSTEAMS_MANAGED_IDENTITY_CLIENT_ID` | `managedIdentityClientId` | ユーザー割り当てマネージド ID の場合のみ  |

## メンバー情報アクション

OpenClaw は Microsoft Teams 向けに Graph ベースの `member-info` アクションを公開し、エージェントや自動化が設定済みの会話について検証済みのメンバー一覧情報を解決できるようにします。

要件：

- `ChannelSettings.Read.Group` および `TeamMember.Read.Group` RSC アクセス許可（推奨マニフェストにすでに含まれています）。

Graph 資格情報が設定されていれば、このアクションを利用できます。個別の `channels.msteams.actions.memberInfo` トグルはありません。
標準チャネルの検索では、一致するチームメンバー一覧の ID、表示名、メールアドレス、ロールが返されます。
現在の DM またはグループチャットでは、このアクションは信頼済み送信者の安定したユーザー ID を返せます。
プライベート/共有チャネル、および現在のチャット以外のメンバー検索には追加のメンバー一覧アクセス許可が必要であり、
デフォルトのアクセス許可ベースラインでは拒否されます。

## 履歴コンテキスト

- `channels.msteams.historyLimit` は、プロンプトに組み込む最近のチャネル/グループメッセージ数を制御します。`messages.groupChat.historyLimit` にフォールバックし、その後デフォルト値の 50 が使用されます。無効にするには `0` を設定します。
- 取得したスレッド履歴は送信者許可リスト（`allowFrom` / `groupAllowFrom`）でフィルタリングされるため、スレッドコンテキストの初期投入には許可された送信者からのメッセージのみが含まれます。
- 引用された添付ファイルのコンテキスト（返信自体の添付ファイルにある Skype Reply スキーマの HTML から解析）はフィルタリングされずに渡されます。現在、送信者許可リストのフィルターが適用されるのはスレッド履歴の初期投入だけです。
- DM 履歴は `channels.msteams.dmHistoryLimit`（ユーザーターン数）で制限できます。ユーザーごとのオーバーライド：`channels.msteams.dms["<user_id>"].historyLimit`。

## 現在の Teams RSC アクセス許可（マニフェスト）

以下は、Teams アプリのマニフェストにある**既存の resourceSpecific アクセス許可**です。アプリがインストールされているチーム/チャット内でのみ適用されます。

**チャネル用（チームスコープ）：**

- `ChannelMessage.Read.Group`（Application）- @メンションなしですべてのチャネルメッセージを受信
- `ChannelMessage.Send.Group`（Application）
- `Member.Read.Group`（Application）
- `Owner.Read.Group`（Application）
- `ChannelSettings.Read.Group`（Application）
- `TeamMember.Read.Group`（Application）
- `TeamSettings.Read.Group`（Application）

**グループチャット用：**

- `ChatMessage.Read.Chat`（Application）- @メンションなしですべてのグループチャットメッセージを受信

Teams CLI を使用して RSC アクセス許可を追加します：

```bash
teams app rsc add <teamsAppId> ChannelMessage.Read.Group --type Application
```

## Teams マニフェストの例（編集済み）

必須フィールドを含む最小限の有効な例です。ID と URL を置き換えてください。

```json5
{
  $schema: "https://developer.microsoft.com/en-us/json-schemas/teams/v1.23/MicrosoftTeams.schema.json",
  manifestVersion: "1.23",
  version: "1.0.0",
  id: "00000000-0000-0000-0000-000000000000",
  name: { short: "OpenClaw" },
  developer: {
    name: "Your Org",
    websiteUrl: "https://example.com",
    privacyUrl: "https://example.com/privacy",
    termsOfUseUrl: "https://example.com/terms",
  },
  description: { short: "OpenClaw in Teams", full: "OpenClaw in Teams" },
  icons: { outline: "outline.png", color: "color.png" },
  accentColor: "#5B6DEF",
  bots: [
    {
      botId: "11111111-1111-1111-1111-111111111111",
      scopes: ["personal", "team", "groupChat"],
      isNotificationOnly: false,
      supportsCalling: false,
      supportsVideo: false,
      supportsFiles: true,
    },
  ],
  webApplicationInfo: {
    id: "11111111-1111-1111-1111-111111111111",
  },
  authorization: {
    permissions: {
      resourceSpecific: [
        { name: "ChannelMessage.Read.Group", type: "Application" },
        { name: "ChannelMessage.Send.Group", type: "Application" },
        { name: "Member.Read.Group", type: "Application" },
        { name: "Owner.Read.Group", type: "Application" },
        { name: "ChannelSettings.Read.Group", type: "Application" },
        { name: "TeamMember.Read.Group", type: "Application" },
        { name: "TeamSettings.Read.Group", type: "Application" },
        { name: "ChatMessage.Read.Chat", type: "Application" },
      ],
    },
  },
}
```

### マニフェストの注意事項（必須フィールド）

- `bots[].botId` は Azure Bot App ID と**一致する必要があります**。
- `webApplicationInfo.id` は Azure Bot App ID と**一致する必要があります**。
- `bots[].scopes` には、使用予定のサーフェス（`personal`、`team`、`groupChat`）を含める必要があります。
- パーソナルスコープでファイルを処理するには `bots[].supportsFiles: true` が必要です。
- `authorization.permissions.resourceSpecific` には、チャネル通信のためのチャネル読み取り/送信を含める必要があります。

### 既存アプリの更新

```bash
# マニフェストをダウンロード、編集、再アップロード
teams app manifest download <teamsAppId> manifest.json
# manifest.json をローカルで編集...
teams app manifest upload manifest.json <teamsAppId>
# 内容が変更されている場合、バージョンは自動的に増分される
```

更新後、各チームにアプリを再インストールし、キャッシュされたアプリメタデータを消去するために、**Teams を完全に終了して再起動します**（ウィンドウを閉じるだけでは不十分です）。

<details>
<summary>マニフェストの手動更新（CLI を使用しない場合）</summary>

1. `manifest.json` を新しい設定で更新します。
2. **`version` フィールドの値を増やします**（例：`1.0.0` → `1.1.0`）。
3. アイコン（`manifest.json`、`outline.png`、`color.png`）とともにマニフェストを**再度 zip 圧縮**します。
4. 新しい zip をアップロードします：
   - **Teams Admin Center：** Teams apps → Manage apps → 対象のアプリを見つける → Upload new version。
   - **サイドロード：** Teams → Apps → Manage your apps → Upload a custom app。

</details>

## 機能：RSC のみと Graph の比較

### **Teams RSC のみ**の場合（アプリはインストール済み、Graph API アクセス許可なし）

機能するもの：

- チャネルメッセージの**テキスト**コンテンツの読み取り。
- チャネルメッセージの**テキスト**コンテンツの送信。
- **パーソナル（DM）**ファイル添付の受信。

機能しないもの：

- チャネル/グループの**画像またはファイルの内容**（ペイロードには HTML スタブのみが含まれます）。
- SharePoint/OneDrive に保存された添付ファイルのダウンロード。
- ライブ Webhook イベントを超えるメッセージ履歴の読み取り。

### **Teams RSC + Microsoft Graph Application アクセス許可**の場合

追加される機能：

- ホストされたコンテンツ（メッセージに貼り付けられた画像）のダウンロード。
- SharePoint/OneDrive に保存されたファイル添付のダウンロード。
- Graph 経由でのチャネル/チャットメッセージ履歴の読み取り。

### RSC と Graph API の比較

| 機能                      | RSC 権限                     | Graph API                              |
| ----------------------- | -------------------- | ----------------------------------- |
| **リアルタイムメッセージ**  | はい（Webhook 経由）    | いいえ（ポーリングのみ）                   |
| **過去のメッセージ** | いいえ                   | はい（履歴を照会可能）             |
| **セットアップの複雑さ**    | アプリマニフェストのみ    | 管理者の同意とトークンフローが必要 |
| **オフラインでの動作**       | いいえ（実行中である必要あり） | はい（いつでも照会可能）                 |

**要点:** RSC はリアルタイムのリスニング用で、Graph API は履歴へのアクセス用です。オフライン中に見逃したメッセージを取得するには、`ChannelMessage.Read.All` を備えた Graph API が必要です（管理者の同意が必要）。

## Graph を利用したメディアと履歴

使用する Teams のスコープとデータに必要な Microsoft Graph アプリケーション権限のみを有効にします。

1. Entra ID (Azure AD) の **App Registration** → Graph の **Application permissions** を追加:
   - チャネルの添付ファイルとチャネル履歴には `ChannelMessage.Read.All`。
   - グループチャットの添付ファイルとグループチャット履歴には `Chat.Read.All`。
   - 添付ファイルのバイトデータを SharePoint/OneDrive ストレージからダウンロードする必要がある場合は `Files.Read.All`。履歴のみのセットアップでは不要です。
2. テナントに対して **Grant admin consent** を実行します。
3. Teams アプリの **manifest version** を上げ、再アップロードして、**Teams にアプリを再インストール**します。
4. キャッシュされたアプリメタデータを消去するため、**Teams を完全に終了して再起動**します。

### チャネル／グループのファイル復元 (`graphMediaFallback`)

Teams は、ボットに送信する HTML アクティビティからファイルマーカーを削除する場合があります。その場合、Bot Framework アクティビティは通常の HTML メッセージと区別できず、完全な添付ファイル参照は Graph 側のメッセージにのみ存在します。

上記の権限を付与した後、フォールバックを有効にします。

```json5
{
  channels: {
    msteams: {
      graphMediaFallback: true,
    },
  },
}
```

これはチャネルとグループチャットのみに適用されます。通常のメッセージやメンションのみのメッセージを含め、HTML アクティビティから直接ダウンロード可能なメディアが得られなかった場合は、Graph メッセージの照会が 1 回追加されます。既存のインストールで追加の Graph トラフィックや権限エラーが自動的に発生しないよう、デフォルトは `false` です。

**ユーザーのメンション:** すでに会話に参加しているユーザーへの @メンションは、そのまま機能します。**現在の会話に参加していない**ユーザーを動的に検索してメンションするには、`User.Read.All`（Application）権限を追加し、管理者の同意を付与します。

## 既知の制限事項

### Webhook のタイムアウト

Teams は HTTP Webhook 経由でメッセージを配信します。OpenClaw は、その Webhook リスナーに固定の HTTP サーバー
タイムアウトを適用します。非アクティブ状態は 30s、リクエスト全体は 30s、ヘッダー受信は 15s
です。オプションの受信メディアとコンテキスト拡充には、共有の
10 秒のバジェットがあります。SDK は生のアクティビティが永続的に追加された後に処理を返し、
エージェントのターンは独立して処理され、プロアクティブに返信します。リクエストの
処理または永続的な受け入れがトランスポートの時間枠に間に合わない場合、Teams は
アクティビティを再試行することがあり、イングレスのトゥームストーンが重複するイベント ID を拒否します。

### Teams クラウドとサービス URL のサポート

この SDK ベースの Teams パスは、Microsoft Teams パブリッククラウドでライブ検証されています。

受信返信では、受信した Teams SDK のターンコンテキストを使用します。コンテキスト外のプロアクティブ操作（送信、編集、削除、カード、投票、ファイル同意メッセージ、キューに入れられた長時間実行の返信）では、保存された会話参照 `serviceUrl` を使用します。パブリッククラウドのデフォルトでは Teams SDK のパブリッククラウド環境を使用し、パブリック Teams Connector ホスト上の保存済み参照を許可します: `https://smba.trafficmanager.net/`。

パブリッククラウドがデフォルトです。通常のパブリッククラウドボットでは、`channels.msteams.cloud` または `channels.msteams.serviceUrl` を設定する必要はありません。

パブリック以外の Teams クラウドでは、Microsoft が公開している場合、`cloud` と対応するプロアクティブ境界を設定します。

- `channels.msteams.cloud` は、認証、JWT 検証、トークンサービス、Graph スコープに使用する Teams SDK クラウドプリセットを選択します。
- `channels.msteams.serviceUrl` は、プロアクティブな送信、編集、削除、カード、投票、ファイル同意メッセージ、およびキューに入れられた長時間実行の返信の前に、保存された会話参照を検証するための Bot Connector エンドポイント境界を選択します。USGov および DoD SDK クラウドでは必須です。China/21Vianet の場合、OpenClaw は SDK の `China` プリセットを使用し、Azure China Bot Framework チャネルホスト上の保存済み／設定済みサービス URL のみを許可します。

Microsoft は、Teams のプロアクティブメッセージングに関するドキュメントの「[会話を作成する](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/send-proactive-messages?tabs=dotnet#create-the-conversation)」セクションで、グローバルなプロアクティブ Bot Connector エンドポイントを公開しています。受信アクティビティの `serviceUrl` が利用可能な場合はそれを使用し、それ以外の場合は以下の Microsoft の表を使用します。

| Teams 環境 | OpenClaw の設定                                             | プロアクティブ `serviceUrl`                             |
| ----------------- | ----------------------------------------------------------- | -------------------------------------------------- |
| Public            | cloud/serviceUrl の設定は不要                           | `https://smba.trafficmanager.net/teams`            |
| GCC               | `serviceUrl` を設定。独立した Teams SDK クラウドプリセットは存在しない | `https://smba.infra.gcc.teams.microsoft.com/teams` |
| GCC High          | `cloud: "USGov"` + `serviceUrl`                             | `https://smba.infra.gov.teams.microsoft.us/teams`  |
| DoD               | `cloud: "USGovDoD"` + `serviceUrl`                          | `https://smba.infra.dod.teams.microsoft.us/teams`  |
| China/21Vianet    | `cloud: "China"`                                            | 受信アクティビティの `serviceUrl` を使用           |

Microsoft が個別のプロアクティブサービス URL を文書化している一方で、Teams SDK が個別の GCC クラウドプリセットを公開していない GCC の例:

```json
{
  "channels": {
    "msteams": {
      "serviceUrl": "https://smba.infra.gcc.teams.microsoft.com/teams"
    }
  }
}
```

GCC High の例:

```json
{
  "channels": {
    "msteams": {
      "cloud": "USGov",
      "serviceUrl": "https://smba.infra.gov.teams.microsoft.us/teams"
    }
  }
}
```

`channels.msteams.serviceUrl` は、サポートされている Microsoft Teams Bot Connector ホストに制限されます。サービス URL が設定されている場合、OpenClaw はプロアクティブな送信、編集、削除、カード、投票、またはキューに入れられた長時間実行の返信を実行する前に、保存された会話の `serviceUrl` が同じホストを使用していることを確認します。デフォルトのパブリッククラウド設定では、保存された会話がパブリック Teams Connector ホスト外を指している場合、OpenClaw はフェイルクローズします。クラウド／サービス URL の設定を変更した後、会話から新しいメッセージを受信し、保存された会話参照を最新の状態にします。

China/21Vianet には、Microsoft の Teams プロアクティブエンドポイント表に独立したグローバルプロアクティブ `smba` URL がありません。Teams SDK が Azure China の認証、トークン、JWT エンドポイントを使用するように、`cloud: "China"` を設定します。その後のプロアクティブ送信には、受信した China Teams アクティビティから保存された会話参照、または Azure China Bot Framework チャネル境界（`*.botframework.azure.cn`）上で明示的に設定されたサービス URL が必要です。OpenClaw が Azure China Graph エンドポイント経由で Graph リクエストをルーティングするまでは、`cloud: "China"` では Graph ベースの Teams ヘルパーが無効になります。

### 書式設定

Teams の Markdown は Slack や Discord より制限されています。

- 基本的な書式設定は機能します: **太字**、_斜体_、`code`、リンク。
- 複雑な Markdown（表、ネストされたリスト）は正しくレンダリングされない場合があります。
- Adaptive Cards は、投票とセマンティックなプレゼンテーション送信でサポートされています（以下を参照）。

## 設定

主要な設定（チャネル共通のパターンについては [/gateway/configuration](/ja-JP/gateway/configuration) を参照）:

- `channels.msteams.enabled`: チャネルを有効化/無効化します。
- `channels.msteams.appId`、`channels.msteams.appPassword`、`channels.msteams.tenantId`: ボットの認証情報。
- `channels.msteams.cloud`: Teams SDK のクラウド環境（`Public`、`USGov`、`USGovDoD`、または `China`。デフォルトは `Public`）。USGov/DoD SDK クラウドでは `serviceUrl` で設定します。中国では SDK プリセットと、Azure China Bot Framework に保存された会話参照を使用します。Azure China Graph のルーティングが提供されるまでは、Graph ベースのヘルパーは無効になります。
- `channels.msteams.serviceUrl`: SDK のプロアクティブ操作に使用する Bot Connector サービス URL の境界。パブリッククラウドでは SDK のデフォルトを使用します。GCC（`https://smba.infra.gcc.teams.microsoft.com/teams`）、GCC High、または DoD では設定してください。保存された会話参照が 21Vianet 運営の Teams から取得された場合、中国では Azure China Bot Framework のチャネルホストを使用できます。
- `channels.msteams.webhook.port`（デフォルトは `3978`）。
- `channels.msteams.webhook.path`（デフォルトは `/api/messages`）。
- `channels.msteams.dmPolicy`: `pairing | allowlist | open | disabled`（デフォルトは `pairing`）。
- `channels.msteams.allowFrom`: DM の許可リスト（AAD オブジェクト ID を推奨）。Graph にアクセスできる場合、ウィザードはセットアップ中に名前を ID に解決します。
- `channels.msteams.dangerouslyAllowNameMatching`: 可変の UPN/表示名による照合と、チーム名/チャネル名による直接ルーティングを再び有効にする緊急時用トグル。
- `channels.msteams.textChunkLimit`: 送信テキストのチャンクサイズ（文字数、デフォルトは `4000`。設定値がそれより大きい場合でも、上限は `4000`）。
- `channels.msteams.streaming.chunkMode`: 長さによる分割の前に空行（段落境界）で分割するには、`length`（デフォルト）または `newline`。
- `channels.msteams.mediaAllowHosts`: 受信添付ファイルのホスト許可リスト（デフォルトは Microsoft/Teams ドメイン: Graph、SharePoint/OneDrive、Teams CDN、Bot Framework、Azure Media Services）。
- `channels.msteams.mediaAuthAllowHosts`: メディアの再試行時に Authorization ヘッダーを付加するホストの許可リスト（デフォルトは Graph と Bot Framework のホスト）。
- `channels.msteams.graphMediaFallback`: チャネル/グループの HTML にファイルマーカーがない場合に、Graph でメッセージを検索する機能を有効化します（デフォルトは `false`。[チャネル/グループのファイル復旧](#channelgroup-file-recovery-graphmediafallback)を参照）。
- `channels.msteams.mediaMaxMb`: チャネルごとのメディアサイズ上限の上書き値（MB）。未設定の場合は `agents.defaults.mediaMaxMb` にフォールバックします。
- `channels.msteams.requireMention`: チャネル/グループで @メンションを必須にします（デフォルトは `true`）。
- `channels.msteams.replyStyle`: `thread | top-level`（[返信スタイル](#reply-style-threads-vs-posts)を参照）。
- `channels.msteams.teams.<teamId>.replyStyle`: チームごとの上書き。
- `channels.msteams.teams.<teamId>.requireMention`: チームごとの上書き。
- `channels.msteams.teams.<teamId>.tools`: チャネルの上書きがない場合に使用する、チームごとのデフォルトのツールポリシー上書き（`allow`/`deny`/`alsoAllow`）。
- `channels.msteams.teams.<teamId>.toolsBySender`: チームおよび送信者ごとのデフォルトのツールポリシー上書き（`"*"` ワイルドカードをサポート）。
- `channels.msteams.teams.<teamId>.channels.<conversationId>.replyStyle`: チャネルごとの上書き。
- `channels.msteams.teams.<teamId>.channels.<conversationId>.requireMention`: チャネルごとの上書き。
- `channels.msteams.teams.<teamId>.channels.<conversationId>.tools`: チャネルごとのツールポリシー上書き（`allow`/`deny`/`alsoAllow`）。
- `channels.msteams.teams.<teamId>.channels.<conversationId>.toolsBySender`: チャネルおよび送信者ごとのツールポリシー上書き（`"*"` ワイルドカードをサポート）。
- `toolsBySender` のキーには、明示的なプレフィックス `channel:`、`id:`、`e164:`、`username:`、`name:` を使用してください（従来のプレフィックスなしキーは、引き続き `id:` のみにマッピングされます）。
- `channels.msteams.authType`: 認証タイプ - `"secret"`（デフォルト）または `"federated"`。
- `channels.msteams.certificatePath`: PEM 証明書ファイルへのパス（フェデレーション + 証明書認証）。
- `channels.msteams.certificateThumbprint`: 証明書の拇印。指定できますが、認証には必須ではありません。
- `channels.msteams.useManagedIdentity`: マネージド ID 認証を有効にします（フェデレーションモード）。
- `channels.msteams.managedIdentityClientId`: ユーザー割り当てマネージド ID のクライアント ID。
- `channels.msteams.sharePointSiteId`: グループチャット/チャネルでのファイルアップロードに使用する SharePoint サイト ID（[グループチャットでのファイル送信](#sending-files-in-group-chats)を参照）。
- `channels.msteams.welcomeCard`、`channels.msteams.groupWelcomeCard`、`channels.msteams.promptStarters`: 最初の DM/グループ連絡時に表示するウェルカム Adaptive Card と、その推奨プロンプトボタン。
- `channels.msteams.responsePrefix`: 送信する返信の先頭に付加するテキスト。
- `channels.msteams.feedbackEnabled`（デフォルトは `true`）、`channels.msteams.feedbackReflection`（デフォルトは `true`）、`channels.msteams.feedbackReflectionCooldownMs`: 返信に対する高評価/低評価のフィードバックと、否定的なフィードバックに続く振り返り。
- `channels.msteams.sso`、`channels.msteams.delegatedAuth`: SSO ベースのフローに使用する Bot Framework OAuth 接続と委任された Graph スコープ。`sso.enabled: true` には `sso.connectionName` が必要です。

## ルーティングとセッション

- セッションキーは標準のエージェント形式に従います（[/concepts/session](/ja-JP/concepts/session)を参照）。
  - ダイレクトメッセージはメインセッション（`agent:<agentId>:<mainKey>`）を共有します。
  - チャネル/グループメッセージでは会話 ID を使用します。
    - `agent:<agentId>:msteams:channel:<conversationId>`
    - `agent:<agentId>:msteams:group:<conversationId>`

## 返信スタイル: スレッドと投稿

Teams には、同じ基盤データモデル上に 2 種類のチャネル UI スタイルがあります。

| スタイル                 | 説明                                                      | 推奨 `replyStyle` |
| ------------------------ | --------------------------------------------------------- | ------------------------ |
| **投稿**（クラシック）   | メッセージがカードとして表示され、その下にスレッド形式の返信が表示される | `thread`（デフォルト） |
| **スレッド**（Slack 風） | Slack のようにメッセージが直線的に流れる                  | `top-level`       |

**問題:** Teams API では、チャネルがどちらの UI スタイルを使用しているか公開されません。誤った `replyStyle` を使用すると、次のようになります。

- スレッドスタイルのチャネルで `thread` → 返信が不自然にネストされて表示されます。
- 投稿スタイルのチャネルで `top-level` → 返信がスレッド内ではなく、独立した最上位の投稿として表示されます。

**解決策:** チャネルの設定方法に応じて、チャネルごとに `replyStyle` を設定します。

```json5
{
  channels: {
    msteams: {
      replyStyle: "thread",
      teams: {
        "19:abc...@thread.tacv2": {
          channels: {
            "19:xyz...@thread.tacv2": {
              replyStyle: "top-level",
            },
          },
        },
      },
    },
  },
}
```

### 解決の優先順位

ボットがチャネルに返信を送信するとき、`replyStyle` は最も具体的な上書きからデフォルトに向かって解決されます。最初の非 `undefined` 値が適用されます。

1. **チャネルごと** - `channels.msteams.teams.<teamId>.channels.<conversationId>.replyStyle`
2. **チームごと** - `channels.msteams.teams.<teamId>.replyStyle`
3. **グローバル** - `channels.msteams.replyStyle`
4. **暗黙のデフォルト** - `requireMention` から派生:
   - `requireMention: true` → `thread`
   - `requireMention: false` → `top-level`

明示的な `replyStyle` を指定せずに `requireMention: false` をグローバルに設定すると、受信メッセージがスレッドへの返信であっても、投稿スタイルのチャネルではメンションが最上位の投稿として表示されます。予期しない動作を避けるには、グローバル、チーム、またはチャネルレベルで `replyStyle: "thread"` を固定してください。

保存済みのチャネル会話へのプロアクティブ送信（キューに入ったツール呼び出しへの返信、長時間実行されるエージェント）でも、同じチーム/チャネルの解決が適用されます。グループチャットと個人（DM）の会話では、`replyStyle` に関係なく、プロアクティブ送信は常に `top-level` に解決されます。

### スレッドコンテキストの保持

`replyStyle: "thread"` が有効で、チャネルスレッド内からボットが @メンションされた場合、OpenClaw は元のスレッドルートを送信先の会話参照（`19:...@thread.tacv2;messageid=<root>`）に再付加し、返信が同じスレッド内に表示されるようにします。これは、ライブ（ターン内）送信と、Bot Framework のターンコンテキストの有効期限が切れた後に行われるプロアクティブ送信（長時間実行されるエージェント、`mcp__openclaw__message` 経由でキューに入ったツール呼び出しへの返信など）の両方に適用されます。

スレッドルートは、会話参照に保存された `threadId` から取得されます。`threadId` より前の古い保存済み参照では、`activityId`（最後に会話を初期化した受信アクティビティ）にフォールバックするため、既存のデプロイは再初期化せずに引き続き動作します。

`replyStyle: "top-level"` が有効な場合、チャネルスレッドへの受信メッセージには、意図的に新しい最上位の投稿として応答します。スレッドのサフィックスは付加されません。これはスレッドスタイルのチャネルでは正しい動作です。スレッド形式の返信を期待しているのに最上位の投稿になる場合、そのチャネルの `replyStyle` が誤って設定されています。

## 添付ファイルと画像

**現在の制限事項:**

- **DM:** 画像とファイル添付は、Teams ボットのファイル API 経由で動作します。
- **チャネル/グループ:** 添付ファイルは M365 ストレージ（SharePoint/OneDrive）に保存されます。Webhook ペイロードには HTML スタブのみが含まれ、実際のファイルバイトは含まれません。チャネルの添付ファイルをダウンロードするには、**Graph API のアクセス許可が必要です**。
- ファイルを明示的に先に送信する場合は、`action=upload-file` を `media` / `filePath` / `path` とともに使用します。省略可能な `message` は付随するテキスト/コメントになり、`filename`（または `title`）はアップロード名を上書きします。

Graph のアクセス許可がない場合、画像を含むチャネルメッセージはテキストのみで届きます（ボットは画像コンテンツにアクセスできません）。
デフォルトでは、OpenClaw は Microsoft/Teams のホスト名からのみメディアをダウンロードします。`channels.msteams.mediaAllowHosts` で上書きします（任意のホストを許可するには `["*"]` を使用）。
Authorization ヘッダーは、`channels.msteams.mediaAuthAllowHosts` に含まれるホストにのみ付加されます（デフォルトは Graph と Bot Framework のホスト）。このリストは厳格に保ってください（マルチテナントのサフィックスは避けてください）。

## グループチャットでのファイル送信

ボットは、組み込みの FileConsentCard フローを使用して DM でファイルを送信できます。**グループチャット/チャネルでのファイル送信**には、追加のセットアップが必要です。

| コンテキスト             | ファイルの送信方法                             | 必要なセットアップ                                |
| ------------------------ | ---------------------------------------------- | ------------------------------------------------- |
| **DM**                   | FileConsentCard → ユーザーが承認 → ボットがアップロード | そのまま動作                                      |
| **グループチャット/チャネル** | SharePoint にアップロード → ネイティブファイルカード | `sharePointSiteId` + Graph のアクセス許可が必要   |
| **画像（すべてのコンテキスト）** | Base64 エンコードされたインライン            | そのまま動作                                      |

### グループチャットで SharePoint が必要な理由

ボットはアプリケーション ID を使用しますが、Microsoft Graph の `/me` リソースは[サインイン済みユーザーを必要とします](https://learn.microsoft.com/en-us/graph/api/user-get?view=graph-rest-1.0)。グループチャット/チャネルでファイルを送信するため、ボットはファイルを **SharePoint サイト**にアップロードし、共有リンクを作成します。

### セットアップ

1. Entra ID（Azure AD）→ App Registration で **Graph API のアクセス許可を追加**:
   - `Sites.ReadWrite.All`（Application）- SharePoint にファイルをアップロードします。
   - `ChatMember.Read.All`（Application）- グループチャットでのファイル送信に使用する、最小権限のテナント全体のアクセス許可。`Chat.Read.All` も使用でき、グループチャット履歴が有効な場合はすでにこの用途をカバーします。チャットごとの代替手段として、`ChatMember.Read.Chat` の[リソース固有の同意アクセス許可](https://learn.microsoft.com/en-us/microsoftteams/platform/graph-api/rsc/resource-specific-consent)を使用します。
2. テナントの **管理者の同意を付与**します。
3. **SharePoint サイト ID を取得:**

   ```bash
   # Graph Explorer または有効なトークンを使用した curl 経由:
   curl -H "Authorization: Bearer $TOKEN" \
     "https://graph.microsoft.com/v1.0/sites/{hostname}:/{site-path}"

   # 例: "contoso.sharepoint.com/sites/BotFiles" にあるサイトの場合
   curl -H "Authorization: Bearer $TOKEN" \
     "https://graph.microsoft.com/v1.0/sites/contoso.sharepoint.com:/sites/BotFiles"

   # レスポンスに含まれるもの: "id": "contoso.sharepoint.com,guid1,guid2"
   ```

4. **OpenClaw を設定する:**

   ```json5
   {
     channels: {
       msteams: {
         // ... その他の設定 ...
         sharePointSiteId: "contoso.sharepoint.com,guid1,guid2",
       },
     },
   }
   ```

### 共有の動作

| コンテキストと権限                                                  | 共有の動作                                          |
| ----------------------------------------------------------------------- | --------------------------------------------------------- |
| チャネル + `Sites.ReadWrite.All`                                         | 組織全体の共有リンク（組織内の誰でもアクセス可能） |
| グループチャット + `Sites.ReadWrite.All` + サポートされているチャットメンバー読み取り権限 | ユーザーごとの共有リンク（チャットメンバーのみアクセス可能）      |
| サポートされているチャットメンバー読み取り権限がないグループチャット                   | 送信はフェイルクローズする                                         |

ユーザーごとの共有ではチャット参加者のみがファイルにアクセスできるため、より安全です。OpenClaw では、グループチャットに対してメンバー検索が成功する必要があります。タイムアウト、転送エラー、空の結果、Graph API による拒否が発生した場合、アクセス範囲を組織全体に広げるのではなく、送信に失敗します。

### フォールバックの動作

| シナリオ                                                         | 結果                                           |
| ---------------------------------------------------------------- | ------------------------------------------------ |
| グループチャット + ファイル + SharePoint とメンバー権限が設定済み | SharePoint にアップロードし、ネイティブファイルカードを送信    |
| グループチャット + ファイル + SharePoint またはメンバー権限が不足     | 対処可能な設定エラーで失敗      |
| チャネル + ファイル + `sharePointSiteId` が設定済み                   | SharePoint にアップロードし、ネイティブファイルカードを送信    |
| 個人チャット + ファイル                                             | FileConsentCard フロー（SharePoint なしで動作）  |
| 任意のコンテキスト + 画像                                              | Base64 エンコードされたインライン形式（SharePoint なしで動作） |

### ファイルの保存場所

アップロードされたファイルは、設定された SharePoint サイトの既定のドキュメントライブラリ内にある `/OpenClawShared/` フォルダーに保存されます。

## 投票（Adaptive Cards）

OpenClaw は、Teams の投票を Adaptive Cards として送信します（Teams にはネイティブの投票 API がありません）。

- CLI: `openclaw message poll --channel msteams --target conversation:<id> --poll-question "..." --poll-option "..." --poll-option "..."`。
- 投票は、Gateway によって OpenClaw の Plugin 状態用 SQLite の `state/openclaw.sqlite` に記録されます。
- 既存の `msteams-polls.json` ファイルは、実行中の Plugin ではなく `openclaw doctor --fix` によってインポートされます。
- 投票を記録するには、Gateway をオンラインのままにする必要があります。
- 投票結果の概要は自動投稿されず、投票結果用の CLI もまだありません。

## プレゼンテーションカード

`message` ツール、CLI、または通常の返信配信を使用して、セマンティックなプレゼンテーションペイロードを Teams のユーザーまたは会話に送信します。OpenClaw は、汎用プレゼンテーション契約に基づいて、これらを Teams Adaptive Cards としてレンダリングします。

`presentation` パラメーターはセマンティックブロックを受け入れます。`presentation` が指定されている場合、メッセージテキストは省略できます。ボタンは Adaptive Card の送信アクションまたは URL アクションとしてレンダリングされます。選択メニューは Teams レンダラーではネイティブ対応していないため、OpenClaw は配信前に読みやすいテキストへダウングレードします。

**エージェントツール:**

```json5
{
  action: "send",
  channel: "msteams",
  target: "user:<id>",
  presentation: {
    title: "こんにちは",
    blocks: [{ type: "text", text: "こんにちは！" }],
  },
}
```

**CLI:**

```bash
openclaw message send --channel msteams \
  --target "conversation:19:abc...@thread.tacv2" \
  --presentation '{"title":"こんにちは","blocks":[{"type":"text","text":"こんにちは！"}]}'
```

ターゲット形式の詳細については、後述の[ターゲット形式](#target-formats)を参照してください。

## ターゲット形式

MSTeams のターゲットでは、ユーザーと会話を区別するためにプレフィックスを使用します。

| ターゲットの種類         | 形式                           | 例                                                                                                |
| ------------------- | -------------------------------- | ------------------------------------------------------------------------------------------------------ |
| ユーザー（ID 指定）        | `user:<aad-object-id>`           | `user:40a1a0ed-4ff2-4164-a219-55518990c197`                                                            |
| ユーザー（名前指定）      | `user:<display-name>`            | `user:John Smith`（Graph API が必要）                                                                 |
| グループ／チャネル       | `conversation:<conversation-id>` | `conversation:19:abc123...@thread.tacv2`                                                               |
| グループ／チャネル（生形式） | `<conversation-id>`              | `19:abc123...@thread.tacv2`、`19:...@unq.gbl.spaces`、またはプレフィックスなしの `a:`/`8:orgid:`/`29:` Bot Framework ID |

**CLI の例:**

```bash
# ID を指定してユーザーに送信
openclaw message send --channel msteams --target "user:40a1a0ed-..." --message "こんにちは"

# 表示名を指定してユーザーに送信（Graph API 検索が実行される）
openclaw message send --channel msteams --target "user:John Smith" --message "こんにちは"

# グループチャットまたはチャネルに送信
openclaw message send --channel msteams --target "conversation:19:abc...@thread.tacv2" --message "こんにちは"

# 会話にプレゼンテーションカードを送信
openclaw message send --channel msteams --target "conversation:19:abc...@thread.tacv2" \
  --presentation '{"title":"こんにちは","blocks":[{"type":"text","text":"こんにちは"}]}'
```

**エージェントツールの例:**

```json5
{
  action: "send",
  channel: "msteams",
  target: "user:John Smith",
  message: "こんにちは！",
}
```

```json5
{
  action: "send",
  channel: "msteams",
  target: "conversation:19:abc...@thread.tacv2",
  presentation: {
    title: "こんにちは",
    blocks: [{ type: "text", text: "こんにちは" }],
  },
}
```

<Note>
`user:` プレフィックスがない場合、名前は既定でグループまたはチームとして解決されます。表示名で人物をターゲットにする場合は、必ず `user:` を使用してください。
</Note>

## プロアクティブメッセージング

- OpenClaw はその時点で会話参照を保存するため、プロアクティブメッセージを送信できるのは、ユーザーが操作した**後**に限られます。
- `dmPolicy` と許可リストによる制御については、[/gateway/configuration](/ja-JP/gateway/configuration)を参照してください。

## チーム ID とチャネル ID（よくある落とし穴）

Teams の URL にある `groupId` クエリパラメーターは、設定に使用するチーム ID では**ありません**。代わりに、URL パスから ID を抽出してください。

**チーム URL:**

```text
https://teams.microsoft.com/l/team/19%3ABk4j...%40thread.tacv2/conversations?groupId=...
                                    └────────────────────────────┘
                                    チーム会話 ID（URL デコードする）
```

**チャネル URL:**

```text
https://teams.microsoft.com/l/channel/19%3A15bc...%40thread.tacv2/ChannelName?groupId=...
                                      └─────────────────────────┘
                                      チャネル ID（URL デコードする）
```

**設定では:**

- チームキー = `/team/` の後のパスセグメント（URL デコード済み。例: `19:Bk4j...@thread.tacv2`。古いテナントでは `@thread.skype` と表示される場合がありますが、これも有効です）。
- チャネルキー = `/channel/` の後のパスセグメント（URL デコード済み）。
- OpenClaw のルーティングでは、`groupId` クエリパラメーターを**無視**してください。これは Microsoft Entra のグループ ID であり、受信する Teams アクティビティで使用される Bot Framework の会話 ID ではありません。

## プライベートチャネル

プライベートチャネルでのボットのサポートは限定的です。

| 機能                      | 標準チャネル | プライベートチャネル       |
| ---------------------------- | ----------------- | ---------------------- |
| ボットのインストール             | はい               | 限定的                |
| リアルタイムメッセージ（Webhook） | はい               | 動作しない場合がある           |
| RSC 権限              | はい               | 動作が異なる場合がある |
| @メンション                    | はい               | ボットにアクセス可能な場合   |
| Graph API 履歴            | はい               | はい（権限が必要） |

**プライベートチャネルが動作しない場合の回避策:**

1. ボットとのやり取りには標準チャネルを使用してください。
2. DM を使用してください。ユーザーはいつでもボットに直接メッセージを送信できます。
3. 過去の履歴へのアクセスには Graph API を使用してください（`ChannelMessage.Read.All` が必要）。

## トラブルシューティング

### よくある問題

- **チャネルに画像が表示されない:** Graph の権限または管理者の同意がありません。Teams アプリを再インストールし、Teams を完全に終了してから再度開いてください。
- **チャネルで応答がない:** 既定ではメンションが必要です。`channels.msteams.requireMention=false` を設定するか、チーム／チャネルごとに設定してください。
- **バージョンの不一致（Teams に古いマニフェストが引き続き表示される）:** アプリを削除して再度追加し、Teams を完全に終了して更新してください。
- **Webhook からの 401 Unauthorized:** Azure JWT なしで手動テストする場合は想定される結果です。エンドポイントには到達できるものの、認証に失敗したことを意味します。適切にテストするには Azure Web Chat を使用してください。

### マニフェストのアップロードエラー

- **「Icon file cannot be empty」:** マニフェストが参照しているアイコンファイルのサイズが 0 バイトです。有効な PNG アイコン（`outline.png` は 32x32、`color.png` は 192x192）を作成してください。
- **「webApplicationInfo.Id already in use」:** アプリが別のチーム／チャットにまだインストールされています。まずそのアプリを見つけてアンインストールするか、反映されるまで 5-10 分待ってください。
- **アップロード時の「Something went wrong」:** 代わりに [https://admin.teams.microsoft.com](https://admin.teams.microsoft.com) 経由でアップロードし、ブラウザーの DevTools（F12）→ Network タブを開いて、レスポンス本文で実際のエラーを確認してください。
- **サイドロードに失敗する:** 「Upload a custom app」ではなく「Upload an app to your org's app catalog」を試してください。これにより、サイドロードの制限を回避できる場合がよくあります。

### RSC 権限が機能しない

1. `webApplicationInfo.id` がボットの App ID と完全に一致していることを確認してください。
2. アプリを再アップロードし、チーム／チャットに再インストールしてください。
3. 組織の管理者が RSC 権限をブロックしていないか確認してください。
4. 正しいスコープを使用していることを確認してください。チームには `ChannelMessage.Read.Group`、グループチャットには `ChatMessage.Read.Chat` を使用します。

## 参考資料

- [Azure Bot の作成](https://learn.microsoft.com/en-us/azure/bot-service/bot-service-quickstart-registration) - Azure Bot のセットアップガイド
- [Teams Developer Portal](https://dev.teams.microsoft.com/apps) - Teams アプリの作成／管理
- [Teams アプリのマニフェストスキーマ](https://learn.microsoft.com/en-us/microsoftteams/platform/resources/schema/manifest-schema)
- [RSC を使用したチャネルメッセージの受信](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/channel-messages-with-rsc)
- [RSC 権限リファレンス](https://learn.microsoft.com/en-us/microsoftteams/platform/graph-api/rsc/resource-specific-consent)
- [Teams ボットのファイル処理](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/bots-filesv4)（チャネル／グループには Graph が必要）
- [プロアクティブメッセージング](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/send-proactive-messages)
- [@microsoft/teams.cli](https://www.npmjs.com/package/@microsoft/teams.cli) - ボット管理用 Teams CLI

## 関連項目

- [チャンネル概要](/ja-JP/channels) - サポートされているすべてのチャンネル
- [ペアリング](/ja-JP/channels/pairing) - DM 認証とペアリングの流れ
- [グループ](/ja-JP/channels/groups) - グループチャットの動作とメンションによる制御
- [チャンネルルーティング](/ja-JP/channels/channel-routing) - メッセージのセッションルーティング
- [セキュリティ](/ja-JP/gateway/security) - アクセスモデルと堅牢化
