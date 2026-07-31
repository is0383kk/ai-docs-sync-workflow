---
read_when:
    - デスクトップまたはサーバーアプリケーションへの OpenClaw の組み込み
    - Gateway を子プロセスとして監視する
    - ログをスクレイピングせずに Gateway の準備完了、再起動、シャットダウン、または無効な設定を処理する
summary: Electron または別のホストアプリから OpenClaw Gateway を子プロセスとして監視する
title: OpenClaw の組み込み
x-i18n:
    generated_at: "2026-07-26T09:03:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ca67e03994f21446bfeca58c95c2cb624dde767b9983a89982627145f80dfb90
    source_path: gateway/embedding.md
    workflow: 16
---

埋め込みホストは、インストール済みの `openclaw` 実行ファイルを監督し、Gateway WebSocket プロトコルを制御プレーンとして使用し、子プロセスを交換可能なランタイムとして扱う必要があります。これにより、OpenClaw の非公開の状態レイアウトに依存せず、プロセスの所有権、準備完了状態、障害復旧、アップグレードを明示的に管理できます。

クライアント認証と再接続状態については、
[Gateway クライアントの構築](https://docs.openclaw.ai/gateway/clients)を参照してください。

## 埋め込みプリセットで子プロセスを起動する

実際の `node_modules` インストールを使用し、パッケージの実行ファイルを起動します。検出、再起動、チャネルのライフサイクルを所有するホストには、次の構成が有用なベースラインとなります。

```ts
import { spawn } from "node:child_process";
import { dirname, resolve } from "node:path";
import { fileURLToPath } from "node:url";

// ホストアプリケーションが管理する実際の Node ランタイムへの絶対パスを指定します。
declare const hostNodeExecutable: string;

const packageEntry = fileURLToPath(import.meta.resolve("openclaw"));
const openclawEntry = resolve(dirname(packageEntry), "..", "openclaw.mjs");
const gateway = spawn(hostNodeExecutable, [openclawEntry, "gateway", "--allow-unconfigured"], {
  env: {
    ...process.env,
    OPENCLAW_DISABLE_BONJOUR: "1",
    OPENCLAW_EXEC_SHELL_SNAPSHOT: "0",
    OPENCLAW_NO_RESPAWN: "1",
    OPENCLAW_SKIP_CHANNELS: "1",
  },
  stdio: ["ignore", "inherit", "inherit"],
});
```

図のように、インストール済みパッケージを通じて OpenClaw を解決してください。プロジェクトローカルの `openclaw` バイナリがホストプロセスの `PATH` に存在すると想定しないでください。この例では出力を継承するため、stdout または stderr のパイプが満杯になって子プロセスがブロックすることはありません。代わりにホストがこれらのストリームをキャプチャする場合は、起動直後にコンシューマーを接続してください。

| 設定                             | 埋め込みへの効果                                                                                                                                                                                  |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_DISABLE_BONJOUR=1`     | ホストが検出を所有する場合に、Gateway が所有する LAN マルチキャスト広告を無効にします。                                                                                                           |
| `OPENCLAW_NO_RESPAWN=1`          | 管理されていない埋め込み子プロセスで、OpenClaw が更新後の再起動をデタッチされた子プロセスへ引き渡すことを防ぎます。通常の再起動はプロセス内に留まるため、ホストは追跡対象 PID の所有権を維持します。 |
| `OPENCLAW_EXEC_SHELL_SNAPSHOT=0` | ホストの exec コマンドに対するログインシェルのスナップショット取得を無効にします。                                                                                                                |
| `OPENCLAW_SKIP_CHANNELS=1`       | チャネルの起動と再読み込みをスキップします。埋め込みアプリが制御プレーン専用または WebChat 専用の Gateway を必要とする場合にのみ設定してください。                                                |

`--allow-unconfigured` がバイパスするのは `gateway.mode=local` 起動ガードだけです。設定の書き込みや無効なファイルの修復は行いません。埋め込みアプリがオンボーディング、設定 CLI、または Gateway RPC を通じて通常のローカル設定をプロビジョニングする場合は、省略してください。

### Electron のシェルスナップショットに関する警告

シェルスナップショットの取得では、ログインシェルから `process.execPath -e <script>` を実行します。通常の Node プロセスでは、`process.execPath` は Node 実行ファイルです。Electron 環境では Electron バイナリとなり、この呼び出しをアプリケーションの起動として解釈して「Unable to find Electron app」というポップアップを表示する場合があります。`OPENCLAW_EXEC_SHELL_SNAPSHOT=0` はレンダラープロセスだけでなく、Gateway 子プロセスの環境にも設定してください。同じ理由から、`hostNodeExecutable` は Electron の `process.execPath` ではなく、実際の Node ランタイムを指す必要があります。

## 終了コードで無効な設定を処理する

Gateway の起動では、無効な設定を含む設定関連の起動エラーに終了コード `78`（`EX_CONFIG`）を使用します。人間向けの stderr を解析するのではなく、終了コードで分岐してください。

1. Gateway 子プロセスと同じ設定および状態環境に対して `openclaw doctor --fix --yes --non-interactive` を実行します。
2. doctor が正常終了した後、Gateway の起動を 1 回再試行します。
3. 子プロセスが再び `78` で終了した場合は、修復ループを停止し、設定エラーをユーザーに提示します。

診断用に stderr は保持しますが、その文言に基づいてライフサイクルを判断しないでください。

正常に起動した後は、実行中に無効な設定変更を行っても影響は比較的小さくなります。設定ウォッチャーは再読み込みをスキップしたことをログに記録し、最後に受け入れたメモリ内設定でサービスを継続します。ファイルを修復すると、ウォッチャーが次の有効なスナップショットを受け入れます。

## プロトコルの準備完了を待つ

ログの部分文字列ではなく、WebSocket シグナルを使用します。

1. Gateway WebSocket を開きます。
2. `connect.challenge` イベントを待ちます。これは、リスナーが WebSocket を受け入れ、チャレンジハンドシェイクを開始できることを示します。
3. チャレンジに紐づけられたデバイス署名を付けて `connect` を送信します。
4. 認証済み RPC におけるアプリケーションの準備完了状態として `hello-ok` を扱います。

チャレンジは、完全な初期化よりも意図的に早い段階で送信されます。起動時のサイドカーがまだ処理中の場合、`connect` は `details.reason: "startup-sidecars"` と、上限のある `retryAfterMs` を含む再試行可能な `UNAVAILABLE` エラーを返し、その後コード `1013`、理由 `gateway starting` で接続を閉じます。`@openclaw/gateway-protocol/startup-unavailable` の `resolveGatewayStartupRetryAfterMs` またはリファレンスクライアントに組み込まれたポリシーを使用してから、再接続してください。

## 再起動とシャットダウンを解釈する

正常に接続を閉じる前に、Gateway は `reason` と `restartExpectedMs` を含む `shutdown` イベントをブロードキャストします。null ではない `restartExpectedMs` は、プロセス内または監督下での再起動が予定されていることを意味します。`null` は最終的なシャットダウンを意味します。

その後の WebSocket 終了コードは、どちらの場合も `1012` です。通常のクライアント終了理由も、どちらの場合も `service restart` であるため、終了コードと理由のどちらでも再起動とシャットダウンを区別できません。先行する `shutdown` ペイロードを受信した場合は保持し、ホスト自身の停止意図および子プロセスの終了ステータスと組み合わせて判断してください。イベントなしで接続が消失した場合は、通常の上限付き再接続および子プロセス監督ポリシーを使用してください。

## 状態ファイルではなく RPC を使用する

OpenClaw の状態は Gateway だけが所有するようにしてください。一般的な埋め込み操作には、すでに RPC メソッドが用意されています。

| タスク                           | RPC メソッド                                           |
| ------------------------------- | ------------------------------------------------------ |
| セッションのカタログとライフサイクル | `sessions.list`, `sessions.patch`, `sessions.delete` |
| トランスクリプトの表示           | `chat.history`                                       |
| コストと使用量のレポート         | `usage.cost`, `sessions.usage`                       |
| モデル認証情報の状態             | `models.authStatus`                                  |
| 設定                             | `config.get`, `config.patch`                         |

`config.get` は、スナップショットを返す前に機密値と SecretRef 識別子を秘匿化します。書き込みメソッドも秘匿化された設定を返します。クライアントは秘匿化センチネルを不透明な値として扱い、文書化された設定書き込み契約を使用する必要があります。Gateway が平文のシークレットを返すことを決して期待してはいけません。

アプリ機能を実装するために、`~/.openclaw` 配下のファイル、SQLite テーブル、トランスクリプトファイル、キャッシュディレクトリを読み取ったり変更したりしないでください。これらのレイアウトは非公開のランタイム実装詳細であり、プロトコル互換性を維持したまま移動または変更される可能性があります。

## フラット化せずにインストールする

ルートの `openclaw` パッケージは、単一ファイルとしてベンダー化するためのものではありません。`dist/extensions` 配下のバンドル済みランタイムファイルには、`openclaw/plugin-sdk/*` のようなベアな自己インポートが残っている一方、npm パッケージは拡張機能ごとの `node_modules` ツリーを意図的に除外しています。

Node がパッケージの exports とルート依存関係ツリーを解決できるように、npm、pnpm、またはその他の通常の Node パッケージインストール方法で OpenClaw をインストールしてください。インストール済みの `openclaw` 実行ファイルを起動します。`dist` だけをコピーしたり、パッケージをアプリバンドルへフラット化したり、選択した拡張機能ファイルをベンダー化したりしないでください。

## 関連項目

- [Gateway クライアントの構築](https://docs.openclaw.ai/gateway/clients)
- [Gateway プロトコル](https://docs.openclaw.ai/gateway/protocol)
- [Gateway CLI](https://docs.openclaw.ai/cli/gateway)
- [外部アプリ向け Gateway 統合](https://docs.openclaw.ai/gateway/external-apps)
