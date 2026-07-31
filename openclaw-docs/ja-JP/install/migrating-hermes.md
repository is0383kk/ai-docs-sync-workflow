---
read_when:
    - Hermes から移行し、モデル設定、プロンプト、メモリ、Skills を引き続き使用したい場合
    - OpenClaw が自動的にインポートするものと、アーカイブ専用のまま保持されるものを確認する場合
    - クリーンでスクリプト化された移行手順が必要な場合（CI、新しいノートPC、自動化）
summary: プレビュー可能で元に戻せるインポートを使用して、Hermes から OpenClaw に移行する
title: Hermes からの移行
x-i18n:
    generated_at: "2026-07-26T09:46:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f8cdb7a77cfb8ecb0504ccc322b5600c6ed671a8bf9ac866d964fdf4b3494000
    source_path: install/migrating-hermes.md
    workflow: 16
---

バンドルされている Hermes 移行プロバイダーは `HERMES_HOME` とアクティブな Hermes プロファイルに従い、macOS/Linux では `~/.hermes`、Windows では `%LOCALAPPDATA%\hermes` にフォールバックします。適用前にすべての変更をプレビューし、計画とレポートではシークレットを秘匿します。スタンドアロンの `openclaw migrate` は検証済みのバックアップを書き込みます。新規オンボーディングのパスでは設定、認証情報、ファイルをステージングし、インポートした推論が検証された後にのみ公開します。明示的な `--from` パスが常に優先されます。

<Note>
インポートには新規の OpenClaw セットアップが必要です。ローカルに OpenClaw の状態がすでにある場合は、まず設定、認証情報、セッション、ワークスペースをリセットするか、計画を確認した後に `--overwrite` とともに `openclaw migrate apply hermes` を直接使用してください。
</Note>

## 2 つのインポート方法

<Tabs>
  <Tab title="オンボーディングウィザード">
    アクティブな Hermes ホーム/プロファイルを検出し、適用前にプレビューを表示します。

    ```bash
    openclaw onboard --flow import
    ```

    または、特定のソースを指定します。

    ```bash
    openclaw onboard --import-from hermes --import-source ~/.hermes
    ```

  </Tab>
  <Tab title="CLI">
    スクリプト化された実行や反復可能な実行には `openclaw migrate` を使用します。完全なリファレンスについては、[`openclaw migrate`](/ja-JP/cli/migrate) を参照してください。

    ```bash
    openclaw migrate hermes --dry-run    # プレビューのみ
    openclaw migrate apply hermes --yes  # 確認を省略して適用
    ```

    Hermes ホーム/プロファイルの検出を上書きするには、`--from <path>` を追加します。

  </Tab>
</Tabs>

## インポートされる内容

<AccordionGroup>
  <Accordion title="モデル設定">
    - Hermes `config.yaml` からのデフォルトモデルの選択。
    - 現在の Hermes Chat Completions、Codex Responses、Anthropic Messages トランスポートを含む、`model`、`providers`、`custom_providers` からの設定済みモデルプロバイダーとカスタムエンドポイント。

  </Accordion>
  <Accordion title="MCP サーバー">
    `mcp_servers` または `mcp.servers` からの MCP サーバー定義。無効状態、タイムアウト、並列ツール対応、OAuth スコープ、互換性のある TLS フィールド、ネイティブ/リソース/プロンプトツールポリシーが含まれます。リテラルの環境変数とヘッダーには、認証情報のインポートへの同意が必要です。Hermes 専用のライフサイクル、サンプリング、情報要求、プリフライト、キープアライブ、CA バンドル、パスワード保護されたクライアントキー、事前登録済み OAuth クライアントの設定は、無効な OpenClaw 設定になるのではなく、手動確認項目になります。
  </Accordion>
  <Accordion title="ワークスペースファイル">
    - `SOUL.md` と `AGENTS.md` は OpenClaw エージェントワークスペースにコピーされます。
    - `memories/MEMORY.md` と `memories/USER.md` は、対応する OpenClaw メモリファイルを上書きせず、そこに**追記**されます。
    - メモリ専用のサーフェスでは動作が異なります。オンボーディングのメモリページと Control UI の Memory インポートページでは、この 2 つのファイルをインデックス付きの想起用として `memory/imports/hermes/` 配下にコピーし、既存のワークスペースメモリは変更しません。

  </Accordion>
  <Accordion title="メモリ設定">
    OpenClaw ファイルメモリ用のメモリ設定のデフォルト値。Honcho などの外部メモリプロバイダーは、意図的に移行できるよう、アーカイブ項目または手動確認項目として記録されます。
  </Accordion>
  <Accordion title="Skills">
    `skills/` 配下の任意の場所に `SKILL.md` ファイルがある Skills は再帰的に検出され、OpenClaw ワークスペースの Skills ディレクトリにフラット化され、サポートファイルとともにコピーされます。`skills.config` の Skills ごとの設定値は保持されます。
  </Accordion>
  <Accordion title="認証情報">
    対話型の `openclaw migrate` は認証情報をインポートする前に確認し、デフォルトでは「はい」が選択されています。受け入れられるインポートには、現在の Hermes OpenAI Codex OAuth エントリ、OpenCode OpenAI OAuth と GitHub Copilot のエントリ、および[サポート対象の Hermes `.env` キー](/ja-JP/cli/migrate#supported-env-keys)が含まれます。非対話型インポートには `--include-secrets`、認証情報をスキップするには `--no-auth-credentials`、またはオンボーディングの `--import-secrets` フラグを使用します。Hermes OAuth をインポートした後は、Hermes と OpenClaw で同じ更新許可を使用し続けないでください。両方を実行する前に、どちらか一方を再認証してください。
  </Accordion>
</AccordionGroup>

## アーカイブのみに保持される内容

プロバイダーは手動確認用として以下を移行レポートディレクトリにコピーしますが、稼働中の OpenClaw 設定や認証情報には読み込み**ません**。

- `plugins/`
- `sessions/`
- `logs/`
- `cron/`
- `mcp-tokens/`
- `plans/`、`workspace/`、`skins/`、`kanban/`
- `pairing/` と `platforms/` のストア、および Gateway のルーティング/プロセス状態
- `state.db`、`hermes_state.db`、`projects.db`、`response_store.db`、`memory_store.db`、`verification_evidence.db`、`kanban.db`、`retaindb_queue.db`

システム間で形式や信頼に関する前提が変化する可能性があるため、OpenClaw はこの状態を自動的に実行したり信頼したりしません。アーカイブを確認した後、必要なものを手動で移動してください。

## 推奨フロー

<Steps>
  <Step title="計画をプレビュー">
    ```bash
    openclaw migrate hermes --dry-run
    ```

    計画には、競合、スキップされた項目、機密項目を含め、変更されるすべての内容が一覧表示されます。ネストされたシークレットらしいキーは出力で秘匿されます。

  </Step>
  <Step title="バックアップ付きで適用">
    ```bash
    openclaw migrate apply hermes --yes
    ```

    OpenClaw は適用前にバックアップを作成して検証します。この非対話型の例では、シークレットではない状態のみをインポートします。認証情報の確認に対話形式で回答するには `--yes` なしで実行し、無人実行にサポート対象の認証情報を含めるには `--include-secrets` を追加します。

  </Step>
  <Step title="doctor を実行">
    ```bash
    openclaw doctor
    ```

    [Doctor](/ja-JP/gateway/doctor) は保留中の設定移行を再適用し、インポート中に発生した問題がないか確認します。

  </Step>
  <Step title="再起動して検証">
    ```bash
    openclaw gateway restart
    openclaw status
    ```

    Gateway が正常であり、インポートしたモデル、メモリ、Skills が読み込まれていることを確認します。

  </Step>
</Steps>

## 競合処理

計画で競合（対象にファイルまたは設定値がすでに存在する）が報告されると、適用は続行を拒否します。

<Warning>
既存の対象を置き換えることが意図した操作である場合に限り、`--overwrite` を指定して再実行してください。プロバイダーは、上書きされたファイルについて、移行レポートディレクトリに項目単位のバックアップを引き続き書き込む場合があります。
</Warning>

新規インストールで競合が発生することはまれです。通常は、ユーザーによる編集がすでに存在するセットアップに対してインポートを再実行した場合に発生します。

適用の途中で競合が発生した場合（たとえば、設定ファイルで予期しない競合状態が発生した場合）、その項目は競合として報告されますが、独立したファイル、Skills、認証情報、アーカイブ、設定エントリの処理は続行されます。競合した項目を解決してインポートを再実行してください。同一のメモリインポートは冪等です。

## シークレット

対話型の `openclaw migrate` は、検出された認証情報をインポートするかどうかを確認し、デフォルトでは「はい」が選択されています。

- 受け入れると、現在の Hermes OpenAI Codex OAuth エントリ、OpenCode OpenAI OAuth と GitHub Copilot のエントリ、および[サポート対象の `.env` キー](/ja-JP/cli/migrate#supported-env-keys)がインポートされます。
- シークレットではない状態のみをインポートするには、`--no-auth-credentials` を使用するか、プロンプトで「いいえ」と回答します。
- 無人の `--yes` 実行で認証情報をインポートするには、`--include-secrets` を使用します。
- ウィザードから認証情報をインポートするには、オンボーディングウィザードの `--import-secrets` フラグを使用します。

## 自動化用の JSON 出力

```bash
openclaw migrate hermes --dry-run --json
openclaw migrate apply hermes --json --yes
```

`--json` を指定し、`--yes` を指定しない場合、適用は計画を出力しますが状態を変更しません。これは CI と共有スクリプトにとって最も安全なモードです。

## トラブルシューティング

<AccordionGroup>
  <Accordion title="競合により適用が拒否される">
    計画の出力を確認してください。各競合には、ソースパスと既存の対象が示されます。項目ごとに、スキップするか、対象を編集するか、`--overwrite` を指定して再実行するかを決定します。
  </Accordion>
  <Accordion title="Hermes が ~/.hermes 以外にある">
    `--from /actual/path`（CLI）または `--import-source /actual/path`（オンボーディング）を渡します。
  </Accordion>
  <Accordion title="既存のセットアップでオンボーディングがインポートを拒否する">
    オンボーディングによるインポートには新規セットアップが必要です。状態をリセットしてオンボーディングをやり直すか、`--overwrite` と明示的なバックアップ制御をサポートする `openclaw migrate apply hermes` を直接使用してください。
  </Accordion>
  <Accordion title="API キーがインポートされなかった">
    対話型の `openclaw migrate` は、認証情報のプロンプトで同意した場合にのみ API キーをインポートします。非対話型の `--yes` 実行には `--include-secrets` が必要であり、オンボーディングによるインポートには `--import-secrets` が必要です。[サポート対象の `.env` キー](/ja-JP/cli/migrate#supported-env-keys)のみが認識され、その他の `.env` 変数は無視されます。
  </Accordion>
</AccordionGroup>

## 関連項目

- [`openclaw migrate`](/ja-JP/cli/migrate): 完全な CLI リファレンス、Plugin コントラクト、JSON 形式。
- [オンボーディング](/ja-JP/cli/onboard): ウィザードのフローと非対話型フラグ。
- [移行](/ja-JP/install/migrating): OpenClaw のインストール環境をマシン間で移動する方法。
- [Doctor](/ja-JP/gateway/doctor): 移行後の健全性チェック。
- [エージェントワークスペース](/ja-JP/concepts/agent-workspace): `SOUL.md`、`AGENTS.md`、メモリファイルの保存場所。
