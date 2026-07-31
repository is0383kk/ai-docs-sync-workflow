---
read_when:
    - 正在开发遥测/隐私控制功能
    - 关于收集哪些数据的问题
summary: ClawHub CLI 收集的安装遥测数据以及如何选择退出。
x-i18n:
    generated_at: "2026-07-26T06:10:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a02bb1c76fea3105255235f6314ade73f260f692d6eb1b41f8001dc84db6ded7
    source_path: clawhub/telemetry.md
    workflow: 16
---

# 遥测

ClawHub 使用最低限度的 CLI 遥测数据来计算 Skills 和插件的汇总安装数量。

## 何时收集遥测数据

仅在以下情况下发送遥测数据：

- 你已登录 CLI。
- 你运行 `clawhub install <slug>`，或完成一次经过身份验证的
  `openclaw plugins install clawhub:<package>` 安装。
- 遥测功能**未被禁用**（请参阅下方的“如何禁用”）。

如果你未登录，则不会报告任何信息。

## 我们收集的内容

Skills 或插件安装完成并将其本地安装记录持久化后，CLI 会尽力发送一次安装事件。

该事件包括：

- 已安装 Skills 的 slug 或插件的规范包名称。
- `version`：已安装的版本（如果已知）。

### 我们_不会_收集的内容

- 不收集文件夹路径或由文件夹派生的标识符。
- 不收集文件内容。
- 不收集每次运行的日志、提示词或其他 CLI 输出。

## 安装数量

对于 Skills，ClawHub 会维护：

- `installsAllTime`：已报告至少安装过一次该 Skills 的 CLI 唯一用户数。
- `installsCurrent`：已报告安装且尚未删除其
  遥测数据的唯一用户数。

对于插件，ClawHub 会统计每个用户和软件包首次报告的成功安装。重复安装和更新会刷新记录的版本，但不会增加汇总安装数量。

## 透明度与用户控制

所有人只能看到**汇总安装计数器**。

删除你的账户也会删除你的遥测数据，并从安装计数器中移除其贡献。

## 如何禁用遥测功能

设置环境变量：

```bash
export CLAWHUB_DISABLE_TELEMETRY=1
```

设置后，CLI 将不会发送安装遥测数据。
