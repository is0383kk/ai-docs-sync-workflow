---
read_when:
    - 別のアプリケーションで OpenClaw のモデル転送機能を再利用したい場合
    - packages/ai または AI トランスポートのホストポートを変更している場合
    - OpenClaw のリリースでルートパッケージ以外に npm へ公開されるものをレビューしています
summary: '@openclaw/ai npm パッケージ：再利用可能なモデル転送、分離されたランタイム、ホストポリシーポート'
title: '@openclaw/ai パッケージ'
x-i18n:
    generated_at: "2026-07-26T10:00:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 610057caae0a9bbf9f74074cda75fc40c0b9aa9d3441f8263151f08f1a3f35a8
    source_path: reference/openclaw-ai.md
    workflow: 16
---

`@openclaw/ai` は、OpenClaw のモデル実行レイヤーを公開可能なライブラリにしたものです。プロバイダーに依存しないメッセージ、ツール、ストリームのコントラクト、検証、診断、イベントストリーム、分離されたランタイムレジストリ、および組み込みの 8 つの API ファミリー（Anthropic Messages、OpenAI Completions、OpenAI Responses、Azure OpenAI Responses、ChatGPT/Codex Responses、Google Generative AI、Google Vertex、Mistral Conversations）用の遅延読み込みアダプターを提供します。

これはリリースごとにルートの `openclaw` パッケージとともに公開され、同じバージョンに固定されます。また、独自の `npm-shrinkwrap.json` を備えているため、推移的依存関係ツリーはインストール時に固定されます。`openclaw` をインストールすると、対応する `@openclaw/ai` が自動的にインストールされます。ライブラリ利用者は、OpenClaw のアプリケーションコードを一切使用せずに、このパッケージへ直接依存できます。

## クイックスタート

```js
import { createLlmRuntime } from "@openclaw/ai";
import { registerBuiltInApiProviders } from "@openclaw/ai/providers";

const runtime = createLlmRuntime();
registerBuiltInApiProviders(runtime.registry);

const stream = runtime.streamSimple(model, { messages }, { apiKey });
for await (const event of stream) {
  if (event.type === "text_delta") process.stdout.write(event.delta);
}
const result = await stream.result();
```

実行可能なバージョンは、リポジトリ内の `examples/ai-chat` にあります。

## 設計コントラクト

- **デフォルトではインスタンス単位。** パッケージをインポートしても、グローバルには何も登録されません。`createApiRegistry()` / `createLlmRuntime()` は分離されたインスタンスを返し、`registerBuiltInApiProviders(registry)` は 1 つのレジストリで組み込みトランスポートを有効にします。プロバイダー SDK モジュールは、初回使用時に遅延読み込みされます。
- **ホストポリシーは組み込まれず、注入されます。** リクエストの fetch ガード（SSRF ポリシーなど）、ツール結果を再生するテキストからのシークレットの秘匿化、OpenAI の厳格なツールのデフォルト、および診断ログは、`configureAiTransportHost` で設定する `AiTransportHost` ポートです。ライブラリのデフォルトは何も実行しません。OpenClaw は、ストリームファサードに実際の実装を導入します。
- **単一のイベントストリーム ID。** `@openclaw/ai/event-stream` は、OpenClaw コア、agent-core、および外部利用者が共有する正規の `EventStream` コンストラクターです。
- **`internal/*` サブパスは API ではありません。** これらは OpenClaw アプリケーション自体のために存在し、semver の保証はありません。
- プロバイダー ID、認証情報、モデルカタログ、再試行、およびフェイルオーバーは、引き続きアプリケーション側の責務です。OpenClaw はこのパッケージの周囲にそれらを重ねます。ライブラリ利用者は、`Model` オブジェクトとオプションを直接指定します。

## サブパスのエクスポート

| サブパス          | 内容                                                                       |
| ---------------- | ------------------------------------------------------------------------------ |
| `.`              | コントラクト、`createApiRegistry`、`createLlmRuntime`、`configureAiTransportHost` |
| `./providers`    | `registerBuiltInApiProviders`、`resetApiProviders`                             |
| `./types`        | モデル、メッセージ、ツール、ストリームの型                                                |
| `./validation`   | ツール引数の検証                                                       |
| `./diagnostics`  | 診断コントラクト                                                          |
| `./event-stream` | 共有の `EventStream` 実装                                            |
| `./internal/*`   | OpenClaw 内部用、semver の保証なし                                         |
