---
read_when:
    - 你想在保留 CLI 安装的同时清除本地状态
    - 你想预演将会移除哪些内容
summary: '`openclaw reset` 的 CLI 参考（重置本地状态/配置）'
title: 重置
x-i18n:
    generated_at: "2026-07-26T06:38:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 54f1d320ee368dae4a4bfb32dea73d19eb35f9f30edd12d9c2580ab7e6a26fa6
    source_path: cli/reset.md
    workflow: 16
---

# `openclaw reset`

重置本地配置/状态（保留已安装的 CLI）。

```bash
openclaw reset
openclaw reset --dry-run
openclaw reset --scope config --yes --non-interactive
openclaw reset --scope config+creds+sessions --yes --non-interactive
openclaw reset --scope full --yes --non-interactive
```

## 选项

- `--scope <scope>`：`config`、`config+creds+sessions` 或 `full`
- `--yes`：跳过确认提示
- `--non-interactive`：禁用提示；需要 `--scope` 和 `--yes`
- `--dry-run`：输出操作，但不删除文件

## 范围

| 范围                    | 删除内容                                                                    | 是否先停止 Gateway 网关 |
| ----------------------- | --------------------------------------------------------------------------- | ----------------------- |
| `config`                | 仅配置文件                                                                  | 否                      |
| `config+creds+sessions` | 配置文件、OAuth/凭据目录、各智能体的会话目录                                 | 是                      |
| `full`                  | 状态目录（包括共享 SQLite 数据库）以及工作区目录                             | 是                      |

`config+creds+sessions` 和 `full` 会在删除状态前停止正在运行的托管 Gateway 网关服务。

## 注意事项

- 删除本地状态前，请先运行 `openclaw backup create`，以创建可恢复的快照。
- 工作区设置状态和证明以行的形式存储在共享 SQLite 数据库中，因此 `full` 会随状态目录一起删除它们；目前没有需要单独删除的证明附属文件。
- 如果未使用 `--scope`，`openclaw reset` 会通过交互式提示要求选择要删除的范围。
- `--non-interactive` 仅在同时设置 `--scope` 和 `--yes` 时有效。
- `config+creds+sessions` 和 `full` 完成后会输出 `Next: openclaw onboard --install-daemon`。

## 相关内容

- [CLI 参考](/zh-CN/cli)
