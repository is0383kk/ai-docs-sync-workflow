---
read_when:
    - Firecrawl を利用したウェブ抽出が必要な場合
    - キーレスの Firecrawl Search（無料）またはキーレスの web_fetch を使用する場合
    - 検索または上限の引き上げには Firecrawl API キーが必要です
    - web_search プロバイダーとして Firecrawl を使用したい場合
    - web_fetch にボット対策を回避した抽出機能を求めている
summary: Firecrawl の検索、スクレイピング、web_fetch フォールバック
title: Firecrawl
x-i18n:
    generated_at: "2026-07-26T09:54:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 98b8af0839b1759e3be9393879a6d9a92fa0c505bf475bafd73c3f32d20fa106
    source_path: tools/firecrawl.md
    workflow: 16
---

OpenClaw は **Firecrawl** を 3 つの方法で使用できます。

- `web_search` プロバイダーとして
- 明示的な Plugin ツールとして: `firecrawl_search` および `firecrawl_scrape`
- `web_fetch` のフォールバック抽出ツールとして

これは、ボット回避とキャッシュをサポートするホスト型の抽出・検索サービスです。JS を多用するサイトや、通常の HTTP フェッチをブロックするページで役立ちます。

## Plugin のインストール

公式 Plugin をインストールしてから、Gateway を再起動します。

```bash
openclaw plugins install @openclaw/firecrawl-plugin
openclaw gateway restart
```

## キーレスアクセスと API キー

Firecrawl は 2 つの `web_search` プロバイダーを登録します。

- **Firecrawl 検索** (`firecrawl`) — キーを使用して、ホスト型の `/v2/search` API を
  使用します。キーが存在する場合は自動検出されます。
- **Firecrawl 検索（無料）** (`firecrawl-free`) — ホスト型のキーレススターター
  ティアを使用し、API キーは不要です。これは**明示的に選択した場合のみ**使用され、
  自動選択されることはありません。選択すると、検索クエリが Firecrawl の無料ティアに送信されるためです。

明示的に選択された Firecrawl の `web_fetch` フォールバックもキーレスです。明示的な
`firecrawl_search` および `firecrawl_scrape` ツールには API キーが必要です。上限を引き上げるには、
Gateway 環境に `FIRECRAWL_API_KEY` を追加するか、設定してください。

## Firecrawl 検索の設定

```json5
{
  tools: {
    web: {
      search: {
        provider: "firecrawl",
      },
    },
  },
  plugins: {
    entries: {
      firecrawl: {
        enabled: true,
        config: {
          webSearch: {
            apiKey: "FIRECRAWL_API_KEY_HERE",
            baseUrl: "https://api.firecrawl.dev",
          },
        },
      },
    },
  },
}
```

注:

- オンボーディングまたは `openclaw configure --section web` で Firecrawl を選択すると、インストール済みの Firecrawl Plugin が自動的に有効になります。
- API キーなしでキーレス実行するには、オンボーディングで **Firecrawl 検索（無料）** を選択します（または `provider: "firecrawl-free"` を設定します）。キーを使用する **Firecrawl 検索** プロバイダーは、`plugins.entries.firecrawl.config.webSearch.apiKey` または `FIRECRAWL_API_KEY` を送信します。
- Firecrawl を使用する `web_search` は、`query` と `count` をサポートします。
- `sources`、`categories`、結果のスクレイピングなど、Firecrawl 固有の制御には `firecrawl_search` を使用します。
- `baseUrl` のデフォルトは、`https://api.firecrawl.dev` のホスト型 Firecrawl です。セルフホストのオーバーライドはプライベートまたは内部エンドポイントにのみ許可され、HTTP はそれらのプライベートターゲットに対してのみ受け入れられます。
- `FIRECRAWL_BASE_URL` は、Firecrawl の検索およびスクレイピングのベース URL に共通する環境変数フォールバックです。
- Firecrawl 検索リクエストのデフォルトのタイムアウトは 30 秒です。`firecrawl_search` の `timeoutSeconds` パラメーターを指定すると、呼び出しごとに上書きできます。

## Firecrawl web_fetch フォールバックの設定

```json5
{
  tools: {
    web: {
      fetch: {
        provider: "firecrawl", // 明示的に選択するとキーレスフォールバックが有効になります
      },
    },
  },
  plugins: {
    entries: {
      firecrawl: {
        enabled: true,
        config: {
          webFetch: {
            baseUrl: "https://api.firecrawl.dev",
            onlyMainContent: true,
            maxAgeMs: 172800000,
            timeoutSeconds: 60,
          },
        },
      },
    },
  },
}
```

注:

- 明示的に選択された Firecrawl の `web_fetch` フォールバックは、API キーなしで動作します。設定されている場合、OpenClaw は上限を引き上げるために `plugins.entries.firecrawl.config.webFetch.apiKey` または `FIRECRAWL_API_KEY` を送信します。
- オンボーディングまたは `openclaw configure --section web` で Firecrawl を選択すると、Plugin が有効になり、別のフェッチプロバイダーがすでに設定されていない限り、`web_fetch` に Firecrawl が選択されます。
- `firecrawl_scrape` には API キーが必要です。
- `maxAgeMs` は、キャッシュ結果をどの程度古いものまで許可するか（ms）を制御します。デフォルトは 172,800,000 ms（2 日）です。
- `onlyMainContent` のデフォルトは `true`、`timeoutSeconds` のデフォルトは 60 です。
- 従来の `tools.web.fetch.firecrawl.*` および `tools.web.search.firecrawl.*` の設定は、`openclaw doctor --fix` によって自動移行されます。
- Firecrawl のスクレイピングおよびベース URL のオーバーライドには、検索と同じホスト型・プライベートのルールが適用されます。公開ホスト型トラフィックには `https://api.firecrawl.dev` を使用し、セルフホストのオーバーライドはプライベートまたは内部エンドポイントに解決される必要があります。
- `firecrawl_scrape` は、明らかなプライベート、ループバック、メタデータ、および HTTP(S) 以外のターゲット URL を Firecrawl に転送する前に拒否します。これは、明示的な Firecrawl スクレイピング呼び出しにおける `web_fetch` のターゲット安全性契約と一致します。

`firecrawl_scrape` は、必須の API キーを含め、同じ `plugins.entries.firecrawl.config.webFetch.*` 設定と環境変数を再利用します。

### セルフホスト Firecrawl

Firecrawl を自分で実行する場合は、`plugins.entries.firecrawl.config.webSearch.baseUrl`、`plugins.entries.firecrawl.config.webFetch.baseUrl`、または `FIRECRAWL_BASE_URL` を設定します。OpenClaw が `http://` を受け入れるのは、ループバック、プライベートネットワーク、`.local`、`.internal`、または `.localhost` のターゲットに対してのみです。Firecrawl API キーが誤って任意のエンドポイントに送信されないように、公開カスタムホストは拒否されます。

## Firecrawl Plugin ツール

### `firecrawl_search`

汎用の `web_search` ではなく、Firecrawl 固有の検索制御を使用する場合に使用します。API キーが必要です。

パラメーター:

- `query`
- `count` (1-100)
- `sources`
- `categories`
- `includeDomains` / `excludeDomains`（ホスト名のみ。相互排他）
- `tbs`（時間フィルター。例: `qdr:d`、`qdr:w`、`sbd:1`）
- `location` および `country`（地域ターゲティング）
- `scrapeResults`
- `timeoutSeconds`

### `firecrawl_scrape`

通常の `web_fetch` では十分に処理できない、JS を多用するページやボット保護されたページに使用します。

パラメーター:

- `url`
- `extractMode`
- `maxChars`
- `onlyMainContent`
- `maxAgeMs`
- `proxy`
- `storeInCache`
- `timeoutSeconds`

## ステルス / ボット回避

`firecrawl_scrape` および `web_fetch` の Firecrawl フォールバックでは、呼び出し元がこれらのパラメーターを上書きしない限り、デフォルトで `proxy: "auto"` と `storeInCache: true` を使用します。`firecrawl_search` および `web_search` の Firecrawl プロバイダーには、`proxy`/`storeInCache` の制御はありません。ステルスプロキシモードは、スクレイピングおよびフェッチリクエストにのみ適用されます。

Firecrawl の `proxy` モードは、ボット回避（`basic`、`stealth`、または `auto`）を制御します。基本的な試行が失敗した場合、`auto` はステルスプロキシで再試行します。そのため、基本スクレイピングのみの場合より多くのクレジットを消費することがあります。

## `web_fetch` が Firecrawl を使用する仕組み

`web_fetch` の抽出順序:

1. Readability（ローカル）
2. Firecrawl などの設定済みフェッチプロバイダー（選択されている場合、または設定済みの認証情報から自動検出された場合）
3. 基本的な HTML クリーンアップ（最後のフォールバック）

選択設定は `tools.web.fetch.provider` です。省略すると、OpenClaw は利用可能な認証情報から、準備ができている最初のウェブフェッチプロバイダーを自動検出します。公式 Firecrawl Plugin がそのフォールバックを提供します。

## 関連項目

- [Web 検索の概要](/ja-JP/tools/web) -- すべてのプロバイダーと自動検出
- [Web フェッチ](/ja-JP/tools/web-fetch) -- Firecrawl フォールバックを備えた web_fetch ツール
- [Tavily](/ja-JP/tools/tavily) -- 検索および抽出ツール
