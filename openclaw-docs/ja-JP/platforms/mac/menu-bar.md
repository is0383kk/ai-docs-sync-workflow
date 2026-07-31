---
read_when:
    - macOS メニューの UI またはステータスロジックの調整
summary: メニューバーのステータス判定ロジックとユーザーに表示される内容
title: メニューバー
x-i18n:
    generated_at: "2026-07-26T09:40:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d53cd15109864b88010f41ccf4c46ea7fff6721bc6632630d83a558084cb2d62
    source_path: platforms/mac/menu-bar.md
    workflow: 16
---

## 表示内容

- 現在のエージェント作業状態は、メニューバーアイコンとメニューの最初のステータス行に表示されます。
- 作業中はヘルスステータスが非表示になり、すべてのセッションがアイドル状態になると再び表示されます。
- ルートの「Context」項目を開くと、ルートメニュー内で展開する代わりに、最近のセッションを含むサブメニューが表示されます。
- ルートメニューの「Nodes」ブロックには、クライアントやプレゼンスのエントリではなく、ペアリング済みの**デバイス**のみ（`node.list` から）が一覧表示されます。
- プロバイダーの使用量スナップショットが利用可能な場合、ルートの「Usage」セクションが Context の下に表示され、コストの詳細が利用可能な場合はその後に表示されます。
- **Quick Chat**を選択すると、フローティング形式のメインセッション作成画面が開きます。項目の横には、現在のグローバルショートカットが表示されます。

## 状態モデル

- ソース: `WorkActivityStore`（`apps/macos/Sources/OpenClaw/WorkActivityStore.swift`）。
- イベントは、`runId` を伴う `ControlAgentEvent` として到着します。ハンドラー（`ControlChannel.routeWorkActivity`）はイベントペイロードから `sessionKey` を読み取り、存在しない場合はデフォルトで `"main"` を使用します。
- 優先順位: メインセッション（デフォルトでは `sessionKey == "main"`）が常に優先されます。メインがアクティブな場合、その状態が即座に表示されます。メインがアイドル状態の場合は、直近でアクティブだったメイン以外のセッションが代わりに表示されます。ストアはアクティビティの途中では切り替わらず、現在のセッションがアイドル状態になるか、メインがアクティブになった場合にのみ切り替わります。
- アクティビティの種類:
  - `job`: 上位レベルのコマンド実行（`state: started|streaming|done|error|...`）。
  - `tool`: `name` を伴う `phase: start|result`。`meta`/`args` は任意です。

## IconState 列挙型（Swift）

- `idle`
- `workingMain(ActivityKind)`
- `workingOther(ActivityKind)`
- `overridden(ActivityKind)`（デバッグ用オーバーライド）

### ActivityKind -> バッジシンボル

`ActivityKind` は、`ToolKind`（`bash`、`read`、`write`、`edit`、`attach`、`other`）または単独の `job` をラップします。それぞれは、クリッターアイコン（`IconState.badgeSymbolName`）の上に描画される SF Symbol バッジにマッピングされます。

| 種類            | シンボル                             |
| --------------- | ---------------------------------- |
| `bash`          | `chevron.left.slash.chevron.right` |
| `read`          | `doc`                              |
| `write`         | `pencil`                           |
| `edit`          | `pencil.tip`                       |
| `attach`        | `paperclip`                        |
| `other` / `job` | `gearshape.fill`                   |

### 視覚的なマッピング

- `idle`: 通常のクリッター。バッジはありません。
- `workingMain`: シンボル付きバッジ、完全な色合い（`.primary` の強調度）、脚の「作業中」アニメーション。
- `workingOther`: シンボル付きバッジ、抑えた色合い（`.secondary` の強調度）、小走りアニメーションなし。
- `overridden`: 実際のアクティビティに関係なく、選択されたシンボルと色合いを使用します。

## Context サブメニュー

- ルートメニューには、セッション数とステータスを表示する「Context」行が1つあり、そこからサブメニュー（`MenuSessionsInjector`）が開きます。
- サブメニューのヘッダーには、過去24時間のアクティブなセッション数が表示されます。
- 各セッション行には、トークンバー、経過時間、プレビュー、思考/詳細表示の切り替え、リセット、圧縮、削除の各アクションが引き続き表示されます。
- 読み込み中、切断、セッション読み込みエラーのメッセージは、Context サブメニュー内に表示されます。
- 使用量とコストのセクションは Context の下にあるルートレベルに残るため、サブメニューを開かなくても一目で確認できます。

## ステータス行のテキスト（メニュー）

- 作業中: `<Session role> · <activity label>`（`MenuContentView` 内の `"\(roleLabel) · \(activity.label)"`）。ロールラベルは `Main` または `Other` です。
- アイドル時: ヘルス概要にフォールバックします。

## イベントの取り込み

- ソース: `ControlChannel.routeWorkActivity(from:)` によってルーティングされる、制御チャネルの `agent` イベント。
- 解析されるフィールド:
  - 開始/停止用の `data.state` を伴う `stream: "job"`。
  - `data.phase`、`data.name`、および任意の `data.meta`/`data.args` を伴う `stream: "tool"`。
- ツールラベルは `ToolDisplayRegistry.resolve(name:args:meta:)` から取得されます。解決できない名前には、生のツール名がフォールバックとして使用されます。

## デバッグ用オーバーライド

- Settings > Debug > 「Icon override」ピッカー:
  - `System (auto)`（デフォルト）
  - `Working: main` / `Working: other`（ツールの種類ごと: bash、read、write、edit、other）
  - `Idle`
- `UserDefaults` のキー `openclaw.iconOverride` に保存され、`IconState.overridden` にマッピングされます。

## テストチェックリスト

- メインセッションのジョブをトリガーする: アイコンが即座に切り替わり、ステータス行にメインラベルが表示されます。
- メインがアイドル状態の間にメイン以外のセッションのジョブをトリガーする: アイコンとステータスにメイン以外のセッションが表示され、完了するまで安定した状態が維持されます。
- 別のセッションがアクティブな間にメインを開始する: アイコンが即座にメインへ切り替わります。
- ツールの高速連続実行: バッジがちらつきません（完了したツールをクリアするまでの2秒間の猶予期間、`WorkActivityStore.toolResultGrace`）。
- すべてのセッションがアイドル状態になると、ヘルス行が再び表示されます。

## 関連項目

- [macOS アプリ](/ja-JP/platforms/macos)
- [メニューバーアイコン](/ja-JP/platforms/mac/icon)
