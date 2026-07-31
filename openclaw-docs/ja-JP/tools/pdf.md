---
read_when:
    - エージェントからの PDF を分析したい場合
    - pdf ツールの正確なパラメータと制限が必要です
    - ネイティブ PDF モードと抽出フォールバックの違いをデバッグしています
summary: ネイティブプロバイダーサポートと抽出フォールバックを使用して、1つ以上の PDF ドキュメントを分析する
title: PDF ツール
x-i18n:
    generated_at: "2026-07-26T10:24:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e0e5b897e1e122af4b2f6f9a3eaeb73f6e93af1051d306ad82539b258de90c49
    source_path: tools/pdf.md
    workflow: 16
---

`pdf` は 1 つ以上の PDF ドキュメントを解析し、テキストを返します。Anthropic および Google のモデルではネイティブのドキュメント入力を使用し、その他のすべてのプロバイダーではテキスト／画像抽出にフォールバックします。

## 利用可能性

このツールは、OpenClaw がエージェント用の PDF 対応モデルを解決できる場合にのみ登録されます。解決順序は次のとおりです。

1. `agents.defaults.pdfModel`（明示的なプライマリ／フォールバック）
2. `agents.defaults.imageModel`（明示的なプライマリ／フォールバック）
3. プロバイダーがネイティブ PDF 入力（Anthropic、Google）をサポートしているか、ビジョンモデルがすでに設定されている場合は、エージェントで解決されたセッション／デフォルトモデル
4. 使用可能な認証を持つ、自動検出された画像／ビジョン対応プロバイダー（ネイティブ PDF 対応プロバイダーを優先）

各フォールバック候補は使用前に認証チェックされるため、設定済みの `provider/model` が候補として扱われるのは、OpenClaw がそのエージェントについて当該プロバイダーで認証できる場合のみです。使用可能なモデルを解決できない場合、`pdf` ツールは公開されません。

## 入力リファレンス

<ParamField path="pdf" type="string">
1 つの PDF パスまたは URL。
</ParamField>

<ParamField path="pdfs" type="string[]">
複数の PDF パスまたは URL。合計で最大 10 個。
</ParamField>

<ParamField path="prompt" type="string" default="Analyze this PDF document.">
解析プロンプト。
</ParamField>

<ParamField path="pages" type="string">
`1-5` や `1,3,7-9` のようなページフィルター。ネイティブプロバイダーモードではサポートされません。
</ParamField>

<ParamField path="password" type="string">
暗号化された PDF のパスワード。リクエスト内のすべての PDF に適用され、抽出フォールバックモードでのみ使用されます。
</ParamField>

<ParamField path="model" type="string">
`provider/model` 形式の任意のモデルオーバーライド。
</ParamField>

<ParamField path="maxBytesMb" type="number">
PDF ごとのサイズ上限（MB）。デフォルトは `agents.defaults.pdfMaxMb`、未設定の場合は `10`。
</ParamField>

注記：

- `pdf` と `pdfs` は、読み込み前に統合および重複排除されます。少なくとも一方が必要です。
- `pages` は 1 始まりのページ番号として解析され、重複排除、並べ替えの後、`agents.defaults.pdfMaxPages`（デフォルト `20`）の範囲内に制限されます。範囲内のページに 1 つも一致しない場合、モデル呼び出し前にエラーになります。

## サポートされる PDF リファレンス

- ローカルファイルパス（`~` の展開を含む）
- `file://` URL
- `http://` および `https://` URL
- `media://inbound/<id>` など、OpenClaw が管理する受信リファレンス

その他の URI スキーム（例：`ftp://`）は `details.error = "unsupported_pdf_reference"` を返します。ツールがサンドボックス内で実行されている場合、リモートの `http(s)` URL は拒否されます。ワークスペース限定のファイルポリシーが有効な場合、許可されたルート外のローカルパスは拒否されますが、OpenClaw の受信メディアストア配下にある管理対象の受信リファレンスおよび再生されたパスは引き続き許可されます。

## 実行モード

### ネイティブプロバイダーモード

プロバイダー `anthropic` および `google`（現在ネイティブ PDF ドキュメント対応を宣言している唯一のプロバイダー）で使用されます。PDF の生バイトは、ファイルごとにネイティブドキュメント／インライン PDF パートとしてプロバイダー API に直接送信されます。

制限：

- `pages` はサポートされません。設定されている場合、ツールは `pages is not supported with native PDF providers` をスローします。
- `password` はサポートされません。設定されている場合、ツールは `password is not supported with native PDF providers` をスローします。暗号化された PDF には非ネイティブモデルを使用してください。

### 抽出フォールバックモード

その他のすべてのプロバイダーで使用されます。

1. バンドルされた `document-extract` Plugin を介して、選択されたページ（最大 `agents.defaults.pdfMaxPages`、デフォルト `20`）からテキストを抽出します。この Plugin は、テキストおよび画像の抽出に `clawpdf` パッケージ（PDFium WebAssembly）を使用します。
2. 抽出されたテキストが `200` 文字未満の場合、同じページを PNG 画像としてレンダリングします。レンダリング予算は合計 `4,000,000` ピクセルで、画像が必要なすべてのページ間で共有されます（ページごとではなく、残りの各ページに比例配分されます）。そのため、すでに十分なテキストがあるページではレンダリングを完全にスキップします。
3. 抽出したテキスト（およびレンダリングされた画像）とプロンプトを、選択したモデルに送信します。

詳細：

- 暗号化された PDF は、トップレベルの `password` パラメーターを使用して開かれます。
- モデルが画像入力に対応しておらず、抽出可能なテキストもない場合、ツールはエラーになります。
- 画像のレンダリングに失敗した場合、OpenClaw は画像を破棄し、抽出されたテキストで処理を続行します。
- 対象モデルがテキスト専用で、抽出によって画像が生成された場合、OpenClaw は画像を破棄してテキストのみを送信します。

## 設定

```json5
{
  agents: {
    defaults: {
      pdfModel: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["openai/gpt-5.4-mini"],
      },
      pdfMaxBytesMb: 10,
      pdfMaxPages: 20,
    },
  },
}
```

| キー                           | デフォルト | 意味                                                                                   |
| ----------------------------- | ------- | ----------------------------------------------------------------------------------------- |
| `agents.defaults.pdfModel`    | 未設定   | 明示的なプライマリ／フォールバック PDF モデル。`imageModel`、次にセッションモデルへフォールバックします。 |
| `agents.defaults.pdfMaxMb`    | `10`    | PDF ごとのサイズ上限（MB）。                                                                   |
| `agents.defaults.pdfMaxPages` | `20`    | PDF ごとに処理する最大ページ数。                                                              |

フィールドの詳細については、[設定リファレンス](/ja-JP/gateway/config-agents#agent-defaults)を参照してください。

## 出力の詳細

ツールは `content[0].text` にテキストを、`details` に構造化メタデータを返します。

一般的な `details` フィールド：

- `model`：解決されたモデルリファレンス（`provider/model`）
- `native`：ネイティブプロバイダーモードでは `true`、フォールバックでは `false`
- `attempts`：成功前に失敗したフォールバック試行

パスフィールド：

- 単一 PDF 入力：`details.pdf`
- 複数 PDF 入力：`pdf` エントリを持つ `details.pdfs[]`
- サンドボックスのパス書き換えメタデータ（該当する場合）：`rewrittenFrom`

## エラー動作

| 条件                         | 結果                                                         |
| --------------------------------- | -------------------------------------------------------------- |
| PDF 入力なし                      | `pdf required: provide a path or URL to a PDF document` をスロー |
| PDF が 10 個を超える                 | `details.error = "too_many_pdfs"`                              |
| サポートされていないリファレンススキーム      | `details.error = "unsupported_pdf_reference"`                  |
| ネイティブプロバイダーでの `pages`    | `pages is not supported with native PDF providers` をスロー      |
| ネイティブプロバイダーでの `password` | `password is not supported with native PDF providers` をスロー   |

## 例

単一 PDF：

```json
{
  "pdf": "/tmp/report.pdf",
  "prompt": "このレポートを 5 つの箇条書きで要約してください"
}
```

複数 PDF：

```json
{
  "pdfs": ["/tmp/q1.pdf", "/tmp/q2.pdf"],
  "prompt": "両方のドキュメントにおけるリスクとタイムラインの変更を比較してください"
}
```

ページフィルターを指定したフォールバックモデル：

```json
{
  "pdf": "https://example.com/report.pdf",
  "pages": "1-3,7",
  "model": "openai/gpt-5.4-mini",
  "prompt": "顧客に影響するインシデントのみを抽出してください"
}
```

抽出フォールバックを使用する暗号化 PDF：

```json
{
  "pdf": "/tmp/locked.pdf",
  "password": "example-password",
  "model": "openai/gpt-5.4-mini",
  "prompt": "この契約書を要約してください"
}
```

## 関連項目

- [ツールの概要](/ja-JP/tools) - 利用可能なすべてのエージェントツール
- [設定リファレンス](/ja-JP/gateway/config-agents#agent-defaults) - pdfMaxBytesMb および pdfMaxPages の設定
