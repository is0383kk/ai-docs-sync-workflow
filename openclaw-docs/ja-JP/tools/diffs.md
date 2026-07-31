---
read_when:
    - エージェントにコードや Markdown の編集内容を差分として表示させたい場合
    - キャンバス対応のビューアー URL またはレンダリング済みの差分ファイルが必要です
    - 安全なデフォルト設定を備えた、管理された一時的な差分アーティファクトが必要です
sidebarTitle: Diffs
summary: エージェント向けの読み取り専用差分ビューアーおよびファイルレンダラー（オプションのPluginツール）
title: 差分
x-i18n:
    generated_at: "2026-07-26T09:54:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: baeb5dd1277120e57178f092e3ae1616edd3389a54721c929d8711301535d302
    source_path: tools/diffs.md
    workflow: 16
---

`diffs` は、変更前/変更後のテキストまたは unified patch を読み取り専用の差分アーティファクトに変換する、オプションのバンドル Plugin ツールです。また、システムプロンプトの先頭にエージェント向けの短いガイダンスを追加し、より詳しい手順を提供する付属の skill も同梱されています。

入力: `before` + `after` テキスト、または unified `patch`（相互排他）。

出力: canvas 表示用の Gateway ビューアー URL、メッセージ配信用にレンダリングされた PNG/PDF ファイルパス、またはその両方。

## クイックスタート

<Steps>
  <Step title="Plugin をインストール">
    ```bash
    openclaw plugins install diffs
    ```
  </Step>
  <Step title="Plugin を有効化">
    ```json5
    {
      plugins: {
        entries: {
          diffs: {
            enabled: true,
          },
        },
      },
    }
    ```
  </Step>
  <Step title="モードを選択">
    <Tabs>
      <Tab title="view">
        canvas 優先のフロー: エージェントは `diffs` を `mode: "view"` で呼び出し、`details.viewerUrl` を `canvas present` で開きます。
      </Tab>
      <Tab title="file">
        チャットでのファイル配信: エージェントは `diffs` を `mode: "file"` で呼び出し、`details.filePath` を `message` で、`path` または `filePath` を使用して送信します。
      </Tab>
      <Tab title="both">
        組み合わせ（デフォルト）: エージェントは `diffs` を `mode: "both"` で呼び出し、1 回の呼び出しで両方のアーティファクトを取得します。
      </Tab>
    </Tabs>
  </Step>
</Steps>

## 組み込みのシステムガイダンスを無効化

ツールを維持したまま、先頭に追加されるシステムプロンプトのガイダンスを削除するには、`plugins.entries.diffs.hooks.allowPromptInjection` を `false` に設定します。

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
      },
    },
  },
}
```

これにより、ツールと skill を利用可能な状態に保ちながら、Plugin の `before_prompt_build` フックがブロックされます。ガイダンスとツールの両方を無効化するには、代わりに Plugin を無効化します。

## ツール入力リファレンス

特記がない限り、すべてのフィールドは任意です。

<ParamField path="before" type="string">
  元のテキスト。`patch` を省略する場合、`after` とともに必須です。
</ParamField>
<ParamField path="after" type="string">
  更新後のテキスト。`patch` を省略する場合、`before` とともに必須です。
</ParamField>
<ParamField path="patch" type="string">
  unified diff テキスト。`before` および `after` とは相互排他です。
</ParamField>
<ParamField path="path" type="string">
  変更前/変更後モードで表示するファイル名。
</ParamField>
<ParamField path="lang" type="string">
  変更前/変更後モードの言語上書きヒント。不明な値やデフォルトのビューアーセットに含まれない言語は、Diff Viewer Language Pack Plugin がインストールされていない限り、プレーンテキストにフォールバックします。
</ParamField>
<ParamField path="title" type="string">
  ビューアータイトルの上書き。
</ParamField>
<ParamField path="mode" type='"view" | "file" | "both"'>
  出力モード。Plugin のデフォルト `defaults.mode`（`both`）が使用されます。非推奨のエイリアス: `"image"` は `"file"` と同じように動作します。
</ParamField>
<ParamField path="theme" type='"light" | "dark"'>
  ビューアーのテーマ。Plugin のデフォルト `defaults.theme` が使用されます。
</ParamField>
<ParamField path="layout" type='"unified" | "split"'>
  差分レイアウト。Plugin のデフォルト `defaults.layout` が使用されます。
</ParamField>
<ParamField path="expandUnchanged" type="boolean">
  完全なコンテキストを利用できる場合に、変更されていないセクションを展開します。呼び出しごとのオプションのみです（Plugin のデフォルトキーではありません）。
</ParamField>
<ParamField path="fileFormat" type='"png" | "pdf"'>
  レンダリングされるファイル形式。Plugin のデフォルト `defaults.fileFormat` が使用されます。
</ParamField>
<ParamField path="fileQuality" type='"standard" | "hq" | "print"'>
  PNG/PDF レンダリングの品質プリセット。
</ParamField>
<ParamField path="fileScale" type="number">
  デバイススケールの上書き（`1`-`4`）。
</ParamField>
<ParamField path="fileMaxWidth" type="number">
  CSS ピクセル単位の最大レンダリング幅（`640`-`2400`）。
</ParamField>
<ParamField path="ttlSeconds" type="number" default="1800">
  ビューアーおよび単独ファイル出力のアーティファクト TTL（秒単位）。最大 `21600`。
</ParamField>
<ParamField path="baseUrl" type="string">
  ビューアー URL のオリジンの上書き。Plugin の `viewerBaseUrl` を上書きします。クエリ/ハッシュを含まない `http` または `https` でなければなりません。
</ParamField>

<AccordionGroup>
  <Accordion title="検証と制限">
    - `before`/`after`: それぞれ最大 512 KiB。
    - `patch`: 最大 2 MiB。
    - `path`: 最大 2048 バイト。
    - `lang`: 最大 128 バイト。
    - `title`: 最大 1024 バイト。
    - パッチ複雑度の上限: 最大 128 ファイル、合計 120000 行。
    - `patch` と `before`/`after` の併用は拒否されます。
    - レンダリングされるファイルの安全制限（PNG および PDF）:
      - `fileQuality: "standard"`: 最大 8 MP（レンダリングされるピクセル数 8,000,000）。
      - `fileQuality: "hq"`: 最大 14 MP。
      - `fileQuality: "print"`: 最大 24 MP。
      - PDF はさらに最大 50 ページに制限されます。

  </Accordion>
</AccordionGroup>

## 構文ハイライト

組み込み言語:

`javascript`、`typescript`、`tsx`、`jsx`、`json`、`markdown`、`yaml`、`css`、`html`、`sh`、`python`、`go`、`rust`、`java`、`c`、`cpp`、`csharp`、`php`、`sql`、`docker`、`ruby`、`swift`、`kotlin`、`r`、`dart`、`lua`、`powershell`、`xml`、および `toml`。

一般的なエイリアス（`js`、`ts`、`bash`、`md`、`yml`、`c++`、`dockerfile`、`rb`、`kt`、`ps1` など）は、これらの言語に正規化されます。

より多くの言語（Astro、Vue、Svelte、MDX、GraphQL、Terraform/HCL、Nix、Clojure、Elixir、Haskell、OCaml、Scala、Zig、Solidity、Verilog/VHDL、Fortran、MATLAB、LaTeX、Mermaid、Sass/Less/SCSS、Nginx、Apache、CSV、dotenv、INI、diff など）に対応するには、Diff Viewer Language Pack Plugin をインストールします。

```bash
openclaw plugins install clawhub:@openclaw/diffs-language-pack
```

パックがなくても、未対応の言語は読みやすいプレーンテキストとしてレンダリングされます。アップストリームのカタログについては、[Diffs Language Pack Plugin](/ja-JP/plugins/reference/diffs-language-pack) および [Shiki の言語](https://shiki.style/languages) を参照してください。

## 出力詳細のコントラクト

成功したすべての結果には `changed` が含まれます。変更前と変更後の入力が同一の場合、アーティファクトを作成せずに `false` が返され、レンダリングされた結果では `true` が返されます。

<AccordionGroup>
  <Accordion title="ビューアーフィールド（view および both モード）">
    - `changed`
    - `artifactId`
    - `viewerUrl`
    - `viewerPath`
    - `title`
    - `expiresAt`
    - `inputKind`
    - `fileCount`
    - `mode`
    - `context`（利用可能な場合は `agentId`、`sessionId`、`messageChannel`、`agentAccountId`）

  </Accordion>
  <Accordion title="ファイルフィールド（file および both モード）">
    - `changed`
    - `artifactId`
    - `expiresAt`
    - `filePath`
    - `path`（メッセージツールとの互換性のため、`filePath` と同じ値）
    - `fileBytes`
    - `fileFormat`
    - `fileQuality`
    - `fileScale`
    - `fileMaxWidth`

  </Accordion>
</AccordionGroup>

| モード     | 返される内容                                                                                         |
| -------- | ----------------------------------------------------------------------------------------------- |
| `"view"` | ビューアーフィールドのみ。                                                                             |
| `"file"` | ファイルフィールドのみ。ビューアーアーティファクトはありません。                                                           |
| `"both"` | ビューアーフィールドとファイルフィールド。ファイルのレンダリングに失敗しても、ビューアーは `fileError` とともに返されます。 |

### 折りたたまれた未変更セクション

ビューアーには `N unmodified lines` のような行が表示されます。展開コントロールは、レンダリングされた差分に展開可能なコンテキストデータがある場合にのみ表示されます（通常は変更前/変更後の入力）。多くの unified patch ではハンク内のコンテキスト本体が省略されるため、展開コントロールなしで行が表示されることがあります。これは想定された動作であり、バグではありません。`expandUnchanged` は、展開可能なコンテキストが存在する場合にのみ適用されます。

### 複数ファイルのナビゲーション

複数のファイルに変更を加えるパッチは、変更されたファイルの概要カードから始まります。合計 `+N` / `-N` 件数、ファイルごとの件数、追加/削除/名前変更のバッジ、および各ファイルへ移動するアンカーリンクが表示されます。レンダリングされた PNG/PDF ファイルではファイルごとのヘッダー件数は維持されますが、静的ファイルでは機能しないため、インタラクティブな表示切り替えは削除されます。

## Plugin のデフォルト

Plugin 全体のデフォルトを `~/.openclaw/openclaw.json` に設定します。

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          defaults: {
            fontFamily: "Fira Code",
            fontSize: 15,
            lineSpacing: 1.6,
            layout: "unified",
            showLineNumbers: true,
            diffIndicators: "bars",
            wordWrap: true,
            background: true,
            theme: "dark",
            fileFormat: "png",
            fileQuality: "standard",
            fileScale: 2,
            fileMaxWidth: 960,
            mode: "both",
            ttlSeconds: 21600,
          },
        },
      },
    },
  },
}
```

対応する `defaults` キー: `fontFamily`、`fontSize`、`lineSpacing`、`layout`、`showLineNumbers`、`diffIndicators`、`wordWrap`、`background`、`theme`、`fileFormat`、`fileQuality`、`fileScale`、`fileMaxWidth`、`mode`、`ttlSeconds`。明示的なツール呼び出しパラメーターは、これらを上書きします。

### 永続的なビューアー URL 設定

<ParamField path="viewerBaseUrl" type="string">
  ツール呼び出しで `baseUrl` が渡されない場合に返されるビューアーリンク用の、Plugin が所有するフォールバック。クエリ/ハッシュを含まない `http` または `https` でなければなりません。
</ParamField>

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          viewerBaseUrl: "https://gateway.example.com/openclaw",
        },
      },
    },
  },
}
```

## セキュリティ設定

<ParamField path="security.allowRemoteViewer" type="boolean" default="false">
  `false`: ビューアールートへの非ループバックリクエストは拒否されます。`true`: トークン化されたパスが有効な場合、リモートビューアーが許可されます。
</ParamField>

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          security: {
            allowRemoteViewer: false,
          },
        },
      },
    },
  },
}
```

## アーティファクトのライフサイクルとストレージ

- ビューアーの HTML とメタデータは、Diffs Plugin の blob 名前空間にある共有 `state/openclaw.sqlite` データベース内に保存されます。HTML は gzip 圧縮されます。SQLite に保存されるのはランダムな URL トークンの SHA-256 ハッシュのみで、トークン自体は保存されません。
- レンダリングされた PNG/PDF ファイルは、チャンネル配信にファイルパスが必要なため、`$TMPDIR/openclaw-diffs` 配下に一時的な実体として残ります。有効期限のメタデータは SQLite が管理し、JSON サイドカーは書き込まれません。
- デフォルトの成果物 TTL: 30 分。受け入れ可能な最大 TTL: 6 時間。
- クリーンアップは、成果物を作成する各呼び出しの後に随時実行されます。期限切れの SQLite 行が最初に削除され、続いて対応する PNG/PDF ディレクトリが削除されます。
- フォールバックのスイープ処理により、対応する行がなく、24 時間を超えて古い一時フォルダーが削除されます。従来の `meta.json`、`file-meta.json`、`viewer.html` キャッシュはインポートも読み取りもされません。

## ビューアー URL とネットワーク動作

ビューアールート: `/plugins/diffs/view/{artifactId}/{token}`

ビューアーアセット:

- `/plugins/diffs/assets/viewer.js`
- `/plugins/diffs/assets/viewer-runtime.js`
- `/plugins/diffs-language-pack/assets/viewer.js`（diff が言語パックの言語を使用する場合のみ）

ビューアードキュメントは、これらのアセットをビューアー URL からの相対パスで解決するため、オプションの `baseUrl` パスプレフィックスもアセットリクエストに引き継がれます。

URL の解決順序: ツール呼び出しの `baseUrl`（厳密な検証後）-> Plugin の `viewerBaseUrl` -> loopback のデフォルト `127.0.0.1`。Gateway のバインドモードが `custom` で、`gateway.customBindHost` が設定されている場合は、loopback の代わりにそのホストが使用されます。

`baseUrl` のルール: `http://` または `https://` である必要があります。クエリとハッシュは拒否されます。オリジンにオプションのベースパスを加えた形式が許可されます。

## セキュリティモデル

<AccordionGroup>
  <Accordion title="ビューアーの堅牢化">
    - デフォルトでは loopback のみ。
    - 厳密な ID およびトークンのパターン検証を伴う、トークン化されたビューアーパス。
    - ビューアーレスポンスの CSP: `default-src 'none'`。スクリプトとアセットは同一オリジンからのみ許可され、外部への `connect-src` はありません。
    - リモートアクセスが有効な場合のリモートミスのスロットリング: 60 秒間に 40 回失敗すると、60 秒間ロックアウトされます（`429 Too Many Requests`）。

  </Accordion>
  <Accordion title="ファイルレンダリングの堅牢化">
    - スクリーンショット用ブラウザーのリクエストルーティングは、デフォルトで拒否されます。
    - `http://127.0.0.1/plugins/diffs/assets/*` からのローカルビューアーアセットのみが許可されます。
    - 外部ネットワークリクエストはブロックされます。

  </Accordion>
</AccordionGroup>

## ファイルモードのブラウザー要件

`mode: "file"` と `mode: "both"` には Chromium 互換ブラウザーが必要です。

解決順序:

<Steps>
  <Step title="設定">
    OpenClaw 設定内の `browser.executablePath`。
  </Step>
  <Step title="環境変数">
    - `OPENCLAW_BROWSER_EXECUTABLE_PATH`
    - `BROWSER_EXECUTABLE_PATH`
    - `PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH`

  </Step>
  <Step title="プラットフォームのフォールバック">
    Chrome、Chromium、Edge、Brave の一般的なインストールパスと `PATH` 検索。
  </Step>
</Steps>

一般的なエラーテキスト: `Diff PNG/PDF rendering requires a Chromium-compatible browser...`。Chrome、Chromium、Edge、Brave のいずれかをインストールするか、上記の実行可能ファイルパスオプションのいずれかを設定して修正します。

## トラブルシューティング

<AccordionGroup>
  <Accordion title="入力検証エラー">
    - `Provide patch or both before and after text.` -- `before` と `after` の両方を含めるか、`patch` を指定してください。
    - `Provide either patch or before/after input, not both.` -- 入力モードを混在させないでください。
    - `Invalid baseUrl: ...` -- オプションのパスを含む `http(s)` オリジンを使用し、クエリやハッシュは含めないでください。
    - `{field} exceeds maximum size (...)` -- ペイロードサイズを減らしてください。
    - 大きなパッチの拒否 -- パッチファイル数または合計行数を減らしてください。

  </Accordion>
  <Accordion title="ビューアーのアクセス性">
    - ビューアー URL はデフォルトで `127.0.0.1` に解決されます。
    - リモートアクセスには、Plugin の `viewerBaseUrl` を設定するか、呼び出しごとに `baseUrl` を渡すか、`gateway.bind=custom` を `gateway.customBindHost` とともに使用します。
    - 同一ホスト上のプロキシ（たとえば Tailscale Serve）のために `gateway.trustedProxies` に loopback が含まれている場合、転送されたクライアント IP ヘッダーのない直接の loopback ビューアーリクエストは、設計上フェイルクローズします。
    - そのプロキシトポロジでは、添付ファイルには `mode: "file"`/`"both"` を優先してください。共有可能なビューアーリンクには、`security.allowRemoteViewer` に加えて Plugin の `viewerBaseUrl`/プロキシの `baseUrl` を意図的に有効化してください。
    - 外部からのビューアーアクセスを意図する場合にのみ、`security.allowRemoteViewer` を有効にしてください。

  </Accordion>
  <Accordion title="未変更行の行に展開ボタンがない">
    展開可能なコンテキストを含まないパッチ入力では想定どおりの動作であり、ビューアーの障害ではありません。
  </Accordion>
  <Accordion title="成果物が見つからない">
    - TTL により成果物の有効期限が切れました。
    - トークンまたはパスが変更されました。
    - クリーンアップにより古いデータが削除されました。

  </Accordion>
</AccordionGroup>

## 運用ガイダンス

- キャンバスでローカルの対話型レビューを行う場合は、`mode: "view"` を優先してください。
- 添付ファイルを必要とする外向きのチャットチャンネルには、`mode: "file"` を優先してください。
- デプロイでリモートビューアー URL が必要な場合を除き、`allowRemoteViewer` は無効のままにしてください。
- 機密性の高い diff には、明示的に短い `ttlSeconds` を設定してください。
- 必要でない場合は、diff 入力にシークレットを送信しないでください。
- チャンネルが画像を強く圧縮する場合（たとえば Telegram や WhatsApp）は、PDF 出力（`fileFormat: "pdf"`）を優先してください。

<Note>
diff レンダリングエンジンは [Diffs](https://diffs.com) を使用しています。
</Note>

## 関連項目

- [ブラウザー](/ja-JP/tools/browser)
- [Plugin](/ja-JP/tools/plugin)
- [ツールの概要](/ja-JP/tools)
