---
read_when:
    - デフォルトのメモリバックエンドについて理解したい場合
    - 埋め込みプロバイダーまたはハイブリッド検索を設定する場合
summary: キーワード検索、ベクトル検索、ハイブリッド検索に対応したデフォルトの SQLite ベースのメモリバックエンド
title: 組み込みメモリエンジン
x-i18n:
    generated_at: "2026-07-26T09:00:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c3efb6f1449d9b55717b3c117444ba7d4519d0111b842b48790ad85551511433
    source_path: concepts/memory-builtin.md
    workflow: 16
---

builtin エンジンはデフォルトのメモリバックエンドです。メモリインデックスを
エージェントごとの SQLite データベースに保存し、利用を開始するための
追加の依存関係は必要ありません。

## 提供される機能

- FTS5 全文インデックス（BM25 スコアリング）による**キーワード検索**。
- サポートされている任意のプロバイダーの埋め込みによる**ベクトル検索**。
- 最良の結果を得るために両方を組み合わせる**ハイブリッド検索**。
- 中国語、日本語、韓国語向けのトライグラムトークン化による**CJK サポート**。
- データベース内ベクトルクエリ向けの **sqlite-vec アクセラレーション**（オプション）。

## はじめに

デフォルトでは、builtin エンジンは OpenAI の埋め込みを使用します。`OPENAI_API_KEY` または
`models.providers.openai.apiKey` がすでに設定されている場合、追加のメモリ設定なしで
ベクトル検索を利用できます。

プロバイダーを明示的に設定するには、次のようにします。

```json5
{
  memory: {
    search: {
      provider: "openai",
    },
  },
}
```

埋め込みプロバイダーがない場合は、キーワード検索のみ利用できます。

ローカルの GGUF 埋め込みを強制的に使用するには、公式の llama.cpp プロバイダー
Plugin をインストールし、`local.modelPath` で GGUF ファイルを指定します。

```bash
openclaw plugins install @openclaw/llama-cpp-provider
```

```json5
{
  memory: {
    search: {
      provider: "local",
      fallback: "none",
      local: {
        modelPath: "~/.node-llama-cpp/models/embeddinggemma-300m-qat-Q8_0.gguf",
      },
    },
  },
}
```

## サポートされている埋め込みプロバイダー

| プロバイダー      | ID                  | 備考                                |
| ----------------- | ------------------- | ----------------------------------- |
| Bedrock           | `bedrock`           | AWS 認証情報チェーンを使用          |
| DeepInfra         | `deepinfra`         | デフォルト: `BAAI/bge-m3`              |
| Gemini            | `gemini`            | マルチモーダル（画像 + 音声）をサポート |
| GitHub Copilot    | `github-copilot`    | Copilot サブスクリプションを使用    |
| LM Studio         | `lmstudio`          | ローカル／セルフホスト              |
| ローカル          | `local`             | `@openclaw/llama-cpp-provider`      |
| Mistral           | `mistral`           |                                     |
| Ollama            | `ollama`            | ローカル／セルフホスト              |
| OpenAI            | `openai`            | デフォルト: `text-embedding-3-small`   |
| OpenAI 互換       | `openai-compatible` | 汎用 `/v1/embeddings` エンドポイント   |
| Voyage            | `voyage`            |                                     |

OpenAI 以外に切り替えるには、`memory.search.provider` を設定します。

## インデックス作成の仕組み

OpenClaw は `MEMORY.md` と `memory/*.md` をチャンク（デフォルトでは 400 トークン、
80 トークンのオーバーラップ）に分割してインデックス化し、エージェントごとの SQLite データベースに保存します。

- **インデックスの場所:** 所有エージェントのデータベース
  `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- **ストレージのメンテナンス:** SQLite WAL サイドカーは、定期チェックポイントと
  シャットダウン時のチェックポイントによって上限が保たれます。
- **ファイル監視:** メモリファイルへの変更により、デバウンスされた再インデックスが
  トリガーされます（デフォルトは 1.5 秒）。
- **自動再インデックス:** 埋め込みプロバイダー、モデル、チャンク設定、
  設定済みソース、またはスコープが変更されると、インデックスは自動的に再構築されます。
- **オンデマンド再インデックス:** `openclaw memory index --force`

<Info>
`memory.search.extraPaths` を使用すると、ワークスペース外の Markdown ファイルも
インデックス化できます。詳しくは
[設定リファレンス](/ja-JP/reference/memory-config#additional-memory-paths)を参照してください。
</Info>

## 使用する場合

builtin エンジンは、ほとんどのユーザーに適しています。

- 追加の依存関係なしで、そのまま利用できます。
- キーワード検索とベクトル検索の両方を適切に処理します。
- すべての埋め込みプロバイダーをサポートします。
- ハイブリッド検索により、両方の検索手法の長所を組み合わせます。

再ランキングやクエリ拡張が必要な場合、またはワークスペース外のディレクトリを
インデックス化したい場合は、[QMD](/ja-JP/concepts/memory-qmd)への切り替えを検討してください。

自動ユーザーモデリングを備えたセッション横断メモリが必要な場合は、
[Honcho](/ja-JP/concepts/memory-honcho)を検討してください。

## トラブルシューティング

**メモリ検索が無効になっていますか？** `openclaw memory status` を確認してください。プロバイダーが
検出されない場合は、明示的に設定するか、API キーを追加します。

**ローカルプロバイダーが検出されませんか？** ローカルパスが存在することを確認し、次を実行します。

```bash
openclaw memory status --deep --agent main
openclaw memory index --force --agent main
```

スタンドアロンの CLI コマンドと Gateway は、どちらも同じ `local` プロバイダー ID を使用します。
ローカル埋め込みを使用する場合は、`memory.search.provider: "local"` を設定します。

**結果が古いですか？** 再構築するには `openclaw memory index --force` を実行します。まれなエッジケースでは、
ウォッチャーが変更を見逃すことがあります。

**sqlite-vec が読み込まれませんか？** OpenClaw は自動的にプロセス内のコサイン
類似度へフォールバックします。`openclaw memory status --deep` はローカル
ベクトルストアを埋め込みプロバイダーとは別に報告するため、`Vector store:
unavailable` は sqlite-vec の読み込みを示し、`Embeddings: unavailable`
はプロバイダー／認証またはモデルの準備状況を示します。具体的な読み込み
エラーについては、ログを確認してください。

## 設定

埋め込みプロバイダーのセットアップ、ハイブリッド検索のチューニング（重み、MMR、時間的
減衰）、バッチインデックス作成、マルチモーダルメモリ、sqlite-vec、追加パス、および
その他すべての設定項目については、
[メモリ設定リファレンス](/ja-JP/reference/memory-config)を参照してください。

## 関連項目

- [メモリの概要](/ja-JP/concepts/memory)
- [メモリ検索](/ja-JP/concepts/memory-search)
- [Active Memory](/ja-JP/concepts/active-memory)
