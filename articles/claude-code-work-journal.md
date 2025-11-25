---
title: "Claude Code に作業ジャーナルを書かせる - 1週間ぶりでも30秒で復帰"
emoji: "📓"
type: "tech"
topics: ["claudecode", "claude", "ai", "開発効率化"]
published: false
---

## これはなに？

Claude Code にセッション終了時の作業ジャーナルを書かせる仕組みを紹介します。サイドプロジェクトを久しぶりに開いたときの「前回何やってたっけ」問題を解消できます。

実際のジャーナルと設定は GitHub で公開しています。
https://github.com/techtalkjp/slidecraft

## なぜジャーナルを書かせるのか

サイドプロジェクトを1週間ぶりに開いて「あれ、前回何やってたっけ」となった経験はないでしょうか。私はあります。毎回あります。

git log を見ればコミット履歴はわかります。でも「なぜその実装にしたのか」「どんな試行錯誤があったのか」「次に何をやろうとしていたのか」は残りません。コードの差分だけでは、1週間前の自分の思考を再現できないんですよね。

ジャーナルがあれば、久しぶりにプロジェクトを開いても「ああ、ここまでやったのか」とすぐ思い出せます。特に試行錯誤の経緯が残っているのが大きいです。「最初はこの方法を試したけど、うまくいかなかったからこっちにした」という流れがわかります。

もう一つの理由は、Claude Code へのコンテキスト共有です。作業を依頼するとき、前提となる情報を伝える必要があります。ジャーナルがあれば「前回のジャーナル読んで続きをやろう」と言えば済みます。前回のセッションで「次はこれをやろう」と決めていた内容がそのまま残っているので、指示を書く手間が省けます。

## ジャーナルの構造

ジャーナルは以下の構造で書かせています。

1. 私の発言をそのまま引用
2. Claude がその意図をどう解釈したかを記載
3. 実際に行った作業の詳細

実際のジャーナルから一例を示します。

> **ユーザーの依頼**: 「useEffect 使ってるところを重点的にコードレビューしたい。メモリリークとかないか」
>
> **意図の解釈**: React のメモリリーク問題は本番環境で検出が難しく、ユーザー体験に深刻な影響を与える可能性がある。特に useEffect でのクリーンアップ処理の欠如や、Promise の完了後に unmount されたコンポーネントで setState が実行される問題を防ぎたい。
>
> **作業内容**: 12ファイル23箇所を調査し、2箇所の重大な問題と1箇所の軽度な問題を特定・修正。Promise のメモリリークを防ぐための mounted フラグの追加、無限ループのリスクがある依存配列の修正など。

この記録があるから、1週間後に見ても「メモリリークのレビューをやって、この2箇所を直したのか」とわかります。git diff だけでは「なぜこの修正をしたのか」までは追えません。プルリクエストに都度ちゃんと書けばいいんでしょうけど、毎回やるのは大変ですよね。

実際のジャーナルはこちらです。
https://github.com/techtalkjp/slidecraft/blob/main/docs/journals/2025-11-24/journal.md

## CLAUDE.md への設定

Claude Code は CLAUDE.md というファイルを読んでプロジェクト固有のルールを理解します。ここにジャーナルの書き方を定義しておけば、セッション終了時に記録してくれます。

私が使っている設定を紹介します。

```markdown
## Documentation and Workflow

### Document Storage

All documentation is stored in date-based directories under `docs/journals/`:

- **Location**: `docs/journals/YYYY-MM-DD/`
- **Work Journal**: `docs/journals/YYYY-MM-DD/journal.md`
- **Other documents**: Research notes, analysis reports, guides in the same directory
- **File names**: Use alphanumeric + Japanese for clarity (e.g., `editor-mutation-action-migration-report.md`)

### Work Journal (Claude Code)

Record development sessions in `docs/journals/YYYY-MM-DD/journal.md`.

**When to record:**

- Session end indicators ("thanks", "done for today", etc.)
- After creating/updating 2+ documents
- After completing technical investigations or design documents
- On explicit user request
- **Per task**: Append each new task instruction as received

**Content structure:**

- **User instruction**: Quote user's request verbatim, add 1-2 sentence context
- **User intent (inferred)**: 2-3 sentences inferring user's goal in "wants to..." format
- **Work done**: Prose for complex work, bullets for simple tasks
- **Improvement suggestions**: At session end, note 2-3 ways user could have requested more efficiently (focus on request process, not AI reflection)

**Update method**: Append to existing journal for same date

### Decision Process

For critical decisions and user-facing deliverables, human review is required before finalization.
```

設定のポイントをいくつか解説します。

**Document Storage** では、ジャーナルだけでなくその日に作成した調査レポートや設計ドキュメントも同じ日付ディレクトリに保存させています。関連するドキュメントが一箇所にまとまるので、後から探しやすくなります。

**When to record** では、「ありがとう」「今日はここまで」などの発言をトリガーにジャーナルを書くよう指定しています。また Per task でタスクごとに追記させることで、長いセッションでも記録漏れを防いでいます。

**Improvement suggestions** は、セッション終了時に「こう依頼したらもっと効率的だった」という改善提案を書かせる設定です。これが意外と役に立ちます。自分の依頼の仕方の癖がわかるんですよね。

**Update method** で、同じ日付なら既存のジャーナルに追記させます。新規ファイルを作らないので、1日の作業がひとつのファイルにまとまります。

## 運用してみて

### 良かった点

**コンテキストスイッチが楽になった**。サイドプロジェクトに戻るときの「えーっと、どこまでやったっけ」がなくなりました。ジャーナルを読めば30秒で前回の状態に戻れます。自分で読まなくても「前回のジャーナル読んで」と言えば Claude が調べてくれるので、さらに楽です。

**試行錯誤の経緯が残る**。最終的なコードだけでなく、そこに至るまでの経緯がわかります。なぜその実装にしたのか、他にどんな選択肢があったのかが記録されています。

**言葉遣いが丁寧になる**。コーディングエージェントが思い通りに動いてくれないとき、つい口汚くなりがちなんですよね。でもジャーナルに自分の発言がそのまま引用されて残ると思うと、自然と丁寧な依頼になります。これが普段の癖になると人と話すときにも出そうで嫌だったので、良い抑止力になっています。

### 課題

**自動で書いてくれないことが多い**。CLAUDE.md に書いているだけでは Claude がジャーナルを書いてくれないことがよくあります。「ありがとう」と言ってもセッションが終わるだけで、結局「ジャーナル書いて」と自分で指示しないといけません。コミットした後にジャーナル更新を忘れていることに気づき、ジャーナル更新だけのコミットが発生することもあります。

**トークン消費が増える**。ジャーナルが長くなると読み込みでトークンを消費します。私は1日ごとにファイルを分けて、古いジャーナルは明示的に参照しない限り読ませないようにしています。また Claude は丁寧に書こうとするので、放っておくとジャーナルが冗長になりがちです。CLAUDE.md で簡潔さを求める指示を入れておくと安心です。

## まとめ

Claude Code に作業ジャーナルを書かせることで、サイドプロジェクトの「1週間ぶりに開いて何やってたか忘れる問題」が解消されました。やり方は CLAUDE.md にジャーナルの書き方ルールを書いておくだけです。

実際のジャーナルと CLAUDE.md の設定は GitHub で公開しているので、参考にしてみてください。
https://github.com/techtalkjp/slidecraft

---

このジャーナルを書いているプロジェクト「SlideCraft」は、AI生成スライドをピンポイント修正できるツールです。Napkin や Google NotebookLM で生成したスライドの、気になる部分だけを1枚単位で修正できます。

「30枚のうち3枚だけ直したいのに、再生成すると数分待つ上に完璧だった27枚まで変わってしまう」という問題を解決するために作りました。よければ試してみてください。
https://www.slidecraft.work
