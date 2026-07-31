---
read_when:
    - OpenClaw が Nostr 経由で DM を受信できるようにする場合
    - 分散型メッセージングをセットアップしています
summary: NIP-04 暗号化メッセージを使用する Nostr DM チャンネル
title: Nostr
x-i18n:
    generated_at: "2026-07-26T08:54:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 31fa283f706036a37795ddad71602058ba94388a9cb01044927c4bb2d83ba4a8
    source_path: channels/nostr.md
    workflow: 16
---

Nostr は、Nostr リレー経由で NIP-04 暗号化ダイレクトメッセージを OpenClaw が受信して応答できるようにする、ダウンロード可能なチャンネル Plugin (`@openclaw/nostr`) です。Gateway ごとに 1 アカウントで、DM のみをサポートします。

## インストール

```bash
openclaw plugins install @openclaw/nostr
```

現在の公式リリースタグに追随するには、バージョンを付けないパッケージ指定を使用します。再現可能なインストールが必要な場合にのみ、正確なバージョンを固定してください。

ローカルチェックアウトからインストールする場合（開発ワークフロー）:

```bash
openclaw plugins install --link <path-to-local-nostr-plugin>
```

Plugin をインストールまたは有効化した後は、Gateway を再起動してください。Plugin がインストールされると、オンボーディング (`openclaw onboard`) と `openclaw channels add` に、共有チャンネルカタログから Nostr が表示されます。

### 非対話型セットアップ

```bash
openclaw channels add --channel nostr --private-key "$NOSTR_PRIVATE_KEY"
openclaw channels add --channel nostr --private-key "$NOSTR_PRIVATE_KEY" --relay-urls "wss://relay.damus.io,wss://relay.primal.net"
```

設定にキーを保存せず、`NOSTR_PRIVATE_KEY` を環境に保持するには、`--use-env` を使用します（デフォルトアカウントのみ）。

## クイックセットアップ

1. Nostr キーペアを生成します（必要な場合）:

```bash
# nak を使用
nak key generate
```

2. 設定に追加します:

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
    },
  },
}
```

3. キーをエクスポートします:

```bash
export NOSTR_PRIVATE_KEY="nsec1..."
```

4. Gateway を再起動します。

## 設定リファレンス

| キー          | 型     | デフォルト                                     | 説明                                              |
| ------------ | -------- | ------------------------------------------- | -------------------------------------------------------- |
| `privateKey` | string   | 必須                                    | `nsec` または 16 進数形式の秘密鍵。シークレット参照を使用可能 |
| `relays`     | string[] | `['wss://relay.damus.io', 'wss://nos.lol']` | リレー URL（WebSocket）                                   |
| `dmPolicy`   | string   | `pairing`                                   | DM アクセスポリシー                                         |
| `allowFrom`  | string[] | `[]`                                        | 許可する送信者の公開鍵                                   |
| `enabled`    | boolean  | `true`                                      | チャンネルの有効化／無効化                                   |
| `name`       | string   | -                                           | 表示名                                             |
| `profile`    | object   | -                                           | NIP-01 プロファイルメタデータ                                  |

## プロファイルメタデータ

プロファイルデータは、NIP-01 `kind:0` イベントとして公開されます。Control UI（Channels -> Nostr -> Profile）から管理するか、設定で直接指定できます。

例:

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      profile: {
        name: "openclaw",
        displayName: "OpenClaw",
        about: "パーソナルアシスタント DM ボット",
        picture: "https://example.com/avatar.png",
        banner: "https://example.com/banner.png",
        website: "https://example.com",
        nip05: "openclaw@example.com",
        lud16: "openclaw@example.com",
      },
    },
  },
}
```

注意:

- プロファイル URL には `https://` を使用する必要があります。
- リレーからインポートすると、フィールドがマージされ、ローカルの上書き設定が保持されます。

## アクセス制御

### DM ポリシー

- **pairing**（デフォルト）: 未知の送信者にはペアリングコードが送られます。
- **allowlist**: `allowFrom` に含まれる公開鍵のみが DM を送信できます。
- **open**: 公開受信 DM（`allowFrom: ["*"]` が必要）。
- **disabled**: 受信 DM を無視します。

適用に関する注意:

- 受信イベントの署名は、送信者ポリシーの適用と NIP-04 復号より前に検証されるため、偽造イベントは早期に拒否されます。
- ペアリング応答は、元の DM 本文を復号または処理せずに送信されます。
- 受信 DM にはレート制限（全体および送信者ごと）が適用され、サイズ超過のペイロードは復号前に破棄されます。

### 許可リストの例

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      dmPolicy: "allowlist",
      allowFrom: ["npub1abc...", "npub1xyz..."],
    },
  },
}
```

## キー形式

使用可能な形式:

- **秘密鍵:** `nsec...` または 64 文字の 16 進数
- **公開鍵（`allowFrom`）:** `npub...` または 16 進数

## リレー

デフォルト: `relay.damus.io` および `nos.lol`。

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      relays: ["wss://relay.damus.io", "wss://relay.primal.net", "wss://nostr.wine"],
    },
  },
}
```

ヒント:

- 冗長性を確保するため、2～3 個のリレーを使用してください。
- リレーを増やしすぎないでください（遅延、重複の原因になります）。
- 有料リレーを使用すると、信頼性が向上する場合があります。
- テストにはローカルリレーを使用できます（`ws://localhost:7777`）。

## プロトコルサポート

| NIP    | 状態    | 説明                           |
| ------ | --------- | ------------------------------------- |
| NIP-01 | 対応 | 基本イベント形式 + プロファイルメタデータ |
| NIP-04 | 対応 | 暗号化 DM（`kind:4`）              |
| NIP-17 | 計画中   | ギフトラップされた DM                      |
| NIP-44 | 計画中   | バージョン付き暗号化                  |

## テスト

### ローカルリレー

```bash
# strfry を起動
docker run -p 7777:7777 ghcr.io/hoytech/strfry
```

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      relays: ["ws://localhost:7777"],
    },
  },
}
```

### 手動テスト

1. Gateway のログまたは `openclaw channels status` からボットの公開鍵を確認します（16 進数。必要に応じてクライアントで npub に変換してください）。
2. Nostr クライアント（Amethyst、Damus など）を開きます。
3. ボットの公開鍵宛てに DM を送信します。
4. 応答を確認します。

## トラブルシューティング

### メッセージを受信できない

- 秘密鍵が有効であることを確認します。
- リレー URL に接続でき、`wss://`（ローカルの場合は `ws://`）が使用されていることを確認します。
- `enabled` が `false` ではないことを確認します。
- Gateway のログでリレー接続エラーを確認します。

### 応答を送信できない

- リレーが書き込みを受け付けることを確認します。
- 外向きの接続を確認します。
- リレーのレート制限に注意してください。

### 応答が重複する

- 複数のリレーを使用している場合は想定される動作です。
- メッセージはイベント ID で重複排除され、最初の配信のみが応答をトリガーします。

## セキュリティ

- 秘密鍵を絶対にコミットしないでください。
- キーには環境変数を使用してください。
- 本番環境のボットでは `allowlist` の使用を検討してください。
- 署名は送信者ポリシーより前に検証され、送信者ポリシーは復号より前に適用されるため、偽造イベントは早期に拒否され、未知の送信者が完全な暗号処理を強制することはできません。

## 制限事項（MVP）

- ダイレクトメッセージのみ（グループチャットは非対応）。
- メディア添付ファイルは非対応。
- NIP-04 のみ（NIP-17 ギフトラップは対応予定）。

## 関連項目

- [チャンネル概要](/ja-JP/channels) — 対応しているすべてのチャンネル
- [ペアリング](/ja-JP/channels/pairing) — DM 認証とペアリングの流れ
- [グループ](/ja-JP/channels/groups) — グループチャットの動作とメンションゲーティング
- [チャンネルルーティング](/ja-JP/channels/channel-routing) — メッセージのセッションルーティング
- [セキュリティ](/ja-JP/gateway/security) — アクセスモデルと堅牢化
