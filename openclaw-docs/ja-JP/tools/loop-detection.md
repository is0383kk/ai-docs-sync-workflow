---
read_when:
    - ユーザーから、エージェントがツール呼び出しを繰り返して停止できなくなるとの報告がある
    - 反復呼び出し保護を制御する必要があります
    - エージェントのツール／ランタイムポリシーを編集しています
    - コンテキストオーバーフロー後の再試行後に `compaction_loop_persisted` の中断が発生する
summary: 反復的なツール呼び出しループを検出するガードレールを有効にする方法
title: ツールループの検出
x-i18n:
    generated_at: "2026-07-26T10:23:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 79b5aa1d85e02b8cf46a95b3bcebb255178b91456517cab804cce77b8f3b818e
    source_path: tools/loop-detection.md
    workflow: 16
---

OpenClaw には、反復的なツール呼び出しパターンを防ぐために連携して動作する 2 つのガードレールがあり、
どちらも `tools.loopDetection` で設定されます。

1. **ループ検出**（`enabled`）- デフォルトでは無効です。ローリング形式の
   ツール呼び出し履歴を監視し、反復パターンや不明なツールの再試行を検出します。
2. **Compaction 後ガード** - 
   `enabled` が明示的に `false` でない場合は常に有効です。Compaction の再試行後に毎回作動し、
   エージェントがウィンドウ内で同じ `(tool, args, result)` の組み合わせを
   繰り返すと、実行を中止します。

両方のガードレールを無効にするには、`tools.loopDetection.enabled: false` を設定します。

## この機能が存在する理由

- 進展のない反復シーケンスを検出します。
- 高頻度で結果が得られないループ（同じツール、同じ入力、反復する
  エラー）を検出します。
- 既知のポーリングツールに固有の反復呼び出しパターンを検出します。
- コンテキストオーバーフロー -> Compaction -> 同じループというサイクルを、
  無期限に実行させる代わりに中断します。

## 設定ブロック

グローバル設定：

```json5
{
  tools: {
    loopDetection: {
      enabled: false, // ローリング履歴検出機能のマスタースイッチ
    },
  },
}
```

エージェントごとのオーバーライド（任意、`agents.entries.*.tools.loopDetection`）：

```json5
{
  agents: {
    list: [
      {
        id: "safe-runner",
        tools: {
          loopDetection: {
            enabled: true,
          },
        },
      },
    ],
  },
}
```

エージェントごとの設定はグローバル設定をオーバーライドします。

### フィールドの動作

| フィールド     | デフォルト | 効果                                                                                            |
| --------- | ------- | ------------------------------------------------------------------------------------------------- |
| `enabled` | `false` | ローリング履歴検出機能のマスタースイッチです。`false` にすると Compaction 後ガードも無効になります。 |

`exec` では、進展なしのハッシュ処理は安定したコマンド結果（ステータス、
終了コード、タイムアウトフラグ、出力）を比較し、実行時間、PID、セッション ID、作業ディレクトリなどの
変動するランタイムメタデータを無視します。外向けメッセージ送信の
結果は、呼び出しごとに変動する ID（メッセージ ID、ファイル ID、タイムスタンプ）を
除外してハッシュ化されるため、ある「送信済み」の結果が別の「送信済み」の
結果と同一とはみなされません。実行 ID が利用可能な場合、履歴はその実行内でのみ評価されるため、
スケジュールされた Heartbeat サイクルや新規実行が、以前の実行から古いループ回数を
引き継ぐことはありません。

## 推奨設定

- 小規模なモデルでは、`enabled: true` を設定します。フラッグシップモデルでローリング履歴検出が必要になることはほとんどなく、
マスタースイッチを `false` のままにしても、
Compaction 後ガードの恩恵を受けられます。
- Compaction 後ガードを含むすべての機能を無効にするには、
  `tools.loopDetection.enabled: false` を明示的に設定します。

## Compaction 後ガード

コンテキストオーバーフロー後に Compaction の再試行が行われると、ランナーは
続く数回のツール呼び出しに対して短いウィンドウのガードを作動させます。エージェントがそのウィンドウ内で同じ
`(toolName, argsHash, resultHash)` の組み合わせを十分な回数出力すると、ガードは Compaction によって
ループが解消されなかったと判断し、`compaction_loop_persisted` エラーで実行を中止します。

このガードはマスター `tools.loopDetection.enabled` フラグによって制御されますが、1 つ例外があります。
フラグが未設定または `true` の場合は **有効なまま** であり、フラグが明示的に
`false` の場合にのみ無効になります。これは意図的な動作です。このガードは、
無制限にトークンを消費しかねない Compaction ループから抜け出すために存在するため、
設定を行っていないユーザーにも保護が適用されます。

```json5
{
  tools: {
    loopDetection: {
      // マスタースイッチ。ローリング検出機能とともにガードを無効にするには false に設定する
      enabled: true,
    },
  },
}
```

- 結果が変化している間、ガードが実行を中止することはありません。ウィンドウ全体で
  バイト単位で同一の結果が得られた場合にのみ作動します。
- 実行中の他の時点ではなく、Compaction の再試行直後にのみ作動します。

<Note>
  マスターフラグが明示的に `false` でない限り、`tools.loopDetection` ブロックを一度も記述していなくても、Compaction 後ガードは動作します。確認するには、Compaction イベントの直後に Gateway ログで `post-compaction guard armed for N attempts` を探してください。
</Note>

## ログと想定される動作

ループが検出されると、OpenClaw はループイベントをログに記録し、重大度に応じて
次のツールサイクルに警告を出すかブロックします。これにより、通常のツールアクセスを維持しながら、
制御不能なトークン消費やロックアップを防ぎます。

- 最初に警告が出されます。
- パターンが警告しきい値を超えて継続すると、ブロックされます。
- 重大なしきい値に達すると次のツールサイクルをブロックし、実行記録に明確な
  ループ検出理由を表示します。
- Compaction 後ガードは、問題のあるツールと同一呼び出し回数を示す
  `compaction_loop_persisted` エラーを出力します。

## 関連項目

<CardGroup cols={2}>
  <Card title="実行の承認" href="/ja-JP/tools/exec-approvals" icon="shield">
    シェル実行の許可・拒否ポリシー。
  </Card>
  <Card title="思考レベル" href="/ja-JP/tools/thinking" icon="brain">
    推論エフォートのレベルとプロバイダーポリシーの相互作用。
  </Card>
  <Card title="サブエージェント" href="/ja-JP/tools/subagents" icon="users">
    制御不能な動作を制限するための分離エージェントの生成。
  </Card>
  <Card title="設定リファレンス" href="/ja-JP/gateway/config-tools#toolsloopdetection" icon="gear">
    `tools.loopDetection` の完全なスキーマとマージのセマンティクス。
  </Card>
</CardGroup>
