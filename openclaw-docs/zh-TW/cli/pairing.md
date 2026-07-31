---
read_when:
    - 你正在使用配對模式的私訊，因此需要核准傳送者
summary: '`openclaw pairing` 的命令列介面參考（核准／列出配對請求）'
title: 配對
x-i18n:
    generated_at: "2026-07-26T08:13:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e4c6c53f1a3eefe50b4b7a45fa535e9a05faabb50df1ba5195a7635ee13d9da0
    source_path: cli/pairing.md
    workflow: 16
---

# `openclaw pairing`

核准或檢查支援配對之頻道的私訊配對請求（僅限聊天私訊——節點／裝置配對使用 `openclaw devices`）。

相關內容：[配對流程](/zh-TW/channels/pairing)

也可在控制介面的 **設定 →
頻道 → 私訊存取請求** 下檢視相同的待處理請求。控制介面支援核准、選擇性通知
請求者，以及忽略。忽略會移除目前的請求，但不會
永久封鎖傳送者。

## 命令

```bash
openclaw pairing list telegram
openclaw pairing list --channel telegram --account work
openclaw pairing list telegram --json

openclaw pairing approve <code>
openclaw pairing approve telegram <code>
openclaw pairing approve --channel telegram --account work <code> --notify
```

## `pairing list`

列出一個頻道的待處理配對請求。

| 選項                    | 說明                                  |
| ----------------------- | ------------------------------------- |
| `[channel]`      | 位置式頻道 ID                         |
| `--channel <channel>`      | 明確指定頻道 ID                       |
| `--account <accountId>`      | 多帳號頻道的帳號 ID                   |
| `--json`      | 機器可讀的輸出                        |

若已設定多個支援配對的頻道，請以位置引數或 `--channel` 傳入頻道。只要頻道 ID 有效，擴充功能頻道也能使用。

## `pairing approve`

核准待處理的配對碼，並允許該傳送者。

用法：

- `openclaw pairing approve <channel> <code>`
- `openclaw pairing approve --channel <channel> <code>`
當只設定一個支援配對的頻道時，使用
- `openclaw pairing approve <code>`

選項：`--channel <channel>`、`--account <accountId>`、`--notify`（透過相同頻道向請求者傳送確認訊息）。

### 擁有者初始設定

若核准配對碼時 `commands.ownerAllowFrom` 為空，命令列介面也會將已核准的傳送者記錄為命令擁有者，並使用頻道範圍的項目，例如 `telegram:123456789`。這只會初始設定第一位擁有者——後續的配對核准絕不會取代或擴充 `commands.ownerAllowFrom`。控制介面不會自動套用此權限提升，而是將其顯示為另一個受 `operator.admin` 保護的核取方塊。

命令擁有者是獲准執行僅限擁有者之命令，以及核准 `/diagnostics`、`/export-session`、`/export-trajectory`、`/config` 和 exec 核准等危險動作的人類操作者帳號。配對只允許傳送者與代理程式交談；除了這項一次性的初始設定之外，配對本身不會授予擁有者權限。

若你在此初始設定機制存在之前已核准傳送者，請執行 `openclaw doctor`；未設定命令擁有者時，它會顯示警告，並顯示用於修正問題的確切 `openclaw config set commands.ownerAllowFrom ...` 命令。

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [頻道配對](/zh-TW/channels/pairing)
