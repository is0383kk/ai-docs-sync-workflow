---
read_when:
    - 你需要查找之前会话中讨论过的内容
    - 你想了解会话搜索的隐私或索引机制
summary: 搜索过去的会话记录并重新打开匹配的上下文
title: 会话搜索
x-i18n:
    generated_at: "2026-07-26T06:12:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3e9cda6b656b689eef0636592914f4890a64dca5e955aa03908377903aaa29c9
    source_path: concepts/session-search.md
    workflow: 16
---

# 会话搜索

`sessions_search` 会搜索你自己的历史会话中的用户和助手文本。每条结果都包含 `sessionKey`、时间戳、角色和简短的匹配摘录。需要查看上下文对话时，请将返回的 `sessionKey` 传给 `sessions_history`。

## 可见性和输出

搜索采用与 `sessions_history` 相同的会话可见性规则。在应用结果数量限制之前，会移除调用方可见会话树之外的结果。启用已创建会话可见性后，沙箱隔离的智能体仍只能访问由其创建的会话。

摘录会在返回模型前进行脱敏。结果还受数量、摘录长度和响应总大小的限制。

## 索引生命周期

OpenClaw 在每个智能体的 SQLite 数据库中，将全文索引存储在对话记录行旁。新的用户和助手消息会在持久化消息的同一事务中建立索引，因此索引绝不会落后于实时对话；工具结果、推理块和图像不会纳入索引。只有对话记录的活跃分支可供搜索。

早于索引创建时间的对话记录（例如由 `openclaw doctor` 导入的会话）以及活跃分支已回退的会话，会通过从下一次搜索开始的后台协调过程重新建立索引。因此，包含 `indexing: true` 的响应可能不完整；请在索引建立完成后重试。删除会话时，会在同一事务中移除其索引条目。

搜索目前使用 SQLite 的 Unicode 单词分词器，并移除变音符号。使用三元组分词改进 CJK 子字符串匹配是未来的优化方向。

## 会话搜索与记忆搜索

如需在原始会话对话记录中搜索确切的单词或短语，请使用 `sessions_search`。如需搜索持久记忆文件和进行语义回忆，请使用 [`memory_search`](/zh-CN/concepts/memory-search)。实验性的会话记忆语料库是这种精确对话记录搜索的语义补充。
