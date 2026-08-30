---
title: "ターミナルに戻らずにAIの成果物を修正依頼する — Claude Code/Codex/Cursor対応プレビューの設計"
emoji: "🖱️"
type: "tech"
topics: ["claudecode", "codex", "cursor", "cli", "ai"]
published: false
---

## これはなに？

Artifact Share は、AIエージェントが作ったHTMLやMarkdownをURLで共有するサービスです。そのCLIに `preview` というローカルコマンドを追加しました。ブラウザ上でファイルの要素をクリックして指摘を書くと、Claude Code・Codex・Cursor がファイルを直し、ブラウザが自動リロードで結果を見せます。動画が一番早いです。

@[youtube](ooC8CCVog9A)

ここから先は仕組みの話です。サインイン・アップロード不要のローカル機能で、実装は公開されています。

https://github.com/artifactshare/artifactshare/pull/198

核になる技術判断は2つです。

- エージェントを起こす合図が、3つのエージェントで全部別物
- 常駐デーモンなし。1ファイル1プロセス

## 全体像

`preview <file>` を実行すると、そのプロセス自体がローカルサーバーになります。指摘はまとめて1バッチで送信します。エージェントの1周は「指摘を取りに行く → 編集・保存 → 結果を報告する」の2コマンドです。

```mermaid
flowchart LR
    subgraph local["ローカルマシン"]
        file["対象ファイル\n.md / .html"]
        server["preview server\n1ファイル = 1プロセス"]
    end
    browser["ブラウザ\n127.0.0.1"]
    agent["Claude Code / Codex / Cursor"]

    file -- 配信 --> server
    server -- レンダリング --> browser
    browser -- "要素クリック / テキスト選択で指摘\nバッチ送信" --> server
    server -- 指摘を渡す --> agent
    agent -- 編集・保存 --> file
    agent -- 結果を報告 --> server
    server -- 自動リロード --> browser
```

公開URLの発行は viewer の「共有する」を押したときだけ。ローカルの指摘履歴は共有に含まれません。

## 常駐デーモンを持たない

中央の常駐プロセスはありません。`preview <file>` のコマンド自身がサーバーになり、Ctrl-Cで終わります。同じファイルへの2回目の `preview` は既存セッションへ再接続します。

用途を「1ファイルを何度も直す反復ループ」に絞ったからです。複数セッションの横断発見・多重起動のロック・死んだプロセスの回収といった、デーモンが背負う寿命管理が丸ごと要らなくなります。

## 指摘の場所を覚えておく: anchorの3種類

指摘の位置は3種類のanchorで保存します。

```ts
type PreviewAnchor =
  | { kind: 'artifact' }
  | {
      kind: 'text'
      state: 'attached' | 'orphaned'
      quotedText: string
      prefixText: string
      suffixText: string
      cssPath: string | null
    }
  | {
      kind: 'element'
      state: 'attached' | 'orphaned'
      selector: string
      label: string // 例: button "送信"
      contextText: string // クリック時点の周辺テキスト
    }
```

ポイントは `element` をCSSセレクタ単独にしないことです。エージェントの編集でセレクタの指す先がずれても、ラベルと周辺テキストを突き合わせて位置を再解決できます。再解決に失敗したら `orphaned` に落とし、誤った場所に紐づけません。座標(boundingBox)は編集でずれる値なので保存せず、表示のたびに計算します。

## 指摘は状態を持つ

指摘は5状態を移ります。draft側の遷移は人間だけ、requested以降の前進はエージェントだけ、と主体を固定しています。

```mermaid
stateDiagram-v2
    [*] --> draft: 人間が指摘を作成
    draft --> [*]: 人間が削除
    draft --> requested: 人間がバッチ送信
    requested --> in_progress: エージェントが着手
    in_progress --> resolved: エージェントが解決
    in_progress --> dismissed: エージェントが見送り
    resolved --> draft: 人間がreopen
    dismissed --> draft: 人間がreopen
```

遷移はバッチ単位です。エージェントは1バッチを1回の編集でまとめて直すので、1件ずつは刻みません。reopenのたびに `generation` という番号が増え、エージェントの報告はこの番号と紐づきます。古い番号の報告は無視されるので、重複報告しても二重適用されません。

## エージェントを次のターンへ進ませる、3つの別解

CLIの形は共通でも、「指摘が届いたことをエージェントに伝える」方法は3つとも別物です。理想は「届いた瞬間にエージェントが自分で動き出す」ことですが、そのために使える仕組みがエージェントごとに違います。この3エージェントで外部イベント待ちのCLIを作るなら、結論はこの整理です。

- Claude Code: バックグラウンドタスクの完了でターンが自動再開する。これが一番素直
- Codex: 自動再開はないが、実行中セッションのキューへ外部からイベントを注入できる
- Cursor: どちらもないので、ACPでセッションを常駐させて合図を送る
- 共通のフォールバック: 「指摘が届くまで待つ」コマンドをフォアグラウンドで実行させる(ターンは塞がる)

以下、それぞれの動きです。

### Claude Code: バックグラウンドタスクの完了を使う

Claude Codeはバックグラウンドタスクの完了時に、新しいターンを自動で開きます。そこで「指摘が届くまで待つ」コマンドをバックグラウンドに仕込んでおきます。指摘が届いた瞬間にコマンドが終了し、Claude Codeが起きてファイルを直しにいきます。修正後は次の待機を仕込み直してループが続きます。

```mermaid
sequenceDiagram
    participant H as 人間(ブラウザ)
    participant S as preview server
    participant C as Claude Code

    C->>S: 指摘が届くまで待機(バックグラウンド)
    Note over C: ターン終了
    H->>S: 指摘バッチを送信
    S-->>C: バッチを返す(待機が終了)
    Note over C: バックグラウンドタスク完了で新ターンが自動的に開く
    C->>S: 結果を報告
    C->>S: 次の待機を仕込み直し
```

### Codex: 実行中セッションのキューに直接差し込む

Codexにはバックグラウンド完了でターンを再開する仕組みがなく([openai/codex#32188](https://github.com/openai/codex/issues/32188))、Claude Codeの手は使えません。代わりに、実行中セッション(trusted thread)を検出して、そのキューへ通知を直接積みます。エージェントが指摘を取りに来るまでバッチは消費されず、セッションが終わっていても再開すれば受け取れます。

```mermaid
sequenceDiagram
    participant H as 人間(ブラウザ)
    participant S as preview server
    participant Q as Codexセッションのキュー
    participant X as Codex

    H->>S: 指摘バッチを送信
    S->>Q: キューへ投入(処理待ち)
    X->>S: 指摘を取りに行く
    S->>Q: 取り出し
    Q-->>X: バッチを返す
    X->>S: 結果を報告
```

### Cursor: 管理下のACPセッションへ固定プロンプトを送る

Cursorにも自動再開の仕組みはなく、IDEチャットは外部から起こす手段自体がありません。そこで、ACP(Agent Client Protocol)経由の専用セッションを1ワークスペースに1つ常駐させます。指摘が届くと、ランチャーが「バッチが届いた」という固定プロンプトだけをそのセッションへ送り、本文はエージェントが自分で取りに行きます。使用中なら通知は保留され、バッチは保存されたまま残ります。

```mermaid
sequenceDiagram
    participant H as 人間(ブラウザ)
    participant S as preview server
    participant L as 管理下ランチャー(ACP)
    participant Y as Cursor

    H->>S: 指摘バッチを送信
    S->>L: 通知
    L->>Y: 固定プロンプト「バッチが届いた」
    Y->>S: 指摘を取りに行く
    S-->>Y: バッチを返す
    Y->>S: 結果を報告
```

3つに共通する設計判断が1つあります。通知には指摘の本文もアンカーも乗せず、「届いた」という合図だけを運ぶこと。中身はエージェントが毎回取りに行くので、通知経路が漏れても指摘は流出しません。同じように外部イベント待ちのCLIを作る場合は、実装をそのまま参考にできます。

## まとめ

16箇所の指摘をCodexとブラウザの往復だけでさばいた実例です。

https://artifactshare.com/a/y531hqc6jf

CLIとデータ契約は共通、エージェントを起こす合図だけが3つ別々。同じ体験のために、エージェントの数だけ起動経路を作る必要がありました。
