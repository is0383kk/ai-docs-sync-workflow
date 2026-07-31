---
read_when:
    - チャンネルルーティングまたは受信トレイの動作の変更
summary: チャネルごとのルーティングルール（WhatsApp、Telegram、Discord、Slack）と共有コンテキスト
title: チャンネルルーティング
x-i18n:
    generated_at: "2026-07-26T08:54:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aa03f04a55015bf17e0fe1f3a9bc422875124bb64af5891c898a98bc6917d9e8
    source_path: channels/channel-routing.md
    workflow: 16
---

# チャネルとルーティング

OpenClaw は、返信を**メッセージの送信元チャネルへ返します**。モデルはチャネルを選択しません。ルーティングは決定論的で、ホスト構成によって制御されます。デフォルトの DM スコープでは、すべてのチャネルからのダイレクトメッセージがエージェントの[メインセッション](/concepts/main-session)に集約されます。

## 主要用語

- **チャネル**: `discord`、`googlechat`、`imessage`、`irc`、`line`、`signal`、`slack`、`telegram`、`whatsapp` などのバンドル済みチャネル Plugin、およびインストール済み Plugin のチャネル。`webchat` は内部 WebChat UI チャネルであり、構成可能な送信チャネルではありません。
- **AccountId**: チャネルごとのアカウントインスタンス（サポートされている場合）。
- 任意のチャネルデフォルトアカウント: 送信パスで `accountId` が指定されていない場合に使用するアカウントを `channels.<channel>.defaultAccount` で選択します。
  - 複数アカウント構成で 2 つ以上のアカウントを構成する場合は、明示的なデフォルト（`defaultAccount`、または `default` という名前のアカウント）を設定します。設定しない場合、フォールバックルーティングによって正規化後の最初のアカウント ID が選択されることがあります。
- **AgentId**: 分離されたワークスペースとセッションストア（「頭脳」）。
- **SessionKey**: コンテキストの保存と同時実行制御に使用するバケットキー。

## 送信先プレフィックス

明示的な送信先には、`telegram:123` や `tg:123` などのプロバイダープレフィックスを含められます。コアがそのプレフィックスをチャネル選択のヒントとして扱うのは、選択済みチャネルが `last` または未解決であり、かつ読み込まれた Plugin がそのプレフィックスを公開している場合に限られます。呼び出し元がすでに明示的なチャネルを選択している場合、プロバイダープレフィックスはそのチャネルと一致する必要があります。WhatsApp から `telegram:123` へ配信するようなチャネルをまたぐ組み合わせは、Plugin 固有の送信先正規化より前に失敗します。

`channel:<id>`、`user:<id>`、`room:<id>`、`thread:<id>`、`imessage:<handle>`、`sms:<number>` などの送信先種別およびサービスのプレフィックスは、選択したチャネルの文法内にとどまります。それ自体ではプロバイダーを選択しません。

## セッションキーの形式（例）

デフォルトでは、ダイレクトメッセージはエージェントの**メイン**セッションに集約されます。

- `agent:<agentId>:<mainKey>`（デフォルト: `agent:main:main`）

`session.dmScope` は DM の集約を制御します。`main`（デフォルト）は 1 つのメインセッションを共有し、`per-peer`、`per-channel-peer`、`per-account-channel-peer` は DM を個別のセッションに保持します。ルートバインディングでは、一致するピアのスコープを `bindings[].session.dmScope` で上書きできます。

ダイレクトメッセージの会話履歴がメインと共有されている場合でも、サンドボックスおよびツールポリシーでは、外部 DM に対してアカウント別に派生したダイレクトチャットのランタイムキーを使用します。これにより、チャネルから送信されたメッセージがローカルのメインセッション実行として扱われることを防ぎます。

グループとチャネルは、チャネルごとに分離されたままです。

- グループ: `agent:<agentId>:<channel>:group:<id>`
- チャネル／ルーム: `agent:<agentId>:<channel>:channel:<id>`

スレッド:

- Slack／Discord のスレッドでは、ベースキーに `:thread:<threadId>` が追加されます。
- Telegram のフォーラムトピックでは、グループキーに `:topic:<topicId>` が埋め込まれます。

例:

- `agent:main:telegram:group:-1001234567890:topic:42`
- `agent:main:discord:channel:123456:thread:987654`

## メイン DM ルートの固定

`session.dmScope` が `main` の場合、ダイレクトメッセージは 1 つのメインセッションを共有できます。
セッションの `lastRoute` が所有者以外の DM によって上書きされるのを防ぐため、以下の条件をすべて満たす場合、OpenClaw は `allowFrom` から固定所有者を推定します。

- `allowFrom` にワイルドカード以外のエントリがちょうど 1 つある。
- そのエントリを、そのチャネルの具体的な送信者 ID に正規化できる。
- 受信 DM の送信者が、その固定所有者と一致しない。

この不一致が発生した場合も、OpenClaw は受信セッションのメタデータを記録しますが、メインセッションの `lastRoute` の更新はスキップします。

## 保護された受信記録

保護されたパスで新しい OpenClaw セッションを作成してはならない場合、チャネル Plugin は受信セッションレコードを `createIfMissing: false` としてマークできます。このモードでは、OpenClaw は既存セッションのメタデータと `lastRoute` を更新することがありますが、メッセージが確認されたという理由だけでルート専用のセッションエントリを作成することはありません。

## ルーティング規則（エージェントの選択方法）

ルーティングでは、受信メッセージごとに**1 つのエージェント**を選択します。

1. **ピアの完全一致**（`peer.kind` + `peer.id` を含む `bindings`）。
2. **親ピアの一致**（スレッドの継承）。
3. **ピアのワイルドカード一致**（ピア種別に対する `peer.id: "*"`）。
4. `guildId` + `roles` による**ギルドとロールの一致**（Discord）。
5. `guildId` による**ギルドの一致**（Discord）。
6. `teamId` による**チームの一致**（Slack）。
7. **アカウントの一致**（チャネル上の `accountId`）。
8. **チャネルの一致**（そのチャネル上の任意のアカウント、`accountId: "*"`）。
9. **デフォルトエージェント**（`agents.entries.*.default`。なければリストの先頭エントリ、さらに `main` へフォールバック）。

バインディングに複数の一致フィールド（`peer`、`guildId`、`teamId`、`roles`）が含まれる場合、そのバインディングを適用するには、**指定されたすべてのフィールドが一致する必要があります**。

一致したエージェントによって、使用するワークスペースとセッションストアが決まります。

## ブロードキャストグループ（複数エージェントの実行）

ブロードキャストグループを使用すると、**OpenClaw が通常返信する状況で**、同じピアに対して**複数のエージェント**を実行できます（例: WhatsApp グループで、メンション／アクティベーションゲートを通過した後）。

構成:

```json5
{
  broadcast: {
    strategy: "parallel",
    "120363403215116621@g.us": ["alfred", "baerbel"],
    "+15555550123": ["support", "logger"],
  },
}
```

参照: [ブロードキャストグループ](/ja-JP/channels/broadcast-groups)。

## 構成の概要

- `agents.entries`: 名前付きエージェント定義（ワークスペース、モデルなど）。
- `bindings`: 受信チャネル／アカウント／ピアをエージェントにマッピングします。

例:

```json5
{
  agents: {
    list: [{ id: "support", name: "Support", workspace: "~/.openclaw/workspace-support" }],
  },
  bindings: [
    { match: { channel: "slack", teamId: "T123" }, agentId: "support" },
    { match: { channel: "telegram", peer: { kind: "group", id: "-100123" } }, agentId: "support" },
  ],
}
```

## セッションストレージ

ランタイムセッションの行は、状態ディレクトリ（デフォルトは `~/.openclaw`）内にある各エージェントの SQLite データベースに保存されます。

- `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`

古いインストールには、従来のトランスクリプト JSONL ファイルと、`~/.openclaw/agents/<agentId>/sessions/` 配下の `sessions.json` 行ストアが存在する場合があります。Gateway の起動時と `openclaw doctor --fix` は、使用中の従来の行／履歴を SQLite に自動的にインポートします。明示的な移行証拠が必要な場合は、`openclaw doctor --session-sqlite inspect
--session-sqlite-all-agents` と [Doctor](/ja-JP/cli/doctor#session-sqlite-migration) の検証手順を使用してください。
移行およびオフラインメンテナンスのワークフローでは、`session.store` と `{agentId}` のテンプレートを使用して、従来のストアパスを引き続き選択できます。

Gateway と ACP のセッション検出では、デフォルトの `agents/` ルートおよびテンプレート化された `session.store` ルート配下にある、ディスクベースのエージェントストアもスキャンします。検出されるストアは、解決済みのエージェントルート内に存在し、通常の従来型 `sessions.json` ファイルを使用している必要があります。シンボリックリンクとルート外のパスは無視されます。

## WebChat の動作

WebChat は**選択したエージェント**に接続し、デフォルトではそのエージェントのメインセッションを使用します。そのため、WebChat では、そのエージェントのチャネル横断コンテキストを 1 か所で確認できます。

## 返信コンテキスト

受信した返信には以下が含まれます。

- 利用可能な場合は、`ReplyToId`、`ReplyToBody`、`ReplyToSender`。
- 引用されたコンテキストは、`Body` に `[Replying to ...]` ブロックとして追加されます。

これはすべてのチャネルで一貫しています。

## 関連項目

- [グループ](/ja-JP/channels/groups)
- [ブロードキャストグループ](/ja-JP/channels/broadcast-groups)
- [ペアリング](/ja-JP/channels/pairing)
