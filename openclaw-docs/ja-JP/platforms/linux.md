---
read_when:
    - Linux コンパニオンアプリの状況を確認する
    - Linux Node ホストでカメラ、位置情報、または通知を有効にする
    - プラットフォーム対応またはコントリビューションの計画
    - VPS またはコンテナでの Linux OOM kill や終了コード 137 のデバッグ
summary: Linux のサポートとコンパニオンアプリの状況
title: Linux アプリ
x-i18n:
    generated_at: "2026-07-26T09:29:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fe55d3ec63fcf8291a24126c04638f005c03c3d44ff84a26a925e931066b01cc
    source_path: platforms/linux.md
    workflow: 16
---

Gateway は Linux で完全にサポートされており、Node が必要です。Bun は引き続き
依存関係のインストーラーやパッケージスクリプトのランナーとして使用できますが、
`node:sqlite` を提供しないため、OpenClaw を実行することはできません。

## デスクトップコンパニオン

OpenClaw Linux コンパニオンは、ローカル Gateway 用の Tauri デスクトップアプリです。次の機能があります。

- OpenClaw CLI と管理対象の Node ランタイムがない場合にインストールします。リリースビルドでは stable チャネルを自動的にインストールし、開発ビルドでは最初にチャネルを確認します
- サービスの変更を試みる前に、正常な Gateway に接続します
- インストール、開始、停止、再起動の操作を、CLI が管理する systemd ユーザーサービスに委譲します
- 近くの Bonjour Gateway を検出し、それぞれのコントロール UI をルート単位のウィンドウで開くため、複数の
  Gateway ダッシュボードに接続したまま同時に使用できます
- 解決済みの認証 URL を使用して、Gateway が提供するコントロール UI を開きます
- 初回実行時のインストール後、コントロール UI をオンボーディングモードで開きます。ここでは、
  検出された Claude Code、Codex、Hermes のメモリをエージェントワークスペースに
  インポートできます（同じインポート機能は、後から
  設定 → メモリをインポートでも利用できます）
- 同一ホスト上の CLI Node ホスト向けに、エージェント駆動の Canvas と同梱の A2UI コンテンツをレンダリングします
- ウィンドウを閉じてもシステムトレイから引き続き利用できます

`main` からビルドされた stable リリースでは、タグの
[GitHub リリース](https://github.com/openclaw/openclaw/releases)に、
`OpenClaw-<version>-amd64.deb` および `OpenClaw-<version>-amd64.AppImage` という名前の
`.deb` バンドルと AppImage バンドルがアセットとして提供され、
その横に `SHA256SUMS.linux-app.txt` チェックサムファイルが配置されます。
`.deb` をダウンロードし、`sudo apt install ./OpenClaw-<version>-amd64.deb` でインストールするか、
AppImage に実行権限を付与して直接実行します。AppImage ランタイムには
FUSE 2（`sudo apt install libfuse2`、Ubuntu 24.04+ では `libfuse2t64`）が必要です。
利用できない場合は、`APPIMAGE_EXTRACT_AND_RUN=1` を指定して AppImage を実行してください。

ソースチェックアウトから同じバンドルをビルドすることもできます。

```bash
cd apps/linux/src-tauri
pnpm dlx @tauri-apps/cli@2.11.4 build --bundles deb,appimage
```

`Linux App` CI ワークフローは、アプリに変更を加えるプルリクエストおよび
手動実行時に、同じバンドルを `openclaw-linux-companion` アーティファクトとして
アップロードします。Linux のビルド依存関係と開発コマンドについては、
リポジトリ内の `apps/linux/README.md` を参照してください。

### クイックチャット

`Ctrl+Shift+Space` またはトレイの**クイックチャット**項目からクイックチャットを開きます。エージェント
チップには、設定されたアバター、絵文字、またはモノグラムが表示されます。選択するとエージェントを切り替えられます。
メッセージでは、選択したエージェントのメインセッションが使用され、グローバルセッションスコープが適用されます。
ネイティブ Rust クライアントは、永続的な Ed25519 デバイス ID を管理します。
CLI ハンドオフの共有トークンまたはパスワードはペアリングの開始時にのみ使用し、その後は
Gateway が発行したデバイストークンを保存して、以降の接続で優先的に使用します。ID と
デバイストークンは、アプリの設定ディレクトリにあるモード `0600` のファイルに保存されます。
クイックチャットの WebView には、認証情報も WebSocket も渡されません。

ネイティブ接続を利用できない場合、クイックチャットには**Gateway に接続できません —
再試行中**と表示され、再接続するまで送信が無効になります。ペアリング段階に到達した
リモートデバイスには、代わりに**ダッシュボード（ノード）でこのデバイスを承認してください**
と表示され、Gateway から提供された場合は短いデバイス ID も表示されます。必要な共有認証情報が
ない Gateway では、**Gateway には認証情報が必要です — Gateway ホストでダッシュボードを
開いてください**と表示されます。この状態では、承認待ちのペアリング要求はありません。
サーバーからより具体的な修復ガイダンスが提供された場合は、これらの代替通知が置き換えられます。
TLS Gateway の場合、CLI は Gateway 証明書の SHA-256 フィンガープリントをアプリに渡します。
ネイティブクライアントはその証明書をピン留めし、停止状態とは別に
**Gateway の TLS 信頼検証に失敗しました — 証明書のフィンガープリントを確認してください**
と報告します。共有シークレットが SecretRef を介して設定されている Gateway では、
CLI ハンドオフから共有シークレットが省略されます。既存のペアリング済みインストールは、
保存されたデバイストークンにより引き続き動作しますが、新規インストールでは、その起動用認証情報がない場合、
共有シークレット認証下で保留中のペアリング要求を作成できません。
セットアップコードと `bootstrapToken` の引き換えには専用の製品 UI が必要であり、
今後の対応項目です。クイックチャットはどちらのフローも試行しません。

X11 では、クイックチャットの歯車からカスタムショートカットを記録またはリセットできます。
トレイの**クイックチャットのショートカット**切り替えでは、通常の
**クイックチャット**トレイ項目を無効にせず、ショートカットだけを有効または無効にできます。
Wayland ではグローバルショートカットを利用できないため、ショートカット設定は非表示になり、
トレイ項目が引き続き起動手段になります。送信が受け付けられると、クイックチャットは開いたままになり、
選択したエージェントのプレーンテキストの応答が入力欄の下にストリーミング表示されます。
`Esc` を押すとバーと応答を閉じます。
`Ctrl+Enter` では引き続きダッシュボードが開きます。

### Canvas

Linux Canvas は、連携する 2 つのプロセスを使用します。`openclaw node run` は単一の Gateway Node 接続として維持され、同梱の `linux-canvas` Plugin は、`canvas.*` 呼び出しをユーザー専用の Unix ソケット経由で実行中のデスクトップアプリに転送します。アプリは、同梱の A2UI レンダラーとエージェントへのアクションブリッジを含む、オンデマンドの WebView ウィンドウを 1 つ管理します。

Plugin はデフォルトで有効です。デスクトップソケットが `$XDG_RUNTIME_DIR/openclaw-canvas.sock` に存在する場合、または `XDG_RUNTIME_DIR` を利用できないときに `/tmp/openclaw-canvas-$UID.sock` に存在する場合にのみ、Canvas を通知します。`plugins.entries.linux-canvas.enabled: false` で無効にできます。デスクトップアプリのないヘッドレス Linux サーバーでは、Canvas は通知されません。

Linux v1 は Canvas ウィンドウを 1 つ使用します。HTTP および HTTPS ページをレンダリングできますが、A2UI アクションは同梱のレンダラーからのみ受け付けます。

## CLI と SSH による代替手段

ヘッドレスサーバー、VPS、またはリモート Gateway では、CLI が引き続き最も簡単な選択肢です。

1. Node 24.15+（推奨）、Node 22.22.3+（LTS）、または Node 25.9+ をインストールします。
2. `npm i -g openclaw@latest`
3. `openclaw onboard --install-daemon`
4. ノート PC から: `ssh -N -L 18789:127.0.0.1:18789 <user>@<host>`
5. `http://127.0.0.1:18789/` を開き、設定済みの共有
   シークレットで認証します（デフォルトはトークン。`gateway.auth.mode` が `"password"` の場合はパスワード）。

完全なサーバーガイド: [Linux サーバー](/ja-JP/vps)。手順形式の VPS の例:
[exe.dev](/ja-JP/install/exe-dev)。

## Node の機能

同梱の Linux Node Plugin は、デスクトップアプリを必要とせず、CLI に `openclaw node` サービスデバイス機能を提供します。コマンドは、その機能が有効で、必要なローカルツールが存在する場合にのみ Gateway に通知されます。

| 機能                              | デフォルト | 要件                                                           |
| --------------------------------------- | ------- | --------------------------------------------------------------------- |
| デスクトップ通知（`system.notify`） | オン      | libnotify の `notify-send` とデスクトップ通知セッション       |
| カメラの写真とクリップ（`camera.*`）    | オフ     | FFmpeg、V4L2 カメラアクセス、およびクリップ音声用の PulseAudio または PipeWire |
| 位置情報（`location.get`）               | オフ     | GeoClue2 とその `where-am-i` デモ                                    |

`openclaw.json` で Plugin を設定します。

```json5
{
  plugins: {
    entries: {
      "linux-node": {
        config: {
          notify: { enabled: true },
          camera: { enabled: true },
          location: { enabled: true },
        },
      },
    },
  },
}
```

これらの設定を変更した後は、Node サービスを再起動してください。可用性はプロセスごとに一度だけ判定され、Node の通知内容は再起動時に再構築されます。

Gateway は、Node のコマンドおよび機能の範囲を、デバイスのペアリングとは別に承認します。初回起動時、または機能を追加で有効にした後は、保留中の範囲を承認します。

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
```

Node は接続およびデバイスのペアリングが完了していても、この承認が完了するまで、実効的な `caps` と `commands` が空のままになることがあります。

カメラデバイスは、一般的には `video` グループを介して、サービスユーザーが読み取れる必要があります。`includeAudio` が true の場合、カメラクリップではデフォルトの PulseAudio または PipeWire ソースが使用されます。マイク音声はそのクリップの音声トラックとしてのみ存在し、独立したコマンドとしては利用できません。位置情報を使用するには、ホストの GeoClue ポリシーで Node サービスユーザーが許可されている必要があります。

`camera.snap` と `camera.clip` では、`gateway.nodes.commands.allow` を介した明示的な Gateway の有効化も必要です。ペイロード、制限、エラーについては、[カメラキャプチャ](/ja-JP/nodes/camera)と[位置情報コマンド](/ja-JP/nodes/location-command)を参照してください。

## インストール

- [はじめに](/ja-JP/start/getting-started)
- [インストールと更新](/ja-JP/install/updating)
- 任意: [Bun パッケージワークフロー](/ja-JP/install/bun)、[Nix](/ja-JP/install/nix)、[Docker](/ja-JP/install/docker)

## Gateway サービス（systemd）

次のいずれかでインストールします。

```bash
openclaw onboard --install-daemon
openclaw gateway install
openclaw configure   # select "Gateway service" when prompted
```

既存のインストールを修復または移行します。

```bash
openclaw doctor
```

`openclaw gateway install` は、デフォルトで systemd **ユーザー**ユニットを生成します。共有ホストまたは
常時稼働ホスト向けの**システム**レベルのユニット構成を含む完全な
サービスガイダンスについては、[Gateway 運用手順](/ja-JP/gateway#supervision-and-service-lifecycle)を参照してください。

カスタム構成の場合にのみ、ユニットを手動で作成してください。最小限のユーザーユニットの例
（`~/.config/systemd/user/openclaw-gateway[-<profile>].service`）:

```ini
[Unit]
Description=OpenClaw Gateway (profile: <profile>, v<version>)
After=network-online.target
Wants=network-online.target
StartLimitBurst=5
StartLimitIntervalSec=60

[Service]
ExecStart=/usr/local/bin/openclaw gateway --port 18789
Restart=always
RestartSec=5
RestartPreventExitStatus=78
TimeoutStopSec=30
TimeoutStartSec=30
SuccessExitStatus=0 143
OOMPolicy=continue
KillMode=control-group

[Install]
WantedBy=default.target
```

手動で作成したユニットには、`openclaw gateway install` が管理対象の Gateway サービスに設定する適応的ヒープサイズ調整が継承されません。管理対象インストーラーを使用するか、ネイティブメモリ用の余裕を考慮したうえで、カスタムスーパーバイザーに明示的なヒープ制限を設定してください。

有効にします。

```bash
systemctl --user enable --now openclaw-gateway[-<profile>].service
```

## メモリ負荷と OOM kill

Linux では、ホスト、VM、またはコンテナの cgroup がメモリ不足になると、
カーネルが OOM の対象プロセスを選択します。Gateway は長時間存続する
セッションとチャネル接続を管理しているため、OOM の対象として適切ではありません。
そのため OpenClaw は、可能な場合に一時的な子プロセスが先に終了されるよう調整します。

対象となる Linux の子プロセス起動では、OpenClaw はコマンドを短い
`/bin/sh` シムでラップし、子プロセス自身の `oom_score_adj` を `1000` に引き上げた後、
実際のコマンドを `exec` します。これは特権を必要としません。プロセスは常に自身の
OOM スコアを引き上げることができます。

対象となる子プロセスの範囲:

- スーパーバイザーが管理するコマンドの子プロセス
- PTY シェルの子プロセス
- MCP stdio サーバーの子プロセス
- OpenClaw が起動したブラウザー／Chrome プロセス（Plugin SDK のプロセスランタイム経由）

このラッパーは Linux 専用で、`/bin/sh` を利用できない場合、または
子プロセスの環境で `OPENCLAW_CHILD_OOM_SCORE_ADJ` が `0`、`false`、`no`、または
`off` に設定されている場合は使用されません。

子プロセスを確認します。

```bash
cat /proc/<child-pid>/oom_score_adj
```

対象となる子プロセスの期待値は `1000` です。Gateway プロセス自体は
通常のスコア（通常は `0`）を維持します。

systemd ユニットの `OOMPolicy=continue` により、一時的な子プロセスが OOM killer に選択された場合でも、
ユニット全体を失敗扱いにしてすべてのチャネルを再起動することなく、
Gateway サービスが稼働し続けます。失敗した子プロセスまたはセッションは、
それぞれ独自のエラーを報告します。

これは通常のメモリ調整に代わるものではありません。VPS またはコンテナで子プロセスが
繰り返し終了される場合は、メモリ制限を引き上げるか、同時実行数を減らすか、
より強力なリソース制御（systemd `MemoryMax=`、コンテナのメモリ制限）を追加してください。

## 関連項目

- [インストールの概要](/ja-JP/install)
- [Linux サーバー](/ja-JP/vps)
- [Raspberry Pi](/ja-JP/install/raspberry-pi)
- [Gateway 運用手順書](/ja-JP/gateway)
- [Gateway の設定](/ja-JP/gateway/configuration)
