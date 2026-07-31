---
read_when:
    - Discord Activity ウィジェットの設定またはトラブルシューティング
summary: Discord Activities 内で自己完結型の OpenClaw HTML ウィジェットを起動する
title: Discord アクティビティ
x-i18n:
    generated_at: "2026-07-26T09:26:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b1bc04443aef89fd514290c3bebdbdd3e9972298b45cae3806bec99344f6d8cd
    source_path: channels/discord-activities.md
    workflow: 16
---

Discord Activities を使用すると、エージェントはインタラクティブで自己完結型の HTML ウィジェットを現在の Discord チャンネルに投稿できます。メッセージには **Open widget** ボタンが含まれ、クリックすると Discord 内でウィジェットが起動します。

この機能はデフォルトで無効です。OpenClaw は、`channels.discord.activities` が存在し、クライアントシークレットを解決できる場合にのみ、Activity HTTP ルート、`show_widget` エージェントツール、起動ボタンハンドラーを登録します。非推奨の `discord_widget` エイリアスは、1 リリースの間、引き続き使用できます。

## 前提条件

- 既存の [OpenClaw Discord ボット](/ja-JP/channels/discord)
- OpenClaw Gateway に到達可能な公開 HTTPS ホスト名
- ボットの Discord アプリケーションに対して Activities と OAuth2 を設定する権限

任意の HTTPS リバースプロキシまたはトンネルを使用できます。名前付き Cloudflare Tunnel を使用すると、Gateway ポートを直接公開せずに安定したホスト名を利用できます。

```yaml
# ~/.cloudflared/config.yml
tunnel: openclaw-discord
credentials-file: /home/you/.cloudflared/TUNNEL-ID.json
ingress:
  - hostname: openclaw.example.com
    service: http://127.0.0.1:18789
  - service: http_status:404
```

```bash
cloudflared tunnel login
cloudflared tunnel create openclaw-discord
cloudflared tunnel route dns openclaw-discord openclaw.example.com
cloudflared tunnel run openclaw-discord
```

通常の Gateway 認証は有効なままにしてください。公開されるのは Activity プレフィックスのみであり、Plugin 自体が OAuth、Activity インスタンスのメンバーシップ、チャンネルの関連付け、セッション、1 回限りのドキュメントケイパビリティを検証します。

## セットアップ

<Steps>
  <Step title="Gateway を HTTPS 経由で公開する">
    トンネルまたはリバースプロキシを起動し、Activities の設定を追加した後に `https://openclaw.example.com/discord/activity/` が Gateway に到達することを確認します。例のホスト名は自分のものに置き換えてください。
  </Step>

  <Step title="Discord で Activities を有効にする">
    [Discord Developer Portal](https://discord.com/developers/applications) で既存のボットアプリケーションを開きます。**Activities** を開いて Activities を有効にし、URL マッピングを作成します。

    - プレフィックス: `ROOT`（`/`）
    - ターゲット: `openclaw.example.com/discord/activity`

    ターゲットは公開ホスト名に `/discord/activity` を加えたもので、末尾にスラッシュは付けません。

  </Step>

  <Step title="OAuth2 クライアントシークレットをコピーする">
    Developer Portal で **OAuth2** を開きます。Discord ではリダイレクト URI が少なくとも 1 つ必要なため、アプリケーションにまだ設定されていない場合は、ループバックアドレスなどのローカルプレースホルダーを追加してください。Activity のリターンフローは Embedded App SDK が処理します。アプリケーションのクライアントシークレットをコピーするか、リセットします。これは認証情報として扱い、チャット、ログ、またはコミットされる設定ファイルに貼り付けないでください。
  </Step>

  <Step title="OpenClaw を設定する">
    ウィジェットを提供する Discord アカウントに、次のブロックを 1 つ追加します。

    ```json5
    {
      channels: {
        discord: {
          token: "${DISCORD_BOT_TOKEN}",
          activities: {
            clientSecret: "${DISCORD_CLIENT_SECRET}",
            // 省略可能。デフォルトでは、起動時に取得したボットアプリケーション ID が使用されます。
            applicationId: "YOUR_DISCORD_APPLICATION_ID",
          },
        },
      },
    }
    ```

    `DISCORD_CLIENT_SECRET` が設定されている場合は、ブロックから `clientSecret` を省略できます。オプトインするには、ブロック自体は必ず残す必要があります。

    通常の Discord アクセス設定は別に適用されます。たとえば、`allowFrom` は引き続きエージェントに DM を送信できるユーザーを制御しますが、チャンネルにすでに投稿されたウィジェットを開けるユーザーは制御しません。

  </Step>

  <Step title="再起動してテストする">
    Gateway を再起動します。Discord の会話で、インタラクティブなウィジェットを表示するようエージェントに依頼します。エージェントが `show_widget` を呼び出したら、投稿されたメッセージの **Open widget** をクリックします。
  </Step>
</Steps>

## セキュリティモデル

- ウィジェットのメタデータが返される前に、OAuth によって Discord ユーザーを識別します。
- Discord の Get Activity Instance API は、OAuth ユーザーが現在の Activity インスタンスに参加していることを確認する必要があります。インスタンスのチャンネルは、ウィジェットが投稿されたチャンネルと一致する必要があります。
- Discord によってそのチャンネルへの参加を許可された全員が、そのチャンネルのウィジェットを開けます。対象者を限定するには、Discord のチャンネル権限を使用してください。OpenClaw のコマンドおよび DM の許可リストは、すでに投稿されたチャンネルコンテンツへのアクセスを付与または削除しません。
- OAuth セッションは 15 分後に期限切れになります。ウィジェットのドキュメントケイパビリティは 60 秒後に期限切れになり、1 回だけ使用できます。
- ウィジェットは 7 日後に期限切れになり、Discord Plugin インスタンスごとに最大 64 個が保持されます。
- ウィジェットの HTML はエージェントによって作成されるため、信頼できるコンテンツとして扱う必要があります。不具合のあるウィジェットに公開されたくないシークレットを埋め込まないでください。
- ウィジェットは、独自の入れ子になったフレーム内を移動できます。`sandbox="allow-scripts"` iframe はトップレベルのナビゲーション、ポップアップ、同一オリジンへのアクセスをブロックし、Content Security Policy はネットワーク接続と外部リソースをブロックします。これらの制御は多層防御であり、ウィジェットを作成したエージェントに対するセキュリティ境界ではありません。
- Activities が無効な場合、`/discord/activity` は一切登録されません。

有効にすると、公開 Activity シェルとトークン交換ルートがトンネル経由で到達可能になります。有効な OAuth セッションと 1 回限りのドキュメントケイパビリティがなければ、ウィジェットの HTML は公開されません。

## トラブルシューティング

### Activity に「Gateway offline」と表示される

- トンネルが実行中であり、Gateway の実際のバインドポートにルーティングされていることを確認する
- Developer Portal のターゲットに `/discord/activity` が含まれていることを確認する
- Discord または OpenClaw の設定を変更した後、Gateway を再起動する
- Activities のクライアントシークレットが見つからないことを示す 1 行の警告が Gateway ログにないか確認する

### Discord で空白ページが開く、または `blocked:csp` が報告される

- URL マッピングが `ROOT` を使用し、2 つ目の `/discord/activity` セグメントを追加していないことを確認する
- シェル、`shell.js`、SDK モジュールのすべてが Discord プロキシ経由で返されることを確認する
- Gateway ログで `/discord/activity/` 配下のリクエストを調べる

ウィジェットのネットワークリクエストは意図的にブロックされます。ウィジェットに必要な CSS、JavaScript、画像、データはすべてインライン化してください。

### 「Widget unavailable」

エージェントが投稿したチャンネルからボタンを起動してください。クリックされると OpenClaw は起動をサーバー側で追跡するため、Discord がボタンのカスタム ID を省略または破損した場合でも、新しい起動記録から正確なウィジェットを特定できます。カスタム ID と起動記録のどちらからも特定できない場合、OpenClaw はそのチャンネルで最後に投稿された有効なウィジェットを開きます。古いウィジェットも、カスタム ID が保持されたボタンから引き続き指定できます。

### 「You cannot launch Activities in this channel」

Discord はフォーラム投稿のスレッドから Activities を起動しません。OpenClaw はそこにウィジェットのメッセージとボタンを投稿できますが、Activity は代わりに通常のテキストチャンネルから起動してください。この制限は OpenClaw ではなく Discord によるものです。
