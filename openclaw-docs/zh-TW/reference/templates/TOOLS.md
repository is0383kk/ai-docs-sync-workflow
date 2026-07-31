---
read_when:
    - 手動啟動工作區的初始設定
summary: TOOLS.md 的工作區範本
title: TOOLS.md 範本
x-i18n:
    generated_at: "2026-07-26T08:06:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 20eab78b3b117566a1d33a70873e70ff2d5099543aa44e2719dc8d0797099afe
    source_path: reference/templates/TOOLS.md
    workflow: 16
---

# TOOLS.md - 本機備註

Skills 定義工具_如何_運作。此檔案用於記錄_你的_特定資訊，也就是你的設定中獨有的內容：攝影機名稱與位置、SSH 主機與別名、偏好的 TTS 語音、揚聲器／房間名稱、裝置暱稱，以及任何環境特定資訊。

## 範例

```markdown
### 攝影機

- living-room → 主要區域，180° 廣角
- front-door → 入口，動作觸發

### SSH

- home-server → 192.168.1.100，使用者：admin

### TTS

- 偏好語音："Nova"（溫暖、略帶英國口音）
- 預設揚聲器：廚房 HomePod
```

## 為什麼要分開？

Skills 是共用的。你的設定則屬於你。將兩者分開，表示你可以更新 Skills 而不會遺失備註，也能分享 Skills 而不會洩漏基礎架構資訊。

---

加入任何有助於你完成工作的內容。這是你的速查表。

## 相關內容

- [代理程式工作區](/zh-TW/concepts/agent-workspace)
