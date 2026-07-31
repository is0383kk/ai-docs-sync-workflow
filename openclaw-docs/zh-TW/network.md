---
read_when:
    - 你需要網路架構與安全性概觀
    - 你正在偵錯本機與 tailnet 存取或配對問題
    - 你想要網路文件的標準清單
summary: 網路中樞：閘道介面、配對、探索與安全性
title: 網路
x-i18n:
    generated_at: "2026-07-26T08:37:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9751bb0fe71009455b243b109ef7ef4eda08d58f940f7dcef305800a5ed89586
    source_path: network.md
    workflow: 16
---

這個中樞頁面連結了 OpenClaw 如何在 localhost、LAN 與 tailnet 之間連線、配對及保護裝置安全的核心文件。

## 核心模型

大多數操作都經由閘道（`openclaw gateway`）進行；它是單一長時間執行的程序，負責管理頻道連線與 WebSocket 控制平面。

- **優先使用迴路介面**：閘道 WS 預設為 `ws://127.0.0.1:18789`。
  非迴路介面繫結若沒有有效的閘道驗證路徑，就會拒絕啟動：
  共享密鑰權杖／密碼驗證，或正確設定的非迴路介面
  `trusted-proxy` 部署。
- 建議**每台主機使用一個閘道**。如需隔離，請使用相互隔離的設定檔與連接埠執行多個閘道（[多個閘道](/zh-TW/gateway/multiple-gateways)）。
- **Canvas 主機**由與閘道相同的連接埠（`/__openclaw__/canvas/`、`/__openclaw__/a2ui/`）提供服務；繫結至迴路介面以外的位址時，會受到閘道驗證保護。
- **遠端存取**通常使用 SSH 通道或 Tailscale VPN（[遠端存取](/zh-TW/gateway/remote)）。

重要參考資料：

- [閘道架構](/zh-TW/concepts/architecture)
- [閘道通訊協定](/zh-TW/gateway/protocol)
- [閘道操作手冊](/zh-TW/gateway)
- [Web 介面與繫結模式](/zh-TW/web)

## 配對與身分識別

- [配對概覽（DM 與節點）](/zh-TW/channels/pairing)
- [由閘道管理的節點配對](/zh-TW/gateway/pairing)
- [裝置命令列介面（配對與權杖輪替）](/zh-TW/cli/devices)
- [配對命令列介面（DM 核准）](/zh-TW/cli/pairing)

本機信任：

- 直接的本機迴路介面連線（不含轉送／Proxy 標頭）可自動核准
  配對，讓同一主機上的使用體驗保持順暢。
- OpenClaw 也提供一條範圍有限的後端／容器本機自行連線路徑，
  供受信任的共享密鑰輔助程式流程使用。
- Tailnet 與 LAN 用戶端（包括同一主機上的 tailnet 繫結）仍需要
  明確核准配對。

## 探索與傳輸方式

- [探索與傳輸方式](/zh-TW/gateway/discovery)
- [Bonjour／mDNS](/zh-TW/gateway/bonjour)
- [遠端存取（SSH）](/zh-TW/gateway/remote)
- [Tailscale](/zh-TW/gateway/tailscale)

## 節點與傳輸方式

- [節點概覽](/zh-TW/nodes)
- [橋接通訊協定（舊版節點，歷史資料）](/zh-TW/gateway/bridge-protocol)
- [節點操作手冊：iOS](/zh-TW/platforms/ios)
- [節點操作手冊：Android](/zh-TW/platforms/android)

## 安全性

- [安全性概覽](/zh-TW/gateway/security)
- [閘道設定參考](/zh-TW/gateway/configuration)
- [疑難排解](/zh-TW/gateway/troubleshooting)
- [Doctor](/zh-TW/gateway/doctor)

## 相關內容

- [閘道操作手冊](/zh-TW/gateway)
- [遠端存取](/zh-TW/gateway/remote)
