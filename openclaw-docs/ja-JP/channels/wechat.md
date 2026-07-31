---
read_when:
    - OpenClaw を WeChat または Weixin に接続する場合
    - openclaw-weixin チャンネル Plugin をインストール、またはトラブルシューティングしています
    - 外部チャネル Plugin が Gateway と並行してどのように動作するかを理解する必要があります
summary: 外部の openclaw-weixin Plugin を使用した WeChat チャネルのセットアップ
title: WeChat
x-i18n:
    generated_at: "2026-07-26T08:55:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 98faf95f9fb76deedb7df9adf3092083722a77bdd793de98c41a6f715cc0d14a
    source_path: channels/wechat.md
    workflow: 16
---

OpenClaw は、Tencent の外部
`@tencent-weixin/openclaw-weixin` チャンネル Plugin を介して WeChat に接続します。

ステータス: Tencent Weixin チームが保守する外部 Plugin。ダイレクトチャットと
メディアがサポートされています。グループチャットは Plugin のケイパビリティ
メタデータでは公開されていません（ダイレクトチャットのみを宣言しています）。

## 名前

- **WeChat** は、このドキュメントでユーザー向けに使用する名前です。
- **Weixin** は、Tencent のパッケージと Plugin ID で使用される名前です。
- `openclaw-weixin` は OpenClaw のチャンネル ID です（`weixin` と `wechat` はエイリアスとして機能します）。
- `@tencent-weixin/openclaw-weixin` は npm パッケージです。

CLI コマンドと設定パスでは `openclaw-weixin` を使用してください。

## 動作の仕組み

WeChat のコードは OpenClaw のコアリポジトリには含まれていません。OpenClaw は
汎用チャンネル Plugin コントラクトを提供し、外部 Plugin が
WeChat 固有のランタイムを提供します。

1. `openclaw plugins install` は `@tencent-weixin/openclaw-weixin` をインストールします。
2. Gateway は Plugin マニフェストを検出し、Plugin のエントリポイントを読み込みます。
3. Plugin はチャンネル ID `openclaw-weixin` を登録します。
4. `openclaw channels login --channel openclaw-weixin` は QR ログインを開始します。
5. Plugin は OpenClaw の状態ディレクトリ
   （デフォルトでは `~/.openclaw`）にアカウント認証情報を保存します。
6. Gateway が起動すると、Plugin は設定された各アカウントの
   Weixin モニターを開始します。
7. 受信した WeChat メッセージはチャンネルコントラクトを通じて正規化され、
   選択された OpenClaw エージェントにルーティングされ、Plugin の送信パスを通じて返信されます。

この分離は重要です。OpenClaw コアはチャンネル非依存のまま維持されます。WeChat ログイン、
Tencent iLink API 呼び出し、メディアのアップロード／ダウンロード、コンテキストトークン、
アカウント監視は外部 Plugin が担当します。

## インストール

クイックインストール:

```bash
npx -y @tencent-weixin/openclaw-weixin-cli install
```

手動インストール:

```bash
openclaw plugins install "@tencent-weixin/openclaw-weixin"
openclaw config set plugins.entries.openclaw-weixin.enabled true
```

インストール後に Gateway を再起動します。

```bash
openclaw gateway restart
```

## ログイン

Gateway を実行している同じマシンで QR ログインを実行します。

```bash
openclaw channels login --channel openclaw-weixin
```

スマートフォンの WeChat で QR コードをスキャンし、ログインを確認します。スキャンが
成功すると、Plugin はアカウントトークンをローカルに保存します。

別の WeChat アカウントを追加するには、同じログインコマンドをもう一度実行します。複数の
アカウントでは、アカウント、チャンネル、送信者ごとにダイレクトメッセージセッションを分離します。

```bash
openclaw config set session.dmScope per-account-channel-peer
```

## アクセス制御

ダイレクトメッセージでは、チャンネル Plugin 向けの通常の OpenClaw ペアリングおよび
許可リストモデルを使用します。

新しい送信者を承認します。

```bash
openclaw pairing list openclaw-weixin
openclaw pairing approve openclaw-weixin <CODE>
```

アクセス制御モデルの詳細については、[ペアリング](/ja-JP/channels/pairing)を参照してください。

## 互換性

Plugin は起動時にホストの OpenClaw バージョンを確認します。

| Plugin 系列 | OpenClaw バージョン                                                | npm タグ  |
| ----------- | --------------------------------------------------------------- | -------- |
| `2.x`       | `>=2026.5.12`（現在は 2.4.6。初期の 2.x では `>=2026.3.22` も許容） | `latest` |
| `1.x`       | `>=2026.1.0 <2026.3.22`                                         | `legacy` |

Plugin が OpenClaw のバージョンが古すぎると報告した場合は、OpenClaw を更新するか、
レガシー Plugin 系列をインストールします。

```bash
openclaw plugins install @tencent-weixin/openclaw-weixin@legacy
```

## サイドカープロセス

WeChat Plugin は Tencent iLink API を監視しながら、Gateway と並行して
補助処理を実行できます。issue #68451 では、この補助処理のパスによって OpenClaw の
汎用的な古い Gateway クリーンアップにあるバグが顕在化しました。子プロセスが親の
Gateway プロセスをクリーンアップしようとする可能性があり、systemd などのプロセスマネージャー下で
再起動ループが発生していました。

現在の OpenClaw の起動時クリーンアップでは、現在のプロセスとその祖先を除外するため、
チャンネルの補助プロセスが、それを起動した Gateway を終了させることはありません。この修正は
汎用的なものであり、コア内の WeChat 固有のパスではありません。

## トラブルシューティング

インストールとステータスを確認します。

```bash
openclaw plugins list
openclaw channels status --probe
openclaw --version
```

チャンネルがインストール済みと表示されるものの接続されない場合は、Plugin が
有効になっていることを確認して再起動します。

```bash
openclaw config set plugins.entries.openclaw-weixin.enabled true
openclaw gateway restart
```

WeChat を有効にした後に Gateway が繰り返し再起動する場合は、OpenClaw と
Plugin の両方を更新します。

```bash
npm view @tencent-weixin/openclaw-weixin version
openclaw plugins install "@tencent-weixin/openclaw-weixin" --force
openclaw gateway restart
```

起動時に、インストール済みの Plugin パッケージが `requires compiled runtime
output for TypeScript entry` と報告される場合、npm パッケージは OpenClaw に必要なコンパイル済み
JavaScript ランタイムファイルを含めずに公開されています。Plugin の公開者が修正版パッケージを
リリースした後に更新／再インストールするか、Plugin を一時的に無効化／アンインストールしてください。

一時的に無効化する場合:

```bash
openclaw config set plugins.entries.openclaw-weixin.enabled false
openclaw gateway restart
```

## 関連ドキュメント

- チャンネルの概要: [チャットチャンネル](/ja-JP/channels)
- ペアリング: [ペアリング](/ja-JP/channels/pairing)
- チャンネルルーティング: [チャンネルルーティング](/ja-JP/channels/channel-routing)
- Plugin アーキテクチャ: [Plugin アーキテクチャ](/ja-JP/plugins/architecture)
- チャンネル Plugin SDK: [チャンネル Plugin SDK](/ja-JP/plugins/sdk-channel-plugins)
- 外部パッケージ: [@tencent-weixin/openclaw-weixin](https://www.npmjs.com/package/@tencent-weixin/openclaw-weixin)
