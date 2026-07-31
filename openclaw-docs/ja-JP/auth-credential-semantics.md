---
read_when:
    - 認証プロファイルの解決または認証情報のルーティングに取り組む場合
    - モデル認証の失敗またはプロファイル順序のデバッグ
summary: 認証プロファイルにおける認証情報の正規の適格性と解決セマンティクス
title: 認証資格情報のセマンティクス
x-i18n:
    generated_at: "2026-07-26T09:26:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b0516b1bb23f400d5ac5fd39a628736034440216ac22823eef061b38564dff0
    source_path: auth-credential-semantics.md
    workflow: 16
---

これらのセマンティクスにより、選択時と実行時の認証動作が一致します。以下で共有されています。

- `resolveAuthProfileOrder`（プロファイルの順序付け）
- `resolveApiKeyForProfile`（実行時の認証情報解決）
- `openclaw models status --probe`
- `openclaw doctor` の認証チェック（`doctor-auth`）

## 安定したプローブ理由コード

プローブ結果には `status` バケット（`ok`、`auth`、`rate_limit`、`billing`、`timeout`、`format`、`unknown`、`no_model`）が含まれ、プローブがモデル呼び出しに到達しなかった場合は、安定した `reasonCode` も含まれます。

| `reasonCode`             | 意味                                                                      |
| ------------------------ | ---------------------------------------------------------------------------- |
| `excluded_by_auth_order` | そのプロバイダーの明示的な認証順序からプロファイルが除外されています。               |
| `missing_credential`     | インライン認証情報も SecretRef も設定されていません。                             |
| `expired`                | トークンの `expires` が過去の日時です。                                              |
| `invalid_expires`        | `expires` が有効な正の Unix ミリ秒タイムスタンプではありません。                         |
| `unresolved_ref`         | 設定された SecretRef を解決できませんでした。                                  |
| `ineligible_profile`     | プロファイルがプロバイダー設定と互換性がありません（不正な形式のキー入力を含みます）。 |
| `no_model`               | 認証情報は存在しますが、プローブ可能なモデル候補を解決できませんでした。                 |

適格性チェックでは、使用可能な認証情報の理由コードとして `ok` が報告されます。

## トークン認証情報

トークン認証情報（`type: "token"`）は、インラインの `token` と `tokenRef` の一方または両方をサポートします。

### 適格性ルール

1. `token` と `tokenRef` の両方がない場合、トークンプロファイルは不適格です（`missing_credential`）。
2. `expires` は省略可能です。指定する場合は、`0` より大きく、JavaScript の `Date` タイムスタンプの最大値（8640000000000000）以下である有限の Unix エポックミリ秒値でなければなりません。
3. `expires` が無効な場合（型が不正、`NaN`、`0`、負数、非有限値、または最大値超過）、プロファイルは `invalid_expires` により不適格となります。
4. `expires` が過去の日時である場合、プロファイルは `expired` により不適格となります。
5. `tokenRef` を指定しても、`expires` の検証は回避されません。

### 解決ルール

1. `expires` に関するリゾルバーのセマンティクスは、適格性のセマンティクスと一致します。
2. 適格なプロファイルでは、トークン情報をインライン値または `tokenRef` から解決できます。
3. 解決できない参照は、`models status --probe` の出力で `unresolved_ref` となります。

## エージェントコピーの移植性

エージェントの認証継承は読み取り透過型です。エージェントにローカルプロファイルがない場合、シークレット情報を自身の認証情報ストアへコピーすることなく、実行時にデフォルト／メインエージェントのストアからプロファイルを解決します（`agents/<agentId>/agent/openclaw-agent.sqlite`）。

`openclaw agents add` などの明示的なコピーフローでは、次の移植性ポリシーを使用します。

- `copyToAgents: false` でない限り、`api_key` および `token` プロファイルは移植可能です。
- リフレッシュトークンは単一使用またはローテーションの影響を受ける可能性があるため、`oauth` プロファイルはデフォルトでは移植できません。
- プロバイダー所有の OAuth フローでは、エージェント間でのリフレッシュ情報のコピーが安全であることが確認されている場合にのみ、`copyToAgents: true` によりオプトインできます。このオプトインは、プロファイルにインラインのアクセス／リフレッシュ情報が含まれている場合にのみ適用されます。

移植できないプロファイルでも、対象エージェントが個別にサインインして独自のローカルプロファイルを作成しない限り、読み取り透過型の継承を通じて引き続き利用できます。

## 設定専用の認証ルート

`mode: "aws-sdk"` を持つ `auth.profiles` エントリはルーティングメタデータであり、保存された認証情報ではありません。対象プロバイダーが、Plugin 所有の Amazon Bedrock セットアップによって書き込まれるルートである `models.providers.<id>.auth: "aws-sdk"` を使用する場合に有効です。認証情報ストアに一致するエントリが存在しない場合でも、これらのプロファイル ID は `auth.order` およびセッションオーバーライドに現れることがあります。

`type: "aws-sdk"` を認証情報ストアに書き込まないでください。保存できる認証情報は、`api_key`、`token`、`oauth` のみです。従来の `auth-profiles.json` にこのようなマーカーがある場合、`openclaw doctor --fix` はそれを `auth.profiles` へ移動し、ストアからマーカーを削除します。

## 明示的な認証順序のフィルタリング

- プロバイダーに対して `auth.order.<provider>` または認証ストアの順序オーバーライドが設定されている場合、`models status --probe` は、そのプロバイダーについて解決された認証順序に残っているプロファイル ID のみをプローブします。保存されたオーバーライドは `auth.order` の設定より優先されます。
- 明示的な順序から除外された、そのプロバイダー用の保存済みプロファイルが、後から暗黙的に試行されることはありません。プローブ出力では、詳細 `Excluded by auth.order for this provider.` とともに `reasonCode: excluded_by_auth_order` として報告されます。

## プローブ対象の解決

- プローブ対象は、認証プロファイル、環境認証情報、または `models.json` から取得できます（結果 `source`: `profile`、`env`、`models.json`）。
- プロバイダーに認証情報があっても、OpenClaw がそのプロバイダーのプローブ可能なモデル候補を解決できない場合、`models status --probe` は `reasonCode: no_model` とともに `status: no_model` を報告します。

## 外部 CLI 認証情報の検出

- 外部 CLI が所有する実行時専用の認証情報（`claude-cli` の Claude CLI、`openai` の Codex CLI、`minimax-portal` の MiniMax CLI）は、プロバイダー、ランタイム、または認証プロファイルが現在の操作の対象範囲に含まれる場合、あるいはその外部ソース用の保存済みローカルプロファイルがすでに存在する場合にのみ検出されます。
- 認証ストアの呼び出し元は、外部 CLI の明示的な検出モードを選択します。永続化済み／Plugin 認証のみを対象とする `none`、保存済みの外部 CLI プロファイルを更新する `existing`、または具体的なプロバイダー／プロファイルのセットを対象とする `scoped` です。
- 読み取り専用／ステータスパスは `allowKeychainPrompt: false` を渡します。これらのパスはファイルベースの外部 CLI 認証情報のみを使用し、macOS Keychain の結果を読み取ったり再利用したりしません。

## OAuth SecretRef ポリシーガード

SecretRef 入力は静的な認証情報専用です。OAuth 認証情報は実行時に変更可能であり（更新フローでローテーション後のトークンが永続化されます）、SecretRef による OAuth 情報を使用すると、変更可能な状態が複数のストアに分断されます。

- プロファイルの認証情報が `type: "oauth"` の場合、そのプロファイル上のすべての認証情報フィールドで SecretRef オブジェクトが拒否されます。
- `auth.profiles.<id>.mode` が `"oauth"` の場合、そのプロファイルへの SecretRef ベースの `keyRef`/`tokenRef` 入力は拒否されます。
- 違反は、起動時／再読み込み時のシークレット準備パスおよびプロファイル解決パスで、ハードエラー（例外のスロー）となります。

## 従来互換のメッセージ

スクリプトとの互換性を維持するため、プローブエラーでは次の先頭行が変更されずに保持されます。

`Auth profile credentials are missing or expired.`

人が理解しやすい詳細と安定した理由コードが、後続行に `↳ Auth reason [code]: ...` の形式で続きます。

## 関連項目

- [シークレット管理](/ja-JP/gateway/secrets)
- [認証ストレージ](/ja-JP/concepts/oauth)
