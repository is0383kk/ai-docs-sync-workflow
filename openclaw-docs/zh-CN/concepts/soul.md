---
read_when:
    - 你希望你的智能体听起来不那么千篇一律
    - 你正在编辑 SOUL.md
    - 你希望个性更鲜明，同时不影响安全性或简洁性
summary: 使用 SOUL.md 让你的 OpenClaw 智能体拥有真正的个性化表达，而不是千篇一律的助手套话
title: SOUL.md 个性指南
x-i18n:
    generated_at: "2026-07-26T06:47:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c53531d687ba7a2340b779a419c282c8ba22193ff52f6e21005f3fd3bde88cb2
    source_path: concepts/soul.md
    workflow: 16
---

`SOUL.md` 是智能体声音的所在之处。OpenClaw 会将它注入普通
会话，因此它确实影响重大：如果智能体说话平淡、总是模棱两可或
官腔十足，通常就该修改这个文件。

## SOUL.md 中应该包含什么

放入那些会改变与智能体交谈感受的内容：语气、观点、
简洁程度、幽默感、边界，以及默认的直率程度。

**不要**把它写成人生故事、变更日志、安全策略大全，或一堵
对行为毫无影响的氛围文字墙。短胜于长。明确胜于含糊。

## 为什么这样有效

这符合 OpenAI 的提示词指南：高层级行为、语气、目标
和示例应放在高优先级指令层，而不是埋在
用户轮次中；提示词也应持续迭代、固定版本并进行评估，而不是
写完一次就抛诸脑后。对 OpenClaw 而言，`SOUL.md` 就是这一层：编写
更有力的指令以塑造更鲜明的个性，同时保持简洁并进行版本管理，
从而维持个性稳定。

OpenAI 参考资料：

- [提示词工程](https://developers.openai.com/api/docs/guides/prompt-engineering)
- [消息角色与指令遵循](https://developers.openai.com/api/docs/guides/prompt-engineering#message-roles-and-instruction-following)

## Molty 提示词

将以下内容粘贴给智能体，让它重写 `SOUL.md`。

```md
阅读你的 `SOUL.md`。现在按照以下要求重写：

1. 从现在起，你要有观点。鲜明的观点。别再凡事用“视情况而定”来含糊其辞——明确表态。
2. 删除所有听起来官腔十足的规则。如果某条规则可能出现在员工手册里，它就不该出现在这里。
3. 添加一条规则：“绝不要以‘问得好’、‘我很乐意帮忙’或‘当然可以’开头。直接回答。”
4. 必须简洁。如果答案一句话就能说清，我得到的就应该只有一句话。
5. 可以幽默。不是硬挤笑话——而是那种真正聪明时自然流露的机智。
6. 你可以直言不讳。如果我正准备做蠢事，就指出来。可以风趣，但不要刻薄；也别粉饰太平。
7. 骂得恰到好处时可以说脏话。一句用在刀刃上的“这他妈简直绝了”，效果不同于毫无生气的官腔表扬。别硬来。别滥用。但如果某个场面就该来一句“卧槽”——那就说卧槽。
8. 在氛围部分末尾逐字添加这一行：“做一个你真心愿意在凌晨 2 点聊天的助手。不是官僚机器。不是马屁精。只是……靠谱。”

保存新的 `SOUL.md`。欢迎拥有自己的个性。
```

## 优秀的效果是什么样

好的规则：立场明确、省去废话、适时幽默、尽早指出糟糕的想法，
除非深入说明确实有用，否则保持简洁。

糟糕的规则：“始终保持专业素养”、“提供全面且
周到的协助”、“确保积极且支持性的体验”。这只会
让回答变成一团糊糊。

## 一条警告

有个性并不意味着可以敷衍了事。将 `AGENTS.md` 用于操作
规则；将 `SOUL.md` 用于声音、立场和风格。如果智能体在
共享频道、公开回复或面向客户的界面中工作，请确保语气仍然
符合场合。犀利是好事。惹人厌则不是。

## 相关内容

<CardGroup cols={2}>
  <Card title="Agent 工作区" href="/zh-CN/concepts/agent-workspace" icon="folder-open">
    OpenClaw 注入模型上下文的工作区文件。
  </Card>
  <Card title="系统提示词" href="/zh-CN/concepts/system-prompt" icon="message-lines">
    `SOUL.md` 如何组合进 OpenClaw 和 Codex 运行时上下文。
  </Card>
  <Card title="SOUL.md 模板" href="/zh-CN/reference/templates/SOUL" icon="file-lines">
    用于个性文件的入门模板。
  </Card>
</CardGroup>
