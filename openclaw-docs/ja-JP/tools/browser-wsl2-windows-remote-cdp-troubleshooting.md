---
read_when:
    - Chrome が Windows 上にある環境で WSL2 内で OpenClaw Gateway を実行する
    - WSL2 と Windows の両方で重複して発生するブラウザ／コントロール UI エラーの確認
    - 分離ホスト構成でホストローカルの Chrome MCP と未加工のリモート CDP のどちらを選ぶか
summary: WSL2 Gateway + Windows Chrome のリモート CDP をレイヤーごとにトラブルシューティングする
title: WSL2 + Windows + リモート Chrome CDP のトラブルシューティング
x-i18n:
    generated_at: "2026-07-26T09:19:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 66ec4ed5bfccc66b594a43d56296c69242e8b9cf50b36c6cb3990b1d6ea58faa
    source_path: tools/browser-wsl2-windows-remote-cdp-troubleshooting.md
    workflow: 16
---

一般的な分離ホスト構成では、OpenClaw Gateway は WSL2 内で動作し、Chrome は
Windows 上で動作するため、ブラウザー制御は WSL2/Windows の境界を越える必要があります。複数の
独立した問題が同時に発生する可能性があります（
[issue #39369](https://github.com/openclaw/openclaw/issues/39369) を参照）：CDP
トランスポート、Control UI のオリジンセキュリティ、トークン/ペアリングは、それぞれ
単独で失敗しても似たようなエラーを生成することがあります。どれが壊れているかを推測せず、
以下のレイヤーを順番に確認してください。

## まず適切なブラウザーモードを選択する

### オプション 1：WSL2 から Windows への直接リモート CDP

WSL2 から Windows Chrome の CDP
エンドポイントを指すリモートブラウザープロファイルを使用します。Gateway を WSL2 内で稼働させ、Chrome を
Windows 上で実行し、ブラウザー制御で WSL2/Windows の境界を越える必要がある場合に選択してください。

### オプション 2：ホストローカル Chrome MCP

Gateway が Chrome と同じホストで動作し、ローカルのログイン済みブラウザー状態を使用したい場合にのみ、
`existing-session` ドライバー（`user` プロファイル）を使用してください。この構成では、
ホスト間のブラウザートランスポートが不要であり、`responsebody`、
PDF エクスポート、ダウンロードのインターセプト、バッチアクションも不要である必要があります（Chrome MCP プロファイルは
これらをサポートしていません）。

WSL2 Gateway + Windows Chrome では、直接リモート CDP を使用してください。Chrome MCP は
ホストローカルであり、WSL2 から Windows へのブリッジではありません。

## 動作するアーキテクチャ

- WSL2 は `127.0.0.1:18789` で Gateway を実行する
- Windows は通常のブラウザーで `http://127.0.0.1:18789/` の Control UI を開く
- Windows Chrome はポート `9222` で CDP エンドポイントを公開する
- WSL2 からその Windows CDP エンドポイントに到達できる
- OpenClaw は WSL2 から到達可能なアドレスをブラウザープロファイルに指定する

## Control UI に関する重要なルール

Windows から UI を開く場合、意図的に HTTPS を設定している場合を除き、
Windows の localhost を使用してください。

```text
http://127.0.0.1:18789/
```

LAN IP をデフォルトで使用しないでください。LAN または tailnet アドレス上のプレーン HTTP は、
CDP 自体とは無関係な安全でないオリジン/デバイス認証の動作を
引き起こす可能性があります。[Control UI](/ja-JP/web/control-ui) を参照してください。

## レイヤーごとに検証する

上から下へ進め、途中を飛ばさないでください。1 つのレイヤーを修正しても、
さらに下の別のレイヤーのエラーが引き続き表示される場合があります。

### レイヤー 1：Windows 上で Chrome が CDP を提供していることを確認する

```powershell
chrome.exe --remote-debugging-port=9222 --user-data-dir="$env:LOCALAPPDATA\OpenClaw\ChromeCDP"
```

Chrome 136 以降では、デフォルトの Chrome データディレクトリに対する
リモートデバッグのコマンドラインスイッチは無視されます。上記のように、デフォルトではない別のデータディレクトリを
使用してください。Chrome の
[リモートデバッグのセキュリティ変更](https://developer.chrome.com/blog/remote-debugging-port)
を参照してください。これによって、通常のログイン済み Chrome プロファイルをリモート制御できるようになるわけではありません。

まず Windows から Chrome 自体を確認します。

```powershell
curl.exe http://127.0.0.1:9222/json/version
curl.exe http://127.0.0.1:9222/json/list
```

これが失敗する場合は、以下の Windows リスナーを診断してください。この段階ではまだ OpenClaw が
問題ではありません。

#### portproxy を変更する前に IPv4 と IPv6 を診断する

Chromium は最初にリモートデバッグを `127.0.0.1` にバインドしようとし、IPv4 のバインドが失敗した場合にのみ
`[::1]` にフォールバックします。`127.0.0.1:9222` でリッスンする永続的な `v4tov4` ルールが、
Chrome の起動前にそのエンドポイントを占有することがあります。その場合 Chrome は
`[::1]:9222` にフォールバックする一方、古いルールは IPv4 トラフィックを自身のリスナーへ転送し直し、
空の応答を返します。

Chrome のバージョンから推測するのではなく、Windows から実際のリスナーとプロキシルールを
確認してください。

```powershell
netstat -ano | findstr :9222
netsh interface portproxy show all
curl.exe http://127.0.0.1:9222/json/version
curl.exe http://[::1]:9222/json/version
```

`netstat` に表示される各 PID について `tasklist /fi "PID eq <PID>"` を使用してください。

- `chrome.exe` が `127.0.0.1` で応答する場合は、`127.0.0.1:9222` でも
  リッスンしている portproxy ルールを削除してください。WSL2 から到達可能な Windows アダプターの
  アドレスだけを `127.0.0.1` に転送します。
- `chrome.exe` が `[::1]` でのみ応答する場合は、使用されていない IPv4 アドレスへ転送する代わりに、
  `v4tov6` を使用して、WSL2 から到達可能なリスナーを
  `::1` に向けてください。

  ```powershell
  netsh interface portproxy add v4tov6 listenaddress=WINDOWS_HOST_OR_IP listenport=9222 connectaddress=::1 connectport=9222
  ```

リスナーは WSL2 が必要とするアダプターアドレスにバインドしてください。CDP
ポートを `0.0.0.0`、LAN アドレス、または tailnet アドレスで公開しないでください。CDP は
ブラウザーセッションの制御権を付与します。

### レイヤー 2：WSL2 からその Windows エンドポイントに到達できることを確認する

WSL2 から、`cdpUrl` で使用する予定の正確なアドレスをテストします。

```bash
curl http://WINDOWS_HOST_OR_IP:9222/json/version
curl http://WINDOWS_HOST_OR_IP:9222/json/list
```

正常な結果：

- `/json/version` は Browser / Protocol-Version メタデータを含む JSON を返す
- `/json/list` は JSON を返す（ページが開かれていなければ空の配列でも問題ありません）

これが失敗する場合、Windows はまだ WSL2 にポートを公開していない、WSL2 側で
アドレスが正しくない、またはファイアウォール/ポート転送/プロキシが不足しています。OpenClaw の設定を変更する前に、
その問題を修正してください。

### レイヤー 3：適切なブラウザープロファイルを設定する

WSL2 から到達可能なアドレスを OpenClaw に指定します。

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "remote",
    profiles: {
      remote: {
        cdpUrl: "http://WINDOWS_HOST_OR_IP:9222",
        attachOnly: true,
        color: "#00AA00",
      },
    },
  },
}
```

注意事項：

- Windows 上でしか動作しないアドレスではなく、WSL2 から到達可能なアドレスを使用する
- 外部で管理されるブラウザーでは `attachOnly: true` を維持する
- `cdpUrl` には `http://`、`https://`、`ws://`、または `wss://` を指定できる
- OpenClaw に `/json/version` を検出させる場合は HTTP(S) を使用する
- ブラウザープロバイダーから直接 DevTools
  ソケット URL が提供される場合にのみ WS(S) を使用する
- OpenClaw の成功を期待する前に、同じ URL を `curl` でテストする

### レイヤー 4：Control UI レイヤーを個別に確認する

Windows から `http://127.0.0.1:18789/` を開き、次を確認します。

- ページのオリジンが `gateway.controlUi.allowedOrigins` の想定と一致している
- トークン認証またはペアリングが正しく設定されている
- Control UI の認証問題をブラウザーの
  問題としてデバッグしていない

参考ページ：[Control UI](/ja-JP/web/control-ui)。

### レイヤー 5：エンドツーエンドのブラウザー制御を確認する

WSL2 から実行します。

```bash
openclaw browser --browser-profile remote open https://example.com
openclaw browser --browser-profile remote tabs
```

正常な結果：

- Windows Chrome でタブが開く
- `browser tabs` がターゲットを返す
- 後続のアクション（`snapshot`、`screenshot`、`navigate`）が同じ
  プロファイルで動作する

## 誤解を招きやすい一般的なエラー

| メッセージ                                                                                 | 意味                                                                                                                                                                           |
| --------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `control-ui-insecure-auth`                                                              | CDP トランスポートの問題ではなく、UI オリジン/セキュアコンテキストの問題                                                                                                                     |
| `token_missing`                                                                         | 認証設定の問題                                                                                                                                                        |
| `pairing required`                                                                      | デバイス承認の問題                                                                                                                                                           |
| `Remote CDP for profile "remote" is not reachable`                                      | WSL2 から設定済みの `cdpUrl` に到達できない                                                                                                                                         |
| portproxy 経由で空の CDP 応答 / `other side closed`                               | Windows リスナーの不一致または自己ループ。両方のループバックファミリーと `netsh interface portproxy show all` を確認する                                                                 |
| `Browser attachOnly is enabled and CDP websocket for profile "remote" is not reachable` | HTTP エンドポイントは応答したが、DevTools WebSocket を開けなかった                                                                                                        |
| リモートセッション後も古いビューポート / ダークモード / ロケール / オフラインのオーバーライドが残る          | `openclaw browser --browser-profile remote stop` を実行してセッションを閉じ、Gateway や外部ブラウザーを再起動せずにキャッシュされた Playwright/CDP 接続を解放する |
| CDP 到達性の確認中にタイムアウト                                                         | 通常は引き続き CDP 到達性の問題、または低速/到達不能なリモートエンドポイント                                                                                                             |
| `Playwright page enumeration timed out after 3000ms`                                    | リモート CDP には接続したが、永続タブの読み取りが停止した                                                                                                                     |
| `No Chrome tabs found for profile="user"`                                               | ホストローカルのタブが利用できない場所でローカル Chrome MCP プロファイルが選択されている                                                                                                          |

## 高速トリアージチェックリスト

1. Windows：`127.0.0.1` と `[::1]` のどちらが `/json/version` で応答し、
   そのリスナーは `chrome.exe` に属しているか？
2. WSL2：`curl http://WINDOWS_HOST_OR_IP:9222/json/version` は動作するか？
3. OpenClaw の設定：`browser.profiles.<name>.cdpUrl` は、その WSL2 から到達可能な正確な
   アドレスを使用しているか？
4. Control UI：LAN IP ではなく `http://127.0.0.1:18789/` を開いているか？
5. 直接リモート CDP ではなく、WSL2 と Windows をまたいで `existing-session` を使用しようとして
   いないか？

まず Windows Chrome のエンドポイントをローカルで確認し、次に同じエンドポイントを
WSL2 から確認してから、OpenClaw の設定または Control UI の認証をデバッグしてください。

## 関連項目

- [ブラウザー](/ja-JP/tools/browser)
- [ブラウザーログイン](/ja-JP/tools/browser-login)
- [ブラウザーの Linux トラブルシューティング](/ja-JP/tools/browser-linux-troubleshooting)
