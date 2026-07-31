---
read_when:
    - チャンネルメッセージ UI、インタラクティブペイロード、またはネイティブチャンネルレンダラーのリファクタリング
    - メッセージツールの機能、配信ヒント、またはコンテキスト間マーカーの変更
    - Discord Carbon のインポートファンアウトまたはチャンネル Plugin のランタイム遅延読み込みのデバッグ
summary: メッセージの意味的な表現を、チャネル固有の UI レンダラーから分離する。
title: チャネルプレゼンテーションのリファクタリング計画
x-i18n:
    generated_at: "2026-07-26T09:29:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b0f0c4f64e0c503209ac0a5b763b1b5483bf8d55a28ceacffbbcd1337d4371e
    source_path: plan/ui-channels.md
    workflow: 16
---

## ステータス

共有エージェント、CLI、Plugin 機能、および送信配信サーフェスに実装済みです。

- `ReplyPayload.presentation` はセマンティックなメッセージ UI を保持します。
- `ReplyPayload.delivery.pin` は送信済みメッセージのピン留め要求を保持します。
- 共有メッセージアクションでは、プロバイダー固有の `components`、`blocks`、`buttons`、`card` の代わりに、`presentation`、`delivery`、`pin` を公開します。
- コアは、Plugin が宣言した送信機能を通じてプレゼンテーションをレンダリングするか、自動的にデグレードします。
- Discord、Slack、Telegram、Mattermost、MS Teams、Feishu のレンダラーは汎用コントラクトを使用します。
- Discord チャンネルのコントロールプレーンコードは、Carbon ベースの UI コンテナーをインポートしなくなりました。

正式なドキュメントは現在、[メッセージプレゼンテーション](/ja-JP/plugins/message-presentation)にあります。
この計画は実装の履歴的なコンテキストとして保持し、コントラクト、レンダラー、またはフォールバック動作を変更する場合は、
正式なガイドを更新してください。

## 問題

チャンネル UI は現在、互換性のない複数のサーフェスに分かれています。

- コアは、`buildCrossContextComponents` を通じて Discord 形式のクロスコンテキストレンダラーフックを所有しています。
- Discord の `channel.ts` は、`DiscordUiContainer` を通じてネイティブ Carbon UI をインポートできるため、ランタイム UI の依存関係がチャンネル Plugin のコントロールプレーンに取り込まれます。
- エージェントと CLI は、Discord の `components`、Slack の `blocks`、Telegram または Mattermost の `buttons`、Teams または Feishu の `card` など、ネイティブペイロードへのエスケープハッチを公開しています。
- `ReplyPayload.channelData` は、トランスポートヒントとネイティブ UI エンベロープの両方を保持します。
- 汎用の `interactive` モデルは存在しますが、Discord、Slack、Teams、Feishu、LINE、Telegram、Mattermost ですでに使用されている、よりリッチなレイアウトよりも限定的です。

これにより、コアがネイティブ UI の形式を認識することになり、Plugin ランタイムの遅延読み込みが弱まり、同じメッセージ意図を表現するためのプロバイダー固有の方法がエージェントに多く与えられすぎています。

## 目標

- コアは、宣言された機能からメッセージに最適なセマンティックプレゼンテーションを決定します。
- 拡張機能は機能を宣言し、セマンティックプレゼンテーションをネイティブのトランスポートペイロードにレンダリングします。
- Web Control UI はチャットのネイティブ UI とは分離したままにします。
- ネイティブのチャンネルペイロードを、共有エージェントまたは CLI のメッセージサーフェスを通じて公開しません。
- サポートされていないプレゼンテーション機能は、最適なテキスト表現へ自動的にデグレードします。
- 送信済みメッセージのピン留めなどの配信動作は、プレゼンテーションではなく汎用の配信メタデータです。

## 対象外

- `buildCrossContextComponents` の後方互換性シムは提供しません。
- `components`、`blocks`、`buttons`、`card` に対する公開ネイティブエスケープハッチは提供しません。
- コアからチャンネルネイティブ UI ライブラリをインポートしません。
- バンドルされたチャンネル向けのプロバイダー固有 SDK シームは提供しません。

## ターゲットモデル

コアが所有する `presentation` フィールドを `ReplyPayload` に追加します。

```ts
type MessagePresentationTone = "neutral" | "info" | "success" | "warning" | "danger";

type MessagePresentation = {
  tone?: MessagePresentationTone;
  title?: string;
  blocks: MessagePresentationBlock[];
};

type MessagePresentationBlock =
  | { type: "text"; text: string }
  | { type: "context"; text: string }
  | { type: "divider" }
  | { type: "buttons"; buttons: MessagePresentationButton[] }
  | { type: "select"; placeholder?: string; options: MessagePresentationOption[] };

type MessagePresentationButton = {
  label: string;
  value?: string;
  url?: string;
  style?: "primary" | "secondary" | "success" | "danger";
};

type MessagePresentationOption = {
  label: string;
  value: string;
};
```

移行中、`interactive` は `presentation` のサブセットになります。

- `interactive` テキストブロックは `presentation.blocks[].type = "text"` にマッピングされます。
- `interactive` ボタンブロックは `presentation.blocks[].type = "buttons"` にマッピングされます。
- `interactive` 選択ブロックは `presentation.blocks[].type = "select"` にマッピングされます。

外部エージェントと CLI のスキーマは現在 `presentation` を使用します。`interactive` は、既存の返信生成元向けの内部レガシーパーサー／レンダリングヘルパーとして残ります。
公開される生成元向け API では、`interactive` を非推奨として扱います。既存の承認ヘルパーと古い Plugin が引き続き
動作するよう、ランタイムサポートは維持しますが、新しいコードは `presentation` を出力します。

## 配信メタデータ

UI ではない送信動作のために、コアが所有する `delivery` フィールドを追加します。

```ts
type ReplyPayloadDelivery = {
  pin?:
    | boolean
    | {
        enabled: boolean;
        notify?: boolean;
        required?: boolean;
      };
};
```

セマンティクス：

- `delivery.pin = true` は、最初に正常配信されたメッセージをピン留めすることを意味します。
- `notify` のデフォルトは `false` です。
- `required` のデフォルトは `false` です。未対応のチャンネルまたはピン留めの失敗時は、配信を継続することで自動的にデグレードします。
- 既存メッセージ向けの手動 `pin`、`unpin`、`list-pins` メッセージアクションは維持します。

現在の Telegram ACP トピックバインディングは、`channelData.telegram.pin = true` から `delivery.pin = true` に移動する必要があります。

## ランタイム機能コントラクト

プレゼンテーションと配信のレンダーフックは、コントロールプレーンのチャンネル Plugin ではなく、ランタイム送信アダプターに追加します。

```ts
type ChannelPresentationCapabilities = {
  supported: boolean;
  buttons?: boolean;
  selects?: boolean;
  context?: boolean;
  divider?: boolean;
  tones?: MessagePresentationTone[];
  limits?: {
    actions?: {
      maxActions?: number;
      maxActionsPerRow?: number;
      maxRows?: number;
      maxLabelLength?: number;
      maxValueBytes?: number;
      supportsStyles?: boolean;
      supportsDisabled?: boolean;
      supportsLayoutHints?: boolean;
    };
    selects?: {
      maxOptions?: number;
      maxLabelLength?: number;
      maxValueBytes?: number;
    };
    text?: {
      maxLength?: number;
      encoding?: "characters" | "utf8-bytes" | "utf16-units";
      markdownDialect?: "plain" | "markdown" | "html" | "slack-mrkdwn" | "discord-markdown";
      supportsEdit?: boolean;
    };
  };
};

type ChannelDeliveryCapabilities = {
  pinSentMessage?: boolean;
};

type ChannelOutboundAdapter = {
  presentationCapabilities?: ChannelPresentationCapabilities;

  renderPresentation?: (params: {
    payload: ReplyPayload;
    presentation: MessagePresentation;
    ctx: ChannelOutboundSendContext;
  }) => ReplyPayload | null;

  deliveryCapabilities?: ChannelDeliveryCapabilities;

  pinDeliveredMessage?: (params: {
    cfg: OpenClawConfig;
    accountId?: string | null;
    to: string;
    threadId?: string | number | null;
    messageId: string;
    notify: boolean;
  }) => Promise<void>;
};
```

コアの動作：

- 対象チャンネルとランタイムアダプターを解決します。
- プレゼンテーション機能を問い合わせます。
- レンダリング前に、未対応のブロックをデグレードし、汎用の機能制限を
  適用します。
- `renderPresentation` を呼び出します。
- レンダラーが存在しない場合、プレゼンテーションをテキストフォールバックに変換します。
- 送信に成功した後、`delivery.pin` が要求され、かつサポートされている場合は `pinDeliveredMessage` を呼び出します。

## チャンネルマッピング

Discord：

- `presentation` を、ランタイム専用モジュール内の components v2 と Carbon コンテナーにレンダリングします。
- アクセントカラーヘルパーは軽量モジュールに保持します。
- チャンネル Plugin のコントロールプレーンコードから `DiscordUiContainer` のインポートを削除します。

Slack：

- `presentation` を Block Kit にレンダリングします。
- エージェントと CLI の `blocks` 入力を削除します。

Telegram：

- テキスト、コンテキスト、区切り線をテキストとしてレンダリングします。
- 設定済みで対象サーフェスに許可されている場合、アクションと選択をインラインキーボードとしてレンダリングします。
- インラインボタンが無効な場合はテキストフォールバックを使用します。
- ACP トピックのピン留めを `delivery.pin` に移動します。

Mattermost：

- 設定されている場合、アクションをインタラクティブボタンとしてレンダリングします。
- その他のブロックはテキストフォールバックとしてレンダリングします。

MS Teams：

- `presentation` を Adaptive Cards にレンダリングします。
- 手動のピン留め／ピン留め解除／ピン一覧アクションは維持します。
- 対象の会話で Graph のサポートが信頼できる場合は、必要に応じて `pinDeliveredMessage` を実装します。

Feishu：

- `presentation` をインタラクティブカードにレンダリングします。
- 手動のピン留め／ピン留め解除／ピン一覧アクションは維持します。
- API の動作が信頼できる場合は、送信済みメッセージのピン留め用に、必要に応じて `pinDeliveredMessage` を実装します。

LINE：

- 可能な場合、`presentation` を Flex またはテンプレートメッセージにレンダリングします。
- 未対応のブロックはテキストにフォールバックします。
- `channelData` から LINE UI ペイロードを削除します。

プレーンまたは機能が限定されたチャンネル：

- 控えめな書式でプレゼンテーションをテキストに変換します。

## リファクタリング手順

1. `ui-colors.ts` を Carbon ベースの UI から分離し、`extensions/discord/src/channel.ts` から `DiscordUiContainer` を削除する Discord リリース修正を再適用します。
2. `ReplyPayload`、送信ペイロードの正規化、配信サマリー、フックペイロードに、`presentation` と `delivery` を追加します。
3. 限定的な SDK／ランタイムサブパスに `MessagePresentation` スキーマとパーサーヘルパーを追加します。
4. メッセージ機能の `buttons`、`cards`、`components`、`blocks` を、セマンティックプレゼンテーション機能に置き換えます。
5. プレゼンテーションのレンダリングと配信時のピン留め用に、ランタイム送信アダプターフックを追加します。
6. クロスコンテキストのコンポーネント構築を `buildCrossContextPresentation` に置き換えます。
7. `src/infra/outbound/channel-adapters.ts` を削除し、チャンネル Plugin 型から `buildCrossContextComponents` を削除します。
8. ネイティブパラメーターの代わりに `presentation` を付加するよう `maybeApplyCrossContextMarker` を変更します。
9. セマンティックプレゼンテーションと配信メタデータのみを使用するよう、Plugin ディスパッチの送信パスを更新します。
10. エージェントと CLI のネイティブペイロードパラメーター、`components`、`blocks`、`buttons`、`card` を削除します。
11. ネイティブのメッセージツールスキーマを作成する SDK ヘルパーを削除し、プレゼンテーションスキーマヘルパーに置き換えます。
12. `channelData` から UI／ネイティブエンベロープを削除します。残りの各フィールドがレビューされるまでは、トランスポートメタデータのみを保持します。
13. Discord、Slack、Telegram、Mattermost、MS Teams、Feishu、LINE のレンダラーを移行します。
14. メッセージ CLI、チャンネルページ、Plugin SDK、機能クックブックのドキュメントを更新します。
15. Discord および影響を受けるチャンネルエントリーポイントのインポートファンアウトをプロファイリングします。

このリファクタリングでは、共有エージェント、CLI、Plugin 機能、送信アダプターのコントラクトについて、手順 1～11 および 13～14 が実装されています。手順 12 は、プロバイダー専用の `channelData` トランスポートエンベロープを対象とする、より詳細な内部クリーンアップとして残っています。手順 15 は、型／テストゲートを超えて定量化されたインポートファンアウトの数値が必要な場合の、後続の検証として残っています。

## テスト

以下を追加または更新します。

- プレゼンテーション正規化テスト。
- 未対応ブロックに対するプレゼンテーションの自動デグレードテスト。
- Plugin ディスパッチとコア配信パスのクロスコンテキストマーカーテスト。
- Discord、Slack、Telegram、Mattermost、MS Teams、Feishu、LINE、およびテキストフォールバックのチャンネルレンダリングマトリックステスト。
- ネイティブフィールドが削除されたことを証明するメッセージツールスキーマテスト。
- ネイティブフラグが削除されたことを証明する CLI テスト。
- Carbon を対象とする Discord エントリーポイントのインポート遅延読み込み回帰テスト。
- Telegram と汎用フォールバックを対象とする配信ピン留めテスト。

## 未解決の質問

- 最初の実装では `delivery.pin` を Discord、Slack、MS Teams、Feishu に対応させるべきですか、それともまず Telegram のみに対応させるべきですか？
- 最終的に `delivery` に `replyToId`、`replyToCurrent`、`silent`、`audioAsVoice` などの既存フィールドを統合すべきですか、それとも送信後の動作に重点を置いたままにすべきですか？
- プレゼンテーションで画像やファイル参照を直接サポートすべきですか、それとも現時点ではメディアを UI レイアウトから分離したままにすべきですか？

## 関連項目

- [チャンネルの概要](/ja-JP/channels)
- [メッセージのプレゼンテーション](/ja-JP/plugins/message-presentation)
