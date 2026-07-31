---
read_when:
    - macOS ログの取得または個人データのログ記録の調査
    - 音声ウェイク／セッションのライフサイクル問題のデバッグ
summary: OpenClaw のログ：ローテーション式診断ファイルログ + 統合ログのプライバシーフラグ
title: macOS のログ記録
x-i18n:
    generated_at: "2026-07-26T09:30:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ef0fd91bd7fc0a8b5f598cfe8f5de551795a4badd0f6634c5bcbd4f3916bfc64
    source_path: platforms/mac/logging.md
    workflow: 16
---

# ログ記録（macOS）

## ローテーション診断ファイルログ（Debug ペイン）

macOS アプリは swift-log を通じてログを記録し（デフォルトでは統合ログ）、永続的な記録用としてローテーションするローカルファイルログにも書き込めます（`DiagnosticsFileLog`）。

- 有効化：**Debug pane -> Logs -> App logging -> "Write rolling diagnostics log (JSONL)"**（デフォルトではオフ）。
- 詳細度：**Debug pane -> Logs -> App logging -> Verbosity** ピッカー。
- 場所：`~/Library/Logs/OpenClaw/diagnostics.jsonl`。
- ローテーション：5 MB でローテーションし、接尾辞 `.1`...`.5` が付いたバックアップを最大 5 個保持します（最も古いものは削除されます）。
- 消去：**Debug pane -> Logs -> App logging -> "Clear"** を実行すると、アクティブなファイルとすべてのバックアップが削除されます。

このファイルは機密情報として扱い、内容を確認せずに共有しないでください。

## macOS の統合ログにおける非公開データ

統合ログでは、サブシステムが `privacy -off` を有効にしない限り、ほとんどのペイロードが編集されます。これは `/Library/Preferences/Logging/Subsystems/` 内の plist によって制御され、サブシステム名をキーとして設定します。このフラグは新しいログエントリにのみ適用されるため、問題を再現する前に有効化してください。背景情報：[macOS のログプライバシーに関する不可解な挙動](https://steipete.me/posts/2025/logging-privacy-shenanigans)。

## OpenClaw で有効化する（`ai.openclaw`）

まず plist を一時ファイルに書き込み、その後 root としてアトミックにインストールします。

```bash
cat <<'EOF' >/tmp/ai.openclaw.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>DEFAULT-OPTIONS</key>
    <dict>
        <key>Enable-Private-Data</key>
        <true/>
    </dict>
</dict>
</plist>
EOF
sudo install -m 644 -o root -g wheel /tmp/ai.openclaw.plist /Library/Preferences/Logging/Subsystems/ai.openclaw.plist
```

再起動は不要です。logd はすぐにファイルを読み込みますが、非公開ペイロードが含まれるのは新しいログ行のみです。より詳細な出力は `./scripts/clawlog.sh --category WebChat --last 5m` で表示できます（`--last`/`-l` は時間範囲を設定し、デフォルトは `5m`、`--category`/`-c` はカテゴリでフィルタリングします）。

## デバッグ後に無効化する

- オーバーライドを削除します：`sudo rm /Library/Preferences/Logging/Subsystems/ai.openclaw.plist`。
- 必要に応じて `sudo log config --reload` を実行し、logd にオーバーライドを直ちに破棄させます。
- このログには電話番号やメッセージ本文が含まれる可能性があります。plist は実際に必要な間だけ配置してください。

## 関連項目

- [macOS アプリ](/ja-JP/platforms/macos)
- [Gateway のログ記録](/ja-JP/gateway/logging)
