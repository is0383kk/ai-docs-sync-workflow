---
read_when:
    - ペアリングモードの DM を使用しているため、送信者を承認する必要があります
summary: '`openclaw pairing` の CLI リファレンス（ペアリング要求の承認/一覧表示）'
title: ペアリング
x-i18n:
    generated_at: "2026-07-26T09:36:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e4c6c53f1a3eefe50b4b7a45fa535e9a05faabb50df1ba5195a7635ee13d9da0
    source_path: cli/pairing.md
    workflow: 16
---

# `openclaw pairing`

ペアリングをサポートするチャンネルの DM ペアリングリクエストを承認または確認します（チャット DM のみ。Node/デバイスのペアリングには `openclaw devices` を使用します）。

関連項目: [ペアリングの流れ](/ja-JP/channels/pairing)

同じ保留中のリクエストは、Control UI の **Settings →
Channels → DM access requests** でも確認できます。Control UI では、承認、リクエスト送信者への任意の通知、破棄が可能です。破棄すると現在のリクエストは削除されますが、送信者が永続的にブロックされることはありません。

## コマンド

```bash
openclaw pairing list telegram
openclaw pairing list --channel telegram --account work
openclaw pairing list telegram --json

openclaw pairing approve <code>
openclaw pairing approve telegram <code>
openclaw pairing approve --channel telegram --account work <code> --notify
```

## `pairing list`

1 つのチャンネルについて、保留中のペアリングリクエストを一覧表示します。

| オプション                  | 説明                           |
| ----------------------- | ------------------------------------- |
| `[channel]`             | 位置引数で指定するチャンネル ID                 |
| `--channel <channel>`   | 明示的に指定するチャンネル ID                   |
| `--account <accountId>` | 複数アカウント対応チャンネルのアカウント ID |
| `--json`                | 機械可読形式の出力               |

ペアリング対応チャンネルが複数設定されている場合は、チャンネルを位置引数または `--channel` で指定します。チャンネル ID が有効であれば、拡張チャンネルも使用できます。

## `pairing approve`

保留中のペアリングコードを承認し、その送信者を許可します。

使用方法:

- `openclaw pairing approve <channel> <code>`
- `openclaw pairing approve --channel <channel> <code>`
- `openclaw pairing approve <code>`（ペアリング対応チャンネルが 1 つだけ設定されている場合）

オプション: `--channel <channel>`、`--account <accountId>`、`--notify`（同じチャンネルでリクエスト送信者に確認を返信します）。

### オーナーの初期設定

ペアリングコードを承認した時点で `commands.ownerAllowFrom` が空の場合、CLI は承認された送信者をコマンドオーナーとしても記録し、`telegram:123456789` のようなチャンネル単位のエントリを使用します。これは最初のオーナーを初期設定する場合にのみ行われます。その後のペアリング承認によって `commands.ownerAllowFrom` が置き換えられたり、拡張されたりすることはありません。Control UI では、この権限昇格は自動的に適用されず、`operator.admin` で保護された別個のチェックボックスとして表示されます。

コマンドオーナーとは、オーナー専用コマンドの実行と、`/diagnostics`、`/export-session`、`/export-trajectory`、`/config`、および exec の承認などの危険な操作の承認を許可された人間のオペレーターアカウントです。ペアリングで可能になるのは、送信者がエージェントと会話することだけです。この 1 回限りの初期設定を除き、ペアリング自体によってオーナー権限が付与されることはありません。

この初期設定機能が導入される前に送信者を承認していた場合は、`openclaw doctor` を実行してください。コマンドオーナーが設定されていない場合に警告が表示され、修正に使用する正確な `openclaw config set commands.ownerAllowFrom ...` コマンドが示されます。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [チャンネルのペアリング](/ja-JP/channels/pairing)
