---
read_when:
    - macOS 開発環境のセットアップ
summary: OpenClaw macOS アプリに取り組む開発者向けセットアップガイド
title: macOS 開発環境のセットアップ
x-i18n:
    generated_at: "2026-07-26T10:20:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ff72bb449e70b94b8a13504414955ab7fe411a674b65e670939484a5863b5f48
    source_path: platforms/mac/dev-setup.md
    workflow: 16
---

# macOS 開発環境のセットアップ

OpenClaw macOS アプリケーションをソースからビルドして実行します。

## 前提条件

- **Xcode 26.2+**（Swift 6.2 ツールチェーン）。ソフトウェアアップデートで入手できる最新の macOS 上で使用してください。
- Gateway、CLI、パッケージ化スクリプトには **Node.js 24.15+ および pnpm** が必要です。Node 22.22.3+ も動作します。

## 1. 依存関係をインストールする

```bash
pnpm install
```

## 2. アプリをビルドしてパッケージ化する

```bash
./scripts/package-mac-app.sh
```

出力先は `dist/OpenClaw.app` です。Apple Developer ID 証明書がない場合、スクリプトはアドホック署名にフォールバックします。

開発用の実行モード、署名フラグ、Team ID のトラブルシューティングについては、[apps/macos/README.md](https://github.com/openclaw/openclaw/blob/main/apps/macos/README.md)を参照してください。
リポジトリのルートから高速な開発ループを実行するには `scripts/restart-mac.sh` を使用します（アドホック署名には `--no-sign` を追加してください。`--no-sign` では TCC 権限が維持されません）。

<Note>
アドホック署名されたアプリでは、セキュリティプロンプトが表示される場合があります。アプリが「Abort trap 6」と表示して即座にクラッシュする場合は、[トラブルシューティング](#troubleshooting)を参照してください。
</Note>

## 3. CLI と Gateway をインストールする

パッケージ化されたアプリには、標準の `scripts/install-cli.sh` インストーラーが組み込まれています。新しいプロファイルでは、オンボーディング中に **This Mac** を選択してください。アプリは Gateway ウィザードを開始する前に、対応するユーザー空間の CLI とランタイムをインストールします。

開発環境を手動で復旧する場合は、対応する CLI を自身でインストールします。

```bash
npm install -g openclaw@<version>
```

`pnpm add -g openclaw@<version>` と `bun add -g openclaw@<version>` も動作します。Gateway 自体には引き続き Node が推奨ランタイムです。

## トラブルシューティング

### ビルドが失敗する：ツールチェーンまたは SDK の不一致

macOS アプリのビルドには、最新の macOS SDK と Swift 6.2 ツールチェーン（Xcode 26.2+）が必要です。

```bash
xcodebuild -version
xcrun swift --version
```

バージョンが一致しない場合は、macOS と Xcode を更新してからビルドを再実行してください。

### 権限付与時にアプリがクラッシュする

**Speech Recognition** または **Microphone** へのアクセスを許可しようとした際にアプリがクラッシュする場合、TCC キャッシュの破損または署名の不一致が原因である可能性があります。

1. デバッグ用バンドル ID の TCC 権限をリセットします。

   ```bash
   tccutil reset All ai.openclaw.mac.debug
   ```

2. 失敗する場合は、macOS で状態を完全にリセットするため、[`scripts/package-mac-app.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-app.sh)内の `BUNDLE_ID` を一時的に変更します。

### Gateway が「Starting...」のまま進まない

ゾンビプロセスがポートを占有していないか確認します。

```bash
openclaw gateway status
openclaw gateway stop

# LaunchAgent を使用していない場合（開発モード／手動実行）、リスナーを探します。
lsof -nP -iTCP:18789 -sTCP:LISTEN
```

手動実行したプロセスがポートを占有している場合は停止（Ctrl+C）するか、最後の手段として上記で検出した PID を強制終了します。

## 関連項目

- [macOS アプリ](/ja-JP/platforms/macos)
- [インストールの概要](/ja-JP/install)
