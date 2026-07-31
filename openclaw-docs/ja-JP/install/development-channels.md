---
read_when:
    - stable/extended-stable/beta/dev を切り替えたい場合
    - 特定のバージョン、タグ、または SHA に固定したい場合
    - プレリリースにタグを付けるか、公開しようとしています
sidebarTitle: Release Channels
summary: 安定版、延長安定版、ベータ版、開発版の各チャンネル：セマンティクス、切り替え、バージョン固定、タグ付け
title: リリースチャンネル
x-i18n:
    generated_at: "2026-07-26T09:44:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a99e31f5121c0ab8696e638cb10a7ce16e8f32c81e4b2bef1f703eef71191494
    source_path: install/development-channels.md
    workflow: 16
---

OpenClaw は 4 つの更新チャンネルを提供します。

- **stable**: npm dist-tag `latest`。ほとんどのユーザーに推奨されます。
- **extended-stable**: npm dist-tag `extended-stable`。新規に追加された、サポート対象月を追従する
  パッケージチャンネルです。パッケージ専用で、インストールは
  フォアグラウンドでのみ実行されます。保存された選択では、
  `update.checkOnStart` が有効な場合に読み取り専用の更新通知を受け取りますが、
  自動的に適用されることはありません。
- **beta**: npm dist-tag `beta`。`beta` が存在しない場合、または
  現在の stable リリースより古い場合は、`latest` にフォールバックします。
- **dev**: `main`（git）の移動する先端です。公開されている場合は npm dist-tag `dev`。`main`
  は実験および活発な開発向けであり、未完成の
  機能や破壊的変更が含まれる場合があります。本番環境の Gateway では実行しないでください。

stable ビルドは通常、最初に **beta** としてリリースされ、そこで検証された後、
バージョンを上げずに **latest** に昇格されます。メンテナーは
`latest` に直接公開することもできます。npm インストールでは dist-tag が信頼できる唯一の情報源です。

## チャンネルの切り替え

```bash
openclaw update --channel stable
openclaw update --channel extended-stable
openclaw update --channel beta
openclaw update --channel dev
```

`--channel` は選択を設定の `update.channel` に永続化し、両方の
インストール方法を制御します。

| チャンネル        | npm／パッケージインストール                                                                                                                                                            | git インストール                                                                                                                                                   |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `stable`          | dist-tag `latest`                                                                                                                                                                      | 最新の stable git タグ（`-alpha.N`、`-beta.N`、`-rc.N`、`-dev.N`、`-next.N`、`-preview.N`、`-canary.N`、`-nightly.N`、およびその他の名前付きプレリリースサフィックスを除外） |
| `extended-stable` | 公開 npm の `extended-stable` セレクターを解決し、選択されたパッケージを厳密に検証して、その正確なバージョンをインストールします。`latest`、`beta`、`dev` へのフォールバックは行わず、安全側で失敗します。 | 未サポート：OpenClaw はチェックアウトを変更せず、パッケージインストールを使用するよう求めます                                                                      |
| `beta`            | dist-tag `beta`。`beta` が存在しないか古い場合は `latest` にフォールバック                                                                                       | 最新の beta git タグ。beta が存在しないか古い場合は最新の stable git タグにフォールバック                                                                          |
| `dev`             | dist-tag `dev`（まれ。ほとんどの dev ユーザーは git インストールを使用）                                                                                                             | フェッチし、チェックアウトを upstream の `main` ブランチ上にリベースして、ビルドし、グローバル CLI を再インストールします                              |

`dev` の git インストールでは、デフォルトのチェックアウトは `~/openclaw` です
（`OPENCLAW_HOME` が設定されている場合は `$OPENCLAW_HOME/openclaw`）。上書きするには
`OPENCLAW_GIT_DIR` を使用します。

<Tip>
stable と dev を並行して維持するには、2 つの個別のチェックアウトを使用し、各 Gateway がそれぞれ専用のチェックアウトを参照するようにしてください。
</Tip>

## 1 回限りのバージョンまたはタグ指定

永続化されたチャンネルを変更**せずに**、1 回の更新で特定の dist-tag、バージョン、
またはパッケージ指定を対象にするには、`--tag` を使用します。

```bash
# 特定のバージョンをインストール
openclaw update --tag 2026.4.1-beta.1

# beta dist-tag からインストール（1 回限り、永続化しない）
openclaw update --tag beta

# 移動する GitHub main チェックアウトに切り替え（永続化）
openclaw update --channel dev

# 特定の npm パッケージ指定をインストール
openclaw update --tag openclaw@2026.4.1-beta.1

# チャンネルを永続化せずに GitHub main から 1 回だけインストール
openclaw update --tag main
```

注：

- `--tag` は**パッケージ（npm）インストールにのみ**適用されます。git インストールでは無視されます。
- タグは永続化されません。次回の `openclaw update` では、設定済みの
  チャンネルが使用されます。
- `--tag main` は、その 1 回の実行に対して npm 互換の指定 `github:openclaw/openclaw#main`
  にマッピングされます。移動する `main` インストールを永続化するには、
  `openclaw update --channel dev` を使用するか（パッケージインストールは git チェックアウトに切り替わります）、
  インストーラーの git 方式で再インストールします：
  `curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git --version main`。
  npm インストール方法では GitHub／git ソースターゲットが完全に拒否され、
  代わりに git 方式を使用するよう案内されます。
- ダウングレード保護：対象バージョンが現在の
  バージョンより古い場合、OpenClaw は確認を求めます（`--yes` でスキップできます）。
- extended-stable は常に、検証済みの正確なパッケージターゲットを使用します。これは
  `--tag extended-stable` の 1 回限りの別名ではなく、`--tag` を
  有効な extended-stable チャンネルと組み合わせることはできません。
- `--channel beta` は `--tag beta` とは異なります。チャンネルフローでは beta が
  存在しないか古い場合に stable／latest にフォールバックできますが、`--tag beta` は
  常にその 1 回の実行で未加工の `beta` dist-tag を対象にします。

## ドライラン

変更を加えずに、`openclaw update` が実行する内容をプレビューします。

```bash
openclaw update --dry-run
openclaw update --channel beta --dry-run
openclaw update --tag 2026.4.1-beta.1 --dry-run
openclaw update --dry-run --json
```

ドライランでは、有効なチャンネル、対象バージョン、予定されているアクション、
およびダウングレードの確認が必要かどうかが報告されます。

## Plugin とチャンネル

`openclaw update` でチャンネルを切り替えると、Plugin のソースも同期されます。

- `dev` は、バンドル版に対応するインストール済み Plugin を、
  バンドル版（git チェックアウト）のソースに戻します。
- `stable` と `beta` は、npm または ClawHub からインストールされた Plugin
  パッケージを復元します。
- `extended-stable` は、bare／default
  または `latest` の指定を持つ対象の公式 npm Plugin を、インストール済みの core とまったく同じバージョンに解決します。実行時に
  Plugin の `@extended-stable` タグを照会することはありません。
- npm でインストールされた Plugin は、core の更新完了後に更新されます。

## 現在の状態の確認

```bash
openclaw update status
```

アクティブなチャンネル（それを決定したソース：設定、git タグ、
git ブランチ、インストール済みバージョン、またはデフォルト）、インストール種別（git またはパッケージ）、
現在のバージョン、および更新の有無を表示します。

## タグ付けのベストプラクティス

- git チェックアウトの到達先にするリリースにはタグを付けます。stable には `vYYYY.M.PATCH`、
  beta には `vYYYY.M.PATCH-beta.N` を使用します。
  `-alpha.N`、`-rc.N`、`-next.N` などの名前付きプレリリースサフィックスは、stable または beta のターゲットではありません。
- `vYYYY.M.PATCH-1` や `v1.0.1-1` などの従来の数値形式の stable タグも、
  互換性のため stable git タグとして引き続き認識されます。
- `vYYYY.M.PATCH.beta.N`（ドット区切り）も互換性のため認識されます。
  `-beta.N` を推奨します。
- タグを不変に保ってください。タグの移動や再利用は絶対に行わないでください。
- npm インストールでは、npm dist-tag が引き続き信頼できる唯一の情報源です：
  - `latest` -> stable
  - `extended-stable` -> サポート対象月を追従するパッケージリリース
  - `beta` -> 候補ビルドまたは beta を先行させる stable ビルド
  - `dev` -> main スナップショット（任意）

## macOS アプリの提供状況

beta および dev ビルドには、macOS アプリのリリースが**含まれない**場合があります。これは問題ありません。

- git タグと npm dist-tag は、それぞれ単独でも公開できます。
- リリースノートまたは変更履歴に「この beta には macOS ビルドがありません」と明記してください。

## 関連項目

- [更新](/ja-JP/install/updating)
- [インストーラーの内部構造](/ja-JP/install/installer)
