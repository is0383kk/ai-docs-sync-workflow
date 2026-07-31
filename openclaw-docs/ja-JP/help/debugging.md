---
read_when:
    - 推論の漏洩がないか、モデルの生出力を確認する必要があります
    - 反復開発中に Gateway をウォッチモードで実行したい場合
    - 再現可能なデバッグワークフローが必要です
summary: デバッグツール：監視モード、生のモデルストリーム、推論漏洩のトレース
title: デバッグ
x-i18n:
    generated_at: "2026-07-26T09:24:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 45a1196c03e4deede3ce47553e1b2b3e1903ee04fe6855d929e0c32bf4e5e686
    source_path: help/debugging.md
    workflow: 16
---

ストリーミング出力、Gateway の反復開発、起動プロファイリング用のデバッグヘルパー。

## ランタイムデバッグのオーバーライド

`/debug` は、**ランタイムのみ**の設定オーバーライド（メモリ上、ディスク上ではない）を設定します。デフォルトでは無効です。`commands.debug: true` で有効にします。

```text
/debug show
/debug set channels.whatsapp.responsePrefix="[openclaw]"
/debug unset channels.whatsapp.responsePrefix
/debug reset
```

`/debug reset` はすべてのオーバーライドを消去し、ディスク上の設定に戻します。

## セッショントレース出力

`/trace` は、完全な詳細モードを有効にせずに、1 つのセッションについて Plugin が所有するトレース／デバッグ行を表示します。Active Memory のデバッグ概要など、Plugin の診断にはこれを使用し、通常のステータス／ツール出力には `/verbose` を使用します。

```text
/trace
/trace on
/trace off
```

## Plugin ライフサイクルトレース

Plugin のメタデータ、検出、レジストリ、ランタイムミラー、設定の変更、更新処理をフェーズごとに分析するには、`OPENCLAW_PLUGIN_LIFECYCLE_TRACE=1` を設定します。stderr に書き込むため、JSON コマンド出力は引き続き解析できます。
このトレースが有効な間、Plugin の読み込み失敗にはスタックトレースが含まれます。

```bash
OPENCLAW_PLUGIN_LIFECYCLE_TRACE=1 openclaw plugins install tokenjuice --force
```

```text
[plugins:lifecycle] phase="config read" ms=6.83 status=ok command="install"
[plugins:lifecycle] phase="slot selection" ms=94.31 status=ok command="install" pluginId="tokenjuice"
[plugins:lifecycle] phase="registry refresh" ms=51.56 status=ok command="install" reason="source-changed"
```

CPU プロファイラーを使用する前に、まずこれを使用してください。ソースチェックアウトでは、`pnpm build` の後に `node dist/entry.js ...` を使用してビルド済みランタイムを測定します。`pnpm openclaw ...` では、ソースランナーのオーバーヘッドも測定されます。

同期的なモジュール読み込み時間には、Plugin 専用の別の環境スイッチではなく、共通の診断サーフェスを使用します。

```bash
OPENCLAW_DIAGNOSTICS=plugin.load-profile openclaw plugins list
```

## CLI の起動とコマンドのプロファイリング

リポジトリに登録済みの起動ベンチマーク：

```bash
pnpm test:startup:bench:smoke
pnpm tsx scripts/bench-cli-startup.ts --preset real --case status --runs 3
pnpm tsx scripts/bench-cli-startup.ts --preset real --cpu-prof-dir .artifacts/cli-cpu
```

通常のソースランナーを介して一時的にプロファイリングするには、`OPENCLAW_RUN_NODE_CPU_PROF_DIR` を設定します。

```bash
OPENCLAW_RUN_NODE_CPU_PROF_DIR=.artifacts/cli-cpu pnpm openclaw status
```

ソースランナーは Node の CPU プロファイルフラグを追加し、コマンドの `.cpuprofile` を書き込みます。コマンドコードに一時的な計測処理を追加する前に、これを使用してください。

同期的なファイルシステム処理やモジュールローダー処理に見える起動停止を調査するには、ソースランナーを介して Node の同期 I/O トレースフラグを追加します。

```bash
OPENCLAW_TRACE_SYNC_IO=1 pnpm openclaw gateway --force
```

`pnpm gateway:watch` では、監視対象の Gateway 子プロセスに対して、このフラグはデフォルトで無効のままです。監視モードでも同期 I/O トレース出力が必要な場合は、`OPENCLAW_TRACE_SYNC_IO=1` を設定します。

## Gateway 監視モード

```bash
pnpm gateway:watch
```

デフォルトでは、`openclaw-gateway-watch-<profile>` という名前（たとえば `openclaw-gateway-watch-main`）の tmux セッションを開始または再起動します。`OPENCLAW_GATEWAY_PORT` がデフォルトポート `18789` と異なる場合のみ、`openclaw-gateway-watch-dev-19001` のようなポートサフィックスが追加されます。対話型ターミナルからは自動的にアタッチされます。非対話型シェル、CI、エージェントの exec 呼び出しはデタッチされたままとなり、代わりにアタッチ手順が表示されます。

```bash
tmux attach -t openclaw-gateway-watch-main
# アタッチせずに最近の出力を読み取る
tmux capture-pane -ep -t openclaw-gateway-watch-main -S -200
```

ペインは tmux の `remain-on-exit` を使用するため、起動失敗時にもセッションが削除されず、アタッチまたはキャプチャできます。`pnpm gateway:watch` を再実行すると、そのペインが再生成されます。

tmux ペインは次の未加工のウォッチャーを実行します。

```bash
node scripts/watch-node.mjs gateway --force
```

設定済み／デフォルトのポートを監視する前に、tmux ラッパーはアクティブなプロファイルにインストールされている Gateway サービスを停止します。これにより、launchd、systemd、Scheduled Task が再生成して置き換えることなく、ポートをソースウォッチャーに引き渡せます。サービスはインストールされたままです。監視セッションの終了後、次のコマンドで復元します。

```bash
pnpm openclaw gateway start
```

明示的な `--port` または `OPENCLAW_GATEWAY_PORT` が、インストール済みサービスの実効ポートと異なる場合、ラッパーはサービスを実行したままにするため、両方の Gateway を並行して実行できます。

tmux を使用しないフォアグラウンドモード：

```bash
pnpm gateway:watch:raw
# または
OPENCLAW_GATEWAY_WATCH_TMUX=0 pnpm gateway:watch
```

未加工モードでは、インストール済みサービスは管理されません。同じポートを使用する場合は、先に `pnpm openclaw gateway stop` を実行してください。

tmux 管理を維持しつつ、自動アタッチを無効にするには：

```bash
OPENCLAW_GATEWAY_WATCH_ATTACH=0 pnpm gateway:watch
```

起動時／ランタイムのホットスポットをデバッグする際に、監視対象 Gateway の CPU 時間をプロファイリングします。

```bash
pnpm gateway:watch --benchmark
```

監視ラッパーは、Gateway を呼び出す前に `--benchmark` を消費し、Gateway 子プロセスが終了するたびに、`.artifacts/gateway-watch-profiles/` 配下へ V8 の `.cpuprofile` を 1 つ書き込みます。現在のプロファイルをフラッシュするには、監視対象 Gateway を停止または再起動し、Chrome DevTools または Speedscope で開きます。

```bash
npx speedscope .artifacts/gateway-watch-profiles/*.cpuprofile
```

- `--benchmark-dir <path>`：プロファイルを別の場所に書き込みます。
- `--benchmark-no-force`：デフォルトの `--force` ポートのクリーンアップをスキップし、Gateway ポートがすでに使用中の場合は即座に失敗します。

ベンチマークモードでは、同期 I/O トレースの大量出力がデフォルトで抑制されます。CPU プロファイルと同期 I/O スタックトレースの両方を取得するには、`--benchmark` とともに `OPENCLAW_TRACE_SYNC_IO=1` を設定します。ベンチマークモードでは、それらのトレースブロックはベンチマークディレクトリ配下の `gateway-watch-output.log` に出力され（ターミナルペインでは除外されます）、通常の Gateway ログは引き続き表示されます。

tmux ラッパーは、`OPENCLAW_PROFILE`、`OPENCLAW_CONFIG_PATH`、`OPENCLAW_STATE_DIR`、`OPENCLAW_GATEWAY_PORT`、`OPENCLAW_SKIP_CHANNELS` など、一般的な非機密のランタイムセレクターをペインに引き継ぎます。プロバイダーの認証情報は通常のプロファイル／設定に保存するか、一時的なシークレットを一度だけ使用する場合は未加工のフォアグラウンドモードを使用してください。

監視対象 Gateway が起動中に終了した場合、ウォッチャーは `openclaw doctor --fix --non-interactive` を一度実行し、Gateway 子プロセスを再起動します。開発専用の修復処理を行わずに元の起動エラーを確認するには、`OPENCLAW_GATEWAY_WATCH_AUTO_DOCTOR=0` を設定します。

管理対象の tmux ペインでは、Gateway ログはデフォルトで色付きになります。ANSI 出力を無効にするには、`pnpm gateway:watch` の起動時に `FORCE_COLOR=0` を設定します。

ウォッチャーは、`src/` 配下のビルド関連ファイル、拡張機能のソースファイル、拡張機能の `package.json` および `openclaw.plugin.json` メタデータ、`tsconfig.json`、`package.json`、`tsdown.config.ts` の変更時に再起動します。拡張機能のメタデータ変更では、強制的に再ビルドせずに Gateway を再起動します。ソースおよび設定の変更時には、引き続き先に `dist` を再ビルドします。

`gateway:watch` の後に Gateway CLI フラグを追加すると、再起動のたびにそのまま渡されます。同じ監視コマンドを再実行すると、指定された tmux ペインが再生成されます。未加工のウォッチャーは単一ウォッチャーロックを維持するため、重複した親ウォッチャーが積み重ならず、置き換えられます。

## 開発プロファイルと開発 Gateway（--dev）

**別々の** `--dev` フラグが 2 つあります。

- **グローバル `--dev`（プロファイル）：** 状態を `~/.openclaw-dev` 配下に分離し、Gateway ポートのデフォルトを `19001` に設定します（派生ポートも連動して移動します）。
- **`gateway --dev`：** 設定とワークスペースが存在しない場合に、デフォルトの設定とワークスペースを Gateway が自動作成するよう指定します（ブートストラップはスキップします）。

推奨フロー（開発プロファイルと開発ブートストラップ）：

```bash
pnpm gateway:dev
OPENCLAW_PROFILE=dev openclaw tui
```

グローバルインストールがない場合は、`pnpm openclaw ...` を介して CLI を実行します。

実行される処理：

1. **プロファイルの分離**（グローバル `--dev`）
   - `OPENCLAW_PROFILE=dev`
   - `OPENCLAW_STATE_DIR=~/.openclaw-dev`
   - `OPENCLAW_CONFIG_PATH=~/.openclaw-dev/openclaw.json`
   - `OPENCLAW_GATEWAY_PORT=19001`（ブラウザー／キャンバスポートもそれに応じて移動します）

2. **開発ブートストラップ**（`gateway --dev`）
   - 存在しない場合は最小限の設定を書き込みます（`gateway.mode=local`、local loopback にバインド）。
   - `agents.defaults.workspace` を開発ワークスペースと `agents.defaults.skipBootstrap=true` に設定します。
   - 存在しない場合は、ワークスペースファイル `AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md` を作成します。
   - デフォルトのアイデンティティ：**C3-PO**（プロトコル・ドロイド）。
   - `pnpm gateway:dev` は、チャンネルプロバイダーをスキップするために `OPENCLAW_SKIP_CHANNELS=1` も設定します。

開発用 Gateway は、デフォルトで環境からのチャンネルトリガーを無視するため、シェルから継承された認証情報によって開発インスタンスが実際のチャンネルサービスに接続されることはありません。明示的な `channels.<id>` 設定は引き続き機能します。その実行で環境からのチャンネル自動設定を復元するには、`--dev` とともに `--dev-ambient-channels` を渡します。

リセットフロー（新規開始）：

```bash
pnpm gateway:dev:reset
```

<Note>
`--dev` は**グローバル**プロファイルフラグであり、一部のランナーによって消費されます。明示的に指定する必要がある場合は、環境変数形式を使用してください。

```bash
OPENCLAW_PROFILE=dev openclaw gateway --dev --reset
```

</Note>

`--reset` は、設定、認証情報、セッション、開発ワークスペースを消去し（削除ではなくゴミ箱へ移動）、デフォルトの開発セットアップを再作成します。

<Tip>
開発用ではない Gateway がすでに実行中（launchd または systemd）の場合は、先に停止してください。

```bash
openclaw gateway stop
```

</Tip>

## 未加工ストリームのログ記録

OpenClaw は、フィルタリング／フォーマット処理の前に、**未加工のアシスタントストリーム**をログに記録できます。これは、推論がプレーンテキストの差分（または個別の思考ブロック）として到着しているかどうかを確認する最適な方法です。

CLI で有効にします。

```bash
pnpm gateway:watch --raw-stream
```

任意のパスを指定するには：

```bash
pnpm gateway:watch --raw-stream --raw-stream-path ~/.openclaw/logs/raw-stream.jsonl
```

同等の環境変数：

```bash
OPENCLAW_RAW_STREAM=1
OPENCLAW_RAW_STREAM_PATH=~/.openclaw/logs/raw-stream.jsonl
```

デフォルトファイル：`~/.openclaw/logs/raw-stream.jsonl`

## 安全上の注意

- 未加工ストリームのログには、完全なプロンプト、ツール出力、ユーザーデータが含まれる可能性があります。
- ログはローカルに保持し、デバッグ後に削除してください。
- ログを共有する場合は、最初にシークレットと個人を特定できる情報（PII）を除去してください。

## VSCode でのデバッグ

ビルドによって生成ファイル名がハッシュ化されるため、ソースマップが必要です。同梱の `launch.json` は Gateway サービスを対象としています。

1. **Rebuild and Debug Gateway** - Gateway を起動する前に `/dist` を削除し、デバッグを有効にして再ビルドします。
2. **Debug Gateway** - `/dist` に変更を加えず、既存のビルドをデバッグします。

### セットアップ

1. **Run and Debug**（Activity Bar、または `Ctrl`+`Shift`+`D`）を開きます。
2. **Rebuild and Debug Gateway** を選択し、**Start Debugging** を押します。

代わりにビルド／デバッグサイクルを手動で管理するには：

1. ターミナルでソースマップを有効にします。
   - **Linux/macOS**：`export OUTPUT_SOURCE_MAPS=1`
   - **Windows (PowerShell)**：`$env:OUTPUT_SOURCE_MAPS="1"`
   - **Windows (CMD)**：`set OUTPUT_SOURCE_MAPS=1`
2. 再ビルド：`pnpm clean:dist && pnpm build`
3. **Debug Gateway** を選択し、**Start Debugging** を押します。

`src/` の TypeScript ファイルにブレークポイントを設定します。デバッガーはソースマップを介して、コンパイル済み JavaScript にマッピングします。

### 注記

- **Rebuild and Debug Gateway** は、起動するたびに `/dist` を削除し、ソースマップを有効にして完全な `pnpm build` を実行します。
- **Debug Gateway** は `/dist` に影響を与えずに開始／停止できますが、ビルドサイクルは別のターミナルで管理します。
- 他の CLI サブコマンドをデバッグするには、`launch.json` の `args` を編集します。
- ビルド済み CLI を他のタスクで使用する場合（たとえば、デバッグセッションによって新しい認証トークンが生成された場合の `dashboard --no-open`）、別のターミナルから `node ./openclaw.mjs`、または `alias openclaw-build="node $(pwd)/openclaw.mjs"` のようなエイリアスを実行します。

## 関連項目

- [トラブルシューティング](/ja-JP/help/troubleshooting)
- [よくある質問](/ja-JP/help/faq)
