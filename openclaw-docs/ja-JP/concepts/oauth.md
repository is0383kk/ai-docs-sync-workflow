---
read_when:
    - OpenClaw OAuth の全体像をエンドツーエンドで理解したい場合
    - トークンの無効化／ログアウトの問題が発生する
    - Claude CLI または OAuth 認証フローを使用したい場合
    - 複数のアカウントまたはプロファイルのルーティングを使用したい場合
summary: OpenClaw における OAuth：トークン交換、保存、マルチアカウントのパターン
title: OAuth
x-i18n:
    generated_at: "2026-07-26T10:12:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3ef94af0601b7d57bb7e2d53c3d8231708b401251eca7dc1bb1e7e4fc09b46da
    source_path: concepts/oauth.md
    workflow: 16
---

OpenClaw は、対応しているプロバイダー向けに OAuth（「サブスクリプション認証」）をサポートしています。
特に **OpenAI Codex（ChatGPT OAuth）** と **Anthropic Claude CLI の再利用** が該当します。
Anthropic については、実際には次のように分かれます。

- **Anthropic API キー**：通常の Anthropic API 課金。
- **OpenClaw 内での Anthropic Claude CLI／サブスクリプション認証**：Anthropic のスタッフから、この使用は再び許可されているとの回答を得たため、Anthropic が新しいポリシーを公開しない限り、OpenClaw は Claude CLI の再利用と
  `claude -p` の使用を、この連携で認められた方法として扱います。本番環境で Anthropic を使用する場合は、引き続き API キー認証のほうが安全な推奨方法です。

OpenClaw は、OpenAI API キー認証と ChatGPT/Codex OAuth の両方を、正規のプロバイダー ID
`openai` の下に保存します。古い `openai-codex:*` プロファイル ID と
`auth.order.openai-codex` エントリは、`openclaw doctor --fix` によって修復されるレガシー状態です。新しい設定には
`openai:*` プロファイル ID と `auth.order.openai` を使用してください。

このページでは、以下について説明します。

- OAuth の **トークン交換** の仕組み（PKCE）
- トークンが**保存される場所**（およびその理由）
- **複数アカウント**の扱い方（プロファイル＋セッション単位のオーバーライド）

独自の OAuth または API キーフローを提供するプロバイダー Plugin も、同じエントリポイントを使用します。

```bash
openclaw models auth login --provider <id>
```

## トークンシンク（存在する理由）

OAuth プロバイダーは一般に、ログインまたは更新のたびに新しいリフレッシュトークンを発行します。
プロバイダーによっては、同じユーザー／アプリに対して新しいリフレッシュトークンを発行すると、以前のリフレッシュトークンを無効化します。実際に現れる症状は、OpenClaw _と_
Claude Code／Codex CLI の両方でログインすると、後からどちらか一方が不規則にログアウトされることです。

これを減らすため、OpenClaw は認証プロファイルストアを**トークンシンク**として扱います。

- ランタイムは、エージェントごとに 1 か所から認証情報を読み取る
- 複数のプロファイルが共存でき、決定論的にルーティングできる
- 外部 CLI の再利用はプロバイダー固有です。OpenClaw がプロバイダーのローカル OAuth
  プロファイルを所有した後は、そのローカルリフレッシュトークンが正規のものになります。そのローカル
  リフレッシュトークンが拒否された場合、OpenClaw は外部 CLI のトークン情報へフォールバックせず、
  再認証が必要なプロファイルとして報告します。
  Codex CLI のブートストラップはさらに限定的です。OpenClaw がそのプロバイダーの OAuth を所有する前に限り、
  空の `openai:default` 形式のプロファイルへ初期情報を設定できます。それ以降は、
  OpenClaw が管理する更新が引き続き正規のものになります
- ステータス／起動処理では、外部 CLI の検出対象を、すでに設定されているプロバイダーの集合に限定します。
  そのため、単一プロバイダー構成で無関係な CLI ログインストアが調査されることはありません

## ストレージ（トークンの保存場所）

シークレットはエージェントごとに保存され、論理名 `auth-profiles.json` をキーとして使用します（基盤となるストアはエージェントの SQLite データベースです。JSON 名は互換性とツール表示のために維持されています）。

- 認証プロファイル（OAuth＋API キー＋任意の値レベル参照）：
  `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- レガシー互換ファイル：`~/.openclaw/agents/<agentId>/agent/auth.json`
  （静的な `api_key` エントリは、検出時に消去されます）

レガシーのインポート専用ファイル（引き続きサポートされていますが、メインストアではありません）：

- `~/.openclaw/credentials/oauth.json`（初回使用時に認証プロファイルストアへインポートされます）

上記のすべてで `$OPENCLAW_STATE_DIR`（状態ディレクトリのオーバーライド）も適用されます。完全なリファレンス：[/gateway/configuration-reference#auth-storage](/ja-JP/gateway/configuration-reference#auth-storage)

静的なシークレット参照とランタイムスナップショットの有効化動作については、[シークレット管理](/ja-JP/gateway/secrets)を参照してください。

セカンダリエージェントにローカル認証プロファイルがない場合、OpenClaw はデフォルト／メインエージェントのストアから読み取り時に継承します。読み取り時にメインエージェントのストアを複製することはありません。OAuth リフレッシュトークンは特に機密性が高く、一部のプロバイダーでは使用後にリフレッシュトークンがローテーションまたは無効化されるため、通常のコピー処理ではデフォルトで除外されます。エージェントに独立したアカウントが必要な場合は、そのエージェント用に別の OAuth ログインを設定してください。

## Anthropic Claude CLI の再利用

OpenClaw は、認められた認証方法として Anthropic Claude CLI の再利用と `claude -p` をサポートしています。ホスト上ですでにローカルの Claude ログインがある場合、オンボーディング／設定処理でそれを直接再利用できます。Anthropic setup-token はサポート対象のトークン認証方法として引き続き利用できますが、Claude CLI を再利用できる場合、OpenClaw はそちらを優先します。

<Warning>
Anthropic の公開 Claude Code ドキュメントには、Claude Code の直接使用には Claude サブスクリプションの制限が適用されると記載されており、Anthropic のスタッフからは、OpenClaw 形式の Claude
CLI 使用が再び許可されているとの回答を得ています。そのため OpenClaw は、Anthropic が新しいポリシーを公開しない限り、Claude CLI の再利用と
`claude -p` の使用を、この連携で認められた方法として扱います。

Anthropic の現在の Claude Code 直接利用プランに関するドキュメントについては、[Pro または Max
プランで Claude Code を使用する](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
および [Team または Enterprise
プランで Claude Code を使用する](https://support.anthropic.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan/)を参照してください。

OpenClaw でほかのサブスクリプション形式の選択肢を使用する場合は、[OpenAI
Codex](/ja-JP/providers/openai)、[Qwen Cloud Coding
Plan](/ja-JP/providers/qwen)、[MiniMax Coding Plan](/ja-JP/providers/minimax)、
および [Z.AI / GLM Coding Plan](/ja-JP/providers/zai)を参照してください。
</Warning>

## OAuth 交換（ログインの仕組み）

OpenClaw の対話型ログインフローは `openclaw/plugin-sdk/llm.ts` に実装され、ウィザード／コマンドに接続されています。

### Anthropic setup-token

フローの概要：

1. Claude Code がある任意のマシンで `claude setup-token` を実行してトークンを作成し、OpenClaw から Anthropic setup-token または paste-token を開始する
2. OpenClaw は、生成された Anthropic 認証情報を認証プロファイルに保存する
3. モデル選択は `anthropic/...` のまま維持される
4. 既存の Anthropic 認証プロファイルは、ロールバック／順序制御に引き続き使用できる

### OpenAI Codex（ChatGPT OAuth）

OpenAI Codex OAuth は、OpenClaw のワークフローを含め、Codex CLI の外部での使用が明示的にサポートされています。

ログインコマンドでは、正規の OpenAI プロバイダー ID を使用します。

```bash
openclaw models auth login --provider openai
```

1 つのエージェントで複数の ChatGPT/Codex OAuth アカウントを使用するには、`--profile-id openai:<name>` を使用します。
新しいプロファイルに `openai-codex:<name>` を使用しないでください。Doctor は、その古いプレフィックスを衝突しない
`openai:*` プロファイル ID に移行します。修復後、プロファイル ID を
`auth.order` または `/model ...@<profileId>` にコピーする前に、
`openclaw models auth list --provider openai` を実行してください。

フローの概要（PKCE）：

1. PKCE verifier／challenge とランダムな `state` を生成する
2. `https://auth.openai.com/oauth/authorize?...`（スコープ
   `openid profile email offline_access`）を開く
3. `http://localhost:1455/auth/callback` でコールバックの取得を試みる（
   コールバックホストのデフォルトは `localhost` で、ループバックホストのみを受け入れます。
   `OPENCLAW_OAUTH_CALLBACK_HOST` でオーバーライドします）
4. コールバックが到着する前にコードを貼り付けられる場合（またはリモート／ヘッドレス環境でコールバックをバインドできない場合）は、
   代わりにリダイレクト URL／コードを貼り付ける。手動貼り付けとブラウザーのコールバックは競合し、
   先に完了したほうが使用される
5. `https://auth.openai.com/oauth/token` でコードを交換する
6. アクセストークンから `accountId` を抽出し、`{ access, refresh, expires, accountId }` を保存する

ウィザードのパスは `openclaw onboard` → 認証方式 `openai` です。

## 更新＋有効期限

プロファイルには `expires` タイムスタンプが保存されます。ランタイムでは次のように動作します。

- `expires` が未来の時刻なら、保存されているアクセストークンを使用する
- 期限切れなら、ファイルロック下で更新し、保存されている認証情報を上書きする
- セカンダリエージェントが継承されたメインエージェントの OAuth プロファイルを読み取る場合、
  リフレッシュトークンをセカンダリエージェントのストアへコピーするのではなく、更新結果をメインエージェントのストアへ書き戻す
- 外部管理される CLI 認証情報（Claude CLI、および限定的な Codex CLI ブートストラップ。
  [トークンシンク](#the-token-sink-why-it-exists)を参照）は、コピーしたリフレッシュトークンを消費する代わりに
  再読み込みされます。管理対象の更新に失敗した場合、OpenClaw は外部 CLI のトークン情報を返すのではなく、
  影響を受けたプロファイルを再認証が必要なものとして報告します。

更新フローは自動です。通常、トークンを手動で管理する必要はありません。

## 複数アカウント（プロファイル）＋ルーティング

2 つのパターンがあります。

### 1) 推奨：エージェントを分ける

「個人用」と「仕事用」を一切連携させたくない場合は、分離されたエージェント（個別のセッション＋認証情報＋ワークスペース）を使用します。

```bash
openclaw agents add work
openclaw agents add personal
```

その後、エージェントごとに認証を設定し（ウィザード）、チャットを適切なエージェントへルーティングします。

### 2) 高度：1 つのエージェントで複数のプロファイルを使用する

認証プロファイルストアでは、同じプロバイダーに複数のプロファイル ID を設定できます。
使用するプロファイルは次の方法で選択します。

- 設定の順序（`auth.order`）によりグローバルに指定
- `/model ...@<profileId>` によりセッション単位で指定

例（セッションのオーバーライド）：

- `/model Opus@anthropic:work`

既存のプロファイル ID を一覧表示するには、次を実行します。

```bash
openclaw models auth list --provider <id>
```

関連ドキュメント：

- [モデルのフェイルオーバー](/ja-JP/concepts/model-failover)（ローテーション＋クールダウンのルール）
- [スラッシュコマンド](/ja-JP/tools/slash-commands)（コマンドサーフェス）

## 関連項目

- [認証](/ja-JP/gateway/authentication) - モデルプロバイダー認証の概要
- [シークレット](/ja-JP/gateway/secrets) - 認証情報の保存と SecretRef
- [設定リファレンス](/ja-JP/gateway/configuration-reference#auth-storage) - 認証設定キー
