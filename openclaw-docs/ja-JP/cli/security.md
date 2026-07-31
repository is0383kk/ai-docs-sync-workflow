---
read_when:
    - 設定/状態に対して簡単なセキュリティ監査を実行したい場合
    - 安全な「修正」の提案（権限、デフォルト設定の厳格化）を適用したい場合
summary: '`openclaw security` の CLI リファレンス（一般的なセキュリティ上の落とし穴を監査して修正）'
title: セキュリティ
x-i18n:
    generated_at: "2026-07-26T08:59:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b5f9ea5cb746bfd29ff4d096062e81595abe99a883fc3b1113b45a3527d42d9
    source_path: cli/security.md
    workflow: 16
---

# `openclaw security`

セキュリティツール：監査と、必要に応じた安全な修正。関連：[セキュリティ](/ja-JP/gateway/security)。

```bash
openclaw security audit
openclaw security audit --deep
openclaw security audit --deep --password <password>
openclaw security audit --deep --token <token>
openclaw security audit --auth password --password <password>
openclaw security audit --fix
openclaw security audit --json
```

## 監査モード

通常の `security audit` は、コールドな設定／ファイルシステム／読み取り専用のパスで実行されます。Plugin ランタイムのセキュリティコレクターを検出しないため、定期監査でインストール済みのすべての Plugin ランタイムが読み込まれることはありません。`--deep` は、ベストエフォートのライブ Gateway プローブと、Plugin が所有するセキュリティ監査コレクターを追加します（明示的な内部呼び出し元も、適切なランタイムスコープをすでに保持している場合は、それらのコレクターを使用できます）。

Gateway のパスワード認証を起動時にのみ指定している場合は、監査で `hooks.token` と照合できるよう、`--auth password --password <password>` に同じ値を渡してください。

## チェック内容

**DM／信頼モデル**

- 複数の DM 送信者がメインセッションを共有している場合に警告し、共有受信トレイには安全な DM モードである `session.dmScope="per-channel-peer"`（マルチアカウントチャネルの場合は `per-account-channel-peer`）を推奨します。これは協調運用／共有受信トレイの強化であり、互いに信頼できないオペレーター間の分離ではありません。そのような場合は、個別の Gateway（または個別の OS ユーザー／ホスト）を使用して信頼境界を分離してください。
- 設定から複数ユーザーによる受信が見込まれる場合（たとえば、オープンな DM／グループポリシー、設定済みのグループターゲット、ワイルドカード送信者ルール）に `security.trust_model.multi_user_heuristic` を出力します。OpenClaw のデフォルトの信頼モデルはパーソナルアシスタント（オペレーター 1 人）であり、敵対的なマルチテナント分離ではありません。意図的に複数ユーザーで使用する場合は、すべてのセッションをサンドボックス化し、ファイルシステムアクセスをワークスペースのスコープ内に制限し、個人用／非公開の ID や認証情報をそのランタイムに置かないでください。
- 小規模モデル（パラメーター数 `<=300B`）が、サンドボックス化されずに Web／ブラウザツールを有効にして使用されている場合に警告します。

**Webhook／フック**

起動ログに致命的でないセキュリティ警告が記録され、監査では、有効な Gateway の共有シークレット認証値（`gateway.auth.token`／`OPENCLAW_GATEWAY_TOKEN`、`gateway.auth.password`／`OPENCLAW_GATEWAY_PASSWORD`）を再利用している `hooks.token` にフラグを付けます。また、次の場合にも警告します。

- `hooks.token` が短い
- `hooks.path="/"`
- `hooks.defaultSessionKey` が未設定
- `hooks.allowedAgentIds` が無制限
- リクエストの `sessionKey` オーバーライドが有効
- `hooks.allowedSessionKeyPrefixes` なしでオーバーライドが有効

`openclaw doctor --fix` を実行して、永続化されている再利用済みの `hooks.token` をローテーションし、外部フックの送信元を更新して新しいトークンを使用するようにしてください。

**サンドボックス／ツール**

- サンドボックスモードがオフの状態でサンドボックスの Docker 設定が構成されている場合に警告します。
- `gateway.nodes.commands.deny` で効果のないパターン風のエントリや不明なエントリが使用されている場合に警告します（照合はノードコマンド名との完全一致のみであり、シェルテキストのフィルタリングではありません）。
- `gateway.nodes.commands.allow` が危険なノードコマンドを明示的に有効にしている場合に警告します。
- グローバルな `tools.profile="minimal"` がエージェントのツールプロファイルによって上書きされている場合に警告します。
- 書き込み／編集ツールが無効でも、制約を設けるサンドボックスのファイルシステム境界なしで `exec` が引き続き利用可能な場合に警告します。
- オープンな DM またはグループで、サンドボックス／ワークスペースのガードなしにランタイム／ファイルシステムツールが公開されている場合に警告します。
- 制限の緩いツールポリシーのもとで、インストール済み Plugin のツールにアクセスできる可能性がある場合に警告します。

**サンドボックスブラウザ**

- サンドボックスブラウザが `sandbox.browser.cdpSourceRange` なしで Docker の `bridge` ネットワークを使用している場合に警告します。
- `host` や `container:*` の名前空間への参加など、危険なサンドボックス Docker ネットワークモードにフラグを付けます。
- 既存のサンドボックスブラウザの Docker コンテナでハッシュラベルが欠落している、または古くなっている場合（たとえば、移行前のコンテナに `openclaw.browserConfigEpoch` がない場合）に警告し、`openclaw sandbox recreate --browser --all` を推奨します。

**ネットワーク／検出**

- `gateway.allowRealIpFallback=true` にフラグを付けます（プロキシの設定に誤りがある場合、ヘッダー偽装のリスクがあります）。
- `discovery.mdns.mode="full"` にフラグを付けます（mDNS TXT レコードによるメタデータ漏えい）。
- `gateway.auth.mode="none"` によって、共有シークレットなしで Gateway HTTP API（`/tools/invoke` と、有効なすべての `/v1/*` エンドポイント）にアクセス可能な状態になっている場合に警告します。

**Plugin／チャネル**

- npm ベースの Plugin／フックのインストール記録が固定されていない、整合性メタデータがない、または現在インストールされているパッケージのバージョンと乖離している場合に警告します。
- チャネルの許可リストが、安定した ID ではなく変更可能な名前／メールアドレス／タグに依存している場合に警告します（該当する Discord、Slack、Google Chat、Microsoft Teams、Mattermost、IRC のスコープ）。

`dangerous`／`dangerously` で始まる設定は、緊急時に使用するオペレーターの明示的なオーバーライドです。いずれかを有効にしただけでは、セキュリティ脆弱性の報告にはなりません。危険なパラメーターの完全な一覧については、[セキュリティ](/ja-JP/gateway/security)の「安全でない、または危険なフラグの概要」を参照してください。

## SecretRef の動作

`security audit` は、対象パスでサポートされている SecretRef を読み取り専用モードで解決します。現在のコマンドパスで SecretRef を使用できない場合、監査はクラッシュせずに続行し、代わりに `secretDiagnostics` を報告します。`--token` と `--password` は、そのコマンド呼び出しに対するディーププローブ認証のみを上書きします。設定や SecretRef のマッピングは書き換えません。

## 抑制

意図的に残している検出結果は、`security.audit.suppressions` で許容します。各抑制は正確な `checkId` と一致し、大文字と小文字を区別しない `titleIncludes` や `detailIncludes` の部分文字列、またはその両方を使用して対象を絞り込めます。

```json
{
  "security": {
    "audit": {
      "suppressions": [
        {
          "checkId": "plugins.tools_reachable_permissive_policy",
          "detailIncludes": "Enabled extension plugins: gbrain",
          "reason": "trusted local operator plugin"
        }
      ]
    }
  }
}
```

抑制された検出結果は、アクティブな `summary` および `findings` のリストから除外されます。JSON 出力では、監査可能性を確保するため `suppressedFindings` に保持されます。抑制が設定されている場合、監査がフィルタリングされたことを読者が判別できるよう、アクティブな出力には抑制不能な `security.audit.suppressions.active` の情報検出結果も保持されます。危険な設定フラグはフラグごとに 1 件の検出結果として出力されるため、危険なフラグを 1 つ許容しても、同じ `config.insecure_or_dangerous_flags` checkId を共有する他の有効なフラグは非表示になりません。

抑制によって継続的なリスクが隠れる可能性があるため、エージェントが実行するシェルコマンドで抑制を追加または削除するには、信頼できるローカル自動化用の `security="full"` と `ask="off"` を使用して exec がすでに実行されている場合を除き、exec の承認が必要です。

## JSON 出力

```bash
openclaw security audit --json | jq '.summary'
openclaw security audit --deep --json | jq '.findings[] | select(.severity=="critical") | .checkId'
```

`--fix --json` を指定すると、出力には修正アクションと最終レポートの両方が含まれます。

```bash
openclaw security audit --fix --json | jq '{fix: .fix.ok, summary: .report.summary}'
```

## `--fix` による変更内容

安全かつ決定的な修復を適用します。

- 一般的な `groupPolicy="open"` を `groupPolicy="allowlist"` に切り替えます（サポート対象チャネルのアカウント別バリアントを含む）
- WhatsApp のグループポリシーを `allowlist` に切り替える際、保存済みの `allowFrom` ファイルが存在し、設定で `allowFrom` がまだ定義されていない場合は、そのファイルから `groupAllowFrom` の初期値を設定します
- `logging.redactSensitive` を `"off"` から `"tools"` に設定します
- 状態／設定ファイルおよび一般的な機密ファイル（`credentials/*.json`、`auth-profiles.json`、`openclaw-agent.sqlite`、旧形式のセッションアーティファクト）の権限を厳格化します
- `openclaw.json` から参照される設定インクルードファイルの権限も厳格化します
- POSIX ホストでは `chmod`、Windows では `icacls` のリセットを使用します

`--fix` は次の操作を**行いません**。

- トークン／パスワード／API キーのローテーション
- ツール（`gateway`、`cron`、`exec` など）の無効化
- Gateway のバインド／認証／ネットワーク公開範囲の選択の変更
- Plugin／Skills の削除または書き換え

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [セキュリティ監査](/ja-JP/gateway/security)
