---
read_when:
    - 初回のエージェント実行時に何が起こるかを理解する
    - ブートストラップファイルの配置場所の説明
    - オンボーディングのアイデンティティ設定をデバッグする
sidebarTitle: Bootstrapping
summary: ワークスペースとアイデンティティファイルを初期設定するエージェントのブートストラップ手順
title: エージェントのブートストラップ
x-i18n:
    generated_at: "2026-07-26T10:21:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: efb47e1a6a86d68aef1aa1662fe9c5def9a4e5b45649b84aeb9060bfcba21a5d
    source_path: start/bootstrapping.md
    workflow: 16
---

ブートストラップは、新しいエージェントワークスペースの初期データを作成し、エージェントがアイデンティティを選択できるよう案内する初回実行時の手順です。オンボーディングの直後、エージェントの最初の実際のターンで一度だけ実行されます。

## 実行内容

まったく新しいワークスペース（デフォルトは `~/.openclaw/workspace`）で初めて実行すると、OpenClaw は次の処理を行います。

- `AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md`、`HEARTBEAT.md`、`BOOTSTRAP.md` の初期データを作成します。
- エージェントに、最大 3 段階の誕生シーケンスを実行させます。エージェントの呼び名を尋ね、短い個性や雰囲気を表す一文を伝えた後、推奨される最小限の Plugin セットと、利便性を最大限に高めるセットのどちらを希望するか尋ねます。
- 合意したアイデンティティを 2 か所に保存します。`IDENTITY.md` と `SOUL.md`（エージェントが自身について読み取る情報）、および `openclaw agents set-identity`（チャンネルと UI に表示される情報）です。
- オンボーディング中にすでに保存されたアプリの推奨事項を、再スキャンせずに読み取ります。公式 Plugin には `openclaw plugins install <id>` を使用します。サードパーティー製 ClawHub Skills は、明示的なオプトインのままです。選択の処理後、エージェントは保存された提案を確認済みにし、二度と尋ねないようにします。
- ワークスペースが設定済みと判断されると `BOOTSTRAP.md` を削除し、この手順が一度だけ実行されるようにします。

`SOUL.md`、`IDENTITY.md`、`USER.md` のいずれかが初期テンプレートから変更されているか、`memory/` フォルダーが存在する場合、ワークスペースは設定済みと見なされます。

<Note>
`BOOTSTRAP.md` には、アイデンティティに関する会話全体が記載されています。内容については、[BOOTSTRAP.md テンプレート](/ja-JP/reference/templates/BOOTSTRAP)を参照してください。
</Note>

## 組み込みモデルおよびローカルモデルでの実行

組み込みモデルまたはローカルモデルで実行する場合、OpenClaw は `BOOTSTRAP.md` を特権システムコンテキストに含めません。主要な対話形式の初回実行では、引き続きユーザープロンプトを通じてファイルの内容を渡すため、`read` ツールを確実に呼び出せないモデルでも、この手順を完了できます。現在の実行でワークスペースに安全にアクセスできない場合、エージェントには一般的な挨拶ではなく、短い制限付きブートストラップの注記が渡されます。

## ブートストラップをスキップする

事前に初期データが作成されたワークスペースでこの処理をスキップするには、次を実行します。

```bash
openclaw onboard --skip-bootstrap
```

## 実行場所

ブートストラップは常に Gateway ホスト上で実行されます。macOS アプリがリモート Gateway に接続する場合、ワークスペースとそのブートストラップファイルは Mac ではなく、そのリモートマシン上に配置されます。

<Note>
Gateway が別のマシンで実行されている場合は、Gateway ホスト上でワークスペースファイル（たとえば `user@gateway-host:~/.openclaw/workspace`）を編集してください。
</Note>

## 関連ドキュメント

- macOS アプリのオンボーディング：[オンボーディング](/ja-JP/start/onboarding)
- ワークスペースの構成：[エージェントワークスペース](/ja-JP/concepts/agent-workspace)
- テンプレートの内容：[BOOTSTRAP.md テンプレート](/ja-JP/reference/templates/BOOTSTRAP)
