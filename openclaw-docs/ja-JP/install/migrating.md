---
read_when:
    - OpenClaw を新しいノートパソコンまたはサーバーに移行する場合
    - 別のエージェントシステムから移行し、状態を維持したい場合
    - 既存の Plugin をその場でアップグレードしています
summary: 移行ハブ：システム間インポート、マシン間移行、Plugin のアップグレード
title: 移行ガイド
x-i18n:
    generated_at: "2026-07-26T10:07:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e9ceb80045ab082c9cfc9e1aca59e079b6bf28b1d047265a0be40c03ebe5dac6
    source_path: install/migrating.md
    workflow: 16
---

OpenClaw は、別のエージェントシステムからのインポート、既存のインストール環境の新しいマシンへの移動、Plugin のインプレースアップグレードという 3 つの移行パスをサポートしています。

## 別のエージェントシステムからインポートする

同梱の移行プロバイダーは、指示、MCP サーバー、Skills、モデル設定、および（オプトインの）API キーを OpenClaw に取り込みます。変更を加える前にプランがプレビューされ、レポートではシークレットが秘匿されます。スタンドアロンの `openclaw migrate` は検証済みのバックアップによって保護されます。一方、新規オンボーディングでのインポートでは、ローカルアーティファクトをステージングして検証してから公開し、不可逆な外部アクティベーションを行う前に設定をコミットします。

<CardGroup cols={2}>
  <Card title="Claude からの移行" href="/ja-JP/install/migrating-claude" icon="brain">
    `CLAUDE.md`、MCP サーバー、Skills、プロジェクトコマンドを含む Claude Code と Claude Desktop の状態をインポートします。
  </Card>
  <Card title="Hermes からの移行" href="/ja-JP/install/migrating-hermes" icon="feather">
    Hermes の設定、プロバイダー、MCP サーバー、メモリ、Skills、およびサポートされている `.env` キーをインポートします。
  </Card>
</CardGroup>

CLI のエントリーポイントは [`openclaw migrate`](/ja-JP/cli/migrate) です。既知のソース（`openclaw onboard --flow import`）を検出した場合、オンボーディングでも移行を提案できます。

## OpenClaw を新しいマシンに移動する

以下を保持するには、**状態ディレクトリ**（デフォルトでは `~/.openclaw/`）と**ワークスペース**をコピーします。

- **設定** — `openclaw.json` とすべての Gateway 設定。
- **認証** — エージェントごとの `auth-profiles.json`（API キーと OAuth）、および `credentials/` 配下のチャンネルまたはプロバイダーの状態。
- **セッション** — 会話履歴とエージェントの状態。
- **チャンネルの状態** — WhatsApp のログイン、Telegram のセッションなど。
- **ワークスペースファイル** — `MEMORY.md`、`USER.md`、Skills、プロンプト。

<Tip>
古いマシンで `openclaw status` を実行し、状態ディレクトリのパスを確認してください。カスタムプロファイルでは、`~/.openclaw-<profile>/`、または `OPENCLAW_STATE_DIR` で設定したパスが使用されます。
</Tip>

### 移行手順

<Steps>
  <Step title="Gateway を停止してバックアップする">
    **古い**マシンで、コピー中にファイルが変更されないように Gateway を停止してから、アーカイブを作成します。

    ```bash
    openclaw gateway stop
    cd ~
    tar -czf openclaw-state.tgz .openclaw
    ```

    複数のプロファイル（例: `~/.openclaw-work`）を使用している場合は、それぞれを個別にアーカイブします。

  </Step>

  <Step title="新しいマシンに OpenClaw をインストールする">
    新しいマシンに CLI（必要に応じて Node も）を[インストール](/ja-JP/install)します。オンボーディングによって新しい `~/.openclaw/` が作成されても問題ありません。次の手順で上書きします。
  </Step>

  <Step title="状態ディレクトリとワークスペースをコピーする">
    `scp`、`rsync -a`、または外付けドライブを使用してアーカイブを転送し、展開します。

    ```bash
    cd ~
    tar -xzf openclaw-state.tgz
    ```

    隠しディレクトリが含まれていること、およびファイルの所有者が Gateway を実行するユーザーと一致していることを確認します。

  </Step>

  <Step title="Doctor を実行して検証する">
    新しいマシンで [Doctor](/ja-JP/gateway/doctor) を実行し、設定の移行とサービスの修復を行います。

    ```bash
    openclaw doctor
    openclaw gateway restart
    openclaw status
    ```

  </Step>
</Steps>

Telegram または Discord がデフォルトの環境変数フォールバック（`TELEGRAM_BOT_TOKEN` または `DISCORD_BOT_TOKEN`）を使用している場合は、シークレット値を出力せずに、移行した状態ディレクトリの `.env` にそれらのキーが含まれていることを確認します。

```bash
awk -F= '/^(TELEGRAM_BOT_TOKEN|DISCORD_BOT_TOKEN)=/ { print $1 "=present" }' ~/.openclaw/.env
```

有効なデフォルトの Telegram または Discord アカウントにトークンが設定されておらず、対応する環境変数を Doctor プロセスで利用できない場合、`openclaw doctor` も警告します。

### よくある問題

<AccordionGroup>
  <Accordion title="プロファイルまたは状態ディレクトリの不一致">
    古い Gateway が `--profile` または `OPENCLAW_STATE_DIR` を使用していて、新しい Gateway が使用していない場合、チャンネルはログアウト状態として表示され、セッションは空になります。移行したものと**同じ**プロファイルまたは状態ディレクトリを指定して Gateway を起動し、`openclaw doctor` を再実行してください。
  </Accordion>

  <Accordion title="openclaw.json のみをコピーする">
    設定ファイルだけでは不十分です。モデル認証プロファイルは `agents/<agentId>/agent/auth-profiles.json` 配下にあり、チャンネルとプロバイダーの状態は `credentials/` 配下にあります。必ず状態ディレクトリ**全体**を移行してください。
  </Accordion>

  <Accordion title="権限と所有権">
    root としてコピーした場合やユーザーを切り替えた場合、Gateway が認証情報を読み取れないことがあります。状態ディレクトリとワークスペースが、Gateway を実行するユーザーによって所有されていることを確認してください。
  </Accordion>

  <Accordion title="リモートモード">
    UI が**リモート**の Gateway を参照している場合、セッションとワークスペースはリモートホストが所有しています。ローカルのノートパソコンではなく、Gateway ホスト自体を移行してください。[FAQ](/ja-JP/help/faq#where-things-live-on-disk) を参照してください。
  </Accordion>

  <Accordion title="バックアップ内のシークレット">
    状態ディレクトリには、認証プロファイル、チャンネルの認証情報、その他のプロバイダーの状態が含まれています。バックアップは暗号化して保存し、安全でない転送経路を避け、漏洩した疑いがある場合はキーをローテーションしてください。
  </Accordion>
</AccordionGroup>

### 検証チェックリスト

新しいマシンで、以下を確認します。

- [ ] `openclaw status` に Gateway が実行中であると表示される。
- [ ] チャンネルが引き続き接続されている（再ペアリングは不要）。
- [ ] ダッシュボードが開き、既存のセッションが表示される。
- [ ] ワークスペースファイル（メモリ、設定）が存在する。

## Plugin をインプレースアップグレードする

Plugin のインプレースアップグレードでは、同じ Plugin ID と設定キーが維持されますが、ディスク上の状態が現在のレイアウトに移動される場合があります。Plugin 固有のアップグレードガイドは、それぞれのチャンネルとともに提供されています。

- [Matrix の移行](/ja-JP/channels/matrix-migration): 暗号化された状態の復旧制限、自動スナップショットの動作、手動復旧コマンド。

## 関連項目

- [`openclaw migrate`](/ja-JP/cli/migrate): システム間インポート用の CLI リファレンス。
- [インストールの概要](/ja-JP/install): すべてのインストール方法。
- [Doctor](/ja-JP/gateway/doctor): 移行後の健全性チェック。
- [アンインストール](/ja-JP/install/uninstall): OpenClaw を完全に削除する方法。
