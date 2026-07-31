---
read_when:
    - 執行或偵錯閘道程序
    - 調查單一執行個體強制機制
summary: 閘道單例防護：檔案鎖定加上 WebSocket/HTTP 綁定
title: 閘道鎖定
x-i18n:
    generated_at: "2026-07-26T07:52:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f5ac6d42c437b481c68a23a0aa4c00aeac9131acd76f3516ce3e949f325e265b
    source_path: gateway/gateway-lock.md
    workflow: 16
---

## 原因

- 只有一個閘道程序可擁有一個狀態目錄；若要執行其他閘道，請使用彼此隔離的設定檔、狀態目錄、設定與連接埠。
- 即使發生當機或 SIGKILL，也不會留下過期的鎖定檔案。
- 當另一個閘道已占用該連接埠時，立即失敗並顯示明確錯誤。

## 三層機制

啟動程序會依序透過三個步驟強制執行擁有權：

1. **狀態擁有權鎖定**會取得以標準狀態目錄為鍵的鎖定。每個閘道都會參與，包括以 `OPENCLAW_ALLOW_MULTI_GATEWAY=1` 啟動的閘道，因此具破壞性的 SQLite 維護作業不會與運作中的擁有者發生競爭。
2. **設定鎖定**會取得既有的個別設定鎖定，並記錄執行階段連接埠。多閘道模式會略過此設定單例限制，但仍保留狀態擁有權鎖定。
3. **通訊端繫結**會將 HTTP/WebSocket 接聽器（預設為 `ws://127.0.0.1:18789`）繫結為獨占 TCP 接聽器。

每一層都可能獨立失敗，並擲回各自的 `GatewayLockError`。

### 狀態與設定鎖定

- 鎖定是否仍有效，是根據記錄的 PID、平台程序啟動識別資訊（若有），以及閘道程序識別資訊來判定。經驗證的擁有者在啟動期間仍具有決定權，即使其連接埠尚未開始接聽也是如此。
- 專用的 SQLite 協調器會依序處理中繼資料檢查、過期擁有者回收及鎖定替換。若擁有該鎖定的程序當機，其獨占交易會自動釋放。
- 如果鎖定檔案不存在，或記錄的擁有者程序已結束，啟動程序會回收鎖定並繼續。
- 如果任一鎖定正被占用，啟動程序會重試最多 5 秒（預設值），然後放棄：

  ```text
  GatewayLockError("閘道已在執行（pid <pid>）；鎖定在 <ms>ms 後逾時")
  ```

### 通訊端繫結

- 發生 `EADDRINUSE` 時，啟動程序會以 500ms 的間隔重試繫結最多 20 次（總計約 10 秒），以等待最近結束的程序所造成的 `TIME_WAIT` 時段結束。
- 如果重試後連接埠仍在使用中：

  ```text
  GatewayLockError("另一個閘道執行個體已在 ws://127.0.0.1:<port> 上接聽")
  ```

- 其他繫結失敗：

  ```text
  GatewayLockError("無法在 ws://127.0.0.1:<port> 上繫結閘道通訊端：<cause>")
  ```

關閉時，閘道會關閉 HTTP/WebSocket 伺服器，並移除其狀態與設定鎖定檔案。

## 操作注意事項

- 如果連接埠被其他非閘道程序占用，錯誤訊息也會相同；請釋放該連接埠，或使用 `openclaw gateway --port <port>` 選擇另一個連接埠。
- `OPENCLAW_ALLOW_MULTI_GATEWAY=1` 允許多個設定／執行階段執行個體，但不允許共用可變狀態。每個執行個體仍需要唯一的 `OPENCLAW_STATE_DIR`。
- 在服務監督程式下，遇到上述任一錯誤的新閘道程序會先探測現有程序上的 `/healthz`。如果該程序運作正常，新程序會讓其繼續掌控，而不會失敗。在 systemd 上，新程序會以代碼 `78` 結束；單元的 `RestartPreventExitStatus=78` 會防止 `Restart=always` 因鎖定或 `EADDRINUSE` 衝突而循環。如果現有程序始終未恢復正常，健康狀態探測的重試會受時間限制，之後啟動程序會以上述鎖定錯誤失敗，而不會無限循環。
- macOS 應用程式會在產生閘道程序前保留自己的輕量級 PID 防護；上述檔案鎖定與通訊端繫結才是實際的執行階段強制機制。

## 相關內容

- [多個閘道](/zh-TW/gateway/multiple-gateways) - 使用唯一連接埠執行多個執行個體
- [疑難排解](/zh-TW/gateway/troubleshooting) - 診斷 `EADDRINUSE` 與連接埠衝突
