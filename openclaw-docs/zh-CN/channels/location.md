---
read_when:
    - 添加或修改渠道位置解析
    - 在智能体提示词或工具中使用位置上下文字段
summary: 渠道位置解析和可移植的出站位置载荷
title: 渠道位置解析
x-i18n:
    generated_at: "2026-07-26T05:39:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c7e5647d02643ad6d95024b362228377690d7fdff66441fae367f0f5307217fb
    source_path: channels/location.md
    workflow: 16
---

OpenClaw 将聊天渠道共享的位置标准化为：

- 附加到入站正文的简洁坐标文本，以及
- 自动回复上下文负载中的结构化字段。渠道提供的标签、地址和说明文字/评论通过共享的不受信任元数据 JSON 块呈现到提示词中，而不是内联到用户正文中。

目前支持：

- **LINE**（带标题/地址的位置消息）
- **Matrix**（带有 `geo_uri` 的 `m.location`）
- **Telegram**（位置图钉 + 地点 + 实时位置）
- **WhatsApp**（`locationMessage` + `liveLocationMessage`）

## 文本格式

位置会呈现为不带方括号的易读文本行。坐标使用六位小数；精度四舍五入到整数米：

- 图钉：
  - `📍 48.858844, 2.294351 ±12m`
- 命名地点（在同一行；名称/地址仅进入元数据块）：
  - `📍 48.858844, 2.294351 ±12m`
- 实时共享：
  - `🛰 Live location: 48.858844, 2.294351 ±12m`

如果渠道包含标签、地址或说明文字/评论，则会将其保留在上下文负载中，并在提示词里显示为带围栏的不受信任 JSON（字段不存在时会省略）：

````text
位置（不受信任的元数据）：
```json
{
  "latitude": 48.858844,
  "longitude": 2.294351,
  "accuracy_m": 12,
  "source": "place",
  "name": "埃菲尔铁塔",
  "address": "巴黎战神广场",
  "caption": "在这里见面"
}
```
````

## 上下文字段

存在位置时，会将以下字段添加到 `ctx`：

- `LocationLat`（数字）
- `LocationLon`（数字）
- `LocationAccuracy`（数字，单位为米；可选）
- `LocationName`（字符串；可选）
- `LocationAddress`（字符串；可选）
- `LocationSource`（`pin | place | live`）
- `LocationIsLive`（布尔值）
- `LocationCaption`（字符串；可选）

当渠道未设置明确来源时，OpenClaw 会进行推断：实时共享为 `live`，具有名称或地址的位置为 `place`，其他所有位置均为 `pin`。

提示词渲染器将 `LocationName`、`LocationAddress` 和 `LocationCaption` 视为不受信任的元数据，并通过用于其他渠道上下文的同一有界 JSON 路径对其进行序列化。

## 出站负载

消息工具和插件 SDK 对可移植的出站位置使用相同的 `NormalizedLocation` 结构。仅含坐标的负载表示一个图钉。支持原生地点功能的渠道可以将 `name` 和 `address` 映射为地点卡片。

Telegram 目前通过 `message(action="send")` 提供此功能。其首个实现有意保持独立：位置负载不能与文本或媒体混合使用，并且不完整的地点字段对会导致失败，而不是静默丢弃名称或地址。不支持的渠道不会公布位置参数。

## 渠道说明

- **LINE**：位置消息 `title`/`address` 映射到 `LocationName`/`LocationAddress`；不支持实时位置。
- **Matrix**：`geo_uri` 解析为图钉位置；`u`（不确定度）参数映射到 `LocationAccuracy`，事件正文填充 `LocationCaption`，忽略海拔，并且 `LocationIsLive` 始终为 false。
- **Telegram**：地点映射到 `LocationName`/`LocationAddress`；通过 `live_period` 检测实时位置。
- **WhatsApp**：`locationMessage.comment` 和 `liveLocationMessage.caption` 填充 `LocationCaption`。

## 相关内容

- [位置命令（节点）](/zh-CN/nodes/location-command)
- [相机拍摄](/zh-CN/nodes/camera)
- [媒体理解](/zh-CN/nodes/media-understanding)
