---
read_when:
    - メッセージカード、グラフ、表、ボタン、または選択コントロールのレンダリングの追加または変更
    - リッチな送信メッセージをサポートするチャンネル Plugin の構築
    - メッセージツールの表示機能または配信機能の変更
    - プロバイダー固有のカード／ブロック／コンポーネントのレンダリング回帰をデバッグする
summary: チャンネル Plugin 向けのセマンティックなメッセージカード、グラフ、表、コントロール、フォールバックテキスト、配信ヒント
title: メッセージ表示
x-i18n:
    generated_at: "2026-07-26T09:51:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1fce3874c99627eb87ceb83aebe381b8a8466722703ec6322c609f187d15d9ae
    source_path: plugins/message-presentation.md
    workflow: 16
---

メッセージプレゼンテーションは、リッチな送信チャット UI のための OpenClaw 共通コントラクトです。
これにより、エージェント、CLI コマンド、承認フロー、plugins はメッセージの意図を一度記述するだけで済み、各チャネル plugin が可能な限り最適なネイティブ形式でレンダリングします。

テキストセクション、小さなコンテキスト／フッターテキスト、区切り線、チャート、テーブル、ボタン、選択メニュー、カードのタイトル／トーンなど、移植可能なメッセージ UI にはプレゼンテーションを使用します。

Discord `components`、Slack
`blocks`、Telegram `buttons`、Teams `card`、Feishu `card` など、プロバイダー固有の新しいフィールドを共有メッセージツールに追加しないでください。これらはチャネル plugin が所有するレンダラー出力です。

## コントラクト

Plugin の作成者は、次の場所から公開コントラクトをインポートします。

```ts
import type {
  MessagePresentation,
  ReplyPayloadDelivery,
} from "openclaw/plugin-sdk/interactive-runtime";
```

形式：

```ts
type MessagePresentation = {
  title?: string;
  tone?: "neutral" | "info" | "success" | "warning" | "danger";
  blocks: MessagePresentationBlock[];
};

type MessagePresentationBlock =
  | { type: "text"; text: string }
  | { type: "context"; text: string }
  | { type: "divider" }
  | { type: "buttons"; buttons: MessagePresentationButton[] }
  | { type: "select"; placeholder?: string; options: MessagePresentationOption[] }
  | {
      type: "chart";
      chartType: "pie";
      title: string;
      segments: Array<{ label: string; value: number }>;
    }
  | {
      type: "chart";
      chartType: "bar" | "area" | "line";
      title: string;
      categories: string[];
      series: Array<{ name: string; values: number[] }>;
      xLabel?: string;
      yLabel?: string;
    }
  | {
      type: "table";
      caption: string;
      headers: string[];
      rows: Array<Array<string | number>>;
      rowHeaderColumnIndex?: number;
    };

type MessagePresentationAction =
  | { type: "command"; command: string }
  | { type: "callback"; value: string }
  | {
      type: "approval";
      approvalId: string;
      approvalKind: "exec" | "plugin";
      decision: "allow-once" | "allow-always" | "deny";
    }
  | {
      type: "question";
      questionId: string;
      optionValue: string;
    }
  | { type: "url"; url: string }
  | {
      type: "web-app";
      url: string;
      widgetId?: string;
    }
  | {
      type: "web-app";
      url?: string;
      widgetId: string;
    };

type MessagePresentationButton = {
  label: string;
  action?: MessagePresentationAction;
  /** 従来のコールバック値。新しいコントロールでは action を優先してください。 */
  value?: string;
  /** @deprecated type が "url" の action を使用してください。 */
  url?: string;
  /** @deprecated type が "web-app" の action を使用してください。 */
  webApp?: { url: string };
  /** @deprecated type が "web-app" の action を使用してください。 */
  web_app?: { url: string };
  priority?: number;
  disabled?: boolean;
  reusable?: boolean;
  style?: "primary" | "secondary" | "success" | "danger";
};

type MessagePresentationOption = {
  label: string;
  action?: Extract<MessagePresentationAction, { type: "command" | "callback" }>;
  /** 従来のコールバック値。新しいコントロールでは action を優先してください。 */
  value?: string;
};

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

ボタンのセマンティクス：

- `action.type: "command"` は、コアのコマンドパスを通じてネイティブのスラッシュコマンドを実行します。
  組み込みのコマンドボタンとメニューにはこれを使用します。
- `action.type: "callback"` は、チャネルのインタラクションパスを通じて不透明な plugin データを伝達します。
  チャネル plugins は、コールバックデータをスラッシュコマンドとして再解釈してはなりません。
- `action.type: "approval"` は、単一の永続的なオペレーター承認、その明示的な
  `exec` または `plugin` の種別、および要求された判断を識別します。チャネル plugins は、そのアクションをトランスポート専用のコールバックにエンコードし、承認サービスを通じて解決します。`/approve` コマンドテキストを解析したり、ID から種別を推測したりしてはなりません。
- `action.type: "question"` は、実行時に作成された有効な
  `ask_user` 質問に対する単一の選択肢を識別します。`approval` と同様、これは OpenClaw ランタイムアクションです。エージェントと plugins は質問 ID を生成してはなりません。Telegram、Discord、Slack はこれをトランスポート専用のネイティブコールバックにマッピングし、Gateway を通じて選択を解決します。質問が回答済み、期限切れ、またはキャンセル済みになると、これらのチャネルは配信済みメッセージを編集し、そのアクションを削除して、最終ステータスを追記します。WhatsApp、Signal、iMessage は、最大 4 つの単一選択肢を `1️⃣` から `4️⃣` のリアクションとしてレンダリングします。その他の形式の質問はラベルテキストにフォールバックし、ユーザーはプレーンテキストの返信で回答できます。
- `action.type: "url"` は通常のリンクを開きます。
- `action.type: "web-app"` はチャネルネイティブの Web アプリを起動します。URL ベースのアプリには `url` を、起動メカニズムをチャネルが所有する OpenClaw ホスト型ウィジェットには `widgetId` を設定します。少なくとも一方が必要です。両方が存在する場合、チャネルはネイティブのホスト型ウィジェット起動を優先し、そのメカニズムを使用できない場所では URL を使用できます。
- `value` は、従来の不透明なコールバック値です。新しいコントロールでは `action` を使用し、チャネル plugins がテキストから推測せずにコマンドとコールバックをマッピングできるようにしてください。
- `url`、`webApp`、`web_app` は、非推奨の境界入力として引き続き受け入れられます。
  ノーマライザーはこれらのフィールドを保持するため、レンダラーはリリース済みの従来のセマンティクスと明示的に型付けされたアクションを区別できます。新しい生成元では `action` を使用してください。
- `label` は必須であり、テキストフォールバックでも使用されます。
- `style` は参考情報です。レンダラーは未対応のスタイルを安全なデフォルトにマッピングし、送信を失敗させないようにする必要があります。
- `priority` は任意です。チャネルがアクション数の上限を通知しており、コントロールを削除する必要がある場合、コアは優先度の高いボタンを先に保持し、優先度が同じボタン間では元の順序を維持します。すべてのコントロールが収まる場合、作成時の順序が維持されます。
- `disabled` は任意です。チャネルは `supportsDisabled` で明示的に対応を宣言する必要があります。それ以外の場合、コアは無効化されたコントロールを非インタラクティブなフォールバックテキストに変換します。無効化されたボタンは、`command` アクションを持つ場合でも、フォールバックテキストでは常にラベルのみでレンダリングされます。
- `reusable` は任意です。再利用可能なネイティブコールバックに対応するチャネルは、インタラクションが成功した後もアクションを利用可能な状態に保つことができます。更新、調査、詳細表示など、反復可能または冪等なアクションに使用してください。通常の一度限りの承認や破壊的アクションでは設定しないでください。

選択のセマンティクス：

- `options[].action` は `command` または `callback` のみを受け入れます。承認アクションとリンクアクションはボタン専用です。
- `options[].value` は、従来の選択済みアプリケーション値です。
- `placeholder` は参考情報であり、ネイティブの選択機能に対応していないチャネルでは無視される場合があります。
- チャネルが選択機能に対応していない場合、フォールバックテキストにラベルの一覧が表示されます。

チャートのセマンティクス：

- `pie` には正のセグメント値が必要です。
- `bar`、`area`、`line` は、単一の順序付き `categories` 配列を使用します。各系列は、カテゴリごとに有限値を同じ順序でちょうど 1 つ指定します。
- カテゴリラベルと系列名は一意でなければなりません。無効または不完全なチャートブロックは、データを暗黙に変更するのではなく、正規化時に削除されます。
- ネイティブのチャートレンダリングは、`presentationCapabilities.charts` を通じて明示的に有効化します。
  その他のチャネルには、チャートタイトル、軸、カテゴリ、系列、値が決定論的なテキストとして提供されます。これはアクセシビリティ用のフォールバックでもあります。

テーブルのセマンティクス：

- `caption` は必須の短い見出しです。`headers` には、一意で空でない列ラベルを少なくとも 1 つ含める必要があります。
- `rows` には少なくとも 1 行を含める必要があります。各行はヘッダーごとにちょうど 1 つのセルを持ち、各セルは空でない文字列または有限数でなければなりません。
- `rowHeaderColumnIndex` は、ネイティブレンダラーがセルを行ヘッダーとして公開する列を識別する、任意のゼロベースインデックスです。
- テーブルの正規化は不可分に行われます。キャプション、ヘッダー、行幅、セル、または行ヘッダーインデックスが無効な場合、データを切り詰めたり修復したりせず、テーブルブロック全体が削除されます。
- ネイティブのテーブルレンダリングは、`presentationCapabilities.tables` を通じて明示的に有効化します。
  その他のチャネルには、キャプションとすべての行が決定論的な線形テキストとして提供され、内部の空白はまとめられます。

  ```text
  進行中のパイプライン（テーブル）
  - アカウント: Acme; ステージ: 受注; ARR: 125000
  - アカウント: Globex; ステージ: レビュー; ARR: 82000
  ```

独立した `report` 判別子はありません。`title`、
`tone`、`text`、`context`、`chart`、`table`、およびアクションブロックからレポートを構成します。これにより、各ブロックを独立してレンダリングでき、レポート全体にも同じ決定論的なテキストフォールバックが提供されます。

## 生成例

シンプルなカード：

```json
{
  "title": "デプロイの承認",
  "tone": "warning",
  "blocks": [
    { "type": "text", "text": "カナリアを昇格する準備ができました。" },
    { "type": "context", "text": "ビルド 1234、ステージングに合格しました。" },
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "承認",
          "action": { "type": "callback", "value": "deploy:approve" },
          "style": "success"
        },
        {
          "label": "却下",
          "action": { "type": "callback", "value": "deploy:decline" },
          "style": "danger"
        }
      ]
    }
  ]
}
```

URL のみのリンクボタン：

```json
{
  "blocks": [
    { "type": "text", "text": "リリースノートの準備ができました。" },
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "ノートを開く",
          "action": { "type": "url", "url": "https://example.com/release" }
        }
      ]
    }
  ]
}
```

Telegram Mini App ボタン：

```json
{
  "blocks": [
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "起動",
          "action": { "type": "web-app", "url": "https://example.com/app" }
        }
      ]
    }
  ]
}
```

選択メニュー：

```json
{
  "title": "環境を選択",
  "blocks": [
    {
      "type": "select",
      "placeholder": "環境",
      "options": [
        { "label": "カナリア", "value": "env:canary" },
        { "label": "本番環境", "value": "env:prod" }
      ]
    }
  ]
}
```

チャート：

```json
{
  "blocks": [
    {
      "type": "chart",
      "chartType": "line",
      "title": "四半期売上高",
      "categories": ["第1四半期", "第2四半期", "第3四半期"],
      "series": [
        { "name": "製品", "values": [120, 145, 138] },
        { "name": "サービス", "values": [80, 95, 104] }
      ],
      "xLabel": "四半期",
      "yLabel": "売上高"
    }
  ]
}
```

テーブルレポート：

```json
{
  "title": "パイプラインレポート",
  "tone": "info",
  "blocks": [
    { "type": "text", "text": "ステージ別の現在の商談です。" },
    {
      "type": "table",
      "caption": "進行中のパイプライン",
      "headers": ["アカウント", "ステージ", "ARR"],
      "rows": [
        ["Acme", "受注", 125000],
        ["Globex", "レビュー", 82000]
      ],
      "rowHeaderColumnIndex": 0
    },
    { "type": "context", "text": "CRM スナップショットから更新されました。" }
  ]
}
```

CLI 送信：

```bash
openclaw message send --channel slack \
  --target channel:C123 \
  --message "デプロイの承認" \
  --presentation '{"title":"デプロイの承認","tone":"warning","blocks":[{"type":"text","text":"カナリアの準備ができました。"},{"type":"buttons","buttons":[{"label":"承認","value":"deploy:approve","style":"success"},{"label":"却下","value":"deploy:decline","style":"danger"}]}]}'
```

ピン留め配信：

```bash
openclaw message send --channel telegram \
  --target -1001234567890 \
  --message "トピックを開きました" \
  --pin
```

明示的な JSON を使用したピン留め配信:

```json
{
  "pin": {
    "enabled": true,
    "notify": true,
    "required": false
  }
}
```

## レンダラーのコントラクト

チャンネル Plugin は、送信アダプターでレンダリング対応を宣言します:

```ts
const adapter: ChannelOutboundAdapter = {
  deliveryMode: "direct",
  presentationCapabilities: {
    supported: true,
    buttons: true,
    selects: true,
    context: true,
    divider: true,
    charts: false,
    tables: false,
    limits: {
      actions: {
        maxActions: 25,
        maxActionsPerRow: 5,
        maxRows: 5,
        maxLabelLength: 80,
        maxValueBytes: 100,
        supportsStyles: true,
        supportsDisabled: false,
      },
      selects: {
        maxOptions: 25,
        maxLabelLength: 100,
        maxValueBytes: 100,
      },
      text: {
        maxLength: 2000,
        encoding: "characters",
        markdownDialect: "discord-markdown",
      },
    },
  },
  deliveryCapabilities: {
    pin: true,
  },
  renderPresentation({ payload, presentation, ctx }) {
    return renderNativePayload(payload, presentation, ctx);
  },
  async pinDeliveredMessage({ target, messageId, pin }) {
    await pinNativeMessage(target, messageId, { notify: pin.notify === true });
  },
};
```

ケイパビリティの真偽値は、レンダラーがインタラクティブにできるものを示します。オプションの
`limits` は、レンダラーを呼び出す前にコアが適合させられる汎用エンベロープを示します:

```ts
type ChannelPresentationCapabilities = {
  supported?: boolean;
  buttons?: boolean;
  selects?: boolean;
  context?: boolean;
  divider?: boolean;
  charts?: boolean;
  tables?: boolean;
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
```

コアはレンダリング前にセマンティックコントロールへ汎用制限を適用します。ネイティブブロック数、カードサイズ、URL 制限、および汎用コントラクトでは表現できないプロバイダー固有の仕様に対する最終的な検証と切り詰めは、引き続きレンダラーが担当します。制限によってブロックからすべてのコントロールが削除された場合、配信メッセージに表示可能なフォールバックが残るよう、コアはラベルを非インタラクティブなコンテキストテキストとして保持します。

## コアのレンダリングフロー

CLI と標準メッセージアクションが使用する正規の送信パスでは、コアは次の処理を行います:

1. プレゼンテーションペイロードを正規化します。
2. 対象チャンネルの送信アダプターを解決します。
3. `presentationCapabilities`を読み取ります。
4. アダプターが宣言している場合、アクション数、ラベル長、選択肢数などの汎用ケイパビリティ制限を適用します。アダプターがそれぞれ
   `charts: true` または `tables: true` を明示的に宣言していない限り、グラフブロックとテーブルブロックは決定論的なテキストになります。
5. アダプターがペイロードをレンダリングできる場合、`renderPresentation`を呼び出します。
6. アダプターが存在しない、またはレンダリングできない場合は、保守的なテキストへフォールバックします。
7. 生成されたペイロードを通常のチャンネル配信パスで送信します。
8. 最初のメッセージが正常に送信された後、`delivery.pin`などの配信メタデータを適用します。

`ReplyPayload`を直接使用するチャンネルローカルの返信またはプレビュー経路は、その正規パスに入るか、ペイロードをプレーンテキスト／メディアへ投影する前に同じプレゼンテーションフォールバックを実体化する必要があります。

プロデューサーがチャンネルに依存しない状態を維持できるよう、フォールバック動作はコアが担当します。ネイティブレンダリングとインタラクション処理はチャンネル Plugin が担当します。

## デグラデーションルール

プレゼンテーションは、機能が制限されたチャンネルにも安全に送信できる必要があります。

フォールバックテキストには次のものが含まれます:

- `title`を最初の行として表示
- `text`ブロックを通常の段落として表示
- `context`ブロックを簡潔なコンテキスト行として表示
- `divider`ブロックを視覚的な区切りとして表示
- リンクボタンの URL を含むボタンラベル
- 選択肢のラベル
- グラフのタイトル、種類、軸、カテゴリ、系列、値
- テーブルのキャプション、ヘッダー、すべての行の値

### ボタン値のフォールバック表示

チャンネルがインタラクティブコントロールをレンダリングできない場合、ボタンと選択値はプレーンテキストへフォールバックします。このフォールバック動作は、不透明なコールバックデータを非公開に保ちながら、使いやすさを維持します:

- **`command`型のアクション**は `` label: `command` `` としてレンダリングされるため、ユーザーはコマンドをコピーし、チャンネル入力で手動実行できます。
- **`callback`型のアクション**と従来の **`value`** フィールドは、ラベルのみでレンダリングされます。不透明なコールバック値はフォールバックテキストに公開されません。
- **`approval`型のアクション**は、ラベルのみでレンダリングされます。承認 ID と判断結果はトランスポートデータであり、汎用スカラーヘルパーやフォールバックテキストを通じて公開されません。
- **`url`アクション**、URL を持つ **`web-app`アクション**、および非推奨の **`url` /
  `webApp` / `web_app`** 入力では、URL がユーザー向けであるため、ボタンラベルとともに URL テキストがレンダリングされます。ホスト型ウィジェット専用のアクションは、ネイティブのウィジェット起動機能がないチャンネルではラベルのみでレンダリングされます。
- **選択肢**はラベルのみでレンダリングされます。基になる選択値はフォールバックテキストに公開されません。

フォールバック UI に手動コマンドの案内を追加するチャンネルアダプター（Feishu のドキュメントコメント手順など）は、フォールバックレンダラーが使用するものと同じプレゼンテーションブロックからコマンドの存在チェックを導出する必要があります。これにより、手動コマンドが実際に表示される場合にのみ案内テキストが表示されます。

未対応のネイティブコントロールは、送信全体を失敗させるのではなくデグレードする必要があります。
例:

- インラインボタンが無効な Telegram は、テキストフォールバックを送信します。
- 選択機能に対応していないチャンネルでは、選択肢をテキストとして一覧表示します。
- ネイティブのグラフ機能に対応していないチャンネルでは、グラフデータをテキストとして一覧表示します。
- ネイティブのテーブル機能に対応していないチャンネルでは、すべてのテーブル行をテキストとして一覧表示します。
- URL 専用ボタンは、ネイティブリンクボタンまたはフォールバック URL 行のいずれかになります。
- オプションのピン留めが失敗しても、配信済みメッセージは失敗扱いになりません。

主な例外は `delivery.pin.required: true` です。ピン留めが必須として要求され、チャンネルが送信済みメッセージをピン留めできない場合、配信は失敗を報告します。

## プロバイダーのマッピング

現在バンドルされているレンダラー:

| チャンネル      | ネイティブのレンダリング先                  | 注記                                                                                                                                                                                                             |
| --------------- | ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Discord         | コンポーネントとコンポーネントコンテナ       | 既存のプロバイダーネイティブなペイロードプロデューサー向けに従来の `channelData.discord.components` を維持しますが、新しい共有送信では `presentation` を使用する必要があります。                                                                 |
| Feishu          | インタラクティブカード                       | カードヘッダーでは `title` を使用できます。本文ではそのタイトルの重複を避けます。                                                                                                                                                  |
| Matrix          | テキストフォールバックと構造化イベントフィールド | ボタン／選択は対応済みとして宣言されますが、現在はすべてのブロックがネイティブのインタラクティブウィジェットではなく、`com.openclaw.presentation` イベントフィールドに格納された `renderMessagePresentationFallbackText` 出力としてレンダリングされます。 |
| Mattermost      | テキストとインタラクティブプロパティ           | 選択と区切りには対応していません。これらのブロックはテキストにデグレードします。                                                                                                                                             |
| Microsoft Teams | Adaptive Cards                            | 両方が指定されている場合、プレーンな `message` テキストがカードとともに含まれます。選択、スタイル、無効状態には対応していません。                                                                                     |
| Slack           | Block Kit                                 | `chart` をネイティブの `data_visualization` として、`table` をネイティブの `data_table` としてレンダリングします。従来の `channelData.slack.blocks` を維持しますが、新しい共有送信では `presentation` を使用する必要があります。                                   |
| Telegram        | テキストとインラインキーボード                 | ボタン／選択には、対象サーフェスのインラインボタン機能が必要です。それ以外の場合はテキストフォールバックが使用されます。                                                                                                         |
| プレーンチャンネル | テキストフォールバック                        | レンダラーのないチャンネルでも、読みやすい出力が得られます。                                                                                                                                                            |

プロバイダーネイティブなペイロードとの互換性は、既存の返信プロデューサー向けの移行上の便宜です。新しい共有ネイティブフィールドを追加する理由にはなりません。

## プレゼンテーションと InteractiveReply の比較

`InteractiveReply` は、承認およびインタラクションヘルパーで使用される旧来の内部サブセットです。次のものに対応します:

- テキスト
- ボタン
- 選択

`MessagePresentation` は、正規の共有送信コントラクトです。次のものを追加します:

- タイトル
- トーン
- コンテキスト
- 区切り
- グラフ
- テーブル
- URL 専用ボタン
- `ReplyPayload.delivery`を介した汎用配信メタデータ

旧来のコードを橋渡しする場合は、`openclaw/plugin-sdk/interactive-runtime` のヘルパーを使用します:

```ts
import {
  adaptMessagePresentationForChannel,
  applyPresentationActionLimits,
  hasMessagePresentationBlocks,
  interactiveReplyToPresentation,
  isMessagePresentationInteractiveBlock,
  normalizeMessagePresentation,
  presentationPageSize,
  presentationToInteractiveControlsReply,
  presentationToInteractiveReply,
  renderMessagePresentationChartFallbackText,
  renderMessagePresentationFallbackText,
  renderMessagePresentationTableFallbackText,
  resolveMessagePresentationActionValue,
  resolveMessagePresentationButtonAction,
  resolveMessagePresentationControlValue,
  resolveMessagePresentationOptionAction,
} from "openclaw/plugin-sdk/interactive-runtime";
```

新しいコードでは、`MessagePresentation` を直接受け入れるか生成する必要があります。既存の
`interactive` ペイロードは、`presentation` の非推奨サブセットです。旧来のプロデューサー向けのランタイム対応は維持されます。

知っておくべき非推奨ではないヘルパー:

- `normalizeMessagePresentation(raw)` / `hasMessagePresentationBlocks(value)`
  型指定のないペイロード（たとえば、CLI の
  `--presentation` フラグからの JSON）を検証および型変換して `MessagePresentation` にします。
- `isMessagePresentationInteractiveBlock(block)` はブロックを
  `buttons` | `select` 共用体型に絞り込みます。
- `resolveMessagePresentationButtonAction(button)` と
  `resolveMessagePresentationOptionAction(option)` は、非推奨の境界フィールドを受け入れながら、
  正規の型付きアクションを返します。明示的な `action` が
  常に優先されます。
- `resolveMessagePresentationActionValue(action)` /
  `resolveMessagePresentationControlValue(control)` は、コマンド／コールバックの
  スカラー値のみを読み取ります。スカラーでない正規アクションが従来のシャドウ
  `value` にフォールスルーすることはないため、承認 ID とリンク先は型付きのまま維持されます。
- `renderMessagePresentationChartFallbackText(block)` /
  `renderMessagePresentationTableFallbackText(block)` は、構造化データブロックを 1 つずつ、
  チャネル固有のフォールバックパス向けに決定論的なテキストとしてレンダリングします。

従来の `InteractiveReply*` 型と変換ヘルパーは、SDK で
`@deprecated` としてマークされています。

- `InteractiveReply`、`InteractiveReplyBlock`、`InteractiveReplyButton`、および
  `InteractiveReplyOption`
- `normalizeInteractiveReply(...)`
- `hasInteractiveReplyBlocks(...)`
- `interactiveReplyToPresentation(...)`
- `presentationToInteractiveReply(...)`
- `presentationToInteractiveControlsReply(...)`
- `resolveInteractiveTextFallback(...)`
- `reduceInteractiveReply(...)`

`presentationToInteractiveReply(...)` と
`presentationToInteractiveControlsReply(...)` は、従来のチャネル実装向けのレンダラーブリッジとして
引き続き利用できます。新しい生成側コードではこれらを呼び出さず、
`presentation` を送信し、コア／チャネルの適応処理にレンダリングを任せてください。

承認ヘルパーにも、プレゼンテーション優先の代替があります。

- `buildApprovalInteractiveReply(...)` の代わりに
  `buildApprovalPresentation(...)` を使用してください
- `buildExecApprovalInteractiveReply(...)` の代わりに
  `buildExecApprovalPresentation(...)` を使用してください

リリース済みのこれらのビルダーは、Plugin の互換性を保つため、引き続きコマンドを基盤としています。永続的な承認種別を所有する Gateway
および同梱チャネルのコードでは、
`buildTypedApprovalPresentation(...)`、
`buildTypedExecApprovalPendingReplyPayload(...)`、または
`buildTypedPluginApprovalPendingReplyPayload(...)` を使用してください。これにより、トランスポートは `/approve` テキストから意味を推測するのではなく、
明示的な `approval` アクションを受け取ります。

`renderMessagePresentationFallbackText(...)` は、区切り線のみの
プレゼンテーションなど、テキストのフォールバックがないプレゼンテーションブロックに対して空文字列を返します。
空でない送信本文を必要とするトランスポートは、`emptyFallback` を渡すことで、
デフォルトのフォールバック契約を変更せずに最小限の本文を使用できます。

## 配信時のピン留め

ピン留めはプレゼンテーションではなく、配信の動作です。`channelData.telegram.pin` のような
プロバイダー固有のフィールドではなく、`delivery.pin` を使用してください。

セマンティクス：

- `pin: true` は、最初に正常配信されたメッセージをピン留めします。
- `pin.notify` のデフォルトは `false` です。
- `pin.required` のデフォルトは `false` です。
- 任意指定のピン留めが失敗した場合は処理を縮退させ、送信済みメッセージをそのまま維持します。
- 必須のピン留めが失敗した場合は、配信を失敗させます。
- 分割されたメッセージでは、末尾のチャンクではなく、最初に配信されたチャンクをピン留めします。

既存メッセージに対する手動の `pin`、`unpin`、および `pins` メッセージアクションは、プロバイダーがそれらの操作をサポートしている場合、引き続き利用できます。

## Plugin 作成者向けチェックリスト

- チャネルがセマンティックなプレゼンテーションをレンダリングできるか、安全に縮退できる場合は、`describeMessageTool(...)` から `presentation` を宣言します。
- ランタイムのアウトバウンドアダプターに `presentationCapabilities` を追加します。
- コントロールプレーンの Plugin セットアップコードではなく、ランタイムコードに `renderPresentation` を実装します。
- ネイティブ UI ライブラリを、高頻度で実行されるセットアップ／カタログパスに含めないでください。
- 汎用的な機能制限が判明している場合は、`presentationCapabilities.limits` で宣言します。
- 最終的なプラットフォーム制限をレンダラーとテストで維持します。
- 未対応のグラフ、表、ボタン、選択項目、URL
  ボタン、タイトル／テキストの重複、および `message` と `presentation` が混在する
  送信に対するフォールバックテストを追加します。
- プロバイダーが送信済みメッセージ ID をピン留めできる場合にのみ、`deliveryCapabilities.pin` と
  `pinDeliveredMessage` を通じて配信時のピン留めをサポートします。
- 新しいプロバイダー固有のカード／ブロック／コンポーネント／ボタンのフィールドを、
  共有メッセージアクションスキーマを通じて公開しないでください。

## 関連ドキュメント

- [メッセージ CLI](/ja-JP/cli/message)
- [Plugin SDK の概要](/ja-JP/plugins/sdk-overview)
- [Plugin アーキテクチャ](/ja-JP/plugins/architecture-internals#message-tool-schemas)
- [チャネルプレゼンテーションのリファクタリング計画](/ja-JP/plan/ui-channels)
