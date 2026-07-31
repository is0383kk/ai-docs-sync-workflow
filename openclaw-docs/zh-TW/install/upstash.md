---
read_when:
    - 將 OpenClaw 部署至 Upstash Box
    - 你想要一個用於 OpenClaw 的代管 Linux 環境，並透過 SSH 通道存取儀表板
summary: 在 Upstash Box 上託管 OpenClaw，並使用持續連線與 SSH 通道存取
title: Upstash Box
x-i18n:
    generated_at: "2026-07-26T07:59:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 29232c43e0e4940b7445ab8896c9ccd3e81d0fdbdd522d7f50cb8c8057ac18f0
    source_path: install/upstash.md
    workflow: 16
---

在 Upstash Box（具備保持運作生命週期支援的受管理 Linux 環境）上執行持續運作的 OpenClaw 閘道。

使用 SSH 通道存取儀表板。請勿將閘道連接埠直接暴露於公用網際網路。

## 先決條件

- Upstash 帳戶
- 保持運作的 Upstash Box
- 本機上的 SSH 用戶端

## 建立 Box

在 Upstash Console 中建立保持運作的 Box。記下 Box ID（例如
`right-flamingo-14486`）和你的 Box API 金鑰。

Upstash 在以下頁面維護其最新的 OpenClaw Box 操作指南：
[OpenClaw 設定](https://upstash.com/docs/box/guides/openclaw-setup)。

## 使用 SSH 通道連線

將 OpenClaw 儀表板連接埠轉送至本機。系統提示時，使用你的 Box API 金鑰
作為 SSH 密碼：

```bash
ssh -o ServerAliveInterval=15 -o ServerAliveCountMax=3 -L 18789:127.0.0.1:18789 <box-id>@us-east-1.box.upstash.com
```

保持連線選項可減少上線設定期間因閒置而中斷通道的情況。

## 安裝 OpenClaw

在 Box 內：

```bash
sudo npm install -g openclaw
```

## 執行上線設定

```bash
openclaw onboard --install-daemon
```

依照提示操作。上線設定完成後，複製儀表板 URL 和權杖。

## 啟動閘道

設定閘道以使用 Box 網路，並在背景啟動：

```bash
openclaw config set gateway.bind lan
nohup openclaw gateway > gateway.log 2>&1 &
```

在 SSH 通道保持連線時，於本機開啟儀表板 URL：

```text
http://127.0.0.1:18789/#token=<your-token>
```

## 自動重新啟動

將此命令設為 Box 初始化指令碼，以便閘道在 Box 啟動時重新啟動：

```bash
nohup openclaw gateway > gateway.log 2>&1 &
```

## 疑難排解

如果 SSH 在上線設定期間停止回應，請使用乾淨的 SSH 設定和保持連線選項
重新連線：

```bash
ssh -F /dev/null -o ControlMaster=no -o ServerAliveInterval=15 -o ServerAliveCountMax=3 -L 18789:127.0.0.1:18789 <box-id>@us-east-1.box.upstash.com
```

這會略過過時的本機 `~/.ssh/config` 設定，並讓通道在網路閒置期間
保持連線。

## 相關內容

- [遠端存取](/zh-TW/gateway/remote)
- [閘道安全性](/zh-TW/gateway/security)
- [更新 OpenClaw](/zh-TW/install/updating)
