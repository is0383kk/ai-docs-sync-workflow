---
read_when:
    - Skills の公開
    - 公開失敗のデバッグ
summary: Skill フォルダーの形式、必須ファイル、補助アーティファクト、制限。
x-i18n:
    generated_at: "2026-07-26T09:15:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fdf16a589b8961ccd9181a53a9fa92a358952b9147d22eaf977f23e0b4b4d653
    source_path: clawhub/skill-format.md
    workflow: 16
---

# Skill の形式

## ディスク上

Skill はフォルダーです。

必須:

- `SKILL.md`（または `skill.md`。従来の `skills.md` も使用できます）

任意:

- 補助となる通常ファイル（「Skill ファイル」を参照）
- `.clawhubignore`（公開時の無視パターン。従来の `.clawdhubignore`）
- `.gitignore`（こちらも適用されます）

## GitHub からのインポート

Web の GitHub インポーターは、ローカルでの公開や同期よりも制限が厳格です。サインイン中の GitHub アカウントが所有する、公開されている非フォークリポジトリ内の
`SKILL.md` または従来の `skills.md` ファイルのみを検出します。非公開リポジトリ、フォーク、
アーカイブ済みまたは無効化されたリポジトリ、第三者の公開リポジトリはインポートしません。

ローカルインストールのメタデータ（CLI により書き込まれます）:

- `<skill>/.clawhub/origin.json`（従来の `.clawdhub`）

作業ディレクトリのインストール状態（CLI により書き込まれます）:

- `<workdir>/.clawhub/lock.json`（従来の `.clawdhub`）

## `SKILL.md`

- 任意の YAML フロントマターを含む Markdown。
- サーバーは公開時にフロントマターからメタデータを抽出します。
- `description` は、UI や検索で Skill の概要として使用されます。

移植可能な Agent Skills では、`name` は親ディレクトリと一致し、
1～64 文字の小文字、数字、またはハイフンを使用する必要があります。ClawHub ではルーティング可能なスラッグと
カタログ表示名が分離されているため、他のクライアントで使用されている既存の名前も
公開でき、暗黙に書き換えられることはありません。カタログの一覧では、保存された名前を変更せずに、
長い名前を視覚的に短縮して表示する場合があります。

## フロントマターのメタデータ

Skill のメタデータは、`SKILL.md` の先頭にある YAML フロントマターで宣言します。これにより、その Skill の実行に必要なものがレジストリ（およびセキュリティ分析）に伝えられます。

### 基本的なフロントマター

```yaml
---
name: my-skill
description: この Skill の機能についての短い概要。
version: 1.0.0
---
```

### ランタイムメタデータ（`metadata.openclaw`）

Skill のランタイム要件を `metadata.openclaw`（別名: `metadata.clawdbot`、`metadata.clawdis`）の下に宣言します。

```yaml
---
name: my-skill
description: Todoist API を使用してタスクを管理します。
metadata:
  openclaw:
    requires:
      env:
        - TODOIST_API_KEY
      bins:
        - curl
    primaryEnv: TODOIST_API_KEY
---
```

Skill を実行する前に存在している必要がある環境変数には、`requires.env` を使用します。`required: false` を指定した任意の変数を含め、変数ごとのメタデータが必要な場合は `envVars` を使用します。

### 全フィールドのリファレンス

| フィールド              | 型       | 説明                                                                                                                                  |
| ------------------ | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `requires.env`     | `string[]` | Skill が必要とする必須の環境変数。                                                                                           |
| `requires.bins`    | `string[]` | すべてインストールされている必要がある CLI バイナリ。                                                                                                     |
| `requires.anyBins` | `string[]` | 少なくとも 1 つが存在する必要がある CLI バイナリ。                                                                                                  |
| `requires.config`  | `string[]` | Skill が読み取る設定ファイルのパス。                                                                                                          |
| `primaryEnv`       | `string`   | Skill の主要な認証情報用環境変数。                                                                                                  |
| `envVars`          | `array`    | `name`、任意の `required`、任意の `description` を含む環境変数宣言。任意の環境変数には `required: false` を設定します。 |
| `always`           | `boolean`  | `true` の場合、Skill は常に有効です（明示的なインストールは不要です）。                                                                              |
| `skillKey`         | `string`   | Skill の呼び出しキーを上書きします。                                                                                                         |
| `emoji`            | `string`   | Skill の表示用絵文字。                                                                                                                 |
| `homepage`         | `string`   | Skill のホームページまたはドキュメントの URL。                                                                                                         |
| `os`               | `string[]` | OS の制限（例: `["macos"]`、`["linux"]`）。                                                                                             |
| `install`          | `array`    | 依存関係のインストール仕様（下記参照）。                                                                                                  |
| `nix`              | `object`   | Nix Plugin の仕様（README を参照）。                                                                                                                |
| `config`           | `object`   | Clawdbot の設定仕様（README を参照）。                                                                                                           |

### インストール仕様

Skill で依存関係のインストールが必要な場合は、`install` 配列で宣言します。

```yaml
metadata:
  openclaw:
    install:
      - kind: brew
        formula: jq
        bins: [jq]
      - kind: node
        package: typescript
        bins: [tsc]
```

サポートされるインストール種別: `brew`、`node`、`go`、`uv`。

### 任意の環境変数

任意の環境変数を `metadata.openclaw.envVars` の下に宣言し、`required: false` を設定します。`requires.env` は、その変数なしでは Skill を実行できないことを意味するため、任意の項目を `requires.env` に追加しないでください。

```yaml
metadata:
  openclaw:
    primaryEnv: TODOIST_API_KEY
    envVars:
      - name: TODOIST_API_KEY
        required: true
        description: 認証済みリクエストに使用する Todoist API トークン。
      - name: TODOIST_PROJECT_ID
        required: false
        description: ユーザーが指定しなかった場合に使用する任意のデフォルトプロジェクト ID。
```

### これが重要な理由

ClawHub のセキュリティ分析では、Skill が宣言している内容と実際の動作が一致するかを確認します。コードが `TODOIST_API_KEY` を参照しているにもかかわらず、フロントマターの `requires.env`、`primaryEnv`、または `envVars` で宣言されていない場合、分析によりメタデータの不一致として報告されます。宣言を正確に保つことで、Skill がレビューに合格しやすくなり、ユーザーもインストール内容を理解しやすくなります。

### 例: 完全なフロントマター

```yaml
---
name: todoist-cli
description: コマンドラインから Todoist のタスク、プロジェクト、ラベルを管理します。
version: 1.2.0
metadata:
  openclaw:
    requires:
      env:
        - TODOIST_API_KEY
      bins:
        - curl
    primaryEnv: TODOIST_API_KEY
    envVars:
      - name: TODOIST_API_KEY
        required: true
        description: Todoist API トークン。
      - name: TODOIST_PROJECT_ID
        required: false
        description: 任意のデフォルトプロジェクト ID。
    emoji: "\u2705"
    homepage: https://github.com/example/todoist-cli
---
```

## Skill ファイル

公開時には、拡張子に関係なく Skill フォルダー内のすべての通常ファイルを使用できます。無視対象ファイル、
隠しパス、シンボリックリンク、macOS メタデータ、およびサーバー側のサイズ制限は引き続き適用されます。

- 有効な UTF-8 を含む制限範囲内のファイルは、エスケープされたプレーンテキストとしてプレビューでき、
  制限範囲内のテキスト分析に含まれます。
- その他のファイルは正確なバイト列を維持し、ダウンロードできます。
- セキュリティスキャナーには、保存された完全な成果物が渡されます。テキスト検出はレンダリングと
  分析に関するものであり、アップロードの許可リストではありません。

制限（サーバー側）:

- バンドルの合計サイズ: 50MB。
- 埋め込みテキストには、`SKILL.md` と最大約 40 個の制限範囲内の UTF-8 ファイルが含まれます（ベストエフォート方式の上限）。

## スラッグ

- デフォルトではフォルダー名から生成されます。
- パッケージスコープは、ClawHub の公開者ハンドルと完全に一致する必要があります。公開者ハンドルには、小文字、数字、ハイフン、ピリオド、アンダースコアを使用できます。先頭と末尾は小文字または数字である必要があります。
- パッケージスラッグは小文字かつ npm で使用可能な形式にする必要があります。例: `@example.tools/demo-plugin` または `demo-plugin`。

## バージョン管理とタグ

- 公開するたびに新しいバージョン（semver）が作成されます。
- タグはバージョンを指す文字列ポインターです。一般的には `latest` が使用されます。

## ライセンス

- ClawHub で公開されるすべての Skill には、`MIT-0` が適用されます。
- 公開された Skill は、商用利用を含め、誰でも使用、変更、再配布できます。
- 帰属表示は必要ありません。
- `SKILL.md` に競合するライセンス条項を追加しないでください。ClawHub は Skill ごとのライセンス上書きをサポートしていません。

## 有料 Skill

- ClawHub は、有料 Skill、Skill ごとの価格設定、ペイウォール、収益分配をサポートしていません。
- `SKILL.md` に価格メタデータを追加しないでください。これは Skill の形式には含まれず、公開された Skill が有料になることもありません。
- Skill が有料の第三者サービスと連携する場合は、Skill の手順および環境変数の宣言で、外部費用と必要なアカウントを明確に記載してください（必須変数には `requires.env`、任意の変数には `required: false` を指定した `envVars`）。
