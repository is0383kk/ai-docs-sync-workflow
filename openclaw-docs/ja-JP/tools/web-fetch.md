---
read_when:
    - URL を取得して読みやすいコンテンツを抽出する場合
    - web_fetch またはその Firecrawl フォールバックを設定する必要があります
    - web_fetch の制限とキャッシュについて理解したい場合
sidebarTitle: Web Fetch
summary: web_fetch ツール -- 読みやすいコンテンツ抽出を伴う HTTP フェッチ
title: Web フェッチ
x-i18n:
    generated_at: "2026-07-26T09:23:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ddf312245064672dcf489e8714740fa3e034827e16b33be8fb6a87db04f19ef8
    source_path: tools/web-fetch.md
    workflow: 16
---

`web_fetch` は通常の HTTP GET を実行し、読み取り可能なコンテンツ（HTML から
Markdown またはテキスト）を抽出します。JavaScript は実行**しません**。JS を多用するサイトや
ログインで保護されたページには、代わりに [Web Browser](/ja-JP/tools/browser) を使用してください。

## クイックスタート

デフォルトで有効になっており、設定は不要です。

```javascript
await web_fetch({ url: "https://example.com/article" });
```

## ツールパラメーター

<ParamField path="url" type="string" required>
取得する URL。`http(s)` のみ。
</ParamField>

<ParamField path="extractMode" type="'markdown' | 'text'" default="markdown">
メインコンテンツ抽出後の出力形式。
</ParamField>

<ParamField path="maxChars" type="number">
出力をこの文字数で切り詰めます。`tools.web.fetch.maxCharsCap` の範囲に制限されます。
</ParamField>

## 結果

`web_fetch` は、次のフィールドを持つ閉じた構造化結果を返します。

- リクエストメタデータ: `url`、`finalUrl`、`status`、`extractMode`、`extractor`
- 任意のレスポンスメタデータ: `contentType`、`title`、`warning`（存在しない場合は省略）
- ラップされたコンテンツのメタデータ: `externalContent`、`truncated`、`length`、`rawLength`、
  `fetchedAt`、`tookMs`、`text`
- キャッシュヒット時の任意の `cached: true`
- 切り詰められたコンテンツが非公開の一時ファイルに書き込まれた場合の任意の
  `spill: { path, chars, truncated? }`。`truncated` は、そのファイルにソースコンテンツの
  一部が含まれる場合にのみ存在します

`length` はラップされた `text` の長さです。`rawLength` は外部コンテンツでラップされる前の
抽出済みコンテンツの長さです。

## 動作の仕組み

<Steps>
  <Step title="取得">
    Chrome に似た User-Agent と `Accept-Language`
    ヘッダーを使用して HTTP GET を送信します。プライベート／内部ホスト名をブロックし、リダイレクトを再確認します。
  </Step>
  <Step title="抽出">
    HTML レスポンスに対して Readability（メインコンテンツ抽出）を実行します。
  </Step>
  <Step title="フォールバック（任意）">
    Readability が失敗し、取得プロバイダーが利用可能な場合、そのプロバイダーを通じて
    再試行します（たとえば Firecrawl のボット回避モード）。
  </Step>
  <Step title="キャッシュ">
    同じ URL の繰り返し取得を減らすため、結果は 15 分間（設定可能）
    キャッシュされます。
  </Step>
</Steps>

## 進行状況の更新

`web_fetch` は、5 秒後も取得が保留中の場合にのみ、公開の進行状況行を出力します。

```text
ページのコンテンツを取得しています...
```

高速なキャッシュヒットや短時間のネットワークレスポンスはタイマーが作動する前に完了するため、
進行状況行は表示されません。呼び出しをキャンセルするとタイマーがクリアされます。
進行状況行はチャンネル UI の状態にすぎず、取得したページコンテンツが含まれることはありません。

## 設定

```json5
{
  tools: {
    web: {
      fetch: {
        enabled: true, // デフォルト: true
        provider: "firecrawl", // 任意。自動検出する場合は省略
        maxChars: 20000, // デフォルトの出力文字数。maxCharsCap により上限設定
        maxCharsCap: 20000, // maxChars パラメーターのハード上限
        maxResponseBytes: 750000, // 切り詰め前の最大ダウンロードサイズ（32000-10000000）
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        useTrustedEnvProxy: false, // 信頼できる HTTP(S) 環境プロキシによる DNS 解決を許可
        readability: true, // Readability 抽出を使用
        userAgent: "Mozilla/5.0 ...", // User-Agent を上書き
        ssrfPolicy: {
          allowRfc2544BenchmarkRange: true, // 198.18.0.0/15 を使用する信頼できる偽 IP プロキシ向けのオプトイン
          allowIpv6UniqueLocalRange: true, // fc00::/7 を使用する信頼できる偽 IP プロキシ向けのオプトイン
        },
      },
    },
  },
}
```

## Firecrawl フォールバック

Readability 抽出が失敗した場合、`web_fetch` は、ボット回避と抽出精度の向上のために
[Firecrawl](/ja-JP/tools/firecrawl) へフォールバックできます。

```json5
{
  tools: {
    web: {
      fetch: {
        provider: "firecrawl", // 任意。利用可能な認証情報から自動検出する場合は省略
      },
    },
  },
  plugins: {
    entries: {
      firecrawl: {
        enabled: true,
        config: {
          webFetch: {
            // apiKey: "fc-...", // 任意。キーなしのスターターアクセスでは省略
            baseUrl: "https://api.firecrawl.dev",
            onlyMainContent: true,
            maxAgeMs: 172800000, // キャッシュ期間（2 日）
            timeoutSeconds: 60,
          },
        },
      },
    },
  },
}
```

`plugins.entries.firecrawl.config.webFetch.apiKey` は任意で、SecretRef オブジェクトをサポートします。
従来の `tools.web.fetch.firecrawl.*` 設定は、`openclaw doctor --fix` を介して
`plugins.entries.firecrawl.config.webFetch` に自動移行されます。

<Note>
  Firecrawl API キーの SecretRef を設定し、それが解決されず、
  `FIRECRAWL_API_KEY` 環境フォールバックもない場合、Gateway の起動は即座に失敗します。
</Note>

<Note>
  Firecrawl の `baseUrl` オーバーライドは厳しく制限されています。ホスト型トラフィックでは
  `https://api.firecrawl.dev` を使用します。セルフホスト型のオーバーライドはプライベートまたは
  内部エンドポイントを対象にする必要があり、`http://` はそれらのプライベートターゲットに対してのみ受け付けられます。
</Note>

現在のランタイム動作:

- `tools.web.fetch.provider` は取得フォールバックプロバイダーを明示的に選択します。
- `provider` が省略された場合、OpenClaw は設定済みの認証情報から、準備が整った最初の Web 取得
  プロバイダーを自動検出します。サンドボックス化されていない `web_fetch` は、
  `contracts.webFetchProviders` を宣言し、ランタイムで対応するプロバイダーを登録する
  インストール済み Plugin を使用できます。現在、公式 Firecrawl Plugin がこの
  フォールバックを提供しています。
- サンドボックス化された `web_fetch` 呼び出しでは、バンドルされたプロバイダーに加え、
  公式 npm または ClawHub の出自が検証されたインストール済みプロバイダーを許可します。現在許可されるのは
  公式 Firecrawl Plugin です。サードパーティーの外部取得 Plugin は引き続き除外されます。
- Readability が無効な場合、`web_fetch` は選択した
  プロバイダーフォールバックへ直接進みます。利用可能なプロバイダーがない場合は、フェイルクローズします。

## 信頼できる環境プロキシ

デプロイ環境で `web_fetch` を信頼できる送信 HTTP(S) プロキシ経由にする必要がある場合は、
`tools.web.fetch.useTrustedEnvProxy: true` を設定します。

このモードでも、OpenClaw はリクエスト送信前にホスト名ベースの SSRF チェックを適用しますが、
ローカル DNS ピンニングを行う代わりに、プロキシによる DNS 解決を許可します。
プロキシが運用者の管理下にあり、DNS 解決後に送信ポリシーを適用する場合にのみ有効にしてください。

<Note>
  HTTP(S) プロキシ環境変数が設定されていない場合、または対象ホストが
  `NO_PROXY` により除外されている場合、`web_fetch` はローカル DNS
  ピンニングを使用する通常の厳格なパスにフォールバックします。
</Note>

## 制限と安全性

- `maxChars` は `tools.web.fetch.maxCharsCap`（デフォルト `20000`）の範囲に制限されます
- レスポンス本文は解析前に `maxResponseBytes`（デフォルト `750000`、32000-10000000 の範囲に制限）
  を上限とします。上限を超えるレスポンスは警告付きで切り詰められます
- プライベート／内部ホスト名はブロックされます
- `tools.web.fetch.ssrfPolicy.allowRfc2544BenchmarkRange` と
  `tools.web.fetch.ssrfPolicy.allowIpv6UniqueLocalRange` は、信頼できる偽 IP プロキシスタック向けの限定的な
  オプトインです。プロキシがこれらの合成範囲を所有し、独自の宛先ポリシーを
  適用する場合を除き、未設定のままにしてください
- リダイレクトはチェックされ、`maxRedirects`（デフォルト `3`）によって制限されます
- `useTrustedEnvProxy` は明示的なオプトインであり、
  DNS 解決後も送信ポリシーを適用する運用者管理下のプロキシに対してのみ有効にしてください
- `web_fetch` はベストエフォートです。一部のサイトでは [Web Browser](/ja-JP/tools/browser) が必要です

## ツールプロファイル

ツールプロファイルまたは許可リストを使用する場合は、`web_fetch` または `group:web` を追加します。

```json5
{
  tools: {
    allow: ["web_fetch"],
    // または: allow: ["group:web"]  （web_fetch、web_search、x_search を含む）
  },
}
```

## 関連項目

- [Web Search](/ja-JP/tools/web) -- 複数のプロバイダーで Web を検索
- [Web Browser](/ja-JP/tools/browser) -- JS を多用するサイト向けの完全なブラウザー自動化
- [Firecrawl](/ja-JP/tools/firecrawl) -- Firecrawl の検索およびスクレイピングツール
