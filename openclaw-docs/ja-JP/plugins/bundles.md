---
read_when:
    - Codex、Claude、または Cursor と互換性のあるバンドルをインストールしたい場合
    - OpenClaw がバンドルのコンテンツをネイティブ機能にどのようにマッピングするかを理解する必要があります
    - バンドル検出または不足している機能をデバッグしている場合
summary: Codex、Claude、Cursor のバンドルを OpenClaw Plugin としてインストールして使用する
title: Plugin バンドル
x-i18n:
    generated_at: "2026-07-26T09:41:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d44006866238f53ee2e3e8126cc4f7ed6f7413534257775f7904c9b877778c59
    source_path: plugins/bundles.md
    workflow: 16
---

OpenClaw は、**Codex**、**Claude**、**Cursor** という 3 つの外部エコシステムから Plugin をインストールできます。これらは **バンドル**と呼ばれ、OpenClaw が Skills、フック、MCP ツールなどのネイティブ機能にマッピングするコンテンツとメタデータのパックです。

<Info>
  バンドルはネイティブ OpenClaw Plugin と**同じではありません**。ネイティブ Plugin はプロセス内で実行され、あらゆる機能を登録できます。バンドルは、機能が選択的にマッピングされ、信頼境界がより狭いコンテンツパックです。
</Info>

## バンドルが存在する理由

多くの便利な Plugin は Codex、Claude、または Cursor 形式で公開されています。作成者にネイティブ OpenClaw Plugin として書き直すことを求める代わりに、OpenClaw はこれらの形式を検出し、対応しているコンテンツをネイティブ機能セットにマッピングします。Claude コマンドパックや Codex スキルバンドルをインストールして、すぐに使用できます。

## バンドルをインストールする

<Steps>
  <Step title="ディレクトリ、アーカイブ、またはマーケットプレイスからインストールする">
    ```bash
    # ローカルディレクトリ
    openclaw plugins install ./my-bundle

    # アーカイブ
    openclaw plugins install ./my-bundle.tgz

    # Claude マーケットプレイス
    openclaw plugins marketplace list <source>
    openclaw plugins install <plugin> --marketplace <source>
    ```

    `<source>` はローカルのマーケットプレイスパス／リポジトリ、または git／GitHub ソースです。

  </Step>

  <Step title="検出を確認する">
    ```bash
    openclaw plugins list
    openclaw plugins inspect <id>
    ```

    バンドルには、`Format: bundle` に加えて、値が `codex`、`claude`、または `cursor` の `Bundle format:` が表示されます。

  </Step>

  <Step title="再起動して使用する">
    ```bash
    openclaw gateway restart
    ```

    マッピングされた機能（Skills、フック、MCP ツール、LSP のデフォルト）は次のセッションで利用できます。

  </Step>
</Steps>

## OpenClaw がバンドルからマッピングするもの

現在、すべてのバンドル機能が OpenClaw で動作するわけではありません。ここでは、動作するものと、検出されるもののまだ接続されていないものを示します。

### 現在対応しているもの

| 機能          | マッピング方法                                                                                    | 対象           |
| ------------- | ------------------------------------------------------------------------------------------------- | -------------- |
| スキルコンテンツ | バンドルのスキルルートを通常の OpenClaw Skills として読み込む                                     | すべての形式   |
| コマンド      | `commands/` と `.cursor/commands/` をスキルルートとして扱う                                        | Claude、Cursor |
| フックパック  | OpenClaw 形式の `HOOK.md` + `handler.ts` レイアウト                                        | Codex          |
| MCP ツール    | バンドルの MCP 設定を埋め込み OpenClaw 設定にマージし、対応する stdio および HTTP サーバーを読み込む | すべての形式   |
| LSP サーバー  | Claude の `.lsp.json` とマニフェストで宣言された `lspServers` を埋め込み OpenClaw の LSP デフォルトにマージする | Claude         |
| 設定          | Claude の `settings.json` を埋め込み OpenClaw のデフォルトとしてインポートする                       | Claude         |

#### スキルコンテンツ

- バンドルのスキルルートは通常の OpenClaw スキルルートとして読み込まれます。
- Claude の `commands/` ルートは追加のスキルルートとして扱われます。
- Cursor の `.cursor/commands/` ルートは追加のスキルルートとして扱われます。

Claude の Markdown コマンドファイルと Cursor のコマンド Markdown は、どちらも通常の OpenClaw スキルローダーを通じて動作します。

#### フックパック

バンドルのフックルートは、通常の OpenClaw フックパックレイアウト（`HOOK.md` と `handler.ts` または `handler.js`）を使用する場合に**のみ**動作します。現在、これは主に Codex 互換の場合に該当します。

#### 埋め込み OpenClaw の MCP

- 有効なバンドルは MCP サーバー設定を提供できます。
- OpenClaw はバンドルの MCP 設定を `mcpServers` として、有効な埋め込み OpenClaw 設定にマージします。
- OpenClaw は、stdio サーバーを起動するか HTTP サーバーに接続することで、埋め込み OpenClaw エージェントのターン中に、対応するバンドル MCP ツールを公開します。
- `coding` および `messaging` ツールプロファイルには、デフォルトでバンドル MCP ツールが含まれます。エージェントまたは Gateway で無効にするには `tools.deny: ["bundle-mcp"]` を使用します。
- プロジェクトローカルの埋め込みエージェント設定は、バンドルのデフォルト適用後も有効になるため、必要に応じてワークスペース設定でバンドル MCP エントリを上書きできます。
- バンドル MCP ツールカタログは登録前に決定論的にソートされるため、上流の `listTools()` の順序が変わってもプロンプトキャッシュのツールブロックが頻繁に変動することはありません。

##### トランスポート

MCP サーバーは stdio または HTTP トランスポートを使用できます。

**Stdio** は子プロセスを起動します。

```json
{
  "mcp": {
    "servers": {
      "my-server": {
        "command": "node",
        "args": ["server.js"],
        "env": { "PORT": "3000" }
      }
    }
  }
}
```

**HTTP** は実行中の MCP サーバーに接続します。`streamable-http` が要求されない限り、デフォルトは `sse` です。

```json
{
  "mcp": {
    "servers": {
      "my-server": {
        "url": "http://localhost:3100/mcp",
        "transport": "streamable-http",
        "headers": {
          "Authorization": "Bearer ${MY_SECRET_TOKEN}"
        },
        "connectionTimeoutMs": 30000
      }
    }
  }
}
```

- `transport` は `"streamable-http"` または `"sse"` を受け付けます。省略時のデフォルトは `sse` です。
- `type: "http"` は CLI ネイティブのダウンストリーム形式です。OpenClaw 設定では `transport: "streamable-http"` を使用してください。`openclaw mcp set` と `openclaw doctor --fix` は一般的なエイリアスを正規化します。
- 許可される URL スキームは `http:` と `https:` のみです。
- `headers` の値では `${ENV_VAR}` 補間を使用できます。
- `command` と `url` の両方を含むサーバーエントリは拒否されます。
- URL の認証情報（ユーザー情報およびクエリパラメータ）は、ツールの説明とログから秘匿されます。
- `connectionTimeoutMs` は、stdio と HTTP の両方のトランスポートで、デフォルトの 30 秒の接続タイムアウトを上書きします。リクエストタイムアウトのデフォルトは 60 秒で、`requestTimeoutMs` で上書きできます。

##### ツールの命名

OpenClaw はバンドル MCP ツールを `serverName__toolName` 形式のプロバイダーで安全に使用できる名前で登録します。たとえば、キーが `"vigil-harbor"` のサーバーが `memory_search` ツールを公開している場合、`vigil-harbor__memory_search` として登録されます。

- `A-Za-z0-9_-` に含まれない文字は `-` に置き換えられます。
- 先頭が英字以外になるフラグメントには英字の接頭辞が付けられるため、`12306` のような数値のサーバーキーも、プロバイダーで安全に使用できるツール接頭辞になります。
- サーバー接頭辞は最大 30 文字です。
- 完全なツール名は最大 64 文字です。
- 空のサーバー名は `mcp` にフォールバックします。
- サニタイズ後の名前が衝突する場合は、数値の接尾辞で区別されます。
- 最終的に公開されるツールの順序は安全な名前によって決定論的に定まり、埋め込みエージェントのターンを繰り返してもキャッシュが安定します。
- プロファイルフィルタリングでは、1 つのバンドル MCP サーバーのすべてのツールを `bundle-mcp` が所有する Plugin として扱うため、プロファイルの許可／拒否リストでは、公開された個別のツール名または `bundle-mcp` Plugin キーのいずれかを参照できます。

#### 埋め込み OpenClaw 設定

バンドルが有効な場合、Claude の `settings.json` は埋め込み OpenClaw のデフォルト設定としてインポートされます。OpenClaw は適用前にシェル上書きキーをサニタイズします。

- `shellPath`
- `shellCommandPrefix`

#### 埋め込み OpenClaw LSP

- 有効な Claude バンドルは LSP サーバー設定を提供できます。
- OpenClaw は `.lsp.json` と、マニフェストで宣言されたすべての `lspServers` パスを読み込みます。
- バンドルの LSP 設定は、有効な埋め込み OpenClaw LSP のデフォルトにマージされます。
- 現在実行できるのは、対応する stdio ベースの LSP サーバーのみです。未対応のトランスポートも `openclaw plugins inspect <id>` には表示されます。

### 検出されるが実行されないもの

以下は認識され、診断に表示されますが、OpenClaw は実行しません。

- Claude の `agents`、`hooks/hooks.json` 自動化、`outputStyles`
- Cursor の `.cursor/agents`、`.cursor/hooks.json`、`.cursor/rules`
- 機能レポート以外の Codex `.app.json` メタデータ

## バンドル形式

<AccordionGroup>
  <Accordion title="Codex バンドル">
    マーカー：`.codex-plugin/plugin.json`

    オプションのコンテンツ：`skills/`、`hooks/`、`.mcp.json`、`.app.json`

    Codex バンドルは、スキルルートと OpenClaw 形式のフックパックディレクトリ（`HOOK.md` + `handler.ts`）を使用すると、OpenClaw に最も適合します。

  </Accordion>

  <Accordion title="Claude バンドル">
    2 つの検出モード：

    - **マニフェストベース：** `.claude-plugin/plugin.json`
    - **マニフェストなし：** デフォルトの Claude レイアウト（`skills/`、`commands/`、`agents/`、`hooks/`、`.mcp.json`、`.lsp.json`、`settings.json`）

    Claude 固有の動作：

    - `commands/` はスキルコンテンツとして扱われます
    - `settings.json` は埋め込み OpenClaw 設定にインポートされます（シェル上書きキーはサニタイズされます）
    - `.mcp.json` は対応する stdio ツールを埋め込み OpenClaw に公開します
    - `.lsp.json` とマニフェストで宣言された `lspServers` パスは、埋め込み OpenClaw LSP のデフォルトに読み込まれます
    - `hooks/hooks.json` は検出されますが実行されません
    - マニフェスト内のカスタムコンポーネントパスは追加的です。デフォルトを置き換えるのではなく拡張します

  </Accordion>

  <Accordion title="Cursor バンドル">
    マーカー：`.cursor-plugin/plugin.json`

    オプションのコンテンツ：`skills/`、`.cursor/commands/`、`.cursor/agents/`、`.cursor/rules/`、`.cursor/hooks.json`、`.mcp.json`

    - `.cursor/commands/` はスキルコンテンツとして扱われます
    - `.cursor/rules/`、`.cursor/agents/`、`.cursor/hooks.json` は検出のみ行われます

  </Accordion>
</AccordionGroup>

## 検出の優先順位

OpenClaw は最初にネイティブ Plugin 形式を確認します。

1. `openclaw.plugin.json`、または `openclaw.extensions` を含む有効な `package.json` — **ネイティブ Plugin**として扱われます
2. バンドルマーカー（`.codex-plugin/`、`.claude-plugin/`、またはデフォルトの Claude／Cursor レイアウト）— **バンドル**として扱われます

ディレクトリに両方が含まれている場合、OpenClaw はネイティブパスを使用します。これにより、デュアルフォーマットパッケージがバンドルとして部分的にインストールされることを防ぎます。

## ランタイム依存関係とクリーンアップ

- サードパーティ製の互換バンドルでは、起動時の `npm install` 修復は行われません。`openclaw plugins install` を通じてインストールし、必要なものをすべてインストール済み Plugin ディレクトリに同梱する必要があります。
- OpenClaw が所有するバンドル済み Plugin は、コアに軽量な形で同梱されるか、Plugin インストーラーからダウンロードできます。Gateway の起動時に、これらのためにパッケージマネージャーが実行されることはありません。
- `openclaw doctor --fix` は古いローカルのバンドル済み Plugin インストールレコードを削除し、設定から引き続き参照されているものの、ローカル Plugin インデックスに存在しないダウンロード可能な Plugin を復旧できます。

## セキュリティ

バンドルの信頼境界は、ネイティブ Plugin よりも狭くなっています。

- OpenClaw は任意のバンドルランタイムモジュールをプロセス内に読み込み**ません**。
- Skills とフックパックのパスは Plugin ルート内に収まる必要があります（境界チェック済み）。
- 設定ファイルは同じ境界チェックを使用して読み込まれます。
- 対応する stdio MCP サーバーはサブプロセスとして起動される場合があります。

このため、バンドルはデフォルトでより安全ですが、サードパーティ製バンドルが公開する機能については、信頼できるコンテンツとして扱う必要があります。

## トラブルシューティング

<AccordionGroup>
  <Accordion title="バンドルは検出されるが、機能が実行されない">
    `openclaw plugins inspect <id>` を実行します。機能が一覧に表示されていても、
    未接続と記載されている場合、それはインストールの不具合ではなく、製品上の制限です。
  </Accordion>

  <Accordion title="Claude コマンドファイルが表示されない">
    バンドルが有効になっており、Markdown ファイルが検出済みの
    `commands/` または `skills/` ルート内にあることを確認します。
  </Accordion>

  <Accordion title="Claude の設定が適用されない">
    `settings.json` の埋め込み OpenClaw 設定のみがサポートされています。OpenClaw は
    バンドル設定を未加工の設定パッチとして扱いません。
  </Accordion>

  <Accordion title="Claude フックが実行されない">
    `hooks/hooks.json` は検出専用です。実行可能なフックが必要な場合は、
    OpenClaw のフックパックレイアウトを使用するか、ネイティブ Plugin を配布してください。
  </Accordion>
</AccordionGroup>

## 関連項目

- [Plugin のインストールと設定](/ja-JP/tools/plugin)
- [Plugin の構築](/ja-JP/plugins/building-plugins) - ネイティブ Plugin を作成する
- [Plugin マニフェスト](/ja-JP/plugins/manifest) - ネイティブマニフェストのスキーマ
