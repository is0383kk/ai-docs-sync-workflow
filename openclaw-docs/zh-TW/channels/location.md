---
read_when:
    - 新增或修改頻道位置解析
    - 在代理提示詞或工具中使用位置情境欄位
summary: 頻道位置資訊解析與可攜式外送位置資訊承載內容
title: 頻道位置解析
x-i18n:
    generated_at: "2026-07-26T07:08:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c7e5647d02643ad6d95024b362228377690d7fdff66441fae367f0f5307217fb
    source_path: channels/location.md
    workflow: 16
---

OpenClaw 會將聊天頻道中的共用位置正規化為：

- 附加至傳入本文的簡潔座標文字，以及
- 自動回覆情境內容承載中的結構化欄位。頻道提供的標籤、地址與說明文字／留言，會透過共用的不受信任中繼資料 JSON 區塊呈現在提示詞中，而不是直接內嵌於使用者本文。

目前支援：

- **LINE**（包含標題／地址的位置訊息）
- **Matrix**（`m.location` 搭配 `geo_uri`）
- **Telegram**（位置圖釘＋地點＋即時位置）
- **WhatsApp**（`locationMessage`＋`liveLocationMessage`）

## 文字格式

位置會呈現為不含括號、易於閱讀的文字行。座標使用六位小數；精確度則四捨五入至整數公尺：

- 圖釘：
  - `📍 48.858844, 2.294351 ±12m`
- 具名地點（同一行；名稱／地址只會放入中繼資料區塊）：
  - `📍 48.858844, 2.294351 ±12m`
- 即時分享：
  - `🛰 Live location: 48.858844, 2.294351 ±12m`

如果頻道包含標籤、地址或說明文字／留言，系統會將其保留在情境內容承載中，並以設有圍欄的不受信任 JSON 形式出現在提示詞內（欄位不存在時會省略）：

````text
位置（不受信任的中繼資料）：
```json
{
  "latitude": 48.858844,
  "longitude": 2.294351,
  "accuracy_m": 12,
  "source": "place",
  "name": "艾菲爾鐵塔",
  "address": "巴黎戰神廣場",
  "caption": "在這裡碰面"
}
```
````

## 情境欄位

存在位置資訊時，會將下列欄位加入 `ctx`：

- `LocationLat`（數字）
- `LocationLon`（數字）
- `LocationAccuracy`（數字，公尺；選填）
- `LocationName`（字串；選填）
- `LocationAddress`（字串；選填）
- `LocationSource`（`pin | place | live`）
- `LocationIsLive`（布林值）
- `LocationCaption`（字串；選填）

當頻道未設定明確的來源時，OpenClaw 會進行推斷：即時分享會成為 `live`，具有名稱或地址的位置會成為 `place`，其他所有位置則為 `pin`。

提示詞轉譯器會將 `LocationName`、`LocationAddress` 與 `LocationCaption` 視為不受信任的中繼資料，並透過其他頻道情境所使用的同一條有限制 JSON 路徑將其序列化。

## 傳出內容承載

訊息工具與外掛 SDK 對可攜式傳出位置使用相同的 `NormalizedLocation` 形狀。僅包含座標的內容承載代表圖釘。原生支援地點的頻道可將 `name` 加上 `address` 對應至地點卡片。

Telegram 目前透過 `message(action="send")` 提供此功能。其初始實作刻意保持獨立：位置內容承載無法與文字或媒體混用，而且不完整的地點配對會失敗，而不是默默捨棄名稱或地址。不支援的頻道不會公開位置參數。

## 頻道注意事項

- **LINE**：位置訊息 `title`/`address` 會對應至 `LocationName`/`LocationAddress`；不支援即時位置。
- **Matrix**：`geo_uri` 會解析為圖釘位置；`u`（不確定度）參數會對應至 `LocationAccuracy`，事件本文會填入 `LocationCaption`，高度會被忽略，而 `LocationIsLive` 一律為 false。
- **Telegram**：地點會對應至 `LocationName`/`LocationAddress`；系統會透過 `live_period` 偵測即時位置。
- **WhatsApp**：`locationMessage.comment` 與 `liveLocationMessage.caption` 會填入 `LocationCaption`。

## 相關內容

- [位置命令（節點）](/zh-TW/nodes/location-command)
- [相機擷取](/zh-TW/nodes/camera)
- [媒體理解](/zh-TW/nodes/media-understanding)
