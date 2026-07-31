---
read_when:
    - OpenClaw 向け Zalo Personal のセットアップ
    - Zalo Personal のログインまたはメッセージフローのデバッグ
summary: ネイティブ zca-js（QR ログイン）による Zalo 個人アカウントのサポート、機能、および設定
title: Zalo 個人版
x-i18n:
    generated_at: "2026-07-26T10:07:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 09cecad1a9a5b34b932c5e68e2b3164b360fb6af1dcd2fd5b5979d1b2a1bd62b
    source_path: channels/zalouser.md
    workflow: 16
---

ステータス: 実験的。この連携は外部 CLI バイナリを使用せず、プロセス内でネイティブ `zca-js` を介して**個人用 Zalo アカウント**を自動化します。

<Warning>
これは非公式の連携であり、アカウントが一時停止または禁止される可能性があります。自己責任で使用してください。
</Warning>

## インストール

Zalo Personal は公式の外部 Plugin であり、コアにはバンドルされていません。使用前にインストールしてください。

```bash
openclaw plugins install @openclaw/zalouser
```

- バージョンを固定: `openclaw plugins install @openclaw/zalouser@<version>`
- ソースチェックアウトから: `openclaw plugins install ./path/to/local/zalouser-plugin`
- 詳細: [Plugin](/ja-JP/tools/plugin)

## クイックセットアップ

1. Plugin をインストールします（上記参照）。
2. ログインします（QR、Gateway マシン上）。
   - `openclaw channels login --channel zalouser`
   - Zalo モバイルアプリで QR コードをスキャンします。
3. チャネルを有効にします。

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      dmPolicy: "pairing",
    },
  },
}
```

4. Gateway を再起動します（またはセットアップを完了します）。
5. DM アクセスのデフォルトはペアリングです。初回の連絡時にペアリングコードを承認してください。

## 概要

- `zca-js` ライブラリを介して、完全にプロセス内で実行されます（外部 `zca`/`openzca` バイナリは不要です）。
- 受信メッセージの受け取りには、ネイティブイベントリスナー（`message`、`error`）を使用します。
- JS API を介して返信を直接送信します（テキスト、メディア、リンク）。
- Zalo Bot API を利用できない「個人アカウント」のユースケース向けに設計されています。

## 命名

チャネル ID は `zalouser` です。これは、**個人用 Zalo ユーザーアカウント**を自動化する非公式機能であることを明示するためです。`zalo` は、将来の公式 Zalo API 連携用として予約されています。

## ID の検索（ディレクトリ）

```bash
openclaw directory self --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory groups list --channel zalouser --query "work"
```

## 制限

- 送信テキストは 2000 文字単位に分割されます（Zalo クライアントの制限）。
- ストリーミングはサポートされていません。
- 処理済みの受信メッセージ ID は 30 日間保持され、アカウントごとに最新 1000 件までに制限されます。

## 受信処理の耐久性

OpenClaw は、各生の `zca-js` メッセージコールバックを処理前に保存します。保留中のメッセージは Gateway の再起動後にアカウントキューから再開され、処理はダイレクトチャットまたはグループごとに直列化されたままになります。

`zca-js` ソケットリスナーは配信確認を公開せず、再接続後に古いメッセージを自動的に再生することもありません。そのため、永続キューが保護できるのは、コールバックが OpenClaw に到達した後のローカルクラッシュ期間です。ソケットから配信されなかったメッセージは復元できません。再生用トゥームストーンは主に、同じ Zalo メッセージ ID を持つコールバックが繰り返された場合に備えるための保護策です。

## アクセス制御（DM）

`channels.zalouser.dmPolicy`: `pairing | allowlist | open | disabled`（デフォルト: `pairing`）。

`channels.zalouser.allowFrom` には、安定した Zalo ユーザー ID を使用してください。静的な送信者アクセスグループ（`accessGroup:<name>`）を参照することもできます。対話型セットアップでは、入力した名前を Plugin のプロセス内連絡先検索によって ID に解決できます。

未加工の名前が設定に残っている場合、起動時に解決されるのは `channels.zalouser.dangerouslyAllowNameMatching: true` が有効な場合のみです。このオプトインがない場合、実行時の送信者チェックは ID のみを使用し、未加工の名前は認可で無視されます。

次の方法で承認します。

- `openclaw pairing list zalouser`
- `openclaw pairing approve zalouser <code>`

## グループアクセス（任意）

- デフォルト: `channels.zalouser.groupPolicy = "allowlist"`（グループには明示的な許可リストエントリが必要です）。
- すべてのグループを開放: `channels.zalouser.groupPolicy = "open"`。
- すべてのグループをブロック: `channels.zalouser.groupPolicy = "disabled"`。
- `groupPolicy = "allowlist"` の場合:
  - `channels.zalouser.groups` のキーには安定したグループ ID を使用してください。名前が起動時に ID に解決されるのは、`channels.zalouser.dangerouslyAllowNameMatching: true` が有効な場合のみです。
  - `channels.zalouser.groupAllowFrom` は、許可されたグループ内でどの送信者がボットを起動できるかを制御します。静的な送信者アクセスグループは `accessGroup:<name>` で参照できます。
- 設定ウィザードでは、グループ許可リストの入力を求めることができます。
- グループ許可リストの照合は、デフォルトでは ID のみを使用します。`channels.zalouser.dangerouslyAllowNameMatching: true` が有効でない限り、解決されていない名前は認証で無視されます。
- `channels.zalouser.dangerouslyAllowNameMatching: true` は、変更可能な起動時の名前解決と実行時のグループ名照合を再び有効にする、緊急時用の互換モードです。
- `groupAllowFrom` は、通常のグループメッセージでは `allowFrom` に**フォールバックしません**。許可リストに登録されたグループでこれを空のままにすると、そのグループはすべての送信者に開放されます。認可済みの制御コマンド（例: `/new`）は例外です。`groupAllowFrom` が空の場合、コマンド送信者のチェックは `allowFrom` にフォールバックします。

例:

```json5
{
  channels: {
    zalouser: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["1471383327500481391"],
      groups: {
        "123456789": { enabled: true },
        "Work Chat": { enabled: true },
      },
    },
  },
}
```

<Note>
`channels.zalouser.groups.<id>.allow` は従来のフィールド名です。現在の設定では `enabled` を使用します。`openclaw doctor --fix` は `allow` を `enabled` に自動的に移行します。
</Note>

### グループメンションによる制御

- `channels.zalouser.groups.<group>.requireMention` は、グループへの返信にメンションが必要かどうかを制御します。
- 解決順序: グループ ID -> `group:<id>` エイリアス -> グループ名/スラッグ（名前ベースの候補が適用されるのは `dangerouslyAllowNameMatching: true` の場合のみ）-> `*` -> デフォルト（`true`）。
- 許可リストに登録されたグループと、グループ開放モードの両方に適用されます。
- ボットのメッセージを引用すると、グループを起動する暗黙的なメンションとして扱われます。
- 認可済みの制御コマンド（例: `/new`）は、メンションによる制御を回避できます。
- メンションが必要なためグループメッセージがスキップされた場合、OpenClaw はそのメッセージを保留中のグループ履歴として保存し、次に処理されるグループメッセージに含めます。
- グループ履歴の上限: `channels.zalouser.historyLimit`、次に `messages.groupChat.historyLimit`、その後はフォールバック値の `50`。

例:

```json5
{
  channels: {
    zalouser: {
      groupPolicy: "allowlist",
      groups: {
        "*": { enabled: true, requireMention: true },
        "Work Chat": { enabled: true, requireMention: false },
      },
    },
  },
}
```

## 複数アカウント

アカウントは OpenClaw の状態内の `zalouser` プロファイルにマッピングされます。例:

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      defaultAccount: "default",
      accounts: {
        work: { enabled: true, profile: "work" },
      },
    },
  },
}
```

## 環境変数

プロファイルの選択には環境変数も使用できます。

| 変数                | 用途                                                                    |
| ------------------ | -------------------------------------------------------------------------- |
| `ZALOUSER_PROFILE` | チャネルまたはアカウント設定に `profile` が設定されていない場合に使用するプロファイル名。 |
| `ZCA_PROFILE`      | 従来のフォールバック。`ZALOUSER_PROFILE` が設定されていない場合にのみ使用されます。             |

プロファイル名は、OpenClaw の状態に保存された Zalo ログイン認証情報を選択します。解決順序:

1. 設定内の明示的な `profile`。
2. `ZALOUSER_PROFILE`。
3. `ZCA_PROFILE`。
4. デフォルト以外のアカウントではアカウント ID、デフォルトアカウントでは `default`。

複数アカウントのセットアップでは、1 つの環境変数によって複数のアカウントが同じログインセッションを共有しないように、設定内の各アカウントに `profile` を指定することを推奨します。

## 入力中表示、リアクション、配信確認

- OpenClaw は返信を送信する前に、入力中イベントを送信します（ベストエフォート）。
- メッセージリアクションアクション `react` は、チャネルアクション内の `zalouser` でサポートされています。
  - メッセージから特定のリアクション絵文字を削除するには、`remove: true` を使用します。
  - リアクションの動作: [リアクション](/ja-JP/tools/reactions)
- イベントメタデータを含む受信メッセージに対して、OpenClaw は配信済みおよび既読の確認を送信します（ベストエフォート）。

## トラブルシューティング

**ログイン状態が保持されない:**

- `openclaw channels status --probe`
- 再ログイン: `openclaw channels logout --channel zalouser && openclaw channels login --channel zalouser`

**許可リストまたはグループ名を解決できない:**

- `allowFrom`/`groupAllowFrom` には数値 ID を使用し、`groups` には安定したグループ ID を使用してください。友人名またはグループ名との完全一致が意図的に必要な場合は、`channels.zalouser.dangerouslyAllowNameMatching: true` を有効にします。

**古い外部 `zca`/CLI ベースのセットアップからアップグレードした場合:**

- 外部 `zca` プロセスを前提とする設定をすべて削除してください。現在、このチャネルは外部 CLI バイナリを使用せず、`zca-js` を介して完全にプロセス内で実行されます。

## 関連項目

- [チャネルの概要](/ja-JP/channels) - サポートされているすべてのチャネル
- [ペアリング](/ja-JP/channels/pairing) - DM の認証とペアリングの流れ
- [グループ](/ja-JP/channels/groups) - グループチャットの動作とメンションによる制御
- [チャネルルーティング](/ja-JP/channels/channel-routing) - メッセージのセッションルーティング
- [セキュリティ](/ja-JP/gateway/security) - アクセスモデルと堅牢化
