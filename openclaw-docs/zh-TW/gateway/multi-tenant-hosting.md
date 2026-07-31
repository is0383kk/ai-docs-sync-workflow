---
doc-schema-version: 1
read_when:
    - 你正為多位使用者或多個組織託管 OpenClaw
    - 你需要為租戶工作負載選擇隔離邊界
summary: 將多個租戶信任網域託管為每個租戶一個隔離的 OpenClaw 閘道單元
title: 多租戶託管
x-i18n:
    generated_at: "2026-07-26T07:53:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 383d32331b45d40db6fb4ff8242dd9a3cf8898a3ccab19f0372cd06bbd83fc05
    source_path: gateway/multi-tenant-hosting.md
    workflow: 16
---

# 多租戶託管

OpenClaw 的預設安全模型是每個閘道各自有一個受信任的操作人員邊界，而不是在單一共用閘道內提供敵對多租戶隔離。因此，若要託管不共用信任邊界的使用者或組織，就必須為每個租戶執行一個獨立且完整的 OpenClaw 執行個體。

`openclaw fleet` 將每個隔離的執行個體稱為一個**單元**。單元是在強化容器中執行的完整閘道，擁有自己的狀態、認證資訊、工作區、頻道帳號、權杖，以及僅限回環介面的主機連接埠。

Fleet 為**實驗性功能**：其命令、旗標和容器設定檔可能在不同版本間變更，且不提供棄用過渡期。

Fleet 已在 Linux 和 macOS 主機上測試。目前尚未測試 Windows 主機。

## 為何每個租戶都需要一個單元

單一閘道內通過驗證的操作人員具有受信任的控制平面角色。工作階段 ID 用於選擇路由；它們不會授權租戶彼此存取。代理程式沙箱可以降低不受信任內容和工具執行所造成的影響，但無法將單一共用閘道轉變為租戶授權邊界。

每個租戶使用一個單元，讓每個信任網域都有獨立的閘道程序、容器、持久狀態樹和閘道認證資訊。這遵循[閘道安全模型](/zh-TW/gateway/security)：請勿將互不信任的使用者共同置於同一個 OpenClaw 程序或作業系統使用者下。

## 架構

Fleet 命令列介面是主機端的生命週期監督程式。它會將單元記錄在 OpenClaw 狀態資料庫中，並要求本機 Docker 或 Podman 執行階段建立、檢查、啟動、停止、取代及移除其容器。由於 Fleet 的繫結路徑和回環 URL 屬於本機主機，因此不支援遠端執行階段端點。Fleet 不會代理租戶訊息，也不會在單元之間新增共用的應用程式層級資料路徑。

每個單元都會在各自的使用者定義橋接網路上執行官方 `ghcr.io/openclaw/openclaw` 映像檔。分離的橋接網路可防止單元之間透過容器 IP 直接傳輸流量，同時保留供提供者和頻道使用的對外 NAT 存取。預設不限制對外輸出流量。Podman 單元可以使用 `--network internal` 封鎖對外輸出流量，同時保留已發布的回環閘道連接埠。Docker 內部網路會破壞該已發布連接埠，因此 Fleet 會拒絕這種組合；請改用主機防火牆規則（例如 `DOCKER-USER` 鏈）強制執行 Docker 對外輸出政策。單元閘道會在容器內接聽連接埠 `18789`，而執行階段只會將其發布至主機上的 `127.0.0.1:<allocated-port>`。需要遠端存取時，操作人員可以在該回環端點前方部署經核准的反向 Proxy、SSH 通道或 tailnet。

持久閘道狀態來自 `<state-dir>/fleet/cells/<tenant>/`，並掛載於 `/home/node/.openclaw`。驗證設定檔加密金鑰來自獨立的 `<state-dir>/fleet/auth-profile-secrets/<tenant>/` 主機路徑，並掛載於 `/home/node/.config/openclaw`，與官方的 [Docker 持久化配置](/zh-TW/install/docker#storage-and-persistence)一致。該金鑰不會巢狀放置於一般狀態掛載點下。每個租戶的頻道帳號都會在其所屬單元內終止；Fleet 不提供共用頻道帳號或輸入訊息路由器。

官方映像檔預設使用 UID 1000 的非 root `node` 使用者。Fleet 使用與主機相容的使用者對應，確保私有繫結掛載點保持可寫入：Podman 使用 `keep-id`，以 root 模式執行的 Docker 使用叫用者的非 root 身分，而無 root Docker 則將容器 root 對應至不具特權的常駐程式使用者。當主機啟用 SELinux 時，Docker 和 Podman 會套用私有 `:Z` 重新標記。容器設定檔會避開具特權的主機功能，並適合無 root 操作，但無 root 操作是主機執行階段的選擇與先決條件，並非 Fleet 會自動啟用的功能。

## 信任邊界

多租戶機制會保護租戶彼此隔離。每個租戶都信任 Fleet 操作人員和主機。抵禦遭入侵的主機並非設計目標。

這表示主機管理員可以檢查容器設定和環境、讀取已掛載的單元資料、取代映像檔或進入容器。管理員可透過 Docker 或 Podman 檢查功能查看閘道權杖以及透過 `--env` 傳遞的值。請據此使用主機控制措施、管理存取政策、監控、備份及經核准的秘密管理工具。

基準設定可避免意外暴露萬用字元網路，並移除常見的容器權限提升機制，但無法使不受信任的主機變得安全。

## 隔離層級

請選擇符合所託管租戶需求的邊界：

1. **強化容器基準。** Fleet 會捨棄所有 Linux 權能、啟用 `no-new-privileges`、套用 PID、記憶體、CPU 和選用的可寫入層磁碟限制、使用獨立的持久掛載點和各單元專用網路，並且僅發布至主機回環介面。橋接網路不限制對外輸出流量；當單元不得主動建立對外連線時，請使用 Podman `--network internal` 或 Docker 主機防火牆政策。這是適用於信任操作人員和主機之租戶的預設設定檔。
2. **更強的容器或 VM 隔離。** 對於風險較高的工作負載，請將 Docker 或 Podman 設定為使用更強的 OCI 隔離執行階段，例如 gVisor 或 Kata Containers，或將單元置於 microVM 中。這屬於執行階段或基礎架構設定；Fleet 的 `--runtime docker|podman` 選項只會選擇容器命令列介面，而非 OCI 隔離後端。請參閱 Docker 的[替代容器執行階段](https://docs.docker.com/engine/daemon/alternative-runtimes/)和 [Docker VM 執行階段指南](/zh-TW/install/docker-vm-runtime)。
3. **為敵對租戶使用獨立機器。** 請勿將敵對租戶共同置於同一個 OpenClaw 程序或作業系統使用者下。當租戶不信任同一位主機操作人員或需要更強的管理邊界時，請使用具有獨立執行階段管理的不同 VM 或實體主機。

此隔離層級中的任何一層都不會改變 OpenClaw 應用程式的信任模型：一個閘道仍然是一個受信任的操作人員網域。

## 快速開始

建立一個單元。此命令只會顯示一次產生的閘道權杖，因此請立即儲存：

```bash
openclaw fleet create acme
```

在 Fleet 主機上開啟回報的 `http://127.0.0.1:<port>` URL，使用該租戶的權杖進行驗證，然後在單元內設定提供者認證資訊和頻道帳號。

檢查容器狀態和閘道存活情形：

```bash
openclaw fleet status acme
```

升級時保留主機連接埠、已掛載資料、資源設定檔、使用者提供的環境和閘道權杖：

```bash
openclaw fleet upgrade acme
```

移除容器和登錄資料列，同時保留租戶資料：

```bash
openclaw fleet rm acme --force
```

若也要刪除持久租戶資料，請加入 `--purge-data`。清除操作需要 `--force`、無法復原，且會在刪除任何內容前執行解析路徑包含範圍檢查：

```bash
openclaw fleet rm acme --purge-data --force
```

如需所有命令和選項，請參閱 [`openclaw fleet` 命令列介面參考文件](/zh-TW/cli/fleet)。

## 目前範圍

Fleet 不提供以下介面：

- 共用頻道帳號或共用輸入路由器
- 以精簡的每租戶主機程序取代完整的 OpenClaw 執行個體
- 由單一監督程式管理的遠端單元主機
- 租戶自助服務入口網站、計費平面或委派管理 UI

這些功能需要明確的身分、路由、授權和故障網域合約。請勿透過在租戶之間共用單一閘道或其認證資訊來近似實作這些功能。Fleet 是單一主機的生命週期監督程式；跨機器且由身分治理的 Fleet 需要獨立的控制平面層。

## 相關內容

- [`openclaw fleet`](/zh-TW/cli/fleet)
- [閘道安全性](/zh-TW/gateway/security)
- [多個閘道](/zh-TW/gateway/multiple-gateways)
- [Docker](/zh-TW/install/docker)
- [Podman](/zh-TW/install/podman)
