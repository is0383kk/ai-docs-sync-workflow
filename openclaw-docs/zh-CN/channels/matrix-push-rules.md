---
read_when:
    - 为自托管的 Synapse 或 Tuwunel 设置 Matrix 静默流式传输
    - 用户只希望在分块完成时收到通知，而不是每次编辑预览时都收到通知
summary: 针对安静完成的预览编辑，按收件人配置 Matrix 推送规则
title: Matrix 静默预览的推送规则
x-i18n:
    generated_at: "2026-07-26T06:08:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1c58e7e796c3ae6d1ee25de229e4592ab8b4fb4d0d50a9cf868ab5ef35b1dab5
    source_path: channels/matrix-push-rules.md
    workflow: 16
---

当 `channels.matrix.streaming.mode` 为 `"quiet"` 时，OpenClaw 会通过原位编辑单个预览事件来流式传输回复。预览以不发送通知的 `m.notice` 事件形式发送，最终编辑则使用 `content["com.openclaw.finalized_preview"] = true` 标记。仅当每用户推送规则与该标记匹配时，Matrix 客户端才会针对该最终编辑发送通知。本页面面向自行托管 Matrix，并希望为每个接收者账户安装该规则的运维人员。

`streaming.mode: "progress"` 通过同一路径完成其草稿，因此同一规则也会针对进度模式的最终编辑触发。

如果只需要 Matrix 的标准通知行为，请使用 `streaming.mode: "partial"` 或关闭流式传输。请参阅 [Matrix 频道设置](/zh-CN/channels/matrix#streaming-previews)。

## 前提条件

- 接收者用户 = 应接收通知的人
- 机器人用户 = 发送回复的 OpenClaw Matrix 账户
- 在下方 API 调用中使用接收者用户的访问令牌
- 在推送规则中，将 `sender` 与机器人用户的完整 MXID 进行匹配
- 接收者账户必须已配置正常工作的推送器；只有在 Matrix 常规推送投递正常时，静默预览规则才能工作

## 步骤

<Steps>
  <Step title="配置静默预览">

```json5
{
  channels: {
    matrix: {
      streaming: { mode: "quiet" },
    },
  },
}
```

  </Step>

  <Step title="获取接收者的访问令牌">
    尽可能复用现有的客户端会话令牌。要生成新令牌：

```bash
curl -sS -X POST \
  "https://matrix.example.org/_matrix/client/v3/login" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "m.login.password",
    "identifier": { "type": "m.id.user", "user": "@alice:example.org" },
    "password": "REDACTED"
  }'
```

  </Step>

  <Step title="验证推送器是否存在">

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushers"
```

如果未返回任何推送器，请先修复此账户的 Matrix 常规推送投递，然后再继续。

  </Step>

  <Step title="安装覆盖推送规则">
    安装一条同时匹配最终预览标记和作为发送者的机器人 MXID 的规则：

```bash
curl -sS -X PUT \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname" \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "conditions": [
      { "kind": "event_match", "key": "type", "pattern": "m.room.message" },
      {
        "kind": "event_property_is",
        "key": "content.m\\.relates_to.rel_type",
        "value": "m.replace"
      },
      {
        "kind": "event_property_is",
        "key": "content.com\\.openclaw\\.finalized_preview",
        "value": true
      },
      { "kind": "event_match", "key": "sender", "pattern": "@bot:example.org" }
    ],
    "actions": [
      "notify",
      { "set_tweak": "sound", "value": "default" },
      { "set_tweak": "highlight", "value": false }
    ]
  }'
```

    运行前请替换：

    - `https://matrix.example.org`：你的主服务器基础 URL
    - `$USER_ACCESS_TOKEN`：接收者用户的访问令牌
    - `openclaw-finalized-preview-botname`：每个接收者对应的每个机器人均应使用唯一的规则 ID（格式：`openclaw-finalized-preview-<botname>`）
    - `@bot:example.org`：你的 OpenClaw 机器人 MXID，而非接收者的 MXID

  </Step>

  <Step title="验证">

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

然后测试流式回复。在静默模式下，房间会显示静默草稿预览，并在分块或轮次完成时发送一次通知。

  </Step>
</Steps>

若要稍后移除该规则，请使用接收者的令牌对同一规则 URL 执行 `DELETE`。

## 多机器人说明

推送规则以 `ruleId` 为键：使用同一 ID 重新执行 `PUT` 会更新单条规则。如果多个 OpenClaw 机器人要向同一接收者发送通知，请为每个机器人创建一条规则，并使用不同的发送者匹配条件。

新建的用户自定义 `override` 规则会插入到服务器默认的抑制规则之前，因此无需额外的排序参数。该规则仅影响可原位完成的纯文本预览编辑；媒体回复、过期预览的回退，以及会触发 Matrix 提及的最终文本，仍会作为普通的通知消息投递。

## 主服务器说明

<AccordionGroup>
  <Accordion title="Synapse">
    无需特殊更改 `homeserver.yaml`。如果 Matrix 常规通知已能送达该用户，主要设置步骤就是使用接收者令牌执行上述 `pushrules` 调用。

    如果 Synapse 运行在反向代理或工作进程之后，请确保 `/_matrix/client/.../pushrules/` 能够正确到达 Synapse。推送投递由主进程或 `synapse.app.pusher`／已配置的推送器工作进程处理，请确保这些进程运行正常。

    该规则使用 `event_property_is` 推送规则条件（MSC3758，推送规则 v1.10），Synapse 于 2023 年添加了该条件。较旧的 Synapse 版本会接受 `PUT pushrules/...` 调用，但不会提示任何错误，也始终无法匹配该条件。如果最终预览编辑没有触发通知，请升级 Synapse。

  </Accordion>

  <Accordion title="Tuwunel">
    流程与 Synapse 相同；最终预览标记不需要任何 Tuwunel 专用配置。

    如果用户在另一台设备上处于活跃状态时通知消失，请检查是否启用了 `suppress_push_when_active`。Tuwunel 在 1.4.2（2025 年 9 月）中添加了此选项；当一台设备处于活跃状态时，它可以有意抑制向其他设备发送推送。

  </Accordion>
</AccordionGroup>

## 相关内容

- [Matrix 频道设置](/zh-CN/channels/matrix)
- [流式传输概念](/zh-CN/concepts/streaming)
