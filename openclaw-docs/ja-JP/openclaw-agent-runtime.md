---
read_when:
    - OpenClaw エージェントランタイムのコードまたはテストでの作業
    - エージェントランタイムの lint、型チェック、ライブテストフローの実行
summary: OpenClaw エージェントランタイムの開発者ワークフロー：ビルド、テスト、ライブ検証
title: OpenClaw エージェントランタイムのワークフロー
x-i18n:
    generated_at: "2026-07-26T09:48:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 044f05779bef4ad18478081ba44d84356723c8a0be764440aa9d2b976d167324
    source_path: openclaw-agent-runtime.md
    workflow: 16
---

OpenClaw リポジトリ内のエージェントランタイム（`src/agents/`）向け開発ワークフロー。

## 型チェックとリント

- デフォルトのローカルゲート: `pnpm check`（型チェック、リント、ポリシーガード）
- ビルドゲート: 変更がビルド出力、パッケージング、遅延読み込み、またはモジュール境界に影響する可能性がある場合は `pnpm build`
- プッシュ前の完全なゲート: `pnpm build && pnpm check && pnpm check:test-types && pnpm test`

## エージェントランタイムテストの実行

エージェントランタイムのユニットテストスイートを実行します。

```bash
pnpm test \
  "src/agents/agent-*.test.ts" \
  "src/agents/embedded-agent-*.test.ts" \
  "src/agents/agent-hooks/**/*.test.ts"
```

最初の glob は、`agent-tools*`、`agent-settings`、および
`agent-tool-definition-adapter*` の各スイートも対象に含みます。

ライブテストはユニットテスト設定から除外されています。ライブテスト用の
ラッパーを介して実行してください（`OPENCLAW_LIVE_TEST=1` が設定され、プロバイダーの認証情報が必要です）。

```bash
pnpm test:live src/agents/embedded-agent-runner-extraparams.live.test.ts
```

## 手動テスト

- Gateway を開発モードで実行します（`OPENCLAW_SKIP_CHANNELS=1` によりチャンネル接続をスキップします）: `pnpm gateway:dev`
- Gateway を介してエージェントのターンを 1 回トリガーします: `pnpm openclaw agent --message "Hello" --thinking low`
- 対話的なデバッグには TUI を使用します: `pnpm tui`

ツール呼び出しの動作を確認するには、`read` または `exec` のアクションを実行するように指示し、
ツールのストリーミングとペイロード処理を観察できるようにします。

## クリーンな状態へのリセット

状態は OpenClaw の状態ディレクトリに保存されます。デフォルトは `~/.openclaw` で、
`$OPENCLAW_STATE_DIR` が設定されている場合はその値が使用されます。そのディレクトリからの相対パスは次のとおりです。

| パス                                           | 保存内容                                                              |
| ---------------------------------------------- | ------------------------------------------------------------------ |
| `openclaw.json`                                | 設定                                                             |
| `state/openclaw.sqlite`                        | 共有ランタイム状態データベース                                      |
| `agents/<agentId>/agent/openclaw-agent.sqlite` | エージェントごとのモデル認証プロファイル（API キー + OAuth）とランタイム状態 |
| `credentials/`                                 | 認証プロファイルストア外のプロバイダー／チャンネル認証情報        |
| `agents/<agentId>/sessions/`                   | トランスクリプト履歴と従来のセッション移行元            |
| `sessions/`                                    | 従来の単一エージェント用セッションストア（古いインストールのみ）              |
| `workspace/`                                   | デフォルトのエージェントワークスペース（追加のエージェントは `workspace-<agentId>` を使用）   |

完全にリセットするには、これらのパスを削除します。より限定的にリセットする場合:

- セッションのみ: `agents/<agentId>/agent/openclaw-agent.sqlite` は削除しないでください。セッション行は、エージェントごとの他の状態とともにそこに保存されています。1 つのチャットで新しいセッションを開始するには `/new` または `/reset` を、セッションのメンテナンスには `openclaw sessions cleanup` を使用します。
- 認証を維持: `agents/<agentId>/agent/openclaw-agent.sqlite` と `credentials/` はそのまま残します。

従来の `auth-profiles.json` ファイルは、ランタイムでは読み込まれなくなりました。
`openclaw doctor --fix` により、これらのファイルが SQLite ストアにインポートされます。

## 参考資料

- [テスト](/ja-JP/help/testing)
- [はじめに](/ja-JP/start/getting-started)

## 関連項目

- [OpenClaw エージェントランタイムのアーキテクチャ](/ja-JP/agent-runtime-architecture)
