---
read_when:
    - 你在一台機器上託管多個租戶信任網域
    - 你需要建立、檢查、升級或移除叢集單元
summary: 用於佈建與管理隔離式每租戶 OpenClaw 單元的命令列介面參考資料
title: 機群
x-i18n:
    generated_at: "2026-07-26T07:37:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: be589500e4715541f175caf0d5135a96baee4874e64c60c8b6f188ff1f70bc9f
    source_path: cli/fleet.md
    workflow: 16
---

# `openclaw fleet`

`openclaw fleet` 管理稱為 **cells** 的完整 OpenClaw 執行個體。每個 cell 都有自己的閘道、狀態、認證資訊、頻道帳號、容器，以及僅限迴送介面的主機連接埠。每個租戶信任邊界應使用一個 cell；請勿將單一共用閘道用作惡意多租戶邊界。

Fleet 為**實驗性**功能。命令名稱、旗標、輸出格式及容器設定檔可能在不同版本間變更，且不提供棄用過渡期。

Fleet 支援 Docker 和 Podman。預設映像檔為 `ghcr.io/openclaw/openclaw:latest`。

Fleet 已在 Linux 和 macOS 主機上測試。目前尚未測試 Windows 主機。

## 快速開始

```bash
openclaw fleet create acme
openclaw fleet status acme
openclaw fleet list
```

`fleet create` 會連同 cell URL 一次性顯示產生的閘道權杖。請立即儲存權杖，然後在每個租戶自己的 cell 內設定該租戶的頻道帳號。

## 租戶 ID

租戶 ID 必須符合：

```text
^[a-z0-9](?:[a-z0-9-]{0,38}[a-z0-9])?$
```

可使用 1 至 40 個小寫字母、數字及內部連字號。ID 的開頭與結尾必須是字母或數字。大寫字母、底線、斜線、句點、空白，以及 `../acme` 等路徑遊走字串均會遭到拒絕。

此 ID 會成為容器名稱的一部分：`openclaw-cell-<tenant>`。

## `fleet create`

建立並啟動 cell：

```bash
openclaw fleet create acme
```

在固定連接埠上建立 Podman cell，但不啟動：

```bash
openclaw fleet create acme \
  --runtime podman \
  --port 19125 \
  --no-start
```

重複使用 `--env` 傳入租戶專屬的環境變數：

```bash
openclaw fleet create acme \
  --env TZ=America/Los_Angeles \
  --env OPENCLAW_DISABLE_BONJOUR=1
```

環境變數鍵可使用字母、數字和底線，但不能以數字開頭。值必須為單行，因為 Fleet 會透過受保護的執行環境檔案傳遞這些值。Fleet 會拒絕嘗試覆寫[儲存空間與容器配置](#storage-and-container-layout)中列出的受管理容器路徑與閘道權杖變數。

### 建立選項

| 選項                    | 預設值                               | 說明                                                                                    |
| ------------------------- | ------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `--image <ref>`           | `ghcr.io/openclaw/openclaw:latest`    | cell 的容器映像檔。                                                                  |
| `--runtime <runtime>`     | `docker`                              | 容器命令列介面：`docker` 或 `podman`。                                                           |
| `--port <number>`         | 從 `19100` 自動配置  | 迴送主機連接埠。明確選取的連接埠不得屬於另一個已註冊的 cell。    |
| `--memory <value>`        | `2g`                                  | 使用 Docker/Podman 語法表示的容器記憶體限制。                                                |
| `--cpus <value>`          | `2`                                   | 容器 CPU 限制。                                                                           |
| `--disk <size>`           | 無                                  | 在儲存後端支援配額時，限制容器可寫入層。                     |
| `--network <mode>`        | `bridge`                              | 輸出網路模式：`bridge` 或 `internal`。                                                 |
| `--pids-limit <number>`   | `512`                                 | 容器內的處理程序數量上限。                                                  |
| `--env <KEY=VALUE>`       | 無                                  | 將環境變數傳入 cell。若有多個值，請重複使用此選項。                          |
| `--gateway-token <value>` | 隨機 32 字元十六進位權杖 | 使用提供的閘道權杖，而非產生權杖。請參閱[權杖處理](#token-handling)。 |
| `--no-start`              | 啟動 cell                           | 建立容器但不啟動。                                                      |
| `--json`                  | 人類可讀輸出                 | 顯示機器可讀輸出。                                                                 |

自動配置會選取大於或等於 `19100` 的第一個未使用登錄連接埠。Fleet 會拒絕重複的租戶 ID，以及已指派給另一個 cell 的明確連接埠。

映像檔參照會作為單一容器執行階段引數傳入。空白參照及以 `-` 開頭的值會遭到拒絕，避免映像檔被解讀為 Docker 或 Podman 選項。

所選的 Docker 或 Podman 端點必須位於本機。Fleet 會在保留連接埠或建立本機狀態前，拒絕遠端 Docker context、`DOCKER_HOST` 端點及遠端 Podman 服務。不支援遠端 cell 主機。

當 Fleet 啟動新的 cell 時，create 會等待最多約一分鐘，直到其閘道回應 `/healthz`。若 cell 未恢復正常，Fleet 會保留其容器和登錄資料列，以供 `fleet status`、`fleet logs` 或明確移除使用。`--no-start` 會略過此健康狀態閘門。狀態異常之新 cell 所產生的閘道權杖不會遺失，而會保留在容器環境中（`docker|podman inspect`）；此外，由於該 cell 尚未處理任何流量，因此先執行 `fleet rm --force` 再重新建立，永遠都是安全的替代方案。

### 使用摘要固定版本

create 和 upgrade 接受以摘要固定的映像檔參照，例如 `--image ghcr.io/openclaw/openclaw@sha256:<digest>`。Fleet 會將映像檔參照原封不動地傳給 Docker 或 Podman，讓操作人員能將 cell 保持在不可變的映像檔位元組，而非隨標籤變動。

建立結果包括租戶 ID、容器名稱、主機連接埠、閘道權杖及本機 URL。即使採用 JSON 輸出，也應將結果視為含有機密資訊，因為其中包含權杖。

### 磁碟限制

`--disk` 僅限制容器可寫入層。透過繫結掛載的各租戶狀態與驗證目錄仍使用主機儲存空間；若這些目錄也需要硬性限制，請使用主機檔案系統的專案配額。

| 執行階段／儲存後端 | `--disk` 支援                                                             |
| ----------------------- | ---------------------------------------------------------------------------- |
| XFS 上的 Docker overlay2  | 需要 XFS 的 `pquota` 掛載選項。                                      |
| Docker btrfs 或 zfs     | 由儲存驅動程式支援。                                             |
| Podman overlay          | 需要 XFS 作為後備儲存空間。                                                |
| 其他後端          | 容器建立會失敗，並顯示常駐程式錯誤及 Fleet 的後端指引。 |

### 輸出流量政策

| 模式       | Docker                                                                                                | Podman                                                                              |
| ---------- | ----------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `bridge`   | 支援；預設不限制輸出流量。                                                | 支援；預設不限制輸出流量。                              |
| `internal` | 拒絕，因為 Docker 不會在內部網路上保留已發布的迴送閘道連接埠。 | 支援；封鎖輸出流量時，迴送閘道仍會保持發布狀態。 |

使用 Docker 時，請保留 bridge 模式，並使用 `DOCKER-USER` 鏈結等主機防火牆規則強制執行輸出政策。

## `fleet list`

依租戶 ID 順序列出 cell：

```bash
openclaw fleet list
openclaw fleet ls
openclaw fleet list --json
```

表格包含：

| 欄位    | 意義                                                                                                                                                                                                                                                                               |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `tenant`  | 租戶 ID。                                                                                                                                                                                                                                                                            |
| `state`   | 來自 Docker 或 Podman 檢查的即時容器狀態。`unknown` 表示執行階段無法使用，或存在名稱與 cell 相同的容器，但其 Fleet 擁有權標籤與登錄記錄不符（這是名稱衝突或遭竄改的訊號——採取動作前請先手動檢查）。 |
| `port`    | 對應至 cell 閘道的迴送主機連接埠。                                                                                                                                                                                                                                        |
| `image`   | 記錄的容器映像檔。                                                                                                                                                                                                                                                             |
| `created` | cell 建立時間。                                                                                                                                                                                                                                                                   |

Docker 或 Podman 無法使用時，登錄資料列仍然可見；只有即時狀態會變為 `unknown`。

## `fleet status`

檢查一個 cell：

```bash
openclaw fleet status acme
openclaw fleet status acme --json
```

狀態會合併 Fleet 登錄資料列、即時容器檢查，以及對下列端點發出的簡短盡力而為要求：

```text
http://127.0.0.1:<host-port>/healthz
```

健康狀態結果為 `ok`、`failed` 或 `skipped`。`/healthz` 證明閘道仍在運作，但不代表每個已設定的頻道或外掛均已完全就緒。沒有可供檢查的可用本機端點時，會略過探測。

## `fleet logs`

將 cell 的容器日誌直接串流至終端機：

```bash
openclaw fleet logs acme
openclaw fleet logs acme --follow
openclaw fleet logs acme --tail 200
openclaw fleet logs acme --since 10m
```

Fleet 會先驗證已註冊容器的擁有權標籤，再讀取任何日誌，因此會拒絕使用預期 cell 名稱的外來容器。串流會固定至該受檢查容器的 ID，因此同時進行的替換作業無法將其重新導向至較新的世代。按下 Ctrl-C 可結束 `--follow`，且不會將操作人員的停止動作視為命令失敗。日誌輸出會先經過遮蔽篩選器，將 cell 目前的閘道權杖替換為 `<redacted>`，之後才會有任何內容送達終端機。

`fleet logs` 沒有 `--json` 模式，因為容器日誌是原始 stdout/stderr 串流。若用於指令碼，請使用 `--tail` 限制輸出，並使用一般的 shell 重新導向或管線。

## `fleet start`、`fleet stop` 與 `fleet restart`

使用已記錄的執行環境控制現有的單元：

```bash
openclaw fleet start acme
openclaw fleet stop acme
openclaw fleet restart acme
```

這些命令會操作已註冊的容器名稱。如果租戶未知，或已記錄的執行環境無法執行該操作，命令就會失敗。

## `fleet upgrade`

重新拉取已記錄的映像，並取代單元容器：

```bash
openclaw fleet upgrade acme
```

將單元移至另一個映像：

```bash
openclaw fleet upgrade acme --image ghcr.io/openclaw/openclaw:<version>
```

升級會拉取目標映像、檢查現有容器與各單元網路、停止並移除容器，然後重新建立並啟動容器。替代容器會保留相同的主機連接埠、資料目錄、各單元橋接網路、執行環境設定檔、資源限制、重新啟動原則、由 Fleet 管理的環境，以及最初透過 `--env` 提供的值。掛載的狀態在容器替換後仍會保留；映像預設環境可能隨目標映像而變更。

只有在替代容器的閘道於單元的回送連接埠上回應 `/healthz`，符合官方 compose 檔案所使用的健康狀態合約後，才會提交替換。如果替代容器結束、陷入當機循環，或未能在約一分鐘內轉為健康狀態，系統會將其移除並還原先前的容器，因此損壞的映像不會導致運作中的單元停機。

閘道權杖刻意不儲存在 Fleet 登錄中。移除舊容器前，Fleet 會讀取其環境，並將 `OPENCLAW_GATEWAY_TOKEN` 帶入替代容器。如果權杖未存在於你控制的其他位置，請勿在升級前手動移除舊容器。

## `fleet backup` 與 `fleet restore`

備份一個已停止的單元：

```bash
openclaw fleet stop acme
openclaw fleet backup acme --out ./acme.tgz
```

將該封存檔還原至已註冊的單元：

```bash
openclaw fleet restore acme --from ./acme.tgz
```

這些是需要主機操作員權限的命令。封存檔包含租戶狀態與驗證祕密，建立時的模式為 `0600`，且必須像認證資訊一樣妥善保存。備份會拒絕執行中的單元，以便一致地擷取 SQLite 狀態。除非提供 `--force`，否則還原會拒絕執行中的單元；還原只會取代該租戶的狀態、輪替閘道權杖，並僅顯示一次新權杖。Fleet 一次只會備份一個租戶；備份所有租戶是另一項獨立的操作員操作。

還原需要現有且已停止的容器，因為從該容器檢查到的執行環境設定檔會提供替代容器的限制、使用者對應、環境來源與映像。如果已註冊的容器遭到外部移除，請先執行不含 `--purge-data` 的 `fleet rm <tenant> --force`，使用預定的映像與 `--no-start` 重新建立單元，再重試還原。第一次移除會完整保留兩個租戶資料目錄。

兩個命令都接受 `--max-bytes <bytes>`，以限制封存或解壓縮的檔案資料；兩者也會套用相同的固定一百萬個封存路徑區段預算，因此僅含中繼資料的封存炸彈無法耗盡主機 inode，且每個接受的備份都能還原。備份接受 `--out <path>`，兩個命令都支援 `--json`。

封存檔只能包含一般檔案與目錄。備份絕不會跟隨或儲存符號連結、硬連結、通訊端或裝置節點；結果中會回報略過的數量。還原會拒絕包含任何其他項目類型的封存檔。工作區 `node_modules` 等可重新建立的符號連結樹狀結構，必須在還原後於單元內重新安裝。

## `fleet doctor`

稽核所有單元或單一租戶，而不變更執行環境或檔案系統狀態：

```bash
openclaw fleet doctor
openclaw fleet doctor acme --json
```

Doctor 會檢查執行環境的本機性、擁有權標籤、健康狀態、安全強化、資源限制、回送連接埠繫結、權杖是否存在、網路擁有權與輸出模式，以及私有狀態目錄權限。警告會說明已停止的單元或擁有權差異；任何失敗的檢查結果都會設定非零的程序結束代碼。

## `fleet rm`

從執行環境與登錄中移除已停止的單元，同時保留租戶資料：

```bash
openclaw fleet rm acme
```

執行中的容器需要 `--force`：

```bash
openclaw fleet rm acme --force
```

同時永久移除單元資料：

```bash
openclaw fleet rm acme --purge-data --force
```

Fleet 會先移除單元容器，再移除其專用橋接網路。`--purge-data` 需要 `--force`。在遞迴刪除之前，Fleet 會解析兩個由 Fleet 擁有的根目錄，以及兩個各租戶目錄。每個目標都必須是完全符合預期的租戶葉節點、嚴格位於其根目錄內，且不得為符號連結。這些包含範圍檢查可防止損壞的登錄路徑或跨租戶符號連結將刪除操作重新導向其他位置。

如果完全符合預期的租戶目錄已不存在，清除操作仍可重試。如此一來，後續叫用便能在部分檔案系統失敗後完成清理，而不會放寬對仍然存在之目錄的路徑檢查。

## 儲存空間與容器配置

單元狀態與驗證設定檔加密金鑰會使用作用中 OpenClaw 狀態目錄下不同的各租戶主機路徑：

```text
<state-dir>/fleet/cells/<tenant>/
<state-dir>/fleet/auth-profile-secrets/<tenant>/
```

第一個目錄掛載於 `/home/node/.openclaw`。第二個目錄掛載於 `/home/node/.config/openclaw`，與官方 Docker 設定的加密金鑰掛載方式一致。因此，加密金鑰不會暴露在一般狀態掛載之下，也不會在僅備份或共用單元狀態目錄時納入其中。兩個目錄在一般移除與升級後都會保留；`fleet rm --purge-data --force` 會在分別執行包含範圍檢查後刪除兩者。

首次啟動前，Fleet 會使用 `gateway.mode=local`、權杖驗證、LAN 容器繫結，以及分配之主機連接埠的 Control UI 來源來初始化單元設定。權杖值不會寫入該設定，而是保留在容器環境中。

Fleet 會使用以下環境值固定官方映像的容器路徑：

| 變數                     | 容器值                               |
| ------------------------ | ------------------------------------ |
| `HOME`                   | `/home/node`                         |
| `OPENCLAW_HOME`          | `/home/node`                         |
| `OPENCLAW_STATE_DIR`     | `/home/node/.openclaw`               |
| `OPENCLAW_CONFIG_PATH`   | `/home/node/.openclaw/openclaw.json` |
| `OPENCLAW_WORKSPACE_DIR` | `/home/node/.openclaw/workspace`     |
| `OPENCLAW_GATEWAY_TOKEN` | 產生或提供的單元權杖                 |

官方映像預設使用 UID 1000 的非 root `node` 使用者。Fleet 會讓私有 `0700` 繫結掛載保持可寫入，而不使其可由所有人存取。有 root Docker 會使用叫用者的非 root UID 與 GID 執行單元；無 root Docker 則使用容器 UID 0，該 UID 會在常駐程式的使用者命名空間內對應至叫用者在主機上的非特權使用者。Podman 使用 `keep-id` 搭配叫用者的 UID 與 GID。當 Fleet 本身以 root 身分搭配有 root 執行環境執行時，會保留映像使用者，並將初始掛載檔案指派給 UID/GID 1000。

在 SELinux 主機上，Docker 與 Podman 掛載會套用私有 `:Z` 重新標記。如果還原或重新安置單元資料，請確保繫結掛載路徑可由有效的容器使用者寫入。此設定檔適合無 root 環境，但主機上的 Docker 或 Podman 必須已設定為無 root 作業；Fleet 不會將有 root 常駐程式轉換為無 root 常駐程式。

## 安全性設定檔

Fleet 會將以下設定檔套用至每個單元：

| 控制項               | 套用的設定檔                                         | 原因                                                                                     |
| -------------------- | ---------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Linux 權能           | `--cap-drop=ALL`                                     | 閘道是 Node.js 程序，不需要新增任何 Linux 權能。                                        |
| 權限提升             | `--security-opt no-new-privileges`                   | 防止程序透過 setuid 或 setgid 二進位檔取得權限。                                        |
| Init 程序            | `--init`                                             | 回收後代程序並轉送容器生命週期訊號。                                                     |
| 程序限制             | 預設為 `--pids-limit 512`                        | 限制分叉與程序耗盡。                                                                     |
| 記憶體限制           | 預設為 `--memory 2g`                             | 限制單元記憶體用量。                                                                     |
| CPU 限制             | 預設為 `--cpus 2`                                | 限制單元 CPU 用量。                                                                      |
| 可寫入層磁碟         | 選用的 `--disk`                                    | 當執行環境的儲存後端支援配額時，限制容器層。                                             |
| 重新啟動原則         | `--restart unless-stopped`                           | 重新啟動失敗的單元，而不覆寫刻意停止的操作。                                             |
| 主機發布             | 僅限 `127.0.0.1:<host-port>:18789`                   | 讓閘道不會暴露於主機萬用字元介面。                                                       |
| 單元網路             | 每個單元一個橋接網路或 Podman 內部網路               | 隔離容器 IP 流量，並可選擇封鎖 Podman 對外輸出流量。                                     |
| 容器身分             | 與主機相符的使用者對應                               | 讓私有繫結掛載保持可寫入，而不授予所有人存取權。                                         |
| 永久狀態             | 各單元掛載；不使用共用狀態掛載                       | 將租戶設定、認證資訊、工作階段與工作區保留在該租戶的資料樹狀結構中。                     |
| 容器命令             | `node dist/index.js gateway --bind lan --port 18789` | 在容器網路上接聽，使僅限回送的主機連接埠對應能夠連線。                                   |

Fleet 絕不會掛載 `/var/run/docker.sock`、使用 `--privileged` 或主機網路，也不會新增權能。各單元橋接網路是跨單元隔離邊界，而不是對外輸出防火牆：單元仍保有供應商與頻道所需的網路輸出能力。請使用符合部署方式的 Proxy、SSH 通道或 tailnet 設定作為回送連接埠的前端。`http://127.0.0.1:<port>` 只能直接從 Fleet 主機連線。

此設定檔會隔離租戶容器，但無法保護租戶免受 Fleet 操作員、容器執行環境管理員或遭入侵主機的影響。如需完整的信任模型與更強的隔離選項，請參閱[多租戶託管](/zh-TW/gateway/multi-tenant-hosting)。

## 權杖處理

根據預設，`fleet create` 會產生一個密碼學安全的隨機 32 字元十六進位閘道權杖，並在建立結果中僅顯示一次。請將其儲存在核准的祕密管理工具中，並避免將建立輸出記錄至日誌。

`--gateway-token` 會將自訂權杖放入本機程序引數，這些引數可能保留在 Shell 歷程記錄中，或顯示於程序清單中。除非現有祕密管理工作流程需要提供指定值，否則請優先使用產生的權杖。

權杖以及透過 `--env` 傳遞的每個值都存在於容器環境中。Fleet 會將它們寫入短期存在、模式為 `0600` 的環境檔案，只將該檔案的路徑傳遞給 Docker 或 Podman，並在執行環境命令完成後移除檔案。明確輸入 `openclaw fleet create --gateway-token ...` 或 `--env KEY=VALUE` 的值，仍可能顯示於外層 `openclaw` 程序引數與 Shell 歷程記錄中。

容器環境值不會對受信任的主機操作員隱藏：Docker 或 Podman 管理員可以透過容器檢查讀取這些值。Fleet 的「僅顯示一次」註記描述的是一般命令列介面輸出，而不是防止主機管理員存取。

## 相關內容

- [多租戶託管](/zh-TW/gateway/multi-tenant-hosting)
- [Docker](/zh-TW/install/docker)
- [Podman](/zh-TW/install/podman)
- [閘道安全性](/zh-TW/gateway/security)
