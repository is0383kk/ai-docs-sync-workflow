---
doc-schema-version: 1
read_when:
    - OpenClaw が提供するツールについて理解したい場合
    - 組み込みツール、Skills、プラグインのどれを使用するか判断しています
    - ツールポリシー、自動化、またはエージェント連携に適したドキュメントの入口が必要です
summary: OpenClaw のツール、Skills、プラグインの概要：エージェントが呼び出せる機能とその拡張方法
title: 概要
x-i18n:
    generated_at: "2026-07-26T09:22:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 45745bd5f2008a84cb6c4c1c9840073bfa8a9c40a0ff65bfefc682c5d99af09b
    source_path: tools/index.md
    workflow: 16
---

このページでは、適切なケイパビリティサーフェスを選択できます。**ツール**は
呼び出し可能なアクション、**Skills**はエージェントに作業方法を教えるものであり、**Plugin**は
ツール、プロバイダー、チャネル、フック、パッケージ化された
Skillsなどのランタイムケイパビリティを追加します。

これは概要および案内ページです。ツールポリシー、デフォルト、
グループメンバーシップ、プロバイダー制限、設定フィールドの詳細については、
[ツールとカスタムプロバイダー](/ja-JP/gateway/config-tools)を参照してください。

## ここから始める

ほとんどのエージェントでは、組み込みツールカテゴリから始め、エージェントに表示する
ツールを減らす必要がある場合や、明示的なホストアクセスが必要な場合にのみポリシーを調整します。

| 必要なこと                                           | 最初に使用するもの                                      | 次に読むもの                                                                                                                                                 |
| ---------------------------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 既存のケイパビリティを使用してエージェントを動作させる | [組み込みツール](#built-in-tool-categories)             | [ツールカテゴリ](#built-in-tool-categories)                                                                                                                  |
| エージェントが呼び出せるものを制御する                 | [ツールポリシー](#configure-access-and-approvals)        | [ツールとカスタムプロバイダー](/ja-JP/gateway/config-tools)                                                                                                        |
| エージェントにワークフローを教える                     | [Skills](#choose-tools-skills-or-plugins)                | [Skills](/ja-JP/tools/skills)、[Skillsの作成](/ja-JP/tools/creating-skills)、[Skillsワークショップ](/ja-JP/tools/skill-workshop)、[自己学習](/ja-JP/tools/self-learning) |
| 新しい統合またはランタイムサーフェスを追加する         | [Plugin](#extend-capabilities)                           | [Plugin](/ja-JP/tools/plugin)と[Pluginの構築](/ja-JP/plugins/building-plugins)                                                                                            |
| 後で、またはバックグラウンドで作業を実行する           | [自動化](/ja-JP/automation)                                    | [自動化の概要](/ja-JP/automation)                                                                                                                                  |
| 複数のエージェントまたはハーネスを連携させる           | [サブエージェント](/ja-JP/tools/subagents)                     | [ACPエージェント](/ja-JP/tools/acp-agents)と[エージェント送信](/ja-JP/tools/agent-send)                                                                                   |
| コードから並行エージェントをオーケストレーションする   | [Swarm](/tools/swarm)                                    | [コードモード](/ja-JP/tools/code-mode)と[サブエージェント](/ja-JP/tools/subagents)                                                                                        |
| 大規模なOpenClawツールカタログを検索する               | [ツール検索](/ja-JP/tools/tool-search)                         | [ツール検索](/ja-JP/tools/tool-search)                                                                                                                             |
| 複数のツールを1つのコンパクトなプログラムにまとめる    | [コードモード](/ja-JP/tools/code-mode)                         | [コードモード](/ja-JP/tools/code-mode)                                                                                                                             |

## ツール、Skills、Pluginの選択

<Steps>
  <Step title="エージェントがアクションを実行する必要がある場合はツールを使用する">
    ツールは、`exec`、`browser`、
    `web_search`、`message`、`image_generate`など、エージェントが呼び出せる型付き関数です。エージェントが
    データを読み取り、ファイルを変更し、メッセージを送信し、プロバイダーを呼び出し、または
    別のシステムを操作する必要がある場合はツールを使用します。表示されるツールは、構造化された
    関数定義としてモデルに送信されます。

    モデルに表示されるのは、アクティブなプロファイル、許可/拒否
    ポリシー、プロバイダー制限、サンドボックスの状態、チャネル権限、および
    Pluginの可用性による条件を満たしたツールだけです。

  </Step>

  <Step title="エージェントに指示が必要な場合はSkillsを使用する">
    Skillsは、エージェントのプロンプトに読み込まれる`SKILL.md`指示パックです。
    エージェントが必要なツールをすでに備えているものの、反復可能な
    ワークフロー、レビュー基準、コマンドシーケンス、または運用上の
    制約が必要な場合はSkillsを使用します。

    Skillsは、ワークスペース、共有Skillsディレクトリ、管理対象のOpenClaw
    Skillsルート、またはPluginパッケージに配置できます。

    [Skills](/ja-JP/tools/skills) | [Skillsワークショップ](/ja-JP/tools/skill-workshop) | [自己学習](/ja-JP/tools/self-learning) | [Skillsの作成](/ja-JP/tools/creating-skills) | [Skills設定](/ja-JP/tools/skills-config)

  </Step>

  <Step title="OpenClawに新しいケイパビリティが必要な場合はPluginを使用する">
    Pluginは、ツール、Skills、チャネル、モデルプロバイダー、音声、
    リアルタイム音声、メディア生成、ウェブ検索、ウェブ取得、フック、その他の
    ランタイムケイパビリティを追加できます。ケイパビリティにコード、
    認証情報、ライフサイクルフック、マニフェストメタデータ、またはインストール可能な
    パッケージが含まれる場合はPluginを使用します。既存のPluginは、ClawHub、npm、git、
    ローカルディレクトリ、またはアーカイブからインストールできます。

    [Pluginのインストールと設定](/ja-JP/tools/plugin) | [Pluginの構築](/ja-JP/plugins/building-plugins) | [Plugin SDK](/ja-JP/plugins/sdk-overview)

  </Step>
</Steps>

## 組み込みツールカテゴリ

この表には、サーフェスを識別できるように代表的なツールを示しています。これは
完全なポリシーリファレンスではありません。正確なグループ、デフォルト、および許可/拒否の
セマンティクスについては、[ツールとカスタムプロバイダー](/ja-JP/gateway/config-tools)を参照してください。

| カテゴリ                 | エージェントが必要とすること                                                                          | 代表的なツール                                                                                                        | 次に読むもの                                                                                                                |
| ------------------------ | ----------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| ランタイム               | コマンドの実行、プロセスの管理、またはプロバイダー対応のPython分析の使用                              | `exec`、`process`、`terminal`、`code_execution`                                       | [Exec](/ja-JP/tools/exec)、[Control UIターミナル](/ja-JP/web/control-ui#operator-terminal)、[コード実行](/ja-JP/tools/code-execution)          |
| ファイル                 | ワークスペースファイルの読み取りと変更                                                                | `read`、`write`、`edit`、`apply_patch`                                       | [パッチの適用](/ja-JP/tools/apply-patch)                                                                                           |
| 人による入力             | ユーザーが決定権を持つ構造化された判断のために一時停止する                                            | `ask_user`                                                                                                    | [ユーザーに質問](/tools/ask-user)                                                                                            |
| ウェブ                   | ウェブの検索、X投稿の検索、または読み取り可能なページコンテンツの取得                                 | `web_search`、`x_search`、`web_fetch`                                                           | [ウェブツール](/ja-JP/tools/web)、[ウェブ取得](/ja-JP/tools/web-fetch)                                                                   |
| ブラウザー               | ブラウザーセッションの操作                                                                            | `browser`                                                                                                    | [ブラウザー](/ja-JP/tools/browser)                                                                                                 |
| オペレーターUI           | 接続されたControl UIのペイン、パネル、ナビゲーションの配置                                           | `screen`                                                                                                    | [画面](/tools/screen)                                                                                                       |
| メッセージングとチャネル | 返信またはチャネルアクションの送信                                                                    | `message`                                                                                                    | [エージェント送信](/ja-JP/tools/agent-send)                                                                                        |
| セッションとエージェント | セッションの検査、作業の委任、コレクターのオーケストレーション、別の実行の誘導、またはステータスの報告 | `sessions_*`、`agents_wait`、`subagents`、`agents_list`、`session_status`、`get_goal`、`create_goal`、`update_goal` | [目標](/ja-JP/tools/goal)、[Swarm](/tools/swarm)、[サブエージェント](/ja-JP/tools/subagents)、[セッションツール](/ja-JP/concepts/session-tool) |
| 自動化                   | 作業のスケジュール設定またはバックグラウンドイベントへの応答                                          | `cron`、`heartbeat_respond`                                                                                | [自動化](/ja-JP/automation)                                                                                                       |
| GatewayとNode            | Gatewayの状態またはペアリングされた対象デバイスの検査                                                 | `gateway`、`nodes`                                                                                | [Gateway設定](/ja-JP/gateway/configuration)、[Node](/ja-JP/nodes)                                                                        |
| メディア                 | メディアの分析、生成、または音声化                                                                     | `image`、`image_generate`、`music_generate`、`video_generate`、`tts`                   | [メディアの概要](/ja-JP/tools/media-overview)                                                                                      |
| 大規模なOpenClawカタログ | すべてのスキーマをモデルに送信することなく、多数の利用可能なツールを検索、呼び出し、組み合わせる       | `exec`、`wait`、`tool_search_code`、`tool_search`、`tool_describe`                   | [コードモード](/ja-JP/tools/code-mode)、[ツール検索](/ja-JP/tools/tool-search)                                                           |

<Note>
コードモードとツール検索は、実験的なOpenClawエージェントサーフェスです。Codex
ハーネスの実行では、`tools.codeMode`または`tools.toolSearch`の代わりに、Codexネイティブのコードモード、ネイティブツール検索、遅延動的
ツール、およびネストされたツール呼び出しを使用します。
</Note>

## Pluginが提供するツール

Pluginは追加のツールを登録できます。Plugin作成者は、
`api.registerTool(...)`およびマニフェストの`contracts.tools`を介してツールを接続します。契約の詳細については、
[Plugin SDK](/ja-JP/plugins/sdk-overview)および[Pluginマニフェスト](/ja-JP/plugins/manifest)
を参照してください。

Pluginが提供する一般的なツールには、次のものがあります。

- ファイルおよび Markdown の差分をレンダリングするための [Diffs](/ja-JP/tools/diffs)
- 対応するチャットクライアントで自己完結型のインライン SVG および HTML を表示するための [Show widget](/ja-JP/tools/show-widget)
- 接続された Control UI を配置するための [Screen](/tools/screen)
- JSON のみのワークフローステップのための [LLM Task](/ja-JP/tools/llm-task)
- 再開可能な承認を備えた型付きワークフローのための [Lobster](/ja-JP/tools/lobster)
- ノイズの多い `exec` および `bash` ツールの
  出力を圧縮するための [Tokenjuice](/ja-JP/tools/tokenjuice)
- すべてのスキーマをプロンプトに含めずに、大規模なツールカタログを
  検出して呼び出すための [Tool Search](/ja-JP/tools/tool-search)
- Node の Canvas 制御と A2UI
  レンダリングのための [Canvas](/ja-JP/plugins/reference/canvas)

## アクセスと承認を設定する

ツールポリシーはモデルの呼び出し前に適用されます。ポリシーによってツールが除外された場合、
そのターンではモデルにそのツールのスキーマが提供されません。グローバル設定、エージェントごとの設定、
チャネルポリシー、プロバイダーの制限、サンドボックスルール、チャネル／ランタイムポリシー、
または Plugin の可用性により、実行時にツールが使用できなくなることがあります。

- [ツールとカスタムプロバイダー](/ja-JP/gateway/config-tools)では、ツールプロファイル、
  許可／拒否リスト、プロバイダー固有の制限、ループ検出、および
  プロバイダーを利用するツール設定について説明します。
- [Exec の承認](/ja-JP/tools/exec-approvals)では、ホストコマンドの承認
  ポリシーについて説明します。
- [権限昇格 Exec](/ja-JP/tools/elevated)では、サンドボックス外での制御された実行に
  ついて説明します。
- [サンドボックスとツールポリシーと権限昇格の比較](/ja-JP/gateway/sandbox-vs-tool-policy-vs-elevated)
  では、ファイルおよびプロセスへのアクセスをどのレイヤーが制御するかを説明します。
- [エージェントごとのサンドボックスとツール制限](/ja-JP/tools/multi-agent-sandbox-tools)
  では、委任された実行に対するエージェント固有の制限について説明します。

## 機能を拡張する

OpenClaw に実行させるジョブに応じて、拡張方法を選択します。

- 既存の Plugin を [Plugins](/ja-JP/tools/plugin) でインストールまたは管理します。
- 新しいインテグレーション、プロバイダー、チャネル、ツール、またはフックを
  [Plugin の構築](/ja-JP/plugins/building-plugins)で作成します。
- [Skills](/ja-JP/tools/skills) と
  [Skills の作成](/ja-JP/tools/creating-skills)を使用して、再利用可能なエージェント指示を追加または調整します。
- 実装契約が必要な場合は、[Plugin SDK](/ja-JP/plugins/sdk-overview) と
  [Plugin マニフェスト](/ja-JP/plugins/manifest)を使用します。

## 見つからないツールをトラブルシューティングする

モデルがツールを認識または呼び出せない場合は、まず現在のターンに適用されている
ポリシーを確認します。

1. [ツールとカスタムプロバイダー](/ja-JP/gateway/config-tools)で、アクティブなプロファイル、
   `tools.allow`、および `tools.deny` を確認します。
2. [ツールとカスタムプロバイダー](/ja-JP/gateway/config-tools)でプロバイダー固有の制限を
   確認し、選択した[モデルプロバイダー](/ja-JP/concepts/model-providers)がそのツール形式を
   サポートしていることを確認します。
3. [サンドボックスとツールポリシーと権限昇格の比較](/ja-JP/gateway/sandbox-vs-tool-policy-vs-elevated)
   および[権限昇格 Exec](/ja-JP/tools/elevated)で、チャネル権限、サンドボックスの状態、
   および権限昇格アクセスを確認します。
4. [Plugins](/ja-JP/tools/plugin)で、所有元の Plugin がインストールされ、
   有効になっているかを確認します。
5. 委任された実行では、
   [エージェントごとのサンドボックスとツール制限](/ja-JP/tools/multi-agent-sandbox-tools)でエージェントごとの制限を確認します。
6. 大規模な OpenClaw カタログでは、実行がツールの直接公開、
   [Code Mode](/ja-JP/tools/code-mode)、または [Tool Search](/ja-JP/tools/tool-search) のいずれを使用しているか確認します。

## 関連項目

- Cron、タスク、Heartbeat、フック、
  常設指示、Task Flow については[自動化](/ja-JP/automation)
- エージェントモデル、セッション、メモリ、および
  マルチエージェント連携については[エージェント](/ja-JP/concepts/agent)
- 正式なツールポリシーのリファレンスについては
  [ツールとカスタムプロバイダー](/ja-JP/gateway/config-tools)
- Plugin のインストールと管理については [Plugins](/ja-JP/tools/plugin)
- Plugin 作成者向けリファレンスについては [Plugin SDK](/ja-JP/plugins/sdk-overview)
- Skills の読み込み順序、ゲーティング、および設定については [Skills](/ja-JP/tools/skills)
- 生成およびレビューを経た Skills の作成については
  [Skills ワークショップ](/ja-JP/tools/skill-workshop)
- OpenClaw のツールカタログをコンパクトに検出する方法については
  [Tool Search](/ja-JP/tools/tool-search)
- 非表示の OpenClaw ツールカタログを利用する、コンパクトな JavaScript または TypeScript ワークフローについては
  [Code Mode](/ja-JP/tools/code-mode)
- Code Mode からの構造化されたファンアウトと収集については [Swarm](/tools/swarm)
