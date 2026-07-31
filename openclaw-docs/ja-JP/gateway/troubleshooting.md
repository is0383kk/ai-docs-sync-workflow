---
read_when:
    - トラブルシューティングハブから、より詳細な診断のためにこちらへ案内されました
    - 正確なコマンドを含む、症状別の安定したランブックセクションが必要です
sidebarTitle: Troubleshooting
summary: Gateway、チャンネル、自動化、Node、ブラウザの詳細なトラブルシューティング手順書
title: トラブルシューティング
x-i18n:
    generated_at: "2026-07-26T09:05:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c4bb1e061dbf2767118c24ad1ca2d2d1f7eeeff88e18ed0e6111aebe1cc99a26
    source_path: gateway/troubleshooting.md
    workflow: 16
---

これは詳細なランブックです。まず迅速なトリアージ手順として、[/help/troubleshooting](/ja-JP/help/troubleshooting) から始めてください。

## コマンドの実行順序

次の順序で実行します。

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

正常性を示すシグナル：

- `openclaw gateway status` に `Runtime: running`、`Connectivity probe: ok`、および `Capability: ...` の行が表示される。
- `openclaw doctor` で、処理を妨げる設定やサービスの問題が報告されない。
- `openclaw channels status --probe` にアカウントごとのリアルタイムなトランスポート状態が表示され、サポートされている場合は `works` または `audit ok` も表示される。

## アップデート後

アップデートが完了したものの、Gateway が停止している、チャンネルが空である、またはモデル呼び出しが 401 で失敗する場合に使用します。

```bash
openclaw status --all
openclaw update status --json
openclaw gateway status --deep
openclaw doctor --fix
openclaw gateway restart
```

確認する項目：

- `openclaw status` / `openclaw status --all` 内の `Update restart`。保留中または失敗した引き継ぎには、次に実行するコマンドが含まれます。
- Channels 配下の `plugin load failed: dependency tree corrupted; run openclaw doctor --fix`：チャンネル設定はまだ存在しますが、チャンネルを読み込む前に Plugin の登録が失敗しています。
- 再認証後のプロバイダー 401：`openclaw doctor --fix` は、古くなったエージェントごとの OAuth 認証シャドウを確認して古いコピーを削除し、すべてのエージェントが現在の共有プロファイルを解決できるようにします。

## インストールの分裂状態と新しい設定のガード

アップデート後に Gateway サービスが予期せず停止した場合、またはログで、ある `openclaw` バイナリが最後に `openclaw.json` を書き込んだバージョンより古いことが示される場合に使用します。

OpenClaw は設定の書き込み時に `meta.lastTouchedVersion` を記録します。読み取り専用コマンドでは新しい OpenClaw が書き込んだ設定を検査できますが、古いバイナリからのプロセスやサービスの変更操作は実行を拒否されます。ブロックされる操作：Gateway サービスの開始、停止、再起動、アンインストール、サービスの強制再インストール、サービスモードでの Gateway 起動、および `gateway --force` ポートのクリーンアップ。

```bash
which openclaw
openclaw --version
openclaw gateway status --deep
openclaw config get meta.lastTouchedVersion
```

<Steps>
  <Step title="PATH を修正">
    `openclaw` が新しいインストールを指すように `PATH` を修正してから、操作を再実行します。
  </Step>
  <Step title="Gateway サービスを再インストール">
    新しいインストールから目的の Gateway サービスを再インストールします。

    ```bash
    openclaw gateway install --force
    openclaw gateway restart
    ```

  </Step>
  <Step title="古いラッパーを削除">
    古い `openclaw` バイナリを引き続き参照している、古いシステムパッケージまたはラッパーのエントリを削除します。
  </Step>
</Steps>

<Warning>
意図的なダウングレードまたは緊急復旧の場合に限り、単一のコマンドに対して `OPENCLAW_ALLOW_OLDER_BINARY_DESTRUCTIVE_ACTIONS=1` を設定します。通常の運用では未設定のままにしてください。
</Warning>

## ロールバック後のプロトコル不一致

ダウングレードまたはロールバック後も、ログに `protocol mismatch` が繰り返し出力される場合に使用します。古い Gateway が実行されていますが、新しいローカルクライアントプロセスが、古い Gateway では扱えないプロトコル範囲を使用して再接続を続けています。

```bash
openclaw --version
which -a openclaw
openclaw gateway status --deep
openclaw doctor --deep
openclaw logs --follow
```

確認する項目：

- Gateway ログ内の `protocol mismatch ... client=... v<version> min=<n> max=<n> expected=<n>`。
- `openclaw gateway status --deep` 内の `Established clients:`、または `openclaw doctor --deep` 内の `Gateway clients`：Gateway ポートに接続しているアクティブな TCP クライアント。OS が許可する場合は PID とコマンドラインも表示されます。
- コマンドラインが、ロールバック元の新しい OpenClaw インストールまたはラッパーを指しているクライアントプロセス。

修正方法：

1. `gateway status --deep` に表示された古い OpenClaw クライアントプロセスを停止または再起動します。
2. OpenClaw を組み込んでいるアプリまたはラッパー（ローカルダッシュボード、エディター、アプリサーバーヘルパー、長時間実行中の `openclaw logs --follow` シェル）を再起動します。
3. `openclaw gateway status --deep` または `openclaw doctor --deep` を再実行し、古いクライアント PID がなくなったことを確認します。

古い Gateway が新しい非互換プロトコルを受け入れるようにはしないでください。プロトコルのバージョン更新は通信規約を保護するものです。ロールバックからの復旧は、プロセスとバージョンのクリーンアップで対処する問題です。

## パスエスケープとしてスキップされるスキルのシンボリックリンク

ログに次の内容が含まれる場合に使用します。

```text
設定されたルートの外部にエスケープしたスキルパスをスキップしています：... reason=symlink-escape
```

各スキルルートは包含境界です。`~/.agents/skills`、`<workspace>/.agents/skills`、`<workspace>/skills`、または `~/.openclaw/skills` 配下のシンボリックリンクは、実体の参照先がそのルートの外部に解決される場合、参照先が明示的に信頼されていない限りスキップされます。

リンクを調べます。

```bash
ls -l ~/.agents/skills/<name>
realpath ~/.agents/skills/<name>
openclaw config get skills.load
```

参照先が意図したものである場合は、スキルの直接ルートと許可するシンボリックリンク先の両方を設定します。

```json5
{
  skills: {
    load: {
      extraDirs: ["~/Projects/manager/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
  },
}
```

その後、新しいセッションを開始するか、Skills ウォッチャーが更新されるまで待ちます。実行中のプロセスが設定変更より前に起動されている場合は、Gateway を再起動します。

`~`、`/`、または同期対象のプロジェクトフォルダー全体のような広範な参照先は使用しないでください。`allowSymlinkTargets` は、信頼済みの `SKILL.md` ディレクトリを含む実際のスキルルートに限定してください。

Skill Workshop の適用時に、信頼済みのシンボリックリンクされたワークスペースのスキルパスにも書き込む必要がある場合は、`skills.workshop.allowSymlinkTargetWrites` を有効にします。読み取り専用の共有スキルルートでは無効のままにしてください。

関連項目：

- [Skills の設定](/ja-JP/tools/skills-config#symlinked-skill-roots)
- [設定例](/ja-JP/gateway/configuration-examples#symlinked-sibling-skill-repo)

## 長いコンテキストで Anthropic 429 の追加使用権限が必要

ログまたはエラーに `HTTP 429: rate_limit_error: Extra usage is required for long context requests` が含まれる場合に使用します。

```bash
openclaw logs --follow
openclaw models status
openclaw config get agents.defaults.models
```

確認する項目：

- 選択された Anthropic モデルが一般提供されている 1M 対応の Claude 4.x モデル（Opus 4.6/4.7/4.8、Sonnet 4.6）である、またはモデル設定に古い `params.context1m: true` がまだ含まれている。
- 現在の Anthropic 認証情報には、長いコンテキストを使用する資格がない。
- 1M コンテキストの経路を必要とする長いセッションまたはモデル実行でのみ、リクエストが失敗する。

修正オプション：

<Steps>
  <Step title="標準コンテキストウィンドウを使用">
    標準ウィンドウのモデルに切り替えるか、1M コンテキストが一般提供されていない古い
    モデル設定から `context1m` を削除します。
  </Step>
  <Step title="使用資格のある認証情報を使用">
    長いコンテキストのリクエストを使用できる Anthropic 認証情報を使用するか、Anthropic API キーに切り替えます。
  </Step>
  <Step title="フォールバックモデルを設定">
    Anthropic の長いコンテキストのリクエストが拒否された場合でも実行を継続できるように、フォールバックモデルを設定します。
  </Step>
</Steps>

関連項目：

- [Anthropic](/ja-JP/providers/anthropic)
- [トークンの使用量とコスト](/ja-JP/reference/token-use)
- [Anthropic から HTTP 429 が返されるのはなぜですか？](/ja-JP/help/faq-first-run#why-am-i-seeing-http-429-ratelimiterror-from-anthropic)

## アップストリームからの 403 ブロック応答

アップストリームの LLM プロバイダーが、`Your request was blocked` などの一般的な `403` を返す場合に使用します。

これが常に OpenClaw の設定問題であると想定しないでください。この応答は、OpenAI 互換エンドポイントの前段にある CDN、WAF、ボット管理ルール、リバースプロキシなど、アップストリームのセキュリティレイヤーから返される場合があります。

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
```

確認する項目：

- 同じプロバイダー配下の複数のモデルが同じように失敗する。
- 通常のプロバイダー API エラーではなく、HTML または一般的なセキュリティメッセージが返される。
- 同じリクエスト時刻に対応するプロバイダー側のセキュリティイベント。
- 通常の SDK 形式のリクエストが失敗する一方で、ごく小さな直接 `curl` プローブは成功する。

証拠が WAF/CDN によるブロックを示している場合は、まずプロバイダー側のフィルタリングを修正します。OpenClaw が使用する API パスに限定した許可ルールまたはスキップルールを推奨します。サイト全体の保護を無効にすることは避けてください。

<Warning>
最小限の `curl` が成功しても、実際の SDK 形式のリクエストが同じアップストリームのセキュリティレイヤーを通過できるとは限りません。
</Warning>

関連項目：

- [OpenAI 互換エンドポイント](/ja-JP/gateway/configuration-reference#openai-compatible-endpoints)
- [プロバイダー設定](/ja-JP/providers)
- [ログ](/ja-JP/logging)

## ローカルの OpenAI 互換バックエンドでは直接プローブが成功するが、エージェント実行は失敗する

次の場合に使用します。

- `curl ... /v1/models` が動作する。
- ごく小さな直接 `/v1/chat/completions` 呼び出しが動作する。
- OpenClaw のモデル実行が、通常のエージェントターンでのみ失敗する。

```bash
curl http://127.0.0.1:1234/v1/models
curl http://127.0.0.1:1234/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"<id>","messages":[{"role":"user","content":"hi"}],"stream":false}'
openclaw infer model run --model <provider/model> --prompt "hi" --json
openclaw logs --follow
```

確認する項目：

- ごく小さな直接呼び出しは成功するが、OpenClaw の実行は大きなプロンプトでのみ失敗する。
- 同じ修飾なしモデル ID を使用した直接 `/v1/chat/completions` は動作するにもかかわらず、`model_not_found` または 404 エラーが発生する。
- `messages[].content` に文字列が必要であることを示すバックエンドエラー。
- OpenAI 互換のローカルバックエンドで断続的に発生する `incomplete turn detected ... stopReason=stop payloads=0` 警告。
- プロンプトトークン数が多い場合、または完全なエージェントランタイムプロンプトの場合にのみ発生するバックエンドのクラッシュ。

<AccordionGroup>
  <Accordion title="一般的な症状">
    - ローカルの MLX/vLLM 形式サーバーでの `model_not_found`：`baseUrl` に `/v1` が含まれていること、`/v1/chat/completions` バックエンドでは `api` が `"openai-completions"` であること、`models.providers.<provider>.models[].id` が修飾なしのプロバイダーローカル ID であることを確認します。選択時にはプロバイダー接頭辞を一度だけ付けます（例：`mlx/mlx-community/Qwen3-30B-A3B-6bit`）。カタログエントリは `mlx-community/Qwen3-30B-A3B-6bit` のままにします。
    - `messages[...].content: invalid type: sequence, expected a string`：バックエンドが構造化された Chat Completions のコンテンツ部分を拒否しています。修正方法：`models.providers.<provider>.models[].compat.requiresStringContent: true` を設定します。
    - `validation.keys`、または `["role","content"]` のような許可対象のメッセージキー：バックエンドが Chat Completions メッセージ上の OpenAI 形式のリプレイメタデータを拒否しています。修正方法：`models.providers.<provider>.models[].compat.strictMessageKeys: true` を設定します。
    - `incomplete turn detected ... stopReason=stop payloads=0`：バックエンドは Chat Completions リクエストを完了しましたが、そのターンでユーザーに表示できるアシスタントのテキストを返しませんでした。OpenClaw は、リプレイしても安全な空の OpenAI 互換ターンを一度再試行します。失敗が続く場合、通常はバックエンドが空またはテキスト以外のコンテンツを出力しているか、最終回答のテキストを抑制しています。
    - ごく小さな直接リクエストは成功するが、OpenClaw のエージェント実行ではバックエンドまたはモデルがクラッシュする場合（例：一部の `inferrs` ビルド上の Gemma）：OpenClaw のトランスポートはすでに正しい可能性が高く、バックエンドがより大きなエージェントランタイムプロンプトの形式で失敗しています。
    - ツールを無効にすると失敗が減るものの解消しない場合：ツールスキーマも負荷の一部でしたが、残る問題は依然としてアップストリームのモデルまたはサーバーの容量、あるいはバックエンドのバグです。

  </Accordion>
  <Accordion title="修正オプション">
    1. 文字列のみを受け付ける Chat Completions バックエンドでは、`compat.requiresStringContent: true` を設定します。
    2. 各メッセージで `role` と `content` のみを受け付ける厳格な Chat Completions バックエンドでは、`compat.strictMessageKeys: true` を設定します。
    3. OpenClaw のツールスキーマの範囲を安定して処理できないモデルまたはバックエンドでは、`compat.supportsTools: false` を設定します。
    4. 可能な範囲でプロンプトの負荷を下げます。ワークスペースのブートストラップを小さくする、セッション履歴を短くする、より軽量なローカルモデルを使用する、または長いコンテキストをより強力にサポートするバックエンドを使用します。
    5. ごく小さな直接リクエストは引き続き成功する一方で、OpenClaw のエージェントターンがバックエンド内でなおクラッシュする場合は、アップストリームのサーバーまたはモデルの制限として扱い、受け入れられたペイロード形式を添えてそちらに再現報告を提出します。
  </Accordion>
</AccordionGroup>

関連項目：

- [設定](/ja-JP/gateway/configuration)
- [ローカルモデル](/ja-JP/gateway/local-models)
- [OpenAI 互換エンドポイント](/ja-JP/gateway/configuration-reference#openai-compatible-endpoints)

## 応答がない

チャネルが稼働しているのに何も応答しない場合は、再接続する前にルーティングとポリシーを確認します。

```bash
openclaw status
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw config get channels
openclaw logs --follow
```

確認項目:

- DM 送信者のペアリングが保留中。
- グループのメンション制限（`requireMention`、`mentionPatterns`）。
- チャネルまたはグループの許可リストの不一致。

一般的なシグネチャ:

- `drop guild message (mention required` → メンションされるまでグループメッセージは無視されます。
- `pairing request` → 送信者の承認が必要です。
- `blocked` / `allowlist` → 送信者またはチャネルがポリシーによって除外されました。

関連項目:

- [チャネルのトラブルシューティング](/ja-JP/channels/troubleshooting)
- [グループ](/ja-JP/channels/groups)
- [ペアリング](/ja-JP/channels/pairing)

## ダッシュボードのコントロール UI 接続

ダッシュボードまたはコントロール UI が接続できない場合は、URL、認証モード、およびセキュアコンテキストに関する前提を検証します。

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --json
```

確認項目:

- 正しいプローブ URL とダッシュボード URL。
- クライアントと Gateway 間の認証モードまたはトークンの不一致。
- デバイス ID が必要な箇所での HTTP 使用。

更新後にローカルブラウザーが `127.0.0.1:18789` に接続できない場合は、まずローカル Gateway サービスを復旧し、ダッシュボードを提供していることを確認します。

```bash
openclaw gateway restart
lsof -i :18789
curl http://127.0.0.1:18789
```

`curl` が OpenClaw の HTML を返す場合、Gateway は動作しており、残る問題はブラウザーキャッシュ、古いディープリンク、または古いタブ状態である可能性があります。`http://127.0.0.1:18789` を直接開き、ダッシュボードから移動してください。再起動後もサービスが稼働し続けない場合は、`openclaw gateway start` を実行し、`openclaw gateway status` を再確認します。

<AccordionGroup>
  <Accordion title="接続 / 認証のシグネチャ">
    - `device identity required` → 非セキュアコンテキストまたはデバイス認証の欠如。
    - `origin not allowed` → ブラウザーの `Origin` が `gateway.controlUi.allowedOrigins` に含まれていません（または、明示的な許可リストを設定せずに非ループバックのブラウザーオリジンから接続しています）。
    - `device nonce required` / `device nonce mismatch` → クライアントがチャレンジベースのデバイス認証フロー（`connect.challenge` + `device.nonce`）を完了していません。
    - `device signature invalid` / `device signature expired` → クライアントが現在のハンドシェイクに対して誤ったペイロード（または古いタイムスタンプ）に署名しました。
    - `AUTH_TOKEN_MISMATCH` と `canRetryWithDeviceToken=true` → クライアントは、キャッシュされたデバイストークンを使用して、信頼された再試行を 1 回実行できます。
    - そのキャッシュトークンによる再試行では、ペアリング済みデバイストークンとともに保存されたキャッシュ済みスコープセットが再利用されます。明示的な `deviceToken` / 明示的な `scopes` の呼び出し元では、代わりに要求したスコープセットが維持されます。
    - `AUTH_SCOPE_MISMATCH` → デバイストークンは認識されましたが、承認済みスコープがこの接続要求を満たしていません。共有 Gateway トークンをローテーションするのではなく、再ペアリングするか、要求されたスコープ契約を承認してください。
    - この再試行パス以外では、接続認証の優先順位は、明示的な共有トークンまたはパスワード、明示的な `deviceToken`、保存済みデバイストークン、ブートストラップトークンの順です。
    - 非同期の Tailscale Serve コントロール UI パスでは、同じ `{scope, ip}` からの失敗した試行は、リミッターが失敗を記録する前に直列化されます。そのため、同じクライアントから不正な再試行を 2 つ同時に行うと、単純な不一致が 2 回発生する代わりに、2 回目の試行で `retry later` が発生する場合があります。
    - ブラウザーオリジンのループバッククライアントからの `too many failed authentication attempts (retry later)` → 同じ正規化済み `Origin` から失敗が繰り返されると、一時的にロックアウトされます。別の localhost オリジンは別のバケットを使用します。
    - その再試行後も `unauthorized` が繰り返される → 共有トークンまたはデバイストークンのずれです。トークン設定を更新し、必要に応じてデバイストークンを再承認またはローテーションしてください。
    - `gateway connect failed:` → ホスト、ポート、または URL の接続先が間違っています。

  </Accordion>
</AccordionGroup>

### 認証詳細コードのクイックマップ

失敗した `connect` レスポンスの `error.details.code` を使用して、次のアクションを選択します。

| 詳細コード                  | 意味                                                                                                                                                                                      | 推奨アクション                                                                                                                                                                                                                                                                       |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AUTH_TOKEN_MISSING`         | クライアントが必須の共有トークンを送信しませんでした。                                                                                                                                                 | クライアントにトークンを貼り付けるか設定して、再試行します。ダッシュボードパスの場合: `openclaw config get gateway.auth.token` を実行し、コントロール UI の設定に貼り付けます。                                                                                                                                              |
| `AUTH_TOKEN_MISMATCH`        | 共有トークンが Gateway の認証トークンと一致しませんでした。                                                                                                                                               | `canRetryWithDeviceToken=true` の場合は、信頼された再試行を 1 回許可します。キャッシュトークンによる再試行では、保存済みの承認済みスコープが再利用されます。明示的な `deviceToken` / `scopes` の呼び出し元では、要求したスコープが維持されます。それでも失敗する場合は、[トークンのずれの復旧チェックリスト](/ja-JP/cli/devices#token-drift-recovery-checklist)を実行します。 |
| `AUTH_DEVICE_TOKEN_MISMATCH` | キャッシュされたデバイス単位のトークンが古いか、失効しています。                                                                                                                                                 | [デバイス CLI](/ja-JP/cli/devices) を使用してデバイストークンをローテーションまたは再承認してから、再接続します。                                                                                                                                                                                                        |
| `AUTH_SCOPE_MISMATCH`        | デバイストークンは有効ですが、承認済みのロールまたはスコープがこの接続要求を満たしていません。                                                                                                       | デバイスを再ペアリングするか、要求されたスコープ契約を承認してください。これを共有トークンのずれとして扱わないでください。                                                                                                                                                                                     |
| `PAIRING_REQUIRED`           | デバイス ID の承認が必要です。`error.details.reason` で `not-paired`、`scope-upgrade`、`role-upgrade`、または `metadata-upgrade` を確認し、存在する場合は `requestId` / `remediationHint` を使用します。 | 保留中の要求を承認します: `openclaw devices list`、次に `openclaw devices approve <requestId>`。要求されたアクセスを確認した後、スコープまたはロールのアップグレードにも同じフローを使用します。                                                                                                               |

<Note>
共有 Gateway トークンまたはパスワードで認証された直接のループバックバックエンド RPC は、CLI のペアリング済みデバイスのスコープベースラインに依存しないはずです。サブエージェントまたはその他の内部呼び出しが引き続き `scope-upgrade` で失敗する場合は、呼び出し元が `client.id: "gateway-client"` と `client.mode: "backend"` を使用しており、明示的な `deviceIdentity` またはデバイストークンを強制していないことを確認してください。
</Note>

デバイス認証 v2 の移行確認:

```bash
openclaw --version
openclaw doctor
openclaw gateway status
```

ログに nonce または署名のエラーが表示される場合は、接続するクライアントを更新し、次を確認します。

<Steps>
  <Step title="connect.challenge を待機">
    クライアントは、Gateway が発行する `connect.challenge` を待機します。
  </Step>
  <Step title="ペイロードに署名">
    クライアントは、チャレンジに関連付けられたペイロードに署名します。
  </Step>
  <Step title="デバイスの nonce を送信">
    クライアントは、同じチャレンジ nonce とともに `connect.params.device.nonce` を送信します。
  </Step>
</Steps>

`openclaw devices rotate` / `revoke` / `remove` が予期せず拒否された場合:

- ペアリング済みデバイストークンのセッションが管理できるのは、呼び出し元が `operator.admin` も持っている場合を除き、**そのセッション自身の**デバイスのみです。
- `openclaw devices rotate --scope ...` が要求できるのは、呼び出し元のセッションがすでに保持しているオペレータースコープのみです。

関連項目:

- [設定](/ja-JP/gateway/configuration)（Gateway の認証モード）
- [コントロール UI](/ja-JP/web/control-ui)
- [デバイス](/ja-JP/cli/devices)
- [リモートアクセス](/ja-JP/gateway/remote)
- [信頼済みプロキシ認証](/ja-JP/gateway/trusted-proxy-auth)

## Gateway サービスが実行されていない

サービスはインストールされているものの、プロセスが稼働し続けない場合に使用します。

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --deep   # システムレベルのサービスもスキャン
```

確認項目:

- 終了に関するヒントを伴う `Runtime: stopped`。
- サービス設定の不一致（`Config (cli)` と `Config (service)`）。
- ポートまたはリスナーの競合。
- `--deep` の使用時に追加された launchd/systemd/schtasks のインストール。
- `Other gateway-like services detected (best effort)` のクリーンアップに関するヒント。

<AccordionGroup>
  <Accordion title="一般的なシグネチャ">
    - `Gateway start blocked: set gateway.mode=local` または `existing config is missing gateway.mode` → ローカル Gateway モードが有効になっていないか、設定ファイルが上書きされて `gateway.mode` が失われています。修正方法: 設定で `gateway.mode="local"` を設定するか、`openclaw onboard --mode local` / `openclaw setup` を再実行して、想定されるローカルモード設定を再適用します。Podman 経由で OpenClaw を実行している場合、デフォルトの設定パスは `~/.openclaw/openclaw.json` です。
    - `refusing to bind gateway ... without auth` → 有効な Gateway 認証パス（トークンまたはパスワード、あるいは設定済みの場合は信頼済みプロキシ）がない非ループバックバインドです。
    - `another gateway instance is already listening` / `EADDRINUSE` → ポートの競合。
    - `Other gateway-like services detected (best effort)` → 古い、または並行して動作する launchd/systemd/schtasks ユニットが存在します。ほとんどの構成では、マシンごとに Gateway を 1 つ維持する必要があります。複数必要な場合は、ポート、設定、状態、ワークスペースを分離してください。[/gateway#multiple-gateways-same-host](/ja-JP/gateway#multiple-gateways-same-host) を参照してください。
    - doctor からの `System-level OpenClaw gateway service detected` → ユーザーレベルのサービスが存在しない一方で、systemd のシステムユニットが存在します。doctor にユーザーサービスをインストールさせる前に重複を削除または無効化するか、そのシステムユニットが意図したスーパーバイザーである場合は `OPENCLAW_SERVICE_REPAIR_POLICY=external` を設定します。
    - `Gateway service port does not match current gateway config` → インストール済みのスーパーバイザーが、引き続き古い `--port` に固定されています。`openclaw doctor --fix` または `openclaw gateway install --force` を実行してから、Gateway サービスを再起動します。

  </Accordion>
</AccordionGroup>

関連項目:

- [バックグラウンド実行とプロセスツール](/ja-JP/gateway/background-process)
- [設定](/ja-JP/gateway/configuration)
- [Doctor](/ja-JP/gateway/doctor)

## macOS の Gateway が応答を停止し、ダッシュボードを操作すると再開する

macOS ホスト上のチャンネル（Telegram、WhatsApp など）が数分から数時間にわたって応答しなくなり、Control UI を開く、SSH で接続する、またはホストを操作した瞬間に Gateway が復帰したように見える場合に使用します。確認する頃には Gateway が再び稼働しているため、通常 `openclaw status` には明らかな症状がありません。

```bash
ls ~/.openclaw/logs/stability/ | tail -5
openclaw gateway stability --bundle latest
pmset -g log | grep -iE "sleep|wake|maintenance" | tail -50
launchctl print gui/$UID/ai.openclaw.gateway | grep -E "state|last exit|runs"
```

確認事項：

- `~/.openclaw/logs/stability/` 内に 1 つ以上の `*-uncaught_exception.json` バンドルがあり、`error.code` が `ENETDOWN`、`ENETUNREACH`、`EHOSTUNREACH`、`ECONNREFUSED` などの一時的なネットワークコードに設定されている。
- クラッシュのタイムスタンプと一致する、`Entering Sleep state due to 'Maintenance Sleep'` や `en0 driver is slow (msg: WillChangeState to 0)` などの `pmset -g log` 行。Power Nap / Maintenance Sleep は Wi-Fi ドライバーを一時的に状態 0 にします。その時間帯に発生した送信 `connect()` は、それ以外の時間には完全なネットワーク接続があるホストでも `ENETDOWN` で失敗する可能性があります。
- `launchctl print` の出力に、終了コードと複数の最近の `runs` を伴う `state = not running` が表示されている。特に、クラッシュから次回起動までの間隔が数秒ではなく 1 時間程度の場合に該当します。macOS の launchd は、短時間にクラッシュが集中すると文書化されていない再生成保護ゲートを適用します。その結果、対話型ログイン、ダッシュボード接続、`launchctl kickstart` などの外部トリガーによって再作動するまで、`KeepAlive=true` が機能しなくなることがあります。

一般的な兆候：

- `error.code` が `ENETDOWN` または同種のコードであり、コールスタックが Node の `net` `lookupAndConnect` / `Socket.connect` を指している安定性バンドル。OpenClaw `2026.5.26` 以降では、これらを無害な一時的ネットワークエラーとして分類するため、最上位の未捕捉ハンドラーには伝播しなくなりました。古いリリースを使用している場合は、まずアップグレードしてください。
- Control UI に接続するか、ホストへ SSH 接続した瞬間に終了する長時間の無応答期間：再作動させているのはユーザーに見えるアクティビティであり、ダッシュボードが Gateway に対して行う処理ではありません。
- `~/Library/Logs/openclaw/gateway.log` に対応する `received SIG*; shutting down` 行がないまま、1 日を通して `runs` のカウントが増加している：正常なシャットダウンではシグナルが記録されますが、一時的なクラッシュでは記録されません。

対処方法：

1. `2026.5.26` より前のリリースを実行している場合は、**Gateway をアップグレードします**。アップグレード後、今後の `ENETDOWN` エラーはプロセスを終了させる代わりに警告として記録されます。
2. 常時稼働サーバーとして運用する Mac mini / デスクトップホストでは、**メンテナンススリープの動作を減らします**：

   ```bash
   sudo pmset -a sleep 0 disksleep 0 standby 0 powernap 0
   ```

   これにより、根本原因であるドライバーの瞬断は大幅に減少しますが、完全には解消されません。これらのフラグにかかわらず、システムは TCP keepalive と mDNS の維持管理のために一部のメンテナンススリープを実行することがあります。

3. 今後、クラッシュの集中後に launchd によって停止状態に置かれても迅速に検出できるよう、**生存監視ウォッチドッグを追加します**：

   ```bash
   # 5 分間隔の cron または LaunchAgent に適した、launchd 対応の生存確認例
   state=$(launchctl print gui/$UID/ai.openclaw.gateway 2>/dev/null | awk -F'= ' '/state =/ {print $2; exit}')
   if [ "$state" != "running" ]; then
     launchctl kickstart -k gui/$UID/ai.openclaw.gateway
   fi
   ```

   目的は、再生成ゲートを外部から再作動させることです。macOS ではクラッシュが集中した後、`KeepAlive=true` だけでは不十分です。

関連項目：

- [macOS プラットフォームの注意事項](/ja-JP/platforms/macos)
- [ログ記録](/ja-JP/logging)
- [Doctor](/ja-JP/gateway/doctor)

## Gateway/node の LaunchAgent が重複している場合の macOS launchd 監視ループ

macOS へのインストール環境で数秒ごとに再起動が繰り返され、`openclaw`
ヘルスチェックが正常と利用不可の間で変動し、サービスが実行中に見えるにもかかわらず
チャンネルディスパッチが停止する場合に使用します。

これは、`ai.openclaw.gateway` と
`ai.openclaw.node` の両方の LaunchAgent がアクティブで、それぞれが
`OPENCLAW_LAUNCHD_LABEL` を注入していた古いインストール環境で確認されました。この状態では、OpenClaw が launchd
による監視を検出し、再起動処理を launchd に戻そうとすることで、安定した単一の Gateway プロセスではなく、高速な
`EADDRINUSE`/再生成ループに陥る可能性があります。

```bash
for i in 1 2 3 4; do
  ps aux | grep 'openclaw.*index.js' | grep -v grep | awk '{print $2}'
  sleep 10
done

openclaw gateway status --deep
openclaw node status
launchctl print gui/$UID/ai.openclaw.gateway | grep -E 'state|last exit|runs'
tail -n 80 ~/Library/Logs/openclaw/gateway.log
```

確認事項：

- 30 秒間のサンプルで、安定した単一のプロセスではなく、複数の Gateway PID が
  表示される。
- `gateway.log` に `EADDRINUSE`、`another gateway instance is already listening`、または繰り返される
  再起動/引き継ぎ行がある。
- 管理対象の Gateway サービスを 1 つだけ実行する必要があるホストで、
  `~/Library/LaunchAgents/ai.openclaw.gateway.plist` と
  `~/Library/LaunchAgents/ai.openclaw.node.plist` の両方が同時に読み込まれている。

対処方法：

1. このホストで Gateway サービスのみを実行する必要がある場合は、OpenClaw を使用して管理対象の Node
   サービスを削除します。リモート Node 機能のために Node
   サービスを実際に使用している場合は、**この手順を省略してください**。アンインストールすると、このホスト上のそれらの機能が
   停止します：

   ```bash
   openclaw node uninstall
   ```

2. OpenClaw を起動する前に、継承された launchd
   マーカーを消去する永続的な Gateway ラッパーをインストールします。サポートされている `--wrapper` オプションを使用してください。
   サービスの再インストール、更新、および Doctor による修復ではファイルが再生成されるため、
   `~/.openclaw/service-env/` 配下の生成済みファイルを編集しないでください：

   ```bash
   mkdir -p ~/.local/bin
   cat >~/.local/bin/openclaw-launchd-workaround <<'EOF'
   #!/bin/sh
   set -eu
   unset OPENCLAW_LAUNCHD_LABEL LAUNCH_JOB_LABEL LAUNCH_JOB_NAME XPC_SERVICE_NAME || true
   exec openclaw "$@"
   EOF
   chmod 700 ~/.local/bin/openclaw-launchd-workaround

   openclaw gateway install \
     --wrapper ~/.local/bin/openclaw-launchd-workaround \
     --force
   ```

   `gateway install` は、強制再インストール、
   更新、および Doctor による修復を行ってもラッパーのパスを保持します。

3. Gateway が単にリッスンしているだけでなく、安定して RPC を提供していることを確認します：

   ```bash
   openclaw gateway status --deep --require-rpc

   for i in 1 2 3 4; do
     ps aux | grep 'openclaw.*index.js' | grep -v grep | awk '{print $2}'
     sleep 10
   done
   ```

   PID のサンプルには、入れ替わり続ける複数の
   PID ではなく、安定した単一のプロセスが表示され、受信チャンネルのディスパッチが再開する必要があります。

4. 根本原因である二重 LaunchAgent ループが
   修正されたリリースへアップグレードした後、回避策を削除して通常の管理対象サービスを再インストールします：

   ```bash
   OPENCLAW_WRAPPER= openclaw gateway install --force
   rm ~/.local/bin/openclaw-launchd-workaround
   ```

関連項目：

- [macOS プラットフォームの注意事項](/ja-JP/platforms/mac/bundled-gateway)
- [Doctor](/ja-JP/gateway/doctor)
- [Gateway CLI](/ja-JP/cli/gateway)

## メモリ使用量が多いときに Gateway が終了する

負荷がかかると Gateway が消失する場合、スーパーバイザーが OOM 形式の再起動を報告する場合、またはログに `critical memory pressure bundle written` と記録されている場合に使用します。

```bash
openclaw gateway status --deep
openclaw logs --follow
openclaw gateway stability --bundle latest
openclaw gateway diagnostics export
```

確認事項：

- 最新の安定性バンドル内の `Reason: diagnostic.memory.pressure.critical`。
- `critical/rss_threshold`、`critical/heap_threshold`、または `critical/rss_growth` を伴う `Memory pressure:`。
- ヒープ上限に近い `V8 heap:` の値。
- `agents/<agent>/sessions/<session>.jsonl` や `sessions/<session>.jsonl` などの `Largest session files:` エントリ。
- Gateway がコンテナまたはメモリ制限付きサービス内で実行されている場合の Linux cgroup メモリカウンター。

一般的な兆候：

- 再起動の直前に `critical memory pressure bundle written` が表示される → OpenClaw が OOM 発生前の安定性バンドルを取得しています。`openclaw gateway stability --bundle latest` で調査してください。
- Gateway のログに `memory pressure: level=critical` が表示される → OpenClaw が重大なメモリプレッシャーを検出し、利用可能なプロセス内メモリ情報を記録しています。
- `Largest session files:` が非常に大きな墨消し済みトランスクリプトパスを指している → 保持するセッション履歴を減らし、セッションの増加を調査するか、再起動前に古いトランスクリプトをアクティブストアの外へ移動してください。
- `V8 heap:` の使用バイト数がヒープ上限に近い → まずプロンプト/セッションの負荷を下げるか、同時実行作業を減らしてください。管理対象サービスでは、`openclaw gateway status` 内の `Gateway heap:` を確認してください。`not set` と表示される場合は、`openclaw gateway install --force` で古いサービスメタデータを再生成してください。周囲のシェルにある `NODE_OPTIONS` は意図的に無視されます。明示的なスーパーバイザーレベルのヒープ上書きは、継続的なワークロードを確認し、ネイティブメモリに十分な余裕を確保した後にのみ使用してください。
- `Memory pressure: critical/rss_growth` → 1 回のサンプリング期間内にメモリが急増しています。最新のログで、大規模なインポート、暴走したツール出力、繰り返される再試行、またはキューに入ったエージェント作業の一群を確認してください。
- 重大なメモリプレッシャーがログに表示されるが、バンドルが存在しない → イベント発生後に `openclaw gateway diagnostics export` を取得し、利用可能な運用上の証拠を収集してください。

安定性バンドルにはペイロードが含まれません。メッセージ本文、Webhook 本文、認証情報、トークン、Cookie、生のセッション ID ではなく、運用上のメモリ情報と墨消し済みの相対ファイルパスが含まれます。生のログをコピーする代わりに、診断エクスポートをバグ報告へ添付してください。

関連項目：

- [Gateway のヘルス](/ja-JP/gateway/health)
- [診断エクスポート](/ja-JP/gateway/diagnostics)
- [セッション](/ja-JP/cli/sessions)

## Gateway が無効な設定を拒否した

Gateway の起動が `Invalid config` で失敗する場合、またはホットリロードのログに無効な編集をスキップしたと記録されている場合に使用します。

```bash
openclaw logs --follow
openclaw config file
openclaw config validate
openclaw doctor
```

確認事項：

- `Invalid config at ...`
- `config reload skipped (invalid config): ...`
- `Config write rejected: ...`
- アクティブな設定の隣にある、タイムスタンプ付きの `openclaw.json.rejected.*` ファイル。
- `doctor --fix` が壊れた直接編集を修復した場合の、タイムスタンプ付き `openclaw.json.clobbered.*` ファイル。
- OpenClaw は設定パスごとに最新の 32 個の `.clobbered.*` ファイルを保持し、それより古いものをローテーションします。

<AccordionGroup>
  <Accordion title="発生したこと">
    - 起動、ホットリロード、または OpenClaw が管理する書き込み時に、設定の検証に失敗しました。
    - Gateway の起動は、`openclaw.json` を書き換える代わりにフェイルクローズします。
    - ホットリロードは無効な外部編集をスキップし、現在のランタイム設定を有効なまま維持します。
    - OpenClaw が管理する書き込みは、コミット前に無効または破壊的なペイロードを拒否し、`.rejected.*` を保存します。
    - 修復は `openclaw doctor --fix` が担当します。JSON ではないプレフィックスを削除するか、拒否されたペイロードを `.clobbered.*` として保持しながら、最後に正常だったコピーを復元できます。
    - 1 つの設定パスに対して多数の修復が行われると、OpenClaw は古い `.clobbered.*` ファイルをローテーションし、最新の修復済みペイロードを引き続き利用できるようにします。

  </Accordion>
  <Accordion title="検査と修復">
    ```bash
    CONFIG="$(openclaw config file)"
    ls -lt "$CONFIG".clobbered.* "$CONFIG".rejected.* 2>/dev/null | head
    diff -u "$CONFIG" "$(ls -t "$CONFIG".clobbered.* 2>/dev/null | head -n 1)"
    openclaw config validate
    openclaw doctor
    ```
  </Accordion>
  <Accordion title="一般的な兆候">
    - `.clobbered.*` が存在する → doctor が有効な設定を修復する際、壊れた外部編集を保持しました。
    - `.rejected.*` が存在する → OpenClaw が管理する設定の書き込みが、コミット前のスキーマまたは上書きチェックに失敗しました。
    - `Config write rejected:` → 書き込みが必須の構造を削除しようとした、ファイルを大幅に縮小しようとした、または無効な設定を永続化しようとしました。
    - `config reload skipped (invalid config):` → 直接編集が検証に失敗し、実行中の Gateway に無視されました。
    - `Invalid config at ...` → Gateway サービスが起動する前に、起動処理が失敗しました。
    - `missing-meta-vs-last-good`、`gateway-mode-missing-vs-last-good`、または `size-drop-vs-last-good:*` → OpenClaw が管理する書き込みは、既知の正常な最新バックアップと比べてフィールドまたはサイズが失われたため拒否されました。
    - `Config last-known-good promotion skipped` → 候補に `***` などの秘匿化されたシークレットのプレースホルダーが含まれていました。

  </Accordion>
  <Accordion title="修復方法">
    1. doctor に接頭辞付き／上書きされた設定を修復させるか、既知の正常な最新設定を復元するには、`openclaw doctor --fix` を実行します。
    2. `.clobbered.*` または `.rejected.*` から意図したキーのみをコピーし、`openclaw config set` または `config.patch` で適用します。
    3. 再起動する前に `openclaw config validate` を実行します。
    4. 手動で編集する場合は、変更したい部分オブジェクトだけでなく、JSON5 設定全体を保持してください。
  </Accordion>
</AccordionGroup>

関連項目：

- [設定](/ja-JP/cli/config)
- [設定：ホットリロード](/ja-JP/gateway/configuration#config-hot-reload)
- [設定：厳格な検証](/ja-JP/gateway/configuration#strict-validation)
- [Doctor](/ja-JP/gateway/doctor)

## Gateway プローブの警告

`openclaw gateway probe` が何らかの対象に到達するものの、引き続き警告ブロックが表示される場合に使用します。

```bash
openclaw gateway probe
openclaw gateway probe --json
openclaw gateway probe --ssh user@gateway-host
```

確認事項：

- JSON 出力内の `warnings[].code` と `primaryTargetId`。
- 警告が SSH フォールバック、複数の Gateway、スコープ不足、未解決の認証参照のいずれに関するものか。

一般的な兆候：

- `SSH tunnel failed to start; falling back to direct probes.` → SSH のセットアップに失敗しましたが、コマンドは設定済みの直接接続先／ループバック接続先への試行を続けました。
- `multiple reachable gateway identities detected` → 異なる Gateway が応答したか、OpenClaw が到達可能な接続先を同一の Gateway であると確認できませんでした。同じ Gateway への SSH トンネル、プロキシ URL、または設定済みリモート URL は、トランスポートのポートが異なる場合でも、複数のトランスポートを持つ 1 つの Gateway として扱われます。
- `Read-probe diagnostics are limited by gateway scopes (missing operator.read)` → 接続には成功しましたが、詳細 RPC はスコープによって制限されています。デバイス ID をペアリングするか、`operator.read` を持つ認証情報を使用してください。
- `Gateway accepted the WebSocket connection, but follow-up read diagnostics failed` → 接続には成功しましたが、診断 RPC 一式がタイムアウトまたは失敗しました。診断機能が低下しているものの到達可能な Gateway として扱い、`--json` の出力で `connect.ok` と `connect.rpcOk` を比較してください。
- `Capability: pairing-pending` または `gateway closed (1008): pairing required` → Gateway は応答しましたが、このクライアントが通常のオペレーターアクセスを行うには、引き続きペアリング／承認が必要です。
- 未解決の `gateway.auth.*` / `gateway.remote.*` SecretRef 警告テキスト → 失敗した接続先について、このコマンドパスでは認証情報を利用できませんでした。

関連項目：

- [Gateway](/ja-JP/cli/gateway)
- [同じホスト上の複数の Gateway](/ja-JP/gateway#multiple-gateways-same-host)
- [リモートアクセス](/ja-JP/gateway/remote)

## チャンネルは接続済みだが、メッセージが流れない

チャンネルの状態が接続済みでもメッセージフローが停止している場合は、ポリシー、権限、チャンネル固有の配信ルールを重点的に確認します。

```bash
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw status --deep
openclaw logs --follow
openclaw config get channels
```

確認事項：

- DM ポリシー（`pairing`、`allowlist`、`open`、`disabled`）。
- グループの許可リストとメンション要件。
- 不足しているチャンネル API の権限／スコープ。

一般的な兆候：

- `mention required` → グループのメンションポリシーにより、メッセージが無視されました。
- `pairing` / 承認待ちのトレース → 送信者が承認されていません。
- `missing_scope`、`not_in_channel`、`Forbidden`、`401/403` → チャンネルの認証／権限の問題です。

関連項目：

- [チャンネルのトラブルシューティング](/ja-JP/channels/troubleshooting)
- [Discord](/ja-JP/channels/discord)
- [Telegram](/ja-JP/channels/telegram)
- [WhatsApp](/ja-JP/channels/whatsapp)

## Cron と Heartbeat の配信

Cron または Heartbeat が実行されなかった、あるいは配信されなかった場合は、まずスケジューラーの状態を確認し、次に配信先を確認します。

```bash
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
```

確認事項：

- Cron が有効で、次回のウェイク時刻が存在すること。
- ジョブ実行履歴の状態（`ok`、`skipped`、`error`）。
- Heartbeat のスキップ理由（`quiet-hours`、`requests-in-flight`、`cron-in-progress`、`lanes-busy`、`alerts-disabled`、`empty-heartbeat-file`）。

<AccordionGroup>
  <Accordion title="一般的な兆候">
    - `cron: scheduler disabled; jobs will not run automatically` → Cron が無効です。
    - `cron: timer tick failed` → スケジューラーのティックに失敗しました。ファイル／ログ／ランタイムエラーを確認してください。
    - `heartbeat skipped` と `reason=quiet-hours` → アクティブ時間帯の範囲外です。
    - `heartbeat skipped` と `reason=empty-heartbeat-file` → Heartbeat モニターのスクラッチには、空白、コメント、ヘッダー、フェンス、または空のチェックリストのひな形しか含まれていないため、OpenClaw はモデル呼び出しをスキップします。
    - `heartbeat: unknown accountId` → Heartbeat の配信先に対するアカウント ID が無効です。
    - `heartbeat skipped` と `reason=dm-blocked` → `agents.defaults.heartbeat.directPolicy`（またはエージェントごとのオーバーライド）が `block` に設定されている一方で、Heartbeat の接続先が DM 形式の宛先として解決されました。

  </Accordion>
</AccordionGroup>

関連項目：

- [Heartbeat](/ja-JP/gateway/heartbeat)
- [スケジュール済みタスク](/ja-JP/automation/cron-jobs)
- [スケジュール済みタスク：トラブルシューティング](/ja-JP/automation/cron-jobs#troubleshooting)

## Node はペアリング済みだが、ツールが失敗する

Node がペアリング済みでもツールが失敗する場合は、フォアグラウンド、権限、承認の状態を切り分けます。

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
openclaw logs --follow
openclaw status
```

確認事項：

- Node がオンラインで、想定される機能を備えていること。
- カメラ／マイク／位置情報／画面に対する OS 権限の付与。
- 実行承認と許可リストの状態。

一般的な兆候：

- `NODE_BACKGROUND_UNAVAILABLE` → Node アプリをフォアグラウンドにする必要があります。
- `*_PERMISSION_REQUIRED` / `LOCATION_PERMISSION_REQUIRED` → OS 権限が不足しています。
- `SYSTEM_RUN_DENIED: approval required` → 実行承認が保留中です。
- `SYSTEM_RUN_DENIED: allowlist miss` → コマンドが許可リストによってブロックされました。

関連項目：

- [実行承認](/ja-JP/tools/exec-approvals)
- [Node のトラブルシューティング](/ja-JP/nodes/troubleshooting)
- [Node](/ja-JP/nodes/index)

## ブラウザツールが失敗する

Gateway 自体は正常でも、ブラウザツールの操作が失敗する場合に使用します。

```bash
openclaw browser status
openclaw browser start --browser-profile openclaw
openclaw browser profiles
openclaw logs --follow
openclaw doctor
```

確認事項：

- `plugins.allow` が設定され、`browser` が含まれているか。
- ブラウザの実行可能ファイルへのパスが有効か。
- CDP プロファイルに到達可能か。
- `existing-session` / `user` プロファイルでローカル Chrome を利用できるか。

<AccordionGroup>
  <Accordion title="Plugin／実行可能ファイルの兆候">
    - `unknown command "browser"` または `unknown command 'browser'` → バンドルされたブラウザ Plugin が `plugins.allow` によって除外されています。
    - `browser.enabled=true` の状態でブラウザツールが存在しない／利用できない → `plugins.allow` が `browser` を除外しているため、Plugin が読み込まれていません。
    - `Failed to start Chrome CDP on port` → ブラウザプロセスの起動に失敗しました。
    - `browser.executablePath not found` → 設定されたパスが無効です。
    - `browser.cdpUrl must be http(s) or ws(s)` → 設定された CDP URL が、`file:` や `ftp:` などのサポートされていないスキームを使用しています。
    - `browser.cdpUrl has invalid port` → 設定された CDP URL のポートが不正または範囲外です。
    - `Playwright is not available in this gateway build; '<feature>' is unsupported.` → 現在の Gateway インストールに、コアブラウザのランタイム依存関係がありません。OpenClaw を再インストールまたは更新してから、Gateway を再起動してください。ARIA スナップショットと基本的なページのスクリーンショットは引き続き機能する場合がありますが、ナビゲーション、AI スナップショット、CSS セレクターによる要素のスクリーンショット、PDF エクスポートは利用できないままです。

  </Accordion>
  <Accordion title="Chrome MCP／既存セッションの兆候">
    - `Could not find DevToolsActivePort for chrome` → Chrome MCP の既存セッションが、選択されたブラウザのデータディレクトリにまだ接続できませんでした。ブラウザの検査ページを開き、リモートデバッグを有効にし、ブラウザを開いたままにして、初回接続のプロンプトを承認してから再試行してください。サインイン状態が不要な場合は、管理対象の `openclaw` プロファイルを推奨します。
    - `No browser tabs found for profile="user"` → Chrome MCP の接続プロファイルに、開いているローカル Chrome タブがありません。
    - `Remote CDP for profile "<name>" is not reachable` → 設定されたリモート CDP エンドポイントに Gateway ホストから到達できません。
    - `Browser attachOnly is enabled ... not reachable` または `Browser attachOnly is enabled and CDP websocket ... is not reachable` → 接続専用プロファイルに到達可能な接続先がないか、HTTP エンドポイントは応答したものの CDP WebSocket を開けませんでした。

  </Accordion>
  <Accordion title="要素／スクリーンショット／アップロードの兆候">
    - `fullPage is not supported for element screenshots` → スクリーンショット要求で `--full-page` と `--ref` または `--element` が混在しています。
    - `element screenshots are not supported for existing-session profiles; use ref from snapshot.` → Chrome MCP / `existing-session` のスクリーンショット呼び出しでは、CSS の `--element` ではなく、ページキャプチャまたはスナップショットの `--ref` を使用する必要があります。
    - `existing-session file uploads do not support element selectors; use ref/inputRef.` → Chrome MCP のアップロードフックには、CSS セレクターではなくスナップショット参照が必要です。
    - `existing-session file uploads currently support one file at a time.` → Chrome MCP プロファイルでは、1 回の呼び出しにつき 1 件のアップロードを送信してください。
    - `existing-session dialog handling does not support timeoutMs.` → Chrome MCP プロファイルのダイアログフックは、タイムアウトのオーバーライドに対応していません。
    - `existing-session type does not support timeoutMs overrides.` → `profile="user"` / Chrome MCP の既存セッションプロファイルで `act:type` を使用する場合は `timeoutMs` を省略するか、カスタムタイムアウトが必要な場合は管理対象／CDP ブラウザプロファイルを使用してください。
    - `response body is not supported for existing-session profiles yet.` → `responsebody` には、引き続き管理対象ブラウザまたは Raw CDP プロファイルが必要です。
    - 接続専用またはリモート CDP プロファイルに古いビューポート／ダークモード／ロケール／オフラインのオーバーライドが残っている → `openclaw browser stop --browser-profile <name>` を実行し、Gateway 全体を再起動せずにアクティブな制御セッションを閉じて、Playwright／CDP のエミュレーション状態を解放します。

  </Accordion>
</AccordionGroup>

関連項目：

- [ブラウザ（OpenClaw 管理）](/ja-JP/tools/browser)
- [ブラウザのトラブルシューティング](/ja-JP/tools/browser-linux-troubleshooting)

## アップグレード後に突然何かが壊れた場合

アップグレード後の不具合の大半は、設定のずれ、または厳格化されたデフォルトが新たに適用されたことが原因です。

<AccordionGroup>
  <Accordion title="1. 認証と URL オーバーライドの動作が変更された">
    ```bash
    openclaw gateway status
    openclaw config get gateway.mode
    openclaw config get gateway.remote.url
    openclaw config get gateway.auth.mode
    ```

    確認事項：

    - `gateway.mode=remote` の場合、ローカルサービスは正常でも、CLI 呼び出しがリモートを対象にしている可能性があります。
    - 明示的な `--url` 呼び出しでは、保存済みの認証情報にフォールバックしません。

    よくある兆候:

    - `gateway connect failed:` → URL の接続先が正しくありません。
    - `unauthorized` → エンドポイントには到達できますが、認証が正しくありません。

  </Accordion>
  <Accordion title="2. バインドと認証のガードレールが厳格化されました">
    ```bash
    openclaw config get gateway.bind
    openclaw config get gateway.auth.mode
    openclaw config get gateway.auth.token
    openclaw gateway status
    openclaw logs --follow
    ```

    確認事項:

    - 非ループバックバインド（`lan`、`tailnet`、`custom`）には、有効な Gateway 認証経路が必要です。共有トークンまたはパスワードによる認証、あるいは正しく構成された非ループバックの `trusted-proxy` デプロイを使用してください。
    - `gateway.token` のような古いキーは、`gateway.auth.token` の代わりにはなりません。

    よくある兆候:

    - `refusing to bind gateway ... without auth` → 有効な Gateway 認証経路がない非ループバックバインドです。
    - ランタイムの実行中に `Connectivity probe: failed` → Gateway は稼働していますが、現在の認証または URL ではアクセスできません。

  </Accordion>
  <Accordion title="3. ペアリングとデバイス ID の状態が変更されました">
    ```bash
    openclaw devices list
    openclaw pairing list --channel <channel> [--account <id>]
    openclaw logs --follow
    openclaw doctor
    ```

    確認事項:

    - ダッシュボードまたは Node に対する保留中のデバイス承認。
    - ポリシーまたは ID の変更後に保留中となっている DM ペアリング承認。

    よくある兆候:

    - `device identity required` → デバイス認証の要件が満たされていません。
    - `pairing required` → 送信者またはデバイスの承認が必要です。

  </Accordion>
</AccordionGroup>

確認後もサービス設定とランタイムが一致しない場合は、同じプロファイルおよび状態ディレクトリからサービスのメタデータを再インストールします。

```bash
openclaw gateway install --force
openclaw gateway restart
```

関連項目:

- [認証](/ja-JP/gateway/authentication)
- [バックグラウンド実行とプロセスツール](/ja-JP/gateway/background-process)
- [Node のペアリング](/ja-JP/gateway/pairing)

## 関連項目

- [Doctor](/ja-JP/gateway/doctor)
- [FAQ](/ja-JP/help/faq)
- [Gateway 運用手順書](/ja-JP/gateway)
