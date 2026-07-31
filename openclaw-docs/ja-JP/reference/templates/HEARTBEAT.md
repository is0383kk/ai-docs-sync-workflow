---
read_when:
    - ワークスペースを手動で初期化する
summary: HEARTBEAT.md のワークスペーステンプレート
title: HEARTBEAT.md テンプレート
x-i18n:
    generated_at: "2026-07-26T10:01:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d5b02cd62708a87515c4ae59bd2ffab3e4c8ebf81f4126fdd43ced756241b151
    source_path: reference/templates/HEARTBEAT.md
    workflow: 16
---

# HEARTBEAT.md テンプレート

`HEARTBEAT.md` はエージェントワークスペースにあり、定期的な Heartbeat のチェックリストを保持します。OpenClaw が Heartbeat のモデル呼び出しを完全にスキップするようにするには、このファイルを空にするか、空白、Markdown コメント、ATX 見出し、空のリストスタブ（`- `、`* [ ]`）、またはフェンスマーカーのみを含めます（`reason=empty-heartbeat-file`）。

出荷時のデフォルト内容：

```markdown
<!-- Heartbeat template; comments-only content prevents scheduled heartbeat API calls. -->

# Heartbeat API 呼び出しをスキップするには、このファイルを空にしてください（コメントのみでも可）。

# Heartbeat で共有コンテキストを確認する場合は、以下に短いチェックリストを追加してください。
```

1 回の Heartbeat ターンで複数の項目をまとめて確認する必要がある場合にのみ、コメント行の下に短いチェックリストを追加してください。簡潔に保ってください。Heartbeat はティックごと（デフォルトでは 30 分ごと）にこのファイルを読み取るため、指示が肥大化すると起動のたびにトークンを消費します。

個別にスケジュールするチェックや、期限到来時のみ実行するチェックについては、[Cron ジョブ](/ja-JP/automation/cron-jobs)を作成してください。Heartbeat のスクラッチは、スケジューラ構文をサポートしなくなりました。古い `tasks:` ブロックを変換するには、`openclaw doctor --fix` を実行してください。

## 関連項目

- [Heartbeat](/ja-JP/gateway/heartbeat)
- [Heartbeat の設定](/ja-JP/gateway/config-agents)
