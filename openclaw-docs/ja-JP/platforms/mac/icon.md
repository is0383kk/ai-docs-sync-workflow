---
read_when:
    - メニューバーアイコンの動作を変更する
summary: macOS 上の OpenClaw のメニューバーアイコンの状態とアニメーション
title: メニューバーアイコン
x-i18n:
    generated_at: "2026-07-26T09:08:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8a38f1253f0c376ef2ce6c0ae339b67084c472c764964bcc7ad21e10133e2b47
    source_path: platforms/mac/icon.md
    workflow: 16
---

# メニューバーアイコンの状態

対象範囲: macOS アプリ（`apps/macos`）。レンダリング: `CritterIconRenderer.makeIcon(...)`。アニメーション／状態の配線: `CritterStatusLabel` + `CritterStatusLabel+Behavior.swift`。

## 状態

| 状態                  | トリガー                                  | 表示                                                                                              |
| --------------------- | ----------------------------------------- | --------------------------------------------------------------------------------------------------- |
| アイドル              | デフォルト                                | 通常のまばたき／小刻みな動きのアニメーション。開いた目には光沢のある輝きを維持                                        |
| 一時停止              | `isPaused=true`                           | 目を開いたまま触角が垂れる（「勤務外」）。動きなし                                               |
| スリープ              | Gateway が切断済み／未設定                | 触角が垂れ、目が閉じて `⌣ ⌣` のまぶたになる。動きなし                                            |
| お祝い                | メッセージ送信時（`sendCelebrationTick`）      | 目が約0.9秒間、嬉しそうな `∩ ∩` の弧に変わり、脚を蹴り上げる                                               |
| 音声ウェイク（大きな耳） | ウェイクワードを検出                    | 触角がまっすぐ高く立つ（`earScale=1.9`）。無音状態になると元に戻る                          |
| 作業中                | `isWorking=true` またはアクティブな `IconState` | 脚の小刻みな動きが高速化（`legWiggle` から最大 `1.0`）し、水平方向に少しずれる。アイドル時の小刻みな動きに加算される |

セッションにアクティブなジョブまたはツールがある場合、ツールアクティビティバッジ（SF Symbol のパック。例: exec の場合は `chevron.left.slash.chevron.right`）を同じクリッターアイコンの上に表示できます。このバッジは `IconState`/`ActivityKind` から提供されます。完全な状態モデルについては、[メニューバー](/ja-JP/platforms/mac/menu-bar)を参照してください。

## 音声ウェイクの耳

- トリガー: `AppStateStore.shared.triggerVoiceEars(ttl: nil)`。音声ウェイクのキャプチャパイプライン（`VoiceWakeRuntime`）と、音声ウェイクのデバッグ／テストツール（`VoiceWakeTester`、`VoiceWakeOverlayController`）から呼び出されます。
- 停止: `stopVoiceEars()`。キャプチャの完了時に呼び出されます。
- 完了までの無音時間: 通常は `2.0s`。トリガーワードだけが検出され、その後に発話が続かなかった場合は `5.0s`（`VoiceWakeRuntime.silenceWindow` / `triggerOnlySilenceWindow`）。
- ブースト中は、アイドル時のまばたき／小刻みな動き／脚／耳のタイマーが一時停止します（`earBoostActive` が `CritterStatusLabel+Behavior` のアニメーションタスクを制御します）。

## 形状とサイズ

- キャンバス: 18x18pt のテンプレート画像。Retina でアイコンを鮮明に保つため、36x36px のビットマップバッキングストア（2x）にレンダリングされます。
- 耳のスケールのデフォルトは `1.0`。音声ブーストでは、全体のフレームを変更せずに `earScale=1.9` を設定します。
- `antennaDroop`（0～1）は、一時停止およびスリープの姿勢で触角を下向きに折りたたみます。
- 脚の素早い動きには `legWiggle` から最大 `1.0` を使用し、水平方向に少し揺れます。

## 動作上の注意

- 耳や作業中状態を切り替える外部 CLI／ブローカーのトグルはありません。意図しないばたつきを避けるため、どちらもアプリのシグナル（`AppState.setWorking`、`AppState.triggerVoiceEars`）によって内部的に駆動されます。
- ジョブが停止した場合でもアイコンがすぐに基準状態へ戻るよう、新しい TTL は短く（10秒を十分に下回る値に）保ってください。

## 関連項目

- [メニューバー](/ja-JP/platforms/mac/menu-bar)
- [macOS アプリ](/ja-JP/platforms/macos)
