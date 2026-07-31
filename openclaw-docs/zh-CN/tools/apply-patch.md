---
read_when:
    - 你需要对多个文件进行结构化编辑
    - 你想要记录或调试基于补丁的编辑操作
summary: 使用 apply_patch 工具应用多文件补丁
title: apply_patch 工具
x-i18n:
    generated_at: "2026-07-26T06:34:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1c0422550ea8d9b0cb6b0ea22d7dcaecc462426f9600003f70c177746f30a3d9
    source_path: tools/apply-patch.md
    workflow: 16
---

使用结构化补丁格式应用文件更改。这非常适合多文件
或多区块编辑，因为使用单个 `edit` 调用会很脆弱。

该工具接受单个 `input` 字符串，其中封装一个或多个文件操作：

```text
*** Begin Patch
*** Add File: path/to/file.txt
+第 1 行
+第 2 行
*** Update File: src/app.ts
@@ 可选的更改上下文
-旧行
+新行
*** Delete File: obsolete.txt
*** End Patch
```

## 参数

- `input`（必需）：完整的补丁内容，包括 `*** Begin Patch` 和 `*** End Patch`。

## 注意事项

- 补丁路径支持相对路径（相对于工作区目录）和绝对路径。
- `tools.exec.applyPatch.workspaceOnly` 默认为 `true`（限制在工作区内）。仅当你有意让 `apply_patch` 在工作区目录之外写入/删除时，才将其设置为 `false`。
- 在 `*** Update File:` 区块内使用 `*** Move to:` 来重命名文件。
- `*** End of File` 在需要时标记仅在 EOF 处插入。
- 默认对每个模型启用。设置 `tools.exec.applyPatch.enabled: false`
  可将其禁用，或使用 `tools.exec.applyPatch.allowModels` 将其限制为特定模型（接受
  `gpt-5.4` 这样的原始 ID，或 `openai/gpt-5.4`
  这样的完整 ID）。
- 配置位于 `tools.exec.applyPatch.*` 下。

## 示例

```json
{
  "tool": "apply_patch",
  "input": "*** Begin Patch\n*** Update File: src/index.ts\n@@\n-const foo = 1\n+const foo = 2\n*** End Patch"
}
```

## 相关内容

<CardGroup cols={2}>
  <Card title="Diffs" href="/zh-CN/tools/diffs" icon="code-compare">
    用于呈现更改的只读差异查看器。
  </Card>
  <Card title="Exec 工具" href="/zh-CN/tools/exec" icon="terminal">
    由智能体执行 Shell 命令。
  </Card>
  <Card title="代码执行" href="/zh-CN/tools/code-execution" icon="square-code">
    使用 xAI 进行沙箱隔离的远程 Python 分析。
  </Card>
</CardGroup>
