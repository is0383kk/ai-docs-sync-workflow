---
read_when:
    - 你想將 OpenClaw 與主要的 macOS 環境隔離
    - 你想要在沙箱中整合 iMessage
    - 你想要一個可重設且可複製的 macOS 環境
    - 你想比較本機與託管的 macOS 虛擬機器選項
summary: 當你需要隔離環境或 iMessage 時，請在沙箱化的 macOS 虛擬機器（本機或託管）中執行 OpenClaw
title: macOS 虛擬機器
x-i18n:
    generated_at: "2026-07-26T07:58:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7e6b963faaf40f65adce1081715bc295059b8bed278a8c71a05a86e04ad7a7a5
    source_path: install/macos-vm.md
    workflow: 16
---

## 建議的預設選擇（適合大多數使用者）

- 使用**小型 Linux VPS**，以低成本維持閘道全天候運作。請參閱 [VPS 託管](/zh-TW/vps)。
- 如果你想要完全掌控，並使用**住宅 IP** 進行瀏覽器自動化，請選擇**專用硬體**（Mac mini 或 Linux 主機）。許多網站會封鎖資料中心 IP，因此在本機瀏覽通常效果更好。
- **混合方式**：將閘道放在便宜的 VPS 上，並在需要瀏覽器／UI 自動化時，將你的 Mac 連接為**節點**。請參閱[節點](/zh-TW/nodes)和[遠端閘道](/zh-TW/gateway/remote)。

只有在你明確需要 macOS 專屬功能（例如 iMessage），或想與日常使用的 Mac 嚴格隔離時，才使用 macOS VM。

## macOS VM 選項

### Apple Silicon Mac 上的本機 VM（Lume）

使用 [Lume](https://cua.ai/docs/lume)，在你現有的 Apple Silicon Mac 上，於沙箱化的 macOS VM 中執行 OpenClaw。這可提供：

- 隔離的完整 macOS 環境（主機可保持乾淨）
- 透過 `imsg` 支援 iMessage；在 Linux／Windows 上無法使用預設的本機路徑
- 透過複製 VM 立即重設
- 不需要額外的硬體或雲端費用

### 託管式 Mac 供應商（雲端）

如果你想在雲端使用 macOS，也可以使用託管式 Mac 供應商：

- [MacStadium](https://www.macstadium.com/)（託管式 Mac）
- 其他託管式 Mac 供應商也適用；請依照其 VM 與 SSH 文件操作

取得 macOS VM 的 SSH 存取權後，請繼續進行下方的[安裝 OpenClaw](#6-install-openclaw)。

## 快速流程（Lume，適合有經驗的使用者）

1. 安裝 Lume。
2. `lume create openclaw --os macos --ipsw latest`
3. 完成「設定輔助程式」，並啟用 Remote Login（SSH）。
4. `lume run openclaw --no-display`
5. 透過 SSH 登入、安裝 OpenClaw，並設定頻道。
6. 完成。

## 所需項目（Lume）

- Apple Silicon Mac（M1/M2/M3/M4）
- 主機使用 macOS Sequoia 或更新版本
- 每個 VM 約需 60 GB 可用磁碟空間
- 約 20 分鐘

## 1) 安裝 Lume

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/trycua/cua/main/libs/lume/scripts/install.sh)"
```

如果 `~/.local/bin` 不在你的 PATH 中：

```bash
echo 'export PATH="$PATH:$HOME/.local/bin"' >> ~/.zshrc && source ~/.zshrc
```

驗證：

```bash
lume --version
```

文件：[Lume 安裝](https://cua.ai/docs/lume/guide/getting-started/installation)

## 2) 建立 macOS VM

```bash
lume create openclaw --os macos --ipsw latest
```

這會下載 macOS 並建立 VM。VNC 視窗會自動開啟。

<Note>
視你的連線速度而定，下載可能需要一些時間。
</Note>

## 3) 完成「設定輔助程式」

在 VNC 視窗中：

1. 選取語言和地區。
2. 略過 Apple ID（如果之後想使用 iMessage，也可以登入）。
3. 建立使用者帳號（請記住使用者名稱和密碼）。
4. 略過所有選用功能。

設定完成後：

1. 啟用 SSH：System Settings -> General -> Sharing，啟用 "Remote Login"。
2. 若要以無頭模式使用 VM，請啟用自動登入：System Settings -> Users & Groups，選取 "Automatically log in as:"，再選擇 VM 使用者。

## 4) 取得 VM IP 位址

```bash
lume get openclaw
```

尋找 IP 位址（通常是 `192.168.64.x`）。

## 5) 透過 SSH 登入 VM

```bash
ssh youruser@192.168.64.X
```

將 `youruser` 替換成你建立的帳號，並將 IP 替換成 VM 的 IP。

## 6) 安裝 OpenClaw

在 VM 內：

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

依照初始設定提示，設定你的模型供應商（Anthropic、OpenAI 等）。

## 7) 設定頻道

編輯設定檔：

```bash
nano ~/.openclaw/openclaw.json
```

新增你的頻道：

```json5
{
  channels: {
    telegram: {
      botToken: "YOUR_BOT_TOKEN",
    },
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551234567"],
    },
  },
}
```

接著登入 WhatsApp（掃描 QR Code）：

```bash
openclaw channels login
```

## 8) 以無頭模式執行 VM

停止 VM，並在不顯示畫面的情況下重新啟動：

```bash
lume stop openclaw
lume run openclaw --no-display
```

VM 會在背景執行；OpenClaw 的常駐程式會讓閘道持續運作。若要檢查狀態：

```bash
ssh youruser@192.168.64.X "openclaw status"
```

## 額外功能：iMessage 整合

這是在 macOS 上執行的最大亮點。使用 [iMessage](/zh-TW/channels/imessage) 搭配 `imsg`，將「訊息」加入 OpenClaw。

在 VM 內：

1. 登入「訊息」。
2. 安裝 `imsg`。
3. 為執行 OpenClaw／`imsg` 的程序授予「完整磁碟存取權」和「自動化」權限。
4. 使用 `imsg rpc --help` 驗證 RPC 支援。

新增至你的 OpenClaw 設定：

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "imsg",
      dbPath: "~/Library/Messages/chat.db",
    },
  },
}
```

重新啟動閘道。你的代理程式現在可以傳送和接收 iMessage。完整設定詳細資訊：[iMessage 頻道](/zh-TW/channels/imessage)。

## 儲存黃金映像

進一步自訂前，請為乾淨狀態建立快照：

```bash
lume stop openclaw
lume clone openclaw openclaw-golden
```

隨時重設：

```bash
lume stop openclaw && lume delete openclaw
lume clone openclaw-golden openclaw
lume run openclaw --no-display
```

## 執行 24/7

若要讓 VM 持續執行：

- 讓 Mac 保持接上電源
- 在 System Settings -> Energy Saver 中停用睡眠
- 視需要使用 `caffeinate`

若要真正全天候運作，請考慮使用專用 Mac mini 或小型 VPS。請參閱 [VPS 託管](/zh-TW/vps)。

## 疑難排解

| 問題                     | 解決方案                                                                            |
| ------------------------ | ----------------------------------------------------------------------------------- |
| 無法透過 SSH 登入 VM     | 檢查 VM 的 System Settings 中是否已啟用 "Remote Login"                              |
| 未顯示 VM IP             | 等待 VM 完全啟動，再次執行 `lume get openclaw`                                       |
| 找不到 Lume 命令         | 將 `~/.local/bin` 新增至你的 PATH                                                |
| 無法掃描 WhatsApp QR Code | 執行 `openclaw channels login` 時，確認你登入的是 VM（而非主機）                           |

## 相關文件

- [VPS 託管](/zh-TW/vps)
- [節點](/zh-TW/nodes)
- [遠端閘道](/zh-TW/gateway/remote)
- [iMessage 頻道](/zh-TW/channels/imessage)
- [Lume 快速入門](https://cua.ai/docs/lume/guide/getting-started/quickstart)
- [Lume 命令列介面參考](https://cua.ai/docs/lume/reference/cli-reference)
- [無人值守 VM 設定](https://cua.ai/docs/lume/guide/fundamentals/unattended-setup)（進階）
- [Docker 沙箱隔離](/zh-TW/install/docker)（替代的隔離方式）
