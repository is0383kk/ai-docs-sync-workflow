---
read_when:
    - Mac アプリと Gateway のライフサイクルの統合
summary: macOS での Gateway のライフサイクル（launchd）
title: macOS における Gateway のライフサイクル
x-i18n:
    generated_at: "2026-07-26T10:08:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 89a27334afcecb322feb2732cf6282b4c286ef27828a1b57157f9d4fc161aed6
    source_path: platforms/mac/child-process.md
    workflow: 16
---

macOS アプリはデフォルトで **launchd** を介して Gateway を管理し、Gateway を子プロセスとして起動しません。まず、設定されたポートですでに実行中の Gateway への接続を試みます。到達可能な Gateway がない場合は、外部の `openclaw` CLI を介して launchd サービスを有効にします（組み込みランタイムは使用しません）。これにより、ログイン時の確実な自動起動と、クラッシュ時の再起動が可能になります。

子プロセスモード（アプリが Gateway を直接起動するモード）は、現在は**使用されていません**。UI とのより緊密な連携が必要な場合は、ターミナルで Gateway を手動実行してください。

## デフォルトの動作（launchd）

- アプリは、`ai.openclaw.gateway` というラベルのユーザー単位の LaunchAgent をインストールします（`--profile`/`OPENCLAW_PROFILE` を使用する場合は
  `ai.openclaw.<profile>`）。
- ローカルモードが有効な場合、アプリは LaunchAgent が読み込まれていることを確認し、必要に応じて Gateway を起動します。
- ログは launchd の Gateway ログパスに書き込まれます（デバッグ設定で確認できます）。

よく使用するコマンド：

```bash
launchctl kickstart -k gui/$UID/ai.openclaw.gateway
launchctl bootout gui/$UID/ai.openclaw.gateway
```

名前付きプロファイルを実行する場合は、ラベルを `ai.openclaw.<profile>` に置き換えてください。

## 署名なしの開発ビルド

`scripts/restart-mac.sh --no-sign` は、署名キーを使用せずにローカルビルドをすばやく作成するためのものです。launchd が署名なしのリレーバイナリを参照しないように、`~/.openclaw/disable-launchagent` を書き込みます。

`scripts/restart-mac.sh` の署名付き実行では、マーカーが存在する場合、このオーバーライドが解除されます。手動でリセットするには：

```bash
rm ~/.openclaw/disable-launchagent
```

## 接続専用モード

macOS アプリが launchd を一切インストールまたは管理しないようにするには、`--attach-only`（または `--no-launchd`）を指定して起動します。これにより `~/.openclaw/disable-launchagent` が設定されるため、アプリはすでに実行中の Gateway に接続するだけになります。デバッグ設定でも同じ動作を切り替えられます。

## リモートモード

リモートモードでは、ローカルの Gateway は起動されません。アプリはリモートホストへの SSH トンネルを使用し、そのトンネル経由で接続します。

## launchd を推奨する理由

- ログイン時の自動起動。
- 組み込みの再起動／KeepAlive セマンティクス。
- 予測可能なログと監視。

将来、本当の子プロセスモードが再び必要になった場合は、明示的な開発専用モードとして個別に文書化する必要があります。

## 関連項目

- [macOS アプリ](/ja-JP/platforms/macos)
- [Gateway 運用手順書](/ja-JP/gateway)
