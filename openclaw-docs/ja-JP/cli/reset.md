---
read_when:
    - CLI はインストールしたまま、ローカルの状態を消去したい場合
    - 削除される内容をドライランで確認したい場合
summary: '`openclaw reset`（ローカルの状態/設定をリセット）の CLI リファレンス'
title: リセット
x-i18n:
    generated_at: "2026-07-26T09:36:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 54f1d320ee368dae4a4bfb32dea73d19eb35f9f30edd12d9c2580ab7e6a26fa6
    source_path: cli/reset.md
    workflow: 16
---

# `openclaw reset`

ローカルの設定/状態をリセットします（CLI はインストールされたままです）。

```bash
openclaw reset
openclaw reset --dry-run
openclaw reset --scope config --yes --non-interactive
openclaw reset --scope config+creds+sessions --yes --non-interactive
openclaw reset --scope full --yes --non-interactive
```

## オプション

- `--scope <scope>`: `config`、`config+creds+sessions`、または `full`
- `--yes`: 確認プロンプトをスキップ
- `--non-interactive`: プロンプトを無効化。`--scope` と `--yes` が必要
- `--dry-run`: ファイルを削除せずに実行内容を表示

## スコープ

| スコープ                   | 削除対象                                                                     | 最初に Gateway を停止 |
| ----------------------- | --------------------------------------------------------------------------- | ------------------- |
| `config`                | 設定ファイルのみ                                                            | いいえ                  |
| `config+creds+sessions` | 設定ファイル、OAuth/認証情報ディレクトリ、エージェントごとのセッションディレクトリ           | はい                 |
| `full`                  | 状態ディレクトリ（共有 SQLite データベースを含む）とワークスペースディレクトリ | はい                 |

`config+creds+sessions` と `full` は、状態を削除する前に、実行中の管理対象 Gateway サービスを停止します。

## 注記

- ローカル状態を削除する前に、まず `openclaw backup create` を実行して復元可能なスナップショットを作成してください。
- ワークスペースのセットアップ状態と証明は共有 SQLite データベース内の行として保存されるため、`full` は状態ディレクトリとともにこれらを削除します。現在、個別に削除する証明のサイドカーファイルはありません。
- `--scope` を指定しない場合、`openclaw reset` は削除するスコープを対話形式で確認します。
- `--non-interactive` は、`--scope` と `--yes` の両方が設定されている場合にのみ有効です。
- `config+creds+sessions` と `full` は、完了時に `Next: openclaw onboard --install-daemon` を出力します。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
