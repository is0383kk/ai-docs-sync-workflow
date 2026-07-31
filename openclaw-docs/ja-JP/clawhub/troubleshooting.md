---
read_when:
    - ClawHub CLI または OpenClaw レジストリコマンドが失敗する
    - パッケージをインストール、公開、または更新できない
summary: ClawHub のサインイン、インストール、公開、更新、API に関する問題のトラブルシューティング。
x-i18n:
    generated_at: "2026-07-26T09:34:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fc789fcc891cf8c44b5d1a10d38a4e6dd4dec9474d8d13f8058ea1c3392a9f91
    source_path: clawhub/troubleshooting.md
    workflow: 16
---

# トラブルシューティング

## `clawhub login` でブラウザーが開くが完了しない

CLI はブラウザーでのログイン中に、短時間のみ動作するローカルコールバックサーバーを起動します。

- ブラウザーから `http://127.0.0.1:<port>/callback` にアクセスできることを確認してください。
- コールバックが届かない場合は、ローカルファイアウォール、VPN、プロキシのルールを確認してください。
- ヘッドレス環境では、ClawHub の Web UI で API トークンを作成し、次を実行します。

```bash
clawhub login --token clh_...
```

## `whoami` または `publish` が `Unauthorized` (401) を返す

- `clawhub login` で再度サインインしてください。
- カスタム設定パスを使用している場合は、`CLAWHUB_CONFIG_PATH` が現在のトークンを含む
  ファイルを指していることを確認してください。
- API トークンを使用している場合は、Web UI で取り消されていないことを確認してください。

## 検索またはインストールが `Rate limit exceeded` (429) を返す

レスポンス内の再試行情報を確認してください。

- `Retry-After`: 再試行までの待機秒数。
- `RateLimit-Limit`: このリクエストに適用された上限。
- `RateLimit-Remaining`: ヘッダーが存在する場合の正確な残り割り当て量。`429` では `0` です。
- `RateLimit-Reset` または `X-RateLimit-Reset`: リセットのタイミング。

多数のユーザーが 1 つの送信元 IP を共有している場合、各ユーザーが数件のリクエストしか
送信していなくても、匿名 IP の上限に達することがあります。可能な場合はサインインし、
通知された待機時間の経過後に再試行してください。

## プロキシ環境で検索またはインストールに失敗する

CLI は標準のプロキシ変数に対応しています。

```bash
export HTTPS_PROXY=http://proxy.example.com:3128
clawhub search "my query"
```

サポートされる名前には、`HTTPS_PROXY`、`HTTP_PROXY`、`https_proxy`、
`http_proxy` があります。

## 検索に Skill が表示されない

- 正確なスラッグまたは所有者ページがわかっている場合は、それを確認してください。
- リリースが公開されており、スキャンまたはモデレーションによって保留されていないことを確認してください。
- Skill の所有者である場合は、サインインして調査してください。

```bash
clawhub inspect @openclaw/demo
```

所有者にのみ表示される診断情報から、スキャン、アップロードゲート、またはモデレーションの状態を確認できる場合があります。

## 必須メタデータがないため公開に失敗する

Skills の場合は、`SKILL.md` フロントマターを確認してください。ユーザーとスキャナーがパッケージを理解できるように、必須の環境変数と
ツールを宣言する必要があります。

plugins の場合は、`package.json` の互換性メタデータを確認してください。コード Plugin の公開には、
`openclaw.compat.pluginApi` や `openclaw.build.openclawVersion` などの OpenClaw 互換性フィールドが
必要です。

最初に公開ペイロードをプレビューします。

```bash
clawhub package publish <source> --family code-plugin --dry-run
```

## GitHub の所有者またはソースに関するエラーで公開に失敗する

ClawHub は GitHub の ID とソース帰属情報を使用して、パッケージとその
公開者を関連付けます。

- パッケージを所有しているか、公開権限を持つ GitHub アカウントでサインインしていることを確認してください。
- ソース URL が公開されているか、ClawHub からアクセス可能であることを確認してください。
- GitHub ソースには、`owner/repo`、`owner/repo@ref`、または完全な GitHub URL を使用してください。

## 名前空間が取得済みまたは予約済みのため公開に失敗する

所有者ハンドル、組織の名前空間、パッケージスコープ、Skill の
スラッグ、またはパッケージ名がすでに取得済みか予約済みであるため公開に失敗した場合は、まず
名前空間と一致する所有者として公開していることを確認してください。Plugin パッケージでは、
`@example-org/example-plugin` のようなスコープ付き名前を、対応する
`example-org` 所有者として公開する必要があります。

自分の組織、プロジェクト、またはブランドが正当な名前空間所有者であるにもかかわらず、
現在の ClawHub 所有者を管理できない場合は、公開可能な機密性のない証拠を添えて
[組織／名前空間の申請 Issue](https://github.com/openclaw/clawhub/issues/new?template=org-namespace-claim.yml)
を作成してください。証拠に関するガイダンスと、公開 Issue に
含めるべきでない情報については、[組織と名前空間の申請](/clawhub/namespace-claims)を参照してください。

## `sync` に Skill が見つからなかったと表示される

`sync` は、`SKILL.md` または `skill.md` を含むフォルダーを検索します。

スキャンするルートを指定してください。

```bash
clawhub sync --root /path/to/skills
```

公開される内容が不明な場合は、最初にプレビューしてください。

```bash
clawhub sync --all --dry-run --no-input
```

## ローカルの変更が原因で `update` が拒否する

ローカルファイルが、ClawHub が認識しているどのバージョンとも一致しません。次のいずれかを選択してください。

- ローカルの編集内容を保持し、更新をスキップします。
- 公開済みバージョンで上書きします。

```bash
clawhub update @openclaw/demo --force
```

- 編集したコピーを新しいスラッグまたはフォークとして公開します。

## OpenClaw で Plugin のインストールに失敗する

- ClawHub ソースを明示的に指定します。

```bash
openclaw plugins install clawhub:<package>
```

- パッケージの詳細ページで、スキャン状態と互換性メタデータを確認してください。
- 使用中の OpenClaw バージョンが、パッケージで提示されている
  互換性範囲を満たしていることを確認してください。
- パッケージが非表示、保留中、またはブロックされている場合、所有者が問題を解決するまで
  インストールできない可能性があります。

## 公開 API リクエストに失敗する

- `429` の再試行ヘッダーに従い、公開リスト／検索レスポンスをキャッシュしてください。
- ユーザーを ClawHub の正規リストページに誘導してください。
- 非表示、非公開、保留中、またはモデレーションによってブロックされたコンテンツを、公開 API の範囲外に
  ミラーリングしないでください。

エンドポイントの詳細については、[HTTP API](/clawhub/http-api)を参照してください。
