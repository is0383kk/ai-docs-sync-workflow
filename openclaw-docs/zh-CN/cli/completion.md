---
read_when:
    - 你希望为 zsh/bash/fish/PowerShell 启用 shell 补全功能
    - 你需要将补全脚本缓存在 OpenClaw 状态目录下
summary: '`openclaw completion` 的 CLI 参考（生成/安装 shell 补全脚本）'
title: 完成
x-i18n:
    generated_at: "2026-07-26T06:10:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 67cb52a47036745150887c752d18e2dfa84fab2722c27c696142d23080bb2efd
    source_path: cli/completion.md
    workflow: 16
---

# `openclaw completion`

生成 shell 补全脚本，将其缓存到 OpenClaw 状态目录中，并可选择将其安装到 shell 配置文件中。

## 用法

```bash
openclaw completion                          # 将 zsh 脚本输出到 stdout
openclaw completion --shell fish             # 输出 fish 脚本
openclaw completion --write-state            # 缓存所有 shell 的脚本
openclaw completion --write-state --install  # 缓存后一步完成安装
openclaw completion --shell bash --write-state
```

## 选项

- `-s, --shell <shell>`：目标 shell（`zsh`、`bash`、`powershell`、`fish`；默认值：`zsh`）
- `-i, --install`：通过在 shell 配置文件中添加缓存脚本的 source 行来安装补全
- `--write-state`：将补全脚本写入 `$OPENCLAW_STATE_DIR/completions`（默认 `~/.openclaw/completions`），而不输出到 stdout；与 `--shell` 一起使用时仅写入该 shell 的脚本，否则写入全部四种 shell 的脚本
- `-y, --yes`：跳过安装确认提示（非交互式）

## 安装流程

`--install` 会使配置文件指向缓存脚本，因此缓存必须已存在：如果缓存不存在，命令会失败并提示你运行 `openclaw completion --write-state`。结合使用 `--write-state --install` 可在一个步骤中完成这两项操作。如果未指定 `--shell`，`--install` 会根据 `$SHELL` 检测 shell（检测失败时回退到 zsh）。

安装过程会在 shell 配置文件中写入一个简短的 `# OpenClaw Completion` 块，并将所有旧的、速度较慢的 `source <(openclaw completion ...)` 行替换为缓存脚本的 source 行：

| Shell      | 配置文件                                                                                                                                                                                    |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| bash       | `~/.bashrc`（缺少 `~/.bashrc` 时回退到 `~/.bash_profile`）                                                                                                                  |
| fish       | `~/.config/fish/config.fish`                                                                                                                                                               |
| powershell | `~/.config/powershell/Microsoft.PowerShell_profile.ps1`（在 Windows 上：`Documents/PowerShell/Microsoft.PowerShell_profile.ps1`；Windows PowerShell 则为 `Documents/WindowsPowerShell/...`） |
| zsh        | `~/.zshrc`                                                                                                                                                                                 |

## 注意事项

- 未指定 `--install` 或 `--write-state` 时，命令会将脚本输出到 stdout。
- 生成补全时会预先加载完整的命令树，包括插件 CLI 命令，因此也会包含嵌套子命令。
- `openclaw update` 会在成功更新后自动刷新补全缓存；`openclaw doctor` 可以修复缺失或过期的补全设置。

## 相关内容

- [CLI 参考](/zh-CN/cli)
