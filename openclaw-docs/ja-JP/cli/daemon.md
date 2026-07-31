---
read_when:
    - スクリプトでは引き続き `openclaw daemon ...` を使用します
    - サービスのライフサイクルコマンド（install/start/stop/restart/status）が必要です
summary: '`openclaw daemon` の CLI リファレンス（Gateway サービス管理用のレガシーエイリアス）'
title: デーモン
x-i18n:
    generated_at: "2026-07-26T08:57:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 629852ebf3efe86dedc4c84f6ddc9349b25ddde832df5d78521641fe4b137658
    source_path: cli/daemon.md
    workflow: 16
---

# `openclaw daemon`

Gateway サービス管理用の旧エイリアスです。`openclaw daemon ...` は、`openclaw gateway ...` と同じサービス制御コマンドにマッピングされます。最新のドキュメントと例については、[`openclaw gateway`](/ja-JP/cli/gateway) を使用してください。

## 使用方法

```bash
openclaw daemon status
openclaw daemon install
openclaw daemon start
openclaw daemon stop
openclaw daemon restart
openclaw daemon uninstall
```

## サブコマンドとオプション

| サブコマンド  | オプション                                                                                          |
| ----------- | ------------------------------------------------------------------------------------------------ |
| `status`    | `--url`, `--token`, `--password`, `--timeout`, `--no-probe`, `--require-rpc`, `--deep`, `--json` |
| `install`   | `--port`, `--runtime <node>`, `--token`, `--wrapper <path>`, `--force`, `--json`                 |
| `uninstall` | `--json`                                                                                         |
| `start`     | `--json`                                                                                         |
| `stop`      | `--json`, `--disable`（launchd のみ：次回の起動まで KeepAlive/RunAtLoad を継続的に抑制） |
| `restart`   | `--force`, `--safe`, `--skip-deferral`, `--wait <duration>`, `--json`                            |

- `status`：サービスのインストール状態（launchd/systemd/schtasks）を表示し、Gateway の正常性をプローブします。
- `install`：サービスをインストールします。`--force` は既存のインストールを再インストールまたは上書きします。
- `restart --safe`：実行中の Gateway にアクティブな処理の事前確認を要求し、処理が完了した後に統合された再起動を 1 回スケジュールします。上限は 5 分です。この時間制限を超えると、再起動はいずれにしても強制されます。通常の `restart` はサービスマネージャーを直接使用し、`--force` は即時実行のオーバーライドです。
- `restart --safe --skip-deferral`：アクティブな処理による延期ゲートを迂回し、ブロッカーが報告されている場合でも Gateway を直ちに再起動します。`--safe` が必要です。

## 注意事項

- `status`：可能な場合、プローブ認証用に設定された認証 SecretRef を解決します。必要な SecretRef が未解決の場合、`status --json` は `rpc.authWarning` を報告します。`--token`/`--password` を明示的に渡すか、先にシークレットソースを解決してください。それ以外の点でプローブが成功すると、未解決の認証に関する警告は抑制されます。
- `status --deep`：Gateway に類似する他のサービスを対象としたベストエフォートのシステムレベルスキャンを追加し（クリーンアップのヒントを表示しますが、マシンごとに 1 つの Gateway という推奨事項は変わりません）、Plugin を認識するモードで設定検証を実行します。これにより、高速なデフォルトパスでは省略される Plugin マニフェストの警告が表示されます。
- Linux の systemd インストールでは、トークンドリフトのチェックで `Environment=` と `EnvironmentFile=` の両方のユニットソースを検査します。
- トークンドリフトのチェックでは、マージされたランタイム環境（最初にサービスコマンド環境、次にプロセス環境）を使用して `gateway.auth.token` SecretRef を解決します。トークン認証が実質的に有効でない場合（`password`/`none`/`trusted-proxy` の `gateway.auth.mode`、またはパスワードが優先され得る状態で未設定の場合）、設定トークンの解決は省略されます。
- `install`：SecretRef で管理される `gateway.auth.token` が解決可能であることを検証しますが、解決した値をサービス環境メタデータに永続化することはありません。解決できない場合、インストールは安全側に失敗します。
- `gateway.auth.token` と `gateway.auth.password` の両方が設定され、`gateway.auth.mode` が未設定の場合、モードを明示的に設定するまで `install` は処理をブロックします。
- macOS では、`install` は、`EnvironmentVariables` にシークレットを埋め込む代わりに、LaunchAgent の plist と生成された環境ファイル／ラッパーを所有者のみアクセス可能な状態（モード `0600`/`0700`）に維持します。
- 1 台のホストで複数の Gateway を実行する場合は、ポート、設定／状態、ワークスペースを分離してください。[複数の Gateway](/ja-JP/gateway#multiple-gateways-same-host)を参照してください。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [Gateway 運用手順書](/ja-JP/gateway)
