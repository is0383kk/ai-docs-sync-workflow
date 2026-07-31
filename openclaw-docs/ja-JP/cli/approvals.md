---
read_when:
    - CLI から実行承認を編集したい場合
    - Gateway または Node ホストで許可リストを管理する必要があります
    - チャット画面を使わずに、保留中の承認を一覧表示または処理する必要がある場合
summary: '`openclaw approvals` および `openclaw exec-policy` の CLI リファレンス'
title: 承認
x-i18n:
    generated_at: "2026-07-26T08:56:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f8b6f198af718d7b058498dbb960a1eb68ced601e1cd9205070b7199688552d2
    source_path: cli/approvals.md
    workflow: 16
---

# `openclaw approvals`

**ローカルホスト**、**Gateway ホスト**、または **Node ホスト**の exec 承認を管理します。ターゲットフラグを指定しない場合、コマンドはディスク上のローカル承認ファイルを読み書きします。Gateway をターゲットにするには `--gateway`、特定の Node をターゲットにするには `--node <id|name|ip>` を使用します。

エイリアス: `openclaw exec-approvals`

関連項目: [Exec 承認](/ja-JP/tools/exec-approvals)、[Node](/ja-JP/nodes)

## `openclaw exec-policy`

`openclaw exec-policy` は、要求された `tools.exec.*` 設定とローカルホストの承認ファイルを 1 回の操作で同期する、**ローカル専用**の便利なコマンドです。

```bash
openclaw exec-policy show
openclaw exec-policy show --json

openclaw exec-policy preset yolo
openclaw exec-policy preset cautious --json

openclaw exec-policy set --host gateway --security full --ask off --ask-fallback full
```

プリセット（`yolo`、`cautious`、`deny-all`）は、`host`、`security`、`ask`、`askFallback` をまとめて適用します。`set` は渡したフラグのみを適用します。受け付けた各値は検証されます（`--host auto|sandbox|gateway|node`、`--security deny|allowlist|full`、`--ask off|on-miss|always`、`--ask-fallback deny|allowlist|full`）。

スコープ:

- ローカル設定ファイルとローカル承認ファイルをまとめて更新します。ポリシーを Gateway または Node ホストにはプッシュしません。
- `--host node` は拒否されます。Node の exec 承認は実行時に Node から取得されるため、ローカルの `exec-policy` では同期できません。代わりに `openclaw approvals set --node <id|name|ip>` を使用してください。
- `exec-policy show` は、ローカル承認ファイルから有効なポリシーを導出する代わりに、実行時に `host=node` スコープを Node 管理としてマークします。

リモートホストの承認には、`openclaw approvals set --gateway` または `openclaw approvals set --node <id|name|ip>` を直接使用してください。

## 一般的なコマンド

```bash
openclaw approvals get
openclaw approvals get --node <id|name|ip>
openclaw approvals get --gateway
openclaw approvals pending
openclaw approvals resolve <id> <allow-once|allow-always|deny>
```

`get` は、ターゲットに対する有効な exec ポリシー、つまり要求された `tools.exec` ポリシー、ホストの承認ファイルのポリシー、およびマージ後の有効な結果を表示します。Windows コンパニオンなど、ホストネイティブのポリシーを持つ Node では、OpenClaw 承認ファイルのポリシー計算を適用せず、そのポリシーを直接表示します。

ファイルベースの Node では、マージされたビューにホストで解決されたポリシーのスナップショットが必要です。古い Node では、Gateway の要求ポリシーがホストにも適用されると仮定せず、有効なポリシーを利用不可として表示します。

<Note>
セッションごとの `/exec` オーバーライドは含まれません。現在のデフォルトを確認するには、該当するセッションで `/exec` を実行してください。
</Note>

優先順位:

- ホストの承認ファイルが、強制可能な信頼できる唯一の情報源です。
- 要求された `tools.exec` ポリシーによって意図の範囲を狭めたり広げたりできますが、有効な結果はホストのルールから導出されます。
- `--node` は、Node ホストの承認ファイルと Gateway の `tools.exec` ポリシーを組み合わせます（実行時には両方が適用されます）。
- Gateway の設定を利用できない場合、CLI は Node の承認スナップショットにフォールバックし、最終的な実行時ポリシーを計算できなかったことを示します。

## 保留中の承認

Gateway から保留中の exec、Plugin、および OpenClaw システムエージェントの承認を一覧表示します。

```bash
openclaw approvals pending
openclaw approvals pending --json
```

完全な列挙と、それに対応するオペレーター全体の `resolve` フローでは `operator.admin` を使用します。そうしないと、承認レコードに要求者／レビュー担当者によるフィルタリングが残るためです。解決時には、専用の `operator.approvals` スコープも要求します。標準の CLI オペレーター権限には両方のスコープが含まれます。制限されたサードパーティクライアントは、このコマンドを模倣するためだけに管理者権限を要求すべきではありません。

人間向け出力には、承認の種類、エージェント／セッションの帰属、要求からの経過時間、有効期限までの時間、短縮されたコマンドまたは概要、およびシェルに依存しない `id64_<base64url>` ID トークンが表示されます。コンパクトな表の後には必ず `Full request text` ブロックが続き、すべての完全なトークンと可逆的にエスケープされた要求が表示されます。そのため、端末幅による短縮でサフィックスや解決に必要なトークンが隠れることはありません。完全なトークンを `resolve` にコピーしてください。ほかのフィールドに含まれる安全でない端末文字は、可視の Unicode エスケープとして表示されます。JSON 出力では、正規化されたエントリを `approvals` の下に返し、スクリプト用に元の未加工の `id`、`summary`、`createdAtMs`、`expiresAtMs` を保持します。予約済みの `id64_` 表示トークン接頭辞を使用していない限り、未加工の ID も引き続き `resolve` で受け付けられます。

指定された `id64_` 値が、リテラルの未加工 ID と別の承認のデコード済み表示トークンの両方に一致する場合、誤った要求を解決する危険を避けるため、CLI は曖昧として拒否します。

完全な ID を使用して 1 件の承認を解決します。

```bash
openclaw approvals resolve <id> allow-once
openclaw approvals resolve <id> allow-always
openclaw approvals resolve <id> deny --reason "Not expected during maintenance"
```

CLI は統合承認レコードを読み取って種類を選択し、要求された決定をレコードで許可されている決定と照合してから、統合リゾルバーを呼び出します。最初の決定が成功すると `0` で終了します。記録済みの決定を繰り返した場合も `0` で終了し、`already resolved (same decision)` を報告します。競合する決定、存在しない承認、期限切れの承認、またはその承認の種類では利用できない決定の場合、明確なエラーを表示し、0 以外で終了します。

`--reason` は、CLI の確認にローカルな注記を追加します。現在の Gateway 承認レコードには自由記述形式の解決理由フィールドがないため、この注記は永続化されず、ほかの承認サーフェスにも送信されません。

## ファイルから承認を置換

```bash
openclaw approvals set --file ./exec-approvals.json
openclaw approvals set --stdin <<'EOF'
{ version: 1, defaults: { security: "full", ask: "off", askFallback: "full" } }
EOF
openclaw approvals set --node <id|name|ip> --file ./exec-approvals.json
openclaw approvals set --gateway --file ./exec-approvals.json
```

`set` は厳密な JSON だけでなく JSON5 も受け付けます。`--file` または `--stdin` のいずれか一方を使用し、両方を使用しないでください。

ホストネイティブの Windows Node は、独自のポリシー形式を使用します。

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  defaultAction: "deny",
  rules: [{ pattern: "hostname", action: "allow" }]
}
EOF
```

CLI は最初に Node の現在のハッシュを読み取り、更新とともに送信します。そのため、同時に行われたローカル編集は上書きされず、拒否されます。この操作は Node の完全なルール一覧を置き換えるため、`rules` が必須です。`defaultAction` は任意です。ネイティブポリシーが無効であると報告する Node は、リモートで設定できません。まず、そのホストでポリシーを有効化または設定してください。ホストネイティブのポリシーは `allowlist add|remove` ヘルパーをサポートしません。

## 「プロンプトを表示しない」／YOLO の例

exec 承認で停止すべきでないホストについて、ホストの承認デフォルトを `full` + `off` に設定します。

```bash
openclaw approvals set --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

OpenClaw 承認ファイルを公開する Node では、`openclaw approvals set --node <id|name|ip> --stdin` とともに同じ本文を使用します。ホストネイティブの Node では、前述の所有者固有の形式が必要です。

これは **ホストの承認ファイル**のみを変更します。要求された OpenClaw ポリシーとの整合性を保つには、次も設定してください。

```bash
openclaw config set tools.exec.host gateway
openclaw config set tools.exec.mode full
```

ここでは `tools.exec.host=gateway` を明示しています。`host=auto` は引き続き「利用可能な場合はサンドボックス、それ以外は Gateway」を意味するためです。YOLO は承認に関するものであり、ルーティングに関するものではありません。サンドボックスが設定されている場合でもホストで exec を実行するには、`gateway`（または `/exec host=gateway`）を使用してください。

省略した `askFallback` のデフォルトは `deny` です。プロンプトを表示しない動作を維持すべき UI なしのホストをアップグレードする場合は、`askFallback: "full"` を明示的に設定してください。

同じ意図をローカルマシンだけに適用するショートカット:

```bash
openclaw exec-policy preset yolo
```

## 許可リストのヘルパー

```bash
openclaw approvals allowlist add "~/Projects/**/bin/rg"
openclaw approvals allowlist add --agent main --node <id|name|ip> "/usr/bin/uptime"
openclaw approvals allowlist add --agent "*" "/usr/bin/uname"

openclaw approvals allowlist remove "~/Projects/**/bin/rg"
```

## 共通オプション

`get`、`set`、`allowlist add|remove` はすべて次をサポートします。

- `--node <id|name|ip>`（ID、名前、IP、または ID 接頭辞を解決します。`openclaw nodes` と同じリゾルバーです）
- `--gateway`
- 共有 Node RPC オプション: `--url`、`--token`、`--timeout`、`--json`

ターゲットフラグを指定しない場合は、ディスク上のローカル承認ファイルが対象になります。

`allowlist add|remove` は `--agent <id>` もサポートします（デフォルトは `"*"` で、すべてのエージェントに適用されます）。

保留中の要求はライブの Gateway 状態であるため、`pending` と `resolve` は常に Gateway を使用します。これらは共有 Gateway 接続オプション `--url`、`--token`、`--timeout` をサポートします。`pending` は `--json` もサポートします。

## 注記

- Node ホストは `system.execApprovals.get/set` を公開する必要があります（macOS アプリ、ヘッドレス Node ホスト、または Windows コンパニオン）。
- 承認ファイルは、OpenClaw の状態ディレクトリ内にホストごとに保存されます。`$OPENCLAW_STATE_DIR/exec-approvals.json`、または変数が未設定の場合は `~/.openclaw/exec-approvals.json` です。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [Exec 承認](/ja-JP/tools/exec-approvals)
