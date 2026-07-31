---
read_when:
    - 利用可能で実行準備が整っている Skills を確認したい場合
    - ClawHub を検索するか、ClawHub、Git、またはローカルディレクトリから Skills をインストールする場合
    - ClawHub で ClawHub の Skills を検証したい場合
    - Skills のバイナリ／環境変数／設定が見つからない問題をデバッグしたい場合
summary: '`openclaw skills` の CLI リファレンス（検索/インストール/更新/検証/一覧表示/情報表示/チェック/ワークショップ）'
title: Skills
x-i18n:
    generated_at: "2026-07-26T09:31:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3eafd40704b666e6be185aa8148b60613c861a2899fb9b0cc3353212e8e4d678
    source_path: cli/skills.md
    workflow: 16
---

# `openclaw skills`

ローカルの Skills を調査し、ClawHub を検索し、ClawHub/Git/ローカルディレクトリから Skills をインストールし、ClawHub の Skills を検証し、ClawHub で追跡されているインストールを更新します。

関連項目:

- Skills システム: [Skills](/ja-JP/tools/skills)
- Skill ワークショップ: [Skill ワークショップ](/ja-JP/tools/skill-workshop)
- Skills 設定: [Skills 設定](/ja-JP/tools/skills-config)
- ClawHub のインストール: [ClawHub](/ja-JP/clawhub/cli)

## コマンド

```bash
openclaw skills search "calendar"
openclaw skills search --limit 20 --json
openclaw skills install @owner/<slug>
openclaw skills install @owner/<slug> --version <version>
openclaw skills install git:owner/repo
openclaw skills install git:owner/repo@main
openclaw skills install ./path/to/skill --as custom-name
openclaw skills install @owner/<slug> --force
openclaw skills install @owner/<slug> --force-install
openclaw skills install @owner/<slug> --acknowledge-clawhub-risk
openclaw skills install @owner/<slug> --agent <id>
openclaw skills install @owner/<slug> --global
openclaw skills update @owner/<slug>
openclaw skills update @owner/<slug> --force-install
openclaw skills update @owner/<slug> --acknowledge-clawhub-risk
openclaw skills update @owner/<slug> --global
openclaw skills update --all
openclaw skills update --all --agent <id>
openclaw skills update --all --global
openclaw skills verify @owner/<slug>
openclaw skills verify @owner/<slug> --version <version>
openclaw skills verify @owner/<slug> --tag <tag>
openclaw skills verify @owner/<slug> --card
openclaw skills verify @owner/<slug> --global
openclaw skills list
openclaw skills list --eligible
openclaw skills list --json
openclaw skills list --verbose
openclaw skills list --agent <id>
openclaw skills info <name>
openclaw skills info <name> --json
openclaw skills info <name> --agent <id>
openclaw skills check
openclaw skills check --agent <id>
openclaw skills check --json
openclaw skills workshop propose-create --name "qa-check" --description "QA checklist" --proposal ./PROPOSAL.md
openclaw skills workshop propose-update qa-check --proposal ./PROPOSAL.md
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop revise <proposal-id> --proposal ./PROPOSAL.md
openclaw skills workshop apply <proposal-id>
openclaw skills workshop reject <proposal-id> --reason "Not reusable"
openclaw skills workshop quarantine <proposal-id> --reason "Needs security review"
```

`search`、`update`、`verify` は ClawHub を直接使用します。`install @owner/<slug>`
は ClawHub の Skill をインストールし、`install git:owner/repo[@ref]` は Git の Skill をクローンし、
`install ./path` はローカルの Skill ディレクトリをコピーします。デフォルトでは、`install`、
`update`、`verify` はアクティブなワークスペースの `skills/` ディレクトリを対象とします。
`--global` を指定すると、共有の管理対象 Skills ディレクトリを対象とします。`list`/`info`/`check`
は引き続き、現在のワークスペースと設定から参照できるローカルの Skills を調査します。
ワークスペースを使用するコマンドは、まず `--agent <id>` から対象ワークスペースを解決し、
次に現在の作業ディレクトリが設定済みエージェントのワークスペース内にある場合はそのディレクトリを使用し、
最後にデフォルトのエージェントを使用します。

Git およびローカルディレクトリからのインストールでは、ソースルートに `SKILL.md` が必要です。
インストール用のスラッグには、有効な場合は `SKILL.md` の frontmatter にある `name` を使用し、
次にソースディレクトリ名またはリポジトリ名を使用します。上書きするには `--as <slug>` を使用します。
`--version` は ClawHub 専用です。Skill のインストールでは npm パッケージ仕様や
zip/アーカイブのパスはサポートされません。また、`openclaw skills update` は ClawHub で追跡されている
インストールのみを更新します。

オンボーディングまたは Skills 設定からトリガーされる、Gateway を介した Skill の依存関係のインストールでは、
代わりに別の `skills.install` リクエストパスを使用します。

注記:

| フラグ/動作                    | 説明                                                                                                                                                                                                                                                                       |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `search [query...]`              | 任意のクエリ。省略すると、デフォルトの ClawHub 検索フィードを参照します。                                                                                                                                                                                                                |
| `search --limit <n>`             | 返される結果の数を制限します。                                                                                                                                                                                                                                                            |
| `install git:owner/repo[@ref]`   | Git の Skill をインストールします。ブランチ参照には、`git:owner/repo@feature/foo` のようにスラッシュを含めることができます。                                                                                                                                                                                      |
| `install ./path/to/skill`        | ルートに `SKILL.md` が含まれるローカルディレクトリをインストールします。                                                                                                                                                                                                                        |
| `install --as <slug>`            | Git およびローカルディレクトリからのインストールで推測されたスラッグを上書きします。                                                                                                                                                                                                                 |
| `install --version <version>`    | ClawHub の Skill 参照にのみ適用されます。                                                                                                                                                                                                                                               |
| `install --force`                | 同じスラッグの既存のワークスペース Skill フォルダーを上書きします。                                                                                                                                                                                                                  |
| `install/update --force-install` | ClawHub のスキャンが完了する前に、保留中の GitHub を使用する ClawHub Skill をインストールします。                                                                                                                                                                                                   |
| `--global`                       | 共有の管理対象 Skills ディレクトリを対象とします。`--agent <id>` とは併用できません。                                                                                                                                                                                                  |
| `--agent <id>`                   | 設定済みのエージェントワークスペースを 1 つ対象とし、現在の作業ディレクトリからの推測を上書きします。                                                                                                                                                                                            |
| `update @owner/<slug>`           | 追跡されている単一の Skill を更新します。ワークスペースではなく共有の管理対象 Skills ディレクトリを対象とするには、`--global` を追加します。                                                                                                                                                            |
| `update --all`                   | 選択したワークスペース内で追跡されている ClawHub のインストールを更新します。`--global` を指定すると、共有の管理対象 Skills ディレクトリを更新します。                                                                                                                                                               |
| `verify @owner/<slug>`           | デフォルトで ClawHub の `clawhub.skill.verify.v1` JSON エンベロープを出力します。JSON がすでにデフォルトであるため、`--json` フラグはありません。Skill がすでにインストール済みか一意に特定できる場合、互換性のために所有者なしのスラッグも受け付けます。所有者で修飾された参照を使用すると、公開者の曖昧さを回避できます。 |
| `verify` の来歴              | ClawHub がサーバーで解決されたソースの来歴を返す場合、検証 JSON にコミットで固定された `openclaw.verifiedSourceUrl` も含まれます。利用できないソース URL や自己申告されたソース URL は生の来歴エンベロープ内にのみ保持され、昇格されません。                                           |
| `verify` バージョンセレクター        | `verify` はインストール済みの ClawHub Skills に `.clawhub/origin.json` を使用するため、インストール済みバージョンを取得元のレジストリと照合して検証します。`--version` と `--tag` はバージョンセレクターを上書きしますが、オリジンメタデータが存在する場合は、そのインストール元レジストリを維持します。                    |
| `verify --card`                  | JSON の代わりに生成された Skill Card の Markdown を出力します。ClawHub が `ok: false` または `decision: "fail"` を返した場合、0 以外の終了コードで終了します。署名なしの署名は、ClawHub のポリシーが変更されない限り参考情報として扱われます。                                                                             |
| Skill Card のフィンガープリント           | インストール済みの ClawHub バンドルには、生成された `skill-card.md` が含まれる場合があります。OpenClaw は検証を ClawHub サーバーの判断として扱い、その生成済みカードによってバンドルのフィンガープリントが変化したという理由だけで、インストール済みの Skill を拒否することはありません。                                              |
| `check --agent <id>`             | 選択したエージェントのワークスペースを確認し、準備完了の Skills のうち、そのエージェントのプロンプトまたはコマンドサーフェスから実際に参照できるものを報告します。                                                                                                                                              |
| `list`                           | サブコマンドが指定されていない場合のデフォルトアクションです。                                                                                                                                                                                                                                    |
| `list`/`info`/`check` の出力     | レンダリングされた出力は標準出力に送られます。`--json` を指定すると、パイプやスクリプト用の機械可読ペイロードは標準出力に維持されます。                                                                                                                                                                |

コミュニティの ClawHub Skill のインストールと更新では、ダウンロード前に信頼性を確認します。
バージョン付きのコミュニティアーカイブリリースでは、正確なリリースの信頼メタデータを使用します。
リゾルバーを使用する GitHub Skills は、ClawHub のインストールリゾルバーに依存して、
固定されたコミットを返す前にスキャンおよび強制インストールのポリシーを適用します。
そのスキャンが完了する前に保留中の GitHub を使用する Skill をインストールするには、
`--force-install` を使用します。悪意のある、またはブロックされたコミュニティリリースは拒否されます。
リスクのあるコミュニティリリースではレビューが必要であり、そのレビュー後に
非対話型コマンドを続行するには `--acknowledge-clawhub-risk` が必要です。ClawHub の公式
Skill 公開者と OpenClaw にバンドルされた Skill ソースでは、このリリース信頼プロンプトを省略します。

## Skill ワークショップ

`openclaw skills workshop` は、選択したワークスペース内の保留中の Skills 提案を管理します。提案は適用されるまで有効な Skills ではありません。提案の保存、サポートファイルの保護策、Gateway メソッド、承認ポリシーについては、[Skill Workshop](/ja-JP/tools/skill-workshop)を参照してください。

```bash
openclaw skills workshop propose-create \
  --name "qa-check" \
  --description "再利用可能な QA チェックリスト" \
  --proposal ./PROPOSAL.md
openclaw skills workshop propose-create \
  --name "qa-check" \
  --description "再利用可能な QA チェックリスト" \
  --proposal-dir ./qa-check-proposal
openclaw skills workshop propose-update qa-check --proposal ./PROPOSAL.md
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop revise <proposal-id> --proposal ./PROPOSAL.md
openclaw skills workshop apply <proposal-id>
openclaw skills workshop reject <proposal-id> --reason "重複"
openclaw skills workshop quarantine <proposal-id> --reason "セキュリティレビューが必要"
```

`propose-create`、`propose-update`、`revise` では、`--goal <text>` と `--evidence <text>` も指定でき、提案の動機と補足事項を `--proposal`/`--proposal-dir` の内容とともに記録できます。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [Skills](/ja-JP/tools/skills)
