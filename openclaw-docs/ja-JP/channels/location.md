---
read_when:
    - チャネル位置情報の解析を追加または変更する
    - エージェントのプロンプトまたはツールで位置情報コンテキストフィールドを使用する
summary: チャネル位置情報の解析と移植可能な送信位置情報ペイロード
title: チャンネルの場所の解析
x-i18n:
    generated_at: "2026-07-26T08:52:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c7e5647d02643ad6d95024b362228377690d7fdff66441fae367f0f5307217fb
    source_path: channels/location.md
    workflow: 16
---

OpenClaw は、チャットチャンネルから共有された位置情報を次の形式に正規化します。

- 受信本文に追加される簡潔な座標テキスト、および
- 自動返信コンテキストペイロード内の構造化フィールド。チャンネルから提供されたラベル、住所、キャプション／コメントは、ユーザー本文にインラインで含まれるのではなく、共有の信頼できないメタデータ JSON ブロックによってプロンプトにレンダリングされます。

現在サポートされているもの：

- **LINE**（タイトル／住所付きの位置情報メッセージ）
- **Matrix**（`geo_uri` を含む `m.location`）
- **Telegram**（位置ピン、施設、ライブ位置情報）
- **WhatsApp**（`locationMessage` + `liveLocationMessage`）

## テキストの書式

位置情報は、角括弧を使わず読みやすい行としてレンダリングされます。座標は小数点以下 6 桁で表示され、精度はメートル単位の整数に丸められます。

- ピン：
  - `📍 48.858844, 2.294351 ±12m`
- 名前付きの場所（同じ行。名前／住所はメタデータブロックにのみ含まれます）：
  - `📍 48.858844, 2.294351 ±12m`
- ライブ共有：
  - `🛰 Live location: 48.858844, 2.294351 ±12m`

チャンネルにラベル、住所、またはキャプション／コメントが含まれている場合、それらはコンテキストペイロードに保持され、フェンスで囲まれた信頼できない JSON としてプロンプトに表示されます（存在しないフィールドは省略されます）。

````text
位置情報（信頼できないメタデータ）：
```json
{
  "latitude": 48.858844,
  "longitude": 2.294351,
  "accuracy_m": 12,
  "source": "place",
  "name": "Eiffel Tower",
  "address": "Champ de Mars, Paris",
  "caption": "ここで待ち合わせ"
}
```
````

## コンテキストフィールド

位置情報が存在する場合、次のフィールドが `ctx` に追加されます。

- `LocationLat`（数値）
- `LocationLon`（数値）
- `LocationAccuracy`（数値、メートル。省略可能）
- `LocationName`（文字列。省略可能）
- `LocationAddress`（文字列。省略可能）
- `LocationSource`（`pin | place | live`）
- `LocationIsLive`（真偽値）
- `LocationCaption`（文字列。省略可能）

チャンネルが明示的なソースを設定していない場合、OpenClaw が推論します。ライブ共有は `live`、名前または住所を持つ位置情報は `place`、それ以外はすべて `pin` になります。

プロンプトレンダラーは、`LocationName`、`LocationAddress`、および `LocationCaption` を信頼できないメタデータとして扱い、他のチャンネルコンテキストに使用されるものと同じサイズ制限付き JSON パスを介してシリアライズします。

## 送信ペイロード

メッセージツールと Plugin SDK は、移植可能な送信位置情報に同じ `NormalizedLocation` 形式を使用します。座標のみのペイロードはピンを表します。ネイティブの施設サポートを持つチャンネルでは、`name` と `address` を施設カードにマッピングできます。

Telegram では現在、これを `message(action="send")` を介して公開しています。最初の実装は意図的に独立したものになっています。位置情報ペイロードをテキストやメディアと混在させることはできず、不完全な施設情報のペアは、名前または住所を暗黙に破棄するのではなく失敗します。未対応のチャンネルは位置情報パラメーターを公開しません。

## チャンネルに関する注記

- **LINE**：位置情報メッセージの `title`/`address` は `LocationName`/`LocationAddress` にマッピングされます。ライブ位置情報はありません。
- **Matrix**：`geo_uri` はピン位置情報として解析されます。`u`（不確実性）パラメーターは `LocationAccuracy` にマッピングされ、イベント本文が `LocationCaption` に設定されます。高度は無視され、`LocationIsLive` は常に false です。
- **Telegram**：施設は `LocationName`/`LocationAddress` にマッピングされます。ライブ位置情報は `live_period` によって検出されます。
- **WhatsApp**：`locationMessage.comment` と `liveLocationMessage.caption` が `LocationCaption` に設定されます。

## 関連項目

- [位置情報コマンド（Node）](/ja-JP/nodes/location-command)
- [カメラキャプチャ](/ja-JP/nodes/camera)
- [メディア理解](/ja-JP/nodes/media-understanding)
