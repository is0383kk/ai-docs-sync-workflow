---
read_when:
    - auto-compaction と /compact について理解したい場合
    - コンテキスト上限に達する長時間セッションをデバッグしている場合
summary: モデルの制限内に収まるように OpenClaw が長い会話を要約する仕組み
title: Compaction
x-i18n:
    generated_at: "2026-07-26T09:32:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eb1f794fa60affd602378bcff8b07786bfeca55ab3fa09d5fa7214a05fa48806
    source_path: concepts/compaction.md
    workflow: 16
---

すべてのモデルにはコンテキストウィンドウ、つまり処理できるトークンの最大数があります。会話がその上限に近づくと、チャットを継続できるように、OpenClaw は古いメッセージを要約へと **圧縮** します。

## 仕組み

1. 古い会話ターンがコンパクトなエントリに要約されます。
2. 要約がセッショントランスクリプトに保存されます。
3. 最近のメッセージはそのまま保持されます。

OpenClaw は圧縮の分割位置を選ぶ際、アシスタントのツール呼び出しと対応する `toolResult` エントリを対にして保持します。分割位置がツールブロック内になる場合、OpenClaw はその対が分離されず、現在の未要約部分が保持されるように境界を移動します。

会話履歴全体はディスク上に残ります。圧縮によって変わるのは、次のターンでモデルが参照する内容だけです。

<Note>
新しい設定では、`agents.defaults.compaction.mode` のデフォルトは `"safeguard"`（より厳格なガードレール、要約品質監査）です。オプトアウトするには、`mode: "default"` を明示的に設定してください。
</Note>

## 自動圧縮

自動圧縮はデフォルトで有効です。セッションがコンテキスト上限に近づいた場合、またはモデルがコンテキストオーバーフローエラーを返した場合に実行されます（後者の場合、OpenClaw は圧縮して再試行します）。

次の内容が表示されます。

- 通常の Gateway ログでは `embedded run auto-compaction start` / `complete`。
- 詳細モードでは `🧹 Auto-compaction complete`。
- `🧹 Compactions: <count>` を示す `/status`。

<Info>
圧縮の前に、OpenClaw は重要なメモを [メモリ](/ja-JP/concepts/memory) ファイルへ保存するようエージェントに自動で通知します。これにより、コンテキストの消失を防ぎます。
</Info>

<AccordionGroup>
  <Accordion title="OpenClaw が認識するオーバーフローエラーのパターン">
    OpenClaw はプロバイダー固有の多数のオーバーフローエラー文字列（Anthropic、OpenAI、Bedrock、Gemini、Ollama、OpenRouter など）と照合します。一般的な例：

    - `request_too_large`
    - `context length exceeded`
    - `input exceeds the maximum number of tokens`
    - `input token count exceeds the maximum number of input tokens`（Bedrock）
    - `input is too long for the model`
    - `ollama error: context length exceeded`

  </Accordion>
</AccordionGroup>

## 手動圧縮

任意のチャットで `/compact` と入力すると、圧縮を強制実行できます。要約の方針を指定するには、指示を追加します。

```text
/compact API 設計上の決定事項に焦点を当てる
```

`agents.defaults.compaction.keepRecentTokens` が設定されている場合（デフォルト：20,000）、手動圧縮はそのカットポイントに従い、再構築されたコンテキストに最近の部分を保持します。明示的な保持量が指定されていない場合、手動圧縮は厳密なチェックポイントとして動作し、新しい要約のみから継続します。

## 設定

`openclaw.json` の `agents.defaults.compaction` で圧縮を設定します。最も一般的な設定項目を以下に示します。完全なリファレンスについては、[セッション管理の詳細](/ja-JP/reference/session-management-compaction)を参照してください。

### 別のモデルを使用する

デフォルトでは、圧縮にはエージェントのプライマリモデルが使用されます。要約をより高性能または特化したモデルに委任するには、`agents.defaults.compaction.model` を設定します。このオーバーライドには、`provider/model-id` 文字列、または `agents.defaults.models` で設定された修飾なしのエイリアスを指定できます。

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "model": "openrouter/anthropic/claude-sonnet-4-6"
      }
    }
  }
}
```

設定された修飾なしのエイリアスは、圧縮の開始前に正規のプロバイダーとモデルへ解決されます。修飾なしの値がエイリアスと設定済みのリテラルモデル ID の両方に一致する場合、リテラルモデル ID が優先されます。一致しない修飾なしの値は、アクティブなプロバイダー上のモデル ID として扱われます。

これはローカルモデルでも機能します。たとえば、要約専用の2つ目の Ollama モデルを使用できます。

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "model": "ollama/llama3.1:8b"
      }
    }
  }
}
```

未設定の場合、圧縮はアクティブなセッションモデルで開始されます。要約がモデルフォールバックの対象となるプロバイダーエラーで失敗した場合、OpenClaw はセッションの既存のモデルフォールバックチェーンを通じて、その圧縮処理を再試行します。フォールバックの選択は一時的であり、セッション状態には書き戻されません。明示的な `agents.defaults.compaction.model` オーバーライドは厳密に適用され、セッションのフォールバックチェーンを継承しません。

### 識別子の保持

圧縮による要約では、不透明な識別子がデフォルトで保持されます（`identifierPolicy: "strict"`）。無効にするには `identifierPolicy: "off"` でオーバーライドします。カスタムガイダンスは、圧縮プロバイダーの `summarize()` 実装に指定します。

### アクティブトランスクリプトのバイト数ガード

`agents.defaults.compaction.maxActiveTranscriptBytes` が設定されている場合、トランスクリプト履歴が
そのサイズに達すると、OpenClaw は実行前に通常のローカル圧縮を
トリガーします。これは、プロバイダー側のコンテキスト管理によって
モデルコンテキストが正常に保たれる一方で、永続化されたトランスクリプト履歴が
増え続ける長時間実行セッションに役立ちます。生のバイト列を分割するのではなく、
通常の圧縮パイプラインに意味的な要約の作成を要求します。

<Warning>
バイト数ガードは、アクティブな SQLite トランスクリプト履歴に適用されます。従来の JSONL
チェックポイント成果物は、アクティブな圧縮対象ではありません。
</Warning>

### 後継トランスクリプト

`agents.defaults.compaction.truncateAfterCompaction` が有効な場合、OpenClaw は既存のトランスクリプトをその場で書き換えません。圧縮要約、保持された状態、未要約部分から新しいアクティブな後継トランスクリプトを作成し、ブランチ処理や復元処理がその圧縮済み後継を参照するよう、チェックポイントメタデータを記録します。
後継トランスクリプトでは、短い再試行期間内に到着した、完全に重複する長いユーザーターンも
除外されます。そのため、チャネルの再試行ストームが、圧縮後の
次のアクティブなトランスクリプトへ持ち越されることはありません。

OpenClaw は、新しい圧縮に対して個別の `.checkpoint.*.jsonl` コピーを
作成しなくなりました。既存の従来型チェックポイントファイルは、参照されている間は引き続き使用でき、
通常のセッションクリーンアップによって削除されます。

### 圧縮通知

デフォルトでは、圧縮は通知なしで実行されます。圧縮の開始時と完了時に短いステータスメッセージを表示し、圧縮前のメモリフラッシュを使い切っても応答が継続される場合に機能低下通知を表示するには、`notifyUser` を設定します。

```json5
{
  agents: {
    defaults: {
      compaction: {
        notifyUser: true,
      },
    },
  },
}
```

### メモリフラッシュ

圧縮の前に、OpenClaw は永続的なメモをディスクへ保存するため、**通知なしのメモリフラッシュ** ターンを実行できます。この保守ターンでアクティブな会話モデルではなくローカルモデルを使用する場合は、`agents.defaults.compaction.memoryFlush.model` を設定します。

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "memoryFlush": {
          "model": "ollama/qwen3:8b"
        }
      }
    }
  }
}
```

メモリフラッシュのモデルオーバーライドは厳密に適用され、アクティブなセッションのフォールバックチェーンを継承しません。詳細と設定については、[メモリ](/ja-JP/concepts/memory)を参照してください。

## 差し替え可能な圧縮プロバイダー

Plugin は、Plugin API の `registerCompactionProvider()` を介してカスタム圧縮プロバイダーを登録できます。プロバイダーが登録および設定されている場合、OpenClaw は組み込みの LLM パイプラインではなく、そのプロバイダーへ要約を委任します。

登録済みプロバイダーを使用するには、設定にその ID を指定します。

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "provider": "my-provider"
      }
    }
  }
}
```

`provider` を設定すると、`mode: "safeguard"` が自動的に強制されます。プロバイダーは、組み込み経路と同じ圧縮指示および識別子保持ポリシーを受け取り、OpenClaw はプロバイダーの出力後も、最近のターンと分割されたターンの接尾コンテキストを保持します。

<Note>
プロバイダーが失敗するか空の結果を返した場合、OpenClaw は組み込みの LLM 要約へフォールバックします。
</Note>

## 圧縮とプルーニングの比較

|                  | 圧縮                          | プルーニング                     |
| ---------------- | ----------------------------- | -------------------------------- |
| **処理内容**     | 古い会話を要約する            | 古いツール結果を切り詰める       |
| **保存されるか** | はい（セッショントランスクリプト内） | いいえ（リクエストごとのメモリ内のみ） |
| **対象範囲**     | 会話全体                      | ツール結果のみ                   |

[セッションプルーニング](/ja-JP/concepts/session-pruning)は、要約せずにツール出力を切り詰める、より軽量な補完機能です。

## トラブルシューティング

**圧縮が頻繁すぎる場合：** モデルのコンテキストウィンドウが小さいか、ツール出力が大きい可能性があります。[セッションプルーニング](/ja-JP/concepts/session-pruning)を有効にしてみてください。

**圧縮後にコンテキストが古く感じられる場合：** `/compact Focus on <topic>` を使用して要約の方針を指定するか、メモが保持されるように[メモリフラッシュ](/ja-JP/concepts/memory)を有効にします。

**白紙の状態から始める必要がある場合：** `/new` は、圧縮せずに新しいセッションを開始します。

高度な設定（予約トークン、識別子の保持、カスタムコンテキストエンジン、OpenAI サーバー側圧縮）については、[セッション管理の詳細](/ja-JP/reference/session-management-compaction)を参照してください。

## 関連項目

- [セッション](/ja-JP/concepts/session)：セッションの管理とライフサイクル。
- [セッションプルーニング](/ja-JP/concepts/session-pruning)：ツール結果の切り詰め。
- [コンテキスト](/ja-JP/concepts/context)：エージェントターン用のコンテキストを構築する仕組み。
- [フック](/ja-JP/automation/hooks)：圧縮のライフサイクルフック（`before_compaction`、`after_compaction`）。
