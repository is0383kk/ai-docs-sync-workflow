---
read_when:
    - Tailscale + CoreDNS を介した広域ディスカバリー（DNS-SD）を利用する場合
    - You're setting up split DNS for a custom discovery domain (example: openclaw.internal)
summary: '`openclaw dns`（広域検出ヘルパー）の CLI リファレンス'
title: DNS
x-i18n:
    generated_at: "2026-07-26T09:56:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bb07353df03f9d169e1aede2da0b711ffb68e8c9d21d51359e93e92cc0818ca2
    source_path: cli/dns.md
    workflow: 16
---

# `openclaw dns`

広域検出（Tailscale + CoreDNS）用の DNS ヘルパーです。現在は macOS + Homebrew CoreDNS のみに対応しています。

関連項目:

- Gateway の検出: [検出](/ja-JP/gateway/discovery)
- 広域検出の設定: [設定](/ja-JP/gateway/configuration)

## `dns setup`

ユニキャスト DNS-SD 検出用の CoreDNS セットアップを計画または適用します。

```bash
openclaw dns setup
openclaw dns setup --domain openclaw.internal
openclaw dns setup --apply
```

| オプション              | 効果                                                                              |
| ------------------- | ----------------------------------------------------------------------------------- |
| `--domain <domain>` | 広域検出ドメイン（例: `openclaw.internal`）。                       |
| `--apply`           | CoreDNS の設定をインストールまたは更新し、サービスを（再）起動します。sudo が必要で、macOS のみに対応しています。 |

`--domain` を指定しない場合、OpenClaw は設定の `discovery.wideArea.domain` を使用します。

`--apply` を指定しない場合、コマンドは以下のみを出力します:

- 解決された検出ドメインとゾーンファイルのパス
- 現在の tailnet IP
- 推奨される `openclaw.json` 検出設定
- Tailscale 管理コンソールで設定する Tailscale Split DNS のネームサーバーとドメインの値

`--apply` を指定した場合（macOS のみ、Homebrew CoreDNS が必要）:

- ゾーンファイルがない場合は初期作成します
- CoreDNS の import スタンザがない場合は追加します
- `coredns` brew サービスを再起動します

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [検出](/ja-JP/gateway/discovery)
