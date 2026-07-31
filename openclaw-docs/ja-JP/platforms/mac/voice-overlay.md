---
read_when:
    - 音声オーバーレイの動作を調整する
summary: ウェイクワードとプッシュトゥトークが重複する場合の音声オーバーレイのライフサイクル
title: 音声オーバーレイ
x-i18n:
    generated_at: "2026-07-26T09:30:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eef571c3e8d41a97779537b1b373fab25b08f63575b50e5019f6c5fbcb782c52
    source_path: platforms/mac/voice-overlay.md
    workflow: 16
---

# 音声オーバーレイのライフサイクル（macOS）

対象読者：macOS アプリのコントリビューター。目標：ウェイクワードとプッシュトゥトークが重なったときでも、音声オーバーレイが予測可能に動作するようにする。

## 動作

- ウェイクワードによってオーバーレイがすでに表示されている状態でユーザーがホットキーを押すと、ホットキーセッションはテキストをリセットせず、既存のテキストを引き継ぐ。ホットキーを押している間、オーバーレイは表示されたままになる。キーを離した時点で、前後の空白を除去したテキストがあれば送信し、なければ閉じる。
- ウェイクワードのみの場合は、引き続き無音を検出すると自動送信する。プッシュトゥトークの場合は、キーを離すと即座に送信する。

## 実装

- `VoiceSessionCoordinator`（`apps/macos/Sources/OpenClaw/VoiceSessionCoordinator.swift`）は、アクティブな音声セッションを一元管理する。これはアクターではなく、`@MainActor @Observable` シングルトンである。API：`startSession`、`updatePartial`、`finalize`、`sendNow`、`dismiss`、`updateLevel`、`snapshot`。各セッションは `UUID` トークンを保持し、古いトークンまたは一致しないトークンを使用した呼び出しは破棄される。
- `VoiceWakeOverlayController`（`VoiceWakeOverlayController+Session.swift`）はオーバーレイを描画し、ユーザー操作（`requestSend`、`dismiss`）をセッショントークン経由でコーディネーターに転送する。セッション状態自体を管理することはない。
- プッシュトゥトーク（`VoicePushToTalk.begin()`）は、表示中のオーバーレイにあるテキストを（`VoiceSessionCoordinator.shared.snapshot()` 経由で）`adoptedPrefix` として引き継ぐ。そのため、ウェイクオーバーレイの表示中にホットキーを押してもテキストが維持され、新しい発話が追加される。キーを離すと、最終文字起こしを最大 1.5s 待ってから、現在のテキストを使用する。
- `dismiss` の際、オーバーレイは `VoiceSessionCoordinator.overlayDidDismiss` を呼び出し、それによって `VoiceWakeRuntime.refresh(state:)` がトリガーされる。このため、X による手動での閉じる操作、空テキストによる終了、送信後の終了のいずれの場合も、ウェイクワードの待ち受けが再開される。
- 統一された送信経路：前後の空白を除去したテキストが空なら閉じる。それ以外の場合は、`sendNow` が送信チャイムを一度再生し、`VoiceWakeForwarder` 経由で転送してから閉じる。

## ログ

音声サブシステムは `ai.openclaw` であり、各コンポーネントはそれぞれのカテゴリにログを記録する。

| カテゴリ                | コンポーネント                                       |
| ----------------------- | ----------------------------------------------- |
| `voicewake.coordinator` | `VoiceSessionCoordinator`                       |
| `voicewake.overlay`     | `VoiceWakeOverlayController`/`VoiceWakeOverlay` |
| `voicewake.ptt`         | プッシュトゥトークのホットキーと音声キャプチャ                 |
| `voicewake.runtime`     | ウェイクワードランタイム                               |
| `voicewake.chime`       | チャイムの再生                                  |
| `voicewake.sync`        | グローバル設定の同期                            |
| `voicewake.forward`     | 文字起こしの転送                           |
| `voicewake.meter`       | マイクレベルモニター                               |

## デバッグチェックリスト

- オーバーレイが消えずに残る問題を再現しながら、ログをストリーミングする。

  ```bash
  sudo log stream --predicate 'subsystem == "ai.openclaw" AND category CONTAINS "voicewake"' --level info --style compact
  ```

- アクティブなセッショントークンが 1 つだけであることを確認する。古いコールバックはコーディネーターによって破棄される。
- プッシュトゥトークのキーを離した際に、アクティブなトークンを指定して必ず `end()` が呼び出されることを確認する。テキストが空の場合は、チャイムの再生や送信を行わずに閉じることが期待される。

## 関連項目

- [macOS アプリ](/ja-JP/platforms/macos)
- [音声ウェイク（macOS）](/ja-JP/platforms/mac/voicewake)
- [トークモード](/ja-JP/nodes/talk)
