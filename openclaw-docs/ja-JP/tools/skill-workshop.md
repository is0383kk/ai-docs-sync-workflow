---
read_when:
    - チャットからエージェントに Skill の作成または更新を依頼する場合
    - 生成されたスキルのドラフトをレビューし、適用、却下、または隔離する必要があります
    - Skill Workshop の承認、自律性、ストレージ、または制限を設定している場合
    - 自己学習に関する提案がどこでレビューされるかを確認したい場合
sidebarTitle: Skill Workshop
summary: Skill Workshop のレビューを通じてワークスペースの Skills を作成・更新する
title: Skill ワークショップ
x-i18n:
    generated_at: "2026-07-26T09:23:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2c2590f2a1bcad3b22ef8504eac7b3a44611c3fedc0df3832660f8926ce04252
    source_path: tools/skill-workshop.md
    workflow: 16
---

Skill Workshop は、ワークスペースの Skills を作成および更新するために OpenClaw が管理する経路です。エージェントとオペレーターがこの経路を通じて `SKILL.md` を直接書き込むことはありません。代わりに、**提案**（内容、ターゲットの関連付け、スキャナーの状態、ハッシュ、ロールバックメタデータを含む保留中の下書き）を作成し、それが適用された場合にのみ有効な Skill になります。

Skill Workshop が書き込むのはワークスペースの Skills のみです。バンドル済み、Plugin、ClawHub、追加ルート、管理対象、個人エージェント、システムの Skills には一切触れません。

## 仕組み

- **提案が先:** 生成された内容は `SKILL.md` ではなく `PROPOSAL.md` として保存されます。
- **有効な Skill への書き込みは適用時のみ:** 作成、更新、改訂によってアクティブな Skills が変更されることはありません。
- **ワークスペーススコープ:** 作成先はワークスペースの `skills/` ルートです。更新できるのは、書き込み可能なワークスペースの Skills のみです。
- **上書きなし:** ターゲットの Skill がすでに存在する場合、作成は失敗します。
- **ハッシュに関連付け:** 更新提案は現在のターゲットハッシュに関連付けられ、適用前に有効な Skill が変更されると `stale` になります。
- **スキャナーによるゲート:** 適用時には、書き込み前にセキュリティスキャナーが再実行されます。
- **復旧可能:** 適用時には、有効なファイルに触れる前にロールバックメタデータが書き込まれます。
- **一貫したインターフェース:** チャット、CLI、Gateway はすべて同じサービスを呼び出します。

## ライフサイクル

```text
作成/更新 -> 保留中
改訂      -> 保留中
適用      -> 適用済み
却下      -> 却下済み
隔離      -> 隔離済み
ターゲット変更 -> 古い状態
```

改訂、適用、却下、隔離が可能なのは、`pending` の提案のみです。

## ライフサイクルの整理

Gateway は共有状態データベースで Skill の使用状況を集計します。1 日に 1 回、Skill Workshop によって作成および適用された Skills を確認します。30 日を超えて使用されていない Skills は `stale` になり、90 日後には `archived` となって、新しいエージェントの Skill スナップショットから除外されます。アーカイブされた Skill のファイルはディスク上で変更されません。手動で作成された Skills は整理の対象になりません。ライフサイクルの整理対象になるのは、Skill Workshop の提案によって作成された Skills のみです。

ピン留めされた Skills はライフサイクルの遷移を回避します。古い状態の Skill は、使用された後に次のスイープが実行されると `active` に戻ります。アーカイブされた Skills は、明示的な復元によってのみ戻ります。

ライフサイクルの遷移と復元は新しいセッションに適用されます。実行中のセッションでは、現在の Skill スナップショットが維持されます。

```bash
openclaw skills curator status
openclaw skills curator pin <skill>
openclaw skills curator unpin <skill>
openclaw skills curator restore <skill>
```

すべての curator コマンドで `--json` を指定できます。status は、決定論的に検出された重複候補も提案としてのみ報告します。Skills をマージしたり、モデルを呼び出したりすることはありません。

## チャット

必要な Skill をエージェントに依頼すると、エージェントが `skill_workshop` を呼び出し、提案 ID を返します。

### 最近の作業から学習する

`/learn` を使用すると、現在の会話または指定したソースを、標準に基づく 1 つの Skill 提案に変換できます。

```text
/learn
/learn docs/runbook.md と https://example.com/guide; 復旧に重点を置く
```

リクエストがない場合、`/learn` は現在の会話から再利用可能なワークフローを抽出するようエージェントに指示します。リクエストがある場合、エージェントは重点、スコープ、命名の要件を守りながら、パス、URL、貼り付けられたメモ、会話への参照をソースとして扱います。既存のツールでソースを収集してから、`action: "create"` を指定して `skill_workshop` を呼び出します。

生成された提案は `pending` のままです。`/learn` がそれを適用することはありません。通常の承認フローまたは `openclaw skills workshop` を使用して、確認して適用します。

作成:

```text
月曜日の受信トレイルーチンを実行する morning-catchup という Skill を作成して。
```

既存のワークスペースの Skill を更新:

```text
予約前に座席表も確認するよう trip-planning を更新して。
```

保留中の提案を反復修正:

```text
morning-catchup の提案を見せて。
緊急とマークされたものにもフラグを付けるよう改訂して。
morning-catchup の提案を適用して。
```

エージェントが開始した `apply`、`reject`、`quarantine` は、デフォルトでは追加の承認プロンプトなしで実行されます。これらのアクションの前にオペレーターの承認を必須にするには、`skills.workshop.approvalPolicy` を `"pending"` に設定します。

承認が必要な場合、プロンプトには提案 ID とターゲットの Skill が示され、提案の説明、サポートファイル数、本文サイズが表示されます。承認リクエストは、エージェントツールのウォッチドッグより先に完了するよう時間制限が設定されています。プロンプトの有効期限までに決定が行われなかった場合、ライフサイクルアクションは実行されず、提案は保留中かつ変更されないままです。後で Skill Workshop UI で決定するか、`openclaw skills workshop apply|reject|quarantine <proposal-id>` を実行してください。エージェントは、有効期限が切れたライフサイクルアクションをループで再試行しないでください。

## CLI

```bash
# 作成
openclaw skills workshop propose-create \
  --name morning-catchup \
  --description "毎日の受信トレイ整理: 優先順位付け、アーカイブ、重要項目の抽出、下書き、計画" \
  --proposal ./PROPOSAL.md

# 既存のワークスペースの Skill を更新
openclaw skills workshop propose-update trip-planning --proposal ./PROPOSAL.md

# 一覧表示と確認
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>

# 承認前に改訂
openclaw skills workshop revise <proposal-id> --proposal ./PROPOSAL.md

# 完了処理
openclaw skills workshop apply <proposal-id>
openclaw skills workshop reject <proposal-id> --reason "重複"
openclaw skills workshop quarantine <proposal-id> --reason "セキュリティレビューが必要"
```

すべてのサブコマンドで `--agent <id>`（ターゲットワークスペース。デフォルトでは cwd から推定し、次にデフォルトエージェントを使用）と `--json`（構造化出力）を指定できます。`propose-create`、`propose-update`、`revise` ではさらに、`--proposal` とともに提案のコンテキストを記録するため、`--goal <text>` と `--evidence <text>` も指定できます。

## 提案の内容

保留中の提案は、提案専用のフロントマターを持つ `PROPOSAL.md` として保存されます。

```markdown
---
name: "morning-catchup"
description: "毎日の受信トレイ整理: 優先順位付け、アーカイブ、重要項目の抽出、下書き、計画"
status: proposal
version: "v1"
date: "2026-05-30T00:00:00.000Z"
---
```

適用時に、Skill Workshop は有効な `SKILL.md` を書き込み、提案専用フィールドである `status`、提案の `version`、提案の `date` を削除します。

## サポートファイル

提案する Skill に `PROPOSAL.md` と同じ場所に配置するファイルが必要な場合は、`--proposal-dir` を使用します。

```bash
openclaw skills workshop propose-create \
  --name weekly-update \
  --description "金曜日のまとめ: 統計、ハイライト、来週の最重要 3 項目" \
  --proposal-dir ./weekly-update-proposal
```

ディレクトリには `PROPOSAL.md` が含まれている必要があります。サポートファイルは、`assets/`、`examples/`、`references/`、`scripts/`、`templates/` のいずれかの配下に配置する必要があります。Skill Workshop はそれらをスキャンしてハッシュを計算し、提案とともに保存します。その後、適用時にのみ、有効な `SKILL.md` と同じ場所に書き込みます。

拒否されるサポートファイルのパス: 絶対パス、隠しパスセグメント、パストラバーサル、重複するパス、実行可能ファイル、UTF-8 以外のテキスト、null バイト、標準サポートフォルダー外のパス。

## エージェントツール

モデルは、必須の `action` である `create | update | revise | list | inspect | apply | reject | quarantine` を 1 つ指定して `skill_workshop` を使用します。
その他のパラメーターは、アクションに応じて適用されます。

| パラメーター                  | 使用するアクション                                      | 注記                                                                |
| -------------------------- | ---------------------------------------------------- | -------------------------------------------------------------------- |
| `name`                     | `create`、`inspect`、`revise`                        | `create` では必須。それ以外では名前から保留中の提案を解決 |
| `description`              | `create`、`update`、`revise`                         | 最大 160 バイト                                                        |
| `skill_name`               | `update`                                             | 既存の Skill 名またはキー                                           |
| `proposal_content`         | `create`、`update`、`revise`                         | `PROPOSAL.md` として保存。上限は `skills.workshop.maxSkillBytes`   |
| `support_files`            | `create`、`update`、`revise`                         | `{ path, content }` の配列                                         |
| `goal`、`evidence`         | `create`、`update`、`revise`                         | 自由記述のコンテキスト                                                    |
| `proposal_id`              | `inspect`、`revise`、`apply`、`reject`、`quarantine` | ターゲットの提案                                                      |
| `reason`                   | `apply`、`reject`、`quarantine`                      | 任意                                                             |
| `query`、`status`、`limit` | `list`                                               | フィルタリング/ページ分割。`limit` は最大 50、デフォルト 20                          |

エージェントは、生成する Skill の作業に `skill_workshop` を使用する必要があります。`write`、`edit`、`exec`、シェルコマンド、またはファイルシステムの直接操作を通じて、提案ファイルを作成または変更してはなりません。

<Note>
`skill_workshop` は組み込みのエージェントツールであり、`tools.profile: "coding"` に含まれています。より厳格なポリシーによって非表示になっている場合は、アクティブな `tools.allow` リストに `skill_workshop` を追加するか、明示的な `tools.allow` を持たないプロファイルをスコープが使用している場合は `tools.alsoAllow: ["skill_workshop"]` を使用してください。サンドボックス化された実行ではホスト側の Skill Workshop ツールが構築されないため、提案の確認アクションは通常のホスト側エージェントセッションまたは CLI から実行してください。
</Note>

## 提案される Skills

OpenClaw は、対話型ターンの終了時に、失敗したターンを含め、「次回は」「忘れずに」などの永続的な指示や、対応を求める修正を検出します。次のターンで、エージェントは直近に検出されたワークフローを `skill_workshop` を通じて保存することを提案します。提案を作成するかどうかはユーザーが決定します。この組み込みの提案機能自体が Skill を作成または変更することはありません。代わりに保留中の提案を直接作成するには、`skills.workshop.autonomous.enabled` を有効にします。Control UI では、Workshop タブのページヘッダーにある **自己学習** トグルと、空の提案ボードにある有効化ボタンから、同じ設定を利用できます。

### 過去のセッションをスキャンする

Control UI では、自律的な自己学習を有効にせずに、過去の作業を確認できます。
**Plugins → Workshop** を開き、**Skill のアイデアを探す** を選択します。スキャンは対象となる最新のセッションから開始され、実質的な作業を含む、上限のある範囲を確認します。Cron、Heartbeat、フック、サブエージェント、ACP、Plugin 所有、内部レビューのセッションに加え、モデルのターン数が 6 未満の会話はスキップされます。

レビュアーは、選択したエージェントに設定されたモデルを使用し、シークレットを削除してサイズを制限したトランスクリプトのバンドルを受け取ります。経験レビューと同じ保守的な基準を適用します。つまり、具体的な復旧パターン、または将来のモデルやツールの呼び出しを少なくとも 2 回削減できる安定した手順が必要です。日常的な作業や一度限りの事実からは、提案を生成しないようにします。

1 回のスキャンで作成または改訂できる保留中の提案は最大 3 件です。適用、却下、隔離、または有効な Skill の編集はできません。Workshop には、たとえば **20 件のセッションを確認済み · 6 月 18 日〜今日 · 2 件のアイデアを検出** のように、累積の対象範囲が表示されます。永続化された最古セッションのカーソルから続行するには、**さらに過去の作業をスキャン** を選択します。利用可能な履歴をすべて確認した後、アクションは **新しい作業をスキャン** に変わります。

履歴レビューは、`skills.workshop.autonomous.enabled` が `false` の場合でも手動です。クリックするたびにモデル実行が開始されるため、プロバイダーの料金とデータ処理条件が適用されます。カーソルとカバレッジ数は共有 OpenClaw 状態データベースに保存されますが、トランスクリプトの内容はスキャン状態にコピーされません。

自律キャプチャを有効にすると、OpenClaw は、十分な作業が正常に完了した後、およびエージェントシステム全体がアイドル状態になった後にも、保守的なレビューを実行できます。この分離されたレビューが作成または修正できる保留中の提案は、最大 1 件です。`approvalPolicy` が `"auto"` の場合でも、使用中のスキルを更新したり、提案を適用、却下、隔離したりすることはできません。

有効化、適格性、プライバシーとコストの詳細、提案のしきい値、およびトラブルシューティングについては、[自己学習](/ja-JP/tools/self-learning)を参照してください。

## 承認と自律性

```json5
{
  skills: {
    workshop: {
      autonomous: {
        enabled: false,
      },
      allowSymlinkTargetWrites: false,
      approvalPolicy: "auto",
      maxPending: 50,
      maxSkillBytes: 40000,
    },
  },
}
```

| 設定                       | デフォルト | 効果                                                                                                                                                                                                  |
| -------------------------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `autonomous.enabled`         | `false` | 明示的な修正から、またアイドル遅延後には再利用可能な復旧方法または有意な往復処理の削減を伴う、完了済みの十分な作業から、保留中の提案を作成します。                                                    |
| `allowSymlinkTargetWrites`         | `false` | 実体のターゲットが `skills.load.allowSymlinkTargets` に記載されているワークスペーススキルのシンボリックリンクを介して、適用時に書き込めるようにします。                                                               |
| `approvalPolicy`         | `"auto"` | `"auto"` は、エージェントが開始した `apply`、`reject`、または `quarantine` に対する追加プロンプトを省略します（エージェントは引き続きアクションを呼び出す必要があります）。`"pending"` では承認が必要です。 |
| `maxPending`         | `50` | ワークスペースごとの保留中および隔離済みの提案数を制限します（1～200）。                                                                                                                             |
| `maxSkillBytes`         | `40000` | 提案本文のサイズをバイト単位で制限します（1024～200000）。                                                                                                                                           |

自律キャプチャは、将来に向けたルール（たとえば「今後は」）と、事後的な修正（たとえば「依頼した内容と違う」）を認識します。新しい指示をトピック別にグループ化して、1 ターンあたり最大 3 件の提案にまとめ、語彙が一致する既存の書き込み可能なワークスペーススキルに振り分けます。また、別の修正が同じスキルを対象とする場合、自身の保留中の提案を修正します。

明示的な修正なしに十分な作業が正常に完了した場合は、選択されたモデルの分離された実行によって、完了した処理経路が保守的な提案基準を満たすかどうかが判断されます。フォアグラウンドモデルは、応答前に学習するよう促されません。バックグラウンドレビューアーは、提案の来歴としてフォアグラウンド実行を保持しますが、一般的なエージェントツールにはアクセスできず、ライフサイクルに関する判断も行えません。レビューは、フォアグラウンドランタイムが、正確に解決されたモデルと、`skill_workshop` が実際に利用可能であったことの両方を報告した場合にのみ開始されます。そのため、制限的または不明なツールポリシーでは安全側に失敗し、提案は作成されません。

自律レビューの完全な動作と安全モデルについては、[自己学習](/ja-JP/tools/self-learning)を参照してください。

提案の説明は、`maxSkillBytes` に関係なく、常に 160 バイトに制限されます。

## Gateway メソッド

| メソッド                           | スコープ         |
| ---------------------------------- | ---------------- |
| `skills.proposals.list`                 | `operator.read` |
| `skills.proposals.inspect`                 | `operator.read` |
| `skills.proposals.historyStatus`                 | `operator.read` |
| `skills.proposals.historyScan`                 | `operator.admin` |
| `skills.proposals.create`                 | `operator.admin` |
| `skills.proposals.update`                 | `operator.admin` |
| `skills.proposals.revise`                 | `operator.admin` |
| `skills.proposals.requestRevision`                 | `operator.admin` |
| `skills.proposals.apply`                 | `operator.admin` |
| `skills.proposals.reject`                 | `operator.admin` |
| `skills.proposals.quarantine`                 | `operator.admin` |
| `skills.curator.status`                 | `operator.read` |
| `skills.curator.pin`                 | `operator.admin` |
| `skills.curator.unpin`                 | `operator.admin` |
| `skills.curator.restore`                 | `operator.admin` |

`requestRevision` は Gateway 専用です（CLI またはエージェントツールに相当するものはありません）。エージェントに文字どおりの新しい内容を送信させるのではなく、修正を依頼する UI 向けに、`PROPOSAL.md` を直接置き換える代わりに、自由形式の修正指示を所有エージェントのチャットセッションに転送します。

`historyStatus` と `historyScan` は Control UI のサポートメソッドです。`historyScan` は `direction: "older" | "newer"` を受け付けますが、結果は常に保留中の提案として残します。

## ストレージ

```text
<OPENCLAW_STATE_DIR>/skill-workshop/
  proposals.json
  proposals/<proposal-id>/
    proposal.json
    PROPOSAL.md
    rollback.json
    assets/
    examples/
    references/
    scripts/
    templates/
```

デフォルトの状態ディレクトリ: `~/.openclaw`。

- `proposal.json`: 正規の提案レコード。
- `proposals.json`: 高速一覧インデックス。提案フォルダーから再構築できます。
- `PROPOSAL.md`: 保留中のスキル提案。
- `rollback.json`: 適用によって使用中のファイルが変更される前に書き込まれる復旧メタデータ。

## 制限

| 制限                            | 値                                                                   |
| ------------------------------- | -------------------------------------------------------------------- |
| 説明                            | 160 バイト                                                           |
| 提案本文                        | `skills.workshop.maxSkillBytes`（デフォルト 40,000、上限 1 MiB）                  |
| サポートファイル                | 提案ごとに 64                                                        |
| サポートファイルのサイズ        | 1 ファイルあたり 256 KiB、合計 2 MiB                                |
| 保留中 + 隔離済みの提案         | ワークスペースごとに `skills.workshop.maxPending`（デフォルト 50）             |

## トラブルシューティング

| 問題                                           | 解決方法                                                                                                                                                                                                    |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Skill proposal description is too large`                             | `description` を 160 バイト以下に短縮します。                                                                                                                                                          |
| `Skill proposal content is too large`                             | 提案本文を短縮するか、`skills.workshop.maxSkillBytes` を引き上げます。                                                                                                                                                   |
| `Target skill changed after proposal creation`                             | 現在のターゲットに合わせて提案を修正するか、新しい提案を作成します。                                                                                                                                        |
| `Proposal scan failed`                             | スキャナーの検出結果を確認してから、提案を修正または隔離します。                                                                                                                                            |
| `untrusted symlink target`                             | `skills.load.allowSymlinkTargets` を設定し、意図的に共有するスキルルートに対してのみ `skills.workshop.allowSymlinkTargetWrites` を有効にします。                                                                                                   |
| `Support file paths must be under one of...`                             | サポートファイルを `assets/`、`examples/`、`references/`、`scripts/`、または `templates/` の下に移動します。                                                            |
| 提案が一覧に表示されない                       | 選択した `--agent` ワークスペースと `OPENCLAW_STATE_DIR` を確認します。                                                                                                                             |
| エージェントが `skill_workshop` を呼び出せない | 有効なツールポリシーと実行モードを確認します。`coding` にはこのツールが含まれます。制限的な `tools.allow` ポリシーでは明示的に指定する必要があり、サンドボックス化された実行では通常のホスト側エージェントセッションまたは CLI を使用する必要があります。 |

### ツールポリシー診断

自律キャプチャが有効な場合、`openclaw doctor` はデフォルトエージェントに対して `core/doctor/skill-workshop-tool-policy` チェックを実行します。ポリシーによって `skill_workshop` が非表示になっている場合、警告には除外を行っている最初の設定レイヤーと、必要な `allow` または `alsoAllow` の正確な変更内容が示されます。古い運用手順では引き続き `openclaw plugins inspect skill-workshop` が使用されている場合があります。このコマンドは現在、Skill Workshop が組み込みであることを説明し、該当する場合は同じポリシーのヒントを表示します。

## 関連項目

- [Skills](/ja-JP/tools/skills) — 読み込み順序、優先順位、可視性について
- [自己学習](/ja-JP/tools/self-learning) — 実行後の保守的なスキル提案について
- [スキルの作成](/ja-JP/tools/creating-skills) — 手書きの `SKILL.md` の基本について
- [Skills の設定](/ja-JP/tools/skills-config) — `skills.workshop` スキーマ全体について
- [Skills CLI](/ja-JP/cli/skills) — `openclaw skills` コマンドについて
