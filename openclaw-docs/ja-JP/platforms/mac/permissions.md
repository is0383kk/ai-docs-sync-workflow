---
read_when:
    - 表示されない、または応答しない macOS の権限プロンプトのデバッグ
    - Node または CLI ランタイムにアクセシビリティ権限を付与するかどうかを判断する
    - macOS アプリのパッケージ化または署名
    - バンドル ID またはアプリのインストールパスの変更
summary: macOS の権限永続化（TCC）と署名要件
title: macOSの権限
x-i18n:
    generated_at: "2026-07-26T10:08:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e561aa641e44fc1e1b95a3db244f31124e4e51d13ae709bee188d86054301e34
    source_path: platforms/mac/permissions.md
    workflow: 16
---

macOS の権限付与は不安定です。TCC は権限付与を、アプリのコード署名、バンドル識別子、ディスク上のパスに関連付けます。これらのいずれかが変更されると、macOS はアプリを新しいものとして扱い、プロンプトを表示しなくなったり、非表示にしたりすることがあります。

## 権限を安定させるための要件

- 同じパス：アプリを固定された場所から実行します（OpenClaw の場合は `dist/OpenClaw.app`）。
- 同じバンドル識別子：OpenClaw のバンドル ID は `ai.openclaw.mac` です。これを変更すると、新しい権限 ID が作成されます。
- 署名済みアプリ：未署名またはアドホック署名されたビルドでは、権限が保持されません。
- 一貫した署名：再ビルド後も署名を安定させるため、正規の Apple Development または Developer ID 証明書を使用します。

アドホック署名では、ビルドごとに新しい ID が生成されます。macOS は以前の権限付与を忘れ、古いエントリを消去するまでプロンプトがまったく表示されなくなることがあります。

## Node および CLI ランタイムへのアクセシビリティ権限の付与

汎用の `node` バイナリではなく、OpenClaw.app、Peekaboo.app、または独自のバンドル識別子を持つ別の署名済みヘルパーにアクセシビリティ権限を付与することを推奨します。

macOS の TCC は、認識したプロセスのコード ID にアクセシビリティ権限を付与します。Homebrew、nvm、pnpm、または npm のワークフローによって共有の `node` 実行可能ファイルにアクセシビリティ権限が付与された場合、同じ実行可能ファイルを介して起動されるすべての JavaScript パッケージが GUI 自動化権限を継承する可能性があります。

システム設定の `node` エントリは、1 つの npm パッケージに対する権限ではなく、その Node ランタイムに対する広範な権限として扱ってください。その Node インストールを介して起動されるすべてのスクリプトとパッケージを信頼できる場合を除き、`node` にアクセシビリティ権限を付与しないでください。

アクセシビリティの承認によって、アクティビティ共有が有効になることはありません。**Settings -> Permissions -> Active computer detection** は、制限されたアイドル時間を Gateway と共有するための、デフォルトではオフになっている別の設定です。これをオフにすると、アクセシビリティ権限を取り消したり Node を切断したりすることなく、保持されているアクティビティが消去されます。

誤って `node` にアクセシビリティ権限を付与した場合は、System Settings -> Privacy & Security -> Accessibility からそのエントリを削除します。その後、UI 自動化を担う署名済みアプリまたはヘルパーに権限を付与します。

## プロンプトが表示されない場合の復旧チェックリスト

1. アプリを終了します。
2. System Settings -> Privacy & Security からアプリのエントリを削除します。
3. 同じパスからアプリを再起動し、権限を再度付与します。
4. それでもプロンプトが表示されない場合は、`tccutil` を使用して TCC エントリをリセットし、もう一度試します。
5. 一部の権限は、macOS を完全に再起動しないと再表示されません。

リセットの例（OpenClaw のバンドル ID、`ai.openclaw.mac` を使用）：

```bash
sudo tccutil reset Accessibility ai.openclaw.mac
sudo tccutil reset ScreenCapture ai.openclaw.mac
sudo tccutil reset AppleEvents
```

## ファイルとフォルダの権限（Desktop/Documents/Downloads）

macOS は、ターミナルやバックグラウンドのプロセスによる Desktop、Documents、Downloads へのアクセスも制限することがあります。ファイルの読み取りやディレクトリ一覧の取得が停止する場合は、ファイル操作を行うのと同じプロセスコンテキスト（Terminal/iTerm、LaunchAgent から起動されたアプリ、SSH プロセスなど）にアクセス権限を付与してください。

回避策：フォルダごとの権限付与を避ける場合は、ファイルを OpenClaw ワークスペース（`~/.openclaw/workspace`）に移動します。

権限をテストする場合は、必ず正規の証明書で署名してください。アドホックビルドを使用できるのは、権限が重要ではない短時間のローカル実行に限られます。

## 関連項目

- [macOS アプリ](/ja-JP/platforms/macos)
- [macOS の署名](/ja-JP/platforms/mac/signing)
