---
read_when:
    - iOS Node で HealthKit の概要を有効にする
    - health.summary の呼び出し、またはヘルスメトリクスが欠落している場合のトラブルシューティング
    - iOSデバイス外に送信される可能性があるヘルスデータの確認
summary: iOS Node からプライバシー保護された HealthKit サマリーを有効化して呼び出す
title: HealthKit の概要
x-i18n:
    generated_at: "2026-07-26T09:08:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b8ac13d2870c55e2083a5e3a14c3d04238c2780a9e83d091f31923eb738476af
    source_path: platforms/ios-healthkit.md
    workflow: 16
---

# HealthKit サマリー

OpenClaw は、接続された iPhone または iPad の Node から、現在の暦日の読み取り専用サマリーをリクエストできます。デバイス上で集計を行い、歩数、睡眠時間、平均安静時心拍数、ワークアウトの回数と時間のみを返します。個々の HealthKit サンプル、ソース、メタデータ、臨床記録、バックグラウンドでの取り込み、書き込みはサポートされていません。

この機能はデフォルトでは無効です。iOS デバイス上での個別の同意と、Gateway での承認が必要です。

## 要件

- HealthKit がヘルスケアデータを利用可能と報告する、OpenClaw iOS アプリを実行中の iPhone または iPad。
- 接続および承認済みの iOS Node。[iOS アプリのセットアップ](/ja-JP/platforms/ios)を参照してください。
- iOS Node に到達できる最新の Gateway。
- 表示する予定の各指標について、読み取り可能なヘルスケアデータ。Apple Watch は Apple ヘルスケアストアにデータを提供できますが、HealthKit サマリーに OpenClaw watchOS アプリは必要ありません。

## アクセスを有効にする

### 1. Gateway コマンドを承認する

`openclaw.json` の既存の `gateway.nodes.commands.allow` 配列に `health.summary` を追加します。すでに存在するコマンドは維持してください。

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["health.summary"] },
    },
  },
}
```

`health.summary` はプライバシーへの影響が大きいものとして分類され、iOS プラットフォームのデフォルトでは許可されません。`gateway.nodes.commands.deny` のエントリは許可エントリより優先されます。[Node コマンドポリシー](/ja-JP/nodes#command-policy)を参照してください。

### 2. iOS デバイスで共有を有効にする

iOS アプリで、次の操作を行います。

1. **Settings -> Permissions** を開き、常に表示される **Apple Health** セクションで **Apple Health Summaries** を見つけます。
2. **Enable Apple Health Summaries** をタップします。
3. 開示内容を読み、Apple の権限シートで OpenClaw に読み取りを許可するヘルスケアカテゴリを選択します。

このスイッチには、OpenClaw との共有を明示的に選択したことが記録されます。Apple がリクエストされたすべてのカテゴリを許可したことを示すものではありません。

ヘルスケアサマリーを有効にすると、Node が宣言するコマンドサーフェスに `health.summary` が追加されます。その結果生じる Node ペアリングの更新を承認します。

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
```

次に、接続された iOS デバイスが有効な `health.summary` コマンドを公開していることを確認します。

```bash
openclaw nodes describe --node "<iOS device name>"
```

## 今日のサマリーをリクエストする

`today` のみがサポートされています。iOS デバイスの現在のカレンダーとタイムゾーンを使用し、現地時刻の午前 0 時からリクエスト時刻までが対象です。

```bash
openclaw nodes invoke \
  --node "<iOS device name>" \
  --command health.summary \
  --params '{"period":"today"}' \
  --json
```

エージェントは `nodes` ツールを使用して同じコマンドを呼び出せます。

```json
{
  "action": "invoke",
  "node": "<iOS device name>",
  "invokeCommand": "health.summary",
  "invokeParamsJson": "{\"period\":\"today\"}"
}
```

サマリーペイロードには次の内容が含まれます。

| フィールド                    | 意味                                       |
| ------------------------ | --------------------------------------------- |
| `period`                 | 常に `today`                                |
| `startISO`               | ISO インスタントとしてエンコードされた現地時刻での一日の開始 |
| `endISO`                 | ISO インスタントとしてエンコードされたリクエスト時刻       |
| `timeZoneIdentifier`     | iOS デバイスのタイムゾーン識別子               |
| `stepCount`              | 丸められた累積歩数                      |
| `sleepDurationMinutes`   | 重複を排除し、今日の範囲に切り詰めた睡眠時間    |
| `restingHeartRateBpm`    | 平均安静時心拍数                    |
| `workoutCount`           | 今日開始したワークアウト                   |
| `workoutDurationMinutes` | それらのワークアウトの合計時間              |

指標フィールドは任意であり、HealthKit が読み取り可能な値を返さない場合は省略されます。時間を計算する前に睡眠ステージと重複するソースが統合されるため、同じ 1 分間が二重にカウントされることはありません。

## プライバシーに関する動作

- 集計は iOS デバイス上で行われます。生のサンプルがデバイスから外部に送信されることはありません。
- リクエストされた集計は、Gateway を介してデバイスの外部に送信されます。エージェントがリクエストした場合、集計は構成された AI プロバイダーに送られ、チャット履歴に残ることがあります。CLI から直接呼び出した場合は、CLI オペレーターに返されます。
- OpenClaw がリクエストするのは読み取りアクセスのみです。ヘルスケアデータの追加や変更はできません。
- OpenClaw が HealthKit を読み取るのは、`health.summary` が呼び出された場合のみです。バックグラウンドでのヘルスケアデータの取り込みは行われません。
- HealthKit は、読み取りアクセスが拒否されたかどうかを意図的に開示しません。指標がない場合、アクセスが拒否された、該当するサンプルがない、またはデータ型が利用できない可能性があります。OpenClaw はこれらのケースを区別できません。
- このサマリーは個人の健康およびフィットネスに関するコンテキストを目的としており、診断や医学的助言を目的とするものではありません。

共有を停止するには、**Apple Health Summaries** に戻り、**Turn Off Summaries** をタップします。その後、iOS デバイスは Node サーフェスからヘルスケア機能と `health.summary` コマンドを削除します。`gateway.nodes.commands.allow` から `health.summary` を削除して、Gateway 側のゲートを閉じることもできます。

## トラブルシューティング

### コマンドが Node によって宣言されていない

iOS アプリで Apple ヘルスケアサマリーが有効になっており、デバイスが接続されていることを確認します。`openclaw nodes pending` を実行して機能の更新があれば承認し、`openclaw nodes describe --node "<iOS device name>"` を再度確認します。

### コマンドには明示的なオプトインが必要

`gateway.nodes.commands.allow` に `health.summary` を追加します。また、`gateway.nodes.commands.deny` にそれが含まれていないことを確認してください。拒否リストが優先されます。

### `HEALTH_ACCESS_DISABLED`

アプリ側の共有スイッチがオフになっています。iOS デバイスで **Settings -> Permissions -> Apple Health** を開き、**Apple Health Summaries** を有効にします。

### サマリーは成功するが指標がない

Apple のヘルスケアアプリを開き、今日のデータが存在することを確認します。Apple のヘルスケア設定で OpenClaw のアクセス権を確認してください。ただし、結果が空であってもアクセスが拒否された証拠とはみなさないでください。HealthKit は意図的にその違いを隠します。

### 過去の期間を指定すると失敗する

このコマンドで使用できるのは `{"period":"today"}` のみです。複数日および過去のサマリーはサポートされていません。

## 関連項目

- [iOS アプリ](/ja-JP/platforms/ios)
- [Node](/ja-JP/nodes)
- [Gateway 構成リファレンス](/ja-JP/gateway/configuration-reference#gateway)
- [セキュリティ監査](/ja-JP/gateway/security)
