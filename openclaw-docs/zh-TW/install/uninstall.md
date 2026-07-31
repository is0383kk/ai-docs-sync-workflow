---
read_when:
    - 你想要從電腦上移除 OpenClaw
    - 解除安裝後，閘道服務仍在執行
summary: 完整解除安裝 OpenClaw（命令列介面、服務、狀態、工作區）
title: 解除安裝
x-i18n:
    generated_at: "2026-07-26T07:23:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 84f01dc11defe6f19c89232375e48bad383b2e71379f47f43e759d3d7bb908b5
    source_path: install/uninstall.md
    workflow: 16
---

兩種方式：

- 如果仍安裝著 `openclaw`，請使用**簡易方式**。
- 如果命令列介面已移除，但服務仍在執行，請**手動移除服務**。

## 簡易方式（命令列介面仍安裝著）

建議：使用內建解除安裝程式：

```bash
openclaw uninstall
```

移除狀態資料時，會保留已設定的工作區目錄，除非你也選取 `--workspace`。

預覽將移除的項目（安全）：

```bash
openclaw uninstall --dry-run --all
```

非互動模式（自動化 / npx）。請謹慎使用，且僅在確認範圍後執行：

```bash
openclaw uninstall --all --yes --non-interactive
npx -y openclaw uninstall --all --yes --non-interactive
```

旗標：`--service`、`--state`、`--workspace`、`--app` 分別選取個別範圍；`--all` 則選取全部四個範圍。

手動步驟（結果相同）：

1. 停止閘道服務：

```bash
openclaw gateway stop
```

2. 解除安裝閘道服務（launchd/systemd/schtasks）：

```bash
openclaw gateway uninstall
```

3. 刪除狀態資料與設定：

```bash
rm -rf "${OPENCLAW_STATE_DIR:-$HOME/.openclaw}"
```

如果你將 `OPENCLAW_CONFIG_PATH` 設為狀態目錄以外的自訂位置，也請刪除該檔案。
如果你想保留狀態目錄內的工作區（例如 `~/.openclaw/workspace`），請先將它移到其他位置，再執行 `rm -rf`；或是選擇性地刪除狀態目錄的內容。

4. 刪除你的工作區（選用，會移除代理程式檔案）：

```bash
rm -rf ~/.openclaw/workspace
```

5. 移除命令列介面安裝項目（選擇你使用的安裝方式）：

```bash
npm rm -g openclaw
pnpm remove -g openclaw
bun remove -g openclaw
```

6. 如果你安裝了 macOS 應用程式：

```bash
rm -rf /Applications/OpenClaw.app
```

注意事項：

- 如果你使用了設定檔（`--profile` / `OPENCLAW_PROFILE`），請針對每個狀態目錄重複步驟 3（預設為 `~/.openclaw-<profile>`）。
- 在遠端模式中，狀態目錄位於**閘道主機**上，因此也請在該主機上執行步驟 1 至 4。

## 手動移除服務（未安裝命令列介面）

如果閘道服務持續執行，但缺少 `openclaw`，請使用此方式。

### macOS（launchd）

預設標籤為 `ai.openclaw.gateway`（使用設定檔時則為 `ai.openclaw.<profile>`）：

```bash
launchctl bootout gui/$UID/ai.openclaw.gateway
rm -f ~/Library/LaunchAgents/ai.openclaw.gateway.plist
```

如果你使用了設定檔，請將標籤和 plist 名稱替換為 `ai.openclaw.<profile>`。

### Linux（systemd 使用者單元）

預設單元名稱為 `openclaw-gateway.service`（或 `openclaw-gateway-<profile>.service`）。從非常舊的安裝版本升級的機器上，重新命名前的 `clawdbot-gateway.service` 單元可能仍然存在；`openclaw uninstall` / `openclaw gateway uninstall` 會自動偵測並移除它。

```bash
systemctl --user disable --now openclaw-gateway.service
rm -f ~/.config/systemd/user/openclaw-gateway.service
systemctl --user daemon-reload
```

### Windows（排定的工作）

預設工作名稱為 `OpenClaw Gateway`（或 `OpenClaw Gateway (<profile>)`）。
該工作會啟動位於狀態目錄下、不顯示視窗的 `gateway.vbs` 指令碼，而該指令碼接著會
執行 `gateway.cmd`；請將兩者都移除。

```powershell
schtasks /Delete /F /TN "OpenClaw Gateway"
Remove-Item -Force "$env:USERPROFILE\.openclaw\gateway.cmd" -ErrorAction SilentlyContinue
Remove-Item -Force "$env:USERPROFILE\.openclaw\gateway.vbs" -ErrorAction SilentlyContinue
```

如果你使用了設定檔，請刪除對應的工作名稱，以及 `~\.openclaw-<profile>` 下的 `gateway.cmd` /
`gateway.vbs` 檔案。

## 一般安裝與原始碼簽出

### 一般安裝（install.sh / npm / pnpm / bun）

如果你使用了 `https://openclaw.ai/install.sh` 或 `install.ps1`，命令列介面是透過 `npm install -g openclaw@latest` 安裝的。
請使用 `npm rm -g openclaw` 移除（如果你是透過其他方式安裝，則使用 `pnpm remove -g` / `bun remove -g`）。

### 原始碼簽出（git clone）

如果你是從儲存庫簽出目錄執行（`git clone` + `openclaw ...` / `bun run openclaw ...`）：

1. 刪除儲存庫**之前**，請先解除安裝閘道服務（使用上述簡易方式或手動移除服務）。
2. 刪除儲存庫目錄。
3. 依照上述方式移除狀態資料與工作區。

## 相關內容

- [安裝概覽](/zh-TW/install)
- [遷移指南](/zh-TW/install/migrating)
