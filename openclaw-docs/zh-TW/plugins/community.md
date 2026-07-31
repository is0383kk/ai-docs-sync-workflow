---
doc-schema-version: 1
read_when:
    - 你想尋找第三方 OpenClaw 外掛
    - 你想要在 ClawHub 上發布或列出自己的外掛
summary: 尋找並發布由社群維護的 OpenClaw 外掛
title: 社群外掛
x-i18n:
    generated_at: "2026-07-26T08:26:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6a9eb477f20da8171a35c22ea6b112d77ff4afe0878f60314c052746aef4e0ac
    source_path: plugins/community.md
    workflow: 16
---

社群外掛是第三方套件，可透過頻道、工具、提供者、掛鉤或其他功能擴充 OpenClaw。請使用 [ClawHub](/zh-TW/clawhub) 作為探索公開社群外掛的主要管道。

## 尋找外掛

從命令列介面搜尋 ClawHub：

```bash
openclaw plugins search "calendar"
```

使用明確的來源前綴安裝 ClawHub 外掛：

```bash
openclaw plugins install clawhub:<package-name>
```

在推出過渡期間，npm 仍是受支援的直接安裝方式：

```bash
openclaw plugins install npm:<package-name>
```

如需常見的安裝、更新、檢查及解除安裝範例，請參閱[管理外掛](/zh-TW/plugins/manage-plugins)。如需完整的命令參考與來源選擇規則，請參閱 [`openclaw plugins`](/zh-TW/cli/plugins)。

## 發布外掛

在 ClawHub 上發布公開社群外掛，讓 OpenClaw 使用者能夠探索及安裝。ClawHub 負責即時套件清單、版本發布記錄、掃描狀態及安裝提示；文件不維護靜態的第三方外掛目錄。

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

發布前，請確認外掛具有套件中繼資料、外掛資訊清單、設定文件，以及明確的維護負責人。ClawHub 會在建立版本前驗證擁有者範圍、套件名稱、版本、檔案限制及來源中繼資料，接著在審查與驗證完成前，讓新版本不會顯示於一般安裝及下載介面。

發布前檢查清單：

| 要求                 | 原因                                                |
| -------------------- | --------------------------------------------------- |
| 已發布至 ClawHub     | 使用者需要 `openclaw plugins install` 提示才能運作 |
| 公開 GitHub 儲存庫   | 來源審查、問題追蹤及透明度                          |
| 設定與使用文件       | 使用者需要知道如何進行設定                          |
| 積極維護             | 近期有更新或能即時回應問題                          |

完整發布規範：

- [ClawHub 發布](/zh-TW/clawhub/publishing) - 擁有者、範圍、版本發布、審查、套件驗證及套件移轉
- [建置外掛](/zh-TW/plugins/building-plugins) - 外掛套件結構與首次發布工作流程
- [外掛資訊清單](/zh-TW/plugins/manifest) - 原生外掛資訊清單欄位

## 相關內容

- [外掛](/zh-TW/tools/plugin) - 安裝、設定、重新啟動及疑難排解
- [管理外掛](/zh-TW/plugins/manage-plugins) - 命令範例
- [ClawHub 發布](/zh-TW/clawhub/publishing) - 發布與版本發布規則
