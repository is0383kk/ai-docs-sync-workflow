---
read_when:
    - リモート Mac 操作の設定またはデバッグ
summary: リモートの OpenClaw Gateway を制御するための macOS アプリのフロー
title: リモート制御
x-i18n:
    generated_at: "2026-07-26T10:20:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7e558c39fa173a77bf11270a8961c14c6e2350dfc4f458da3633532513b98bf6
    source_path: platforms/mac/remote.md
    workflow: 16
---

このフローにより、macOS アプリを、別のホスト（デスクトップ/サーバー）で実行されている OpenClaw Gateway の完全なリモートコントロールとして使用できます。アプリは信頼済みの LAN/Tailnet Gateway URL に直接接続するか、リモート Gateway がループバック専用の場合は SSH トンネルを管理します。ヘルスチェック、Voice Wake の転送、Web Chat では、_Settings -> General_ の同じリモート設定が再利用されます。

## モード

- **ローカル（この Mac）**：すべてがノート PC 上で実行され、SSH は使用されません。
- **SSH 経由のリモート（デフォルト）**：OpenClaw コマンドはリモートホストで実行されます。アプリは、`-o BatchMode`、選択した ID/鍵、ローカルポートフォワーディングを使用して SSH 接続を開きます。
- **リモート直接接続（ws/wss）**：SSH トンネルを使用せず、アプリが Gateway URL に直接接続します（LAN、Tailscale、Tailscale Serve、または公開 HTTPS リバースプロキシ）。

## リモートトランスポート

- **SSH トンネル**（デフォルト）：`ssh -N -L ...` を使用して Gateway ポートを localhost に転送します。トンネルがループバックであるため、Gateway からは Node の IP が `127.0.0.1` として見えます。
- **直接接続（ws/wss）**：Gateway URL に直接接続します。Gateway からは実際のクライアント IP が見えます。

選択したエイリアスで `ControlMaster` または `ForkAfterAuthentication` が有効になっている場合でも、アプリが対象のプロセスを正確に監視して再起動できるよう、アプリ自身の SSH プロセスでは SSH 接続の多重化と認証後のバックグラウンド化が無効になります。

Gateway の認証情報がこのトンネルを通過するため、SSH ホスト鍵の検証はデフォルトで厳格です。管理対象 SSH エイリアス独自の信頼動作を使用するには、`openclaw-mac configure-remote` で `--ssh-host-key-policy openssh` を設定するか、`gateway.remote.sshHostKeyPolicy` を直接 `"openssh"` に設定します。使用する前に、エイリアス、および一致する `Host *` またはシステム設定を確認してください。SSH ターゲットを変更すると（アプリ内または `configure-remote` を使用）、新しいターゲットに対して明示的に再度有効にしない限り、ポリシーは `strict` に戻ります。

SSH トンネルモードでは、検出された LAN/Tailnet ホスト名が `gateway.remote.sshTarget` として保存されます。CLI、Web Chat、ローカルの Node ホストサービスがすべて同じループバックトランスポートを使用するよう、アプリはローカルトンネルエンドポイント（例：`ws://127.0.0.1:18789`）に `gateway.remote.url` を維持します。検出によって生の Tailnet IP と安定したホスト名の両方が返された場合、アドレス変更後も接続を維持しやすくするため、アプリは Tailscale MagicDNS または LAN 名を優先します。ローカルトンネルポートがリモート Gateway ポートと異なる場合は、`gateway.remote.remotePort` をリモートホスト上のポートに設定します。

リモートモードでのブラウザ自動化は、ネイティブ macOS アプリの Node ではなく、CLI Node ホストが管理します。可能な場合、アプリはインストール済みの Node ホストサービスを起動します。その Mac からブラウザ制御を有効にするには、`openclaw node install ...` と `openclaw node start` を使用してインストール/起動するか（またはフォアグラウンドで `openclaw node run ...` を実行し）、そのブラウザ対応 Node をターゲットにします。

## リモートホストの前提条件

1. Node と pnpm をインストールし、OpenClaw CLI（`pnpm install && pnpm build && pnpm link --global`）をビルド/インストールします。
2. 非対話シェルの PATH に `openclaw` が含まれていることを確認します（必要に応じて `/usr/local/bin` または `/opt/homebrew/bin` にシンボリックリンクを作成します）。
3. SSH トランスポートの場合：鍵ベースの SSH 認証を設定します。LAN 外から安定して接続できるようにするには、Tailscale IP を推奨します。

## macOS アプリの設定

ウェルカムフローを使用せず、SSH 経由でアプリを事前設定するには、次を実行します。

```bash
openclaw-mac configure-remote \
  --ssh-target user@gateway-host \
  --local-port 18789 \
  --remote-port 18789 \
  --token "$OPENCLAW_GATEWAY_TOKEN"
```

または、信頼済みの LAN または Tailnet ですでに到達可能な Gateway の場合は、SSH を完全に省略します。

```bash
openclaw-mac configure-remote \
  --direct-url ws://192.168.0.202:18789 \
  --token "$OPENCLAW_GATEWAY_TOKEN"
```

`openclaw-mac connect`、`wizard`、および `configure-remote` は、`OPENCLAW_CONFIG_PATH`、次に `$OPENCLAW_STATE_DIR/openclaw.json`、最後に `~/.openclaw/openclaw.json` の順でアクティブな設定を解決します。どちらの設定形式でも、そのアクティブファイルに書き込み、オンボーディングを完了済みとしてマークし、次回起動時に選択したトランスポートをアプリが管理できるようにします。`--local-port`/`--remote-port` のデフォルトは `18789` です。その他のフラグ：`--password`、`--identity <path>`、`--ssh-host-key-policy <strict|openssh>`、`--project-root <path>`、`--cli-path <path>`、`--json`。完全なリファレンスを表示するには、`openclaw-mac configure-remote --help` を実行します。

代わりに UI から設定するには、次の手順を実行します。

1. _Settings -> General_ を開きます。
2. **OpenClaw runs** で **Remote** を選択し、次を設定します。
   - **Transport**：**SSH tunnel** または **Direct (ws/wss)**。
   - **SSH target**：`user@host`（`:port` は任意）。Gateway が同じ LAN 上にあり、Bonjour でアドバタイズされている場合は、検出済みリストから選択すると、このフィールドに自動入力されます。
   - **Gateway URL**（Direct のみ）：`wss://gateway.example.ts.net`（ローカル/LAN の場合は `ws://...`）。
   - **Identity file**（詳細設定）：鍵へのパス。
   - **Project root**（詳細設定）：コマンドに使用するリモートチェックアウトのパス。
   - **CLI path**（詳細設定）：実行可能な `openclaw` エントリポイント/バイナリへの任意のパス（アドバタイズされている場合は自動入力されます）。
3. **Test remote** を押します。成功した場合は、リモートの `openclaw status --json` が正しく実行されたことを意味します。失敗は通常、PATH/CLI の問題を示します。終了コード 127 は、リモートで CLI が見つからなかったことを意味します。
4. ヘルスチェックと Web Chat は、選択したトランスポートを介して自動的に実行されるようになります。

## Web Chat

- **SSH トンネル**：転送された WebSocket 制御ポート（デフォルトは 18789）を介して Gateway に接続します。
- **直接接続（ws/wss）**：設定された Gateway URL に直接接続します。
- 独立した Web Chat HTTP サーバーはありません。

## 権限

- リモートホストには、ローカルと同じ TCC 承認（オートメーション、アクセシビリティ、画面収録、マイク、音声認識、通知）が必要です。そのマシンでオンボーディングを一度実行し、承認を付与します。
- エージェントが利用可能な機能を把握できるよう、Node は `node.list` / `node.describe` を介して権限状態をアドバタイズします。

## セキュリティ上の注意

- リモートホストではループバックへのバインドを優先し、SSH、Tailscale Serve、または信頼済みの Tailnet/LAN 直接接続 URL を介して接続します。
- SSH トンネルでは、デフォルトで事前に信頼されたホスト鍵が必要です。最初にホスト鍵を信頼する（設定された known-hosts ファイルに追加する）か、OpenSSH の信頼ポリシーを受け入れられる管理対象エイリアスに対して `gateway.remote.sshHostKeyPolicy: "openssh"` を明示的に設定します。
- Gateway を非ループバックインターフェースにバインドする場合は、有効な Gateway 認証（トークン、パスワード、または `gateway.auth.mode: "trusted-proxy"` を使用する ID 対応リバースプロキシ）を必須にします。
- 直接 `wss://` 接続では、オペレーター/制御トラフィックと Mac コンパニオン Node の両方に同じ証明書ポリシーが適用されます。明示的にピン留めするには、`gateway.remote.tlsFingerprint` を設定します。設定しない場合、通常の macOS の信頼検証に成功した後にのみ、アプリは初回使用時のピンを記録します。
- [セキュリティ](/ja-JP/gateway/security)および [Tailscale](/ja-JP/gateway/tailscale)を参照してください。

## WhatsApp ログインフロー（リモート）

- `openclaw channels login --channel whatsapp --verbose` を**リモートホスト上で**実行します。スマートフォンの WhatsApp で QR コードをスキャンします。
- 認証の有効期限が切れた場合は、そのホストで再度ログインします。ヘルスチェックにリンクの問題が表示されます。

## トラブルシューティング

| 症状                                          | 原因 / 修正方法                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `exit 127` / 見つからない                           | 非ログインシェルの PATH に `openclaw` がありません。`/etc/paths`、使用しているシェルの rc、または `/usr/local/bin`/`/opt/homebrew/bin` へのシンボリックリンクとして追加してください。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ヘルスプローブに失敗                              | SSH の到達可能性、PATH、および Baileys（WhatsApp）がログイン済みであること（`openclaw status --json`）を確認してください。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Web Chat が停止したまま                                   | リモートホストで Gateway が実行中であり、転送ポートが Gateway の WS ポートと一致していることを確認してください。UI には正常な WS 接続が必要です。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Node IP に `127.0.0.1` と表示される                        | SSH トンネル使用時の想定どおりの動作です。Gateway から実際のクライアント IP を参照できるようにするには、**Transport** を **Direct (ws/wss)** に切り替えてください。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ダッシュボードは動作するが Mac の機能がオフライン | オペレーター/制御接続は正常ですが、コンパニオン Node 接続が確立されていないか、コマンドサーフェスがありません。メニューバーのデバイスセクションを開き、Mac が `paired · disconnected` かどうか確認してください。直接の `wss://` オペレーター接続と Node 接続では、構成済みまたは保存済みの同じ証明書ポリシーが使用されます。信頼済みの `wss://*.ts.net` Tailscale Serve エンドポイントでは、証明書のローテーション後に古くなった保存済みリーフピンが置き換えられ、自動的に再試行されます。構成済みのピンは自動的にローテーションされません。新しい証明書を確認した後で `gateway.remote.tlsFingerprint` を更新するか、**Remote over SSH** に切り替えてください。 |
| Voice Wake                                       | リモートモードではトリガーフレーズが自動的に転送されるため、別のフォワーダーは必要ありません。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |

## 通知音

`openclaw nodes notify` を使用して、スクリプトから通知ごとにサウンドを選択します。例：

```bash
openclaw nodes notify --node <id> --title "Ping" --body "Remote gateway ready" --sound Glass
```

アプリにはグローバルなデフォルトサウンドの切り替えはありません。呼び出し元がリクエストごとにサウンドを選択します（またはサウンドなしを選択します）。

## 関連項目

- [macOS アプリ](/ja-JP/platforms/macos)
- [リモートアクセス](/ja-JP/gateway/remote)
