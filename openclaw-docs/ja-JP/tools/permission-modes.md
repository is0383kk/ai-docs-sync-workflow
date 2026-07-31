---
read_when:
    - コマンド権限で auto、ask、allowlist、full、deny のいずれかを選択する
    - tools.exec.mode を使用した Codex Guardian レビュー済み承認の設定
    - OpenClaw の exec 承認と ACPX ハーネス権限の比較
summary: ホスト実行の権限モード、Codex Guardian の承認、および ACPX ハーネスセッション
title: 権限モード
x-i18n:
    generated_at: "2026-07-26T10:33:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f580e66508c1f69e868ed26a62d88a675f86a4d1ca738650dc5af82e967f3ac3
    source_path: tools/permission-modes.md
    workflow: 16
---

権限モードは、エージェントがホストコマンドを実行したり、ファイルを書き込んだり、追加アクセスをバックエンドハーネスに要求したりする前に、どの程度の権限を持つかを決定します。

<Note>
  権限モードは `tools.exec.host=auto` とは別です。`tools.exec.host` は
  コマンドを実行する場所を選択します。`tools.exec.mode` はホストでの実行を
  どのように承認するかを選択します。
</Note>

## 推奨されるデフォルト

すべての不一致で人間への確認を発生させることなく、実用的なホストアクセスを必要とするコーディングエージェントには `auto` を使用します。

```bash
openclaw config set tools.exec.mode auto
openclaw approvals get
openclaw gateway restart
```

次に、有効なポリシーを確認します。

```bash
openclaw exec-policy show
```

## OpenClaw のホスト実行モード

`tools.exec.mode` は、ホスト `exec` の正規化されたポリシーサーフェスです。各モードは、基盤となる `security`（許可リストの厳格度）と `ask`（不一致時の確認）の組み合わせに解決されます。

| モード        | security / ask          | 動作                                                                                      | 使用する状況                                              |
| ----------- | ----------------------- | --------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| `deny`      | `deny` / `off`          | ホストでの実行を完全にブロックします。                                                                     | ホストコマンドを一切許可しない場合。                         |
| `allowlist` | `allowlist` / `off`     | 許可リストにあるコマンドのみを実行し、不一致は通知せず拒否します。                                          | 安全性が確認済みのコマンドセットがある場合。                    |
| `ask`       | `allowlist` / `on-miss` | 許可リストに一致するコマンドを実行し、不一致時は人間に確認します。                                                 | 新しいコマンドを毎回人間がレビューする必要がある場合。              |
| `auto`      | `allowlist` / `on-miss` | 許可リストに一致するコマンドを実行し、不一致は人間の承認にフォールバックする前に自動レビューに送ります。 | コーディングセッションで、保護された実用的なアクセスが必要な場合。        |
| `full`      | `full` / `off`          | 確認なしでホスト上のコマンドを実行します。                                                                | この信頼済みホスト／セッションで承認ゲートを省略する場合。 |

`ask` と `auto` は同じ許可リスト／確認設定を共有します。`auto` ではさらにネイティブの自動レビュアーが有効になり、不一致を自身で判断し、安全に承認できない場合にのみ、設定済みの人間による承認経路へ委ねます。

ホスト実行ポリシーの全容、ローカル承認ファイル、許可リストのスキーマ、安全なバイナリ、転送動作については、[実行の承認](/ja-JP/tools/exec-approvals)を参照してください。

## Codex Guardian のマッピング

ネイティブ Codex app-server セッションでは、ローカルの Codex 要件で許可される場合、`tools.exec.mode: "auto"` によって Codex は Guardian がレビューする承認方式を使用するようになります。通常、結果は次の値になります。

| Codex フィールド         | 通常の値     |
| ------------------- | ----------------- |
| `approvalPolicy`    | `on-request`      |
| `approvalsReviewer` | `auto_review`     |
| `sandbox`           | `workspace-write` |

`auto` モードでは、設定済みの Codex サンドボックス／承認オーバーライドよりもこのポリシーが優先されるため、`approvalPolicy: "never"` と `sandbox: "danger-full-access"` の組み合わせのような従来の安全でない設定は保持されません。`tools.exec.mode: "deny"` と `"allowlist"` は、Codex app-server のローカル実行を完全にブロックします。承認を不要とする構成を意図的に使用する場合にのみ、`tools.exec.mode: "full"` を使用してください。

app-server のセットアップ、認証順序、ネイティブ Codex ランタイムの詳細については、[Codex ハーネス](/ja-JP/plugins/codex-harness)を参照してください。

## ACPX ハーネスの権限

ACPX セッションは非対話型であるため、TTY の権限プロンプトをクリックできません。ACPX は `plugins.entries.acpx.config` 配下にある個別のハーネスレベル設定を使用します。

| 設定                     | 値          | 意味                                     |
| --------------------------- | --------------- | ------------------------------------------- |
| `permissionMode`            | `approve-reads` | 読み取りのみを自動承認します。                    |
| `permissionMode`            | `approve-all`   | 書き込みとシェルコマンドを自動承認します。     |
| `permissionMode`            | `deny-all`      | すべての権限プロンプトを拒否します。                |
| `nonInteractivePermissions` | `fail`          | プロンプトが必要になる場合は中止します。      |
| `nonInteractivePermissions` | `deny`          | プロンプトを拒否し、可能な場合は続行します。 |

ACPX の権限は、OpenClaw の実行承認とは別に設定します。

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions fail
openclaw gateway restart
```

プロンプトなしのハーネスセッションに相当する ACPX の緊急時設定として、`approve-all` を使用します。セットアップの詳細と失敗モードについては、[ACP エージェントのセットアップ](/ja-JP/tools/acp-agents-setup#permission-configuration)を参照してください。

## モードの選択

| 目的                                          | 設定                                                   |
| --------------------------------------------- | ----------------------------------------------------------- |
| ホストコマンドを完全にブロックする                | `tools.exec.mode: "deny"`                                   |
| 安全性が確認済みのコマンドのみ実行を許可する              | `tools.exec.mode: "allowlist"`                              |
| 新しいコマンド形式ごとに人間へ確認する       | `tools.exec.mode: "ask"`                                    |
| 人間による確認の前に Codex／OpenClaw の自動レビューを使用する  | `tools.exec.mode: "auto"`                                   |
| ホスト実行の承認を完全に省略する             | `tools.exec.mode: "full"` と対応するホスト承認ファイル |
| 非対話型 ACPX セッションで書き込み／実行を許可する | `plugins.entries.acpx.config.permissionMode: "approve-all"` |

モードを変更してもコマンドが確認を求める、または失敗する場合は、両方のレイヤーを調べます。

```bash
openclaw approvals get
openclaw exec-policy show
```

ホストでの実行には、OpenClaw 設定とホストローカルの承認ファイルのうち、より厳格な結果が使用されます。ACPX ハーネスの権限によってホスト実行の承認が緩和されることはなく、ホスト実行の承認によって ACPX ハーネスのプロンプトが緩和されることもありません。

## 関連項目

- [実行の承認](/ja-JP/tools/exec-approvals)
- [実行の承認 - 高度な設定](/ja-JP/tools/exec-approvals-advanced)
- [Codex ハーネス](/ja-JP/plugins/codex-harness)
- [ACP エージェントのセットアップ](/ja-JP/tools/acp-agents-setup#permission-configuration)
