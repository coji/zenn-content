---
title: "takt で Codex・Cursor・Claude Code を協調させてみた ― 5回 ABORT して気づいた設計の急所"
emoji: "🎼"
type: "tech"
topics: ["takt", "claudecode", "cursor", "ai", "開発効率化"]
published: true
---

## これはなに？

takt という AI エージェントオーケストレーションツールを使って、自作 OSS に新機能を追加してみた体験記です。Codex CLI で仕様レビュー、Cursor で実装、Claude Code で受け入れ検査という3つのエージェントを1つのワークフローで協調させました。

実際の PR はこちらです。
https://github.com/coji/durably/pull/151

## takt とは

takt は複数の AI コーディングエージェントを順番に動かすためのツールです。「仕様レビュー → 実装 → テスト確認」のような一連の流れを YAML で定義しておくと、各工程を異なる AI エージェントに自動で割り振って実行してくれます。

https://github.com/nrslib/takt

takt ではこの一連の流れを「ピース」、各工程を「ムーブメント」と呼びます。工程ごとに Cursor、Codex CLI、Claude Code など好きなエージェントを指定できるのが特徴です。

正直なところ、takt のリポジトリを見ても最初はどうしたらいいかわかりませんでした。piece、movement、facet、persona、policy、instruction、knowledge、output contract...。機能がたくさんあるのはわかるけど、謎の設定を大量に書かないといけないようにしか見えなくて、とっつきにくかったです。

そんなとき、[@j5ik2o](https://x.com/j5ik2o) さんが X で [ai-tools](https://github.com/j5ik2o/ai-tools) という Claude Code プラグインを教えてくれました。takt のワークフロー作成や実行結果の分析を Claude Code に頼めるようになるスキル群です。これを入れたら、「こういうワークフローにしたい」と伝えるだけで Claude Code が YAML を書いてくれるようになって、一気にハードルが下がりました。

https://x.com/j5ik2o/status/2036770033242910876

## 今回やったこと

自作のジョブ実行フレームワーク [Durably](https://github.com/coji/durably) に `waitForRun(runId)` という API を追加しました。既存の run の完了を新しい run を作らずに待てるようにする機能です。

ワークフローは以下の5工程で設計しました。

```
仕様レビュー (Codex) → 実装 (Cursor) → 受け入れ検査 (Claude Code) → 修正 (Cursor) → コード簡素化 (Claude Code)
```

## ワークフローの定義

takt のワークフロー定義はこんな感じです。各工程で何をして、結果に応じて次にどこへ進むかを書きます。

```yaml
name: spec-implement-accept
initial_movement: spec-review
max_movements: 20

movements:
  - name: spec-review
    edit: false
    persona: architecture-reviewer  # → Codex にルーティング
    rules:
      - condition: 仕様が明確で実装可能
        next: implement
      - condition: 仕様に重大な問題があり実装不可能
        next: ABORT

  - name: implement
    edit: true
    persona: coder  # → Cursor にルーティング
    rules:
      - condition: 実装完了、バリデーションパス
        next: acceptance

  - name: acceptance
    edit: false
    persona: acceptor  # → Claude Code にルーティング
    rules:
      - condition: 全完了条件を満たしバリデーションパス
        next: simplify
      - condition: 完了条件未達成または問題あり
        next: fix
```

`persona` がどのエージェントに割り当てられるかは、別の設定ファイルで決めます。`coder` と書けば Cursor、`architecture-reviewer` なら Codex、設定にないペルソナはデフォルトの Claude Code が担当します。

```yaml
# .takt/config.yaml
provider: claude
model: claude-opus-4-6

persona_providers:
  coder:
    provider: cursor
    model: composer-2-fast
  architecture-reviewer:
    provider: codex
    model: gpt-5.4
```

## 5回連続 ABORT の罠

最初にハマったのが、仕様レビュー工程での Codex の挙動です。5回連続で「仕様に問題あり」と判定されてワークフロー全体が中断しました。

1回目は「PLAN.md が存在しない」、2回目は「API シグネチャが未定義」、3回目は「race condition テストが不足」、4回目は「戻り値型が甘い」、5回目は「コールバックのリプレイ仕様が未定義」。指摘自体はどれも正当なんですが、毎回新しい問題を見つけてくるので永遠に実装に進めません。

原因はワークフローの分岐設計にありました。

```yaml
rules:
  - condition: 仕様が明確で実装可能
    next: implement
  - condition: 仕様に問題あり、修正が必要
    next: ABORT  # ← これが問題
```

「OK → 次へ進む」か「NG → 全体中断」の二択しかありません。「軽微な改善提案はあるけど実装は進められる」という状態を表現できないんです。Codex は丁寧にレビューするほど何かしらの指摘が出るので、毎回中断行きになってしまいます。

修正後はこうしました。条件文で「軽微な指摘は OK 扱い」と明示します。

```yaml
rules:
  - condition: 仕様が明確で実装可能（指摘なし、または軽微な改善提案のみ）
    next: implement
  - condition: 仕様に重大な問題があり実装不可能
    next: ABORT
```

合わせて、レビュー結果の書き方のテンプレートにも判定基準を追記しました。

```markdown
判定基準:
- APPROVE: 指摘なし、または軽微な改善提案のみ（実装に支障なし）
- REJECT: 要件の矛盾、変更対象の致命的な漏れ等

軽微な改善提案は APPROVE として指摘事項に記載する。
実装者が判断できるレベルの曖昧さは REJECT にしない。
```

ただ、5回の中断は完全に無駄だったわけでもありません。Codex の指摘を毎回タスク仕様書に反映していった結果、既存 API との挙動の違い、エラー型の使い分け、競合状態のテストケース、コールバックの振る舞いなどが精緻に定義されていきました。この仕様の精度があったから、次の実装工程が一発で通ったとも言えます。とはいえ、同じことをもっと少ない回数でやれたはずなので、分岐設計は本当に大事です。

## 成功した実行

仕様レビューをスキップして実装から開始した6回目で、ようやく通りました。

```
実装 (Cursor, 約8分) → 受け入れ検査 (Claude Code, 約2分) → 簡素化 (Claude Code, 約1分) → 完了
```

約11分で500行の変更が出てきて、受け入れ検査も一発通過。修正ループは発生しませんでした。仕様書の精度が上がっていたおかげか、Cursor の出力がそのまま受け入れ検査を通っています。自分がやったのは仕様書の修正とワークフローの起動だけで、コードには一切手を触れていません。これはなかなか新鮮な体験でした。

ただし、結果をメインリポジトリに push する段階で失敗したので、ワークツリーからパッチを取り出して手動で適用しました。この辺りはまだ荒削りな部分です。

## よかったこと

一番大きかったのは、自分のプロジェクトの細かい内部仕様と実装の整合性チェックを自分でやらなくて済んだことです。

たとえば今回追加した API は、既存の似た API と内部ロジックを共通化する必要がありました。イベント購読と DB ポーリングの競合状態を防ぐ順序制御、キャンセル時に投げるエラー型の選択、「すでに完了済みの場合は即座に返す」といった細かい分岐が多く、新しい API を足すときの「既存コードとの整合性」を全部自分で追うのは正直しんどいです。

今回のワークフローだと、Codex が仕様段階でそういう穴を見つけ、Cursor が実装し、Claude Code が完了条件と突き合わせて検査してくれます。自分は Codex の指摘を読んで「確かにそうだな」と仕様書を直すだけで済みました。

## 反省点

一方で、PR を作った後に CodeRabbit（GitHub の AI レビューボット）に設計上の穴を指摘されました。takt のワークフロー内では誰も見つけてくれなかったものです。

takt のワークフローはタスク仕様書に書かれた完了条件をチェックする設計なので、仕様書に書いてないことは検査しません。CodeRabbit は仕様書なしでコードの変更差分だけを見て問題を指摘してきます。今回はたまたま補完関係になりましたが、仕様書の完了条件をもっと広く書いておけば takt 側で拾えたはずです。

## どのエージェントに何をやらせるか

今回は「Codex で仕様レビュー、Cursor で実装、Claude Code で受け入れ検査」としましたが、この組み合わせが最適かはまだわかりません。

Codex（gpt-5.4）は仕様の穴を見つける能力が高い一方で、とにかく細かいです。「ここの型は non-generic と明記すべき」「コールバックが live-only か replay かを仕様に書け」といった、実装者なら判断できるレベルの曖昧さも blocking として弾いてきます。仕様レビューには向いているけれど、工程の設計側でちゃんとコントロールしないと止まります。

逆に、受け入れ検査を Codex にやらせていたらどうなっていたか、実装を Claude Code にやらせたらどうだったか、といった比較はまだ試せていません。エージェントの性格とタスクの相性を探るのも、takt を使いこなすうえでの試行錯誤になりそうです。

## まとめ

takt を使ってみて、分岐設計が一番大事だと感じました。「OK か全体中断か」の二択だとレビュアーの厳密さが仇になります。どのエージェントに何をやらせるかも、性格を把握したうえで工程の設計とセットで考える必要がありそうです。

まだ荒削りな部分もありますが、ワークフロー定義が蓄積されていけば、次回以降はもっとスムーズに回せそうです。
