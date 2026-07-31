---
read_when:
    - セッションやチャネルをまたいで機能する永続メモリが必要な場合
    - AI を活用した想起機能とユーザーモデリングが必要な場合
summary: Honcho Plugin による AI ネイティブなセッション間メモリ
title: Honcho メモリ
x-i18n:
    generated_at: "2026-07-26T09:18:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fadcf6d8e2505ab4fe6a81340695b7c8fee49c3cb4889665af13389941619117
    source_path: concepts/memory-honcho.md
    workflow: 16
---

[Honcho](https://honcho.dev) は、外部 Plugin を通じて OpenClaw に AI ネイティブなメモリを追加します。会話を専用サービスに永続化し、時間の経過とともにユーザーとエージェントのモデルを構築することで、ワークスペースの Markdown ファイルを超えたセッション横断コンテキストをエージェントに提供します。

## 提供される機能

- **セッション横断メモリ** - 会話は各ターン後も保持されるため、セッションのリセット、Compaction、チャンネルの切り替えをまたいでコンテキストが引き継がれます。
- **ユーザーモデリング** - Honcho は各ユーザー（好み、事実、コミュニケーションスタイル）とエージェント（個性、学習した振る舞い）のプロファイルを維持します。
- **セマンティック検索** - 現在のセッションだけでなく、過去の会話から得られた観察結果を検索します。
- **マルチエージェント認識** - 親エージェントは生成されたサブエージェントを自動的に追跡し、子セッションでは親がオブザーバーとして追加されます。

## 利用可能なツール

Honcho は、会話中にエージェントが使用できるツールを登録します。

**データ取得（高速、LLM 呼び出しなし）：**

| ツール                        | 機能                                           |
| --------------------------- | ------------------------------------------------------ |
| `honcho_context`            | セッションを横断した完全なユーザー表現               |
| `honcho_search_conclusions` | 保存された結論を対象とするセマンティック検索                |
| `honcho_search_messages`    | セッションを横断してメッセージを検索（送信者、日付で絞り込み） |
| `honcho_session`            | 現在のセッション履歴と要約                    |

**Q&A（LLM 搭載）：**

| ツール         | 機能                                                              |
| ------------ | ------------------------------------------------------------------------- |
| `honcho_ask` | ユーザーについて質問します。事実には `depth='quick'`、統合には `'thorough'` を使用します |

## はじめに

Plugin をインストールしてセットアップを実行します。

```bash
openclaw plugins install @honcho-ai/openclaw-honcho
openclaw honcho setup
openclaw gateway --force
```

セットアップコマンドは API 認証情報の入力を求め、設定を書き込み、必要に応じて既存のワークスペースメモリファイルを移行します。

<Info>
Honcho は完全にローカル（セルフホスト）で実行することも、`api.honcho.dev` のマネージド API を利用することもできます。セルフホストの場合、外部依存関係は必要ありません。
</Info>

## 設定

設定は `plugins.entries["openclaw-honcho"].config` 配下にあります。

```json5
{
  plugins: {
    entries: {
      "openclaw-honcho": {
        config: {
          apiKey: "your-api-key", // セルフホストでは省略
          workspaceId: "openclaw", // メモリの分離
          baseUrl: "https://api.honcho.dev",
        },
      },
    },
  },
}
```

セルフホストインスタンスでは、`baseUrl` にローカルサーバー（例：`http://localhost:8000`）を指定し、API キーを省略します。

## 既存メモリの移行

既存のワークスペースメモリファイル（`USER.md`、`MEMORY.md`、`IDENTITY.md`、`memory/`、`canvas/`）がある場合、`openclaw honcho setup` がそれらを検出し、移行を提案します。

<Info>
移行は非破壊的です。ファイルは Honcho にアップロードされ、元のファイルが削除または移動されることはありません。
</Info>

## 仕組み

AI の各ターン後に、会話が Honcho に永続化されます。ユーザーとエージェントの両方のメッセージが観察されるため、Honcho は時間の経過とともにモデルを構築し、改良できます。

会話中、Honcho ツールは OpenClaw の `before_prompt_build` Plugin フックでサービスに問い合わせ、モデルがプロンプトを参照する前に関連するコンテキストを注入します。

## Honcho と組み込みメモリの比較

|                   | 組み込み / QMD                | Honcho                              |
| ----------------- | ---------------------------- | ----------------------------------- |
| **ストレージ**       | ワークスペースの Markdown ファイル     | 専用サービス（ローカルまたはホスト型） |
| **セッション横断** | メモリファイル経由             | 自動、組み込み                 |
| **ユーザーモデリング** | 手動（MEMORY.md に書き込み）  | 自動プロファイル                  |
| **検索**        | ベクトル + キーワード（ハイブリッド）    | 観察結果を対象とするセマンティック検索          |
| **マルチエージェント**   | 追跡なし                  | 親子関係の認識              |
| **依存関係**  | なし（組み込み）または QMD バイナリ | Plugin のインストール                      |

Honcho と組み込みメモリシステムは併用できます。QMD を設定すると、Honcho のセッション横断メモリとあわせてローカルの Markdown ファイルを検索するための追加ツールが利用可能になります。

## CLI コマンド

```bash
openclaw honcho setup                        # API キーを設定し、ファイルを移行
openclaw honcho status                       # 接続状態を確認
openclaw honcho ask <question>               # ユーザーについて Honcho に問い合わせ
openclaw honcho search <query> [-k N] [-d D] # メモリを対象とするセマンティック検索
```

## 関連資料

- [Plugin のソースコード](https://github.com/plastic-labs/openclaw-honcho)
- [Honcho ドキュメント](https://docs.honcho.dev)
- [Honcho OpenClaw 連携ガイド](https://docs.honcho.dev/v3/guides/integrations/openclaw)

## 関連項目

- [メモリの概要](/ja-JP/concepts/memory)
- [組み込みメモリエンジン](/ja-JP/concepts/memory-builtin)
- [QMD メモリエンジン](/ja-JP/concepts/memory-qmd)
- [コンテキストエンジン](/ja-JP/concepts/context-engine)
