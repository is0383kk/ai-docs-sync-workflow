---
read_when:
    - OpenClaw リリースにおける npm shrinkwrap の意味を知りたい場合
    - パッケージのロックファイル、依存関係の変更、またはサプライチェーンのリスクをレビューしている場合
    - 公開前にルートまたはプラグインの npm パッケージを検証しています
summary: OpenClaw リリースにおける npm shrinkwrap の平易な英語による技術解説
title: npm shrinkwrap
x-i18n:
    generated_at: "2026-07-26T09:43:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d1e6c0d4541da9220d50cde0b9db064e5a91b81d6562cb16ac697de7d4017098
    source_path: gateway/security/shrinkwrap.md
    workflow: 16
---

OpenClaw のソースチェックアウトでは `pnpm-lock.yaml` を使用します。公開される OpenClaw npm パッケージでは、npm の公開可能な依存関係ロックファイルである `npm-shrinkwrap.json` を使用するため、パッケージのインストールにはリリース時にレビューされた依存関係グラフが使用されます。

## 重要な理由

Shrinkwrap は、npm パッケージとともに配布される依存関係ツリーの記録です。インストールする推移的依存関係の正確なバージョンを npm に指示します。

| ファイル                  | 重要となる場所         | 意味                     |
| --------------------- | ------------------------ | --------------------------------- |
| `pnpm-lock.yaml`      | OpenClaw のソースチェックアウト | メンテナー向けの依存関係グラフ       |
| `npm-shrinkwrap.json` | 公開される npm パッケージ    | ユーザー向けの npm インストールグラフ       |
| `package-lock.json`   | ローカルの npm アプリ           | OpenClaw の公開契約ではない |

OpenClaw のリリースでは、これは次を意味します。

- 公開パッケージは、インストール時に新しい依存関係グラフを生成するよう npm に要求しません。
- 依存関係の変更はロックファイルの差分に反映されるため、レビューできます。
- リリース検証では、ユーザーがインストールするものと同じグラフをテストします。
- パッケージサイズやネイティブ依存関係に関する予期しない問題が、公開前に明らかになります。

Shrinkwrap はサンドボックスではありません。それ自体で依存関係を安全にするものではなく、ホストの分離、`openclaw security audit`、パッケージの来歴、インストールのスモークテストに代わるものでもありません。

OpenClaw は Gateway、Plugin ホスト、モデルルーター、エージェントランタイムであるため、デフォルトのインストールは起動時間、ディスク使用量、ネイティブパッケージのダウンロード、サプライチェーンへの露出に影響します。Shrinkwrap はリリースレビューに安定した境界を提供します。レビュー担当者は推移的依存関係の変動を確認でき、検証ツールは予期しないロックファイルのずれを拒否し、Plugin パッケージはルートパッケージに依存せず、独自にロックされた依存関係グラフを保持します。

## 生成と確認

ルートの `openclaw` npm パッケージ、OpenClaw が所有する npm Plugin パッケージ（例: `@openclaw/discord`）、および [`@openclaw/ai`](/ja-JP/reference/openclaw-ai) などの公開可能なワークスペースパッケージには、公開時に `npm-shrinkwrap.json` が含まれます。ワークスペース依存関係はルートパッケージとともに公開されるため、ルートの shrinkwrap から除外されます。代わりに、公開可能な各ワークスペースパッケージが独自の推移的依存関係ツリーを固定します。適切な Plugin パッケージでは、明示的な `bundledDependencies` を使用して公開することもでき、インストール時の解決だけに依存せず、Plugin の tarball にランタイム依存関係ファイルを含められます。

```bash
# shrinkwrap で管理されるすべてのパッケージ（ルート + 公開可能な Plugin）
pnpm deps:shrinkwrap:generate
pnpm deps:shrinkwrap:check

# ルートパッケージのみ
pnpm deps:shrinkwrap:root:generate
pnpm deps:shrinkwrap:root:check

# 現在の変更セットの影響を受けるパッケージのみ
pnpm deps:shrinkwrap:changed:generate
pnpm deps:shrinkwrap:changed:check
```

ジェネレーターは npm の公開可能なロック形式を解決しますが、`pnpm-lock.yaml` にまだ存在しない生成済みパッケージバージョンを拒否します。これにより、pnpm の依存関係の経過期間、オーバーライド、およびパッチレビューの境界が維持されます。

以下はセキュリティ上重要なものとしてレビューしてください。

- `pnpm-lock.yaml`
- `npm-shrinkwrap.json`
- バンドルされた Plugin の依存関係ペイロード
- `package-lock.json` のすべての差分

OpenClaw のパッケージ検証ツールは、新しいルートパッケージの tarball に shrinkwrap が含まれることを要求し、公開パッケージの `package-lock.json` を拒否します。Plugin の npm 公開パスでは、Plugin ローカルの shrinkwrap を確認し、パッケージローカルのバンドル依存関係をインストールしてから、パッケージ化または公開します。

## 公開パッケージの調査

ルートパッケージ:

```bash
npm pack openclaw@<version> --json --pack-destination /tmp/openclaw-pack
tar -tf /tmp/openclaw-pack/openclaw-<version>.tgz | grep '^package/npm-shrinkwrap.json$'
```

Plugin パッケージ:

```bash
npm pack @openclaw/discord@<version> --json --pack-destination /tmp/openclaw-plugin-pack
tar -tf /tmp/openclaw-plugin-pack/openclaw-discord-<version>.tgz | grep '^package/npm-shrinkwrap.json$'
tar -tf /tmp/openclaw-plugin-pack/openclaw-discord-<version>.tgz | grep '^package/node_modules/'
```

背景情報: [npm-shrinkwrap.json](https://docs.npmjs.com/cli/v11/configuring-npm/npm-shrinkwrap-json)。
