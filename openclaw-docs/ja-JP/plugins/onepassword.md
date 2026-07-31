---
read_when:
    - エージェントが厳選された 1Password のシークレットを要求できるようにする
    - シークレットごとの承認ポリシーと監査履歴が必要です
    - OpenClaw 用に 1Password サービスアカウントを設定しています
summary: 監査済みのエージェント用シークレットブローカーとして、オプションの1Password Pluginを使用する
title: 1Password シークレットブローカー
x-i18n:
    generated_at: "2026-07-26T10:23:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 255ab4fd2c63754fef29d3ea87dcedc9ca2bd2f34bec1f81139e2ce5b6acdba2
    source_path: plugins/onepassword.md
    workflow: 16
---

# 1Password シークレットブローカー

同梱の `onepassword` Plugin は、厳選された一連の 1Password フィールドを読み取るための、ポリシーで制御された単一のツールをエージェントに提供します。デフォルトでは無効になっており、`plugins.entries.onepassword.config` が存在するまで何も行いません。

これはエージェントツールであり、SecretRef プロバイダーではありません。環境変数を注入したり、OpenClaw 設定のシークレットを解決したりすることはありません。

## セキュリティモデル

- サービスアカウント認証のみ。トークンはローカルの認証情報ファイルに保持され、`openclaw.json` では決して受け付けられません。
- 厳選されたレジストリのみ。エージェントは設定済みのスラッグを一覧表示できますが、Plugin が 1Password の保管庫を列挙することはありません。
- スラッグごとの `auto`、`approve`、または `deny` ポリシー。
- 承認による許可には有効期限があります。キャッシュされた値が現在のポリシーを回避することはありません。
- すべてのアクセス試行は、OpenClaw の共有 SQLite 状態に記録されます。監査行には指定された理由が含まれるため、理由に機密情報を含めないでください。ブローカーが取得した値やサービストークンを監査行にコピーすることはありません。
- 現在のツール実行後、OpenClaw が管理するトランスクリプト永続化処理により、成功した `get` の値は秘匿化されたメタデータに置き換えられます。
- その実行中、値はモデルから参照できます。モデルが後続のツール呼び出しや応答にその値をコピーした場合、その別個の記録はこの Plugin の永続化フックの対象外です。ポリシーは狭く保ち、モデルに値を復唱させないでください。
- Plugin はキャッシュミスごとに `op` を一度呼び出します。レート制限やその他の失敗時に再試行しません。
- 各 `op` 呼び出しは、1Password デスクトップアプリ連携を無効にする最小限の環境（`OP_LOAD_DESKTOP_APP_SETTINGS=false`、`OP_BIOMETRIC_UNLOCK_ENABLED=false`）で実行されるため、Gateway ホストに 1Password アプリがインストールされていても、生体認証や macOS の権限ダイアログは表示されません。

サービスアカウントには、Plugin 設定に登録された保管庫と項目のみへの読み取りアクセス権を付与してください。

## 始める前に

必要なもの：

- Gateway ホストにインストールされた 1Password CLI（`op`）
- 選択した項目へのアクセス権を持つ 1Password サービスアカウント
- 専用のサービスアカウントトークンファイル

同梱の Plugin を有効にします。

```bash
openclaw plugins enable onepassword
```

OpenClaw 状態ディレクトリ配下にトークン用のディレクトリとファイルを作成します。

```bash
mkdir -p ~/.openclaw/credentials/onepassword
chmod 700 ~/.openclaw/credentials/onepassword
printf '%s' "$OP_SERVICE_ACCOUNT_TOKEN" > \
  ~/.openclaw/credentials/onepassword/service-account-token
chmod 600 ~/.openclaw/credentials/onepassword/service-account-token
unset OP_SERVICE_ACCOUNT_TOKEN
```

`OPENCLAW_STATE_DIR` が設定されている場合は、`~/.openclaw` をそのディレクトリに置き換えてください。トークンファイルがグループまたは他のユーザーから読み取り可能または書き込み可能な場合、Plugin は一度だけ警告します。

## 登録済みシークレットの設定

`openclaw.json` に Plugin 設定を追加します。

```jsonc
{
  "plugins": {
    "entries": {
      "onepassword": {
        "enabled": true,
        "config": {
          "vault": "Automation",
          "defaultPolicy": "approve",
          "cacheTtlSeconds": 300,
          "grantTtlHours": 720,
          "opTimeoutMs": 15000,
          "items": {
            "repository-token": {
              "item": "Repository automation token",
              "field": "credential",
              "policy": "approve",
              "description": "Token for repository automation",
            },
            "model-key": {
              "item": "Model provider key",
              "vault": "Agent credentials",
              "policy": "auto",
            },
          },
        },
      },
    },
  },
}
```

スラッグには小文字、数字、ハイフンを使用し、先頭は文字または数字にする必要があります。最大 64 文字です。レジストリには最大 32 個のスラッグを含めることができ、説明は最大 200 文字です。`field` は 1 つのフィールドラベルまたは ID を受け付けます。カンマを含めることはできず、デフォルトは `credential` です。項目レベルの `vault` は、デフォルトの保管庫を上書きします。`opBin` には `op` 実行可能ファイルへの絶対パスを設定できます。設定しない場合、Plugin は `PATH` から `op` を解決します。項目タイトルの先頭をハイフンにすることはできません。

## エージェントツールの使用

ツール名は `onepassword` です。

登録済みのスラッグを一覧表示します。

```json
{ "action": "list" }
```

結果には、スラッグ、説明、ポリシー、常時許可が有効かどうかのみが含まれます。シークレット値は決して含まれず、1Password への問い合わせも行いません。

シークレットを 1 つ要求します。

```json
{
  "action": "get",
  "slug": "repository-token",
  "reason": "Authenticate the requested repository operation"
}
```

`reason` は必須で、空にすることはできず、300 文字以内に制限されます。成功した `get` は、値に加えて、設定されたスラッグ、項目タイトル、フィールドラベルを返します。

ツールスキーマでは、内部用の `authorizationNonce` パラメーターも宣言されています。ポリシー層はリクエストを評価した後、このパラメーターを注入して、実行するツール呼び出しに認可を引き渡します。手動で設定しないでください。指定した値はポリシーフックによって上書きされ、不明な値の場合はリクエストが失敗します。

## ポリシー階層と承認

- `auto`：直ちに取得し、リクエストを監査します。
- `deny`：リクエストをブロックし、監査します。
- `approve`：有効期限内の常時許可を使用するか、人間に今回のみ許可、常に許可、または拒否を求めます。

今回のみ許可すると、現在のツール呼び出しのみが認可されます。常に許可すると、そのエージェントとスラッグに対する常時許可が SQLite に書き込まれます。他のエージェントは個別に承認を受ける必要があります。OpenClaw は、呼び出し元に具体的なエージェント ID がある場合にのみ、常に許可を提示します。許可は `grantTtlHours` 後に失効します。デフォルトは 720 時間です。未解決またはタイムアウトした承認ではリクエストが拒否されます。承認の最大待機時間は 600 秒です。Plugin は最大 1,024 件の常時許可を保持します。この上限に達すると、最も古い許可が削除され、そのエージェントは次回のアクセス時に承認を受ける必要があります。

評価された各認可は 1 回限り使用でき、共有 SQLite 状態を通じて実行するツール呼び出しに引き渡されます。そのため、Gateway プロセス内で複数の Plugin インスタンスが有効な場合でも、この引き渡しは機能します。未使用の認可は 600 秒の承認期間後に失効します。

メモリ内キャッシュのデフォルトは 300 秒で、設定済みのスラッグレジストリによって上限が定められます。無効にするには、`cacheTtlSeconds` を `0` に設定します。ポリシーはキャッシュ検索のたびに事前評価され、キャッシュヒットも監査されます。ランタイム設定の再読み込みは、各ポリシーおよび実行境界で反映されます。Plugin を無効にした場合、またはスラッグを削除、拒否、もしくは別の対象に変更した場合、保留中の認可とキャッシュされた値は無効になります。

## 状態と監査履歴の確認

準備状態とレジストリ件数を表示します。

```bash
openclaw onepassword status
```

これは、トークンファイルが存在するか、`op` が解決されたかとそのパス、登録済み項目数、およびポリシー別の件数を報告します。トークンやシークレット値を読み取ったり表示したりすることはありません。

最新の 50 件の監査行を表示します。

```bash
openclaw onepassword audit
openclaw onepassword audit --limit 100
```

行は新しい順に並び、タイムスタンプ、エージェント、スラッグ、結果、試行が失敗した場合の `errorCode`、および切り詰められた理由が表示されます。理由は指定されたまま保存されます。ブローカーが取得した値を監査ログに追加することはありません。

## 1Password CLI の動作

キャッシュミスごとに、設定済みの項目、保管庫、完全一致のフィールドセレクター、JSON 出力、制限付きタイムアウト、および `--cache=false` を指定して `op item get` を実行します。子プロセスは項目全体ではなく、そのフィールドのみを受け取ります。子プロセス環境には `OP_SERVICE_ACCOUNT_TOKEN` と `HOME` のみが存在します。

Plugin が試行するのは 1 回だけです。`RATE_LIMITED` エラーが発生した場合は、時間を置いてから後続のエージェントリクエストを行って対処してください。Plugin は自動再試行ループを作成しません。

## エラーコード

失敗した試行では、ツール結果と監査行に限定的なエラーコードが 1 つ記録されます。

1Password アクセスエラー：

| コード              | 意味                                                          |
| ----------------- | ---------------------------------------------------------------- |
| `TOKEN_MISSING`   | トークンファイルが存在しないか、空です                                   |
| `OP_NOT_FOUND`    | `op` バイナリを解決できませんでした                                |
| `ITEM_NOT_FOUND`  | 設定済みの項目が保管庫にありません                              |
| `FIELD_NOT_FOUND` | 設定済みのフィールドが項目にありません。使用可能なラベルが一覧表示されます |
| `RATE_LIMITED`    | 1Password サービスアカウントのレート制限に達しました                     |
| `AUTH_FAILED`     | サービスアカウント認証に失敗しました                            |
| `TIMEOUT`         | `op` が `opTimeoutMs` を超過しました                                      |
| `OP_ERROR`        | その他の `op` の失敗または無効な出力                         |

ポリシーおよび検証エラー：

| コード                                               | 意味                                                                      |
| -------------------------------------------------- | ---------------------------------------------------------------------------- |
| `INVALID_ACTION`、`INVALID_REASON`、`INVALID_SLUG` | リクエストの入力検証に失敗しました                                              |
| `UNKNOWN_SLUG`                                     | スラッグが設定済みのレジストリにありません                                       |
| `TOOL_CALL_ID_MISSING`                             | ツール呼び出し ID なしで呼び出しが到着しました                                          |
| `POLICY_NOT_EVALUATED`                             | この呼び出しに一致する認可がありません。リクエストはポリシーによって承認されていません |
| `POLICY_CHANGED`                                   | 承認から実行までの間に設定が変更されました                                |
| `GRANT_EXPIRED`                                    | 実行前に常時許可が失効しました                                       |
| `APPROVAL_CANCELLED`                               | 承認の保留中に実行が中止されました                           |
