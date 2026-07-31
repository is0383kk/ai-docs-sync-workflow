---
read_when:
    - 複数のファイルにわたる構造化されたファイル編集が必要です
    - パッチベースの編集を文書化またはデバッグしたい場合
summary: apply_patch ツールで複数ファイルにパッチを適用する
title: apply_patch ツール
x-i18n:
    generated_at: "2026-07-26T09:46:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1c0422550ea8d9b0cb6b0ea22d7dcaecc462426f9600003f70c177746f30a3d9
    source_path: tools/apply-patch.md
    workflow: 16
---

構造化パッチ形式を使用してファイルの変更を適用します。単一の `edit` 呼び出しでは壊れやすい、複数ファイルまたは複数ハンクの編集に最適です。

このツールは、1 つ以上のファイル操作をラップした単一の `input` 文字列を受け取ります。

```text
*** Begin Patch
*** Add File: path/to/file.txt
+行 1
+行 2
*** Update File: src/app.ts
@@ 任意の変更コンテキスト
-古い行
+新しい行
*** Delete File: obsolete.txt
*** End Patch
```

## パラメーター

- `input`（必須）: `*** Begin Patch` と `*** End Patch` を含むパッチの全内容。

## 注意事項

- パッチのパスでは、相対パス（ワークスペースディレクトリ基準）と絶対パスを使用できます。
- `tools.exec.applyPatch.workspaceOnly` のデフォルトは `true`（ワークスペース内）です。`apply_patch` でワークスペースディレクトリ外への書き込みや削除を意図的に行う場合にのみ、`false` に設定してください。
- ファイル名を変更するには、`*** Update File:` ハンク内で `*** Move to:` を使用します。
- `*** End of File` は、必要な場合に EOF のみに対する挿入を示します。
- すべてのモデルでデフォルトで有効です。無効にするには `tools.exec.applyPatch.enabled: false`
  を設定します。または、`tools.exec.applyPatch.allowModels` を使用して特定のモデルに限定します
  （`gpt-5.4` のような未加工の ID、または
  `openai/gpt-5.4` のような完全な ID を指定できます）。
- 設定は `tools.exec.applyPatch.*` にあります。

## 例

```json
{
  "tool": "apply_patch",
  "input": "*** Begin Patch\n*** Update File: src/index.ts\n@@\n-const foo = 1\n+const foo = 2\n*** End Patch"
}
```

## 関連項目

<CardGroup cols={2}>
  <Card title="差分" href="/ja-JP/tools/diffs" icon="code-compare">
    変更内容を提示するための読み取り専用差分ビューアー。
  </Card>
  <Card title="実行ツール" href="/ja-JP/tools/exec" icon="terminal">
    エージェントからシェルコマンドを実行します。
  </Card>
  <Card title="コード実行" href="/ja-JP/tools/code-execution" icon="square-code">
    xAI を使用した、サンドボックス化されたリモート Python 分析。
  </Card>
</CardGroup>
