---
read_when:
    - Node プラットフォームでのカメラキャプチャの追加または変更
    - エージェントがアクセス可能な MEDIA 一時ファイルワークフローの拡張
summary: 写真や短い動画クリップを撮影するための、iOS、Android、macOS、Linux Nodeでのカメラキャプチャ
title: カメラ撮影
x-i18n:
    generated_at: "2026-07-26T09:06:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b819f7ff3fc9b51757ae998d27f540975bf6c1194ed32fd36b1fbe909e79400c
    source_path: nodes/camera.md
    workflow: 16
---

OpenClaw は、ペアリングされた **iOS**、**Android**、**macOS**、**Linux** の各 Node 上で、エージェントワークフロー向けのカメラ撮影をサポートします。Gateway `node.invoke` を介して、写真の撮影（`jpg`）または短い動画クリップの撮影（`mp4`、音声は任意）が可能です。

すべてのカメラアクセスは、プラットフォームごとにユーザーが制御できる設定によって制限されます。

## iOS Node

### iOS のユーザー設定

- iOS Settings タブ → **Camera** → **Allow Camera**（`camera.enabled`）。
  - デフォルト: **オン**（キーがない場合は有効として扱われます）。
  - オフの場合: `camera.*` コマンドは `CAMERA_DISABLED` を返します。

### iOS コマンド（Gateway `node.invoke` 経由）

- `camera.list`
  - レスポンスペイロード: `devices` — `{ id, name, position, deviceType }` の配列。

- `camera.snap`
  - パラメーター:
    - `facing`: `front|back`（デフォルト: `front`）
    - `maxWidth`: 数値（任意、デフォルト `1600`）
    - `quality`: `0..1`（任意、デフォルト `0.9`、`[0.05, 1.0]` に制限）
    - `format`: 現在は `jpg`
    - `delayMs`: 数値（任意、デフォルト `0`、内部では `10000` を上限とする）
    - `deviceId`: 文字列（任意、`camera.list` から取得）
  - レスポンスペイロード: `format: "jpg"`、`base64`、`width`、`height`。
  - ペイロード保護: base64 エンコード後のペイロードが 5MB 未満になるよう、写真は再圧縮されます。

- `camera.clip`
  - パラメーター:
    - `facing`: `front|back`（デフォルト: `front`）
    - `durationMs`: 数値（デフォルト `3000`、`[250, 60000]` に制限）
    - `includeAudio`: 真偽値（デフォルト `true`）
    - `format`: 現在は `mp4`
    - `deviceId`: 文字列（任意、`camera.list` から取得）
  - レスポンスペイロード: `format: "mp4"`、`base64`、`durationMs`、`hasAudio`。

### iOS のフォアグラウンド要件

`canvas.*` と同様に、iOS Node は**フォアグラウンド**でのみ `camera.*` コマンドを許可します。バックグラウンドからの呼び出しは `NODE_BACKGROUND_UNAVAILABLE` を返します。

### CLI ヘルパー

メディアファイルを取得する最も簡単な方法は CLI ヘルパーを使用することです。デコードされたメディアが一時ファイルに書き込まれ、保存先のパスが出力されます。

```bash
openclaw nodes camera snap --node <id>                 # デフォルト: 前面 + 背面の両方（MEDIA 行 2 つ）
openclaw nodes camera snap --node <id> --facing front
openclaw nodes camera clip --node <id> --duration 3000
openclaw nodes camera clip --node <id> --no-audio
```

`nodes camera snap` のデフォルトは `--facing both` であり、エージェントに両方の視点を提供するため、前面と背面の両方を撮影します。単一の向きで撮影するには `--device-id` を明示的に指定してください（`--device-id` が設定されている場合、`both` は拒否されます）。独自のラッパーを構築しない限り、出力ファイルは一時的なものです（OS の一時ディレクトリ内）。

## Android Node

### Android のユーザー設定

- Android Settings シート → **Camera** → **Allow Camera**（`camera.enabled`）。
  - **新規インストールではデフォルトでオフです。** この設定が導入される前から存在するインストールは**オン**に移行されるため、アップグレードによって以前は機能していたカメラアクセスが通知なく失われることはありません。
  - オフの場合: `camera.*` コマンドは `CAMERA_DISABLED: enable Camera in Settings` を返します。

### 権限

- `CAMERA` は `camera.snap` と `camera.clip` の両方に必要です。権限がない、または拒否された場合は `CAMERA_PERMISSION_REQUIRED` を返します。
- `includeAudio` が `true` の場合、`camera.clip` には `RECORD_AUDIO` が必要です。権限がない、または拒否された場合は `MIC_PERMISSION_REQUIRED` を返します。

可能な場合、アプリは実行時権限を求めるプロンプトを表示します。

### Android のフォアグラウンド要件

`canvas.*` と同様に、Android Node は**フォアグラウンド**でのみ `camera.*` コマンドを許可します。バックグラウンドからの呼び出しは `NODE_BACKGROUND_UNAVAILABLE: command requires foreground` を返します。

### Android コマンド（Gateway `node.invoke` 経由）

- `camera.list`
  - レスポンスペイロード: `devices` — `{ id, name, position, deviceType }` の配列。

- `camera.snap`
  - パラメーター: `facing`（`front|back`、デフォルト `front`）、`quality`（デフォルト `0.95`、`[0.1, 1.0]` に制限）、`maxWidth`（デフォルト `1600`）、`deviceId`（任意、不明な ID の場合は `INVALID_REQUEST` で失敗）。
  - レスポンスペイロード: `format: "jpg"`、`base64`、`width`、`height`。
  - ペイロード保護: base64 が 5MB 未満になるよう再圧縮されます（iOS と同じ上限）。

- `camera.clip`
  - パラメーター: `facing`（デフォルト `front`）、`durationMs`（デフォルト `3000`、`[200, 60000]` に制限）、`includeAudio`（デフォルト `true`）、`deviceId`（任意）。
  - レスポンスペイロード: `format: "mp4"`、`base64`、`durationMs`、`hasAudio`。
  - ペイロード保護: base64 エンコード前の未加工 MP4 は 18MB を上限とします。上限を超えるクリップは `PAYLOAD_TOO_LARGE` で失敗します（`durationMs` を減らして再試行してください）。

## macOS アプリ

### macOS のユーザー設定

macOS コンパニオンアプリには、次のチェックボックスがあります。

- **Settings → General → Allow Camera**（`openclaw.cameraEnabled`）。
  - デフォルト: **オフ**。
  - オフの場合: カメラリクエストは `CAMERA_DISABLED: enable Camera in Settings` を返します。

### CLI ヘルパー（Node 呼び出し）

メインの `openclaw` CLI を使用して、macOS Node 上のカメラコマンドを呼び出します。

```bash
openclaw nodes camera list --node <id>                     # カメラ ID の一覧
openclaw nodes camera snap --node <id>                     # 保存先のパスを出力
openclaw nodes camera snap --node <id> --max-width 1280
openclaw nodes camera snap --node <id> --delay-ms 2000
openclaw nodes camera snap --node <id> --device-id <id>
openclaw nodes camera clip --node <id> --duration 10s       # 保存先のパスを出力
openclaw nodes camera clip --node <id> --duration-ms 3000   # 保存先のパスを出力（レガシーフラグ）
openclaw nodes camera clip --node <id> --device-id <id>
openclaw nodes camera clip --node <id> --no-audio
```

- 上書きされない限り、`openclaw nodes camera snap` のデフォルトは `maxWidth=1600` です。
- `camera.snap` は、ウォームアップと露出の安定後、撮影前に `delayMs`（デフォルト 2000ms、`[0, 10000]` に制限）待機します。
- base64 が 5MB 未満になるよう、写真のペイロードは再圧縮されます。

## Linux Node ホスト

同梱の Linux Node Plugin は、CLI `openclaw node` サービスにカメラ撮影機能を追加します。ヘッドレスホストで動作し、Linux デスクトップアプリは必要ありません。

カメラアクセスはデフォルトでオフです。Plugin エントリで有効にした後、Gateway のアドバタイズメントが再構築されるよう Node サービスを再起動してください。

```json5
{
  plugins: {
    entries: {
      "linux-node": {
        config: {
          camera: { enabled: true },
        },
      },
    },
  },
}
```

要件:

- V4L2 入力、`libx264`、AAC をサポートする FFmpeg
- Node サービスのユーザーが読み取れる `/dev/video*` デバイス。一般的なディストリビューションでは、そのユーザーを `video` グループに追加します
- デフォルトの `includeAudio: true` でクリップを撮影する場合、デフォルトソースを備えた、動作する PulseAudio サーバーまたは PipeWire の PulseAudio 互換レイヤー

Linux は、`camera.list` から撮影可能かつ読み取り可能な V4L2 デバイスパスを返します。FFmpeg は各 `/dev/video*` 候補をプローブし、メタデータ用または出力専用の Node を除外します。デバイスの `position` は `unknown` であるため、`deviceId` を指定せずに向きを要求すると、前面または背面カメラであると示す代わりに、`unknown` 位置の写真またはクリップを 1 つ生成します。ホストに複数のカメラがある場合は `deviceId` を使用してください。`camera.snap` は `delayMs` に FFmpeg の入力ウォームアップを使用し、幅を制限しながらアスペクト比を維持します。`camera.clip` はマイク音声を MP4 の音声トラックとして録音します。OpenClaw は意図的に、単独のマイクコマンドを公開していません。

この Plugin は MP4 動画に `libx264` を使用し、通知なくコーデックを変更することはありません。必要な入力またはエンコーダーがない FFmpeg ビルドは `CAMERA_UNAVAILABLE` を返します。25MB の base64 ペイロード上限を超える写真やクリップは `PAYLOAD_TOO_LARGE` で失敗します。

`camera.snap` と `camera.clip` は引き続き危険なコマンドです。撮影を有効化する場合にのみ、これらを `gateway.nodes.commands.allow` に追加してください。Plugin を有効にするだけでは、Gateway ポリシーを回避できません。

## 安全性と実用上の制限

- カメラとマイクへのアクセスでは、通常の OS 権限プロンプトが表示されます（また、`Info.plist` に使用目的の説明文が必要です）。
- Node のペイロードが大きくなりすぎるのを防ぐため、動画クリップは 60s を上限とします（base64 のオーバーヘッドとメッセージ上限を考慮）。

## macOS の画面動画（OS レベル）

カメラではなく_画面_の動画には、macOS コンパニオンを使用します。

```bash
openclaw nodes screen record --node <id> --duration 10s --fps 15   # 保存先のパスを出力
```

macOS の **Screen Recording** 権限（TCC）が必要です。

## 関連項目

- [画像とメディアのサポート](/ja-JP/nodes/images)
- [メディアの理解](/ja-JP/nodes/media-understanding)
- [位置情報コマンド](/ja-JP/nodes/location-command)
