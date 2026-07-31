---
read_when:
    - '`.experimental` 設定キーを見つけ、それが安定しているかどうかを確認したい場合'
    - 通常のデフォルトと混同せずに、プレビュー版のランタイム機能を試したい場合
    - 現在文書化されている実験的なフラグを1か所で確認したい場合
summary: OpenClaw における実験的フラグの意味と、現在文書化されているフラグ
title: 実験的な機能
x-i18n:
    generated_at: "2026-07-26T08:59:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6c14b74bbafce77c0d1e1358ad94053675c4aad9e26be78719f58e78f455c3a2
    source_path: concepts/experimental-features.md
    workflow: 16
---

実験的機能は、明示的なフラグの背後にあるプレビュー用の機能です。安定したデフォルトまたは長期的な契約として提供されるには、実環境でさらに利用実績を積む必要があります。

- ドキュメントに限定的な自動セットアップ規則が記載されている場合を除き、デフォルトでは無効です。
- 形式と動作は、安定版の設定よりも速いペースで変更される可能性があります。
- 安定した方法がすでに存在する場合は、そちらを優先してください。
- まず小規模な環境でテストしてから、広範囲に展開してください。

## 現在ドキュメント化されているフラグ

| 機能                     | キー                                                                                          | 使用する場面                                                                                                                      | 詳細                                                                                   |
| ------------------------ | --------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| ローカルモデルランタイム | `agents.defaults.experimental.localModelLean`, `agents.entries.*.experimental.localModelLean` | 小規模または制約の厳しいローカルバックエンドが、OpenClaw の完全なデフォルトツール構成を処理できない場合                           | [ローカルモデル](/ja-JP/gateway/local-models)                                                |
| Codex ハーネス            | `plugins.entries.codex.config.appServer.experimental.sandboxExecServer`                       | Code Mode を無効にせず、ネイティブ Codex app-server 0.132.0 以降から OpenClaw のサンドボックスを基盤とする exec-server を使用する場合 | [Codex ハーネスリファレンス](/ja-JP/plugins/codex-harness-reference#sandboxed-native-execution) |
| 構造化計画ツール         | `tools.experimental.planTool`                                                                 | 対応するランタイムおよび UI で、複数ステップの作業を追跡するための構造化 `update_plan` ツールを公開する場合                   | [Gateway 設定リファレンス](/ja-JP/gateway/config-tools#toolsexperimental)                    |
| Code Mode                | `tools.codeMode.enabled`                                                                      | 非表示の OpenClaw ツールカタログへ、コードによってオーケストレーションされた簡潔なアクセスを使用する場合                         | [Code Mode](/ja-JP/tools/code-mode)                                                          |
| Swarm                    | `tools.swarm.enabled`                                                                         | Code Mode スクリプトから、範囲を限定したサブエージェントのグループを並列でオーケストレーションする場合                           | [Swarm](/tools/swarm)                                                                  |

## Control UI Labs

Control UI に切り替えスイッチがある実験的機能を管理するには、**Settings → Agents & Tools → Labs** を開きます。Lab を有効または無効にすると、正規の Gateway 設定へ即座にパッチが適用されます。このページに再起動の案内が表示されるのは、機能に再起動が必要な場合のみです。

現在提供されている Labs の項目は Code Mode と Swarm です。どちらのスイッチも、検証済みの既存の設定キーに書き込み、通常は Gateway を再起動しなくても、以降のエージェント実行から有効になります。

## ローカルモデルの軽量モード

`agents.defaults.experimental.localModelLean: true` は、各ターンでエージェントに直接公開されるツールから、負荷の高いオプションツールである `browser`、`cron`、`message`、`image_generate`、`music_generate`、`video_generate`、`tts`、および `pdf` を除外します。明示的に許可されたツールや配信に必要なツールは引き続き使用できますが、直接公開されず Tool Search に登録される場合があります。また、`tools.toolSearch` がまだ設定されていない場合、軽量モードでは Plugin/MCP/クライアントのカタログに、構造化 Tool Search（`tool_search`、`tool_describe`、`tool_call`）がデフォルトで適用されます。これを 1 つのエージェントに限定するには、`agents.entries.*.experimental.localModelLean` を使用します。

オンボーディング中に検証済みの `ollama` または `lmstudio` 推論ルートがある場合、その値が未設定であれば `agents.defaults.experimental.localModelLean: true` が自動的に設定されます。OpenClaw は設定がオンボーディングによるものであることを記録するため、後で検証済みの非ローカルルートを使用すると、自動設定された値のみが解除されます。明示的に設定された `true` または `false` は保持されます。その他のセルフホスト型プロバイダーおよび OpenAI 互換プロバイダーは、モデル名や URL から推測されません。

Tool Search をすでにグローバルに調整している場合、OpenClaw はその設定を変更しません。軽量モードによる Tool Search のデフォルト設定を無効にするには、`tools.toolSearch: false` を設定します。

構造化 `tools` モードでは、軽量実行時も `exec` が Tool Search のコントロールと並んで直接表示されるため、コーディング向けに調整されたローカルモデルでも、使い慣れたシェルの経路を選択できます。変更されるのはスキーマの可視性のみです。通常のツールポリシー、サンドボックス化、および exec の承認は引き続き適用されます。明示的な `code` および `directory` モードでは、通常の Compaction 動作が維持されます。

### これらのツールが対象となる理由

これらのツールは、説明が最も長い、パラメーター形式が最も広範である、または小規模なモデルの注意を通常のコーディングや会話の流れから逸らす可能性が最も高いものです。コンテキストが小さい、または制約の厳しい OpenAI 互換バックエンドでは、これにより次の違いが生じます。

- ツールスキーマがプロンプトに収まるか、会話履歴を圧迫するか。
- モデルが適切なツールを選択するか、類似するスキーマが多すぎるため不正なツール呼び出しを出力するか。
- Chat Completions アダプターが構造化出力の制限内に収まるか、ツール呼び出しのペイロードサイズによって 400 エラーになるか。

これらを除外しても、直接表示されるツールの一覧が短くなるだけです。モデルは引き続き `read`、`write`、`edit`、`exec`、`apply_patch`、画像理解、Web 検索/取得（設定されている場合）、メモリ、およびセッション/エージェントツールを使用できます。`tools.toolSearch: false` を設定しない限り、追加のカタログには Tool Search から引き続きアクセスできます。明示的にツールを許可すれば、軽量エージェントで除外されたワークフローを再び使用できます。

### 有効にする場面

モデルが Gateway と通信できることを確認済みでも、完全なエージェントターンが正しく動作しない場合に、軽量モードを有効にしてください。

1. `openclaw infer model run --gateway --model <ref> --prompt "Reply with exactly: pong"` が成功する。
2. 通常のエージェントターンが、不正なツール呼び出し、過大なプロンプト、またはモデルがツールを無視することによって失敗する。
3. `localModelLean: true` を切り替えると失敗が解消される。

### 無効のままにする場面

バックエンドが完全なデフォルトランタイムを問題なく処理できる場合は、無効のままにしてください。これは、より小さなツール構成を必要とするローカルスタック向けの回避策であり、ホスト型モデルや十分なリソースを備えたローカル環境向けのデフォルトではありません。

軽量モードは、`tools.profile`、`tools.allow`/`tools.deny`、またはモデルの `compat.supportsTools: false` エスケープハッチを置き換えるものではありません。特定のエージェントに永続的に狭いツール構成を適用する場合は、これらの安定した設定を優先してください。

### 有効化

```json5
{
  agents: {
    defaults: {
      experimental: {
        localModelLean: true,
      },
    },
  },
}
```

1 つのエージェントのみに適用する場合：

```json5
{
  agents: {
    list: [
      {
        id: "local",
        model: "lmstudio/gemma-4-e4b-it",
        experimental: {
          localModelLean: true,
        },
      },
    ],
  },
}
```

フラグを変更した後、Gateway を再起動してください。`tools.allow` または `tools.alsoAllow` で明示的に保持しない限り、軽量フィルタリングによって `browser`、`cron`、`message`、`image_generate`、`music_generate`、`video_generate`、`tts`、および `pdf` が除外されます。保持されたツールも、直接公開されず Tool Search に登録される場合があります。

## 実験的であることは非表示を意味しない

実験的機能は、安定版のように見えるデフォルト設定の背後に隠すのではなく、ドキュメントと設定パス自体で実験的であることを明示する必要があります。

## 関連項目

- [機能](/ja-JP/concepts/features)
- [リリースチャネル](/ja-JP/install/development-channels)
