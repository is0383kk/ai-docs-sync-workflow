---
read_when:
    - 副作用が実行される前に確認するには、Plugin フックまたはツールが必要です
    - Plugin の承認プロンプトの送信先を設定する必要があります
    - オプションのツール、exec の承認、Plugin の承認のどれを使用するか決定します
sidebarTitle: Permission requests
summary: Plugin のツール呼び出しと Plugin が管理する権限プロンプトについて、ユーザーに承認を求める
title: Plugin の権限リクエスト
x-i18n:
    generated_at: "2026-07-26T09:09:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 675534212e70cc7b2e7bdc801955929c6a8156b08d620483edf0133afc3bfdaa
    source_path: plugins/plugin-permission-requests.md
    workflow: 16
---

Plugin 権限リクエストを使用すると、ユーザーが承認または拒否するまで、Plugin コードがツール呼び出しや Plugin 所有の
操作を一時停止できます。これらは Gateway の
`plugin.approval.*` フローと、チャットの承認ボタンおよび
`/approve` コマンドを処理するものと同じ承認 UI サーフェスを使用します。

Plugin/App の権限には Plugin 権限リクエストを使用します。これは、ホストの exec 承認、
オプションのツール許可リスト、または Codex ネイティブの権限
レビューに代わるものではありません。

## 適切なゲートを選択する

必要な判断ポイントに一致するゲートを選択します。

| ゲート                           | 使用する場面                                                             | 制御対象                                                                                                          |
| -------------------------------- | ------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| オプションツール                 | ユーザーがオプトインするまで、ツールをモデルに表示すべきでない場合。     | `tools.allow` を介したツールの公開。                                                                         |
| Plugin 権限リクエスト            | Plugin フックまたは Plugin 所有の操作で、1 つのアクションを実行する前に確認が必要な場合。 | `plugin.approval.*` を介した実行時の承認。                                                                         |
| Exec 承認                        | ホストコマンドまたはシェルに似たツールにオペレーターの承認が必要な場合。 | ホストの exec ポリシーと永続的な exec 許可リスト。                                                               |
| Codex ネイティブ権限リクエスト   | Codex がネイティブのシェル、ファイル、MCP、または App Server のアクションを実行する前に確認する場合。 | Codex App Server またはネイティブフックの承認処理。OpenClaw がプロンプトを所有する場合は Plugin 承認を介してルーティングされます。 |
| MCP 承認要請                     | Codex MCP サーバーがツール呼び出しの承認をリクエストする場合。           | OpenClaw の Plugin 承認を介してブリッジされる MCP 承認レスポンス。                                                |

オプションツールは検出時のゲートです。Plugin 権限リクエストは
呼び出しごとのゲートです。機密性の高いツールをモデルに表示する前に明示的なオプトインが必要で、
アクションの実行前にも承認が必要な場合は、両方を使用します。

## ツール呼び出し前に承認をリクエストする

Plugin で作成するほとんどのプロンプトは、`before_tool_call` フックから開始する必要があります。このフックは、
モデルがツールを選択した後、OpenClaw がそれを実行する前に動作します。

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "deploy-policy",
  name: "Deploy Policy",
  register(api) {
    api.on("before_tool_call", async (event) => {
      if (event.toolName !== "deploy_service") {
        return;
      }

      const environment =
        typeof event.params.environment === "string" ? event.params.environment : "unknown";

      return {
        requireApproval: {
          title: "Deploy service",
          description: `Deploy service to ${environment}.`,
          severity: environment === "production" ? "critical" : "warning",
          allowedDecisions:
            environment === "production"
              ? ["allow-once", "deny"]
              : ["allow-once", "allow-always", "deny"],
          timeoutMs: 120_000,
          onResolution(decision) {
            console.log(`deploy approval resolved: ${decision}`);
          },
        },
      };
    });
  },
});
```

アクションを承認する人に向けてプロンプトテキストを記述します。

- `title` は短く、アクションに焦点を当ててください。Gateway では 80 文字に制限されます。
- `description` は具体的かつ範囲を限定してください。Gateway では 512
  文字に制限されます。
- アクション、対象、リスクを含めてください。チャットの承認サーフェスに表示すべきでないシークレット、トークン、
  または非公開のペイロードを含めないでください。
- `severity` を省略すると、デフォルトは `"warning"` になります。誤った判断によって
  本番環境の損害やデータ損失が発生する可能性があるアクションにのみ `"critical"` を使用してください。
- `allowedDecisions` を
  省略すると、デフォルトは `["allow-once", "allow-always", "deny"]` になります。そのアクションで永続的な信頼が
  安全でない場合は、`["allow-once", "deny"]` を渡してください。
- `timeoutMs` のデフォルトは 120000（2 分）で、リクエストされた値に関係なく 600000（10
  分）が上限です。

## 判断時の動作

OpenClaw は `plugin:` ID を持つ保留中の承認を作成し、利用可能な
承認サーフェスに配信して、判断を待ちます。

| 判断              | 結果                                                                      |
| ----------------- | ------------------------------------------------------------------------- |
| `allow-once`      | 現在の呼び出しが続行されます。                                            |
| `allow-always`    | 現在の呼び出しが続行され、判断が Plugin に渡されます。                    |
| `deny`            | 呼び出しが拒否されたツール結果によってブロックされます。                  |
| タイムアウト      | 呼び出しがブロックされます。                                              |
| キャンセル        | 実行が中止された場合、呼び出しがブロックされます。                        |
| 承認経路なし      | 接続済みの承認サーフェスで解決できないため、呼び出しがブロックされます。  |

リクエストで許可された正確な `allow-once` および `allow-always` の判断のみが
実行を許可します。不明、不正、対応不一致、欠落、タイムアウトした
判断はフェイルクローズします。従来の `timeoutBehavior` フィールドは Plugin の互換性のため
引き続き受け入れられますが、非推奨で無視されます。新しいフックでは設定しないでください。

`allow-always` が永続的になるのは、リクエスト元の Plugin またはランタイムが
その永続化を実装している場合のみです。通常の `before_tool_call.requireApproval` フックでは、
OpenClaw は `allow-once` と `allow-always` を現在の呼び出しに対する承認判断として扱い、
解決済みの値を `onResolution` に渡します。Plugin が
`allow-always` を提供する場合は、それがどの将来の呼び出しを信頼するのかを正確に文書化して
実装してください。

フックが `params` も返す場合、OpenClaw は承認が成功した後にのみ
それらのパラメーター変更を適用します。優先度の低いフックは、優先度の高いフックが
承認をリクエストした後でもブロックできます。

`allowedDecisions` は、ユーザーに表示されるボタンとコマンドを制限します。
Gateway は、リクエストで提示されなかった判断による解決試行を拒否します。

## 承認プロンプトをルーティングする

承認プロンプトは、ローカル UI サーフェス、または承認処理を
サポートするチャットチャンネルで解決できます。Plugin 承認プロンプトを明示的なチャット
ターゲットに転送するには、`approvals.plugin` を設定します。

```json5
{
  approvals: {
    plugin: {
      enabled: true,
      mode: "targets",
      agentFilter: ["main"],
      targets: [{ channel: "slack", to: "U12345678" }],
    },
  },
}
```

`approvals.plugin` は `approvals.exec` から独立しています。Exec 承認の
転送を有効にしても Plugin 承認プロンプトはルーティングされず、Plugin 承認の
転送を有効にしてもホストの exec ポリシーは変更されません。

プロンプトに手動承認用のテキストが含まれる場合は、提示された
判断のいずれかを使用して解決します。

```text
/approve <id> allow-once
/approve <id> allow-always
/approve <id> deny
```

完全な転送モデル、同一チャットでの承認動作、ネイティブチャンネルへの
配信、チャンネル固有の承認者ルールについては、[高度な exec 承認](/ja-JP/tools/exec-approvals-advanced#plugin-approval-forwarding)
を参照してください。

## Codex ネイティブ権限

Codex ネイティブの権限プロンプトも Plugin 承認を介して伝達できますが、
Plugin で作成されたフックとは所有権が異なります。

- Codex App Server の承認リクエストは、Codex のレビュー後に OpenClaw を介してルーティングされます。
- ネイティブフックの `permission_request` リレーは、そのリレーが有効な場合、
  `plugin.approval.request` を介して確認できます。
- Codex が `_meta.codex_approval_kind` を `"mcp_tool_call"` としてマークした場合、
  MCP ツールの承認要請は Plugin 承認を介してルーティングされます。

Codex 固有の動作とフォールバックルールについては、[Codex ハーネスランタイム](/ja-JP/plugins/codex-harness-runtime#native-permissions-and-mcp-elicitations)
を参照してください。

## トラブルシューティング

**ツールに Plugin 承認を利用できないと表示される。** 承認 UI または設定済みの
承認経路がリクエストを受け付けませんでした。承認対応クライアントを接続するか、
同一チャットの `/approve` をサポートするチャンネルを使用するか、`approvals.plugin` を設定してください。

**`allow-always` が表示されたが、次の呼び出しで再度プロンプトが表示される。** 汎用の Plugin
承認フローは、任意のフックに対する信頼を自動的に永続化しません。`onResolution("allow-always")` の後に
Plugin 所有の信頼を Plugin 内で永続化するか、
`allow-once` と `deny` のみを提示してください。

**`/approve` が判断を拒否する。** リクエストによって
`allowedDecisions` が制限されています。プロンプトに表示された判断のいずれかを使用してください。

**Discord、Matrix、Slack、または Telegram のプロンプトが exec
承認とは異なる方法でルーティングされる。** Plugin 承認と exec 承認では別々の設定を使用し、
異なる認可チェックが適用される場合があります。`approvals.exec` だけを確認するのではなく、
`approvals.plugin` とチャンネルの Plugin 承認サポートを確認してください。

## 関連項目

- [Plugin フック](/ja-JP/plugins/hooks#tool-call-policy)
- [Plugin の構築](/ja-JP/plugins/building-plugins#registering-tools)
- [高度な exec 承認](/ja-JP/tools/exec-approvals-advanced#plugin-approval-forwarding)
- [Gateway プロトコル](/ja-JP/gateway/protocol)
- [Codex ハーネスランタイム](/ja-JP/plugins/codex-harness-runtime#native-permissions-and-mcp-elicitations)
