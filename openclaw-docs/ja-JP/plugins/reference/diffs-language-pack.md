---
read_when:
    - diffs-language-pack Plugin をインストール、設定、または監査しています
summary: デフォルトの差分ビューアーに含まれない言語のシンタックスハイライトを追加します。
title: Diffs 言語パック Plugin
x-i18n:
    generated_at: "2026-07-26T10:24:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e63f896b937be27bd00a7a728b128ec0d1d5eee91d6f1023862274e32afe5db1
    source_path: plugins/reference/diffs-language-pack.md
    workflow: 16
---

# Diffs 言語パック Plugin

デフォルトの Diffs ビューアーに含まれない言語の構文ハイライトを追加します。

## 配布

- パッケージ: `@openclaw/diffs-language-pack`
- インストール経路: npm、ClawHub: `clawhub:@openclaw/diffs-language-pack`

## 対象

Plugin

<!-- openclaw-plugin-reference:manual-start -->

## 追加される言語

基本の `diffs` Plugin は、[Diffs](/ja-JP/tools/diffs) に記載されている一般的な言語をすでにハイライトします。Shiki がサポートするより幅広い言語の構文ハイライトが必要な場合は、この言語パックをインストールしてください。このパックがインストールされていなくても、それらのファイルは読みやすいプレーンテキストとして表示されます。

例として、Astro、Vue、Svelte、MDX、GraphQL、Terraform/HCL、Nix、Clojure、Elixir、Haskell、OCaml、Scala、Zig、Solidity、Verilog/VHDL、Fortran、MATLAB、LaTeX、Mermaid、Sass/Less/SCSS、Nginx、Apache、CSV、dotenv、INI、diff ファイルなどがあります。

Shiki のアップストリームの言語およびエイリアスのカタログについては、[Shiki の言語](https://shiki.style/languages)を参照してください。

<!-- openclaw-plugin-reference:manual-end -->
