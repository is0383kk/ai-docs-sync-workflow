---
read_when: Browser control fails on Linux, especially with snap Chromium
summary: Linux 上の OpenClaw ブラウザ制御における Chrome/Brave/Edge/Chromium の CDP 起動問題を修正する
title: ブラウザのトラブルシューティング
x-i18n:
    generated_at: "2026-07-26T10:32:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8e5db2da2d43129862f0c005213df828f6eae81f5561e57d41795ea90787822a
    source_path: tools/browser-linux-troubleshooting.md
    workflow: 16
---

## 問題: ポート 18800 で Chrome CDP を起動できない

```json
{ "error": "エラー: プロファイル \"openclaw\" の Chrome CDP をポート 18800 で起動できませんでした。" }
```

### 根本原因

Ubuntu およびほとんどの Linux ディストリビューションでは、`apt install chromium` をインストールすると、実際のブラウザではなく snap
ラッパーがインストールされます。

```text
注: 'chromium' の代わりに 'chromium-browser' を選択しています
chromium-browser はすでに最新バージョン (2:1snap1-0ubuntu2) です。
```

Snap の AppArmor による制限が、OpenClaw によるブラウザプロセスの起動と監視を妨げます。

Linux でよく発生するその他の起動エラー:

- `The profile appears to be in use by another Chromium process`: 管理対象プロファイルディレクトリに古い
  `Singleton*` ロックファイルが残っています。ロックが終了済みまたは
  別ホストのプロセスを指している場合、OpenClaw はこれらのロックを削除し、
  1 回再試行します。
- `Missing X server or $DISPLAY`: デスクトップセッションがないホストで、
  表示可能なブラウザが明示的に要求されています。Linux では、`DISPLAY` と `WAYLAND_DISPLAY` の両方が未設定の場合、
  ローカルの管理対象プロファイルはヘッドレスモードにフォールバックします。
  `OPENCLAW_BROWSER_HEADLESS=0`、`browser.headless: false`、または
  `browser.profiles.<name>.headless: false` を設定している場合は、そのヘッド付きオーバーライドを削除するか、
  `OPENCLAW_BROWSER_HEADLESS=1` を設定するか、`Xvfb` を起動するか、
  1 回限りの管理対象起動用に `openclaw browser start --headless` を実行するか、実際のデスクトップセッションで
  OpenClaw を実行してください。

### 解決策 1: Google Chrome をインストールする（推奨）

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
sudo apt --fix-broken install -y  # 依存関係エラーがある場合
```

`~/.openclaw/openclaw.json` を更新します。

```json
{
  "browser": {
    "enabled": true,
    "executablePath": "/usr/bin/google-chrome-stable",
    "headless": true,
    "noSandbox": true
  }
}
```

### 解決策 2: snap Chromium をアタッチ専用モードで使用する

snap Chromium を使い続ける必要がある場合は、OpenClaw がブラウザを起動する代わりに、
手動で起動したブラウザへアタッチするよう設定します。

```json
{
  "browser": {
    "enabled": true,
    "attachOnly": true,
    "headless": true,
    "noSandbox": true
  }
}
```

Chromium を手動で起動します。

```bash
chromium-browser --headless --no-sandbox --disable-gpu \
  --remote-debugging-port=18800 \
  --user-data-dir=$HOME/.openclaw/browser/openclaw/user-data \
  about:blank &
```

必要に応じて、systemd ユーザーサービスで自動起動します。

```ini
# ~/.config/systemd/user/openclaw-browser.service
[Unit]
Description=OpenClaw ブラウザ (Chrome CDP)
After=network.target

[Service]
ExecStart=/snap/bin/chromium --headless --no-sandbox --disable-gpu --remote-debugging-port=18800 --user-data-dir=%h/.openclaw/browser/openclaw/user-data about:blank
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable --now openclaw-browser.service
```

### ブラウザが動作することを確認する

```bash
curl -s http://127.0.0.1:18791/ | jq '{running, pid, chosenBrowser}'
curl -s -X POST http://127.0.0.1:18791/start
curl -s http://127.0.0.1:18791/tabs
```

### 設定リファレンス

| オプション                      | 説明                                                          | デフォルト                                                            |
| --------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `browser.enabled`           | ブラウザ制御を有効にする                                               | `true`                                                             |
| `browser.executablePath`    | Chromium ベースのブラウザバイナリへのパス（Chrome/Brave/Edge/Chromium） | 自動検出（OS のデフォルトブラウザが Chromium ベースの場合はそれを優先） |
| `browser.headless`          | GUI なしで実行する                                                      | `false`                                                            |
| `OPENCLAW_BROWSER_HEADLESS` | ローカルの管理対象ブラウザのヘッドレスモードに対するプロセス単位のオーバーライド         | 未設定                                                              |
| `browser.noSandbox`         | `--no-sandbox` フラグを追加する（一部の Linux 環境で必要）               | `false`                                                            |
| `browser.attachOnly`        | ブラウザを起動せず、既存のブラウザにのみアタッチする              | `false`                                                            |

Raspberry Pi、古い VPS ホスト、または低速なストレージでは、Chrome が CDP HTTP
エンドポイントを公開して準備完了になるまでに管理対象ブラウザの期限より長い時間を要する場合、
`attachOnly` を指定して手動で起動したブラウザを使用してください。

### 問題: profile="user" の Chrome タブが見つからない

`user`（`existing-session` / Chrome MCP）プロファイルを使用していますが、
アタッチできるタブが開いていません。

修正方法:

1. 代わりに管理対象ブラウザを使用します:
   `openclaw browser --browser-profile openclaw start`（または
   `browser.defaultProfile: "openclaw"` を設定）。
2. ローカルの Chrome で少なくとも 1 つのタブを開いたままにしてから、
   `--browser-profile user` で再試行します。

注:

- `user` はホスト専用です。Linux サーバー、コンテナ、またはリモートホストでは、
  代わりに CDP プロファイルを使用してください。
- `user` およびその他の `existing-session` プロファイルには、現在の Chrome MCP
  の制限が共通して適用されます。参照駆動型アクションのみ、アップロードごとに 1 ファイル、ダイアログの `timeoutMs`
  オーバーライドなし、`wait --load networkidle` なし、さらに `responsebody`、PDF エクスポート、
  ダウンロードのインターセプト、バッチアクションも使用できません。
- ローカルの `openclaw` ドライバープロファイルでは、`cdpPort`/`cdpUrl` が自動的に割り当てられます。これらを
  手動で設定するのはリモート CDP の場合だけにしてください。
- リモート CDP プロファイルでは、`http://`、`https://`、`ws://`、`wss://` を使用できます。
  `/json/version` の検出には HTTP(S) を使用します。ブラウザサービスから DevTools ソケットの
  直接 URL が提供される場合は WS(S) を使用します。

## 関連項目

- [ブラウザ](/ja-JP/tools/browser)
- [ブラウザへのログイン](/ja-JP/tools/browser-login)
- [ブラウザの WSL2 トラブルシューティング](/ja-JP/tools/browser-wsl2-windows-remote-cdp-troubleshooting)
