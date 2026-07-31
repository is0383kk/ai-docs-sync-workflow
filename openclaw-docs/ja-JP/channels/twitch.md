---
read_when:
    - OpenClaw の Twitch チャット連携の設定
sidebarTitle: Twitch
summary: Twitchチャットボット：インストール、認証情報、アクセス制御、トークンの更新
title: Twitch
x-i18n:
    generated_at: "2026-07-26T08:53:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d827c742ded5fd0b071443dead27b975e2414419b0facb486d7f9c0c9800b060
    source_path: channels/twitch.md
    workflow: 16
---

Twurple クライアントを介して Twitch のチャット（IRC）インターフェースを使用する Twitch チャット対応。OpenClaw は Twitch ボットアカウントとしてサインインし、設定されたアカウントごとに 1 つのチャンネルへ参加して、そのチャンネルで返信します。

## インストール

Twitch は公式 Plugin として提供されており、コアインストールには含まれません。

<Tabs>
  <Tab title="npm レジストリ">
    ```bash
    openclaw plugins install @openclaw/twitch
    ```
  </Tab>
  <Tab title="ローカルチェックアウト">
    ```bash
    openclaw plugins install ./path/to/local/twitch-plugin
    ```
  </Tab>
</Tabs>

`plugins install` は Plugin を登録して有効にします。`openclaw onboard` または `openclaw channels add` で Twitch を選択すると、必要に応じてインストールされます。現在のリリースに追従するにはバージョンなしのパッケージ名を使用し、再現可能なインストールが必要な場合にのみ正確なバージョンを固定してください。OpenClaw 2026.4.10 以降が必要です。

詳細: [Plugin](/ja-JP/tools/plugin)

## クイックセットアップ

<Steps>
  <Step title="Plugin をインストール">
    上記の[インストール](#install)を参照してください。
  </Step>
  <Step title="Twitch ボットアカウントを作成">
    ボット専用の Twitch アカウントを作成します（既存のアカウントも使用できます）。
  </Step>
  <Step title="認証情報を生成">
    [Twitch Token Generator](https://twitchtokengenerator.com/) を使用します。

    - **Bot Token** を選択
    - スコープ `chat:read` と `chat:write` が選択されていることを確認
    - **Client ID** と **Access Token** をコピー

  </Step>
  <Step title="Twitch ユーザー ID を確認">
    [https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/](https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/) を使用して、ユーザー名を Twitch ユーザー ID に変換します。
  </Step>
  <Step title="トークンを設定">
    - 環境変数: `OPENCLAW_TWITCH_ACCESS_TOKEN=...`（デフォルトアカウントのみ）
    - または設定: `channels.twitch.accessToken`

    両方が設定されている場合は、設定が優先されます（環境変数はデフォルトアカウントのフォールバックとしてのみ使用されます）。

  </Step>
  <Step title="Gateway を起動">
    ```bash
    openclaw gateway run
    ```
  </Step>
</Steps>

<Warning>
許可されていないユーザーによるボットの起動を防ぐため、アクセス制御（`allowFrom` または `allowedRoles`）を追加してください。`requireMention` のデフォルトは `true` です。
</Warning>

最小構成:

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw", // ボットの Twitch アカウント（認証に使用）
      accessToken: "oauth:abc123...", // OAuth アクセストークン（または環境変数 OPENCLAW_TWITCH_ACCESS_TOKEN を使用）
      clientId: "xyz789...", // Token Generator から取得した Client ID
      channel: "yourchannel", // 参加する Twitch チャンネルのチャット（必須）
      allowFrom: ["123456789"], // （推奨）自分の Twitch ユーザー ID のみ
    },
  },
}
```

## 概要

- Gateway が所有する Twitch チャンネル。
- 決定論的ルーティング: 返信は常に、メッセージの送信元である Twitch チャンネルに返されます。
- 参加した各チャンネルは、分離されたグループセッションキー `agent:<agentId>:twitch:group:<channel>` に対応します。
- `username` はボットのアカウント（認証を行う側）、`channel` は参加するチャットルームです。各アカウントエントリは、ちょうど 1 つのチャンネルに参加します。
- トークンは `oauth:` プレフィックスの有無にかかわらず機能します。OpenClaw はどちらの形式も正規化します（セットアップウィザードでは `oauth:` 形式を想定しています）。

## 受信メッセージの耐久性

OpenClaw は、受け入れた各 Twitch チャットメッセージを通常のディスパッチ前に永続キューへ追加します。保留中または再試行可能なメッセージは Gateway の再起動後も維持され、設定されたチャンネル内で直列処理されます。また、有効または保持中の完了レコードが存在する間は、Twitch のメッセージ ID を使用してキューエントリの重複を抑止します。

クライアントが `PRIVMSG` を受け入れた後、Twitch チャットがそれを再送することはありません。これにより、ローカルでの受け入れからディスパッチまでの間に発生するクラッシュには対処できますが、永続キューへの受け入れ前に取りこぼしたメッセージは復元できません。キューへの追加自体が失敗した場合、OpenClaw はその失敗をログに記録します。再接続しても、Twitch にそのメッセージの再送は要求しません。

## トークンの更新（任意）

[Twitch Token Generator](https://twitchtokengenerator.com/) のトークンは OpenClaw では更新できません。期限切れになったら再生成してください（有効期間は数時間で、アプリの登録は不要です）。

自動更新を使用するには、[Twitch Developer Console](https://dev.twitch.tv/console) で独自のアプリを作成し、次を追加します。

```json5
{
  channels: {
    twitch: {
      clientSecret: "your_client_secret",
      refreshToken: "your_refresh_token",
    },
  },
}
```

両方を設定すると、Plugin は有効期限前にトークンを更新する認証プロバイダーを使用し、更新のたびにログを記録します。`refreshToken` がない場合は `token refresh disabled (no refresh token)` をログに記録し、`clientSecret` がない場合は静的な（更新されない）トークンにフォールバックします。

## 複数アカウント対応

アカウントごとの認証情報とともに `channels.twitch.accounts` を使用します。共通のパターンについては、[設定](/ja-JP/gateway/configuration)を参照してください。

例（2 つのチャンネルで 1 つのボットアカウントを使用）:

```json5
{
  channels: {
    twitch: {
      accounts: {
        channel1: {
          username: "openclaw",
          accessToken: "oauth:abc123...",
          clientId: "xyz789...",
          channel: "yourchannel",
        },
        channel2: {
          username: "openclaw",
          accessToken: "oauth:def456...",
          clientId: "uvw012...",
          channel: "secondchannel",
        },
      },
    },
  },
}
```

<Note>
各アカウントエントリには、固有の `accessToken` が必要です（環境変数はデフォルトアカウントのみを対象とします）。1 つのアカウントが参加できるのはちょうど 1 つのチャンネルであるため、2 つのチャンネルに参加するには 2 つのアカウントが必要です。`channels.twitch.defaultAccount` でデフォルトにするアカウントを選択します。
</Note>

## アクセス制御

`allowFrom` は Twitch ユーザー ID の厳格な許可リストです。これが設定されている場合、`allowedRoles` は無視されます。代わりにロールベースのアクセスを使用するには、`allowFrom` を未設定のままにします。

**利用可能なロール:** `"moderator"`、`"owner"`、`"vip"`、`"subscriber"`、`"all"`。

<Tabs>
  <Tab title="ユーザー ID 許可リスト（最も安全）">
    ```json5
    {
      channels: {
        twitch: {
          accounts: {
            default: {
              allowFrom: ["123456789", "987654321"],
            },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="ロールベース">
    ```json5
    {
      channels: {
        twitch: {
          accounts: {
            default: {
              allowedRoles: ["moderator", "vip"],
            },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="@メンション要件を無効化">
    デフォルトでは、`requireMention` は `true` です。許可されたすべてのメッセージに応答するには、次のように設定します。

    ```json5
    {
      channels: {
        twitch: {
          accounts: {
            default: {
              requireMention: false,
            },
          },
        },
      },
    }
    ```

  </Tab>
</Tabs>

<Note>
**ユーザー ID を使用する理由** ユーザー名は変更できるため、なりすましが可能です。ユーザー ID は永続的です。

[ユーザー名から ID への変換ツール](https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/)で自分の ID を確認できます。
</Note>

## トラブルシューティング

最初に、診断コマンドを実行します。

```bash
openclaw doctor
openclaw channels status --probe
```

<AccordionGroup>
  <Accordion title="ボットがメッセージに応答しない">
    - **アクセス制御を確認:** 自分のユーザー ID が `allowFrom` に含まれていることを確認します。または、テストのため一時的に `allowFrom` を削除し、`allowedRoles: ["all"]` を設定します。
    - **メンションゲートを確認:** `requireMention: true`（デフォルト）では、メッセージ内でボットのユーザー名を @メンションする必要があります。
    - **ボットがチャンネルに参加していることを確認:** ボットは `channel` で指定されたチャンネルにのみ参加します。

  </Accordion>
  <Accordion title="トークンの問題">
    「Failed to connect」または認証エラーが発生する場合:

    - `accessToken` が OAuth アクセストークンの値であることを確認します（`oauth:` プレフィックスは任意です）
    - トークンに `chat:read` と `chat:write` のスコープがあることを確認します
    - トークン更新を使用している場合は、`clientSecret` と `refreshToken` が設定されていることを確認します

  </Accordion>
  <Accordion title="トークンの更新が機能しない">
    ログで更新イベントを確認します。

    ```text
    mybot に環境変数のトークンソースを使用
    ユーザー 123456 のアクセストークンを更新（14400s 後に期限切れ）
    ```

    `token refresh disabled (no refresh token)` が表示される場合:

    - `clientSecret` が指定されていることを確認します
    - `refreshToken` が指定されていることを確認します

  </Accordion>
</AccordionGroup>

## 設定

### アカウント設定

<ParamField path="username" type="string" required>
  ボットのユーザー名（認証するアカウント）。
</ParamField>
<ParamField path="accessToken" type="string" required>
  `chat:read` と `chat:write` を持つ OAuth アクセストークン（デフォルトアカウントでは設定または環境変数）。
</ParamField>
<ParamField path="clientId" type="string" required>
  Twitch Client ID（Token Generator または独自のアプリから取得）。スキーマ上は任意ですが、接続には必須です。
</ParamField>
<ParamField path="channel" type="string" required>
  参加するチャンネル。
</ParamField>
<ParamField path="enabled" type="boolean" default="true">
  このアカウントを有効にします。
</ParamField>
<ParamField path="clientSecret" type="string">
  任意: トークンの自動更新に使用します。
</ParamField>
<ParamField path="refreshToken" type="string">
  任意: トークンの自動更新に使用します。
</ParamField>
<ParamField path="expiresIn" type="number">
  トークンの有効期限（秒単位、更新追跡用）。
</ParamField>
<ParamField path="obtainmentTimestamp" type="number">
  トークンを取得した時点のタイムスタンプ（更新追跡用）。
</ParamField>
<ParamField path="allowFrom" type="string[]">
  ユーザー ID の許可リスト。設定されている場合、ロールは無視されます。
</ParamField>
<ParamField path="allowedRoles" type='Array<"moderator" | "owner" | "vip" | "subscriber" | "all">'>
  ロールベースのアクセス制御。
</ParamField>
<ParamField path="requireMention" type="boolean" default="true">
  ボットを起動するために @メンションを必須にします。
</ParamField>
<ParamField path="responsePrefix" type="string">
  このアカウントの送信応答プレフィックスを上書きします。
</ParamField>

### プロバイダーオプション

- `channels.twitch.enabled` - チャンネルの起動を有効化または無効化
- `channels.twitch.username` / `accessToken` / `clientId` / `channel` - 簡略化された単一アカウント設定（暗黙的な `default` アカウント。`accounts.default` より優先）
- `channels.twitch.accounts.<accountName>` - 複数アカウント設定（上記のすべてのアカウントフィールド）
- `channels.twitch.defaultAccount` - デフォルトにするアカウント名
- `channels.twitch.markdown.tables` - Markdown テーブルのレンダリングモード（`off` | `bullets` | `code` | `block`）

完全な例:

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw",
      accessToken: "oauth:abc123...",
      clientId: "xyz789...",
      channel: "yourchannel",
      clientSecret: "secret123...",
      refreshToken: "refresh456...",
      allowFrom: ["123456789"],
      accounts: {
        second: {
          username: "mybot",
          accessToken: "oauth:def456...",
          clientId: "uvw012...",
          channel: "your_channel",
          enabled: true,
          expiresIn: 14400,
          obtainmentTimestamp: 1706092800000,
          allowedRoles: ["moderator"],
        },
      },
    },
  },
}
```

## ツールアクション

エージェントは、メッセージツールの `send` アクションを介して Twitch メッセージを送信できます。

```json5
{
  channel: "twitch",
  action: "send",
  to: "#mychannel",
  message: "Hello Twitch!",
}
```

`to` は任意で、デフォルトではアカウントに設定された `channel` が使用されます。

## 安全性と運用

- **トークンをパスワードのように扱う** - トークンを git にコミットしないでください。
- 長時間稼働するボットには、**トークンの自動更新を使用する**。
- アクセス制御には、ユーザー名ではなく**ユーザー ID の許可リストを使用する**。
- トークン更新イベントと接続状態を確認するため、**ログを監視する**。
- **トークンのスコープを最小限にする** - `chat:read` と `chat:write` のみをリクエストしてください。
- **解決しない場合**: 他のプロセスがセッションを所有していないことを確認してから、Gateway を再起動してください。

## 制限

- 1 メッセージあたり**500 文字**。それより長い返信は、単語の境界で分割されます。
- 送信前に Markdown は除去されます（Twitch チャットはプレーンテキストで、改行はスペースに変換されます）。
- OpenClaw 自体はレート制限を追加しません。Twitch のレート制限は Twurple チャットクライアントが処理します。

## 関連項目

- [チャンネルルーティング](/ja-JP/channels/channel-routing) — メッセージのセッションルーティング
- [チャンネルの概要](/ja-JP/channels) — サポートされているすべてのチャンネル
- [グループ](/ja-JP/channels/groups) — グループチャットの動作とメンションによる制御
- [ペアリング](/ja-JP/channels/pairing) — DM の認証とペアリングの流れ
- [セキュリティ](/ja-JP/gateway/security) — アクセスモデルと堅牢化
