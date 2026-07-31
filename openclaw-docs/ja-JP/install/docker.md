---
read_when:
    - ローカルインストールではなく、コンテナ化された Gateway を使用したい場合
    - Docker フローを検証しています
summary: OpenClaw 用のオプションの Docker ベースのセットアップとオンボーディング
title: Docker
x-i18n:
    generated_at: "2026-07-26T09:06:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c1784bd49f6847db75633840a4d5a8e49205200728bd2e9d59b646a446e508d6
    source_path: install/docker.md
    workflow: 16
---

Docker は**任意**です。分離された使い捨ての Gateway 環境や、ローカルインストールのないホストで使用します。すでに自分のマシンで開発している場合は、代わりに通常のインストールフローを使用してください。

デフォルトのサンドボックスバックエンドでは、`agents.defaults.sandbox` が有効な場合に Docker を使用しますが、サンドボックスはデフォルトで無効になっており、Gateway 自体を Docker で実行する必要はありません。SSH および OpenShell サンドボックスバックエンドも利用できます。[サンドボックス](/ja-JP/gateway/sandboxing)を参照してください。

複数ユーザー向けにホスティングしますか？テナントごとに 1 セルを割り当てるモデルについては、[マルチテナントホスティング](/ja-JP/gateway/multi-tenant-hosting)を参照してください。

## 前提条件

- Docker Desktop（または Docker Engine）+ Docker Compose v2
- イメージのビルドに少なくとも 2 GB の RAM（1 GB のホストでは `pnpm install` が終了コード 137 で OOM Kill される可能性があります）
- イメージとログに十分なディスク容量
- VPS／公開ホストでは、特に Docker の `DOCKER-USER` ファイアウォールチェーンに関する[ネットワーク公開時のセキュリティ強化](/ja-JP/gateway/security)を確認してください

## コンテナ化された Gateway

<Steps>
  <Step title="イメージをビルドする">
    リポジトリのルートから実行します。

    ```bash
    ./scripts/docker/setup.sh
    ```

    これにより、Gateway イメージがローカルで `openclaw:local` としてビルドされます。代わりにビルド済みイメージを使用するには、次のようにします。

    ```bash
    export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
    ./scripts/docker/setup.sh
    ```

    ビルド済みイメージは、最初に [GitHub Container Registry](https://github.com/openclaw/openclaw/pkgs/container/openclaw) に公開されます。GHCR は、リリース自動化、バージョン固定デプロイ、来歴チェックの主要レジストリです。同じリリースの Docker Hub ミラーも `openclaw/openclaw` に公開されます。

    ```bash
    export OPENCLAW_IMAGE="openclaw/openclaw:latest"
    ./scripts/docker/setup.sh
    ```

    `ghcr.io/openclaw/openclaw` または `openclaw/openclaw` を使用し、OpenClaw とリリース時期や保持ポリシーが異なる非公式ミラーは避けてください。バージョン固有のタグには、`2026.2.26` のようなリリースや、`2026.2.26-beta.1` のようなプレリリースがあります。安定版リリースでは `latest` と `main` が更新され、月末の Gateway リリースでは `extended-stable` のみが更新されます。バリアントには、`slim`、`main-slim`、`extended-stable-slim`、`latest-browser`、`main-browser`、`extended-stable-browser` があります。デフォルトイメージには、`codex` および `diagnostics-otel` Plugin が同梱されています。`-browser` バリアントには Chromium も組み込まれており、初回実行時に Playwright をインストールせずに[サンドボックス化されたブラウザ](/ja-JP/gateway/sandboxing#sandboxed-browser)ツールを使用する場合に便利です。

  </Step>

  <Step title="エアギャップ環境で再実行する">
    オフラインホストでは、まずイメージを転送してロードします。

    ```bash
    docker load -i openclaw-image.tar
    export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
    ./scripts/docker/setup.sh --offline
    ```

    `--offline` は、`OPENCLAW_IMAGE` がすでにローカルに存在することを検証し、暗黙的な Compose のプルとビルドを無効にしてから、通常のフロー（`.env` の同期、権限修正、オンボーディング、Gateway 設定の同期、Compose の起動）を実行します。

    `OPENCLAW_SANDBOX=1` の場合、オフラインセットアップでは、`OPENCLAW_DOCKER_SOCKET` の背後にあるデーモン上で、設定済みのデフォルトおよびエージェントごとのサンドボックスイメージも確認します。これには、Docker ベースのブラウザイメージに付与されたブラウザコントラクトラベルも含まれます。必要なイメージが欠落しているか古い場合、セットアップは壊れた状態を成功として報告せず、サンドボックス設定を変更せずに終了します。

  </Step>

  <Step title="オンボーディングを完了する">
    セットアップスクリプトはオンボーディングを自動的に実行します。

    - プロバイダーの API キーの入力を求めます
    - Gateway トークンを生成し、`.env` に書き込みます
    - 認証プロファイルの秘密鍵ディレクトリを作成します
    - Docker Compose 経由で Gateway を起動します

    起動前のオンボーディングと設定の書き込みは、`openclaw-cli` が Gateway のネットワーク名前空間を共有し、Gateway コンテナが存在した後でのみ機能するため、`openclaw-gateway` を直接（`--no-deps --entrypoint node` とともに）介して実行されます。

  </Step>

  <Step title="Control UI を開く">
    `http://127.0.0.1:18789/` を開き、`.env` に書き込まれたトークンを Settings に貼り付けます。コンテナをパスワード認証に切り替えた場合は、代わりにそのパスワードを使用します。

    URL をもう一度確認する必要がありますか？

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    ```

  </Step>

  <Step title="チャンネルを設定する（任意）">
    ```bash
    # WhatsApp（QR）
    docker compose run --rm openclaw-cli channels login

    # Telegram
    docker compose run --rm openclaw-cli channels add --channel telegram --token "<token>"

    # Discord
    docker compose run --rm openclaw-cli channels add --channel discord --token "<token>"
    ```

    ドキュメント：[WhatsApp](/ja-JP/channels/whatsapp)、[Telegram](/ja-JP/channels/telegram)、[Discord](/ja-JP/channels/discord)

  </Step>
</Steps>

### 手動フロー

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

Docker コンテキストでは `.git` が除外されます。イメージの About 画面にチェックアウト済みのコミットと 1 つのビルドタイムスタンプが表示されるよう、上記のとおりソースの識別情報をビルド引数として渡してください。`scripts/docker/setup.sh` は、両方の値を自動的に解決して渡します。

<Note>
`docker compose` はリポジトリのルートから実行してください。`OPENCLAW_EXTRA_MOUNTS` または `OPENCLAW_HOME_VOLUME` を有効にした場合、セットアップスクリプトは `docker-compose.extra.yml` を書き込みます。自身で管理している `docker-compose.override.yml` の後にこれを含めてください（例：`-f docker-compose.yml -f docker-compose.override.yml -f docker-compose.extra.yml`）。
</Note>

### コンテナイメージのアップグレード

同じマウント済みの状態／設定を維持したまま OpenClaw イメージを置き換えると、新しい Gateway は準備完了になる前に、起動時に安全なアップグレード移行と Plugin の収束を実行します。通常のイメージアップグレードでは、別途 `openclaw doctor --fix` を実行する必要はありません。

起動時にこれらの修復を安全に完了できない場合、Gateway は正常と報告せずに終了します。再起動ポリシーが設定されている場合、Docker、Podman、または Kubernetes では Gateway コンテナが再起動を繰り返しているように表示されることがあります。マウント済みの状態ボリュームを維持したまま、Gateway が使用するものと同じ状態／設定マウントを使用し、同じイメージをコンテナコマンドとして `openclaw doctor --fix` を指定して一度実行します。

```bash
docker run --rm -v <openclaw-state>:/home/node/.openclaw <image> openclaw doctor --fix
podman run --rm -v <openclaw-state>:/home/node/.openclaw <image> openclaw doctor --fix
```

doctor の完了後、デフォルトコマンドで Gateway コンテナを再起動します。Kubernetes では、同じ PVC をマウントした一度限りの Job またはデバッグ Pod で同じコマンドを実行してから、Deployment または StatefulSet を再起動します。

### 環境変数

`scripts/docker/setup.sh`（および Gateway コンテナでは `docker-compose.yml` に直接）で使用できる任意の変数：

| 変数                                        | 用途                                                                                                           |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_IMAGE`                                | ローカルでビルドする代わりにリモートイメージを使用する                                                                    |
| `OPENCLAW_IMAGE_APT_PACKAGES`                   | ビルド時に追加の apt パッケージをインストールする（スペース区切り）。レガシーエイリアス：`OPENCLAW_DOCKER_APT_PACKAGES`           |
| `OPENCLAW_IMAGE_PIP_PACKAGES`                   | ビルド時に追加の Python パッケージをインストールする（スペース区切り）                                                      |
| `OPENCLAW_EXTENSIONS`                           | 選択した対応 Plugin をコンパイル／パッケージ化し、そのランタイム依存関係をインストールする（カンマまたはスペース区切りの ID） |
| `OPENCLAW_DOCKER_BUILD_NODE_OPTIONS`            | ローカルのソースビルド用 Node オプションを上書きする（デフォルト：`--max-old-space-size=8192`）                                |
| `OPENCLAW_DOCKER_BUILD_TSDOWN_MAX_OLD_SPACE_MB` | ローカルのソースビルド用 tsdown ヒープを MB 単位で上書きする                                                                 |
| `OPENCLAW_DOCKER_BUILD_SKIP_DTS`                | ランタイム専用のローカルイメージビルド時に宣言の出力をスキップする（デフォルト：`1`）                                      |
| `OPENCLAW_INSTALL_BROWSER`                      | ビルド時に Chromium + Xvfb をイメージに組み込む                                                                 |
| `OPENCLAW_EXTRA_MOUNTS`                         | 追加のホストバインドマウント（カンマ区切りの `source:target[:opts]`）                                                   |
| `OPENCLAW_HOME_VOLUME`                          | `/home/node` を名前付き Docker ボリュームに永続化する                                                                     |
| `OPENCLAW_SANDBOX`                              | サンドボックスのブートストラップを有効にする（`1`、`true`、`yes`、`on`）                                                            |
| `OPENCLAW_SKIP_ONBOARDING`                      | 対話型オンボーディング手順をスキップする（`1`、`true`、`yes`、`on`）                                                   |
| `OPENCLAW_DOCKER_SOCKET`                        | Docker ソケットのパスを上書きする                                                                                   |
| `OPENCLAW_DISABLE_BONJOUR`                      | Bonjour／mDNS のアドバタイズを強制的に有効（`0`）または無効（`1`）にする。[Bonjour／mDNS](#bonjour--mdns)を参照                        |
| `OPENCLAW_DISABLE_BUNDLED_SOURCE_OVERLAYS`      | 同梱 Plugin のソースバインドマウントオーバーレイを無効にする                                                                 |
| `OTEL_EXPORTER_OTLP_ENDPOINT`                   | OpenTelemetry エクスポート用の共有 OTLP／HTTP コレクターエンドポイント                                                      |
| `OTEL_EXPORTER_OTLP_*_ENDPOINT`                 | トレース、メトリクス、またはログ用のシグナル固有 OTLP エンドポイント                                                       |
| `OTEL_EXPORTER_OTLP_PROTOCOL`                   | OTLP プロトコルの上書き。現在サポートされているのは `http/protobuf` のみ                                                   |
| `OTEL_SERVICE_NAME`                             | OpenTelemetry リソースに使用するサービス名                                                                     |
| `OTEL_SEMCONV_STABILITY_OPT_IN`                 | 最新の実験的な GenAI セマンティック属性を有効にする                                                           |
| `OPENCLAW_OTEL_PRELOADED`                       | 1 つがプリロードされている場合に、2 つ目の OpenTelemetry SDK の起動をスキップする                                                    |

公式イメージには Homebrew は含まれていません。オンボーディング中、OpenClaw は `brew` のない Linux コンテナでは brew 専用の Skills 依存関係インストーラーを非表示にします。これらの依存関係はカスタムイメージで提供するか、手動でインストールしてください。Debian パッケージとして提供される依存関係には `OPENCLAW_IMAGE_APT_PACKAGES`、Python の依存関係には `OPENCLAW_IMAGE_PIP_PACKAGES` を使用します（ビルド時に `python3 -m pip install --break-system-packages` を実行するため、バージョンを固定し、信頼できるインデックスのみを使用してください）。

Docker が `ResourceExhausted`、`cannot allocate memory` を報告する場合、または `tsdown` 中に中止する場合は、Docker ビルダーのメモリ上限を増やすか、明示的にヒープを小さくして再試行してください。

```bash
OPENCLAW_DOCKER_BUILD_NODE_OPTIONS=--max-old-space-size=4096 OPENCLAW_DOCKER_BUILD_TSDOWN_MAX_OLD_SPACE_MB=4096
```

### 選択した Plugin を含むソースビルドイメージ

`OPENCLAW_EXTENSIONS` はソースチェックアウトから Plugin マニフェスト ID を選択します。
既存のソースディレクトリ名が異なる場合は、それらも使用できます。Docker
ビルドは、選択内容を一度だけソースディレクトリに解決し、本番環境用の
依存関係をインストールします。また、選択した Plugin が
`openclaw.build.bundledDist: false` とともに個別に公開されている場合は、そのランタイムをルートのバンドル済み
dist にコンパイルします。この Docker 専用のパッケージングによって、Plugin の npm または ClawHub
アーティファクト契約が変更されることはありません。不明、無効、または曖昧な ID があると、イメージのビルドは失敗します。
既知の依存関係専用またはソース専用の ID は、コンパイル済みのルート dist エントリを追加せず、
既存のソースと依存関係のステージングを維持します。統合ビルドエントリを持つ選択済み Plugin は、
正常にコンパイルされる必要があります。選択されていない外部 Plugin の
ソースとランタイム出力は削除されます。

たとえば、以下のコマンドは ClickClack、Slack、Microsoft Teams 用に、個別のマルチアーキテクチャ対応スタンドアロン
FakeCo Gateway イメージをビルドします。ClawRouter は
すでにルート OpenClaw ランタイムの一部であるため、ClickClack イメージでは
`clickclack` のみを選択します。ブラウザー引数を明示的に空にすることで、デフォルトイメージに
Chromium が含まれないようにします。

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

単一のネイティブローカルビルドには `--platform linux/arm64 --load` または `--platform linux/amd64 --load` を使用します。
マルチプラットフォーム出力と、付随する SBOM／プロベナンスには、
アテステーションを保持するレジストリまたは別の Buildx 出力が必要です。プッシュ後、
マニフェストを確認し、可変のソース SHA タグではなく、
不変のダイジェストをデプロイします。

```bash
docker buildx imagetools inspect \
  "${REGISTRY}/openclaw-clickclack:${SOURCE_SHA}"
# デプロイ: registry.example.com/fakeco/openclaw-clickclack@sha256:<manifest-digest>
```

これらのイメージは、スタンドアロンの OCI ベース Gateway と一般的な Docker ユーザー向けです。
Crabhelm が管理する Gateway では使用されません。その配信経路では、
OpenClaw npm tarball を含む別個の x86_64 アプライアンスアーカイブをビルドし、
Node、アーカイブ、マニフェストのダイジェストを固定します。ランディング済みの同じ OpenClaw ソースから、
そのアプライアンスを独立してビルドしてください。

パッケージ化されたイメージに対してバンドル済み Plugin のソースをテストするには、Plugin のソースディレクトリを、そのパッケージ化されたソースパス（例: `OPENCLAW_EXTRA_MOUNTS=/path/to/fork/extensions/synology-chat:/app/extensions/synology-chat:ro`）にマウントします。これにより、同じ Plugin ID に対応するコンパイル済みの `/app/dist/extensions/synology-chat` バンドルが上書きされます。

### オブザーバビリティ

OpenTelemetry のエクスポートは Gateway コンテナから OTLP コレクターへのアウトバウンド通信であり、Docker ポートを公開する必要はありません。ローカルでビルドしたイメージにバンドル済みエクスポーターを含めるには、次のようにします。

```bash
export OPENCLAW_EXTENSIONS="diagnostics-otel"
export OTEL_EXPORTER_OTLP_ENDPOINT="http://otel-collector:4318"
export OTEL_SERVICE_NAME="openclaw-gateway"
./scripts/docker/setup.sh
```

公式のビルド済みイメージには、すでに `diagnostics-otel` がバンドルされています。削除した場合に限り、`clawhub:@openclaw/diagnostics-otel` を自分でインストールしてください。エクスポートを有効にするには、設定で `diagnostics-otel` Plugin を許可して有効化し、`diagnostics.otel.enabled=true` を設定します（完全な例は [OpenTelemetry のエクスポート](/ja-JP/gateway/opentelemetry)を参照してください）。コレクターの認証ヘッダーは Docker 環境変数ではなく、`diagnostics.otel.headers` を介して渡します。

Prometheus メトリクスは、すでに公開されている Gateway ポートを再利用します。`clawhub:@openclaw/diagnostics-prometheus` をインストールし、`diagnostics-prometheus` Plugin を有効にしてから、次の URL をスクレイプします。

```text
http://<gateway-host>:18789/api/diagnostics/prometheus
```

このルートは Gateway 認証によって保護されています。個別の公開 `/metrics` ポートや、認証されていないリバースプロキシパスを公開しないでください。[Prometheus メトリクス](/ja-JP/gateway/prometheus)を参照してください。

### ヘルスチェック

コンテナのプローブエンドポイント（認証不要）:

```bash
curl -fsS http://127.0.0.1:18789/healthz   # liveness
curl -fsS http://127.0.0.1:18789/readyz     # readiness
```

イメージに組み込まれた `HEALTHCHECK` は `/healthz` に ping を送信します。失敗が繰り返されると、コンテナは `unhealthy` とマークされ、オーケストレーターが再起動または置換できるようになります。

認証済みの詳細なヘルススナップショット:

```bash
docker compose exec openclaw-gateway node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"
```

### LAN と loopback

`scripts/docker/setup.sh` のデフォルトは `OPENCLAW_GATEWAY_BIND=lan` であるため、Docker のポート公開を使用してホスト上の `http://127.0.0.1:18789` が動作します。

- `lan`（デフォルト）: ホストのブラウザーとホストの CLI は、公開された Gateway ポートにアクセスできます。
- `loopback`: コンテナのネットワーク名前空間内のプロセスだけが、Gateway に直接アクセスできます。

<Note>
`gateway.bind` では、`0.0.0.0` や `127.0.0.1` のようなホストエイリアスではなく、バインドモード値（`lan` / `loopback` / `custom` / `tailnet` / `auto`）を使用してください。
</Note>

### ホストのローカルプロバイダー

コンテナ内では、`127.0.0.1` はホストではなくコンテナ自体を指します。ホストで実行されているプロバイダーには `host.docker.internal` を使用します。

| プロバイダー  | ホストのデフォルト URL         | Docker セットアップ URL                    |
| --------- | ------------------------ | ----------------------------------- |
| LM Studio | `http://127.0.0.1:1234`  | `http://host.docker.internal:1234`  |
| Ollama    | `http://127.0.0.1:11434` | `http://host.docker.internal:11434` |

バンドル済みのセットアップでは、これらの URL を LM Studio／Ollama のオンボーディングのデフォルトとして使用します。また、`docker-compose.yml` は Linux Docker Engine 上で `host.docker.internal` をホスト Gateway にマッピングします（Docker Desktop は macOS／Windows で同じエイリアスを提供します）。ホストサービスは、Docker からアクセス可能なアドレスで待ち受ける必要があります。

```bash
lms server start --port 1234 --bind 0.0.0.0
OLLAMA_HOST=0.0.0.0:11434 ollama serve
```

独自の Compose ファイルまたは `docker run` を使用していますか？同じマッピングを自分で追加してください（例: `--add-host=host.docker.internal:host-gateway`）。

### Docker 内の Claude CLI バックエンド

公式イメージには Claude Code がプリインストールされていません。コンテナの `node` ユーザーとしてインストールしてログインし、そのコンテナホームを永続化して、イメージのアップグレードでバイナリや認証状態が消去されないようにします。

新規インストールでは、セットアップを実行する前に永続的な `/home/node` ボリュームを有効にします。

```bash
export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
export OPENCLAW_HOME_VOLUME="openclaw_home"
./scripts/docker/setup.sh
```

既存のインストールでは、まずスタックを停止し、現在の `.env` の値を再読み込みします。セットアップスクリプトは常に現在のシェルとデフォルト値から `.env` を書き直し、ファイル自体を自動では読み込みません。

```bash
set -a
. ./.env
set +a
export OPENCLAW_HOME_VOLUME="${OPENCLAW_HOME_VOLUME:-openclaw_home}"
./scripts/docker/setup.sh
```

`.env` にシェルで source できない値が含まれている場合は、依存している値（`OPENCLAW_IMAGE`、ポート、バインドモード、カスタムパス、`OPENCLAW_EXTRA_MOUNTS`、サンドボックス、オンボーディングのスキップ）を最初に手動で再エクスポートしてください。生成されたオーバーレイは、`openclaw-gateway` と `openclaw-cli` の両方にホームボリュームをマウントします。残りのコマンドはそのオーバーレイを使用して実行してください（使用している場合は、最初に `docker-compose.override.yml` も指定します）。

```bash
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  --entrypoint sh openclaw-cli -lc \
  'curl -fsSL https://claude.ai/install.sh | bash'
```

ネイティブインストーラーは `claude` を `/home/node/.local/bin/claude` に書き込みます。
OpenClaw イメージでは `/home/node/.local/bin` が `PATH` に含まれているため、バンドル済みの
Anthropic Plugin はアダプター設定を上書きせずにこれを解決できます。

同じ永続化されたホームからログインして確認します。

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

次に、バンドル済みの `claude-cli` バックエンドを使用します。

```bash
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  openclaw-cli agent \
  --agent main \
  --model claude-cli/claude-sonnet-4-6 \
  --message "Docker Claude CLI からこんにちはと言ってください"
```

`OPENCLAW_HOME_VOLUME` は、`/home/node/.local/bin` と `/home/node/.local/share/claude` 配下のネイティブインストールに加えて、`/home/node/.claude` と `/home/node/.claude.json` 配下の Claude Code の設定／認証を永続化します。`/home/node/.openclaw` だけを永続化しても不十分です。ホームボリュームの代わりに `OPENCLAW_EXTRA_MOUNTS` を使用する場合は、これらすべての Claude パスを両方のサービスにマウントしてください。

<Note>
共有の本番自動化や予測可能な Anthropic の請求には、Anthropic API キーを使用する経路を推奨します。Claude CLI の再利用は、Claude Code のインストール済みバージョン、アカウントログイン、請求、更新の動作に従います。
</Note>

### Bonjour / mDNS

Docker ブリッジネットワークは通常、Bonjour／mDNS マルチキャスト（`224.0.0.251:5353`）を確実には転送しません。`OPENCLAW_DISABLE_BONJOUR` が未設定の場合、バンドル済み Bonjour Plugin はコンテナ内で実行されていることを検出すると LAN アドバタイズを自動的に無効化するため、ブリッジによって破棄されるマルチキャストを再試行してクラッシュループに陥ることはありません。検出結果にかかわらず強制的に無効にするには `OPENCLAW_DISABLE_BONJOUR=1` を、有効にするには `0` を設定します（ホストネットワーク、macvlan、または mDNS マルチキャストが動作することが確認されている別のネットワークでのみ使用してください）。

それ以外の Docker ホストでは、公開された Gateway URL、Tailscale、または広域 DNS-SD を使用してください。注意点とトラブルシューティングについては、[Bonjour ディスカバリー](/ja-JP/gateway/bonjour)を参照してください。

### ストレージと永続化

Docker Compose は `OPENCLAW_CONFIG_DIR` を `/home/node/.openclaw` に、`OPENCLAW_WORKSPACE_DIR` を `/home/node/.openclaw/workspace` に、`OPENCLAW_AUTH_PROFILE_SECRET_DIR` を `/home/node/.config/openclaw` にバインドマウントするため、これらのパスはコンテナを置換しても保持されます。変数が未設定の場合、`docker-compose.yml` は `${HOME}` 配下にフォールバックします。`HOME` 自体が存在しない場合は `/tmp` にフォールバックするため、何も設定されていない環境でも `docker compose up` がソースの空のボリューム指定を生成することはありません。

そのマウントされた設定ディレクトリには、以下が保存されます。

- `openclaw.json`: 動作設定
- `agents/<agentId>/agent/auth-profiles.json`: 保存されたプロバイダーの OAuth／API キー認証
- `.env`: `OPENCLAW_GATEWAY_TOKEN` など、環境変数を基盤とするランタイムシークレット

認証プロファイルのシークレットディレクトリには、OAuth を基盤とする認証プロファイルのトークン情報を暗号化するためのローカル暗号化キーが保存されます。Docker ホストの状態とともに保持しつつ、`OPENCLAW_CONFIG_DIR` とは分けてください。

インストールされたダウンロード可能な Plugin は、マウントされた OpenClaw ホーム配下にパッケージ状態を保存するため、インストール記録とパッケージルートはコンテナを置換しても保持されます。Gateway の起動時に、バンドル済み Plugin の依存関係ツリーが再生成されることはありません。

VM 全体の永続化について詳しくは、[Docker VM ランタイム - 各データの永続化場所](/ja-JP/install/docker-vm-runtime#what-persists-where)を参照してください。

**ディスク使用量が増えやすい箇所:** `media/`、エージェントごとの SQLite データベース、従来のセッション JSONL トランスクリプト、共有 SQLite 状態データベース、インストール済み Plugin のパッケージルート、および `/tmp/openclaw/` 配下のローテーションファイルログ。

### シェルヘルパー（任意）

日常的なコマンドを短くするには、[ClawDock](/ja-JP/install/clawdock)をインストールします。

```bash
mkdir -p ~/.clawdock && curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/clawdock/clawdock-helpers.sh -o ~/.clawdock/clawdock-helpers.sh
echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc
```

以前の `scripts/shell-helpers/clawdock-helpers.sh` パスからインストールした場合は、上記のコマンドを再実行して、ローカルヘルパーが現在の場所を参照するようにしてください。その後、`clawdock-start`、`clawdock-stop`、`clawdock-dashboard` などを使用できます（完全な一覧を表示するには `clawdock-help` を実行してください）。

<AccordionGroup>
  <Accordion title="Docker Gateway のエージェントサンドボックスを有効にする">
    ```bash
    export OPENCLAW_SANDBOX=1
    ./scripts/docker/setup.sh
    ```

    カスタムソケットパス（例: rootless Docker）:

    ```bash
    export OPENCLAW_SANDBOX=1
    export OPENCLAW_DOCKER_SOCKET=/run/user/1000/docker.sock
    ./scripts/docker/setup.sh
    ```

    このスクリプトは、サンドボックスの前提条件を満たした後にのみ `docker.sock` をマウントします。サンドボックスのセットアップを完了できない場合、`agents.defaults.sandbox.mode` を `off` にリセットします。OpenClaw サンドボックスが有効なターンでは、Codex コードモードは無効になります（[サンドボックス化 § Docker バックエンド](/ja-JP/gateway/sandboxing#docker-backend)を参照）。ホストの Docker ソケットをエージェントサンドボックスコンテナにマウントしないでください。

  </Accordion>

  <Accordion title="自動化 / CI（非対話式）">
    `-T` を使用して Compose の疑似 TTY 割り当てを無効にします。

    ```bash
    docker compose run -T --rm openclaw-cli gateway probe
    docker compose run -T --rm openclaw-cli devices list --json
    ```

  </Accordion>

  <Accordion title="共有ネットワークのセキュリティに関する注意">
    `openclaw-cli` は `network_mode: "service:openclaw-gateway"` を使用するため、CLI コマンドは `127.0.0.1` 経由で Gateway にアクセスできます。これを共有の信頼境界として扱ってください。Compose 設定では、`openclaw-gateway` と `openclaw-cli` の両方で `NET_RAW`/`NET_ADMIN` を削除し、`no-new-privileges` を有効にします。
  </Accordion>

  <Accordion title="openclaw-cli での Docker Desktop の DNS エラー">
    一部の Docker Desktop セットアップでは、`NET_RAW` の削除後、共有ネットワークの `openclaw-cli` サイドカーからの DNS ルックアップに失敗し、`openclaw plugins install` のような npm ベースのコマンドで `EAI_AGAIN` として表示されます。通常の運用では、デフォルトの強化済み Compose ファイルを使用してください。以下のオーバーライドは、`openclaw-cli` コンテナだけにデフォルトのケーパビリティを復元します。デフォルトの呼び出し方法としてではなく、レジストリアクセスを必要とする単発のコマンドに使用してください。

    ```bash
    printf '%s\n' \
      'services:' \
      '  openclaw-cli:' \
      '    cap_drop: !reset []' \
      > docker-compose.cli-no-dropped-caps.local.yml

    docker compose -f docker-compose.yml -f docker-compose.cli-no-dropped-caps.local.yml run --rm openclaw-cli plugins install <package>
    ```

    長時間稼働する `openclaw-cli` コンテナをすでに作成している場合は、同じオーバーライドを使用して再作成してください。`docker compose exec`/`docker exec` では、作成済みのコンテナの Linux ケーパビリティを変更できません。

  </Accordion>

  <Accordion title="権限と EACCES">
    イメージは `node`（uid 1000）として実行されます。`/home/node/.openclaw` で権限エラーが発生した場合は、ホストのバインドマウントが uid 1000 によって所有されていることを確認してください。

    ```bash
    sudo chown -R 1000:1000 /path/to/openclaw-config /path/to/openclaw-workspace
    ```

    同じ不一致が、`blocked plugin candidate: suspicious ownership (... uid=1000, expected uid=0 or root)` に続く `plugin present but blocked` として現れることがあります。これは、プロセスの uid とマウントされた Plugin ディレクトリの所有者が一致していないことを示します。デフォルトの uid 1000 で実行し、バインドマウントの所有権を修正する方法を推奨します。OpenClaw を長期的に root として実行する意図がある場合に限り、`/path/to/openclaw-config/npm` を `root:root` に chown してください。

  </Accordion>

  <Accordion title="再ビルドの高速化">
    ロックファイルが変更されない限り `pnpm install` が再実行されないように、依存関係レイヤーがキャッシュされる順序で Dockerfile を構成します。

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

  <Accordion title="上級ユーザー向けコンテナオプション">
    デフォルトのイメージはセキュリティを最優先し、非 root の `node` として実行されます。より多機能なコンテナにするには、以下を使用します。

    1. **`/home/node` を永続化**: `export OPENCLAW_HOME_VOLUME="openclaw_home"`
    2. **システム依存関係を組み込む**: `export OPENCLAW_IMAGE_APT_PACKAGES="git curl jq"`
    3. **Python 依存関係を組み込む**: `export OPENCLAW_IMAGE_PIP_PACKAGES="requests==2.32.5 humanize==4.14.0"`
    4. **Playwright Chromium を組み込む**: `export OPENCLAW_INSTALL_BROWSER=1`、または公式の `-browser` イメージタグを使用
    5. **または、Playwright ブラウザを永続化ボリュームにインストール**:
       ```bash
       docker compose run --rm openclaw-cli \
         node /app/node_modules/playwright-core/cli.js install chromium
       ```
    6. **ブラウザのダウンロードを永続化**: `OPENCLAW_HOME_VOLUME` または `OPENCLAW_EXTRA_MOUNTS` を使用します。OpenClaw は、Linux 上でイメージの Playwright 管理 Chromium を自動検出します。

  </Accordion>

  <Accordion title="OpenAI Codex OAuth（ヘッドレス Docker）">
    ウィザードで OpenAI Codex OAuth を選択すると、ブラウザの URL が開きます。Docker またはヘッドレス環境では、最終的に表示された完全なリダイレクト URL をコピーし、ウィザードに貼り付けて認証を完了します。
  </Accordion>

  <Accordion title="ベースイメージのメタデータ">
    ランタイムイメージは `node:24-bookworm-slim` を使用し、`tini` を PID 1 として実行するため、長時間稼働するコンテナでゾンビプロセスが回収され、シグナルが正しく処理されます。`org.opencontainers.image.base.name` や `org.opencontainers.image.source` を含む OCI ベースイメージアノテーションを公開します。Dependabot は固定された Node ベースダイジェストを更新します。リリースビルドでは、ディストリビューションをアップグレードする別レイヤーは実行されません。[OCI イメージアノテーション](https://github.com/opencontainers/image-spec/blob/main/annotations.md)を参照してください。
  </Accordion>
</AccordionGroup>

### VPS で実行しますか？

バイナリの組み込み、永続化、更新を含む共有 VM へのデプロイ手順については、[Hetzner（Docker VPS）](/ja-JP/install/hetzner)および[Docker VM ランタイム](/ja-JP/install/docker-vm-runtime)を参照してください。

## エージェントサンドボックス

Docker バックエンドで `agents.defaults.sandbox` が有効な場合、Gateway 自体はホスト上に残したまま、エージェントのツール実行（シェル、ファイルの読み書きなど）を隔離された Docker コンテナ内で実行します。Gateway 全体をコンテナ化することなく、信頼できないエージェントセッションやマルチテナントのエージェントセッションを強固に分離できます。

サンドボックスのスコープは、エージェント単位（デフォルト）、セッション単位、または共有に設定できます。各スコープには、`/workspace` にマウントされた独自のワークスペースが割り当てられます。ツールの許可/拒否ポリシー、ネットワーク分離、リソース制限、ブラウザコンテナも設定できます。

完全な設定、イメージ、セキュリティ上の注意事項、マルチエージェントプロファイルについては、以下を参照してください。

- [サンドボックス化](/ja-JP/gateway/sandboxing) -- サンドボックスの完全なリファレンス
- [OpenShell](/ja-JP/gateway/openshell) -- サンドボックスコンテナへの対話型シェルアクセス
- [マルチエージェントのサンドボックスとツール](/ja-JP/tools/multi-agent-sandbox-tools) -- エージェントごとのオーバーライド

### クイック有効化

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // off | non-main | all
        scope: "agent", // session | agent | shared
      },
    },
  },
}
```

デフォルトのサンドボックスイメージをビルドします（ソースチェックアウトから）。

```bash
scripts/sandbox-setup.sh
```

ソースチェックアウトを使用しない npm インストールについては、インラインの `docker build` コマンドを[サンドボックス化 § イメージとセットアップ](/ja-JP/gateway/sandboxing#images-and-setup)で参照してください。

## トラブルシューティング

<AccordionGroup>
  <Accordion title="イメージが見つからない、またはサンドボックスコンテナが起動しない">
    [`scripts/sandbox-setup.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/sandbox-setup.sh)（ソースチェックアウト）または[サンドボックス化 § イメージとセットアップ](/ja-JP/gateway/sandboxing#images-and-setup)のインライン `docker build` コマンド（npm インストール）を使用してサンドボックスイメージをビルドするか、`agents.defaults.sandbox.docker.image` をカスタムイメージに設定します。コンテナは、必要に応じてセッションごとに自動作成されます。
  </Accordion>

  <Accordion title="サンドボックス内の権限エラー">
    `docker.user` を、マウントされたワークスペースの所有権に一致する UID:GID に設定するか、ワークスペースフォルダーを chown します。
  </Accordion>

  <Accordion title="サンドボックス内でカスタムツールが見つからない">
    OpenClaw は `sh -lc`（ログインシェル）でコマンドを実行します。このシェルは `/etc/profile` を読み込み、PATH をリセットする場合があります。`docker.env.PATH` を設定してカスタムツールのパスを先頭に追加するか、Dockerfile 内の `/etc/profile.d/` 配下にスクリプトを追加します。
  </Accordion>

  <Accordion title="イメージビルド中に OOM で強制終了される（終了コード 137）">
    VM には少なくとも 2 GB の RAM が必要です。より大きなマシンクラスを使用して再試行してください。
  </Accordion>

  <Accordion title="Control UI で未認証またはペアリングが必要と表示される">
    新しいダッシュボードリンクを取得し、ブラウザデバイスを承認します。

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    docker compose run --rm openclaw-cli devices list
    docker compose run --rm openclaw-cli devices approve <requestId>
    ```

    詳細: [ダッシュボード](/ja-JP/web/dashboard)、[デバイス](/ja-JP/cli/devices)。

  </Accordion>

  <Accordion title="Gateway のターゲットに ws://172.x.x.x が表示される、または Docker CLI でペアリングエラーが発生する">
    Gateway のモードとバインドをリセットします。

    ```bash
    docker compose run --rm openclaw-cli config set --batch-json '[{"path":"gateway.mode","value":"local"},{"path":"gateway.bind","value":"lan"}]'
    docker compose run --rm openclaw-cli devices list --url ws://127.0.0.1:18789
    ```

  </Accordion>
</AccordionGroup>

## 関連項目

- [インストールの概要](/ja-JP/install) — すべてのインストール方法
- [Podman](/ja-JP/install/podman) — Docker の代替となる Podman
- [ClawDock](/ja-JP/install/clawdock) — コミュニティによる Docker Compose セットアップ
- [更新](/ja-JP/install/updating) — OpenClaw を最新の状態に維持する方法
- [設定](/ja-JP/gateway/configuration) — インストール後の Gateway 設定
