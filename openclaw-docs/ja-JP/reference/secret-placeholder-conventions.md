---
read_when:
    - トークン、API キー、認証情報のスニペットを含むドキュメントの作成
    - シークレット検出ツールによってスキャンされる可能性がある例の更新
summary: ドキュメントと例で使用するシークレットスキャナーに安全なプレースホルダー規則
title: シークレットのプレースホルダー規則
x-i18n:
    generated_at: "2026-07-26T10:00:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0864f0fcc6fb1e4a3147b4b2ce0aac475437a19d694f3d059374782428c7f248
    source_path: reference/secret-placeholder-conventions.md
    workflow: 16
---

# シークレットのプレースホルダー規則

人間が読みやすく、実際のシークレットには見えないプレースホルダーを使用します。

## 推奨スタイル

- `example-openai-key-not-real` や `example-discord-bot-token` のような説明的な値を推奨します。
- シェルスニペットでは、インラインのトークンのような文字列よりも `${OPENAI_API_KEY}` を推奨します。
- 例は明らかに偽物だと分かるものにし、目的（プロバイダー、チャンネル、認証タイプ）に限定してください。

## ドキュメントで避けるべきパターン

- PEM 秘密鍵のヘッダーまたはフッターのリテラルテキスト。
- 実際に使用されている認証情報に見える接頭辞（例: `sk-...`、`xoxb-...`、`AKIA...`）。
- ランタイムログからコピーした、本物らしく見えるベアラートークン。

## 例

```bash
# 良い例
export OPENAI_API_KEY="example-openai-key-not-real"

# より良い例（ドキュメントが環境変数の接続方法に関する場合）
export OPENAI_API_KEY="${OPENAI_API_KEY}"
```
