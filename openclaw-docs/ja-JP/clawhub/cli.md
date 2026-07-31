---
read_when:
    - ClawHub CLI の使用方法
    - インストール、更新、公開のデバッグ
summary: CLI リファレンス：コマンド、フラグ、設定、ロックファイルの動作。
x-i18n:
    generated_at: "2026-07-26T09:32:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eba91a83c5542c4b570bd22a526911633e43d0b4e921c013e6fd29451193f2a7
    source_path: clawhub/cli.md
    workflow: 16
---

# CLI

CLI パッケージ: `clawhub`、バイナリ: `clawhub`。

npm または pnpm を使用してグローバルにインストールします。

```bash
npm i -g clawhub
# または
pnpm add -g clawhub
```

次に、動作を確認します。

```bash
clawhub --help
clawhub login
clawhub whoami
```

## グローバルフラグ

- `--workdir <dir>`: 作業ディレクトリ（デフォルト: cwd。設定されている場合は Clawdbot ワークスペースにフォールバック）
- `--dir <dir>`: workdir 配下のインストールディレクトリ（デフォルト: `skills`）
- `--site <url>`: ブラウザログイン用のベース URL（デフォルト: `https://clawhub.ai`）
- `--registry <url>`: API ベース URL（デフォルト: 検出された URL。検出できない場合は `https://clawhub.ai`）
- `--no-input`: プロンプトを無効化

対応する環境変数:

- `CLAWHUB_SITE`（レガシー: `CLAWDHUB_SITE`）
- `CLAWHUB_REGISTRY`（レガシー: `CLAWDHUB_REGISTRY`）
- `CLAWHUB_WORKDIR`（レガシー: `CLAWDHUB_WORKDIR`）

### HTTP プロキシ

CLI は、企業プロキシの背後や制限されたネットワーク上のシステムで、
標準の HTTP プロキシ環境変数を使用します。

- `HTTPS_PROXY` / `https_proxy`
- `HTTP_PROXY` / `http_proxy`
- `NO_PROXY` / `no_proxy`

これらの変数のいずれかが設定されている場合、CLI は送信リクエストを
指定されたプロキシ経由でルーティングします。HTTPS リクエストには `HTTPS_PROXY`、通常の
HTTP には `HTTP_PROXY` が使用されます。特定のホストまたはドメインでプロキシを
バイパスするために、`NO_PROXY` / `no_proxy` が使用されます。

これは、直接の送信接続がブロックされているシステム
（例: Docker コンテナ、プロキシ経由でのみインターネットに接続できる Hetzner VPS、企業
ファイアウォール）で必要です。

例:

```bash
export HTTPS_PROXY=http://proxy.example.com:3128
export NO_PROXY=localhost,127.0.0.1
clawhub search "my query"
```

プロキシ変数が設定されていない場合、動作は変更されません（直接接続）。

## 設定ファイル

API トークンとキャッシュされたレジストリ URL を保存します。

- macOS: `~/Library/Application Support/clawhub/config.json`
- Linux/XDG: `$XDG_CONFIG_HOME/clawhub/config.json` または `~/.config/clawhub/config.json`
- Windows: `%APPDATA%\\clawhub\\config.json`
- レガシーフォールバック: `clawhub/config.json` がまだ存在せず、`clawdhub/config.json` が存在する場合、CLI はレガシーパスを再利用します
- オーバーライド: `CLAWHUB_CONFIG_PATH`（レガシー: `CLAWDHUB_CONFIG_PATH`）

## コマンド

### `login` / `auth login`

- デフォルト: ブラウザで `<site>/cli/auth` を開き、ループバックコールバック経由で完了します。
- ヘッドレス: `clawhub login --token clh_...`
- リモート／ヘッドレス対話モード: `clawhub login --device` はコードを表示し、`<site>/cli/device` で認可が完了するまで待機します。

### `whoami`

- `/api/v1/whoami` を使用して、保存されたトークンを検証します。

### `token`

- 保存された API トークンを標準出力に表示します。
- ローカルのログイントークンを CI シークレット設定コマンドにパイプする場合に便利です。

### `star <skill>` / `unstar <skill>`

- Skills をブックマークに追加、またはブックマークから削除します。互換性のため、コマンド名は引き続き `star` および
  `unstar` です。
- `POST /api/v1/stars/<slug>` および `DELETE /api/v1/stars/<slug>` を呼び出します。
- `--yes` は確認を省略します。

### `search <query...>`

- `/api/v1/search?q=...` を呼び出します。
- 出力には Skills のスラッグ、所有者ハンドル、表示名、関連度スコアが含まれます。
- 検索では、ダウンロード人気度よりも、スラッグ／名前のトークンとの完全一致が優先されます。`map` のような独立したスラッグトークンは、`amap` 内の部分文字列よりも `personal-map` に強く一致します。
- 人気度はランキングにおける小さな事前要素であり、最上位への掲載を保証するものではありません。
- 表示されるべき Skills が表示されない場合、メタデータの名前を変更する前に、ログインした状態で `clawhub inspect @owner/slug` を実行し、所有者に表示されるモデレーション診断を確認してください。

### `explore`

- `/api/v1/skills?limit=...&sort=createdAt` を使用して最新の Skills を一覧表示します（`createdAt` の降順でソート）。
- フラグ:
  - `--limit <n>`（1～200、デフォルト: 25）
  - `--sort newest|updated|rating|downloads|trending`（デフォルト: 最新順）。互換性のため、レガシーなインストール順のソートエイリアスも引き続き機能します。
  - `--json`（機械可読出力）
- 出力: `<slug>  v<version>  <age>  <summary>`（概要は 50 文字に切り詰められます）。

### `inspect @owner/slug`

- インストールせずに Skills のメタデータとバージョンファイルを取得します。
- `--version <version>`: 特定のバージョンを調査します（デフォルト: 最新）。
- `--tag <tag>`: タグ付きバージョンを調査します（例: `latest`）。
- `--versions`: バージョン履歴を一覧表示します（最初のページ）。
- `--limit <n>`: 一覧表示するバージョンの最大数（1～200）。
- `--files`: 選択したバージョンのファイルを一覧表示します。
- `--file <path>`: 生のファイルバイトを取得します（上限 10MB）。
- `--json`: 機械可読出力。`--file` には、正確なバイト列が base64 として含まれ、利用可能な場合は UTF-8 テキストも含まれます。

### `install @owner/slug`

- 指定された所有者と Skills の最新バージョンを解決します。
- `/api/v1/download` 経由で zip をダウンロードします。
- `<workdir>/<dir>/<slug>` に展開します。
- 固定された Skills の上書きを拒否します。先に `clawhub unpin <skill>` を実行してください。
- 次を書き込みます:
  - `<workdir>/.clawhub/lock.json`（レガシー: `.clawdhub`）
  - `<skill>/.clawhub/origin.json`（レガシー: `.clawdhub`）

### `uninstall <skill>`

- `<workdir>/<dir>/<slug>` を削除し、ロックファイルのエントリを削除します。
- ログイン中はベストエフォート方式でテレメトリを送信し、現在のインストール数を
  無効化できるようにします。
- 対話モード: 確認を求めます。
- 非対話モード（`--no-input`）: `--yes` が必要です。

### `list`

- `<workdir>/.clawhub/lock.json`（レガシー: `.clawdhub`）を読み取ります。
- `clawhub pin` で固定された Skills の横に `pinned` を表示し、任意の理由も併記します。

### `pin <skill>`

- インストール済みの Skills をロックファイルで固定済みとしてマークします。
- `--reason <text>` は Skills を固定した理由を記録します。
- 固定された Skills は `update --all` でスキップされ、`update <skill>` を直接実行した場合は拒否されます。
- 固定された Skills では `install --force` も拒否されるため、ローカルのバイト列が誤って置き換えられることはありません。

### `unpin <skill>`

- インストール済みの Skills からロックファイルの固定を削除し、以降の更新で変更できるようにします。

### `update [@owner/slug]` / `update --all`

- ローカルファイルからフィンガープリントを計算します。
- フィンガープリントが既知のバージョンと一致する場合: プロンプトは表示されません。
- フィンガープリントが一致しない場合:
  - デフォルトでは拒否します
  - `--force` で上書きします（対話モードの場合はプロンプトを表示）
- 固定された Skills は `--force` では更新されません。
- `update <skill>` は固定された Skills に対して即座に失敗し、先に `clawhub unpin <skill>` を実行するよう通知します。
- `update --all` は固定されたスラッグをスキップし、固定されたままの項目の概要を表示します。

### `skill publish <path>`

- ローカルバンドルのフィンガープリントを ClawHub と比較し、コンテンツがすでに
  公開されている場合は正常終了します。
- 新しい Skills のデフォルトは `1.0.0` です。変更された Skills のデフォルトは次のパッチ
  バージョンです。
- `--version <version>` はバージョンを明示的に選択し、コンテンツが
  既存のバージョンと一致する場合でも公開します。
- `--dry-run` はアップロードせずに公開内容を解決します。`--json` は
  機械可読の結果を表示します。
- 実行者に公開権限がある場合、`--owner <handle>` は組織／ユーザーの公開者ハンドルで
  公開します。
- `--migrate-owner` は新しいバージョンを公開すると同時に、既存の Skills を `--owner` に
  移動します。両方の公開者に対する管理者／所有者アクセス権が必要です。
- 所有者とレビューの動作については `docs/publishing.md` で説明しています。
- Skills を公開すると、ClawHub 上で `MIT-0` の下にリリースされます。
- 公開された Skills は、帰属表示なしで自由に使用、変更、再配布できます。
- ClawHub は有料の Skills や Skills ごとの価格設定をサポートしていません。
- レガシーエイリアス: `publish <path>`。

```bash
clawhub skill publish ./my-skill --dry-run
clawhub skill publish ./my-skill
clawhub skill publish ./my-skill --version 2.0.0
```

#### GitHub Actions

ClawHub の再利用可能な
[`skill-publish.yml`](https://github.com/openclaw/clawhub/blob/main/.github/workflows/skill-publish.yml)
ワークフローは、1 つの `skill_path`、または `root`（デフォルト: `skills`）直下の各 Skills
フォルダーに対して `skill publish` を呼び出します。変更されていない Skills はスキップされ、
同じ自動パッチバージョン動作が使用されます。

トークンなしでプレビューするには `dry_run: true` を設定します。実際の公開には
`clawhub_token` シークレットが必要です。

### `sync`

- 現在の workdir、設定された Skills ディレクトリ、および `SKILL.md` または
  `skill.md` を含むローカル Skills フォルダーを対象として、すべての `--root <dir>`
  フォルダーをスキャンします。
- 各ローカル Skills のフィンガープリントを ClawHub と比較し、新規または
  変更された Skills のみを公開します。
- 新しい Skills は `1.0.0` として公開され、変更された Skills はデフォルトで次のパッチバージョンとして
  公開されます。より大きな semver ステップに進める必要がある更新バッチには、
  `--bump minor|major` を使用します。
- `--dry-run` はアップロードせずに公開計画を表示します。`--json` は
  機械可読の計画を表示します。
- `--all` は、新規または変更されたすべての Skills を確認なしで公開します。
  `--all` がない場合、対話型ターミナルでは公開する Skills を選択できます。
- 実行者に公開権限がある場合、`--owner <handle>` は組織／ユーザーの公開者ハンドルで
  公開します。
- `sync` は一方向の公開のみを行います。インストール、更新、ダウンロード、
  インストール／ダウンロードのテレメトリ報告は行いません。

```bash
clawhub sync --all --dry-run
clawhub sync --all
clawhub sync --root ./skills --owner openclaw --bump minor
```

### `scan --slug <slug>`

- `clawhub login` が必要です。
- `POST /api/v1/skills/-/scan` を介して ClawHub ClawScan を実行し、スキャンが終了状態になるまでポーリングします。
- スキャンは非同期であり、完了までに時間がかかる場合があります。キューに入っている間、ターミナルのスピナーには現在の優先スキャン位置と、先に待機しているスキャン数が表示されます。
- 公開済みのスキャンには、所有権または公開者管理アクセス権が必要です。モデレーター／管理者は、`clawhub-admin` を介して同じバックエンドを使用できます。
- `--update` は `--slug` と併用する場合にのみ有効です。成功した公開済みスキャン結果を、選択したバージョンに書き戻します。
- `--output <file.zip>` は、`manifest.json`、`clawscan.json`、`skillspector.json`、`static-analysis.json`、`virustotal.json`、および `README.md` を含む完全なレポートアーカイブをダウンロードします。
- `--json` は、自動化用に完全なポーリング応答を表示します。
- ローカルパスのスキャンはサポートされなくなりました。新しいバージョンをアップロードしてから、`scan download` を使用して、送信したバージョンに保存されたスキャン結果を取得してください。

```bash
clawhub scan --slug gifgrep
clawhub scan --slug gifgrep --version 1.2.3
clawhub scan --slug gifgrep --update --output report.zip
```

### `scan download <name>`

- `clawhub login` が必要です。
- 送信された Skill または Plugin のバージョンについて、保存済みのスキャンレポート ZIP をダウンロードします。ClawHub のセキュリティチェックによってブロックまたは非表示にされたバージョンも対象です。
- Skill のダウンロードでは Skill のスラッグを使用し、デフォルトは `--kind skill` です。
- Plugin のダウンロードではパッケージ名を使用し、`--kind plugin` が必要です。
- 作成者が ClawHub によってブロックされた送信済みの正確なバージョンを検査できるようにするため、`--version` が必要です。
- `--output <file.zip>` で出力先パスを選択します。

```bash
clawhub scan download gifgrep --version 1.2.3
clawhub scan download @scope/demo --version 2.0.0 --kind plugin --output report.zip
```

#### GitHub Actions

ClawHub は、Skill リポジトリおよびカタログリポジトリ向けに、公式の再利用可能なワークフローを
[`/.github/workflows/skill-publish.yml`](https://github.com/openclaw/clawhub/blob/62a697ef1e1b623afd71cf8813b545487a17354f/.github/workflows/skill-publish.yml)
で提供しています。

一般的なカタログ設定：

```yaml
name: Skill Publish

on:
  pull_request:
  workflow_dispatch:

jobs:
  dry-run:
    if: github.event_name == 'pull_request'
    uses: openclaw/clawhub/.github/workflows/skill-publish.yml@v1
    with:
      owner: nvidia
      dry_run: true

  publish:
    if: github.event_name == 'workflow_dispatch'
    uses: openclaw/clawhub/.github/workflows/skill-publish.yml@v1
    with:
      owner: nvidia
      dry_run: false
    secrets:
      clawhub_token: ${{ secrets.CLAWHUB_TOKEN }}
```

注記：

- カタログリポジトリでは、`root` のデフォルトは `skills` です。
- 1 つの Skill フォルダーを処理するには、`skill_path: skills/review-helper` を渡します。
- `owner` は CLI の `--owner` フラグに対応します。認証済みユーザーとして公開する場合は省略します。
- V1 の Skill 公開では `clawhub_token` を使用します。GitHub OIDC の信頼された公開は、現時点ではパッケージのみに対応しています。

### `delete <skill>`

- `--version` を指定しない場合、Skill を論理削除します（所有者、モデレーター、または管理者）。
- `DELETE /api/v1/skills/{slug}` を呼び出します。
- 所有者が開始した論理削除では、スラッグが 30 日間予約されます。コマンドは有効期限を出力します。
- `--version <version>` は、所有する最新ではない 1 つのバージョンを、フェイルクローズ方式の
  バージョン固有ルートを通じて取り下げます。バージョン番号は予約されたままとなり、異なる内容で
  再公開することはできません。現在の最新バージョンを削除する前に、代替バージョンを公開してください。
  このバージョン専用フローでは、プラットフォームスタッフも所有権を迂回できません。
- `--reason <text>` は、Skill 全体の論理削除と監査ログにモデレーション注記を記録します。
- `--note <text>` は `--reason` のエイリアスです。
- `--yes` は確認を省略します。

### `undelete <skill>`

- 非表示の Skill を復元します（所有者、モデレーター、または管理者）。
- `POST /api/v1/skills/{slug}/undelete` を呼び出します。
- `--version <version>` は、同じ所有者アクターが以前に取り下げた、保持されている正確なアーティファクトのみを
  復元します。復元したバージョンを最新にしたり、削除されたタグを再作成したりすることはありません。
- バージョンの復元では `POST /api/v1/skills/{slug}/versions/{version}/restore` を呼び出します。
- `--reason <text>` は、Skill と監査ログにモデレーション注記を記録します。
- `--note <text>` は `--reason` のエイリアスです。
- `--yes` は確認を省略します。

### `hide <skill>`

- Skill を非表示にします（所有者、モデレーター、または管理者）。
- `delete` のエイリアスです。

### `unhide <skill>`

- Skill の非表示を解除します（所有者、モデレーター、または管理者）。
- `undelete` のエイリアスです。

### `skill rename <skill> <new-name>`

- 所有する Skill の名前を変更し、以前のスラッグをリダイレクトエイリアスとして保持します。
- `POST /api/v1/skills/{slug}/rename` を呼び出します。
- `--yes` は確認を省略します。

### `skill merge <source> <target>`

- 所有する 1 つの Skill を、所有する別の Skill にマージします。
- 移行元のスラッグは公開一覧への表示を停止し、移行先へのリダイレクトエイリアスになります。
- `POST /api/v1/skills/{sourceSlug}/merge` を呼び出します。
- `--yes` は確認を省略します。

### `transfer`

- 所有権移管ワークフロー。
- ユーザーハンドルへの移管では、受取人が承認する保留中のリクエストが作成されます。
- 組織／パブリッシャーのハンドルへの移管は、アクターが現在の所有者と移管先パブリッシャーの
  両方に対する管理者アクセス権を持つ場合にのみ、即時適用されます。
- サブコマンド：
  - `transfer request <skill> <handle> [--message "..."] [--yes]`
  - `transfer list [--outgoing]`
  - `transfer accept <skill> [--yes]`
  - `transfer reject <skill> [--yes]`
  - `transfer cancel <skill> [--yes]`
- エンドポイント：
  - `POST /api/v1/skills/{slug}/transfer`
  - `POST /api/v1/skills/{slug}/transfer/accept`
  - `POST /api/v1/skills/{slug}/transfer/reject`
  - `POST /api/v1/skills/{slug}/transfer/cancel`
  - `GET /api/v1/transfers/incoming`
  - `GET /api/v1/transfers/outgoing`

### `package explore [query...]`

- `GET /api/v1/packages` および `GET /api/v1/packages/search` を介して、統合パッケージカタログを閲覧または検索します。
- Plugin やその他のパッケージファミリーのエントリにはこれを使用します。トップレベルの `search` は引き続き Skill の検索インターフェースです。
- フラグ：
  - `--family skill|code-plugin|bundle-plugin`
  - `--official`
  - `--executes-code`
  - `--target <target>`、`--os <os>`、`--arch <arch>`、`--libc <libc>`
  - `--requires-browser`、`--requires-desktop`、`--requires-native-deps`
  - `--requires-external-service`、`--external-service <name>`
  - `--binary <name>`、`--os-permission <name>`
  - `--artifact-kind legacy-zip|npm-pack`
  - `--npm-mirror`
  - `--limit <n>`（1-100、デフォルト：25）
  - `--json`

例：

```bash
clawhub package explore --family code-plugin
clawhub package explore --family code-plugin --os darwin --requires-desktop
clawhub package explore --family code-plugin --artifact-kind npm-pack
clawhub package explore --npm-mirror
clawhub package explore episodic-claw --family code-plugin
```

### `package inspect <name>`

- インストールせずにパッケージのメタデータを取得します。
- Plugin のメタデータ、互換性、検証、ソース、およびバージョン／ファイルの検査に使用します。
- `--version <version>`：特定のバージョンを検査します（デフォルト：最新）。
- `--tag <tag>`：タグ付きバージョンを検査します（例：`latest`）。
- `--versions`：バージョン履歴を一覧表示します（最初のページ）。
- `--limit <n>`：一覧表示するバージョンの最大数（1-100）。
- `--files`：選択したバージョンのファイルを一覧表示します。
- `--file <path>`：サイズ制限付きの UTF-8 テキストプレビューを取得します（上限 200KB）。
- `--json`：機械可読出力。

### `package download <name>`

- `GET /api/v1/packages/{name}/versions/{version}/artifact` を介して
  パッケージバージョンを解決します。
- リゾルバーの `downloadUrl` からアーティファクトをダウンロードします。
- すべてのアーティファクトについて ClawHub SHA-256 を検証します。
- ClawPack の npm-pack アーティファクトでは、npm の `sha512` 整合性、
  npm shasum、および tarball の `package.json` の名前／バージョンも検証します。
- 従来の ZIP バージョンは、従来の ZIP ルートを介してダウンロードします。
- フラグ：
  - `--version <version>`：特定のバージョンをダウンロードします。
  - `--tag <tag>`：タグ付きバージョンをダウンロードします（デフォルト：`latest`）。
  - `-o, --output <path>`：出力ファイルまたはディレクトリ。
  - `--force`：既存の出力ファイルを上書きします。
  - `--json`：機械可読出力。

例：

```bash
clawhub package download @openclaw/example-plugin --tag latest
clawhub package download @openclaw/example-plugin --version 1.2.3 -o artifacts/
```

### `package verify <file>`

- ローカルアーティファクトについて、ClawHub SHA-256、npm の `sha512` 整合性、および npm shasum を
  計算します。
- `--package` を指定すると、ClawHub から期待されるメタデータを解決し、
  ローカルファイルを公開済みアーティファクトのメタデータと比較します。
- ダイジェストフラグを直接指定すると、ネットワーク検索を行わずに検証します。
- フラグ：
  - `--package <name>`：期待されるアーティファクトのメタデータを解決するパッケージ名。
  - `--version <version>` または `--tag <tag>`：期待されるパッケージバージョン。
  - `--sha256 <hex>`：期待される ClawHub SHA-256。
  - `--npm-integrity <sri>`：期待される npm 整合性。
  - `--npm-shasum <sha1>`：期待される npm shasum。
  - `--json`：機械可読出力。

例：

```bash
clawhub package verify ./example-plugin-1.2.3.tgz --package @openclaw/example-plugin --version 1.2.3
clawhub package verify ./example-plugin-1.2.3.tgz --sha256 <hex>
```

### `package validate <source>`

- ローカルの Plugin パッケージフォルダーに対して、ClawHub CLI に同梱された Plugin Inspector を
  実行します。
- ローカルの OpenClaw チェックアウトを検索またはインポートせず、デフォルトではオフライン／静的検証を
  行います。
- 重大な互換性エラーがある場合はゼロ以外で終了します。警告のみの検出結果は出力されますが、
  終了コードはゼロです。
- フラグ：
  - `--out <dir>`：Plugin Inspector のレポートをこのディレクトリに書き込みます。
  - `--openclaw <path>`：明示的に指定したローカルの OpenClaw チェックアウトに対して検査します。
  - `--runtime`：ランタイムキャプチャを有効にします。Plugin コードをインポートします。
  - `--allow-execute`：隔離されたワークスペースでランタイムキャプチャを許可します。
  - `--no-mock-sdk`：ランタイムキャプチャ中にモック化された OpenClaw SDK を無効にします。
  - `--json`：機械可読出力。

例：

```bash
clawhub package validate ./example-plugin
```

検証でパッケージ、マニフェスト、SDK インポート、またはアーティファクトに関する問題が報告された場合は、
[Plugin 検証の修正方法](/ja-JP/clawhub/plugin-validation-fixes)を参照してから、コマンドを再実行してください。

### `package delete <name>`

- `--version` を指定しない場合、パッケージとすべてのリリースを論理削除します。
- `--version <version>` は、所有する最新ではない 1 つのリリースを、フェイルクローズ方式の
  バージョン固有ルートを通じて取り下げます。バージョン番号は予約されたままとなり、異なる内容で
  再公開することはできません。現在の最新バージョンを削除する前に、代替バージョンを公開してください。
  このバージョン専用フローには、パッケージ所有者または組織パブリッシャーの管理者権限が必要です。
  プラットフォームスタッフもパッケージ所有権を迂回できません。
- パッケージ全体の論理削除には、パッケージ所有者、組織パブリッシャーの所有者／管理者、プラットフォームの
  モデレーター、またはプラットフォーム管理者であることが必要です。
- フラグ：
  - `--version <version>`：最新ではない 1 つのバージョンを取り下げます。
  - `--yes`：確認を省略します。
  - `--json`：機械可読出力。

例：

```bash
clawhub package delete @openclaw/example-plugin --yes
clawhub package delete @openclaw/example-plugin --version 1.2.3 --yes
```

### `package undelete <name>`

- 論理削除されたパッケージとリリースを復元します。
- パッケージ所有者、組織パブリッシャーの所有者／管理者、プラットフォームのモデレーター、
  またはプラットフォーム管理者であることが必要です。
- `POST /api/v1/packages/{name}/undelete` を呼び出します。
- `--version <version>` は、同じ所有者アクターが以前に取り下げた、保持されている正確なリリースのみを
  復元します。リリースを最新にしたり、削除されたパッケージタグ／dist-tag を再作成したりすることはありません。
- バージョンの復元では `POST /api/v1/packages/{name}/versions/{version}/restore` を呼び出します。
- フラグ：
  - `--version <version>`：所有者が取り下げた 1 つのリリースを復元します。
  - `--yes`：確認を省略します。
  - `--json`：機械可読出力。

例：

```bash
clawhub package undelete @openclaw/example-plugin --yes
```

### `package transfer <name>`

- パッケージを別のパブリッシャーに移管します。
- プラットフォーム管理者が実行する場合を除き、現在のパッケージ所有者と移管先
  パブリッシャーの両方に対する管理者アクセスが必要です。
- スコープ付きパッケージ名は、対応するスコープ所有者に移管する必要があります。
- `POST /api/v1/packages/{name}/transfer`を呼び出します。
- フラグ:
  - `--to <owner>`: 移管先パブリッシャーのハンドル。
  - `--reason <text>`: 任意の監査理由。
  - `--json`: 機械可読出力。

例:

```bash
clawhub package transfer @openclaw/example-plugin --to openclaw
```

### `package report`

- パッケージをモデレーターに報告するための認証済みコマンドです。
- `POST /api/v1/packages/{name}/report`を呼び出します。
- 報告はパッケージ単位で、任意でバージョンに関連付けられ、モデレーターが
  レビューできるようになります。
- 報告だけでパッケージが自動的に非表示になったり、ダウンロードがブロックされたりすることはありません。
- フラグ:
  - `--version <version>`: 報告に添付する任意のパッケージバージョン。
  - `--reason <text>`: 必須の報告理由。
  - `--json`: 機械可読出力。

例:

```bash
clawhub package report @openclaw/example-plugin --version 1.2.3 --reason "suspicious native payload"
```

### `package moderation-status`

- パッケージのモデレーション上の公開状態を確認するための所有者用コマンドです。
- `GET /api/v1/packages/{name}/moderation`を呼び出します。
- 現在のパッケージスキャン状態、未解決の報告数、最新リリースの手動
  モデレーション状態、ダウンロードのブロック状態、モデレーション理由を表示します。
- フラグ:
  - `--json`: 機械可読出力。

例:

```bash
clawhub package moderation-status @openclaw/example-plugin
```

### `package readiness <name>`

- パッケージが将来のOpenClawでの利用に対応できているかを確認します。
- `GET /api/v1/packages/{name}/readiness`を呼び出します。
- 公式ステータス、ClawPackの可用性、アーティファクトダイジェスト、
  ソースの来歴、OpenClawとの互換性、ホストターゲット、環境メタデータ、
  スキャン状態に関する阻害要因を報告します。
- フラグ:
  - `--json`: 機械可読出力。

例:

```bash
clawhub package readiness @openclaw/example-plugin
```

### `package migration-status <name>`

- OpenClawにバンドルされたPluginを置き換える可能性があるパッケージについて、
  運用担当者向けの移行状態を表示します。
- `package readiness`と同じ算出済み準備状況エンドポイントを呼び出しますが、
  移行に焦点を当てた状態、最新バージョン、公式パッケージの状態、チェック項目、
  阻害要因を出力します。
- フラグ:
  - `--json`: 機械可読出力。

例:

```bash
clawhub package migration-status @openclaw/example-plugin
```

### `publisher create <handle>`

- 認証済みユーザーが所有する組織パブリッシャーを作成します。
- ハンドルは小文字に正規化され、`@`の有無にかかわらず指定できます。
- 新しく作成された組織パブリッシャーは、デフォルトでは信頼済みまたは公式ではありません。
- 既存のパブリッシャー、ユーザー、または予約済みルートがハンドルをすでに使用している場合は失敗します。

```bash
clawhub publisher create opik --display-name "Opik"
```

### `package publish <source>`

- `POST /api/v1/packages`を介してコードPluginまたはバンドルPluginを公開します。
- `<source>`には以下を指定できます:
  - ローカルフォルダーパス: `./my-plugin`
  - ローカルのClawPack npm-pack tarball: `./my-plugin-1.2.3.tgz`
  - GitHubリポジトリ: `owner/repo`または`owner/repo@ref`
  - GitHub URL: `https://github.com/owner/repo`
- メタデータは、`package.json`、`openclaw.plugin.json`、および
  `.codex-plugin/plugin.json`、`.claude-plugin/plugin.json`、
  `.cursor-plugin/plugin.json`などの実際のOpenClawバンドルマーカーから自動検出されます。
- `.tgz`ソースはClawPackとして扱われます。CLIはnpm-packの正確な
  バイト列をアップロードし、展開された`package/`の内容は検証と
  メタデータの事前入力にのみ使用します。
- コードPluginのフォルダーは、アップロード前にClawPack npm tarballへパックされるため、
  OpenClawのインストール時に正確なアーティファクトを検証できます。バンドルPluginのフォルダーでは、
  引き続き展開済みファイルの公開パスを使用します。
- GitHubソースの場合、ソース帰属情報はリポジトリ、解決済みコミット、ref、サブパスから自動入力されます。
- ローカルフォルダーの場合、originリモートがGitHubを指していれば、ソース帰属情報がローカルgitから自動検出されます。
- 外部コードPluginでは、`openclaw.compat.pluginApi`と
  `openclaw.build.openclawVersion`を明示的に宣言する必要があります。
  トップレベルの`package.json.version`は、公開検証のフォールバックとして使用されません。
- `--dry-run`は、アップロードせずに解決済みの公開ペイロードをプレビューします。
- `--json`は、CI向けに機械可読出力を生成します。
- `--owner <handle>`は、実行者がパブリッシャーへのアクセス権を持つ場合に、ユーザーまたは組織パブリッシャーのハンドルで公開します。
- スコープ付きパッケージ名は、選択した所有者と一致する必要があります。`docs/publishing.md`を参照してください。
- 既存のフラグ（`--family`、`--name`、`--version`、`--source-repo`、`--source-commit`、`--source-ref`、`--source-path`）も引き続きオーバーライドとして機能します。
- プライベートGitHubリポジトリには`GITHUB_TOKEN`が必要です。

```bash
clawhub package publish ./plugin.tgz --owner openclaw
```

#### 推奨されるローカルフロー

ライブリリースを作成する前に、解決済みのパッケージメタデータと
ソース帰属情報を確認できるよう、まず`--dry-run`を使用します:

```bash
npm pack
clawhub package publish ./my-plugin-1.2.3.tgz --family code-plugin --dry-run
clawhub package publish ./my-plugin-1.2.3.tgz --family code-plugin
```

#### ローカルフォルダーのフロー

コードPluginでは、フォルダーからの公開時にパッケージフォルダーからClawPackアーティファクトを
ビルドしてアップロードします:

```bash
clawhub package publish ./my-plugin --family code-plugin --dry-run
clawhub package publish ./my-plugin --family code-plugin
```

#### `--family code-plugin`向けの最小限の`package.json`

外部コードPluginでは、`package.json`に少量のOpenClawメタデータが
必要です。次の最小限のマニフェストで正常に公開できます:

```json
{
  "name": "@myorg/openclaw-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "compat": {
      "pluginApi": ">=2026.3.24-beta.2"
    },
    "build": {
      "openclawVersion": "2026.3.24-beta.2"
    }
  }
}
```

必須フィールド:

- `openclaw.compat.pluginApi`
- `openclaw.build.openclawVersion`

注記:

- `package.json.version`はパッケージのリリースバージョンですが、OpenClawの
  互換性やビルド検証のフォールバックとしては使用されません。
- `openclaw.hostTargets`と`openclaw.environment`は任意のメタデータです。
  存在する場合はClawHubに表示されることがありますが、公開には必要ありません。
- `openclaw.compat.minGatewayVersion`と
  `openclaw.build.pluginSdkVersion`は、より詳細な互換性メタデータを公開する場合に
  追加できる任意項目です。
- 古い`clawhub` CLIリリースを使用している場合は、アップロード前に
  ローカルの事前チェックが実行されるよう、公開前にアップグレードしてください。
- 検証で修復コードが報告された場合は、
  [Plugin検証の修正方法](/ja-JP/clawhub/plugin-validation-fixes)を参照してください。

#### GitHub Actions

ClawHubでは、Pluginリポジトリ向けの公式再利用可能ワークフローも
[`/.github/workflows/package-publish.yml`](https://github.com/openclaw/clawhub/blob/62a697ef1e1b623afd71cf8813b545487a17354f/.github/workflows/package-publish.yml)
で提供しています。

一般的な呼び出し側の設定:

```yaml
name: Package Publish

on:
  pull_request:
  workflow_dispatch:
  push:
    tags:
      - "v*"

jobs:
  dry-run:
    if: github.event_name == 'pull_request'
    uses: openclaw/clawhub/.github/workflows/package-publish.yml@v0.12.0
    with:
      dry_run: true

  publish:
    if: github.event_name == 'workflow_dispatch' || startsWith(github.ref, 'refs/tags/')
    permissions:
      contents: read
      id-token: write
    uses: openclaw/clawhub/.github/workflows/package-publish.yml@v0.12.0
    with:
      dry_run: false
    secrets:
      clawhub_token: ${{ secrets.CLAWHUB_TOKEN }}
```

注記:

- 再利用可能ワークフローでは、`source`のデフォルトが呼び出し元リポジトリに設定されます。
- モノレポでは、ワークフローがPluginのパッケージフォルダーを公開するよう、
  `source_path`を渡します。例: `source_path: extensions/codex`。
- 再利用可能ワークフローは、安定版タグまたは完全なコミットSHAに固定してください。`@main`からリリース公開を実行しないでください。
- CIを汚染しないよう、`pull_request`では`dry_run: true`を使用してください。
- 実際の公開は、`workflow_dispatch`やタグのプッシュなど、信頼できるイベントに限定する必要があります。
- シークレットなしの信頼済み公開は`workflow_dispatch`でのみ機能します。タグのプッシュには引き続き`clawhub_token`が必要です。
- 初回公開、未信頼パッケージ、または緊急時の公開に備えて、`clawhub_token`を使用可能な状態にしておいてください。
- ワークフローはJSON結果をアーティファクトとしてアップロードし、ワークフロー出力として公開します。

### `package trusted-publisher get <name>`

- パッケージのGitHub Actions信頼済みパブリッシャー設定を表示します。
- 設定後にこれを使用して、リポジトリ、ワークフローファイル名、
  任意の環境固定を確認します。
- フラグ:
  - `--json`: 機械可読出力。

例:

```bash
clawhub package trusted-publisher get @openclaw/example-plugin
```

### `package trusted-publisher set <name>`

- 既存のパッケージにGitHub Actions信頼済みパブリッシャー設定を
  関連付けるか、置き換えます。
- 最初に、通常の手動またはトークン認証済みの`clawhub package publish`を介して
  パッケージを作成する必要があります。
- 設定後、今後サポート対象となるGitHub Actionsからの公開では、
  長期有効なClawHubトークンなしでOIDC/信頼済み公開を使用できます。
- `--repository <repo>`は`owner/repo`である必要があります。
- `--workflow-filename <file>`は、
  `.github/workflows/`内のワークフローファイル名と一致する必要があります。
- `--environment <name>`は任意です。設定した場合、OIDCクレーム内の
  GitHub Actions環境が完全に一致する必要があります。
- このコマンドの実行時に、ClawHubは設定されたGitHubリポジトリを検証します。
  パブリックリポジトリは、公開されているGitHubメタデータを通じて検証できます。プライベート
  リポジトリでは、将来のClawHub GitHub Appのインストールや、別の認可済み
  GitHubインテグレーションなどを通じて、ClawHubがそのリポジトリへのGitHubアクセス権を
  持っている必要があります。
- フラグ:
  - `--repository <repo>`: GitHubリポジトリ。例: `openclaw/example-plugin`。
  - `--workflow-filename <file>`: ワークフローファイル名。例: `package-publish.yml`。
  - `--environment <name>`: 任意の完全一致GitHub Actions環境。
  - `--json`: 機械可読出力。

例:

```bash
clawhub package trusted-publisher set @openclaw/example-plugin \
  --repository openclaw/example-plugin \
  --workflow-filename package-publish.yml \
  --environment release
```

### `package trusted-publisher delete <name>`

- パッケージから信頼済みパブリッシャー設定を削除します。
- ワークフロー、リポジトリ、または環境固定を無効化または再作成する必要がある場合に、
  ロールバックとして使用します。
- 設定を再度行うまで、今後の実際の公開では通常の認証済み公開を使用する必要があります。
- フラグ:
  - `--json`: 機械可読出力。

例:

```bash
clawhub package trusted-publisher delete @openclaw/example-plugin
```

### インストールテレメトリ

- ログイン中は、`CLAWHUB_DISABLE_TELEMETRY=1`が設定されていない限り、
  `clawhub install <slug>`の後に送信されます。
- 報告はベストエフォートです。テレメトリを利用できなくても、
  インストールコマンドは失敗しません。
- 詳細: `docs/telemetry.md`。
