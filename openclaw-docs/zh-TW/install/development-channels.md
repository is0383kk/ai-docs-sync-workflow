---
read_when:
    - 你想要在 stable/extended-stable/beta/dev 之間切換
    - 你想要固定使用特定版本、標籤或 SHA
    - 你正在標記或發布預發行版本
sidebarTitle: Release Channels
summary: 穩定版、延伸穩定版、Beta 版與開發版通道：語意、切換、版本鎖定與標記
title: 發行通道
x-i18n:
    generated_at: "2026-07-26T08:20:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a99e31f5121c0ab8696e638cb10a7ce16e8f32c81e4b2bef1f703eef71191494
    source_path: install/development-channels.md
    workflow: 16
---

OpenClaw 提供四個更新頻道：

- **穩定版**：npm dist-tag `latest`。建議大多數使用者使用。
- **延伸穩定版**：npm dist-tag `extended-stable`。這是全新的落後一個
  支援月份套件頻道。它僅提供套件，且只能以前景方式安裝。啟用
  `update.checkOnStart` 時，已儲存的選擇會收到唯讀更新提示，但絕不會自動套用。
- **測試版**：npm dist-tag `beta`。當 `beta` 不存在
  或比目前穩定版本更舊時，會回退至 `latest`。
- **開發版**：`main`（git）的移動中最新提交。發布時使用 npm dist-tag `dev`。`main`
  用於實驗與積極開發；可能包含未完成的功能或破壞性變更。請勿將其用於正式環境閘道。

穩定版建置通常會先發布至 **測試版**，在該處通過驗證後，再升級為
**latest**，且不變更版本號。維護者也可以直接發布至
`latest`。dist-tag 是 npm 安裝的真實依據。

## 切換頻道

```bash
openclaw update --channel stable
openclaw update --channel extended-stable
openclaw update --channel beta
openclaw update --channel dev
```

`--channel` 會將選擇保存至設定中的 `update.channel`，並控制兩種
安裝路徑：

| 頻道              | npm／套件安裝                                                                                                                                                                           | git 安裝                                                                                                                                                           |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `stable`          | dist-tag `latest`                                                                                                                                                                      | 最新的穩定版 git 標籤（排除 `-alpha.N`、`-beta.N`、`-rc.N`、`-dev.N`、`-next.N`、`-preview.N`、`-canary.N`、`-nightly.N` 及其他具名預發行版本後綴） |
| `extended-stable` | 解析公開 npm `extended-stable` 選擇器、驗證確切選取的套件，並安裝該確切版本。採取失敗關閉，不會回退至 `latest`、`beta` 或 `dev`。 | 不支援：OpenClaw 會保持簽出內容不變，並要求你改用套件安裝                                                                                                           |
| `beta`            | dist-tag `beta`；當 `beta` 不存在或較舊時，回退至 `latest`                                                                                                              | 最新的測試版 git 標籤；當測試版不存在或較舊時，回退至最新的穩定版 git 標籤                                                                                         |
| `dev`             | dist-tag `dev`（很少使用；大多數開發版使用者會執行 git 安裝）                                                                                                                           | 擷取內容、將簽出內容重定基底至上游 `main` 分支、進行建置，並重新安裝全域命令列介面                                                                      |

對於 `dev` git 安裝，預設簽出為 `~/openclaw`（設定
`OPENCLAW_HOME` 時則為 `$OPENCLAW_HOME/openclaw`）；可使用
`OPENCLAW_GIT_DIR` 覆寫。

<Tip>
若要同時保留穩定版與開發版，請使用兩個獨立的簽出，並將每個閘道分別指向各自的簽出。
</Tip>

## 單次指定版本或標籤

使用 `--tag` 可針對單次更新指定特定 dist-tag、版本或套件規格，且**不會**
變更已保存的頻道：

```bash
# 安裝特定版本
openclaw update --tag 2026.4.1-beta.1

# 從 beta dist-tag 安裝（單次，不保存）
openclaw update --tag beta

# 切換至移動中的 GitHub main 簽出（永久）
openclaw update --channel dev

# 安裝特定 npm 套件規格
openclaw update --tag openclaw@2026.4.1-beta.1

# 從 GitHub main 安裝一次，但不保存頻道
openclaw update --tag main
```

注意事項：

- `--tag` 僅適用於**套件（npm）安裝**；git 安裝會忽略它。
- 標籤不會被保存；下一次 `openclaw update` 會使用設定的
  頻道。
- `--tag main` 會在該次執行中對應至與 npm 相容的規格 `github:openclaw/openclaw#main`。
  若要永久使用移動中的 `main` 安裝，請使用
  `openclaw update --channel dev`（套件安裝會切換至 git 簽出），
  或使用安裝程式的 git 方法重新安裝：
  `curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git --version main`。
  npm 安裝路徑會直接拒絕 GitHub／git 來源目標，並改為引導你使用 git 方法。
- 降級保護：若目標版本比目前版本更舊，OpenClaw 會提示確認（可使用 `--yes` 略過）。
- 延伸穩定版一律使用經驗證的確切套件目標。它不是
  `--tag extended-stable` 的單次別名，且 `--tag` 不能與生效中的延伸穩定版頻道併用。
- `--channel beta` 與 `--tag beta` 不同：當測試版不存在或較舊時，頻道流程可以回退
  至穩定版／latest，而 `--tag beta` 在該次執行中一律
  指向原始 `beta` dist-tag。

## 試執行

預覽 `openclaw update` 將執行的操作，而不進行任何變更：

```bash
openclaw update --dry-run
openclaw update --channel beta --dry-run
openclaw update --tag 2026.4.1-beta.1 --dry-run
openclaw update --dry-run --json
```

試執行會回報生效的頻道、目標版本、規劃的動作，
以及是否需要降級確認。

## 外掛與頻道

使用 `openclaw update` 切換頻道時，也會同步外掛來源：

- `dev` 會將具有內建對應項目的已安裝外掛，切換回其
  內建（git 簽出）來源。
- `stable` 和 `beta` 會還原透過 npm 或 ClawHub 安裝的外掛
  套件。
- `extended-stable` 會將符合資格、使用空白／預設
  或 `latest` 意圖的官方 npm 外掛解析為已安裝核心的確切版本。它不會在執行階段
  查詢外掛的 `@extended-stable` 標籤。
- 透過 npm 安裝的外掛會在核心更新完成後更新。

## 檢查目前狀態

```bash
openclaw update status
```

顯示作用中的頻道（以及決定該頻道的來源：設定、git 標籤、
git 分支、已安裝版本或預設值）、安裝類型（git 或套件）、
目前版本及更新是否可用。

## 標記最佳實務

- 為你希望 git 簽出停駐的發行版本加上標籤：穩定版使用 `vYYYY.M.PATCH`，
  測試版使用 `vYYYY.M.PATCH-beta.N`。如
  `-alpha.N`、`-rc.N` 和 `-next.N` 等具名預發行版本後綴，不是穩定版或測試版目標。
- 為了相容性，仍會將 `vYYYY.M.PATCH-1` 和 `v1.0.1-1` 等舊版純數字穩定標籤
  識別為穩定版 git 標籤。
- 為了相容性，也會識別 `vYYYY.M.PATCH.beta.N`（以句點分隔）；
  建議使用 `-beta.N`。
- 保持標籤不可變：絕不移動或重複使用標籤。
- npm dist-tag 仍是 npm 安裝的真實依據：
  - `latest` -> 穩定版
  - `extended-stable` -> 落後一個支援月份的套件發行版
  - `beta` -> 候選建置或先發布至測試版的穩定版建置
  - `dev` -> main 快照（選用）

## macOS 應用程式可用性

測試版與開發版建置可能**不會**包含 macOS 應用程式發行版。這沒有問題：

- git 標籤與 npm dist-tag 仍可各自獨立發布。
- 請在發行說明或變更日誌中明確註明「此測試版沒有 macOS 建置」。

## 相關內容

- [更新](/zh-TW/install/updating)
- [安裝程式內部機制](/zh-TW/install/installer)
