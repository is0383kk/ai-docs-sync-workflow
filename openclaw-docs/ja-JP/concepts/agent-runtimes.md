---
read_when:
    - OpenClaw、Codex、ACP、または別のネイティブエージェントランタイムから選択しています
    - ステータスや設定に表示されるプロバイダー／モデル／ランタイムのラベルが分かりにくい場合
    - ネイティブハーネスのサポート同等性を文書化しています
summary: OpenClaw がモデルプロバイダー、モデル、チャネル、エージェントランタイムを分離する仕組み
title: エージェントランタイム
x-i18n:
    generated_at: "2026-07-26T09:37:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 980d112946535df1566f2df4e3e71abacc2b073b51717c1e85fbb678691d39cb
    source_path: concepts/agent-runtimes.md
    workflow: 16
---

**エージェントランタイム**は、準備済みのモデルループを1つ所有します。プロンプトを受け取り、
モデル出力を駆動し、ネイティブツール呼び出しを処理して、完了したターンを
OpenClaw に返します。

ランタイムとプロバイダーは、どちらもモデル設定の近くに現れるため、
混同しやすいものです。ただし、両者は異なるレイヤーです。

| レイヤー      | 例                                           | 意味                                                                       |
| ------------- | -------------------------------------------- | -------------------------------------------------------------------------- |
| プロバイダー  | `anthropic`, `github-copilot`, `openai`      | OpenClaw が認証し、モデルを検出し、モデル参照に名前を付ける方法。          |
| モデル        | `claude-opus-4-6`, `gpt-5.6-sol`             | エージェントターン用に選択されたモデル。                                   |
| エージェントランタイム | `claude-cli`, `codex`, `copilot`, `openclaw` | 準備済みのターンを実行する低レベルのループまたはバックエンド。             |
| チャンネル    | Discord, Slack, Telegram, WhatsApp           | メッセージが OpenClaw に出入りする場所。                                   |

**ハーネス**は、エージェントランタイムを提供する実装を指すコード用語です。
たとえば、同梱の Codex ハーネスは `codex` ランタイムを実装します。
公開設定では、プロバイダーまたはモデルのエントリに `agentRuntime.id` を使用します。エージェント全体の
ランタイムキーはレガシーであり、無視されます。`openclaw doctor --fix` は古い
エージェント全体のランタイム固定を削除し、必要に応じてレガシーランタイムのモデル参照を、
正規のプロバイダー／モデル参照とモデルスコープのランタイムポリシーに書き換えます。

ランタイムには2つの系統があります。

- **組み込みハーネス**は OpenClaw の準備済みエージェントループ内で実行されます。
  組み込みの `openclaw` ランタイムに加え、`codex` や
  `copilot` など、登録済みの Plugin ハーネスが含まれます。
- **CLI バックエンド**は、モデル参照を正規のまま維持しつつ、ローカル CLI プロセスを
  実行します。たとえば、モデルスコープの `agentRuntime.id: "claude-cli"` を伴う
  `anthropic/claude-opus-5` は、「Anthropic モデルを選択し、Claude CLI
  経由で実行する」ことを意味します。`claude-cli` は組み込みハーネス ID ではないため、
  AgentHarness の選択に渡してはなりません。

`copilot` ハーネスは、GitHub Copilot CLI 用の独立したオプトインの外部 Plugin ハーネスです。
PI、Codex、GitHub Copilot のエージェントランタイムのうちどれを選ぶかについては、
[GitHub Copilot エージェントランタイム](/ja-JP/plugins/copilot)を参照してください。

## Codex のサーフェス

複数のサーフェスが Codex という名前を共有しています。

| サーフェス                                     | OpenClaw での名前／設定                | 動作内容                                                                                                             |
| ---------------------------------------------- | -------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| ネイティブ Codex app-server ランタイム         | `openai/*` モデル参照                  | Codex app-server を通じて OpenAI の組み込みエージェントターンを実行します。通常の ChatGPT/Codex サブスクリプション設定です。 |
| Codex OAuth 認証プロファイル                   | `openai` OAuth プロファイル          | Codex app-server ハーネスが使用する ChatGPT/Codex サブスクリプション認証を保存します。                               |
| Codex ACP アダプター                           | `runtime: "acp"`, `agentId: "codex"` | 外部 ACP/acpx コントロールプレーンを通じて Codex を実行します。ACP/acpx が明示的に要求された場合にのみ使用します。    |
| ネイティブ Codex チャット制御コマンドセット    | `/codex ...`                         | チャットから Codex app-server スレッドをバインド、再開、誘導、停止、検査します。                                    |
| 非エージェントサーフェス向け OpenAI Platform API ルート | `openai/*` と API キー認証       | 画像、埋め込み、音声、リアルタイムなどの OpenAI API を直接使用します。                                               |

これらのサーフェスは意図的に独立しています。`codex` Plugin を有効にすると、
ネイティブ app-server 機能が利用可能になります。`openclaw doctor --fix` は、
レガシー Codex ルートの修復と古いセッション固定のクリーンアップを担当します。エージェントモデルとして `openai/*`
を選択すると、非エージェントの OpenAI API サーフェスを使用している場合を除き、
現在は「これを Codex 経由で実行する」ことを意味します。

一般的な ChatGPT/Codex サブスクリプション設定では、認証に Codex OAuth を使用しますが、
モデル参照は `openai/*` のまま維持し、`codex` ランタイムを選択します。

```json5
{
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

これは、OpenClaw が OpenAI モデル参照を選択し、その後 Codex
app-server ランタイムに組み込みエージェントターンの実行を依頼することを意味します。
「API 課金を使用する」という意味ではなく、チャンネル、モデルプロバイダーカタログ、
または OpenClaw セッションストアが Codex になるという意味でもありません。

同梱の `codex` Plugin が有効な場合は、ACP の代わりにネイティブの `/codex` コマンド
サーフェス（`/codex bind`、`/codex threads`、`/codex resume`、`/codex steer`、
`/codex stop`）を使用して、自然言語で Codex を制御します。Codex に ACP を使用するのは、
ユーザーが ACP/acpx を明示的に要求した場合、または ACP アダプターパスをテストしている場合に限ります。
Claude Code、Gemini CLI、OpenCode、Cursor、および同様の外部ハーネスでは、引き続き ACP を使用します。

判断ツリー：

1. **Codex のバインド／制御／スレッド／再開／誘導／停止** -> 同梱の `codex` Plugin が有効な場合は、ネイティブの `/codex` コマンドサーフェス。
2. **組み込みランタイムとしての Codex**、または通常のサブスクリプションに基づく Codex エージェント体験 -> `openai/<model>`。
3. **OpenAI モデルに OpenClaw を明示的に選択** -> モデル参照を `openai/<model>` のまま維持し、プロバイダー／モデルのランタイムポリシーを `agentRuntime.id: "openclaw"` に設定します。選択された `openai` OAuth プロファイルは、OpenClaw の Codex 認証トランスポートを通じて内部的にルーティングされます。
4. **設定内のレガシー Codex モデル参照** -> `openclaw doctor --fix` を使用して `openai/<model>` に修復します。古いモデル参照がその意図を示していた場合、doctor はプロバイダー／モデルスコープの `agentRuntime.id: "codex"` を追加して Codex 認証ルートを維持します。レガシーの **`codex-cli/*`** モデル参照も、同じ `openai/<model>` Codex app-server ルートに修復されます。OpenClaw は同梱の Codex CLI バックエンドを保持しなくなりました。
5. **ACP、acpx、または Codex ACP アダプターが明示的に要求された場合** -> `runtime: "acp"` と `agentId: "codex"`。
6. **Claude Code、Gemini CLI、OpenCode、Cursor、Droid、またはその他の外部ハーネス** -> ネイティブのサブエージェントランタイムではなく、ACP/acpx。

| 意図するもの                            | 使用するもの                                      |
| --------------------------------------- | ------------------------------------------------- |
| Codex app-server のチャット／スレッド制御 | 同梱の `codex` Plugin の `/codex ...` |
| Codex app-server の組み込みエージェントランタイム | `openai/*` エージェントモデル参照                  |
| OpenAI Codex OAuth                      | `openai` OAuth プロファイル                      |
| Claude Code またはその他の外部ハーネス  | ACP/acpx                                          |

OpenAI 系プレフィックスの分割については、[OpenAI](/ja-JP/providers/openai)および
[モデルプロバイダー](/ja-JP/concepts/model-providers)を参照してください。Codex ランタイムのサポート契約については、
[Codex ハーネスランタイム](/ja-JP/plugins/codex-harness-runtime#v1-support-contract)を参照してください。

## ランタイムの所有範囲

ランタイムごとに、ループを所有する範囲が異なります。

| サーフェス                  | OpenClaw 組み込み                               | Codex app-server                                                            |
| --------------------------- | ---------------------------------------------- | --------------------------------------------------------------------------- |
| モデルループの所有者        | OpenClaw 組み込みランナーを通じた OpenClaw     | Codex app-server                                                            |
| 正規スレッド状態            | OpenClaw トランスクリプト                       | Codex スレッドと OpenClaw トランスクリプトのミラー                          |
| OpenClaw 動的ツール         | ネイティブ OpenClaw ツールループ                | Codex アダプターを通じてブリッジ                                            |
| ネイティブシェル／ファイルツール | OpenClaw パス                               | Codex ネイティブツール。対応している場合はネイティブフックを通じてブリッジ |
| コンテキストエンジン        | ネイティブ OpenClaw コンテキスト組み立て        | OpenClaw が組み立てたコンテキストを Codex ターンに投影                      |
| Compaction                  | OpenClaw または選択されたコンテキストエンジン  | Codex ネイティブ Compaction と、OpenClaw の通知およびミラー保守             |
| チャンネル配信              | OpenClaw                                       | OpenClaw                                                                    |

設計ルール：OpenClaw がサーフェスを所有している場合、通常の Plugin フック動作を提供できます。
ネイティブランタイムがサーフェスを所有している場合、OpenClaw にはランタイムイベントまたは
ネイティブフックが必要です。ネイティブランタイムが正規スレッド状態を所有している場合、
OpenClaw は未対応の内部構造を書き換えるのではなく、コンテキストをミラーリングして投影します。

## ランタイムの選択

OpenClaw は、プロバイダーとモデルを解決した後、次の順序で
組み込みランタイムを解決します。

1. **モデルスコープのランタイムポリシー**が優先されます。これは、設定済みプロバイダーの
   モデルエントリ、または `agents.defaults.models["provider/model"].agentRuntime`
   ／ `agents.entries.*.models["provider/model"].agentRuntime` にあります。`agents.defaults.models["vllm/*"].agentRuntime` などのプロバイダー
   ワイルドカードは、完全一致のモデルポリシーの後に適用されるため、動的に検出されたプロバイダーモデルは、
   モデルごとの完全一致の例外を上書きせずに、1つのランタイムを共有できます。
2. **プロバイダースコープのランタイムポリシー**：`models.providers.<provider>.agentRuntime`。
3. **`auto` モード**：登録済みの Plugin ランタイムは、対応するプロバイダー／モデルの組み合わせを引き受けることができます。
4. `auto` モードでどのランタイムもターンを引き受けない場合、OpenClaw は互換ランタイムとして
   `openclaw` にフォールバックします。実行を厳密にする必要がある場合は、
   明示的なランタイム ID を使用してください。

セッション全体およびエージェント全体のランタイム固定は無視されます：`OPENCLAW_AGENT_RUNTIME`、
セッションの `agentHarnessId`/`agentRuntimeOverride` 状態、`agents.defaults.agentRuntime`、
および `agents.entries.*.agentRuntime`。`openclaw doctor --fix` を実行して、古い
エージェント全体のランタイム設定を削除し、意図を維持できる場合は
レガシーランタイムのモデル参照を変換してください。

プロバイダー／モデルに明示された Plugin ランタイムはフェイルクローズします。プロバイダーまたはモデル上の `agentRuntime.id: "codex"`
は Codex、または明確な選択／ランタイムエラーを意味し、OpenClaw に暗黙的に
ルーティングし直されることはありません。一致しないターンを OpenClaw にルーティングできるのは
`auto` のみです。

CLI バックエンドのエイリアスは、組み込みハーネス ID とは異なります。推奨される Claude CLI の形式：

```json5
{
  agents: {
    defaults: {
      model: "anthropic/claude-opus-5",
      models: {
        "anthropic/claude-opus-5": {
          agentRuntime: { id: "claude-cli" },
        },
      },
    },
  },
}
```

`claude-cli/claude-opus-4-7` などのレガシー参照は互換性のため引き続き
サポートされますが、新しい設定ではプロバイダー／モデルを正規のまま維持し、
実行バックエンドをプロバイダー／モデルのランタイムポリシーに配置する必要があります。

レガシーの `codex-cli/*` 参照は異なります。doctor はそれらを `openai/*` に移行し、
Codex CLI バックエンドを維持する代わりに、Codex app-server ハーネスを通じて
実行されるようにします。

`auto` モードは、ほとんどのプロバイダーに対して意図的に保守的です。OpenAI エージェント
モデルは例外です。ランタイム未設定と `auto` は、どちらも Codex
ハーネスに解決されます。OpenClaw ランタイムの明示的な設定は、`openai/*` エージェントターン向けの
オプトイン互換ルートとして残ります。選択された `openai` OAuth
プロファイルと組み合わせると、OpenClaw は公開モデル参照を `openai/*` のまま維持しつつ、
そのパスを Codex 認証トランスポートを通じて内部的にルーティングします。古い OpenAI
ランタイムのセッション固定はランタイム選択時に無視され、
`openclaw doctor --fix` でクリーンアップできます。

`openclaw doctor` が、レガシー Codex モデル参照が設定に残っている状態で `codex` Plugin が有効になっていると警告した場合、それをレガシールートの状態として扱い、`openclaw doctor --fix` を実行して Codex ランタイムを使用する `openai/*` に書き換えます。

## GitHub Copilot エージェントランタイム

外部の `@openclaw/copilot` Plugin は、GitHub Copilot CLI（`@github/copilot-sdk`）を基盤とする、オプトインの `copilot` ランタイムを登録します。これは正規のサブスクリプション `github-copilot` プロバイダーを使用し、`auto` によって選択されることは**決してありません**。`agentRuntime.id` を使用して、モデルごとまたはプロバイダーごとにオプトインします。

```json5
{
  agents: {
    defaults: {
      model: "github-copilot/gpt-5.5",
      models: {
        "github-copilot/gpt-5.5": {
          agentRuntime: { id: "copilot" },
        },
      },
    },
  },
}
```

このハーネスは、`openclaw doctor` が自動的に読み込む `extensions/copilot/doctor-contract-api.ts` で、プロバイダー、ランタイム、CLI セッションキー、認証プロファイルのプレフィックスを宣言します。設定、認証、トランスクリプトのミラーリング、Compaction、宣言的な doctor コントラクト、および PI、Codex、Copilot の SDK に関するより広範な選択については、[GitHub Copilot エージェントランタイム](/ja-JP/plugins/copilot)を参照してください。

## 互換性コントラクト

ランタイムが OpenClaw ではない場合、そのドキュメントには、サポートする OpenClaw のサーフェスを明記する必要があります。

| 質問                                   | 重要である理由                                                                                           |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| モデルループを所有するのは誰ですか？   | 再試行、ツールの継続、最終回答の判断がどこで行われるかを決定します。                                     |
| 正規のスレッド履歴を所有するのは誰ですか？ | OpenClaw が履歴を編集できるのか、ミラーリングのみ可能なのかを決定します。                                 |
| OpenClaw の動的ツールは機能しますか？  | メッセージング、セッション、cron、および OpenClaw が所有するツールはこれに依存します。                    |
| 動的ツールフックは機能しますか？       | Plugin は、`before_tool_call`、`after_tool_call`、および OpenClaw が所有するツールを取り囲むミドルウェアを想定しています。 |
| ネイティブツールフックは機能しますか？ | シェル、パッチ、およびランタイムが所有するツールには、ポリシー適用と監視のためのネイティブフック対応が必要です。 |
| コンテキストエンジンのライフサイクルは実行されますか？ | メモリおよびコンテキスト Plugin は、assemble、ingest、after-turn、Compaction のライフサイクルに依存します。 |
| どの Compaction データが公開されますか？ | 通知のみを必要とする Plugin もあれば、保持／破棄されたメタデータを必要とする Plugin もあります。          |
| 意図的にサポートされていないものは何ですか？ | ネイティブランタイムがより多くの状態を所有する場合、ユーザーは OpenClaw と同等であると想定すべきではありません。 |

Codex ランタイムのサポートコントラクトについては、[Codex ハーネスランタイム](/ja-JP/plugins/codex-harness-runtime#v1-support-contract)に記載されています。

## ステータスラベル

ステータス出力には、`Execution` と `Runtime` の両方のラベルが表示される場合があります。これらはプロバイダー名ではなく、診断情報として解釈してください。

- `openai/gpt-5.6-sol` のようなモデル参照は、選択されたプロバイダー／モデルです。
- `codex` のようなランタイム ID は、そのターンを実行するループです。
- Telegram や Discord のようなチャンネルラベルは、会話が行われている場所です。

実行時に予期しないランタイムが表示された場合は、まず選択されたプロバイダー／モデルのランタイムポリシーを確認してください。レガシーセッションのランタイム固定は、ルーティングを決定しなくなりました。

## 関連項目

- [Codex ハーネス](/ja-JP/plugins/codex-harness)
- [Codex ハーネスランタイム](/ja-JP/plugins/codex-harness-runtime)
- [GitHub Copilot エージェントランタイム](/ja-JP/plugins/copilot)
- [OpenAI](/ja-JP/providers/openai)
- [エージェントハーネス Plugin](/ja-JP/plugins/sdk-agent-harness)
- [エージェントループ](/ja-JP/concepts/agent-loop)
- [モデル](/ja-JP/concepts/models)
- [ステータス](/ja-JP/cli/status)
