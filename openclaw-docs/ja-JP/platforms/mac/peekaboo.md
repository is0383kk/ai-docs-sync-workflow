---
read_when:
    - OpenClaw.app で PeekabooBridge をホストする
    - Swift Package Manager を介した Peekaboo の統合
    - PeekabooBridge のプロトコル／パスの変更
    - PeekabooBridge、Codex Computer Use、cua-driver MCP の選択方法
summary: macOS UI オートメーション向け PeekabooBridge 統合
title: Peekaboo ブリッジ
x-i18n:
    generated_at: "2026-07-26T09:49:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 24d4187b2f5c5f11f44a24e25b350adaa3b068f24dce640ec695d52eb61f8e9a
    source_path: platforms/mac/peekaboo.md
    workflow: 16
---

OpenClaw は、ローカルの権限対応 UI 自動化ブローカー（`PeekabooBridgeHostCoordinator`、`steipete/Peekaboo` Swift パッケージを基盤とする）として **PeekabooBridge** をホストできます。これにより、`peekaboo` CLI は macOS アプリの TCC 権限を再利用しながら UI 自動化を実行できます。

## これは何か（そして何ではないか）

- **ホスト**: OpenClaw.app は PeekabooBridge ホストとして動作できます。
- **クライアント**: `peekaboo` CLI（独立した `openclaw ui ...` サーフェスはありません）。
- **UI**: ビジュアルオーバーレイは Peekaboo.app に留まり、OpenClaw は軽量なブローカーホストです。

## 他のデスクトップ制御経路との関係

OpenClaw には、意図的に分離された 4 つのデスクトップ制御経路があります。

- **PeekabooBridge ホスト**: OpenClaw.app はローカルの PeekabooBridge ソケットをホストします。`peekaboo` CLI がクライアントとなり、スクリーンショット、クリック、メニュー、ダイアログ、Dock 操作、ウィンドウ管理に OpenClaw.app の macOS 権限を使用します。
- **エージェント駆動のコンピューター操作（`computer.act`）**: Gateway エージェントの組み込み `computer` ツールは、`screen.snapshot` を介してスクリーンショットを取得し、危険な `computer.act` Node コマンドを通じてポインターとキーボードを操作します。macOS Node は、PeekabooBridge ソケットや `peekaboo` CLI を経由せず、このブリッジが公開する組み込みの Peekaboo 自動化サービスと限定的な CoreGraphics プリミティブを使用して、`computer.act` をプロセス内で実行します。[コンピューター操作](/ja-JP/nodes/computer-use)を参照してください。
- **Codex Computer Use**: バンドルされた `codex` Plugin は、Codex の `computer-use` MCP Plugin（`extensions/codex/src/app-server/computer-use.ts`）を確認し、インストールできます。その後、Codex モードのターン中は、ネイティブのデスクトップ制御ツール呼び出しを Codex が担います。OpenClaw はこれらの操作を PeekabooBridge 経由でプロキシしません。
- **直接 `cua-driver` MCP**: OpenClaw は、TryCua のアップストリーム `cua-driver mcp` サーバーを通常の MCP サーバーとして登録できます。これにより、Codex マーケットプレイスや PeekabooBridge ソケットを経由せずに、CUA ドライバー独自のスキーマと pid／ウィンドウ／要素インデックスのワークフローをエージェントに提供します。

OpenClaw.app の権限対応ブリッジホストを介した広範な macOS 自動化には Peekaboo を使用します。任意のビジョンモデルが操作できる統一された `computer.act` Node コマンドを通じて、Gateway エージェントがデスクトップを表示および制御する必要がある場合は、エージェント駆動のコンピューター操作を使用します。Codex モードのエージェントが Codex のネイティブ Plugin を利用する必要がある場合は、Codex Computer Use を使用します。CUA ドライバーを通常の MCP サーバーとして任意の OpenClaw 管理ランタイムに公開するには、直接 `cua-driver mcp` を使用します。

## ブリッジを有効にする

macOS アプリで、**Settings -> Enable Peekaboo Bridge** を選択します。このトグルを有効にするには、**Allow Computer Control** がオンになっている必要があります。どちらもローカル UI 自動化を許可するためです。Computer Control がオフの場合、このトグルは無効になり、ホストは実行されません。Computer Control を使用せずに Peekaboo を操作するには、代わりに Peekaboo 独自の Mac アプリをホストとして実行します。

有効になっている場合（かつ Computer Control がオンの場合）、OpenClaw は `~/Library/Application Support/OpenClaw/<socket-name>` でローカル UNIX ソケットサーバーを起動します。無効にするとホストは停止し、`peekaboo` は利用可能な他のホストにフォールバックします。コーディネーターは、古い `peekaboo` インストール向けに、現在のソケットを指す従来のソケットシンボリックリンク（Application Support 内の `clawdbot`、`clawdis`、`moltbot`）も維持します。

## クライアントの検出順序

Peekaboo クライアントは通常、次の順序でホストを試行します。

1. Peekaboo.app（完全な UX）
2. Claude.app（インストールされている場合）
3. OpenClaw.app（軽量ブローカー）

どのホストがアクティブで、どのソケットパスが使用されているかを確認するには、`peekaboo bridge status --verbose` を使用します。次の設定で上書きできます。

```bash
export PEEKABOO_BRIDGE_SOCKET=/path/to/bridge.sock
```

## セキュリティと権限

- ブリッジは**呼び出し元のコード署名**を検証します。TeamID の許可リストが適用されます（Peekaboo ホストの TeamID と、実行中のアプリ自身の TeamID）。
- アクセシビリティには、汎用の `node` ランタイムよりも、署名済みのブリッジ／アプリのアイデンティティを優先してください。`node` にアクセシビリティを許可すると、その Node 実行ファイルによって起動されたすべてのパッケージが GUI 自動化アクセスを継承できます。[macOS の権限](/ja-JP/platforms/mac/permissions#accessibility-grants-for-node-and-cli-runtimes)を参照してください。
- リクエストは 10 秒後にタイムアウトします（`requestTimeoutSec: 10`）。
- 必要な権限がない場合、ブリッジは System Settings を起動する代わりに、明確なエラーメッセージを返します。

## スナップショットの動作（自動化）

スナップショットは、有効期間 10 分、上限 50 件（`InMemorySnapshotManager`）でメモリに保存されます。クリーンアップ時に成果物は削除されません。より長く保持する必要がある場合は、クライアントから再取得してください。

## トラブルシューティング

- `peekaboo` が「bridge client is not authorized」と報告する場合は、クライアントが適切に署名されていることを確認するか、**debug** モードでのみ `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` を指定してホストを実行してください。
- ホストが見つからない場合は、ホストアプリ（Peekaboo.app または OpenClaw.app）のいずれかを開き、権限が付与されていることを確認してください。

## 関連項目

- [macOS アプリ](/ja-JP/platforms/macos)
- [macOS の権限](/ja-JP/platforms/mac/permissions)
