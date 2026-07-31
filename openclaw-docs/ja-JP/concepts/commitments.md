---
read_when:
    - 推定されたコミットメントを使用していた構成をアップグレードしています
    - 以前に保存されたフォローアップ記録を確認または破棄したい場合
sidebarTitle: Commitments
summary: 廃止された推定フォローアップコミットメントのステータスとクリーンアップに関するガイダンス
title: 推論されたコミットメント
x-i18n:
    generated_at: "2026-07-26T09:17:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cfaa8c44be4ffb8db48279dba5347d4f598a193bfc4e244aeaed7a93e00ffb79
    source_path: concepts/commitments.md
    workflow: 16
---

推定コミットメントの実験は廃止されました。OpenClaw は新しい
会話のフォローアップを抽出したり、Heartbeat を通じて配信したりしなくなり、以前の
`commitments` 設定ブロックは `openclaw doctor --fix` によって削除されます。

正確なリマインダーとスケジュールされた作業では、引き続き
[スケジュールされたタスク](/ja-JP/automation/cron-jobs)を使用します。永続的な会話上の事実は
[メモリ](/ja-JP/concepts/memory)に保存します。

## 既存のレコード

以前に保存されたコミットメントは共有 SQLite 状態データベースに残るため、
アップグレードによってオペレーターに表示される履歴が失われることはありません。これらの行を確認または解除するには、
レガシーメンテナンス CLI を使用します。

```bash
openclaw commitments --all
openclaw commitments dismiss cm_abc123
```

メンテナンスコマンドのリファレンスについては、[`openclaw commitments`](/ja-JP/cli/commitments)を
参照してください。

## 関連項目

- [スケジュールされたタスク](/ja-JP/automation/cron-jobs)
- [メモリの概要](/ja-JP/concepts/memory)
- [Heartbeat](/ja-JP/gateway/heartbeat)
