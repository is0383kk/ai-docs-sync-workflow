---
read_when:
    - ボットが作成するチャンネルメッセージの設定
    - ボット間ループ保護の調整
sidebarTitle: Bot loop protection
summary: ボット間ループ保護のデフォルト設定とチャンネル別オーバーライド
title: ボットのループ防止
x-i18n:
    generated_at: "2026-07-26T10:04:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d59d3b48dd5506e774282b880334df8970b05c4d001261ff7107e8e1678894db
    source_path: channels/bot-loop-protection.md
    workflow: 16
---

OpenClaw は、`allowBots` をサポートするチャンネル上で、他のボットが書き込んだメッセージを受信できます。この経路が有効な場合、ペアループ保護により、2 つのボット ID が互いに際限なく返信し続けることを防止します。

このガードは、コアの受信返信ランナーによって適用されます。対応する各チャンネルは、受信イベントをアカウントまたはスコープ、会話 ID、送信元ボット ID、受信先ボット ID という汎用情報にマッピングします。コアは参加者ペアを双方向で追跡し（A から B と B から A は同じペアとしてカウント）、スライディングウィンドウの上限を適用し、上限を超えた後はクールダウン期間中そのペアを抑制します。

## デフォルト

チャンネルがボット作成メッセージをディスパッチに到達させる場合、ペアループ保護は常に有効です。組み込みのデフォルトは次のとおりです。

| キー                  | デフォルト | 意味                                             |
| -------------------- | ------- | --------------------------------------------------- |
| `enabled`            | `true`  | 対応するチャンネルでガードを有効にします。          |
| `maxEventsPerWindow` | `20`    | ウィンドウ内でボットペアが交換できるイベント数です。   |
| `windowSeconds`      | `60`    | スライディングウィンドウの長さです。                              |
| `cooldownSeconds`    | `60`    | ペアが上限を超えた後の抑制時間です。 |

このガードは、人間が作成したメッセージ、単一ボットのデプロイ、自己メッセージのフィルタリング、または上限以内に収まるボット返信には影響しません。

## 共有デフォルトの設定

`channels.defaults.botLoopProtection` を一度設定すると、対応するすべてのチャンネルに同じ基準値が適用されます。チャンネルによっては、より限定的なオーバーライドも公開されます。Feishu は意図的に、この共有基準値のみを使用します。

```json5
{
  channels: {
    defaults: {
      botLoopProtection: {
        maxEventsPerWindow: 20,
        windowSeconds: 60,
        cooldownSeconds: 60,
      },
    },
  },
}
```

自動抑制なしのボット間会話をチャンネルポリシーで意図的に許可する場合に限り、`enabled: false` を設定してください。

## チャンネル、アカウント、ルーム単位のオーバーライド

対応するチャンネルは、キーごとに独自の設定を共有デフォルトへ重ねて適用します。優先順位は、適用範囲が狭い順に次のとおりです。

1. `channels.<channel>.<room-or-space>.botLoopProtection`（チャンネルが会話単位のオーバーライドをサポートする場合）
2. `channels.<channel>.accounts.<account>.botLoopProtection`（チャンネルがアカウントをサポートする場合）
3. `channels.<channel>.botLoopProtection`（チャンネルがトップレベルのデフォルトをサポートする場合）
4. `channels.defaults.botLoopProtection`
5. 組み込みのデフォルト

```json5
{
  channels: {
    defaults: {
      botLoopProtection: {
        maxEventsPerWindow: 20,
      },
    },
    discord: {
      botLoopProtection: {
        maxEventsPerWindow: 8,
      },
      accounts: {
        secondary: {
          allowBots: true,
          botLoopProtection: {
            maxEventsPerWindow: 5,
            cooldownSeconds: 90,
          },
        },
      },
    },
    googlechat: {
      allowBots: true,
      groups: {
        "spaces/AAAA": {
          botLoopProtection: {
            maxEventsPerWindow: 5,
          },
        },
      },
    },
    matrix: {
      allowBots: "mentions",
      groups: {
        "!roomid:example.org": {
          botLoopProtection: {
            maxEventsPerWindow: 5,
          },
        },
      },
    },
    slack: {
      allowBots: "mentions",
      botLoopProtection: {
        maxEventsPerWindow: 8,
      },
    },
  },
}
```

## チャンネルの対応状況

- Discord：ネイティブの `author.bot` 情報を使用し、Discord アカウント、チャンネル、ボットペアをキーとします。
- Feishu：受け入れられたボット作成のグループメッセージについて、ネイティブの `sender_type=bot` 情報を使用し、Feishu アカウント、チャット、ボットペアをキーとします。Feishu は `channels.defaults.botLoopProtection` のみを使用します。
- Google Chat：受け入れられたボット作成メッセージについて、ネイティブの `sender.type=BOT` 情報を使用し、アカウント、スペース、ボットペアをキーとします。
- Matrix：設定済みの Matrix ボットアカウントを使用し、Matrix アカウント、ルーム、設定済みボットペアをキーとします。
- Slack：受け入れられたボット作成メッセージについて、ネイティブの `bot_id` 情報を使用し、Slack アカウント、チャンネル、ボットペアをキーとします。

信頼できる受信ボット ID を取得できないチャンネルでは、通常の自己メッセージフィルターとアクセスポリシーフィルターを引き続き使用します。ボットペアの両方の参加者を識別できるようになるまでは、このガードを有効にすべきではありません。

Plugin の実装詳細については、[SDK ランタイム](/ja-JP/plugins/sdk-runtime#reusable-runtime-utilities)を参照してください。
