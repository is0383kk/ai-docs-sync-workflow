---
read_when:
    - Canvas のホスト、ツール、コマンド、ドキュメント、またはプロトコルの所有権の移管
    - Canvas が引き続きコア所有かどうかの監査
    - 実験的な Canvas Plugin の PR の準備またはレビュー
summary: Canvas をコアからバンドルされた実験的 Plugin に移行するための計画および監査チェックリスト。
title: Canvas Plugin のリファクタリング
x-i18n:
    generated_at: "2026-07-26T09:18:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ead3f865ea80acb1e47f45a5ab07acf19a6470035c00c81006b2b1230bedd71e
    source_path: refactor/canvas.md
    workflow: 16
---

# Canvas plugin のリファクタリング

Canvas は利用頻度が低い実験的な機能です。コア機能ではなく、バンドルされた plugin として扱います。コアには汎用的な Gateway、Node、HTTP、認証、設定、ネイティブクライアントの基盤を残せますが、Canvas 固有の動作は `extensions/canvas` 配下に置く必要があります。

## 目標

現在のペアリング済み Node の動作を維持しながら、Canvas の所有権を `extensions/canvas` に移します。

- エージェント向けの `canvas` ツールは Canvas plugin によって登録される
- Canvas Node コマンドは Canvas plugin が登録した場合にのみ許可される
- A2UI のホスト／ソースファイルは Canvas plugin 配下に置かれる
- Canvas ドキュメントの実体化処理は Canvas plugin 配下に置かれる
- CLI コマンドの実装は Canvas plugin 配下に置かれるか、plugin が所有するランタイム barrel を介して委譲される
- ドキュメントと plugin インベントリでは、Canvas を実験的かつ plugin ベースの機能として説明する

## 対象外

- このリファクタリングでは、ネイティブアプリの Canvas UI を再設計しない。
- Canvas を削除するという別途のプロダクト判断がない限り、iOS、Android、macOS から Canvas のプロトコル／クライアントサポートを削除しない。
- 少なくとももう 1 つのバンドル plugin が同じ接続面を必要としない限り、Canvas だけのために広範な plugin サービスフレームワークを構築しない。

## 現在のブランチの状態

完了済み：

- `extensions/canvas` にバンドル plugin パッケージを追加。
- `extensions/canvas/openclaw.plugin.json` を追加。
- エージェントの `canvas` ツールを `src/agents/tools/canvas-tool.ts` から `extensions/canvas/src/tool.ts` に移動。
- `src/agents/openclaw-tools.ts` から `createCanvasTool` のコア登録を削除。
- Canvas ホストの実装を `src/canvas-host` から `extensions/canvas/src/host` に移動。
- `extensions/canvas/runtime-api.ts` は、テスト、パッケージング、外部公開される Canvas ヘルパー向けに、plugin が所有する互換性 barrel として維持。
- Canvas ドキュメントの実体化処理を `src/gateway/canvas-documents.ts` から `extensions/canvas/src/documents.ts` に移動。
- Canvas CLI の実装と A2UI JSONL ヘルパーを `extensions/canvas/src/cli.ts` に移動。
- Canvas ホスト URL とスコープ付きケイパビリティのヘルパーを `extensions/canvas/src` に移動。
- Canvas Node コマンドのデフォルトを、ハードコードされたコアのリストから plugin の `nodeInvokePolicies` に移動。
- `plugins.entries.canvas.config.host` に plugin 所有の Canvas ホスト設定を追加。
- Canvas と A2UI の HTTP 配信を、Canvas plugin の HTTP ルート登録の背後に移動。
- plugin 所有の HTTP ルート向けに、汎用的な plugin WebSocket アップグレードディスパッチを追加。
- Canvas 固有の Gateway ホスト URL と Node ケイパビリティ認証を、汎用的なホスト型 plugin サーフェスおよび Node ケイパビリティヘルパーに置換。
- plugin 所有のホスト型メディアリゾルバーを追加し、コアが Canvas ドキュメントの内部実装をインポートする代わりに、Canvas ドキュメントの URL が Canvas plugin を介して解決されるように変更。
- `api.registerNodeCliFeature(...)` を追加し、Canvas が親コマンドのパスを手動で記述せずに、`openclaw nodes canvas` を plugin 所有の Node 機能として宣言できるように変更。
- `extensions/canvas/runtime-api.js` の本番用 `src/**` インポートを削除。
- A2UI バンドルのソースを `apps/shared/OpenClawKit/Tools/CanvasA2UI` から `extensions/canvas/src/host/a2ui-app` に移動。
- A2UI のビルド／コピー実装を `extensions/canvas/scripts` 配下に移動し、ルートのビルド配線を汎用的なバンドル plugin アセットフックに置換。
- ランタイムのレガシーなトップレベル `canvasHost` 設定エイリアスを削除。
- Canvas の doctor マイグレーションを維持し、`openclaw doctor --fix` が古い `canvasHost` 設定を `plugins.entries.canvas.config.host` に書き換えるようにした。
- Gateway プロトコル v4 より前の旧エージェント向け Canvas プロトコル互換性を削除。現在、ネイティブクライアントと Gateway は `pluginSurfaceUrls.canvas` と `node.pluginSurface.refresh` のみを使用する。非推奨の `canvasHostUrl`、`canvasCapability`、`node.canvas.capability.refresh` の経路は、この実験的なリファクタリングでは意図的にサポートしない。
- 生成される plugin インベントリを更新し、Canvas を追加。
- `docs/plugins/reference/canvas.md` に plugin リファレンスドキュメントを追加。

コアが所有する既知の残存 Canvas サーフェス：

- `apps/` 配下のネイティブアプリ Canvas ハンドラーは、引き続き意図的に Canvas plugin サーフェスを利用する
- `apps/` 配下のネイティブアプリ Canvas プロトコル／クライアントハンドラー
- 公開アーティファクトの出力では、後方互換性のあるランタイム検索のために引き続き `dist/canvas-host/a2ui` を使用するが、コピー処理は現在 plugin が所有している

## 目標とする形

`extensions/canvas` が所有するもの：

- plugin マニフェストとパッケージメタデータ
- エージェントツールの登録
- Node invoke コマンドポリシー
- Canvas ホストと A2UI ランタイム
- Canvas A2UI バンドルのソースとアセットのビルド／コピースクリプト
- Canvas ドキュメントの作成とアセット解決
- Canvas CLI の実装
- Canvas ドキュメントページと plugin インベントリのエントリ

コアが所有するのは汎用的な接続面のみとする：

- plugin の検出と登録
- 汎用的なエージェントツールレジストリ
- 汎用的な Node invoke ポリシーレジストリ
- 汎用的な Gateway HTTP／認証と WebSocket アップグレードディスパッチ
- 汎用的なホスト型 plugin サーフェス URL の解決
- 汎用的なホスト型メディアリゾルバーの登録
- 汎用的な Node ケイパビリティ転送
- 汎用的な設定基盤
- 汎用的なバンドル plugin アセットフックの検出

ネイティブアプリは、プロトコルのクライアントとして Canvas コマンドハンドラーを維持できます。ネイティブアプリは plugin ランタイムの所有者ではありません。

## 移行手順

1. `plugins.entries.canvas.config.host` を plugin 所有の設定サーフェスとして扱う。
2. Canvas を実験的なバンドル plugin として説明するようにドキュメントを更新する。
3. 対象を絞った Canvas テスト、plugin インベントリチェック、plugin SDK API チェック、およびランタイム境界の影響を受けるビルド／型ゲートを実行する。

## 監査チェックリスト

リファクタリングの完了を宣言する前に：

- `rg "src/canvas-host|../canvas-host"` が実際に使用されるソースインポートを返さない。
- `rg "canvas-tool|createCanvasTool" src` で、コア所有の Canvas ツール実装が見つからない。
- `rg "canvas.present|canvas.snapshot|canvas.a2ui" src/gateway` で、汎用的な plugin ポリシーテスト以外にハードコードされた許可リストのデフォルトが見つからない。
- `rg "extensions/canvas/runtime-api" src --glob '!**/*.test.ts'` が空である。
- `rg "canvas-documents" src` が空である。
- `rg "registerNodesCanvasCommands|nodes-canvas" src` が空である。Canvas plugin は、ネストされた plugin CLI メタデータを介して `openclaw nodes canvas` を登録する。
- `rg "createCanvasHostHandler|handleA2uiHttpRequest" src/gateway` が Gateway ランタイムの所有を返さない。
- `rg "apps/shared/OpenClawKit/Tools/CanvasA2UI|canvas-a2ui-copy|extensions/canvas/src/host/a2ui" scripts .github package.json` では、互換性ラッパーまたは plugin 所有のパスのみが見つかる。
- `pnpm plugins:inventory:check` が成功する。
- `pnpm plugin-sdk:api:check` が成功するか、生成された API 契約レコードが意図的に更新され、レビューされている。
- 対象を絞った Canvas テストが成功する。
- Canvas ホスト／A2UI パスの変更レーンテストが成功する。
- PR 本文に、Canvas が実験的かつ plugin ベースであることが明記されている。

## 検証コマンド

反復作業中は、対象を絞ったローカルチェックを使用します：

```sh
pnpm test extensions/canvas/src/host/server.test.ts extensions/canvas/src/host/server.state-dir.test.ts extensions/canvas/src/host/file-resolver.test.ts
pnpm test src/gateway/server.plugin-node-capability-auth.test.ts src/gateway/server-import-boundary.test.ts
pnpm test extensions/canvas/src/config-migration.test.ts src/commands/doctor-legacy-config.migrations.test.ts
pnpm test test/scripts/changed-lanes.test.ts test/scripts/build-all.test.ts extensions/canvas/scripts/bundle-a2ui.test.ts test/scripts/bundled-plugin-assets.test.ts extensions/canvas/scripts/copy-a2ui.test.ts src/infra/run-node.test.ts
pnpm tsgo:extensions
pnpm plugins:inventory:check
pnpm plugin-sdk:api:check
```

ランタイム barrel、遅延インポート、パッケージング、または公開される plugin サーフェスが変更された場合は、push 前に `pnpm build` を実行します。
