---
read_when:
    - 更新 macOS Skills 設定介面
    - 變更 Skills 的門控或安裝行為
summary: macOS Skills 設定介面與閘道支援的狀態
title: Skills（macOS）
x-i18n:
    generated_at: "2026-07-26T07:47:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fd9d8f1190320889029335e008c3605bd4bf0194f83398cedd4ae658fd90065c
    source_path: platforms/mac/skills.md
    workflow: 16
---

macOS App 透過閘道呈現 OpenClaw Skills；它不會在本機剖析 Skills。

## 資料來源

- `skills.status`（閘道）會傳回所有 Skills，以及資格條件和缺少的需求，包括內建 Skills 的允許清單封鎖項目。
- 需求來自每個 `SKILL.md` 中的 `metadata.openclaw.requires`。

## 安裝動作

- `metadata.openclaw.install` 定義安裝選項（brew/node/go/uv/download）。
- App 會呼叫 `skills.install`，在閘道主機上執行安裝程式。
- 由操作員管理的 `security.installPolicy`（`enabled`、`targets`、`exec`）可在執行安裝程式中繼資料前，封鎖由閘道支援的 Skills 安裝。內建的危險程式碼掃描（用於安裝外掛）尚未接入 Skills 安裝流程。
- 如果所有安裝選項都是 `download`，閘道會呈現所有下載選擇。
- 否則，閘道會根據目前的安裝偏好設定（`skills.install.preferBrew`、`skills.install.nodeManager`）和主機上的二進位檔，選擇一個偏好的安裝程式：啟用 `preferBrew` 且存在 `brew` 時，優先使用 Homebrew；接著是 `uv`、設定的 node 管理程式；若 Homebrew 可用則再次選用（即使沒有 `preferBrew`）；再依序是 `go` 和 `download`。
- Node 安裝標籤會反映設定的 node 管理程式，包括 `yarn`。

## 環境變數/API 金鑰

- App 會將金鑰儲存在 `~/.openclaw/openclaw.json` 的 `skills.entries.<skillKey>` 下。
- `skills.update` 會修補 `enabled`、`apiKey` 和 `env`。

## 遠端模式

- 安裝和設定更新會在閘道主機上進行，而非本機 Mac。

## 相關內容

- [Skills](/zh-TW/tools/skills)
- [macOS App](/zh-TW/platforms/macos)
