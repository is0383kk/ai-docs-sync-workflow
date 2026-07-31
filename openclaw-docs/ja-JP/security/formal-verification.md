---
permalink: /security/formal-verification/
read_when:
    - 正式なセキュリティモデルの保証または制限のレビュー
    - TLA+/TLC セキュリティモデルチェックの再現または更新
summary: OpenClaw の最もリスクの高いパス向けの機械検証済みセキュリティモデル。
title: 形式検証（セキュリティモデル）
x-i18n:
    generated_at: "2026-07-26T09:18:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 185ee5c1cff7325f10827330c0c7e55ddc3ca40caf6088d4c930ae5e090d6b27
    source_path: security/formal-verification.md
    workflow: 16
---

OpenClaw の形式的セキュリティモデル（現在は TLA+/TLC）は、明示された前提の下で、特にリスクの高い経路（認可、セッション分離、ツールのゲーティング、設定ミスに対する安全性）が意図されたポリシーを適用することを、機械的に検証された論証によって示します。

> 注: 一部の古いリンクでは、以前のプロジェクト名が使用されている場合があります。

## これは何か

実行可能で、攻撃者の行動を基にしたセキュリティ回帰スイートです。

- 各主張には、有限状態空間で実行可能なモデル検査があります。
- 多くの主張には、現実的なバグの種類について反例トレースを生成する、対応するネガティブモデルがあります。

これは、OpenClaw があらゆる面で安全であることの証明では**なく**、TypeScript 実装全体を検証するものでもありません。

## モデルの場所

モデルは別のリポジトリで管理されています: [vignesh07/openclaw-formal-models](https://github.com/vignesh07/openclaw-formal-models)。

<Note>
現在、そのリポジトリにはアクセスできません（本稿執筆時点では、GitHub が「Repository not found」を返します）。引き続きアクセスできない場合は、モデルが削除されたと判断する前に、OpenClaw のメンテナーチャンネルで現在の場所を確認してください。
</Note>

## 注意事項

- これらはモデルであり、TypeScript 実装全体ではありません。そのため、モデルとコードの間に乖離が生じる可能性があります。
- 結果は、TLC が探索する状態空間によって制限されます。グリーンであっても、モデル化された前提と範囲を超えた安全性を意味するものではありません。
- 一部の主張は、明示的な環境上の前提（たとえば、正しいデプロイと正しい設定入力）に依存します。

## 結果の再現

モデルのリポジトリをクローンし、TLC を実行します。

```bash
git clone https://github.com/vignesh07/openclaw-formal-models
cd openclaw-formal-models

# Java 11+ が必要です（TLC は JVM 上で動作します）。
# このリポジトリには固定バージョンの tla2tools.jar が同梱されており、bin/tlc と Make ターゲットが提供されています。

make <target>
```

このリポジトリへの CI 統合はまだありません。今後の改善では、公開アーティファクト（反例トレース、実行ログ）を伴う CI 実行モデルや、小規模な有界検査向けのホスト型「このモデルを実行」ワークフローを追加できる可能性があります。

## 主張とターゲット

### Gateway の公開とオープン Gateway の設定ミス

**主張:** 認証なしでループバック以外にバインドすると、リモートからの侵害が可能になり、公開範囲が拡大する可能性があります。モデルの前提では、トークンまたはパスワードによって未認証の攻撃者を阻止できます。

| 結果           | ターゲット                                                       |
| -------------- | ---------------------------------------------------------------- |
| グリーン       | `make gateway-exposure-v2`, `make gateway-exposure-v2-protected` |
| レッド（想定） | `make gateway-exposure-v2-negative`                              |

モデルのリポジトリにある `docs/gateway-exposure-matrix.md` も参照してください。

### Node 実行パイプライン（最もリスクの高い機能）

**主張:** モデルでは、`exec host=node` には、(a) Node コマンドの許可リストと宣言済みコマンド、および (b) 設定されている場合のリアルタイム承認が必要です。承認は再利用を防ぐためにトークン化されます。

| 結果           | ターゲット                                                      |
| -------------- | --------------------------------------------------------------- |
| グリーン       | `make nodes-pipeline`, `make approvals-token`                   |
| レッド（想定） | `make nodes-pipeline-negative`, `make approvals-token-negative` |

### ペアリングストア（DM ゲーティング）

**主張:** ペアリング要求には TTL と保留中要求数の上限が適用されます。

| 結果           | ターゲット                                           |
| -------------- | ---------------------------------------------------- |
| グリーン       | `make pairing`, `make pairing-cap`                   |
| レッド（想定） | `make pairing-negative`, `make pairing-cap-negative` |

### 受信ゲーティング（メンションと制御コマンドによる回避）

**主張:** メンションを必要とするグループコンテキストでは、認可されていない制御コマンドがメンションゲーティングを回避することはできません。

| 結果           | ターゲット                   |
| -------------- | ---------------------------- |
| グリーン       | `make ingress-gating`          |
| レッド（想定） | `make ingress-gating-negative` |

### ルーティングとセッションキーの分離

**主張:** 明示的にリンクまたは設定されていない限り、異なる相手からの DM が同じセッションに統合されることはありません。

| 結果           | ターゲット                      |
| -------------- | ------------------------------- |
| グリーン       | `make routing-isolation`          |
| レッド（想定） | `make routing-isolation-negative` |

## v1++ モデル: 並行処理、再試行、トレースの正確性

非アトミックな更新、再試行、メッセージのファンアウトなど、現実の障害モードに対する忠実度を高める追加モデルです。

### ペアリングストアの並行処理と冪等性

**主張:** ペアリングストアは、処理が交錯する場合でも `MaxPending` と冪等性を適用します。確認後の書き込みはアトミックまたはロック済みでなければならず、更新によって重複が作成されてはなりません。具体的には、同時リクエストがチャンネルの `MaxPending` を超えることはなく、同じ `(channel, sender)` に対するリクエストや更新を繰り返しても、有効な保留行が重複して作成されることはありません。

| 結果           | ターゲット                                                                                                                                                                  |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| グリーン       | `make pairing-race`（アトミックまたはロック済みの上限確認）、`make pairing-idempotency`、`make pairing-refresh`、`make pairing-refresh-race`                                              |
| レッド（想定） | `make pairing-race-negative`（非アトミックな開始・コミットによる上限競合）、`make pairing-idempotency-negative`、`make pairing-refresh-negative`、`make pairing-refresh-race-negative` |

### 受信トレースの相関と冪等性

**主張:** 取り込み処理は、ファンアウト全体でトレースの相関を維持し、プロバイダーによる再試行に対して冪等です。1 つの外部イベントが複数の内部メッセージになる場合、各部分は同じトレースおよびイベントの識別情報を維持します。再試行によって二重処理されることはありません。プロバイダーのイベント ID がない場合、重複排除は安全なキー（たとえばトレース ID）にフォールバックし、異なるイベントが破棄されるのを防ぎます。

| 結果           | ターゲット                                                                                                                                    |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| グリーン       | `make ingress-trace`, `make ingress-trace2`, `make ingress-idempotency`, `make ingress-dedupe-fallback`                                     |
| レッド（想定） | `make ingress-trace-negative`, `make ingress-trace2-negative`, `make ingress-idempotency-negative`, `make ingress-dedupe-fallback-negative` |

### ルーティングの dmScope 優先順位と identityLinks

**主張:** `dmScope` の優先順位と ID リンクは決定論的に動作します。デフォルトの `main` スコープでは、1 人の所有者の DM 全体で単一のローリングセッションが共有されます（個人用エージェントのデフォルト）。一方、分離するよう設定されたスコープ（`per-peer`、`per-channel-peer`、`per-account-channel-peer`）では、DM セッションが厳密に分離されます。チャンネル固有の `dmScope` オーバーライドはグローバルデフォルトより優先されます。`identityLinks` は、明示的にリンクされたグループ内でのみセッションを統合し、無関係な相手との間では統合しません。複数ユーザー向けの受信トレイでは、分離スコープを明示的に有効にすることが想定されています（複数ユーザーの DM トラフィックを検出すると、ランタイムのセキュリティ監査がこれを推奨します）。

| 結果           | ターゲット                                                                |
| -------------- | ------------------------------------------------------------------------- |
| グリーン       | `make routing-precedence`, `make routing-identitylinks`                   |
| レッド（想定） | `make routing-precedence-negative`, `make routing-identitylinks-negative` |

## 関連項目

- [脅威モデル](/ja-JP/security/THREAT-MODEL-ATLAS)
- [脅威モデルへの貢献](/ja-JP/security/CONTRIBUTING-THREAT-MODEL)
- [インシデント対応](/ja-JP/security/incident-response)
