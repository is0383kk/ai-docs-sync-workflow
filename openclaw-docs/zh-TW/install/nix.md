---
read_when:
    - 你希望安裝過程可重現且可復原
    - 你已經在使用 Nix/NixOS/Home Manager
    - 你希望所有項目都固定版本，並以宣告式方式管理
summary: 使用 Nix 以宣告式方式安裝 OpenClaw
title: Nix
x-i18n:
    generated_at: "2026-07-26T08:36:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6f74e259ec3d909c73d9184db24d236135db04c29c2e7fab9be9e6fa7f98ba91
    source_path: install/nix.md
    workflow: 16
---

使用第一方、功能齊全的 Home Manager 模組 **[nix-openclaw](https://github.com/openclaw/nix-openclaw)**，以宣告式方式安裝 OpenClaw。

<Info>
[nix-openclaw](https://github.com/openclaw/nix-openclaw) 儲存庫是 Nix 安裝方式的權威來源。本頁提供快速概覽。
</Info>

## 你會獲得什麼

- 閘道 + macOS App + 工具（whisper、spotify、攝影機），全部固定版本
- 重新開機後仍會執行的 launchd 服務
- 採用宣告式設定的外掛系統
- 即時回復：`home-manager switch --rollback`

## 快速開始

<Steps>
  <Step title="安裝 Determinate Nix">
    如果尚未安裝 Nix，請依照 [Determinate Nix 安裝程式](https://github.com/DeterminateSystems/nix-installer)的指示操作。
  </Step>
  <Step title="建立本機 flake">
    使用 nix-openclaw 儲存庫中以代理程式為優先的範本：
    ```bash
    mkdir -p ~/code/openclaw-local
    # 從 nix-openclaw 儲存庫複製 templates/agent-first/flake.nix
    ```
  </Step>
  <Step title="設定機密資訊">
    設定訊息傳遞機器人的權杖和模型供應商 API 金鑰。使用位於 `~/.secrets/` 的純文字檔案即可。
  </Step>
  <Step title="填入範本預留位置並切換">
    ```bash
    home-manager switch
    ```
  </Step>
  <Step title="驗證">
    確認 launchd 服務正在執行，且機器人會回應訊息。
  </Step>
</Steps>

如需完整的模組選項和範例，請參閱 [nix-openclaw README](https://github.com/openclaw/nix-openclaw)。

## Nix 模式的執行階段行為

設定 `OPENCLAW_NIX_MODE=1` 後（使用 nix-openclaw 時會自動設定），OpenClaw 會針對由 Nix 管理的安裝進入確定性模式。其他 Nix 套件也可以設定相同模式；nix-openclaw 是第一方參考實作。

你也可以手動設定：

```bash
export OPENCLAW_NIX_MODE=1
```

在 macOS 上，GUI App 不會繼承 Shell 環境變數。請改用 `defaults` 啟用 Nix 模式：

```bash
defaults write ai.openclaw.mac openclaw.nixMode -bool true
```

### Nix 模式中的變更

- 停用自動安裝和自我修改流程。
- `openclaw.json` 會視為不可變更。啟動時衍生的預設值只會保留在執行階段，而設定寫入工具（設定、初始設定、會修改內容的 `openclaw update`、外掛安裝／更新／解除安裝／啟用、`doctor --fix`、`doctor --generate-gateway-token`、`openclaw config set`）會拒絕編輯該檔案。
- 請改為編輯 Nix 來源。使用 nix-openclaw 時，請採用以代理程式為優先的[快速開始](https://github.com/openclaw/nix-openclaw#quick-start)，並在 `programs.openclaw.config` 或 `instances.<name>.config` 下設定組態。
- 缺少相依套件時，會顯示 Nix 專用的修復訊息。
- UI 會顯示唯讀的 Nix 模式橫幅。

### 設定與狀態路徑

OpenClaw 從 `OPENCLAW_CONFIG_PATH` 讀取 JSON5 設定，並將可變動資料儲存在 `OPENCLAW_STATE_DIR`。在 Nix 下，請將這些位置明確設為由 Nix 管理的路徑，讓執行階段狀態和設定不會存入不可變更的 Store。

| 變數                   | 預設值                                  |
| ---------------------- | --------------------------------------- |
| `OPENCLAW_HOME`        | `HOME` / `USERPROFILE` / `os.homedir()` |
| `OPENCLAW_STATE_DIR`   | `~/.openclaw`                           |
| `OPENCLAW_CONFIG_PATH` | `$OPENCLAW_STATE_DIR/openclaw.json`     |

### 服務 PATH 探索

launchd/systemd 閘道服務會自動探索 Nix 設定檔中的二進位檔，讓外掛和工具在 Shell 中呼叫透過 `nix` 安裝的可執行檔時，不必手動設定 PATH 也能運作：

- 設定 `NIX_PROFILES` 時，每個項目都會依從右至左的優先順序加入服務的 PATH（與 Nix Shell 的優先順序一致：最右側優先）。
- 未設定 `NIX_PROFILES` 時，會加入 `~/.nix-profile/bin` 作為備援。

這適用於 macOS launchd 和 Linux systemd 服務環境。

## 相關內容

<CardGroup cols={2}>
  <Card title="nix-openclaw" href="https://github.com/openclaw/nix-openclaw" icon="arrow-up-right-from-square">
    權威的 Home Manager 模組和完整設定指南。
  </Card>
  <Card title="設定精靈" href="/zh-TW/start/wizard" icon="wand-magic-sparkles">
    非 Nix 的命令列介面設定操作指南。
  </Card>
  <Card title="Docker" href="/zh-TW/install/docker" icon="docker">
    作為非 Nix 替代方案的容器化設定方式。
  </Card>
  <Card title="更新" href="/zh-TW/install/updating" icon="arrow-up-right-from-square">
    隨套件一併更新由 Home Manager 管理的安裝。
  </Card>
</CardGroup>
