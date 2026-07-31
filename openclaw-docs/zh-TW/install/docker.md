---
read_when:
    - 你想要容器化閘道，而不是在本機安裝
    - 你正在驗證 Docker 流程
summary: OpenClaw 的選用 Docker 設定與入門流程
title: Docker
x-i18n:
    generated_at: "2026-07-26T07:21:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c1784bd49f6847db75633840a4d5a8e49205200728bd2e9d59b646a446e508d6
    source_path: install/docker.md
    workflow: 16
---

Docker 是**選用的**。可用於建立隔離、用完即丟的閘道環境，或用於未在本機安裝相關元件的主機。如果你已在自己的機器上進行開發，請改用一般安裝流程。

啟用 `agents.defaults.sandbox` 時，預設沙箱後端會使用 Docker，但沙箱預設為停用，且閘道本身不必在 Docker 中執行。另有 SSH 與 OpenShell 沙箱後端可供使用；請參閱[沙箱](/zh-TW/gateway/sandboxing)。

要代管多位使用者嗎？關於每個租戶使用一個單元的模型，請參閱[多租戶代管](/zh-TW/gateway/multi-tenant-hosting)。

## 先決條件

- Docker Desktop（或 Docker Engine）+ Docker Compose v2
- 建置映像檔至少需要 2 GB RAM（在 1 GB 主機上，`pnpm install` 可能因記憶體不足而遭終止，結束代碼為 137）
- 有足夠的磁碟空間存放映像檔與日誌
- 若使用 VPS／公開主機，請檢閱[網路暴露的安全強化措施](/zh-TW/gateway/security)，尤其是 Docker 的 `DOCKER-USER` 防火牆鏈

## 容器化閘道

<Steps>
  <Step title="建置映像檔">
    從儲存庫根目錄執行：

    ```bash
    ./scripts/docker/setup.sh
    ```

    這會在本機將閘道映像檔建置為 `openclaw:local`。若要改用預先建置的映像檔：

    ```bash
    export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
    ./scripts/docker/setup.sh
    ```

    預先建置的映像檔會優先發布至 [GitHub Container Registry](https://github.com/openclaw/openclaw/pkgs/container/openclaw)。GHCR 是發布自動化、固定版本部署及來源證明檢查的主要登錄檔。同一版本也會在 Docker Hub 發布鏡像 `openclaw/openclaw`：

    ```bash
    export OPENCLAW_IMAGE="openclaw/openclaw:latest"
    ./scripts/docker/setup.sh
    ```

    請使用 `ghcr.io/openclaw/openclaw` 或 `openclaw/openclaw`，並避免使用非官方鏡像，因為它們不遵循 OpenClaw 的發布時程或保留政策。特定版本標籤包括 `2026.2.26` 等正式版本，以及 `2026.2.26-beta.1` 等預發行版本。穩定版本會更新 `latest` 和 `main`；月底閘道版本則只會更新 `extended-stable`。變體包括 `slim`、`main-slim`、`extended-stable-slim`、`latest-browser`、`main-browser` 和 `extended-stable-browser`。預設映像檔內含 `codex` 與 `diagnostics-otel` 外掛。另有 `-browser` 變體內建 Chromium，適合搭配[沙箱瀏覽器](/zh-TW/gateway/sandboxing#sandboxed-browser)工具使用，首次執行時不必安裝 Playwright。

  </Step>

  <Step title="在隔離網路中重新執行">
    在離線主機上，請先傳輸並載入映像檔：

    ```bash
    docker load -i openclaw-image.tar
    export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
    ./scripts/docker/setup.sh --offline
    ```

    `--offline` 會驗證 `OPENCLAW_IMAGE` 已存在於本機、停用隱含的 Compose 提取／建置，然後執行一般流程：`.env` 同步、權限修正、初始設定、閘道設定同步及 Compose 啟動。

    如果 `OPENCLAW_SANDBOX=1`，離線設定也會檢查 `OPENCLAW_DOCKER_SOCKET` 所連線之常駐程式上已設定的預設與各代理沙箱映像檔，包括 Docker 後端瀏覽器映像檔上的瀏覽器合約標籤。如果缺少必要映像檔或其版本過舊，設定程序會直接結束且不變更沙箱設定，而不會錯誤地回報成功。

  </Step>

  <Step title="完成初始設定">
    設定指令碼會自動執行初始設定：

    - 提示輸入提供者 API 金鑰
    - 產生閘道權杖並寫入 `.env`
    - 建立驗證設定檔密鑰目錄
    - 透過 Docker Compose 啟動閘道

    啟動前的初始設定與設定寫入作業會直接透過 `openclaw-gateway` 執行（搭配 `--no-deps --entrypoint node`），因為 `openclaw-cli` 會共用閘道的網路命名空間，只有在閘道容器存在後才能運作。

  </Step>

  <Step title="開啟控制介面">
    開啟 `http://127.0.0.1:18789/`，並將寫入 `.env` 的權杖貼到設定中。如果你已將容器切換為密碼驗證，請改用該密碼。

    再次需要該網址嗎？

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    ```

  </Step>

  <Step title="設定頻道（選用）">
    ```bash
    # WhatsApp（QR Code）
    docker compose run --rm openclaw-cli channels login

    # Telegram
    docker compose run --rm openclaw-cli channels add --channel telegram --token "<token>"

    # Discord
    docker compose run --rm openclaw-cli channels add --channel discord --token "<token>"
    ```

    文件：[WhatsApp](/zh-TW/channels/whatsapp)、[Telegram](/zh-TW/channels/telegram)、[Discord](/zh-TW/channels/discord)

  </Step>
</Steps>

### 手動流程

```bash
BUILD_GIT_COMMIT="$(git rev-parse HEAD)"
BUILD_TIMESTAMP="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
docker build \
  --build-arg "GIT_COMMIT=${BUILD_GIT_COMMIT}" \
  --build-arg "OPENCLAW_BUILD_TIMESTAMP=${BUILD_TIMESTAMP}" \
  -t openclaw:local -f Dockerfile .
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js onboard --mode local --no-install-daemon
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js config set --batch-json '[{"path":"gateway.mode","value":"local"},{"path":"gateway.bind","value":"lan"},{"path":"gateway.controlUi.allowedOrigins","value":["http://localhost:18789","http://127.0.0.1:18789"]}]'
docker compose up -d openclaw-gateway
```

Docker 建置內容會排除 `.git`。請依照上方所示，將原始碼識別資訊作為建置引數傳入，讓映像檔的「關於」畫面顯示目前簽出的提交，以及單一建置時間戳記。`scripts/docker/setup.sh` 會自動解析並傳入這兩個值。

<Note>
請從儲存庫根目錄執行 `docker compose`。如果你已啟用 `OPENCLAW_EXTRA_MOUNTS` 或 `OPENCLAW_HOME_VOLUME`，設定指令碼會寫入 `docker-compose.extra.yml`；請將它放在你自行維護的任何 `docker-compose.override.yml` 之後，例如 `-f docker-compose.yml -f docker-compose.override.yml -f docker-compose.extra.yml`。
</Note>

### 升級容器映像檔

當你更換 OpenClaw 映像檔但保留相同的掛載狀態／設定時，新閘道會在就緒前執行可安全啟動的升級移轉與外掛收斂。例行映像檔升級不應需要另外執行一次 `openclaw doctor --fix`。

如果啟動時無法安全地完成這些修復，閘道將結束，而不會回報為健康狀態。使用重新啟動政策時，Docker、Podman 或 Kubernetes 可能會顯示閘道容器不斷重新啟動。請保留已掛載的狀態磁碟區，然後使用閘道所用的相同狀態／設定掛載，將 `openclaw doctor --fix` 作為容器命令，以相同映像檔執行一次：

```bash
docker run --rm -v <openclaw-state>:/home/node/.openclaw <image> openclaw doctor --fix
podman run --rm -v <openclaw-state>:/home/node/.openclaw <image> openclaw doctor --fix
```

doctor 完成後，使用預設命令重新啟動閘道容器。在 Kubernetes 中，請在掛載相同 PVC 的一次性 Job 或偵錯 Pod 中執行相同命令，然後重新啟動 Deployment 或 StatefulSet。

### 環境變數

`scripts/docker/setup.sh` 可接受的選用變數（閘道容器也可直接透過 `docker-compose.yml` 接受）：

| 變數                                        | 用途                                                                                                           |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_IMAGE`                                | 使用遠端映像檔，而非在本機建置                                                                    |
| `OPENCLAW_IMAGE_APT_PACKAGES`                   | 在建置期間安裝額外的 apt 套件（以空格分隔）。舊版別名：`OPENCLAW_DOCKER_APT_PACKAGES`           |
| `OPENCLAW_IMAGE_PIP_PACKAGES`                   | 在建置期間安裝額外的 Python 套件（以空格分隔）                                                      |
| `OPENCLAW_EXTENSIONS`                           | 編譯／封裝所選且受支援的外掛，並安裝其執行階段相依項目（以逗號或空格分隔的 ID） |
| `OPENCLAW_DOCKER_BUILD_NODE_OPTIONS`            | 覆寫本機原始碼建置的 Node 選項（預設為 `--max-old-space-size=8192`）                                |
| `OPENCLAW_DOCKER_BUILD_TSDOWN_MAX_OLD_SPACE_MB` | 覆寫本機原始碼建置的 tsdown 堆積大小（單位為 MB）                                                                 |
| `OPENCLAW_DOCKER_BUILD_SKIP_DTS`                | 僅執行階段的本機映像檔建置期間略過宣告輸出（預設為 `1`）                                      |
| `OPENCLAW_INSTALL_BROWSER`                      | 在建置時將 Chromium + Xvfb 內建至映像檔                                                                 |
| `OPENCLAW_EXTRA_MOUNTS`                         | 額外的主機繫結掛載（以逗號分隔的 `source:target[:opts]`）                                                   |
| `OPENCLAW_HOME_VOLUME`                          | 將 `/home/node` 持久保存在具名 Docker 磁碟區中                                                                     |
| `OPENCLAW_SANDBOX`                              | 選擇啟用沙箱啟動程序（`1`、`true`、`yes`、`on`）                                                            |
| `OPENCLAW_SKIP_ONBOARDING`                      | 略過互動式初始設定步驟（`1`、`true`、`yes`、`on`）                                                   |
| `OPENCLAW_DOCKER_SOCKET`                        | 覆寫 Docker 通訊端路徑                                                                                   |
| `OPENCLAW_DISABLE_BONJOUR`                      | 強制開啟（`0`）或關閉（`1`）Bonjour／mDNS 公告；請參閱 [Bonjour／mDNS](#bonjour--mdns)                        |
| `OPENCLAW_DISABLE_BUNDLED_SOURCE_OVERLAYS`      | 停用隨附外掛原始碼的繫結掛載覆蓋層                                                                 |
| `OTEL_EXPORTER_OTLP_ENDPOINT`                   | 用於 OpenTelemetry 匯出的共用 OTLP／HTTP 收集器端點                                                      |
| `OTEL_EXPORTER_OTLP_*_ENDPOINT`                 | 追蹤、指標或日誌的訊號專用 OTLP 端點                                                       |
| `OTEL_EXPORTER_OTLP_PROTOCOL`                   | 覆寫 OTLP 通訊協定。目前僅支援 `http/protobuf`                                                   |
| `OTEL_SERVICE_NAME`                             | OpenTelemetry 資源使用的服務名稱                                                                     |
| `OTEL_SEMCONV_STABILITY_OPT_IN`                 | 選擇啟用最新的實驗性 GenAI 語意屬性                                                           |
| `OPENCLAW_OTEL_PRELOADED`                       | 預先載入 OpenTelemetry SDK 時，略過啟動第二個 SDK                                                    |

官方映像檔不隨附 Homebrew。初始設定期間，在不含 `brew` 的 Linux 容器中，OpenClaw 會隱藏僅支援 brew 的 Skills 相依項目安裝程式；請透過自訂映像檔提供這些相依項目，或手動安裝。Debian 套件形式的相依項目請使用 `OPENCLAW_IMAGE_APT_PACKAGES`，Python 相依項目請使用 `OPENCLAW_IMAGE_PIP_PACKAGES`（會在建置時執行 `python3 -m pip install --break-system-packages`，因此請固定版本，且僅使用你信任的索引）。

如果 Docker 回報 `ResourceExhausted`、`cannot allocate memory`，或在 `tsdown` 期間中止，請提高 Docker 建置器的記憶體限制，或使用較小且明確指定的堆積大小重試：

```bash
OPENCLAW_DOCKER_BUILD_NODE_OPTIONS=--max-old-space-size=4096 OPENCLAW_DOCKER_BUILD_TSDOWN_MAX_OLD_SPACE_MB=4096
```

### 包含所選外掛、從原始碼建置的映像檔

`OPENCLAW_EXTENSIONS` 會從原始碼簽出中選取外掛資訊清單 ID；
若現有原始碼目錄名稱不同，也接受使用該名稱。Docker
建置會將選取項目一次解析為原始碼目錄、安裝正式環境
相依套件，且當選取的外掛使用
`openclaw.build.bundledDist: false` 個別發布時，會將其執行階段編譯至根目錄的隨附
dist。這項僅限 Docker 的封裝不會變更此外掛的 npm 或 ClawHub
成品合約。未知、無效或有歧義的 ID 會導致映像檔建置失敗。
已知的相依套件／僅原始碼 ID 會保留其現有的原始碼與相依套件
暫存方式，而不會新增已編譯的根目錄 dist 項目。具有
統一建置項目的選取外掛必須成功編譯；未選取的外部外掛
原始碼與執行階段輸出會遭到裁剪。

例如，以下命令會為 ClickClack、Slack 和 Microsoft Teams 建置各自獨立、
多架構的 FakeCo 閘道映像檔。ClawRouter 已經是
OpenClaw 根執行階段的一部分，因此 ClickClack 映像檔只選取
`clickclack`。明確傳入空白的瀏覽器引數，可讓預設映像檔不含
Chromium：

```bash
SOURCE_SHA="$(git rev-parse HEAD)"
BUILD_TIMESTAMP="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
REGISTRY="registry.example.com/fakeco"

build_gateway_image() {
  gateway="$1"
  selected_plugin="$2"
  docker buildx build \
    --platform linux/amd64,linux/arm64 \
    --build-arg "GIT_COMMIT=${SOURCE_SHA}" \
    --build-arg "OPENCLAW_BUILD_TIMESTAMP=${BUILD_TIMESTAMP}" \
    --build-arg "OPENCLAW_EXTENSIONS=${selected_plugin}" \
    --build-arg OPENCLAW_INSTALL_BROWSER= \
    --provenance=mode=max \
    --sbom=true \
    --tag "${REGISTRY}/openclaw-${gateway}:${SOURCE_SHA}" \
    --push \
    .
}

build_gateway_image clickclack clickclack
build_gateway_image slack slack
build_gateway_image teams msteams
```

單一原生本機建置請使用 `--platform linux/arm64 --load` 或 `--platform linux/amd64 --load`。
多平台輸出與附加的 SBOM／來源證明
需要登錄檔，或其他能保留證明資料的 Buildx 輸出。推送後，
請檢查資訊清單，並部署不可變的摘要，而非
可變的原始碼 SHA 標籤：

```bash
docker buildx imagetools inspect \
  "${REGISTRY}/openclaw-clickclack:${SOURCE_SHA}"
# 部署：registry.example.com/fakeco/openclaw-clickclack@sha256:<manifest-digest>
```

這些映像檔適用於獨立的 OCI 型閘道及一般 Docker 使用者。
由 Crabhelm 管理的閘道不會使用這些映像檔：該交付路徑會建置
獨立的 x86_64 設備封存檔，其中包含 OpenClaw npm tarball，並固定
Node、封存檔及資訊清單摘要。請從同一份已合併的 OpenClaw 原始碼
獨立建置該設備。

若要針對封裝映像檔測試隨附的外掛原始碼，請將一個外掛原始碼目錄掛載到其封裝的原始碼路徑上，例如 `OPENCLAW_EXTRA_MOUNTS=/path/to/fork/extensions/synology-chat:/app/extensions/synology-chat:ro`。這會覆寫相同外掛 ID 對應的已編譯 `/app/dist/extensions/synology-chat` 套件。

### 可觀測性

OpenTelemetry 匯出會從閘道容器向外傳送至你的 OTLP 收集器；不需要發布任何 Docker 連接埠。若要在本機建置的映像檔中納入隨附的匯出器：

```bash
export OPENCLAW_EXTENSIONS="diagnostics-otel"
export OTEL_EXPORTER_OTLP_ENDPOINT="http://otel-collector:4318"
export OTEL_SERVICE_NAME="openclaw-gateway"
./scripts/docker/setup.sh
```

官方預先建置的映像檔已隨附 `diagnostics-otel`；只有在你移除它之後，才需要自行安裝 `clawhub:@openclaw/diagnostics-otel`。若要啟用匯出，請在設定中允許並啟用 `diagnostics-otel` 外掛，然後設定 `diagnostics.otel.enabled=true`（完整範例請參閱 [OpenTelemetry 匯出](/zh-TW/gateway/opentelemetry)）。收集器驗證標頭需透過 `diagnostics.otel.headers` 傳入，而非 Docker 環境變數。

Prometheus 指標會重複使用已發布的閘道連接埠。安裝 `clawhub:@openclaw/diagnostics-prometheus`、啟用 `diagnostics-prometheus` 外掛，然後擷取：

```text
http://<gateway-host>:18789/api/diagnostics/prometheus
```

此路由受閘道驗證保護；請勿公開獨立的公用 `/metrics` 連接埠或未經驗證的反向 Proxy 路徑。請參閱 [Prometheus 指標](/zh-TW/gateway/prometheus)。

### 健全狀態檢查

容器探查端點（不需要驗證）：

```bash
curl -fsS http://127.0.0.1:18789/healthz   # 存活狀態
curl -fsS http://127.0.0.1:18789/readyz     # 就緒狀態
```

映像檔內建的 `HEALTHCHECK` 會偵測 `/healthz`；重複失敗會將容器標記為 `unhealthy`，讓協調器能重新啟動或替換它。

需驗證的深度健全狀態快照：

```bash
docker compose exec openclaw-gateway node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"
```

### 區域網路與迴路介面

`scripts/docker/setup.sh` 預設為 `OPENCLAW_GATEWAY_BIND=lan`，因此主機上的 `http://127.0.0.1:18789` 可搭配 Docker 連接埠發布運作。

- `lan`（預設）：主機瀏覽器與主機命令列介面可以連線至已發布的閘道連接埠。
- `loopback`：只有容器網路命名空間內的程序能直接連線至閘道。

<Note>
請使用 `gateway.bind` 中的繫結模式值（`lan` / `loopback` / `custom` / `tailnet` / `auto`），而非 `0.0.0.0` 或 `127.0.0.1` 等主機別名。
</Note>

### 主機本機提供者

在容器內，`127.0.0.1` 指的是容器本身，而非主機。對於在主機上執行的提供者，請使用 `host.docker.internal`：

| 提供者    | 主機預設 URL             | Docker 設定 URL                     |
| --------- | ------------------------ | ----------------------------------- |
| LM Studio | `http://127.0.0.1:1234`  | `http://host.docker.internal:1234`  |
| Ollama    | `http://127.0.0.1:11434` | `http://host.docker.internal:11434` |

隨附的設定會使用這些 URL 作為 LM Studio／Ollama 的初始設定預設值，而 `docker-compose.yml` 會在 Linux Docker Engine 上將 `host.docker.internal` 對應至主機閘道（Docker Desktop 在 macOS／Windows 上提供相同別名）。主機服務必須監聽 Docker 可連線的位址：

```bash
lms server start --port 1234 --bind 0.0.0.0
OLLAMA_HOST=0.0.0.0:11434 ollama serve
```

使用你自己的 Compose 檔案或 `docker run`？請自行加入相同的對應，例如 `--add-host=host.docker.internal:host-gateway`。

### Docker 中的 Claude 命令列介面後端

官方映像檔不會預先安裝 Claude Code。請在容器的 `node` 使用者環境中安裝並登入，然後保存該容器的家目錄，避免映像檔升級清除二進位檔或驗證狀態。

全新安裝時，請先啟用持久化 `/home/node` 磁碟區，再執行設定：

```bash
export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
export OPENCLAW_HOME_VOLUME="openclaw_home"
./scripts/docker/setup.sh
```

若為現有安裝，請先停止堆疊並重新載入目前的 `.env` 值——設定指令碼每次都會根據目前的 Shell 和預設值重寫 `.env`，不會自行讀取該檔案：

```bash
set -a
. ./.env
set +a
export OPENCLAW_HOME_VOLUME="${OPENCLAW_HOME_VOLUME:-openclaw_home}"
./scripts/docker/setup.sh
```

如果 `.env` 包含 Shell 無法載入的值，請先手動重新匯出所需項目（`OPENCLAW_IMAGE`、連接埠、繫結模式、自訂路徑、`OPENCLAW_EXTRA_MOUNTS`、沙箱、略過初始設定）。產生的覆疊設定會為 `openclaw-gateway` 和 `openclaw-cli` 掛載家目錄磁碟區；請使用該覆疊設定執行其餘命令（若有使用 `docker-compose.override.yml`，也請先加入）：

```bash
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  --entrypoint sh openclaw-cli -lc \
  'curl -fsSL https://claude.ai/install.sh | bash'
```

原生安裝程式會將 `claude` 寫入 `/home/node/.local/bin/claude`。
OpenClaw 映像檔已將 `/home/node/.local/bin` 納入 `PATH`，因此隨附的
Anthropic 外掛無須覆寫轉接器設定即可解析它。

從同一個持久化家目錄登入並驗證：

```bash
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  --entrypoint /home/node/.local/bin/claude openclaw-cli auth login
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  --entrypoint /home/node/.local/bin/claude openclaw-cli auth status --text
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  openclaw-cli models auth login \
  --provider anthropic --method cli --set-default
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  openclaw-cli models list --provider anthropic
```

接著使用隨附的 `claude-cli` 後端：

```bash
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  openclaw-cli agent \
  --agent main \
  --model claude-cli/claude-sonnet-4-6 \
  --message "從 Docker Claude 命令列介面說聲你好"
```

`OPENCLAW_HOME_VOLUME` 會保存 `/home/node/.local/bin` 和 `/home/node/.local/share/claude` 下的原生安裝，以及 `/home/node/.claude` 和 `/home/node/.claude.json` 下的 Claude Code 設定／驗證。只保存 `/home/node/.openclaw` 並不足夠；如果你使用 `OPENCLAW_EXTRA_MOUNTS` 而非家目錄磁碟區，請將所有這些 Claude 路徑掛載至兩個服務中。

<Note>
若為共用正式環境自動化或需要可預測的 Anthropic 計費，建議使用 Anthropic API 金鑰路徑。Claude 命令列介面的重複使用行為取決於 Claude Code 已安裝的版本、帳戶登入、計費及更新行為。
</Note>

### Bonjour / mDNS

Docker 橋接網路通常無法可靠轉送 Bonjour／mDNS 多點傳播（`224.0.0.251:5353`）。當 `OPENCLAW_DISABLE_BONJOUR` 未設定時，隨附的 Bonjour 外掛偵測到自己正在容器中執行後，會自動停用區域網路公告，因此不會因橋接網路丟棄多點傳播而反覆重試並陷入當機迴圈。設定 `OPENCLAW_DISABLE_BONJOUR=1` 可無條件強制停用，或設定 `0` 強制啟用（僅適用於主機網路、macvlan，或其他已知 mDNS 多點傳播可正常運作的網路）。

否則，Docker 主機請使用已發布的閘道 URL、Tailscale 或廣域 DNS-SD。注意事項與疑難排解請參閱 [Bonjour 探索](/zh-TW/gateway/bonjour)。

### 儲存空間與持久性

Docker Compose 會將 `OPENCLAW_CONFIG_DIR` 繫結掛載至 `/home/node/.openclaw`、將 `OPENCLAW_WORKSPACE_DIR` 繫結掛載至 `/home/node/.openclaw/workspace`，並將 `OPENCLAW_AUTH_PROFILE_SECRET_DIR` 繫結掛載至 `/home/node/.config/openclaw`，因此這些路徑能在容器替換後保留。當變數未設定時，`docker-compose.yml` 會改用 `${HOME}` 下的路徑；若連 `HOME` 本身都不存在，則改用 `/tmp`，因此 `docker compose up` 在基本環境中絕不會產生來源為空的磁碟區規格。

該掛載的設定目錄包含：

- `openclaw.json`：行為設定
- `agents/<agentId>/agent/auth-profiles.json`：已儲存的提供者 OAuth／API 金鑰驗證資訊
- `.env`：由環境變數提供的執行階段密鑰，例如 `OPENCLAW_GATEWAY_TOKEN`

驗證設定檔密鑰目錄會儲存 OAuth 型驗證設定檔權杖資料的本機加密金鑰。請將它與 Docker 主機狀態一起保存，但要與 `OPENCLAW_CONFIG_DIR` 分開。

已安裝的可下載外掛會將套件狀態儲存在掛載的 OpenClaw 家目錄下，因此安裝記錄和套件根目錄能在容器替換後保留；閘道啟動時不會重新產生隨附外掛的相依套件樹。

完整的 VM 持久性詳細資料，請參閱 [Docker VM 執行階段－各項資料的持久化位置](/zh-TW/install/docker-vm-runtime#what-persists-where)。

**磁碟成長熱點：**`media/`、各代理程式的 SQLite 資料庫、舊版工作階段 JSONL 文字記錄、共用 SQLite 狀態資料庫、已安裝的外掛套件根目錄，以及 `/tmp/openclaw/` 下的輪替檔案記錄。

### Shell 輔助工具（選用）

若要縮短日常命令，請安裝 [ClawDock](/zh-TW/install/clawdock)：

```bash
mkdir -p ~/.clawdock && curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/clawdock/clawdock-helpers.sh -o ~/.clawdock/clawdock-helpers.sh
echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc
```

如果你先前是從較舊的 `scripts/shell-helpers/clawdock-helpers.sh` 路徑安裝，請重新執行上述命令，讓本機輔助工具追蹤目前位置。接著即可使用 `clawdock-start`、`clawdock-stop`、`clawdock-dashboard` 等命令（執行 `clawdock-help` 可查看完整清單）。

<AccordionGroup>
  <Accordion title="為 Docker 閘道啟用代理程式沙箱">
    ```bash
    export OPENCLAW_SANDBOX=1
    ./scripts/docker/setup.sh
    ```

    自訂通訊端路徑（例如無 root 權限的 Docker）：

    ```bash
    export OPENCLAW_SANDBOX=1
    export OPENCLAW_DOCKER_SOCKET=/run/user/1000/docker.sock
    ./scripts/docker/setup.sh
    ```

    此指令碼只會在沙箱先決條件通過後掛載 `docker.sock`。如果無法完成沙箱設定，它會將 `agents.defaults.sandbox.mode` 重設為 `off`。當 OpenClaw 沙箱處於啟用狀態時，該輪次會停用 Codex 程式碼模式（請參閱[沙箱機制 § Docker 後端](/zh-TW/gateway/sandboxing#docker-backend)）；絕不可將主機的 Docker 通訊端掛載至代理程式沙箱容器中。

  </Accordion>

  <Accordion title="自動化／CI（非互動式）">
    使用 `-T` 停用 Compose 的虛擬 TTY 配置：

    ```bash
    docker compose run -T --rm openclaw-cli gateway probe
    docker compose run -T --rm openclaw-cli devices list --json
    ```

  </Accordion>

  <Accordion title="共用網路安全注意事項">
    `openclaw-cli` 使用 `network_mode: "service:openclaw-gateway"`，讓命令列介面指令可以透過 `127.0.0.1` 連線至閘道。請將此視為共用的信任邊界。Compose 設定會捨棄 `NET_RAW`/`NET_ADMIN`，並在 `openclaw-gateway` 和 `openclaw-cli` 上啟用 `no-new-privileges`。
  </Accordion>

  <Accordion title="openclaw-cli 中的 Docker Desktop DNS 失敗">
    某些 Docker Desktop 設定在捨棄 `NET_RAW` 後，會導致共用網路的 `openclaw-cli` 附屬容器無法進行 DNS 查詢，並在 `openclaw plugins install` 等由 npm 支援的指令中顯示為 `EAI_AGAIN`。一般操作請保留預設的強化版 Compose 檔案。下方的覆寫設定只會為 `openclaw-cli` 容器還原預設權能——請僅用於需要存取登錄檔的一次性指令，不要將其作為預設呼叫方式：

    ```bash
    printf '%s\n' \
      'services:' \
      '  openclaw-cli:' \
      '    cap_drop: !reset []' \
      > docker-compose.cli-no-dropped-caps.local.yml

    docker compose -f docker-compose.yml -f docker-compose.cli-no-dropped-caps.local.yml run --rm openclaw-cli plugins install <package>
    ```

    如果你已建立長時間執行的 `openclaw-cli` 容器，請使用相同的覆寫設定重新建立——`docker compose exec`/`docker exec` 無法變更已建立容器的 Linux 權能。

  </Accordion>

  <Accordion title="權限與 EACCES">
    映像檔會以 `node`（uid 1000）執行。如果你在 `/home/node/.openclaw` 上看到權限錯誤，請確認主機的繫結掛載由 uid 1000 擁有：

    ```bash
    sudo chown -R 1000:1000 /path/to/openclaw-config /path/to/openclaw-workspace
    ```

    相同的不相符情況也可能顯示為 `blocked plugin candidate: suspicious ownership (... uid=1000, expected uid=0 or root)`，接著出現 `plugin present but blocked`——程序 uid 與已掛載的外掛目錄擁有者不一致。建議使用預設的 uid 1000 執行，並修正繫結掛載的擁有權。只有在你刻意要長期以 root 身分執行 OpenClaw 時，才將 `/path/to/openclaw-config/npm` 的擁有者變更為 `root:root`。

  </Accordion>

  <Accordion title="加速重新建置">
    調整 Dockerfile 的順序以快取相依套件層，避免在鎖定檔未變更時重新執行 `pnpm install`：

    ```dockerfile
    FROM node:24-bookworm
    RUN curl -fsSL https://bun.sh/install | bash
    ENV PATH="/root/.bun/bin:${PATH}"
    RUN corepack enable
    WORKDIR /app
    COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
    COPY ui/package.json ./ui/package.json
    COPY scripts ./scripts
    RUN pnpm install --frozen-lockfile
    COPY . .
    RUN pnpm build
    RUN pnpm ui:install
    RUN pnpm ui:build
    ENV NODE_ENV=production
    CMD ["node","dist/index.js"]
    ```

  </Accordion>

  <Accordion title="進階使用者容器選項">
    預設映像檔以安全性為優先，並以非 root 的 `node` 身分執行。如需功能更完整的容器：

    1. **保存 `/home/node`**：`export OPENCLAW_HOME_VOLUME="openclaw_home"`
    2. **預先加入系統相依套件**：`export OPENCLAW_IMAGE_APT_PACKAGES="git curl jq"`
    3. **預先加入 Python 相依套件**：`export OPENCLAW_IMAGE_PIP_PACKAGES="requests==2.32.5 humanize==4.14.0"`
    4. **預先加入 Playwright Chromium**：`export OPENCLAW_INSTALL_BROWSER=1`，或使用官方的 `-browser` 映像標籤
    5. **或者將 Playwright 瀏覽器安裝至持久化磁碟區**：
       ```bash
       docker compose run --rm openclaw-cli \
         node /app/node_modules/playwright-core/cli.js install chromium
       ```
    6. **保存瀏覽器下載項目**：使用 `OPENCLAW_HOME_VOLUME` 或 `OPENCLAW_EXTRA_MOUNTS`。OpenClaw 會在 Linux 上自動偵測映像檔中由 Playwright 管理的 Chromium。

  </Accordion>

  <Accordion title="OpenAI Codex OAuth（無頭 Docker）">
    如果你在精靈中選擇 OpenAI Codex OAuth，它會開啟瀏覽器 URL。在 Docker 或無頭設定中，請複製最終到達頁面的完整重新導向 URL，並將其貼回精靈以完成驗證。
  </Accordion>

  <Accordion title="基礎映像檔中繼資料">
    執行階段映像檔使用 `node:24-bookworm-slim`，並以 `tini` 作為 PID 1 執行，以便在長時間執行的容器中清除殭屍程序並正確處理訊號。它會發布 OCI 基礎映像檔註解，包括 `org.opencontainers.image.base.name` 和 `org.opencontainers.image.source`。Dependabot 會更新固定的 Node 基礎映像摘要；發布建置不會執行獨立的發行版升級層。請參閱 [OCI 映像檔註解](https://github.com/opencontainers/image-spec/blob/main/annotations.md)。
  </Accordion>
</AccordionGroup>

### 要在 VPS 上執行嗎？

請參閱 [Hetzner（Docker VPS）](/zh-TW/install/hetzner)和 [Docker VM 執行階段](/zh-TW/install/docker-vm-runtime)，瞭解共用 VM 的部署步驟，包括預先加入二進位檔、持久化及更新。

## 代理程式沙箱

透過 Docker 後端啟用 `agents.defaults.sandbox` 時，閘道會在隔離的 Docker 容器中執行代理程式工具（Shell、檔案讀取／寫入等），而閘道本身仍在主機上執行——這能在不將整個閘道容器化的情況下，為不受信任或多租戶的代理程式工作階段建立一道堅固的隔離牆。

沙箱範圍可以是每個代理程式（預設）、每個工作階段或共用；每個範圍都有自己的工作區，掛載於 `/workspace`。你也可以設定工具允許／拒絕原則、網路隔離、資源限制及瀏覽器容器。

如需完整設定、映像檔、安全注意事項及多代理程式設定檔：

- [沙箱機制](/zh-TW/gateway/sandboxing) -- 完整的沙箱參考資料
- [OpenShell](/zh-TW/gateway/openshell) -- 以互動式 Shell 存取沙箱容器
- [多代理程式沙箱與工具](/zh-TW/tools/multi-agent-sandbox-tools) -- 各代理程式覆寫設定

### 快速啟用

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // 關閉 | 非主要 | 全部
        scope: "agent", // 工作階段 | 代理程式 | 共用
      },
    },
  },
}
```

建置預設沙箱映像檔（從原始碼簽出目錄）：

```bash
scripts/sandbox-setup.sh
```

若透過 npm 安裝且沒有原始碼簽出目錄，請參閱[沙箱機制 § 映像檔與設定](/zh-TW/gateway/sandboxing#images-and-setup)中的內嵌 `docker build` 指令。

## 疑難排解

<AccordionGroup>
  <Accordion title="缺少映像檔或沙箱容器未啟動">
    使用 [`scripts/sandbox-setup.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/sandbox-setup.sh)（原始碼簽出目錄）或[沙箱機制 § 映像檔與設定](/zh-TW/gateway/sandboxing#images-and-setup)中的內嵌 `docker build` 指令（npm 安裝）建置沙箱映像檔，或將 `agents.defaults.sandbox.docker.image` 設為你的自訂映像檔。系統會視需要自動為每個工作階段建立容器。
  </Accordion>

  <Accordion title="沙箱中的權限錯誤">
    將 `docker.user` 設為與已掛載工作區擁有權相符的 UID:GID，或變更工作區資料夾的擁有者。
  </Accordion>

  <Accordion title="在沙箱中找不到自訂工具">
    OpenClaw 使用 `sh -lc`（登入 Shell）執行指令，它會載入 `/etc/profile`，且可能重設 PATH。設定 `docker.env.PATH`，將自訂工具路徑加到前方，或在 Dockerfile 的 `/etc/profile.d/` 下新增指令碼。
  </Accordion>

  <Accordion title="映像檔建置期間因 OOM 終止（結束代碼 137）">
    VM 至少需要 2 GB RAM。請使用較大的機器類別後重試。
  </Accordion>

  <Accordion title="控制介面中顯示未授權或需要配對">
    取得新的儀表板連結，並核准瀏覽器裝置：

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    docker compose run --rm openclaw-cli devices list
    docker compose run --rm openclaw-cli devices approve <requestId>
    ```

    更多詳細資訊：[儀表板](/zh-TW/web/dashboard)、[裝置](/zh-TW/cli/devices)。

  </Accordion>

  <Accordion title="Docker 命令列介面中的閘道目標顯示 ws://172.x.x.x 或配對錯誤">
    重設閘道模式與繫結：

    ```bash
    docker compose run --rm openclaw-cli config set --batch-json '[{"path":"gateway.mode","value":"local"},{"path":"gateway.bind","value":"lan"}]'
    docker compose run --rm openclaw-cli devices list --url ws://127.0.0.1:18789
    ```

  </Accordion>
</AccordionGroup>

## 相關內容

- [安裝概覽](/zh-TW/install) — 所有安裝方式
- [Podman](/zh-TW/install/podman) — Docker 的 Podman 替代方案
- [ClawDock](/zh-TW/install/clawdock) — 社群提供的 Docker Compose 設定
- [更新](/zh-TW/install/updating) — 讓 OpenClaw 保持最新狀態
- [設定](/zh-TW/gateway/configuration) — 安裝後的閘道設定
