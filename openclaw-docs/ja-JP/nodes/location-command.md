---
read_when:
    - 位置情報 Node のサポートまたは権限 UI の追加
    - Android の位置情報権限またはフォアグラウンド動作の設計
summary: Node の位置情報コマンド、プラットフォームの権限モード、Linux GeoClue のセットアップ
title: 位置情報コマンド
x-i18n:
    generated_at: "2026-07-26T10:07:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 644229c1eafc8fc7b59bc23ba01d4ba95687ea66c4f9bd4a4cda98a87f2b6085
    source_path: nodes/location-command.md
    workflow: 16
---

## TL;DR

- `location.get` は Node コマンドで、`node.invoke` または `openclaw nodes location get` を介して呼び出されます。
- デフォルトではオフです。
- Android のサードパーティビルドでは、Off / While Using / Always のセレクターを使用します。Play ビルドでは引き続き Off / While Using のみです。
- Precise Location は独立したトグルです。

## スイッチだけでなくセレクターを使用する理由

OS の位置情報権限には複数のレベルがあります。正確な位置情報も独立した OS 権限です（iOS 14+ の「Precise」、Android の「fine」と「coarse」）。アプリ内のセレクターによって要求するモードが決まりますが、実際に付与する権限は引き続き OS が決定します。

## 設定モデル

Node デバイスごと:

- `location.enabledMode`: `off | whileUsing | always`
- `location.preciseEnabled`: bool

UI の動作:

- `whileUsing` を選択すると、フォアグラウンド権限を要求します。
- Android のサードパーティビルドで `always` を選択すると、まずフォアグラウンド権限を要求し、バックグラウンドアクセスについて説明した後、独立した **Allow all the time** 権限を付与するために Android のアプリ設定を開きます。
- Android Play ビルドでは、バックグラウンド位置情報権限を宣言せず、`always` も表示しません。
- OS が要求されたレベルを拒否した場合、アプリは付与済みの最高レベルに戻し、ステータスを表示します。

## 権限のマッピング（node.permissions）

任意です。macOS Node は、`node.list`/`node.describe` の `permissions` マップを介して `location` を報告します。iOS/Android では省略される場合があります。

## コマンド: `location.get`

`node.invoke` または CLI ヘルパーを介して呼び出します:

```bash
openclaw nodes location get --node <idOrNameOrIp>
openclaw nodes location get --node <idOrNameOrIp> --accuracy precise --max-age 15000 --location-timeout 10000
```

パラメーター:

```json
{
  "timeoutMs": 10000,
  "maxAgeMs": 15000,
  "desiredAccuracy": "coarse|balanced|precise"
}
```

CLI フラグは直接対応します: `--location-timeout` -> `timeoutMs`、`--max-age` -> `maxAgeMs`、`--accuracy` -> `desiredAccuracy`。

レスポンスペイロード:

```json
{
  "lat": 48.20849,
  "lon": 16.37208,
  "accuracyMeters": 12.5,
  "altitudeMeters": 182.0,
  "speedMps": 0.0,
  "headingDeg": 270.0,
  "timestamp": "2026-01-03T12:34:56.000Z",
  "isPrecise": true,
  "source": "gps|wifi|cell|unknown"
}
```

エラー（安定したコード）:

- `LOCATION_DISABLED`: セレクターがオフです。
- `LOCATION_PERMISSION_REQUIRED`: 要求されたモードに必要な権限がありません。
- `LOCATION_BACKGROUND_UNAVAILABLE`: アプリがバックグラウンドにありますが、While Using のみが付与されています。
- `LOCATION_TIMEOUT`: 時間内に位置情報を取得できませんでした。
- `LOCATION_UNAVAILABLE`: システム障害、またはプロバイダーがありません。

## バックグラウンドでの動作

- Android のサードパーティビルドでは、ユーザーが `Always` を選択し、Android がバックグラウンド位置情報を許可した場合に限り、バックグラウンドでの `location.get` を受け付けます。既存の常駐 Node サービスは `location` サービスタイプを追加し、動作中は `Location: Always` を明示します。
- Android Play ビルドと `While Using` モードでは、バックグラウンド中の `location.get` を拒否します。
- その他の Node プラットフォームでは動作が異なる場合があります。

## Linux Node ホスト

同梱の Linux Node Plugin は、Linux デスクトップアプリのないヘッドレスホストを含む CLI `openclaw node` サービスに `location.get` を追加します。位置情報はデフォルトでオフです。Plugin エントリで有効にしてから、Node サービスを再起動します:

```json5
{
  plugins: {
    entries: {
      "linux-node": {
        config: {
          location: { enabled: true },
        },
      },
    },
  },
}
```

GeoClue2 とその `where-am-i` デモ（Debian および Ubuntu では `geoclue-2-demo`）をインストールします。Node サービスのユーザーは、ホストの GeoClue ポリシーおよび認可エージェントによって許可されている必要があります。

Plugin は、一連の `busctl` 呼び出しの代わりに `where-am-i` を使用します。GeoClue では、クライアントの作成、プロパティ、開始、更新、停止が単一の D-Bus クライアント接続に関連付けられます。デモはこのライフサイクルをまとめて維持しますが、個別の `busctl` サブプロセスでは維持できません。npm 依存関係は追加されません。

Linux は `coarse`、`balanced`、`precise` を、それぞれ GeoClue の精度レベル `4`、`6`、`8` にマッピングします。返されたタイムスタンプに対して `maxAgeMs` を検証します。GeoClue のデモでは選択されたプロバイダーが公開されないため、`source` は `unknown` です。`isPrecise` は、報告された精度が 100 メートル以下の場合にのみ true になります。

Linux でも同じ安定したエラーを使用します: `LOCATION_DISABLED`、`LOCATION_TIMEOUT`、`LOCATION_UNAVAILABLE`。

## モデル／ツール連携

- エージェントツール: `nodes` ツールの `location_get` アクション（Node が必要）。
- CLI: `openclaw nodes location get --node <id>`。
- エージェントのガイドライン: ユーザーが位置情報を有効にし、その範囲を理解している場合にのみ呼び出します。

## UX 文言（推奨）

- Off: 「位置情報の共有は無効です。」
- While Using: 「OpenClaw が開いているときのみ。」
- Always: 「OpenClaw がバックグラウンドにある間も、要求された位置情報の確認を許可します。」
- Precise: 「正確な GPS 位置情報を使用します。おおよその位置情報を共有するにはオフに切り替えてください。」

## 関連項目

- [Node の概要](/ja-JP/nodes)
- [チャンネルの位置情報解析](/ja-JP/channels/location)
- [カメラ撮影](/ja-JP/nodes/camera)
- [トークモード](/ja-JP/nodes/talk)
