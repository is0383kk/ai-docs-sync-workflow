---
read_when:
    - 同じマシン上で複数の Gateway を実行する
    - Gateway ごとに分離された設定、状態、ポートが必要です
summary: 1台のホストで複数のOpenClaw Gatewayを実行する（分離、ポート、プロファイル）
title: 複数の Gateway
x-i18n:
    generated_at: "2026-07-26T09:35:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 655fa865a98064d7c017a7c2eb08ea9a9683002d96a3dbe45a8c16cbd3c86ba1
    source_path: gateway/multiple-gateways.md
    workflow: 16
---

ほとんどのセットアップでは Gateway は 1 つで十分です。1 つの Gateway で複数のメッセージング接続とエージェントを処理できます。より強力な分離や冗長性（例: レスキューボット）が必要な場合にのみ、プロファイルとポートを分離して複数の Gateway を実行してください。

## レスキューボットのクイックスタート

最も簡単なレスキューボットのセットアップ:

- メインボットはデフォルトプロファイルのままにします。
- レスキューボットは、独自の Telegram ボットトークンを使用して `--profile rescue` で実行します。
- レスキューボットには別のベースポート（例: `19789`）を設定します。

これにより、プライマリボットが停止していても、レスキューボットでデバッグや設定変更を行えます。派生するブラウザ/CDP ポートが衝突しないように、ベースポート間には少なくとも 20 ポートの間隔を空けてください。

```bash
# レスキューボット（別の Telegram ボット、別のプロファイル、ポート 19789）
openclaw --profile rescue onboard
openclaw --profile rescue gateway install --port 19789
```

メインボットがすでに実行中であれば、通常はこれだけで十分です。オンボーディングですでにレスキューサービスがインストールされている場合は、最後の `gateway install` を省略してください。

`openclaw --profile rescue onboard` の実行中:

- レスキューアカウント専用の別の Telegram ボットトークンを使用します（オペレーター専用に保ちやすく、メインボットのチャンネル/アプリのインストールから独立し、DM ベースの簡単な復旧経路になります）。
- `rescue` というプロファイル名を維持します。
- メインボットより少なくとも 20 大きいベースポートを使用します。
- 自分ですでに管理しているワークスペースがない限り、デフォルトのレスキューワークスペースを使用します。

### `--profile rescue onboard` による変更内容

`--profile rescue onboard` は通常のオンボーディングフローを実行しますが、すべてを別のプロファイルに書き込むため、レスキューボットには次のものが個別に作成されます:

- プロファイル/設定ファイル
- 状態ディレクトリ
- ワークスペース（デフォルト: `~/.openclaw/workspace-rescue`）
- 管理対象サービス名
- ベースポート（および派生ポート）
- Telegram ボットトークン

それ以外のプロンプトは通常のオンボーディングと同じです。

## 一般的な複数 Gateway のセットアップ

同じ分離パターンは、1 台のホスト上の任意の Gateway の組み合わせやグループに使用できます。追加する各 Gateway に、固有の名前付きプロファイルとベースポートを割り当てます:

```bash
# メイン（デフォルトプロファイル）
openclaw setup
openclaw gateway --port 18789

# 追加の Gateway
openclaw --profile ops setup
openclaw --profile ops gateway --port 19789
```

両方で名前付きプロファイルを使用することもできます:

```bash
openclaw --profile main setup
openclaw --profile main gateway --port 18789

openclaw --profile ops setup
openclaw --profile ops gateway --port 19789
```

サービスも同じパターンに従います:

```bash
openclaw gateway install
openclaw --profile ops gateway install --port 19789
```

フォールバック用のオペレーター経路にはレスキューボットのクイックスタートを使用し、異なるチャンネル、テナント、ワークスペース、または運用ロールにまたがる複数の常駐 Gateway には一般的なプロファイルパターンを使用してください。

## 分離チェックリスト

Gateway インスタンスごとに、以下を一意にしてください:

| 設定                         | 目的                                   |
| ---------------------------- | -------------------------------------- |
| `OPENCLAW_CONFIG_PATH`       | インスタンスごとの設定ファイル         |
| `OPENCLAW_STATE_DIR`         | インスタンスごとのセッション、認証情報、キャッシュ |
| `agents.defaults.workspace`  | インスタンスごとのワークスペースルート |
| `gateway.port`（または `--port`） | インスタンスごとに一意                  |
| 派生するブラウザ/CDP ポート | 以下を参照                             |

これらのいずれかを共有すると、設定、状態、またはポートの競合が発生します。
`OPENCLAW_ALLOW_MULTI_GATEWAY=1` によって設定ごとの単一インスタンス制約がスキップされる場合でも、Gateway の起動時には状態ディレクトリの所有権が一意であることが強制されます。

## ポートマッピング（派生）

ベースポート = `gateway.port`（または `OPENCLAW_GATEWAY_PORT` / `--port`）。

- ブラウザ制御サービスのポート = ベース + 2（local loopback のみ）。
- Canvas ホストは Gateway HTTP サーバー自体（`gateway.port` と同じポート）で提供されます。
- ブラウザプロファイルの CDP ポートは、`browser control port + 9` から `+ 108` の範囲で自動的に割り当てられます。

これらを設定または環境変数で上書きする場合は、インスタンスごとに一意にする必要があります。

## ブラウザ/CDP に関する注意（よくある落とし穴）

- 複数のインスタンスで `browser.cdpUrl` を同じ値に固定しては**いけません**。
- 各インスタンスには、（Gateway ポートから派生する）固有のブラウザ制御ポートと CDP 範囲が必要です。
- CDP ポートを明示的に指定する場合は、インスタンスごとに `browser.profiles.<name>.cdpPort` を設定します。
- リモート Chrome には、プロファイルおよびインスタンスごとに `browser.profiles.<name>.cdpUrl` を使用します。

## 手動での環境変数設定例

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/main.json \
OPENCLAW_STATE_DIR=~/.openclaw \
openclaw gateway --port 18789

OPENCLAW_CONFIG_PATH=~/.openclaw/rescue.json \
OPENCLAW_STATE_DIR=~/.openclaw-rescue \
openclaw gateway --port 19789
```

## 簡易チェック

```bash
openclaw gateway status --deep
openclaw --profile rescue gateway status --deep
openclaw --profile rescue gateway probe
openclaw status
openclaw --profile rescue status
openclaw --profile rescue browser status
```

- `gateway status --deep` は、以前のインストールで残された古い launchd/systemd/schtasks サービスを検出します。
- `gateway probe` の警告テキスト（`multiple reachable gateway identities detected` など）は、意図的に複数の分離された Gateway を実行している場合、または到達可能なプローブ対象が同じ Gateway であることを OpenClaw が確認できない場合にのみ表示されるのが正常です。同じ Gateway への SSH トンネル、プロキシ URL、または設定済みのリモート URL は、転送ポートが異なる場合でも、複数の転送経路を持つ 1 つの Gateway です。

## 関連項目

- [Gateway 運用手順書](/ja-JP/gateway)
- [Gateway ロック](/ja-JP/gateway/gateway-lock)
- [設定](/ja-JP/gateway/configuration)
