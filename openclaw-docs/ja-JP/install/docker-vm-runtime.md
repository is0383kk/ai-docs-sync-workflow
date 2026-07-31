---
read_when:
    - Docker を使用してクラウド VM に OpenClaw をデプロイしています
    - 共有バイナリのビルド、永続化、更新フローが必要です
summary: 長期稼働する OpenClaw Gateway ホスト向けの共有 Docker VM ランタイム手順
title: Docker VM ランタイム
x-i18n:
    generated_at: "2026-07-26T10:17:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d1c474b1f826077ac03c7aaa1e334ed2f38d2de2770f32f2cc907846ecc8bb19
    source_path: install/docker-vm-runtime.md
    workflow: 16
---

GCP、Hetzner、および同様の VPS プロバイダーなど、VM ベースの Docker インストールに共通するランタイム手順です。

## 必要なバイナリをイメージに組み込む

実行中のコンテナ内にバイナリをインストールするのは避けてください。ランタイムにインストールしたものは、再起動すると失われます。スキルに必要なすべての外部バイナリを、ビルド時にイメージへ組み込んでください。

以下の例では、アルファベット順に 3 つのバイナリのみを取り上げます。

- `gog`（`gogcli` 提供）：Gmail へのアクセス用
- `goplaces`：Google Places 用
- `wacli`：WhatsApp 用

これらは例であり、完全な一覧ではありません。同じパターンを使用して、スキルに必要な数だけバイナリをインストールしてください。後から新しいバイナリを必要とするスキルを追加する場合は、次の手順を実行します。

1. Dockerfile を更新します。
2. イメージを再ビルドします。
3. コンテナを再起動します。

**Dockerfile の例**

```dockerfile
FROM node:24-bookworm

RUN apt-get update && apt-get install -y socat && rm -rf /var/lib/apt/lists/*

# バイナリ例 1：Gmail CLI（gogcli — `gog` としてインストール）
# 現在の Linux アセット URL を https://github.com/steipete/gogcli/releases からコピー
RUN curl -L https://github.com/steipete/gogcli/releases/latest/download/gogcli_linux_amd64.tar.gz \
  | tar -xzO gog > /usr/local/bin/gog; \
  chmod +x /usr/local/bin/gog

# バイナリ例 2：Google Places CLI
# 現在の Linux アセット URL を https://github.com/steipete/goplaces/releases からコピー
RUN curl -L https://github.com/steipete/goplaces/releases/latest/download/goplaces_linux_amd64.tar.gz \
  | tar -xzO goplaces > /usr/local/bin/goplaces; \
  chmod +x /usr/local/bin/goplaces

# バイナリ例 3：WhatsApp CLI
# 現在の Linux アセット URL を https://github.com/steipete/wacli/releases からコピー
RUN curl -L https://github.com/steipete/wacli/releases/latest/download/wacli-linux-amd64.tar.gz \
  | tar -xzO wacli > /usr/local/bin/wacli; \
  chmod +x /usr/local/bin/wacli

# 同じパターンを使用して、以下にさらにバイナリを追加

WORKDIR /app
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY ui/package.json ./ui/package.json
COPY scripts ./scripts

RUN corepack enable
RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm build
RUN pnpm ui:install
RUN pnpm ui:build

ENV NODE_ENV=production

CMD ["node","dist/index.js"]
```

<Note>
上記の URL は例です。ARM ベースの VM では、`arm64` アセットを選択してください。再現可能なビルドにするには、バージョンが指定されたリリース URL に固定してください。
</Note>

## ビルドと起動

```bash
docker compose build
docker compose up -d openclaw-gateway
```

`pnpm install --frozen-lockfile` の実行中に、`Killed` または終了コード 137 でビルドが失敗する場合、VM のメモリが不足しています。再試行する前に、より大きなマシンクラスを使用してください。

バイナリを確認します。

```bash
docker compose exec openclaw-gateway which gog
docker compose exec openclaw-gateway which goplaces
docker compose exec openclaw-gateway which wacli
```

期待される出力：

```text
/usr/local/bin/gog
/usr/local/bin/goplaces
/usr/local/bin/wacli
```

Gateway が起動していることを確認します。

```bash
docker compose logs -f openclaw-gateway
curl -fsS http://127.0.0.1:18789/healthz
```

`/healthz` が 200 レスポンスを返せば、Gateway プロセスがリクエストを待ち受けており、正常であることを確認できます。組み込みイメージの `HEALTHCHECK` も同じエンドポイントをポーリングします。

## 永続化される場所

OpenClaw は Docker 内で実行されますが、Docker は信頼できる唯一の情報源ではありません。長期間保持するすべての状態は、再起動、再ビルド、およびリブート後も維持される必要があります。

| コンポーネント              | 場所                                               | 永続化方式  | 注記                                                                                                               |
| ---------------------- | ------------------------------------------------------ | ---------------------- | ------------------------------------------------------------------------------------------------------------------- |
| Gateway 設定         | `/home/node/.openclaw/`                                | ホストボリュームのマウント      | `openclaw.json` を含む                                                                                            |
| チャンネル／プロバイダーの認証情報 | `/home/node/.openclaw/credentials/`                    | ホストボリュームのマウント      | チャンネルおよびプロバイダーの認証情報                                                                            |
| モデル認証プロファイル    | `/home/node/.openclaw/agents/`                         | ホストボリュームのマウント      | `agents/<agentId>/agent/auth-profiles.json`（OAuth、API キー）                                                       |
| レガシー OAuth キーファイル  | `/home/node/.config/openclaw/`                         | ホストボリュームのマウント      | 移行前の OAuth サイドカー向け読み取り専用互換性。`openclaw doctor --fix` はこれらを `auth-profiles.json` に移行する |
| スキル設定          | `/home/node/.openclaw/skills/`                         | ホストボリュームのマウント      | スキルレベルの状態                                                                                                   |
| エージェントワークスペース        | `/home/node/.openclaw/workspace/`                      | ホストボリュームのマウント      | コードおよびエージェントの成果物                                                                                            |
| WhatsApp セッション       | `/home/node/.openclaw/`                                | ホストボリュームのマウント      | QR ログインを維持                                                                                                  |
| Gmail キーリング          | `/home/node/.openclaw/`                                | ホストボリューム＋パスワード | `GOG_KEYRING_PASSWORD` が必要                                                                                     |
| Plugin パッケージ        | `/home/node/.openclaw/npm`, `/home/node/.openclaw/git` | ホストボリュームのマウント      | ダウンロード可能な Plugin パッケージのルート                                                                                   |
| 外部バイナリ      | `/usr/local/bin/`                                      | Docker イメージ           | ビルド時に組み込む必要がある                                                                                         |
| Node ランタイム           | コンテナファイルシステム                                   | Docker イメージ           | イメージのビルドごとに再構築                                                                                           |
| OS パッケージ            | コンテナファイルシステム                                   | Docker イメージ           | ランタイムにはインストールしない                                                                                           |
| Docker コンテナ       | 一時的                                              | 再起動可能            | 破棄しても安全                                                                                                     |

## 更新

VM 上の OpenClaw を更新するには、次を実行します。

```bash
git pull
docker compose build
docker compose up -d
```

## 関連項目

- [Docker](/ja-JP/install/docker)
- [Podman](/ja-JP/install/podman)
- [ClawDock](/ja-JP/install/clawdock)
