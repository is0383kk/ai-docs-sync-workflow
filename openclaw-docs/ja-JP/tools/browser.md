---
read_when:
    - エージェント制御のブラウザ自動化の追加
    - openclaw が自分の Chrome に干渉する原因のデバッグ
    - macOS アプリでのブラウザ設定とライフサイクルの実装
summary: 統合ブラウザ制御サービス + アクションコマンド
title: ブラウザ（OpenClaw 管理）
x-i18n:
    generated_at: "2026-07-26T09:46:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3afa2dda17520ae6c53fe3f1a7a12e7ca8a1414b2c12b79cf4a09ac8906bb3ca
    source_path: tools/browser.md
    workflow: 16
---

OpenClaw は、エージェントが制御する**専用の Chrome/Brave/Edge/Chromium プロファイル**を実行できます。これは Gateway 内部の小さなローカル制御サービス（ループバックのみ）を介して動作し、個人用ブラウザから分離されています。

- これは**エージェント専用の独立したブラウザ**と考えてください。`openclaw` プロファイルが個人用ブラウザプロファイルに触れることはありません。
- エージェントは、この分離された経路でタブを開き、ページを読み、クリックし、入力します。
- 一方、組み込みの `user` プロファイルは、Chrome DevTools MCP を介して、実際にサインイン済みの Chrome セッションに接続します。

## 利用できる機能

- **openclaw** という名前の独立したブラウザプロファイル（デフォルトではオレンジ色のアクセント）。
- 決定論的なタブ制御（一覧表示／開く／フォーカス／閉じる）。
- エージェント操作（クリック／入力／ドラッグ／選択）、スナップショット、スクリーンショット、PDF。
- Playwright ベースのプロファイルは、添付ファイルへの直接ナビゲーションを管理対象のダウンロードディレクトリに保存し、最終 URL のポリシー検証後に `{ url, suggestedFilename, path }` メタデータを返します。
- Playwright ベースのエージェント操作は、その操作によって 1 つ以上のダウンロードが直ちに開始された場合、同じ管理対象メタデータを含む `downloads` 配列を返します。
- ブラウザ Plugin が有効な場合に、スナップショット、
  安定したタブ、古くなった参照、手動ブロッカーからの復旧ループをエージェントに教える、
  バンドル済みの `browser-automation` skill。
- オプションのマルチプロファイル対応（`openclaw`、`work`、`remote`、...）。

このブラウザは、日常利用するためのものでは**ありません**。エージェントによる
自動化と検証のための、安全で分離された操作面です。

macOS では、Chrome 系のシステムプロファイルから独立した管理対象プロファイルへ、Cookie を明示的にコピーできます。管理対象ブラウザは引き続き独自のユーザーデータディレクトリを使用します。コピーされるのは選択した Cookie のみで、ローカルストレージと IndexedDB はコピーされません。インポートコマンドと制限事項については、[プロファイル](#profiles-multi-browser)または[`openclaw browser` CLI リファレンス](/ja-JP/cli/browser)を参照してください。

## クイックスタート

```bash
openclaw browser --browser-profile openclaw doctor
openclaw browser --browser-profile openclaw doctor --deep
openclaw browser --browser-profile openclaw status
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

「Browser disabled」は、Plugin または `browser.enabled` が無効であることを意味します。
[設定](#configuration)と[Plugin の制御](#plugin-control)を参照してください。

`openclaw browser` が完全に見つからない場合、またはエージェントがブラウザツールを
利用できないと報告する場合は、[ブラウザコマンドまたはツールが見つからない場合](#missing-browser-command-or-tool)に進んでください。

## Plugin の制御

デフォルトの `browser` ツールは、バンドル済みの Plugin です。同じ `browser` ツール名を登録する別の Plugin に置き換えるには、これを無効にします。

```json5
{
  plugins: {
    entries: {
      browser: {
        enabled: false,
      },
    },
  },
}
```

デフォルトでは、`plugins.entries.browser.enabled` と `browser.enabled=true` の**両方**が必要です。Plugin のみを無効にすると、`openclaw browser` CLI、`browser.request` Gateway メソッド、エージェントツール、制御サービスが一体として削除されます。置き換え用に `browser.*` 設定はそのまま維持されます。

ブラウザ設定を変更した場合、Plugin がサービスを再登録できるように Gateway の再起動が必要です。

## エージェント向けガイダンス

ツールプロファイルに関する注意: `tools.profile: "coding"` には `web_search` と
`web_fetch` が含まれますが、完全な `browser` ツールは含まれません。エージェントまたは
生成されたサブエージェントがブラウザ自動化を使用できるようにするには、プロファイル
段階で browser を追加します。

```json5
{
  tools: {
    profile: "coding",
    alsoAllow: ["browser"],
  },
}
```

単一のエージェントには `agents.entries.*.tools.alsoAllow: ["browser"]` を使用します。
サブエージェントのポリシーはプロファイルのフィルタリング後に適用されるため、
`tools.subagents.tools.allow: ["browser"]` だけでは不十分です。

ブラウザ Plugin には、2 段階のエージェント向けガイダンスが付属しています。

- `browser` ツールの説明には、常に有効な簡潔な契約が含まれています。適切なプロファイルを選択し、
  参照を同じタブ内に維持し、タブの対象指定には `tabId`/ラベルを使用し、
  複数ステップの作業ではブラウザ skill を読み込みます。
- バンドル済みの `browser-automation` skill には、より詳細な操作ループが含まれています。
  まずステータスとタブを確認し、タスク用タブにラベルを付け、操作前にスナップショットを取得し、UI の変更後に
  再度スナップショットを取得し、古くなった参照から一度だけ復旧を試み、ログイン／2FA／CAPTCHA や
  カメラ／マイクのブロッカーについては推測せず、手動操作が必要であると報告します。

Plugin にバンドルされた Skills は、Plugin が有効な場合、エージェントが利用可能な Skills の一覧に表示されます。
完全な skill の手順は必要に応じて読み込まれるため、通常の
ターンで完全なトークンコストが発生することはありません。

## ブラウザコマンドまたはツールが見つからない場合

アップグレード後に `openclaw browser` が認識されない、`browser.request` が見つからない、またはエージェントがブラウザツールを利用できないと報告する場合、通常の原因は、`browser` を含まない `plugins.allow` リストがあり、ルートの `browser` 設定ブロックが存在しないことです。次のように追加します。

```json5
{
  plugins: {
    allow: ["telegram", "browser"],
  },
}
```

明示的なルート `browser` ブロック（`browser` 配下の任意のキー。たとえば
`browser.enabled=true` または `browser.profiles.<name>`）は、制限的な `plugins.allow` のもとでも
バンドル済みのブラウザ Plugin を有効化し、バンドル済みチャンネルの設定動作と一致します。
`plugins.entries.browser.enabled=true` と `tools.alsoAllow: ["browser"]` は、それだけでは
許可リストへの登録の代わりにはなりません。`plugins.allow` を完全に削除しても、デフォルトが復元されます。

## プロファイル: `openclaw`、`user`、`chrome`

- `openclaw`: 管理された分離ブラウザ（拡張機能は不要）。
- `user`: **実際にサインイン済みの Chrome** セッション用の、組み込み Chrome DevTools MCP 接続プロファイル。OpenClaw が初めて接続するとき、Chrome に処理をブロックする「Allow remote debugging?」
  プロンプトが表示されるため、誰かがコンピューターの前にいる必要があります。
- `chrome`: **実際にサインイン済みの Chrome** セッション用の、組み込み[Chrome 拡張機能](/ja-JP/tools/chrome-extension)プロファイル。
  リモートデバッグポートではなく OpenClaw ブラウザ拡張機能を介してタブを操作するため、
  「Allow remote debugging?」プロンプトは表示されません。そのため、デスクに誰もいなくても
  スマートフォンから動作します。

エージェントによるブラウザツール呼び出しでは、次のようにします。

- デフォルト: 分離された `openclaw` ブラウザを使用します。
- 既存のログイン済みセッションが重要で、ユーザーが**コンピューターから離れている**
  場合（Telegram、WhatsApp など）は、`profile="chrome"`（拡張機能）を優先します。
- 既存のログイン済みセッションが重要で、ユーザーが接続プロンプトを承認するため
  **コンピューターの前にいる**場合は、`profile="user"`（Chrome MCP）を優先します。
- 特定のブラウザモードを使用する場合、`profile` で明示的に上書きします。

管理対象モードをデフォルトにする場合は、`browser.defaultProfile: "openclaw"` を設定します。

## 設定

ブラウザ設定は `~/.openclaw/openclaw.json` にあります。

```json5
{
  browser: {
    enabled: true, // デフォルト: true
    evaluateEnabled: true, // デフォルト: true。false にすると act:evaluate（任意の JS）が無効になる
    ssrfPolicy: {
      // dangerouslyAllowPrivateNetwork: true, // 信頼できるプライベートネットワークへのアクセスでのみ明示的に有効化する
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    // cdpUrl: "http://127.0.0.1:18792", // 従来の単一プロファイル用オーバーライド
    tabCleanup: {
      enabled: true, // デフォルト: true
    },
    // snapshotDefaults: { mode: "efficient" }, // 呼び出し元が省略した場合のデフォルトのスナップショットモード
    defaultProfile: "openclaw",
    color: "#FF4500",
    headless: false,
    noSandbox: false,
    attachOnly: false,
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: {
        cdpPort: 18801,
        color: "#0066CC",
        headless: true,
        executablePath: "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome",
      },
      user: {
        driver: "existing-session",
        attachOnly: true,
        color: "#00AA00",
      },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
  },
}
```

`browser.snapshotDefaults.mode: "efficient"` は、呼び出し元が明示的な `snapshotFormat` または
`mode` を渡さなかった場合の、デフォルトの `snapshot`
抽出モードを変更します。呼び出しごとのスナップショットオプションについては、
[ブラウザ制御 API](/ja-JP/tools/browser-control)を参照してください。

### タブクリーンアップの所有権

セッションのタブクリーンアップは、OpenClaw ブラウザツールが
`action: "open"` で作成したタブにのみ適用されます。OpenClaw は、すでに開かれていたタブ、
ユーザーが開いたタブ、または所有権が不明なタブを引き継ぎません。
`browser.tabCleanup` ブロックは、プライマリセッションに対する定期的なアイドル状態および上限の
スイープを制御します。これを無効にしても、明示的なセッションライフサイクルのクリーンアップは無効になりません。

ホストローカルで開いた場合、安定したネイティブ CDP ターゲットとブラウザ
ID による所有権が共有 SQLite 状態に保存されます。これらのレコードは Gateway の
再起動後も維持され、`/new` およびその他のセッションライフサイクルクリーンアップの対象であり続けます。
セッションライフサイクルクリーンアップには、サブエージェント、Cron、ACP セッションの終了が含まれます。
ツール向けターゲットがネイティブ CDP ターゲットであるレコードは、再起動後も
アイドル状態およびセッションごとの上限スイープの対象であり続けます。Chrome MCP のターゲットハンドルは
プロセスローカルであるため、コールド状態の既存セッションレコードは、再起動後に安全に帰属できない
アクティビティに対してアイドルスイープを行うリスクを避け、ライフサイクルクリーンアップを待機します。
この永続的な経路は、OpenClaw 管理プロファイル、通常のリモート CDP プロファイル、および明示的な
`cdpUrl` を持つ既存セッションプロファイルを対象にできます。ただし、OpenClaw がネイティブターゲットと安定した
ブラウザ ID の両方を解決できる必要があります。永続レコードを閉じる前に、OpenClaw は
設定されたプロファイルとブラウザインスタンスが引き続き一致していることを確認します。

Chrome MCP `--autoConnect`、`/json/version` の応答に安定したブラウザ ID がない
CDP エンドポイント、およびネイティブターゲットを解決できないオープン操作は、
プロセスローカルのベストエフォート追跡のままです。その Gateway プロセスが実行中であれば
クリーンアップできますが、Gateway の再起動後に自動的に閉じられることはありません。
永続追跡が利用可能になる前から開いていたタブは、遡及的に引き継がれません。そのようなタブは手動で閉じてください。

クリーンアップはベストエフォートであり、対象となるすべてのタブが
直ちに閉じられることを保証するものではありません。一時的な所有権確認またはクローズの失敗が発生した場合、永続的な
クリーンアップは後で再試行できるよう保留されます。再試行は無制限ではありません。ブラウザに
到達できない状態が続き、タブが 1 日を超えて使用されていない場合、その追跡行は
廃止され、二度と検証できないタブで永続ストアが埋まらないようにします。

### スクリーンショットの視覚認識（テキスト専用モデルのサポート）

メインモデルがテキスト専用（視覚／マルチモーダル非対応）の場合、ブラウザの
スクリーンショットは、そのモデルが読み取れない画像ブロックを返します。ブラウザのスクリーンショットは
既存の画像理解設定を再利用するため、メディア理解用に設定された画像モデルは、
ブラウザ固有のモデル設定なしでスクリーンショットをテキストとして説明できます。

```json5
{
  tools: {
    media: {
      image: {
        models: [
          { provider: "bytedance", model: "doubao-seed-2.0-pro" },
          // フォールバック候補を追加。最初の成功が採用される
          { provider: "openai", model: "gpt-4o" },
        ],
      },
      // 共有メディアモデルも、画像対応としてタグ付けされていれば使用できる。
      // models: [{ provider: "openai", model: "gpt-4o", capabilities: ["image"] }],
    },
  },
  agents: {
    defaults: {
      // 既存の画像モデルのデフォルトも適用される。
      // imageModel: { primary: "openai/gpt-4o" },
    },
  },
}
```

**仕組み:**

1. エージェントが `browser screenshot` を呼び出すと、通常どおり画像がディスクに保存されます。
2. ブラウザツールは、設定済みのメディア画像モデル、共有メディアモデル、画像モデルのデフォルト、または認証に基づく画像プロバイダーを使用してスクリーンショットを説明できるかどうかを、既存の画像理解ランタイムに問い合わせます。
3. ビジョンモデルはテキストの説明を返します。これは `wrapExternalContent`（プロンプトインジェクションガード）でラップされ、画像ブロックではなくテキストブロックとしてエージェントに返されます。
4. 画像理解が利用できない場合、スキップされた場合、または失敗した場合、ブラウザは元の画像ブロックを返すようフォールバックします。

スクリーンショットの画像ブロックは非公開のツール結果です。エージェントはそれらを確認できますが、OpenClaw がチャンネルへの返信に自動的に添付することはありません。スクリーンショットを共有するには、メッセージツールを使用して明示的に送信するようエージェントに依頼してください。

モデルのフォールバック、タイムアウト、バイト制限、プロファイル、プロバイダーのリクエスト設定には、既存の `tools.media.image` / `tools.media.models` フィールドを使用してください。

アクティブなメインモデルがすでにビジョンをサポートしており、明示的な画像理解モデルが設定されていない場合、OpenClaw は通常の画像結果を維持し、メインモデルがスクリーンショットを直接読み取れるようにします。

<AccordionGroup>

<Accordion title="ポートと到達可能性">

- 制御サービスは、`gateway.port` から派生したポート（デフォルトは `18791` = Gateway + 2）のループバックにバインドします。`OPENCLAW_GATEWAY_PORT` は `gateway.port` より優先され、どちらを使用しても同じポート系列の派生ポートが移動します。
- ローカルの `openclaw` プロファイルは、制御ポートの9ポート上から始まる範囲（デフォルトは `18800`～`18899`）から `cdpPort`/`cdpUrl` を自動割り当てします。これらを設定するのは、リモート CDP プロファイルまたは既存セッションのエンドポイントへの接続の場合だけにしてください。`cdpUrl` が未設定の場合、管理対象のローカル CDP ポートがデフォルトになります。
- リモートおよび `attachOnly` の CDP 到達可能性、WebSocket ハンドシェイク、ローカルの管理対象 Chrome の起動には、組み込みの期限が使用されます。
- 管理対象 Chrome の起動または準備完了が繰り返し失敗すると、プロファイルごとにサーキットブレーカーが作動します。数回連続して失敗すると、ブラウザツールの呼び出しごとに Chromium を起動する代わりに、OpenClaw は新しい起動試行を短時間停止します。起動時の問題を修正するか、ブラウザが不要な場合は無効にするか、修復後に Gateway を再起動してください。

</Accordion>

<Accordion title="SSRF ポリシー">

- ブラウザのナビゲーションおよびタブを開くリクエストは、事前チェックされます。アクションの実行中および制限されたアクション後の猶予期間中、保護された Playwright 操作（クリック、座標クリック、ホバー、ドラッグ、スクロール、選択、キー入力、テキスト入力、フォーム入力、evaluate）は、HTTP リクエストのバイトが送信される前に、ポリシーで拒否されたトップレベルおよびサブフレームのドキュメント読み込みを遮断し、その後、最終的な `http(s)` URL をベストエフォートで再チェックします。
- OpenClaw が管理する Chrome を新たに起動するたびに、OpenClaw はベストエフォートでネットワーク予測を無効化し、拒否された読み込みに対して Chromium で確認されている投機的な事前接続を抑止します。これは多層防御であり、ポリシー境界ではありません。制御サービスの再起動をまたいで再利用されるブラウザや、その他のブラウザバックエンドには、この強化策が適用されない可能性があります。Playwright のルーティングは依然としてネットワークファイアウォールではなく、リダイレクトの各段階、ポップアップの最初のリクエスト、Service Worker のトラフィック、制限されたガード期間後に実行されるページコード、またはすべてのバックグラウンド／サブリソース経路を遮断するものではありません。完全な外向き通信の分離には、所有者側での分離またはポリシーを適用するプロキシが必要です。
- 厳格な SSRF モードでは、リモート CDP エンドポイントの検出および `/json/version` プローブ（`cdpUrl`）もチェックされます。
- Gateway／プロバイダーの `HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY`、および `NO_PROXY` 環境変数によって、OpenClaw 管理ブラウザが自動的にプロキシ経由になることはありません。プロバイダーのプロキシ設定によってブラウザの SSRF チェックが弱まらないように、管理対象 Chrome はデフォルトで直接接続を使用して起動します。
- OpenClaw が管理するローカル CDP の準備完了プローブおよび DevTools WebSocket 接続は、起動された正確なループバックエンドポイントについて管理対象ネットワークプロキシを迂回するため、オペレーターのプロキシがループバックへの外向き通信をブロックしている場合でも `openclaw browser start` は機能します。
- 管理対象ブラウザ自体をプロキシ経由にするには、`browser.extraArgs` を通じて `--proxy-server=...` や `--proxy-pac-url=...` などの明示的な Chrome プロキシフラグを渡してください。厳格な SSRF モードでは、プライベートネットワークへのブラウザアクセスが意図的に有効化されていない限り、明示的なブラウザのプロキシルーティングがブロックされます。
- `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork` はデフォルトで無効です。プライベートネットワークへのブラウザアクセスを意図的に信頼する場合にのみ有効にしてください。
- `browser.ssrfPolicy.allowPrivateNetwork` は従来のエイリアスとして引き続きサポートされます。

</Accordion>

<Accordion title="プロファイルの動作">

- `attachOnly: true` は、ローカルブラウザを起動せず、すでに実行中のブラウザがある場合にのみ接続することを意味します。
- `headless` は、グローバルまたはローカルの管理対象プロファイルごとに設定できます。プロファイルごとの値は `browser.headless` を上書きするため、ローカルで起動した1つのプロファイルをヘッドレスのままにし、別のプロファイルを表示状態にできます。
- `POST /start?headless=true` および `openclaw browser start --headless` は、`browser.headless` またはプロファイル設定を書き換えることなく、ローカルの管理対象プロファイルに対して1回限りのヘッドレス起動を要求します。既存セッション、接続専用、およびリモート CDP プロファイルでは、OpenClaw がそれらのブラウザプロセスを起動しないため、この上書きは拒否されます。
- `DISPLAY` または `WAYLAND_DISPLAY` のない Linux ホストでは、環境またはプロファイル／グローバル設定で表示モードが明示的に選択されていない場合、ローカルの管理対象プロファイルは自動的にヘッドレスをデフォルトとします。曖昧さのないブラウザレベルの形式 `openclaw browser --json status` を使用してください。`status` は独自の `--json` を定義していないため、末尾の `openclaw browser status --json` も機能します。このコマンドは、`headlessSource` を `env`、`profile`、`config`、`request`、`linux-display-fallback`、または `default` として報告します。
- `OPENCLAW_BROWSER_HEADLESS=1` は、現在のプロセスにおけるローカルの管理対象ブラウザの起動を強制的にヘッドレスにします。`OPENCLAW_BROWSER_HEADLESS=0` は通常の起動を強制的に表示モードにし、ディスプレイサーバーのない Linux ホストでは対処方法を示すエラーを返します。ただし、明示的な `start --headless` リクエストは、その1回の起動について引き続き優先されます。
- ブラウザ制御ルートとプログラムクライアントは、ディスプレイがない場合のエラーについて、人間が読める `error` を維持し、安定した理由 `no_display_for_headed_profile` を公開します。その `details` には `profile`、`requestedHeadless`、`headlessSource`、および `displayPresent` のみが含まれるため、API クライアントはメッセージテキストを照合せずに正しい修復方法を選択できます。
- 実行中のローカル管理対象プロファイルに対し、status と doctor は Chrome のブラウザレベル CDP エンドポイントを照会し、レンダラー、バックエンド、デバイス／ドライバー、機能の状態、ドライバーの回避策、アクセラレーション動画機能を取得します。結果はそのブラウザプロセスについてキャッシュされ、`openclaw browser --json status` によって完全な形で公開されます。受動的な status 呼び出しでは Chrome は起動されません。既存セッション、拡張機能、リモート CDP、およびサンドボックスブラウザは引き続き別扱いとなり、この管理対象ホスト経路を通じて検査されることはありません。
- ヘッドレスの管理対象 Chrome でも、保守的な `--disable-gpu` のデフォルトが引き続き使用されます。診断によってアクセラレーションが有効化されたり、グローバルなアクセラレーション設定が追加されたり、サンドボックスブラウザにデバイスアクセスが付与されたりすることはありません。
- `executablePath` は、グローバルまたはローカルの管理対象プロファイルごとに設定できます。プロファイルごとの値は `browser.executablePath` を上書きするため、管理対象プロファイルごとに異なる Chromium ベースのブラウザを起動できます。どちらの形式でも、OS のホームディレクトリを表す `~` を使用できます。
- `color`（トップレベルおよびプロファイルごと）はブラウザ UI に色を付け、どのプロファイルがアクティブかを確認できるようにします。
- デフォルトプロファイルは `openclaw`（管理対象のスタンドアロン）です。サインイン済みのユーザーブラウザを使用するには、`defaultProfile: "user"` を使用してください。
- 自動検出の順序：Chromium ベースの場合はシステムのデフォルトブラウザ、それ以外は Chrome、Brave、Edge、Chromium、Chrome Canary。
- `driver: "existing-session"` は、生の CDP の代わりに Chrome DevTools MCP を使用します。Chrome MCP の自動接続を通じて接続するか、実行中のブラウザ用の DevTools エンドポイントがすでにある場合は `cdpUrl` を通じて接続できます。
- `driver: "extension"` は、[OpenClaw Chrome 拡張機能](/ja-JP/tools/chrome-extension)を通じてサインイン済みの Chrome を操作します。リレーが自身のループバックエンドポイントを所有するため、これらのプロファイルは `cdpUrl` を受け付けません。これは、コンピューターの前に誰もいない状態でも機能する唯一のサインイン済みブラウザモードです。
- 既存セッションのプロファイルをデフォルト以外の Chromium ユーザープロファイル（Brave、Edge など）に接続する場合は、`browser.profiles.<name>.userDataDir` を設定してください。この経路では、OS のホームディレクトリを表す `~` も使用できます。

</Accordion>

</AccordionGroup>

## Brave または別の Chromium ベースのブラウザを使用する

**システムのデフォルト**ブラウザが Chromium ベース（Chrome／Brave／Edge など）の場合、OpenClaw はそれを自動的に使用します。自動検出を上書きするには、`browser.executablePath` を設定してください。トップレベルおよびプロファイルごとの `executablePath` 値では、OS のホームディレクトリを表す `~` を使用できます。

```bash
openclaw config set browser.executablePath "/usr/bin/google-chrome"
openclaw config set browser.profiles.work.executablePath "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
```

または、プラットフォームごとに設定内で指定します。

<Tabs>
  <Tab title="macOS">
```json5
{
  browser: {
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
  },
}
```
  </Tab>
  <Tab title="Windows">
```json5
{
  browser: {
    executablePath: "C:\\Program Files\\BraveSoftware\\Brave-Browser\\Application\\brave.exe",
  },
}
```
  </Tab>
  <Tab title="Linux">
```json5
{
  browser: {
    executablePath: "/usr/bin/brave-browser",
  },
}
```
  </Tab>
</Tabs>

プロファイルごとの `executablePath` は、OpenClaw が起動するローカルの管理対象プロファイルにのみ影響します。`existing-session` プロファイルは代わりにすでに実行中のブラウザへ接続し、リモート CDP プロファイルは `cdpUrl` の接続先ブラウザを使用します。

## ローカル制御とリモート制御

- **ローカル制御（デフォルト）：** Gateway がループバック制御サービスを開始し、ローカルブラウザを起動できます。
- **リモート制御（Node ホスト）：** ブラウザがあるマシンで Node ホストを実行します。Gateway はブラウザ操作をそのホストにプロキシします。
- **リモート CDP：** リモートの Chromium ベースブラウザに接続するには、`browser.profiles.<name>.cdpUrl`（または `browser.cdpUrl`）を設定します。この場合、OpenClaw はローカルブラウザを起動しません。
- ループバック上の外部管理 CDP サービス（たとえば Docker で `127.0.0.1` に公開された Browserless）の場合は、`attachOnly: true` も設定してください。`attachOnly` のないループバック CDP は、OpenClaw が管理するローカルブラウザプロファイルとして扱われます。
- `headless` は、OpenClaw が起動するローカルの管理対象プロファイルにのみ影響します。既存セッションまたはリモート CDP ブラウザを再起動したり変更したりすることはありません。
- `executablePath` にも、同じローカル管理対象プロファイルのルールが適用されます。実行中のローカル管理対象プロファイルでこれを変更すると、そのプロファイルは再起動／再調整の対象としてマークされ、次回の起動で新しいバイナリが使用されます。

停止時の動作は、プロファイルモードによって異なります。

- ローカル管理対象プロファイル：`openclaw browser stop` は、OpenClaw が起動したブラウザプロセスを停止します
- 接続専用およびリモート CDP プロファイル：`openclaw browser stop` は、OpenClaw がブラウザプロセスを起動していなくても、アクティブな制御セッションを閉じ、Playwright／CDP のエミュレーション上書き（ビューポート、カラースキーム、ロケール、タイムゾーン、オフラインモード、および同様の状態）を解除します

リモート CDP URL には認証情報を含めることができます。

- クエリトークン（例：`https://provider.example?token=<token>`）
- HTTP Basic 認証（例：`https://user:pass@provider.example`）

OpenClaw は、`/json/*` エンドポイントの呼び出し時および
CDP WebSocket への接続時に認証情報を保持します。トークンを設定ファイルに
コミットするのではなく、環境変数またはシークレットマネージャーを使用してください。

## Node ブラウザプロキシ（設定不要のデフォルト）

ブラウザがあるマシンで **Node ホスト**を実行すると、OpenClaw は追加のブラウザ設定なしで、
ブラウザツールの呼び出しをその Node に自動的にルーティングできます。
これはリモート Gateway のデフォルト経路です。

注記:

- Node ホストは、ローカルのブラウザ制御サーバーを**プロキシコマンド**経由で公開します。
- プロファイルは Node 自身の `browser.profiles` 設定（ローカルと同じ）から取得されます。
- プロキシコマンドは、`allowProfiles` の設定にかかわらず、永続的なプロファイル変更（`create-profile`、`delete-profile`、`reset-profile`）を許可しません。これらの変更は Node 上で直接行ってください。
- `nodeHost.browserProxy.allowProfiles` は任意です。従来のデフォルト動作を使用する場合は空のままにしてください。設定済みのすべてのプロファイルにプロキシ経由で引き続きアクセスできます。
- `nodeHost.browserProxy.allowProfiles` を設定すると、OpenClaw はこれを最小権限の境界として扱い、プロキシが対象にできるプロファイル名を制限します。
- 使用しない場合は無効にしてください:
  - Node 側: `nodeHost.browserProxy.enabled=false`
  - Gateway 側: `gateway.nodes.browser.mode="off"`（接続済みブラウザ Node を1つ選択するための `"auto"`、または明示的な Node パラメーターを必須にするための `"manual"` も指定できます）

## Browserless（ホスト型リモート CDP）

[Browserless](https://browserless.io) は、HTTPS および WebSocket 経由で
CDP 接続 URL を公開するホスト型 Chromium サービスです。OpenClaw はどちらの形式も使用できますが、
リモートブラウザプロファイルでは、Browserless の接続ドキュメントに記載されている
直接 WebSocket URL を使用するのが最も簡単です。

例:

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserless",
    profiles: {
      browserless: {
        cdpUrl: "wss://production-sfo.browserless.io?token=<BROWSERLESS_API_KEY>",
        color: "#00AA00",
      },
    },
  },
}
```

注記:

- `<BROWSERLESS_API_KEY>` を実際の Browserless トークンに置き換えてください。
- Browserless アカウントに対応するリージョンエンドポイントを選択してください（同社のドキュメントを参照）。
- Browserless から HTTPS ベース URL が提供された場合は、直接 CDP 接続用の
  `wss://` に変換するか、HTTPS URL のままにして OpenClaw に
  `/json/version` を検出させることができます。

### 同一ホスト上の Browserless Docker

Browserless を Docker でセルフホストし、OpenClaw をホスト上で実行する場合は、
Browserless を外部管理の CDP サービスとして扱います:

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserless",
    profiles: {
      browserless: {
        cdpUrl: "ws://127.0.0.1:3000",
        attachOnly: true,
        color: "#00AA00",
      },
    },
  },
}
```

`browser.profiles.browserless.cdpUrl` のアドレスは、
OpenClaw プロセスから到達可能でなければなりません。また、Browserless は到達可能な対応エンドポイントを通知する必要があります。
Browserless の `EXTERNAL` を、`ws://127.0.0.1:3000`、`ws://browserless:3000`、または安定したプライベート Docker
ネットワークアドレスなど、OpenClaw から到達可能な同じ WebSocket ベースに設定してください。
`/json/version` が、OpenClaw から到達できないアドレスを指す
`webSocketDebuggerUrl` を返す場合、CDP HTTP が正常に見えても WebSocket
アタッチは失敗します。

local loopback の Browserless プロファイルでは、`attachOnly` を未設定のままにしないでください。
`attachOnly` がない場合、OpenClaw は local loopback ポートをローカル管理のブラウザ
プロファイルとして扱い、そのポートは使用中だが OpenClaw が所有していないと報告する場合があります。

## 直接 WebSocket CDP プロバイダー

一部のホスト型ブラウザサービスは、標準の HTTP ベースの CDP 検出（`/json/version`）ではなく、
**直接 WebSocket** エンドポイントを公開します。OpenClaw は3種類の
CDP URL 形式を受け付け、適切な接続方式を自動的に選択します:

- **HTTP(S) 検出** - `http://host[:port]` または `https://host[:port]`。
  OpenClaw は `/json/version` を呼び出して WebSocket デバッガー URL を検出し、その後
  接続します。WebSocket へのフォールバックはありません。
- **直接 WebSocket エンドポイント** - `/devtools/browser|page|worker|shared_worker|service_worker/<id>`
  パスを持つ `ws://host[:port]/devtools/<kind>/<id>` または
  `wss://...`。
  OpenClaw は WebSocket ハンドシェイクを介して直接接続し、
  `/json/version` を完全にスキップします。
- **ベア WebSocket ルート** - `/devtools/...` パスを持たない
  `ws://host[:port]` または `wss://host[:port]`（例: [Browserless](https://browserless.io)、
  [Browserbase](https://www.browserbase.com)）。OpenClaw は最初に HTTP
  `/json/version` 検出を試みます（スキームを `http`/`https` に正規化）。
  検出で `webSocketDebuggerUrl` が返された場合はそれを使用し、それ以外の場合は
  ベアルートでの直接 WebSocket ハンドシェイクにフォールバックします。通知された
  WebSocket エンドポイントが CDP ハンドシェイクを拒否しても、設定されたベアルートが
  受け入れる場合、OpenClaw はそのルートにもフォールバックします。これにより、ローカルの Chrome を指すベア
  `ws://` でも接続できます。Chrome は `/json/version` から取得したターゲット固有の
  パスでのみ WebSocket アップグレードを受け入れる一方、ホスト型
  プロバイダーは、検出エンドポイントが Playwright CDP に適さない
  短期間有効な URL を通知する場合でも、ルート WebSocket エンドポイントを使用できます。

`openclaw browser doctor` は、実行時アタッチと同じ検出優先・WebSocket フォールバック
ロジックを使用するため、正常に接続できるベアルート URL が診断で
到達不能と報告されることはありません。

### Browserbase

[Browserbase](https://www.browserbase.com) は、組み込みの CAPTCHA 解決、ステルスモード、住宅用
プロキシを備えたヘッドレスブラウザを実行するためのクラウドプラットフォームです。

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserbase",
    profiles: {
      browserbase: {
        cdpUrl: "wss://connect.browserbase.com?apiKey=<BROWSERBASE_API_KEY>",
        color: "#F97316",
      },
    },
  },
}
```

注記:

- [サインアップ](https://www.browserbase.com/sign-up)し、[Overview dashboard](https://www.browserbase.com/overview) から **API Key**
  をコピーしてください。
- `<BROWSERBASE_API_KEY>` を実際の Browserbase API キーに置き換えてください。
- Browserbase は WebSocket 接続時にブラウザセッションを自動作成するため、
  手動でセッションを作成する必要はありません。
- 現在の無料枠の制限と有料プランについては、[料金](https://www.browserbase.com/pricing)を参照してください。
- 完全な API リファレンス、SDK ガイド、統合例については、
  [Browserbase ドキュメント](https://docs.browserbase.com)を参照してください。

### Notte

[Notte](https://www.notte.cc) は、組み込みのステルス機能、住宅用プロキシ、CDP ネイティブの
WebSocket Gateway を備えたヘッドレスブラウザを実行するためのクラウドプラットフォームです。

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "notte",
    profiles: {
      notte: {
        cdpUrl: "wss://us-prod.notte.cc/sessions/connect?token=<NOTTE_API_KEY>",
        color: "#7C3AED",
      },
    },
  },
}
```

注記:

- [サインアップ](https://console.notte.cc)し、コンソールの設定ページから **API Key** を
  コピーしてください。
- `<NOTTE_API_KEY>` を実際の Notte API キーに置き換えてください。
- Notte は WebSocket 接続時にブラウザセッションを自動作成するため、手動で
  セッションを作成する必要はありません。WebSocket が切断されると、
  セッションは破棄されます。
- 現在の無料枠の制限と有料プランについては、[料金](https://www.notte.cc/#pricing)を参照してください。
- 完全な API リファレンス、SDK ガイド、統合例については、
  [Notte ドキュメント](https://docs.notte.cc)を参照してください。

## セキュリティ

重要なポイント:

- ブラウザ制御は local loopback 限定です。アクセスは Gateway の認証または Node のペアリングを経由します。
- スタンドアロンの local loopback ブラウザ HTTP API は、**共有シークレット認証のみ**を使用します:
  Gateway トークンの Bearer 認証、`x-openclaw-password`、または設定済みの
  Gateway パスワードによる HTTP Basic 認証です。
- Tailscale Serve の ID ヘッダーと `gateway.auth.mode: "trusted-proxy"` は、
  このスタンドアロンの local loopback ブラウザ API を**認証しません**。
- ブラウザ制御が有効で共有シークレット認証が設定されていない場合、OpenClaw は
  起動時にブラウザ制御用の認証情報を自動生成して永続化します:
  `gateway.auth.mode` が `none` の場合はトークン、`trusted-proxy` の場合は
  パスワードです（プロセス外の local loopback クライアントが解決できるように、
  `gateway.auth.password` を介して永続化されます）。そのモード用の明示的な
  文字列認証情報がすでに設定されている場合、または
  `gateway.auth.mode` が `password` の場合、自動生成は行われません。
- 生成されたシークレットではなく、自身で管理する安定したシークレットを使用する場合は、
  `gateway.auth.token`、`gateway.auth.password`、`OPENCLAW_GATEWAY_TOKEN`、または
  `OPENCLAW_GATEWAY_PASSWORD` を明示的に設定してください。

リモート CDP のヒント:

- 可能な場合は、暗号化されたエンドポイント（HTTPS または WSS）と短期間有効なトークンを使用してください。
- 長期間有効なトークンを設定ファイルに直接埋め込まないでください。
- Gateway とすべての Node ホストをプライベートネットワーク（Tailscale）上に配置し、公開を避けてください。
- リモート CDP の URL/トークンはシークレットとして扱い、環境変数またはシークレットマネージャーを使用してください。

## プロファイル（複数ブラウザ）

OpenClaw は複数の名前付きプロファイル（ルーティング設定）をサポートします。プロファイルには次の種類があります:

- **OpenClaw 管理**: 独自のユーザーデータディレクトリと CDP ポートを持つ専用の Chromium ベースのブラウザインスタンス
- **リモート**: 明示的な CDP URL（別の場所で実行されている Chromium ベースのブラウザ）
- **既存のセッション**: Chrome DevTools MCP の自動接続を介した既存の Chrome プロファイル

デフォルト:

- `openclaw` プロファイルが存在しない場合は自動作成されます。
- `user` プロファイルは、Chrome MCP の既存セッションへのアタッチ用として組み込まれています。
- 既存セッションプロファイルは `user` 以外ではオプトインです。`--driver existing-session` で作成してください。
- ローカル CDP ポートはデフォルトで **18800-18899** から割り当てられます。
- プロファイルを削除すると、そのローカルデータディレクトリはゴミ箱に移動されます。

すべての制御エンドポイントは `?profile=<name>` を受け付け、CLI は `--browser-profile` を使用します。

## Chrome DevTools MCP を介した既存セッション

OpenClaw は、公式の Chrome DevTools MCP サーバーを介して、実行中の Chromium ベースの
ブラウザプロファイルにアタッチすることもできます。これにより、そのブラウザプロファイルですでに開いている
タブとログイン状態が再利用されます。

公式の背景情報とセットアップの参考資料:

- [Chrome for Developers: ブラウザセッションで Chrome DevTools MCP を使用する](https://developer.chrome.com/blog/chrome-devtools-mcp-debug-your-browser-session)
- [Chrome DevTools MCP README](https://github.com/ChromeDevTools/chrome-devtools-mcp)

組み込みプロファイル: `user`。別の名前、色、またはブラウザデータディレクトリを使用する場合は、
独自の既存セッションプロファイルを作成してください。

デフォルトでは、組み込みの `user` プロファイルは Chrome MCP の自動接続を使用し、
デフォルトのローカル Google Chrome プロファイルを対象にします。Brave、
Edge、Chromium、またはデフォルト以外の Chrome プロファイルには `userDataDir` を使用してください。`~` は OS のホーム
ディレクトリに展開されます:

```json5
{
  browser: {
    profiles: {
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
    },
  },
}
```

次に、対応するブラウザで以下を行います:

1. リモートデバッグ用のブラウザの検査ページを開きます。
2. リモートデバッグを有効にします。
3. ブラウザを実行したままにし、OpenClaw がアタッチするときに接続プロンプトを承認します。

一般的な検査ページ:

- Chrome: `chrome://inspect/#remote-debugging`
- Brave: `brave://inspect/#remote-debugging`
- Edge: `edge://inspect/#remote-debugging`

ライブアタッチのスモークテスト:

```bash
openclaw browser --browser-profile user start
openclaw browser --browser-profile user status
openclaw browser --browser-profile user tabs
openclaw browser --browser-profile user snapshot --format ai
```

成功時の状態:

- `status` は `driver: existing-session` を表示します
- `status` は `transport: chrome-mcp` を表示します
- `status` は `running: true` を表示します
- `tabs` は、すでに開いているブラウザータブを一覧表示します
- `snapshot` は、選択したライブタブから ref を返します

アタッチが機能しない場合の確認事項:

- 対象の Chromium ベースのブラウザーのバージョンが `144+` である
- そのブラウザーの検査ページでリモートデバッグが有効になっている
- ブラウザーにアタッチ同意プロンプトが表示され、それを承認した
- Chrome が明示的な `--remote-debugging-port` を指定して起動された場合は、Chrome MCP の自動接続に
  依存せず、`browser.profiles.<name>.cdpUrl` をその DevTools エンドポイントに設定する
- `openclaw doctor` は、古い拡張機能ベースのブラウザー設定を移行し、
  デフォルトの自動接続プロファイル向けに Chrome がローカルにインストールされているか確認しますが、
  ブラウザー側のリモートデバッグを有効にすることはできません

エージェントでの使用:

- ユーザーがログイン済みのブラウザー状態が必要な場合は、`profile="user"` を使用します。
- カスタムの既存セッションプロファイルを使用する場合は、その明示的なプロファイル名を渡します。
- ユーザーがコンピューターの前にいて、アタッチプロンプトを承認できる場合にのみ、このモードを選択します。
- Gateway または Node ホストは `npx chrome-devtools-mcp@latest --autoConnect` を起動できます。

注意事項:

- このパスは、ログイン済みのブラウザーセッション内で操作できるため、分離された
  `openclaw` プロファイルよりもリスクが高くなります。
- OpenClaw はこのドライバー用のブラウザーを起動せず、アタッチのみを行います。
- ここでは、OpenClaw は公式の Chrome DevTools MCP `--autoConnect` フローを使用します。
  `userDataDir` が設定されている場合、そのユーザーデータディレクトリを対象にするため、そのまま渡されます。
- 既存セッションは、選択したホスト上、または接続済みのブラウザー Node 経由でアタッチできます。
  Chrome が別の場所にあり、ブラウザー Node が接続されていない場合は、代わりにリモート CDP または Node ホストを使用します。
- Chrome MCP のターゲットとスナップショット ref は、1 つの MCP サブプロセスにスコープされます。
  そのプロセスが再起動した後は、`browser tabs` を再度実行し、ターゲット固有の作業を行う前に
  新しいターゲットを明示的に選択し、ref を使用する前に新しいスナップショットを取得します。
  各 ref は、そのターゲットと最新のスナップショットに対してのみ有効です。URL が一致していても、
  古いエイリアスは置き換え先のタブには引き継がれません。
- Chrome DevTools MCP は現在、プロセスローカルの数値ページ ID に基づいてページツールを
  ルーティングします。プロセススコープのハンドルにより、サブプロセス置換をまたいだ再利用は防止されますが、
  隣接するツール呼び出しの間にプロセス内でブラウザーコンテキストが置き換えられると、操作の対象が
  切り替わる可能性は残ります。完全にアトミックなルーティングには、安定したターゲット ID に対する
  アップストリームのページツールサポートが必要です。

### カスタム Chrome MCP 起動

デフォルトの `npx chrome-devtools-mcp@latest` フローが要件に合わない場合
（オフラインホスト、固定バージョン、ベンダー提供バイナリ）は、プロファイルごとに
起動される Chrome DevTools MCP サーバーを上書きします:

| フィールド        | 動作                                                                                                               |
| ------------ | -------------------------------------------------------------------------------------------------------------------------- |
| `mcpCommand` | `npx` の代わりに起動する実行可能ファイル。そのまま解決され、絶対パスも使用できます。                                          |
| `mcpArgs`    | `mcpCommand` にそのまま渡される引数配列。デフォルトの `chrome-devtools-mcp@latest --autoConnect` 引数を置き換えます。 |

既存セッションプロファイルに `cdpUrl` が設定されている場合、OpenClaw は
`--autoConnect` をスキップし、エンドポイントを Chrome MCP に自動的に転送します:

- `http(s)://...` → `--browserUrl <url>`（DevTools HTTP 検出エンドポイント）。
- `ws(s)://...` → `--wsEndpoint <url>`（直接接続する CDP WebSocket）。

エンドポイントフラグと `userDataDir` は併用できません。`cdpUrl` が設定されている場合、
Chrome MCP はプロファイルディレクトリを開くのではなく、エンドポイントの背後で実行中のブラウザーに
アタッチするため、Chrome MCP の起動時に `userDataDir` は無視されます。

<Accordion title="既存セッション機能の制限">

マネージド `openclaw` プロファイルと比べると、既存セッションドライバーにはより多くの制約があります:

- **スクリーンショット** - ページキャプチャと `--ref` 要素キャプチャは機能しますが、CSS `--element` セレクターは機能しません。ページまたは ref ベースの要素スクリーンショットに Playwright は不要です。（`--full-page` は既存セッションに限らず、どのプロファイルでも `--ref` または `--element` と併用できません。）
- **操作** - `click`、`type`、`hover`、`scrollIntoView`、`drag`、`select` にはスナップショット ref が必要です（CSS セレクターは使用できません）。`click-coords` は表示中のビューポート座標をクリックし、スナップショット ref は不要です。`click` は左ボタンのみに対応します（ボタンの上書きや修飾キーは使用できません）。`type` は `slowly=true` をサポートしていません。`fill` または `press` を使用してください。`press` は `delayMs` をサポートしていません。`type`、`hover`、`scrollIntoView`、`drag`、`select`、`fill` は、呼び出しごとの `timeoutMs` の上書きをサポートしていませんが、`evaluate` はサポートしています。`select` は単一の値を受け付けます。`batch` はサポートされていないため、操作を個別に送信してください。
- **待機 / アップロード / ダイアログ** - `wait --url` は完全一致、部分文字列、glob パターンをサポートします（マネージドと同じ）。`wait --load networkidle` は既存セッションプロファイルではサポートされません（マネージドおよび raw/リモート CDP プロファイルでは機能します）。アップロードフックには `ref` または `inputRef` が必要で、ファイルは一度に 1 つ、CSS `element` は使用できません。ダイアログフックでは、タイムアウトの上書きや `dialogId` はサポートされません。
- **ダイアログの可視性** - マネージドブラウザーの操作レスポンスには、操作によってモーダルダイアログが開かれた場合に `blockedByDialog` と `browserState.dialogs.pending` が含まれます。スナップショットにも保留中のダイアログ状態が含まれます。ダイアログが保留中の間は、`browser dialog --accept/--dismiss --dialog-id <id>` で応答してください。OpenClaw の外部で処理されたダイアログは、`browserState.dialogs.recent` の下に表示されます。
- **マネージド専用機能** - PDF エクスポート、ダウンロードのインターセプト、`responsebody` には、引き続きマネージドブラウザーパスが必要です。

</Accordion>

## 分離の保証

- **専用ユーザーデータディレクトリ**: 個人用ブラウザープロファイルには一切アクセスしません。
- **専用ポート**: 開発ワークフローとの競合を防ぐため、`9222` を回避します。
- **決定的なタブ制御**: `tabs` は、最初に `suggestedTargetId` を返し、続いて
  `t1` などの安定した `tabId` ハンドル、オプションのラベル、生の `targetId` を返します。
  エージェントは `suggestedTargetId` を再利用する必要があります。生の ID は、
  デバッグと互換性のために引き続き利用できます。

## ブラウザーの選択

ローカルで起動する場合、OpenClaw は最初に利用可能なものを選択します:

1. Chrome
2. Brave
3. Edge
4. Chromium
5. Chrome Canary

`browser.executablePath` で上書きできます。

プラットフォーム:

- macOS: `/Applications` と `~/Applications` を確認します。
- Linux: `/usr/bin`、`/snap/bin`、`/opt/google`、`/opt/brave.com`、`/usr/lib/chromium`、
  `/usr/lib/chromium-browser` の配下にある一般的な Chrome/Brave/Edge/Chromium の場所に加え、
  `PLAYWRIGHT_BROWSERS_PATH` または `~/.cache/ms-playwright` の配下にある Playwright 管理の Chromium を確認します。
- Windows: 一般的なインストール場所を確認します。

## 制御 API（オプション）

スクリプト作成とデバッグのために、Gateway は小規模な **local loopback 専用 HTTP
制御 API** と、対応する `openclaw browser` CLI（スナップショット、ref、拡張待機機能、
JSON 出力、デバッグワークフロー）を公開します。完全なリファレンスについては、
[ブラウザー制御 API](/ja-JP/tools/browser-control) を参照してください。

## トラブルシューティング

Linux 固有の問題（特に snap Chromium）については、
[ブラウザーのトラブルシューティング](/ja-JP/tools/browser-linux-troubleshooting) を参照してください。

WSL2 Gateway と Windows Chrome の分離ホスト構成については、
[WSL2 + Windows + リモート Chrome CDP のトラブルシューティング](/ja-JP/tools/browser-wsl2-windows-remote-cdp-troubleshooting) を参照してください。

### CDP 起動失敗とナビゲーションの SSRF ブロック

これらは異なる種類の障害であり、それぞれ異なるコードパスを示します。

- **CDP の起動または準備状態の失敗**は、OpenClaw がブラウザー制御プレーンの正常性を確認できないことを意味します。
- **ナビゲーションの SSRF ブロック**は、ブラウザー制御プレーンは正常ですが、ページナビゲーションの対象がポリシーによって拒否されたことを意味します。

一般的な例:

- CDP の起動または準備状態の失敗:
  - `Chrome CDP websocket for profile "openclaw" is not reachable after start`
  - `Remote CDP for profile "<name>" is not reachable at <cdpUrl>`
  - local loopback の外部 CDP サービスが `attachOnly: true` なしで設定されている場合の
    `Port <port> is in use for profile "<name>" but not by openclaw`
- ナビゲーションの SSRF ブロック:
  - `start` と `tabs` は引き続き機能する一方で、`open`、`navigate`、スナップショット、またはタブを開くフローがブラウザー/ネットワークポリシーエラーで失敗する

次の最小限の手順で両者を切り分けます:

```bash
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw tabs
openclaw browser --browser-profile openclaw open https://example.com
```

結果の読み方:

- `start` が `not reachable after start` で失敗した場合は、まず CDP の準備状態をトラブルシューティングします。
- `start` が成功しても `tabs` が失敗する場合、制御プレーンは依然として正常ではありません。ページナビゲーションの問題ではなく、CDP の到達性の問題として扱います。
- `start` と `tabs` が成功しても、`open` または `navigate` が失敗する場合、ブラウザー制御プレーンは稼働しており、障害はナビゲーションポリシーまたは対象ページにあります。
- `start`、`tabs`、`open` がすべて成功した場合、基本的なマネージドブラウザー制御パスは正常です。

重要な動作の詳細:

- `browser.ssrfPolicy` を設定していない場合でも、ブラウザー設定のデフォルトはフェイルクローズの SSRF ポリシーオブジェクトです。
- local loopback の `openclaw` マネージドプロファイルでは、OpenClaw 自身のローカル制御プレーンに対して、CDP ヘルスチェックがブラウザーの SSRF 到達性適用を意図的にスキップします。
- ナビゲーション保護は別です。`start` または `tabs` が成功しても、後続の `open` または `navigate` の対象が許可されることを意味しません。

セキュリティガイダンス:

- デフォルトではブラウザーの SSRF ポリシーを緩和**しないでください**。
- 広範なプライベートネットワークアクセスよりも、`hostnameAllowlist` や `allowedHostnames` などの限定的なホスト例外を優先してください。
- プライベートネットワークへのブラウザーアクセスが必要で、レビュー済みの意図的に信頼された環境でのみ、`dangerouslyAllowPrivateNetwork: true` を使用してください。

## エージェントツールと制御の仕組み

エージェントには、ブラウザー自動化用の **1 つのツール** が提供されます:

- `browser` - doctor/status/start/stop/tabs/open/focus/close/snapshot/screenshot/navigate/act

対応関係:

- `browser snapshot` は、安定した UI ツリー（AI または ARIA）を返します。
- `browser act` は、スナップショットの `ref` ID を使用して、クリック、入力、ドラッグ、選択を行います。
- `browser screenshot` は、ピクセル（ページ全体、要素、またはラベル付き参照）をキャプチャします。
- `browser doctor` は、Gateway、plugin、プロファイル、ブラウザー、タブの準備状態を確認します。
- `browser` には、以下を指定できます。
  - `profile`：名前付きブラウザープロファイル（openclaw、chrome、またはリモート CDP）を選択します。
  - `target`（`sandbox` | `host` | `node`）：ブラウザーの実行場所を選択します。
  - サンドボックス化されたセッションでは、`target: "host"` に `agents.defaults.sandbox.browser.allowHostControl=true` が必要です。
  - `target` を省略した場合、サンドボックス化されたセッションではデフォルトで `sandbox`、サンドボックス化されていないセッションではデフォルトで `host` が使用されます。
  - ブラウザー対応 Node が接続されている場合、`target="host"` または `target="node"` を固定しない限り、ツールはその Node に自動的にルーティングされることがあります。

これにより、エージェントの動作が決定論的に保たれ、壊れやすいセレクターを回避できます。

## 関連項目

- [ツールの概要](/ja-JP/tools) - 利用可能なすべてのエージェントツール
- [サンドボックス化](/ja-JP/gateway/sandboxing) - サンドボックス化された環境でのブラウザー制御
- [セキュリティ](/ja-JP/gateway/security) - ブラウザー制御のリスクと堅牢化
