---
read_when:
    - ワークスペースを手動でブートストラップする
summary: TOOLS.md のワークスペーステンプレート
title: TOOLS.md テンプレート
x-i18n:
    generated_at: "2026-07-26T09:51:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 20eab78b3b117566a1d33a70873e70ff2d5099543aa44e2719dc8d0797099afe
    source_path: reference/templates/TOOLS.md
    workflow: 16
---

# TOOLS.md - ローカルメモ

Skills はツールがどのように動作するかを定義します。このファイルには、セットアップ固有の情報を記載します。たとえば、カメラの名前と場所、SSH ホストとエイリアス、優先する TTS 音声、スピーカー名や部屋名、デバイスのニックネームなど、環境固有の情報です。

## 例

```markdown
### カメラ

- living-room → メインエリア、180° 広角
- front-door → 玄関、モーション検知式

### SSH

- home-server → 192.168.1.100、ユーザー: admin

### TTS

- 優先する音声: "Nova"（温かみのある、やや英国風の声）
- デフォルトのスピーカー: Kitchen HomePod
```

## 分ける理由

Skills は共有されます。セットアップは自分専用です。両者を分けておけば、メモを失わずに Skills を更新でき、インフラストラクチャを漏らさずに Skills を共有できます。

---

作業に役立つ情報を自由に追加してください。これは自分用の早見表です。

## 関連項目

- [エージェントワークスペース](/ja-JP/concepts/agent-workspace)
