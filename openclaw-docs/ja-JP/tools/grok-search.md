---
read_when:
    - web_search に Grok を使用する場合
    - Web 検索に xAI OAuth または XAI_API_KEY を使用する場合
summary: xAI の Web グラウンディング応答を使用した Grok Web 検索
title: Grok 検索
x-i18n:
    generated_at: "2026-07-26T10:33:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6e39edd660d0ffe8be066ae81317810da691a7dbd8c59a74222a59145cff5c77
    source_path: tools/grok-search.md
    workflow: 16
---

OpenClaw は Grok を `web_search` プロバイダーとしてサポートし、xAI のウェブグラウンディングされた
応答を使用して、ライブ検索結果に基づく引用付きの AI 合成回答を
生成します。

Grok ウェブ検索では、利用可能な既存の xAI OAuth サインインが優先されます。
OAuth プロファイルが存在しない場合、同じ xAI API キーにより、X（旧 Twitter）の投稿検索用の組み込み
`x_search` ツールと `code_execution`
ツールも利用できます。キーを `plugins.entries.xai.config.webSearch.apiKey` に保存すると、
OpenClaw はバンドルされた xAI モデルプロバイダーのフォールバックとしてもそのキーを再利用できます。

投稿単位の X 指標（リポスト、返信、ブックマーク、閲覧数）には、広範な検索クエリではなく、
正確な投稿 URL またはステータス ID を指定して
[`x_search`](/ja-JP/tools/web#x_search) を使用してください。

## オンボーディングと設定

`openclaw onboard` または `openclaw configure --section
web` の際に **Grok** を選択すると、OpenClaw は別のウェブ検索キーの入力を求めることなく、既存の xAI OAuth プロファイルを再利用できます。OAuth がない場合は、xAI API キーの設定にフォールバックします。

その後 OpenClaw は、同じ xAI 認証情報で `x_search` を有効にする追加手順を提示します。
この追加手順は次のとおりです。

- `web_search` に Grok を選択した後にのみ表示されます
- 独立した最上位のウェブ検索プロバイダー選択肢ではありません
- 同じフローで `x_search` モデルを任意で設定できます

スキップすると、後で設定から `x_search` を有効化または変更できます。

## サインインまたは API キーの取得

<Steps>
  <Step title="xAI OAuth を使用">
    オンボーディングまたはモデル認証ですでに xAI にサインインしている場合は、
    `web_search` プロバイダーとして Grok を選択します。別の API キーは不要です。

    ```bash
    openclaw onboard --auth-choice xai-oauth
    openclaw config set tools.web.search.provider grok
    ```

  </Step>
  <Step title="API キーのフォールバックを使用">
    OAuth を利用できない場合、または意図的にキーを使用するウェブ検索設定を使用する場合は、
    [xAI](https://console.x.ai/) から API キーを取得します。
  </Step>
  <Step title="キーを保存">
    Gateway 環境で `XAI_API_KEY` を設定するか、次のコマンドで構成します。

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

## 設定

```json5
{
  plugins: {
    entries: {
      xai: {
        config: {
          webSearch: {
            apiKey: "xai-...", // xAI OAuth または XAI_API_KEY を利用できる場合は任意
            baseUrl: "https://api.x.ai/v1", // 任意の Responses API プロキシ／ベース URL オーバーライド
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "grok",
      },
    },
  },
}
```

**認証情報の代替手段：** Gateway 環境の `openclaw models auth login --provider xai
--method oauth`、`XAI_API_KEY`、または
`plugins.entries.xai.config.webSearch.apiKey`。Gateway インストールでは、環境変数を
`~/.openclaw/.env` に配置します。

## 仕組み

Grok は、Gemini の Google 検索グラウンディング手法と同様に、xAI のウェブグラウンディングされた応答を使用して、
インライン引用付きの回答を合成します。

## サポートされるパラメーター

Grok 検索は `query` をサポートします。共有 `web_search`
との互換性のために `count` も受け付けますが、Grok は N 件の結果リストではなく、
常に引用付きの合成回答を 1 件返します。プロバイダー固有のフィルターはサポートされていません。

xAI Responses のウェブグラウンディング検索は共有の `web_search` のデフォルトより
長くかかる場合があるため、Grok のデフォルトタイムアウトは 60 秒です。
`tools.web.search.timeoutSeconds` でオーバーライドできます。

## ベース URL のオーバーライド

`plugins.entries.xai.config.webSearch.baseUrl` を設定すると、Grok ウェブ検索を
運用者のプロキシまたは xAI 互換の Responses エンドポイント経由でルーティングできます。OpenClaw は
末尾のスラッシュを削除した後、`<baseUrl>/responses` に POST します。`plugins.entries.xai.config.xSearch.baseUrl` が設定されていない限り、
`x_search` は同じ `webSearch.baseUrl` に
フォールバックします。

## 関連項目

- [ウェブ検索の概要](/ja-JP/tools/web) -- すべてのプロバイダーと自動検出
- [ウェブ検索の x_search](/ja-JP/tools/web#x_search) -- xAI を介したファーストクラスの X 検索
- [Gemini 検索](/ja-JP/tools/gemini-search) -- Google グラウンディングによる AI 合成回答
