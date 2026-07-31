---
read_when:
    - 调查提及缺少 __name 辅助函数的 tsx/esbuild 加载器崩溃问题
summary: 历史 Node + tsx “__name is not a function”崩溃及其原因
title: Node + tsx 崩溃
x-i18n:
    generated_at: "2026-07-26T05:47:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 97d2f62d24860cee65753027ba84c14c8d4ffb910ee17bb0032cf0409c427589
    source_path: debug/node-issue.md
    workflow: 16
---

# Node + tsx “\_\_name is not a function” 崩溃

## 状态

已解决。此崩溃在
`package.json` 中锁定的当前 `tsx` 版本（`4.22.3`）或当前 Node 版本上均无法复现。保留此文档，以备未来
`tsx`/esbuild 升级再次引入此问题。

## 原始症状

通过 `tsx` 运行 OpenClaw 开发脚本时，启动失败并显示：

```text
[openclaw] 启动 CLI 失败：TypeError: __name is not a function
    位于 createSubsystemLogger (src/logging/subsystem.ts)
    位于 <caller> (src/agents/auth-profiles/constants.ts)
```

此处省略了行号；自最初发生崩溃以来，这两个文件都已更改，
具体行号已不再匹配。

此问题出现在开发脚本从 Bun 切换到 `tsx`（`2871657e`，
2026-01-06）以使 Bun 成为可选项之后。等效的 Bun 路径未发生崩溃。
最初在 macOS 上的 Node v25.3.0 中观察到此问题；当时认为运行
Node 25 的其他平台也可能受到影响。

## 原因

`tsx` 通过 esbuild 转换 TS/ESM，并在其转换选项中硬编码
`keepNames: true`。该设置会使 esbuild 将具名函数/类声明包装在对
`__name` 辅助函数的调用中，以便 `fn.name` 在压缩和打包后仍得以保留。
此崩溃意味着，在受影响的 `tsx`/Node 组合中，该模块调用点的辅助函数缺失或被遮蔽，因此 `__name(...)`
抛出异常，而没有返回包装后的值。

## 当前复现检查

```bash
node --version
pnpm install
node --import tsx src/entry.ts status
```

最小独立复现（仅加载原始堆栈跟踪中的模块）：

```bash
node --import tsx scripts/repro/tsx-name-repro.ts
```

目前这两个命令都会正常退出。如果其中任一命令再次抛出 `__name is not a
function`，请在向上游提交问题之前，记录确切的 Node 版本、`tsx` 版本
（`node_modules/tsx/package.json`）以及完整的堆栈跟踪。

## 解决方法（如果崩溃再次出现）

- 使用 Bun 而非 `node --import tsx` 运行开发脚本。
- 运行 `pnpm tsgo` 进行类型检查，然后运行构建后的输出，而不是通过
  `tsx` 运行源代码：

  ```bash
  pnpm tsgo
  node openclaw.mjs status
  ```

- 尝试其他 `tsx` 版本（`pnpm add -D tsx@<version>` 属于依赖项
  变更，根据仓库策略需要获得批准），以通过二分排查其内置的 esbuild
  版本是否再次引入了该错误。
- 在不同的 Node 主版本/次版本上测试，以确定该故障是否仅与特定版本有关。

## 参考资料

- [https://esbuild.github.io/api/#keep-names](https://esbuild.github.io/api/#keep-names)
- [https://github.com/evanw/esbuild/issues/1031](https://github.com/evanw/esbuild/issues/1031)

## 相关内容

- [Node.js 安装](/zh-CN/install/node)
- [Gateway 网关故障排除](/zh-CN/gateway/troubleshooting)
