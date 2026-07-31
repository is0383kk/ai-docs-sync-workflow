---
read_when:
    - macOS の Skills 設定 UI の更新
    - Skills のゲーティングまたはインストール動作の変更
summary: macOS の Skills 設定 UI と Gateway 経由のステータス
title: Skills (macOS)
x-i18n:
    generated_at: "2026-07-26T09:08:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fd9d8f1190320889029335e008c3605bd4bf0194f83398cedd4ae658fd90065c
    source_path: platforms/mac/skills.md
    workflow: 16
---

macOS アプリは Gateway 経由で OpenClaw の Skills を表示します。Skills をローカルで解析することはありません。

## データソース

- `skills.status`（Gateway）は、すべての Skills に加え、利用可否と不足している要件（バンドルされた Skills の許可リストによるブロックを含む）を返します。
- 要件は、各 `SKILL.md` の `metadata.openclaw.requires` から取得されます。

## インストール操作

- `metadata.openclaw.install` はインストールオプション（brew/node/go/uv/download）を定義します。
- アプリは `skills.install` を呼び出し、Gateway ホスト上でインストーラーを実行します。
- オペレーターが管理する `security.installPolicy`（`enabled`、`targets`、`exec`）は、インストーラーのメタデータが処理される前に、Gateway を介した Skills のインストールをブロックできます。組み込みの危険コードスキャン（Plugin のインストールに使用）は、Skills のインストールフローには組み込まれていません。
- すべてのインストールオプションが `download` の場合、Gateway はすべてのダウンロード選択肢を表示します。
- それ以外の場合、Gateway は現在のインストール設定（`skills.install.preferBrew`、`skills.install.nodeManager`）とホスト上のバイナリに基づいて、優先するインストーラーを 1 つ選択します。`preferBrew` が有効で `brew` が存在する場合は最初に Homebrew、次に `uv`、次に設定された Node マネージャー、次に利用可能であれば再度 Homebrew（`preferBrew` がなくても）、次に `go`、最後に `download` の順です。
- Node のインストールラベルには、`yarn` を含め、設定された Node マネージャーが反映されます。

## 環境変数／API キー

- アプリは、`skills.entries.<skillKey>` の下にある `~/.openclaw/openclaw.json` にキーを保存します。
- `skills.update` は `enabled`、`apiKey`、`env` にパッチを適用します。

## リモートモード

- インストールと設定の更新は、ローカルの Mac ではなく Gateway ホスト上で行われます。

## 関連項目

- [Skills](/ja-JP/tools/skills)
- [macOS アプリ](/ja-JP/platforms/macos)
