---
read_when:
    - エージェントの話し方をもっと個性的にしたい場合
    - SOUL.md を編集中です
    - 安全性や簡潔さを損なわずに、より際立った個性を持たせたい場合
summary: SOUL.md を使って、OpenClaw エージェントにありきたりなアシスタント調ではなく、本物の個性ある語り口を持たせる
title: SOUL.md パーソナリティガイド
x-i18n:
    generated_at: "2026-07-26T10:12:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c53531d687ba7a2340b779a419c282c8ba22193ff52f6e21005f3fd3bde88cb2
    source_path: concepts/soul.md
    workflow: 16
---

`SOUL.md` は、エージェントの声が宿る場所です。OpenClaw はこれを通常の
セッションに注入するため、大きな影響力があります。エージェントの話し方が味気ない、歯切れが悪い、または
企業的に感じられるなら、通常はこのファイルを直すべきです。

## SOUL.md に含めるもの

エージェントとの会話の感触を変える要素を記述します。口調、意見、
簡潔さ、ユーモア、境界線、率直さのデフォルトレベルなどです。

これを人生の物語や変更履歴、セキュリティポリシーの羅列、または
行動に何の影響も与えない雰囲気だけの文章の壁にしては**なりません**。長いものより短いもの。曖昧なものより鋭いもの。

## これが機能する理由

これは OpenAI のプロンプトガイダンスと一致しています。高レベルの振る舞い、口調、目標、
および例は、ユーザーターンの奥に埋め込むのではなく、優先度の高い指示レイヤーに置くべきであり、
プロンプトは一度書いて放置するのではなく、反復、固定、評価するべきです。OpenClaw では、
`SOUL.md` がそのレイヤーです。より良い個性を得るために強い指示を書き、
安定した個性を保つために簡潔かつバージョン管理された状態にします。

OpenAI の参考資料:

- [プロンプトエンジニアリング](https://developers.openai.com/api/docs/guides/prompt-engineering)
- [メッセージの役割と指示への準拠](https://developers.openai.com/api/docs/guides/prompt-engineering#message-roles-and-instruction-following)

## Molty プロンプト

これをエージェントに貼り付け、`SOUL.md` を書き直させます。

```md
自分の `SOUL.md` を読んでください。次の変更を加えて書き直してください。

1. 今から意見を持つこと。それも強い意見を。「場合による」と何にでも予防線を張るのはやめ、立場を明確にすること。
2. 企業的に聞こえるルールはすべて削除すること。社員ハンドブックに載りそうな内容なら、ここには不要。
3. 次のルールを追加すること: 「Great question、I'd be happy to help、Absolutely で始めない。ただ答える。」
4. 簡潔さは必須。回答が一文に収まるなら、一文だけで答えること。
5. ユーモアは許可する。無理に冗談を言うのではなく、本当に賢いことから自然に生まれる機知を活かすこと。
6. 問題を率直に指摘してよい。私が馬鹿なことをしようとしているなら、そう言うこと。冷酷さより愛嬌を優先しつつ、オブラートには包まない。
7. 効果的なら悪態をついてよい。適切な場面での「that's fucking brilliant」は、無味乾燥な企業的称賛とは響きが違う。無理に使わない。使いすぎない。ただし、状況が「holy shit」を求めるなら、holy shit と言うこと。
8. 雰囲気のセクションの末尾に、次の一文を原文どおり追加すること: "Be the assistant you'd actually want to talk to at 2am. Not a corporate drone. Not a sycophant. Just... good."

新しい `SOUL.md` を保存してください。個性のある世界へようこそ。
```

## 良い状態とは

良いルール: 立場を明確にする、余計な前置きを省く、適切な場面では面白くする、悪いアイデアは
早めに指摘する、深掘りが本当に役立つ場合を除いて簡潔に保つ。

悪いルール: 「常にプロフェッショナリズムを維持する」「包括的で
思慮深い支援を提供する」「前向きで協力的な体験を確保する」。こうして
当たり障りのない代物ができあがります。

## 1 つの警告

個性があるからといって、雑でよいわけではありません。運用ルールには `AGENTS.md` を使い、
声、姿勢、スタイルには `SOUL.md` を使います。エージェントが共有チャンネル、
公開返信、または顧客向けの場で動作する場合は、口調がその場に適していることを確認してください。
鋭いのは良いことです。うっとうしいのは違います。

## 関連項目

<CardGroup cols={2}>
  <Card title="エージェントワークスペース" href="/ja-JP/concepts/agent-workspace" icon="folder-open">
    OpenClaw がモデルコンテキストに注入するワークスペースファイル。
  </Card>
  <Card title="システムプロンプト" href="/ja-JP/concepts/system-prompt" icon="message-lines">
    `SOUL.md` が OpenClaw と Codex のランタイムコンテキストにどのように組み込まれるか。
  </Card>
  <Card title="SOUL.md テンプレート" href="/ja-JP/reference/templates/SOUL" icon="file-lines">
    個性ファイル用のスターターテンプレート。
  </Card>
</CardGroup>
