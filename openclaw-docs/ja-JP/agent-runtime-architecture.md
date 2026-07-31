---
summary: OpenClaw が組み込みエージェントランタイムを構成する仕組み：コード構成、境界、リソースマニフェスト、ランタイムの選択。
title: エージェントランタイムアーキテクチャ
x-i18n:
    generated_at: "2026-07-26T08:53:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3e09ff21b4369a7c102db51e4458ad3ba1e86c9fe43a3a8bff72eef1713d2d51
    source_path: agent-runtime-architecture.md
    workflow: 16
---

OpenClaw は組み込みエージェントランタイムを所有します。ランタイムコードは `src/agents/` に、モデル／プロバイダートランスポートは `src/llm/` に配置され、Plugin 向けコントラクトは `openclaw/plugin-sdk/*` バレルを通じて公開されます。

## ランタイムのレイアウト

| パス                                | 所有範囲                                                                                                                                                                                                                      |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/agents/embedded-agent-runner/` | 組み込みの試行ループ（`run.ts`、`run/`）、モデル選択とプロバイダーの正規化（`model*.ts`）、プロバイダーごとのリクエストパラメーター（`extra-params.*`）、Compaction、トランスクリプトとセッションの接続。                            |
| `src/agents/sessions/`              | セッションの永続化（`session-manager.ts`）、リソース探索（`package-manager.ts`、`resource-loader.ts`）、セッション内での `extensions` 読み込み、プロンプトテンプレート、Skills、テーマ、TUI ベースのツールレンダラー（`tools/`）。 |
| `packages/agent-core/`              | 再利用可能なエージェントコア（`@openclaw/agent-core`）：エージェントループ、ハーネス型、メッセージ、Compaction ヘルパー、プロンプトテンプレート、Skills、セッションストレージのコントラクト。                                                           |
| `src/agents/runtime/`               | `@openclaw/agent-core` を Plugin SDK の LLM ランタイムに接続し、それとローカルプロキシユーティリティを再エクスポートする OpenClaw ファサード。                                                                                             |
| `src/agents/agent-tools*.ts`        | OpenClaw が所有するツール定義、パラメータースキーマ、ツールポリシー、ツール呼び出し前後のアダプター、ホスト／サンドボックス編集ツール。                                                                                            |
| `src/agents/agent-hooks/`           | 組み込みランタイムフック：Compaction セーフガード、Compaction 指示、コンテキストの枝刈り。                                                                                                                                   |
| `src/agents/harness/`               | 組み込みおよび Plugin 登録済みハーネスのレジストリ、選択ポリシー、ライフサイクル。                                                                                                                       |
| `src/llm/`                          | モデル／プロバイダーレジストリ、トランスポートヘルパー、プロバイダー固有のストリーム実装（`src/llm/providers/`）。                                                                                                          |

## 境界

コアは OpenClaw モジュールと SDK バレルを通じて組み込みランタイムを呼び出します。外部のエージェントフレームワークパッケージは残っていません。Plugin は文書化された `openclaw/plugin-sdk/*` エントリーポイントを使用し、`src/**` の内部実装をインポートしません。

`@earendil-works/pi-tui` は引き続きサードパーティ依存関係です。これはローカル TUI とセッションツールレンダラーで使用されるターミナルコンポーネントツールキットです。これを内部化するには、別途ベンダリング作業が必要です。

## マニフェスト

リソースパッケージは `package.json` メタデータで OpenClaw リソースを宣言します。エントリーはパッケージルートからの相対ファイルパスまたは glob です。

```json
{
  "openclaw": {
    "extensions": ["extensions/index.ts"],
    "skills": ["skills/*.md"],
    "prompts": ["prompts/*.md"],
    "themes": ["themes/*.json"]
  }
}
```

マニフェストに記載されていないリソースタイプについては、慣例的な `extensions/`、`skills/`、`prompts/`、`themes/` ディレクトリの探索にフォールバックします。

## ランタイムの選択

- 組み込みランタイム ID は `openclaw` です。レガシーエイリアス `pi` は `openclaw` に正規化され、`codex-app-server` は `codex` に正規化されます。
- Plugin ハーネスは追加のランタイム ID（例：`codex`）を登録します。
- ランタイムポリシーはモデル／プロバイダー単位の `agentRuntime.id` 設定です（モデルのエントリーがプロバイダーのエントリーより優先されます）。未設定または `default` の場合は `auto` に解決されます。
- `auto` は、有効なプロバイダールートをサポートする登録済み Plugin ハーネスを選択し、該当しない場合は組み込みの OpenClaw ランタイムを選択します。プロバイダーまたはモデルのプレフィックスだけでは、ハーネスは選択されません。
- OpenAI が `codex` を暗黙的に選択できるのは、作成者が指定したリクエストオーバーライドがなく、公式 HTTPS Platform Responses または ChatGPT Responses のルートと完全に一致する場合に限られます。Completions アダプター、カスタムエンドポイント、および作成者が指定したリクエスト動作を持つルートでは `openclaw` が維持され、公式のプレーンテキスト HTTP エンドポイントは拒否されます。[OpenAI の暗黙的なエージェントランタイム](/ja-JP/providers/openai#implicit-agent-runtime)を参照してください。

## モデルランタイムの世代

Gateway の起動時、および設定、Plugin、認証の公開時に、設定済みのエージェントごとに準備済みモデルランタイム世代が 1 つ構築されます。各世代は、探索済みの認証テンプレート、モデルレジストリ、射影済みモデルカタログを 1 つのアトミックなスナップショットとして所有します。エージェント実行は、そのスナップショットから可変の認証ストアとレジストリストアをフォークします。ブラウズ、ステータス、Cron、doctor、TUI、PDF、画像の各パスは、ファイルシステム探索を繰り返す代わりに、公開済みカタログを読み取ります。

スタンドアロンの組み込みランタイムは、アクティベーション境界で同じ形状のスナップショットを公開します。失敗した世代または古い世代が、より新しい部分的な世代と並行して提供されることはありません。ライフサイクル所有者は、まず完全な置き換えを公開する必要があります。

## 関連項目

- [OpenClaw エージェントランタイムのワークフロー](/ja-JP/openclaw-agent-runtime)
- [エージェントランタイム](/ja-JP/concepts/agent-runtimes)
