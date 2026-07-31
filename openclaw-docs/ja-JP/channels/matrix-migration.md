---
read_when:
    - 既存の Matrix インストールのアップグレード
    - 暗号化された Matrix の履歴とデバイス状態の移行
summary: 暗号化状態の復旧制限と手動復旧手順を含め、OpenClaw が以前の Matrix Plugin をその場でアップグレードする方法。
title: Matrix の移行
x-i18n:
    generated_at: "2026-07-26T08:54:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 475c96914900a5597f37001264bd3d8f69a69dbd0600f2704c2a1be46924fac4
    source_path: channels/matrix-migration.md
    workflow: 16
---

以前の公開 `matrix` Plugin から現在の実装にアップグレードします。

ほとんどのユーザーは、そのままアップグレードできます。

- Plugin は `@openclaw/matrix` のままです
- チャンネルは `matrix` のままです
- 設定は引き続き `channels.matrix` の下にあります
- キャッシュされた認証情報は共有 `state/openclaw.sqlite` Plugin 状態に移動します
- ランタイム状態は引き続き `~/.openclaw/matrix/` の下にあります

設定キーの名前を変更したり、新しい名前で Plugin を再インストールしたりする必要はありません。
ルートの `openclaw` パッケージには、Matrix ランタイムコードや Matrix SDK の依存関係がバンドルされなくなりました。`openclaw channels status` で Matrix が設定済みであるものの Plugin がインストールされていないと表示される場合は、`openclaw doctor --fix` または `openclaw plugins install @openclaw/matrix` を実行してください。Matrix SDK パッケージをルートの OpenClaw パッケージにインストールしないでください。

## 移行によって自動的に行われること

Matrix の移行は、[`openclaw doctor --fix`](/ja-JP/gateway/doctor) を実行すると行われます。専用の Matrix ストアの隣にあるファイルベースのサイドカーでは、クライアント起動時のフォールバックが維持されますが、認証情報ファイルのインポートは Doctor でのみ行われます。ランタイムは正規の SQLite 認証情報状態のみを読み取ります。

Doctor の移行対象は次のとおりです。

- 廃止された `~/.openclaw/credentials/matrix/credentials*.json` ファイルをアーカイブする前にインポートして検証する
- 同じアカウント選択と `channels.matrix` 設定を維持する
- ファイルベースのサイドカー状態（`bot-storage.json` 同期キャッシュ、`recovery-key.json`、`legacy-crypto-migration.json`、IndexedDB スナップショット）を Matrix SQLite 状態にインポートする。移行されたファイルは `.migrated` サフィックスを付けてアーカイブされます
- 後でアクセストークンが変更された場合に、同じ Matrix アカウント、ホームサーバー、ユーザー、デバイスについて、既存の最も完全なトークンハッシュストレージルートを再利用する

## 2026.4 より前の OpenClaw リリースからのアップグレード

2026.6 系までのリリースでは、元のフラットな単一ストアの Matrix レイアウト（`~/.openclaw/matrix/bot-storage.json` と `~/.openclaw/matrix/crypto/`）も移行し、古い Rust 暗号化ストアから暗号化状態を復旧する準備を行っていました。現在のリリースには、その移行処理は含まれていません。

まだフラットレイアウトを使用しているインストール環境をアップグレードする場合は、まず 2026.6 リリースにアップグレードし、`openclaw doctor --fix` を実行して、Gateway を一度起動してください。これにより、フラットストアと復旧可能なルームキーが移行されます。その後、最新リリースに更新してください。

以前の公開 Matrix Plugin は、Matrix のルームキーバックアップを自動的には作成して**いませんでした**。古いインストール環境に、バックアップされたことのないローカルのみの暗号化履歴があった場合、移行方法にかかわらず、アップグレード後も一部の古い暗号化メッセージを読み取れないことがあります。

## 推奨アップグレード手順

1. OpenClaw と Matrix Plugin を通常どおり更新します。
2. 次を実行します。

   ```bash
   openclaw doctor --fix
   ```

3. Gateway を起動または再起動します。
4. 現在の検証状態とバックアップ状態を確認します。

   ```bash
   openclaw matrix verify status
   openclaw matrix verify backup status
   ```

5. 修復する Matrix アカウントのリカバリーキーを、アカウント固有の環境変数に設定します。デフォルトアカウントが 1 つだけの場合は、`MATRIX_RECOVERY_KEY` で問題ありません。複数のアカウントがある場合は、アカウントごとに 1 つの変数（例: `MATRIX_RECOVERY_KEY_ASSISTANT`）を使用し、コマンドに `--account assistant` を追加します。

6. リカバリーキーが必要であると OpenClaw に表示された場合は、該当するアカウントに対して次のコマンドを実行します。

   ```bash
   printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify backup restore --recovery-key-stdin
   printf '%s\n' "$MATRIX_RECOVERY_KEY_ASSISTANT" | openclaw matrix verify backup restore --recovery-key-stdin --account assistant
   ```

7. このデバイスがまだ未検証の場合は、該当するアカウントに対して次のコマンドを実行します。

   ```bash
   printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify device --recovery-key-stdin
   printf '%s\n' "$MATRIX_RECOVERY_KEY_ASSISTANT" | openclaw matrix verify device --recovery-key-stdin --account assistant
   ```

   リカバリーキーが受け入れられ、バックアップを使用できるにもかかわらず、`Cross-signing verified` がまだ `no` の場合は、別の Matrix クライアントから自己検証を完了します。

   ```bash
   openclaw matrix verify self
   ```

   別の Matrix クライアントでリクエストを承認し、絵文字または数字を比較して、一致する場合にのみ `yes` と入力します。コマンドは、Matrix ID が完全に信頼されるまで待機してから成功を報告します。

8. 復旧できない古い履歴を意図的に破棄し、今後のメッセージ用に新しいバックアップの基準状態を作成する場合は、次を実行します。

   ```bash
   openclaw matrix verify backup reset --yes
   ```

   古いリカバリーキーで新しいバックアップを解除できないようにする場合にのみ、`--rotate-recovery-key` を追加します。

9. サーバー側のキーバックアップがまだ存在しない場合は、今後の復旧用に作成します。

   ```bash
   openclaw matrix verify bootstrap
   ```

## よくあるメッセージとその意味

`Failed migrating legacy Matrix client storage: ...`

- 意味: Matrix のクライアント側フォールバックでファイルベースのサイドカー状態が見つかりましたが、SQLite へのインポートに失敗しました。OpenClaw は、何も通知せずに新しいストアで起動するのではなく、完了した移動をロールバックして、そのフォールバックを中止します。
- 対処方法: ファイルシステムの権限や競合を調べ、古い状態をそのまま保持し、エラーを修正してから再試行します。

`Matrix is installed from a custom path: ...`

- 意味: Matrix がパスインストールに固定されているため、メインラインの更新ではデフォルトの Matrix パッケージに自動的に置き換えられません。
- 対処方法: デフォルトの Matrix Plugin に戻す場合は、`openclaw plugins install @openclaw/matrix` で再インストールします。

`Matrix is installed from a custom path that no longer exists: ...`

- 意味: Plugin のインストールレコードが、存在しなくなったローカルパスを指しています。
- 対処方法: `openclaw plugins install @openclaw/matrix` で再インストールするか、リポジトリのチェックアウトから実行している場合は `openclaw plugins install ./path/to/local/matrix-plugin` を使用します。`openclaw doctor --fix` を使用して、古い Matrix Plugin の参照を削除することもできます。

### 手動復旧メッセージ

このデバイスでルームキーのバックアップが正常でない場合、`openclaw matrix verify status` と `openclaw matrix verify backup status` は、`Backup issue:` 行と `Next steps:` のガイダンスを出力します。

| バックアップの問題                                                      | 意味                                               | 修正方法                                                                                                                                   |
| --------------------------------------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `no room-key backup exists on the homeserver`                         | 復元元がない                                       | `openclaw matrix verify bootstrap` でルームキーのバックアップを作成する                                                                            |
| `backup decryption key is not loaded on this device`                  | キーは存在するが、ここでは有効になっていない       | `openclaw matrix verify backup restore`。それでもキーを読み込めない場合は、`--recovery-key-stdin` を使用してリカバリーキーをパイプで渡す                |
| `backup decryption key could not be loaded from secret storage (...)` | シークレットストレージの読み込みに失敗したか、サポートされていない | リカバリーキーをパイプで渡す: `printf '%s\n' "$MATRIX_RECOVERY_KEY" \| openclaw matrix verify backup restore --recovery-key-stdin`               |
| `backup key mismatch (...)`                                           | 保存されたキーが有効なサーバーバックアップと一致しない | 有効なサーバーバックアップキーを使用して `verify backup restore --recovery-key-stdin` を再実行するか、`verify backup reset --yes` で新しい基準状態を作成する |
| `backup signature chain is not trusted by this device`                | デバイスがクロス署名チェーンをまだ信頼していない   | `verify device --recovery-key-stdin` を実行し、信頼がまだ不完全な場合は別の検証済みクライアントから `verify self` を実行する                        |
| `backup exists but is not active on this device`                      | サーバーバックアップは存在するが、ローカルセッションが無効 | まずデバイスを検証し、`openclaw matrix verify backup status` で再確認する                                                         |
| `backup trust state could not be fully determined`                    | 診断で結論を得られなかった                         | `openclaw matrix verify status --verbose`                                                                                                 |

その他の復旧エラー:

`Matrix recovery key is required`

- 意味: リカバリーキーが必要な復旧手順を、リカバリーキーを指定せずに実行しました。
- 対処方法: `--recovery-key-stdin` を指定してコマンドを再実行します。例: `printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify device --recovery-key-stdin`。

`Invalid Matrix recovery key: ...`

- 意味: 指定されたキーを解析できなかったか、想定される形式と一致しませんでした。
- 対処方法: Matrix クライアントまたはリカバリーキーのエクスポートから取得した正確なリカバリーキーを使用して再試行します。

`Matrix recovery key was applied, but this device still lacks full Matrix identity trust.`

- 意味: リカバリーキーによって使用可能なバックアップデータを解除できましたが、Matrix はこのデバイスに対する完全なクロス署名 ID の信頼をまだ確立していません。コマンド出力で `Recovery key accepted`、`Backup usable`、`Cross-signing verified`、`Device verified by owner` を確認してください。
- 対処方法: `openclaw matrix verify self` を実行し、別の Matrix クライアントでリクエストを承認して SAS を比較し、一致する場合にのみ `yes` と入力します。現在のクロス署名 ID を意図的に置き換える場合にのみ、`printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify bootstrap --recovery-key-stdin --force-reset-cross-signing` を使用してください。

復旧できない古い暗号化履歴を失うことを受け入れる場合は、代わりに `openclaw matrix verify backup reset --yes` を使用して現在のバックアップの基準状態をリセットできます。保存されたバックアップシークレットが破損している場合、このリセットによってシークレットストレージも修復され、再起動後に新しいバックアップキーを正しく読み込めるようになります。

## 暗号化履歴がまだ戻らない場合

次の確認を順番に実行します。

```bash
openclaw matrix verify status --verbose
openclaw matrix verify backup status --verbose
printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify backup restore --recovery-key-stdin --verbose
```

バックアップが正常に復元されても、一部の古いルームの履歴がまだ欠落している場合、それらの欠落したキーは以前の Plugin でバックアップされていなかった可能性があります。

## 今後のメッセージ用に新しく始める場合

復旧できない古い暗号化履歴を失うことを受け入れ、今後に向けたクリーンなバックアップの基準状態のみが必要な場合は、次のコマンドを順番に実行します。

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

その後もデバイスが未検証の場合は、Matrix クライアントで SAS の絵文字または数字コードを比較し、一致することを確認して検証を完了します。

## 関連項目

- [Matrix](/ja-JP/channels/matrix): チャンネルのセットアップと設定。
- [Matrix プッシュルール](/ja-JP/channels/matrix-push-rules): 通知のルーティング。
- [Doctor](/ja-JP/gateway/doctor): ヘルスチェックと自動移行のトリガー。
- [移行ガイド](/ja-JP/install/migrating): すべての移行パス（マシン間の移動、システム間のインポート）。
- [Plugins](/ja-JP/tools/plugin): Plugin のインストールと登録。
