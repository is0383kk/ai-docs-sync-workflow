---
read_when:
    - Gateway エージェントがペアリング済みのデスクトップを表示し、操作できるようにする
    - コンピューター操作の有効化、権限、または安全性
    - computer.act Node コマンドまたはそのフルフィラーの拡張
summary: computer ツールおよび computer.act Node コマンドによる、ケイパビリティベースのデスクトップ制御
title: コンピューターの使用
x-i18n:
    generated_at: "2026-07-26T09:28:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: df8ce87e607ce1b22d91e4ed8702d500bccd4d4f59dab7b0eafac565e730d48a
    source_path: nodes/computer-use.md
    workflow: 16
---

コンピューター操作により、Gateway エージェントはペアリングされた高機能なデスクトップを認識し、制御できます。利用可否は機能に基づいて判定されます。接続された Node は `computer.act` と `screen.snapshot` の両方を通知する必要があり、後者の結果には `displayFrameId` が含まれている必要があります。ツールは参照フレームとしてスクリーンショットを取得し、危険な `computer.act` コマンドを介してポインターとキーボードを操作します。アクションセットは Anthropic の中核的なコンピューター操作アクションに準拠しています。オプションの `computer_20251124` ズームは公開されません。ビジョン対応モデルが、組み込みの `computer` エージェントツールを介して操作します。

エージェントが発行する統一コマンドは `computer.act` のみであり、Node がそれをどのように実行するかは判別できません。同梱の macOS アプリは、組み込みの Peekaboo サービスと限定的な CoreGraphics プリミティブを使用して、プロセス内でコマンドを処理します（適切な TCC 権限が必要で、追加プロセスは不要です）。Windows と Linux では、別途インストールした `cua-driver` バイナリとともに、オプションの実験的な `cua-computer` Plugin を使用できます。どちらの実行機構も、同じペアリングおよび有効化ポリシーを使用します。

## 要件

- `computer.act` と `screen.snapshot` の両方を通知し、`screen.snapshot` が `displayFrameId` を返す、ペアリング済みで接続中の Node。
- **macOS 実行機構:** アプリ設定の **Allow Computer Control** が有効になっていること（デフォルト: オフ）。
- **macOS 実行機構:** OpenClaw に **Accessibility** 権限（ポインターおよびキーボード入力用）と **Screen Recording** 権限（`screen.snapshot` 用）が付与されていること。
- **Windows/Linux 実行機構:** 同梱の `cua-computer` Plugin が有効で、互換性のある `cua-driver` 0.10.x 実行ファイルがインストールされていること。
- Gateway で `computer.act` コマンドが有効化されていること（危険なため、デフォルトでは無効です）。
- ビジョン対応のエージェントモデル。
- `computer` を公開するツールポリシー。デフォルトの `coding` プロファイルでは公開されません。`tools.alsoAllow` に `computer` を追加してください。サンドボックス化されたエージェントでは、`tools.sandbox.tools.alsoAllow` にも追加する必要があります。

## `computer` エージェントツール

組み込みの `computer` ツールは、呼び出しごとに 1 つのアクションを受け取ります。座標は直近のスクリーンショット内の非負整数ピクセルであり、Node がディスプレイ上のポイントに変換します。座標アクションでは、スクリーンショット結果の `frameId` をそのまま返す必要があり、明示的な `screenIndex` はそのフレームと一致する必要があります。また OpenClaw は、Node が発行したディスプレイ ID をスクリーンショットからアクションへ引き継ぐため、ディスプレイの再接続やジオメトリの変更が発生すると、同じインデックスを暗黙に再ターゲットせず、安全側に失敗します。これらの検査により、推測されたトークンや、別の配信済みフレームまたはディスプレイのトークンは拒否されます。トークンは鮮度を保証するものではありません。同じディスプレイでもキャプチャ後にアプリがピクセルを変更できるため、画面が変化した可能性がある場合は必ず新しいスクリーンショットを取得してください。

- 読み取り: `screenshot`。
- ポインター: `left_click`、`right_click`、`middle_click`、`double_click`、`triple_click`、`mouse_move`、`left_click_drag`（`startCoordinate` を指定）、`left_mouse_down`、`left_mouse_up`。
- スクロール: `scroll`。`scrollDirection`（`up|down|left|right`）と `scrollAmount`（ホイールの刻み数）を指定します。
- キーボード: `type`（テキスト）、`key`（`cmd+shift+t` や `Return` などのキーの組み合わせ）、`hold_key`（`text` のキーの組み合わせを `duration` 秒間押し続ける）。
- 待機: `wait`（`duration` 秒）。

修飾キーは、クリックおよびスクロールアクションの `text` フィールドで指定します（`shift`、`ctrl`、`alt`、`cmd`）。入力アクションの後、ツールは新しいスクリーンショットを返し、モデルが結果を確認できるようにします。コンピューター操作に対応する Node が複数接続されている場合は、`node` を明示的に渡してください。

スクリーンショットは **モデル専用** として保持され、チャットチャンネルへ自動配信されることはありません。画面上のすべての内容を信頼できない入力として扱ってください。ツールは、ユーザーの要求と矛盾する画面上の指示に従わないようモデルに警告します。

## Windows と Linux（実験的、cua-driver 経由）

同梱の `cua-computer` Plugin は、Windows および Linux の Node ホスト向けに実験的な実行機構を提供します。デフォルトでは無効であり、プレリリース版の 0.10.x ドライバー契約が必要です。

1. [アップストリームのリリース](https://github.com/trycua/cua/releases)から `cua-driver` 0.10.x バイナリをインストールし、`PATH` で利用できるようにします。別の場所にある実行ファイルを使用するには、`plugins.entries.cua-computer.config.driverPath` を設定します。
2. Plugin を有効にします。

   ```bash
   openclaw plugins enable cua-computer
   ```

3. 対話型デスクトップセッションから `openclaw node run` を起動します。Plugin は、最初のキャプチャまたはアクションを受信したときに、ローカルドライバーデーモンを遅延起動します。

この実行機構が現在制御できるのはプライマリディスプレイのみです。Linux では X11/XWayland が第一選択です。ネイティブ Wayland は引き続きアップストリームでのオプトインが必要です。Node を起動する前に `CUA_DRIVER_RS_ENABLE_WAYLAND` を自分で設定してください。OpenClaw が自動的に設定することはありません。KDE/KWin は、アップストリームのネイティブ Wayland 入力パスではサポートされていません。cua-driver 0.10.x には、デスクトップスコープのクロスプラットフォームな長押し契約がないため、`hold_key`、`left_mouse_down`、`left_mouse_up` は利用できません。修飾キーを押しながらのスクロールとドラッグは両方のプラットフォームで利用できず、修飾キーを押しながらのクリックは Linux では利用できません。`key` アクションは、名前付きキー、英字、および修飾キーの組み合わせ（例: `cmd+c` または `Return`）を受け付けます。数字キーと句読点キーは、ドライバーがキーボードレイアウト依存の Shift 状態を破棄するため拒否されます。代わりに、そのテキストを `type` アクションで送信してください。`type_text` ドライバー呼び出しの途中で、テキスト入力をキャンセルすることはできません。

cua-driver は安定したディスプレイ ID を報告しないため、フレームの認可はドライバー接続と、現在のプライマリディスプレイのジオメトリに関連付けられます。デーモンまたはセッションが再接続されると未使用のフレームは無効になりますが、接続を維持したまま、同じジオメトリの別のプライマリディスプレイへ置き換えられた場合は検出できません。この実行機構では、安定した単一ディスプレイのセッションを推奨します。

OpenClaw は、自身が管理する `mcp` および `serve` プロセスについて、cua-driver のテレメトリと更新確認を無効にします。ドライバーバイナリのダウンロードや更新は行いません。

### トラブルシューティング

`cua-computer` 実行機構は、型付きエラーコードをツール結果と Node ログに出力します。一般的なコードは次のとおりです。

| コード                                                 | 原因                                                                                                                                                           | 修正                                                                                                                                                                                                                                  |
| ---------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `COMPUTER_DRIVER_UNAVAILABLE`                        | `cua-driver` バイナリが `PATH` に存在しない（または `driverPath` が間違っている）、デーモンが時間内に準備完了にならなかった、あるいは Node が Windows/Linux ではありません。                 | `cua-driver` 0.10.x を `PATH` にインストールするか、`driverPath` を設定します。対話型デスクトップセッション内で `openclaw node run` を実行します。Linux では、X11 の `DISPLAY`（または `CUA_DRIVER_RS_ENABLE_WAYLAND` を設定した `WAYLAND_DISPLAY`）が存在することを確認してください。 |
| `COMPUTER_DRIVER_UNSUPPORTED`                        | 接続されたドライバーが `cua-driver` 0.10.x ではないか、機能またはスキーマのバージョンが異なります。                                                                      | サポートされている 0.10.x ビルドをインストールしてください。修正後、Plugin は約 30 秒後に再検査するため、Node の再起動は不要です。                                                                                                          |
| `COMPUTER_REFUSED_<code>`                            | ドライバーが `background_unavailable`、`background_occluded`、`foreground_unavailable`（KDE/KWin Wayland）などの構造化コードを返してアクションを拒否しました。   | 対象ウィンドウを前面に移動するか、X11 に切り替えるか、サポートされているコンポジターを使用してください。前述の互換性に関する注意事項を参照してください。                                                                                                                    |
| `COMPUTER_STALE_FRAME`                               | 座標が参照したスクリーンショットが最新ではありません（コンテキストの Compaction、ディスプレイジオメトリの変更、または参照幅の変更）。                 | 座標アクションの前に、新しい `screenshot` を取得してください。                                                                                                                                                                              |
| `COMPUTER_UNSUPPORTED_ACTION`                        | この実行機構では忠実に実行できないアクションです: `hold_key`、`left_mouse_down`、`left_mouse_up`、修飾キーを押しながらのドラッグやスクロール、または Linux での修飾キーを押しながらのクリック。 | サポートされているアクションを使用してください。cua-driver 0.10.x には、デスクトップスコープの長押し入力契約がありません。                                                                                                                                                  |
| `COMPUTER_UNSUPPORTED_DISPLAY`                       | プライマリ以外の `screenIndex`、キャプチャと画面のジオメトリの不一致、またはプライマリディスプレイの外にあるカーソル。                                                       | プライマリディスプレイのみを操作してください。                                                                                                                                                                                                      |
| `COMPUTER_UNSUPPORTED_KEY`                           | ドライバーが確実に再現できない `key` 値です。たとえば、Shift 状態がレイアウト依存の数字キーや句読点キー、または不明なキーです。                        | 代わりに、そのテキストを `type` アクションで送信してください。                                                                                                                                                                                    |
| `COMPUTER_DRIVER_ERROR` / `COMPUTER_INVALID_REQUEST` | ドライバーが構造化コードなしで失敗したか、アクション引数が不正です。                                                                            | ドライバーの状態を確認してスクリーンショットを再取得し、アクション引数を修正してください。                                                                                                                                                        |

## `computer.act` Node コマンド

`computer.act` は、ツールが入力を転送する単一の Node コマンドです（`node.invoke` と `command: "computer.act"` を使用）。このコマンドには次の特性があります。

- **デフォルトでは危険**: 組み込みの危険な Node コマンドとして一覧に含まれ、明示的に有効化されるまではランタイムの許可リストから除外されます。ただし、macOS、Windows、Linux のデスクトップ Node は、ペアリング時にこのコマンドを宣言できるため、この操作対象への承認は一度で済みます。
- **機能ベース**: ツールを使用するには、接続された Node が `computer.act` と `screen.snapshot` の両方を通知する必要があります。同梱の macOS アプリと、オプトイン式の実験的な `cua-computer` Plugin は、同じコマンドペアを実行します。

読み取りでは `screen.snapshot` を再利用します。2 つ目のキャプチャパスはありません。共有キャプチャコマンドについては、[カメラと画面の Node](/ja-JP/nodes/camera)を参照してください。

## 有効化とアーミング

1. プラットフォームのフルフィラーを有効にします。macOS では、**Settings → Allow Computer Control** を有効にしてから、**Settings → Permissions** で **Accessibility** と **Screen Recording** を許可します。Windows/Linux では、上記の実験的な `cua-computer` のセットアップに従います。
2. Gateway でペアリングの更新を承認します（新しいコマンドを実行すると再ペアリングが強制されます）。
3. 視覚機能を備えたエージェントにツールを公開します。デフォルトの `coding` プロファイルの場合：

   ```json5
   {
     tools: {
       alsoAllow: ["computer"],
       // サンドボックス化されたエージェントには、この第 2 のゲートも必要です：
       sandbox: { tools: { alsoAllow: ["computer"] } },
     },
   }
   ```

4. 制限時間を指定して `computer.act` を有効化します。`phone-control` Plugin は `computer` グループを公開します：

   ```text
   /phone arm computer 30m
   /phone status
   /phone disarm
   ```

   有効化には `operator.admin`（または所有者）が必要で、期限が切れると自動的に無効化されます。従来の `/phone arm all` グループでは、意図的にデスクトップ制御が除外されています。明示的な `computer` グループを使用してください。有効化で切り替わるのは Gateway が呼び出せる対象だけです。Node アプリでは、macOS の **Allow Computer Control**、Accessibility、Screen Recording を含む、プラットフォーム固有の設定と OS の権限が引き続き適用されます。

永続的に認可するには、`computer.act` を `gateway.nodes.commands.allow` に追加し、`gateway.nodes.commands.deny` **から削除します**。拒否リストが優先されます。永続的な認可は自動的に期限切れになりません。`/phone arm` より前から存在するエントリは、`/phone disarm` の後も残ります。一時的な許可が有効化されている間は、永続的な許可に変更しないでください。

認可は、有効化と使用に意図的に分けられています。`computer.act` の有効化または永続的な設定には、管理権限が必要です。
有効化されると、`operator.write` を持つ認証済みオペレーターは、許可が期限切れになるか無効化されるまで、`node.invoke` を介して `computer.act` を呼び出せます。
アクションごとの管理者チェックはありません。`computer.act` を宣言する Node を承認しても、後で有効化できるようにその操作対象が記録されるだけであり、それ自体では呼び出しは有効になりません。

## 安全性

- 認可する前に、すべてのレイヤー（ツールポリシー、Gateway のコマンドポリシー、Node アプリの設定、プラットフォームの権限）で許可されている必要があります。現在の macOS フルフィラーの場合、これには **Allow Computer Control**、Accessibility、Screen Recording が含まれます。有効化されると、期限切れまたは `/phone disarm` まで、アクションごとの確認なしでアクションが実行されます。
- macOS フルフィラーはテキストを 1 書記素ずつ入力するため、キャンセル、切断、一時停止、無効化、またはエンドポイントの置換が発生すると、次の書記素を入力する前に停止します。実験的な cua-driver フルフィラーでは、入力の途中で `type_text` 呼び出しをキャンセルできません。
- スクリーンショットはモデル専用であり、チャットに自動送信されることはありません（Issue [#44759](https://github.com/openclaw/openclaw/issues/44759)）。
- 画面の内容は信頼できないものとして扱ってください。プロンプトインジェクションが含まれている可能性があります。

## 他のデスクトップ制御経路との関係

これはエージェント駆動の経路です。PeekabooBridge ホスト、Codex Computer Use、および直接利用する `cua-driver` MCP との関係については、[Peekaboo ブリッジ](/ja-JP/platforms/mac/peekaboo)を参照してください。
