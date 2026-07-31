---
read_when:
    - セットアップや修復について OpenClaw とチャットしたい場合
    - オンボーディングウィザードを使用して初回セットアップを行っています
    - デフォルトのワークスペースパスを設定したい場合
    - スクリプトにはベースラインのみのセットアップフラグが必要です
summary: '`openclaw setup` の CLI リファレンス（オンボーディングフォールバックを備えたシステムエージェントチャット）'
title: セットアップ
x-i18n:
    generated_at: "2026-07-26T09:16:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b3b4f70f2631683fcb03007a80fe43a06387be3d7e4d533381e5e536333af051
    source_path: cli/setup.md
    workflow: 16
---

# `openclaw setup`

`openclaw setup` はシステムエージェントのエントリーポイントです。構成済みのシステムでは、単独の
`openclaw setup` により対話型の OpenClaw チャットが開きます。新規システムでは、
ガイド付きオンボーディングに移行します。1 回のリクエストには `-m`/`--message` を使用し、
ウィザードを使わずに構成/ワークスペースフォルダーを初期化するには `--baseline` を使用します。

ルーティング順序:

1. オンボーディングオプション（`--wizard`、`--baseline`、ワークスペース、リセット、
   非対話型、フロー、モード、Gateway、デーモン、スキップ、インポート、リモート、または認証
   オプション）を指定すると、`openclaw onboard` とまったく同じようにオンボーディングが実行されます。
2. `-m`/`--message` または `--yes` を指定すると、システムエージェントが実行されます。
3. ルーティングオプションがない場合、構成済みの対話型システムでは OpenClaw が開きます。
   新規システムではオンボーディングが実行されます。構成済みのシステムでは、TTY がなくても `--json` により
   システム概要が出力されます。オンボーディングオプションを指定した場合は、オンボーディングの
   JSON サマリーが維持されます。

ガイド付きモードでは、`--workspace <dir>` が OpenClaw に提案されるワークスペースです。
この提案を承認した後にのみ保存されます。新規インストールでは、ベースライン、クラシック、および
非対話型セットアップにより、指定されたワークスペースが各通常フローを通じて保存されます。
既存のエージェント一覧が再マッピングされる場合、クラシックウィザードでは明示的な確認が必要です。
非対話型セットアップでは現在のフリートワークスペースが維持され、警告が出力されます。

ガイド付き推論検出は、macOS または Linux 上の Gateway ホストで実行されます。CLI
と macOS アプリは、構成済みモデル、対応している CLI ログイン、API キー環境変数、および
インストール済みの Ollama または LM Studio モデルを確認する、同じ Gateway 所有の検出機能を呼び出します。
この自動処理によってローカルモデルがダウンロードされることはありません。検出されたローカルランタイムは、
CLI および API キーの候補の後に自動テストされます。複数のローカルモデルを利用できる場合、OpenClaw は
ツール呼び出しに対応する最も強力な instruct ファミリーを優先します。選択された候補は、そのプロバイダーと
モデルの構成が保存される前に、実際の補完に応答できなければなりません。
インストール済みの Gemini、Antigravity、Pi、および OpenCode CLI は、
ガイド付きセットアップで再利用可能な推論経路として使用できない場合にも報告されます。

`setup` は、認証（`--auth-choice`、`--token`、プロバイダーキーフラグ）、Gateway
（`--gateway-port`、`--gateway-bind`、`--gateway-auth`、`--install-daemon`）、
Tailscale（`--tailscale`）、リセット（`--reset`、`--reset-scope`）、フロー
（`--flow quickstart|advanced|manual|import`）、およびスキップフラグ
（`--skip-channels`、`--skip-skills`、`--skip-bootstrap`、`--skip-search`、
`--skip-health`、`--skip-ui`、`--skip-hooks`）を含む、`openclaw onboard` と同じオンボーディングフラグを受け付けます。`openclaw onboard --tui` と同じ
ターミナル経路を使用するには、`--tui` を渡します。フラグの完全なリファレンスと
非対話型の例については、[オンボーディング](/ja-JP/cli/onboard)および
[CLI 自動化](/ja-JP/start/wizard-cli-automation)を参照してください。`openclaw onboard --modern` は、同じ推論ゲート付き
OpenClaw アシスタントへの互換性エントリとして引き続き利用できます。

<Note>
`openclaw setup` は変更可能な構成のインストール向けです。Nix モード（`OPENCLAW_NIX_MODE=1`）では、構成ファイルが Nix によって管理されているため、OpenClaw はセットアップによる書き込みを拒否します。公式の [nix-openclaw クイックスタート](https://github.com/openclaw/nix-openclaw#quick-start)または別の Nix パッケージ用の同等のソース構成を使用してください。
</Note>

## オプション

| フラグ                       | 説明                                                                                          |
| -------------------------- | ---------------------------------------------------------------------------------------------------- |
| `-m, --message <text>`     | OpenClaw リクエストを 1 回実行します。                                                                            |
| `--yes`                    | 1 回の `--message` リクエストについて、永続的な構成の書き込みを承認します。                                        |
| `--workspace <dir>`        | ワークスペースの提案です。既存のフリートではクラシック確認が必要であり、非対話型では維持されます。 |
| `--baseline`               | オンボーディングを行わずに、ベースライン構成/ワークスペース/セッションフォルダーを作成します。                                 |
| `--wizard`                 | 対話型オンボーディングを強制します。                                                                        |
| `--tui`                    | ブラウザーへの引き継ぎの代わりにターミナル経路を使用します。                                               |
| `--non-interactive`        | プロンプトなしでオンボーディングを実行します。                                                                      |
| `--accept-risk`            | システム全体へのエージェントアクセスのリスクを承認します。`--non-interactive` とともに指定する必要があります。                        |
| `--mode <mode>`            | オンボーディングモード: `local` または `remote`。                                                                |
| `--flow <flow>`            | オンボーディングフロー: `quickstart`、`advanced`、`manual`、または `import`。                                       |
| `--reset`                  | オンボーディング前に構成、認証情報、およびセッションをリセットします（ワークスペースは `--reset-scope full` の場合のみ）。  |
| `--reset-scope <scope>`    | リセット範囲: `config`、`config+creds+sessions`、または `full`。                                           |
| `--import-from <provider>` | オンボーディング中に実行する移行プロバイダー。                                                         |
| `--import-source <path>`   | `--import-from` のソースエージェントホーム。                                                               |
| `--import-secrets`         | オンボーディングの移行中に、対応しているシークレットをインポートします。                                                |
| `--remote-url <url>`       | リモート Gateway の WebSocket URL。                                                                        |
| `--remote-token <token>`   | リモート Gateway トークン（任意）。                                                                     |
| `--json`                   | 構成済みシステム: OpenClaw の概要。オンボーディング経路: オンボーディングのサマリー。                          |

`--classic` と `--non-interactive` は相互に排他的です。クラシックでは
プロンプト付きウィザードが開き、非対話型セットアップでは自動化経路が使用されます。
対話型オンボーディングでは、`--remote-url` と `--remote-token` により
リモート Gateway のステップが事前入力され、その実行では保存済みのリモート値より優先されます。
トークンも渡さない限り、URL を変更しても保存済みの認証情報は再利用されません。
トークンはマスクされたままになり、ウィザードで選択した平文または SecretRef の
保存モードが使用されます。

### ベースラインモード

`openclaw setup --baseline` は従来のベースライン専用の動作を維持します。
構成、ワークスペース、およびセッションのディレクトリを作成し、オンボーディングを
実行せずに終了します。`--workspace` と無害な出力制御は受け付けますが、
明示的なオンボーディング、Gateway、認証、リセット、またはデーモンのオプションは
暗黙に無視せず拒否します。既存の構成が無効な場合、ベースラインセットアップは
その構成を維持し、再試行する前に `openclaw doctor` を実行するよう求めます。

## 例

```bash
openclaw setup
openclaw setup -m "status"
openclaw setup -m "restart gateway" --yes
openclaw setup --json
openclaw setup --wizard
openclaw setup --baseline
openclaw setup --workspace ~/.openclaw/workspace
openclaw setup --import-from hermes --import-source ~/.hermes
openclaw setup --non-interactive --accept-risk --mode remote --remote-url wss://gateway-host:18789 --remote-token <token>
```

## 注記

- ベースラインセットアップ後、完全なガイド付き手順には `openclaw onboard`、対象を絞った変更には `openclaw configure`、チャネルアカウントの追加には `openclaw channels add` を実行します。
- Hermes の状態が検出された場合、対話型オンボーディングで移行が自動的に提案されることがあります。インポートオンボーディングには新規セットアップが必要です。オンボーディング外でのドライラン計画、バックアップ、および上書きモードについては、[移行](/ja-JP/cli/migrate)を使用してください。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [オンボーディング](/ja-JP/cli/onboard)
- [オンボーディング（CLI）](/ja-JP/start/wizard)
- [はじめに](/ja-JP/start/getting-started)
- [インストールの概要](/ja-JP/install)
