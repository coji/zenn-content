---
title: 'ブラウザだけで動くデータ分析AIエージェントを作ってみた'
emoji: '🦆'
type: 'tech'
topics: ['gemini', 'duckdb', 'aiagent', 'webassembly', 'chrome']
published: false
---

## これはなに？

CSVやParquetを読み込んで日本語で質問すると、SQLを書いて集計し、表と文章で答えるAIエージェントを作りました。推論も集計も実行の記録もブラウザのタブの中で動き、サーバーも外部のLLM APIも使っていません。途中でページを閉じても、次に開いたときに保存済みの手順から再開します。

@[youtube](VedgU3kh_GA)

動画では質問したあと要約の途中でページを再読み込みしています。再開後、SQLは実行し直さず、中断した要約だけをやり直しています。

デモはArtifact Shareに静的サイトとして置きました。総務省統計局の家計調査（2025年）を同梱しているので、開けばすぐに質問できます。自分のCSVやParquetも読み込めます。デスクトップ版のChromeが必要です。

https://530juz5j78.artifactshare.link/

https://github.com/artifactshare/examples/tree/main/examples/durable-analyst

たとえば「ぎょうざへの支出が多い市を5つ教えて」と聞くと、Gemini Nanoが次のSQLを書き、浜松市、宇都宮市、宮崎市、さいたま市、大津市の順に支出額を返しました。

```sql
SELECT area, yen FROM household_spending
WHERE item = 'ぎょうざ' AND area_type IN ('都道府県庁所在市', '政令指定都市')
ORDER BY yen DESC LIMIT 5
```

ここから先は仕組みの話です。

## 部品はブラウザに揃っていた

使ったのは次の3つです。保存先はどれもOPFS（ブラウザがサイトごとに持つ、ページからは見えないファイル置き場）です。

| 役割 | 使ったもの |
|---|---|
| SQLを書き、結果を文章にする | Chrome内蔵のGemini Nano（Prompt API） |
| 集計する | DuckDB-WASM（データをOPFSのファイルに保存） |
| 実行を記録する | pi-durable（会話と実行状態をSQLite-WASMでOPFSに保存） |

Gemini NanoはChromeに内蔵されたモデルで、Prompt API（`LanguageModel`）から呼べます。今回はChrome 155で、フラグを立てずに使えました。初回はモデルのダウンロードが必要で、`LanguageModel.availability()`が`downloadable`を返したら、ユーザーの操作をきっかけに`LanguageModel.create()`を呼ぶとダウンロードが始まります。Chromeの案内では、空きディスクが22GB以上あることなどが条件です。

pi-durableはコーディングエージェントの[Pi](https://github.com/earendil-works/pi)と同じリポジトリにある実行基盤です。会話、モデルの返答、ツールの呼び出しと結果を、1つずつストレージにコミットしながら進めます。実行を受け持つ本体はHarnessと呼ばれます。

質問してから答えが出るまでの間、ネットワークへの通信は起きません。外に出るのは、起動時にDuckDB-WASMのエンジンをCDNから、拡張をDuckDBの配布元から読み込む通信と、初回のモデルのダウンロードだけです。

## タブを閉じたら、どこから再開するか

ブラウザのタブはサーバーのプロセスよりずっと気軽に閉じられます。エージェントの実行中にページが閉じられる場面は、大きく3つあります。

| 閉じた場面 | 次に開いたときにしたいこと |
|---|---|
| 質問を送った直後 | 同じ質問を二重に送らない |
| SQLの実行中 | 終わったかどうか分からない手順を、もう一度実行してよいか決める |
| 要約の途中 | 済んだSQLはやり直さず、要約だけやり直す |

厄介なのは2つ目です。閉じた瞬間に動いていた手順は、終わったのかどうかを記録から判断できません。

pi-durableはツールを動かす前に「どのツールをどの引数で呼ぶか」をコミットし、終わったら結果をコミットします。呼ぶ記録だけが残って結果のないツールは、ツール定義の`replay`で扱いを決めます。

```ts
const executeSql = defineTool({
  name: 'execute_sql',
  parameters: Type.Object({ sql: Type.String() }),
  replay: 'safe', // 読み取り専用なので、中断したらもう一度実行してよい
  execute: async (args, api, context) => {
    // ...
  },
})
```

`replay: 'safe'`のツールは再開後にもう一度実行されます。既定の`'unsafe'`のツールは再実行されず、中断したというエラー結果がモデルに返ります。ツールは、スキーマを返す`inspect_schema`、SQLを実行する`execute_sql`、結果を文章にする`summarize_result`の3つで、どれも読み取り専用なので`safe`にしました。書き込みをするツールがないので、`unsafe`のツールが中断されたときの振る舞いは試していません。

3つ目の場面は2つ目の延長です。`summarize_result`は`execute_sql`の中から入れ子のツール呼び出しとして呼んでいるので、要約の途中で閉じると`execute_sql`も途中のまま残り、再開すると`execute_sql`がもう一度実行されます。そこで、SQLの結果を呼び出しのIDをキーにして会話ごとの文書に保存し、保存済みならDuckDBを呼ばずにそれを返すようにしました。入れ子の要約も、pi-durableが同じキーの呼び出しとして付け直します。durable executionでよく使う、冪等キーで結果を覚えておく形です。

1つ目の二重送信は質問に`requestId`を付けて先に記録してから、`conversation.submit({ type: 'input', content, requestId })`で送ることで防ぎました。記録と送信の間で閉じても、次に開いたとき同じ`requestId`で送り直せば、pi-durableは新しい質問を作らず、前の送信を返します。

## pi-durableをブラウザで動かす

pi-durableのストレージは、Node.js用とCloudflare Durable Objects用が用意されています。ただ、SQLiteの中核部分はNode.jsに依存しておらず、非同期の`exec`や`transaction`を持つオブジェクトを渡せば動きます。

Web Worker上のsqlite-wasmへSQLを転送するアダプタを約100行で書きました。

sqlite-wasmのファイルの置き方（VFS）には`opfs-sahpool`を選びました。もう1つの`opfs`はページにCOOP/COEPというヘッダーを付けて分離する必要があり、静的ホスティングでは設定できないことが多いからです。

pi-durableにはストレージの適合テストが付属しているので、同じアダプタをNode.jsのsqlite-wasmにつないで24件すべてを通しました。

再開のテストはHarnessを閉じずに放置し、同じSQLiteファイルを別のHarnessで開き直す形で書いています。SQLの実行中に止めると再開後にSQLが1回だけ実行され、要約中に止めると実行済みのSQLは再実行されません。実際のページの再読み込みでも、要約中にリロードするPlaywrightのテストで、済んだ手順が繰り返されないことを確かめました。ブラウザごと強制終了したときまでは試していません。

## 画面の状態は記録から計算する

画面に出す「計画中」「要約中」「完了」といった状態は、別に保存していません。pi-durableがコミットした質問の処理状態、会話、実行中のツールの一覧から毎回計算しています。状態を別に保存すると、その状態とコミットの内容が食い違ったまま閉じられたとき、再開後の表示が実際と合わなくなるためです。

## つまずいたところ

OPFSのファイルは同時に1つのタブからしか開けません。2つ目のタブではDuckDBもSQLiteもファイルを開けずに失敗するので、Web Locksでタブを1つに限り、2つ目のタブにはその旨を表示しました。

開発中は、ReactのStrictModeで起動処理が2回実行され、1回目が取ったタブのロックのせいで2回目が失敗しました。ViteのHMRでも、DuckDBが同じファイルを二重に開いて書き込めなくなったことがあります（原因は推定です）。起動処理を1回に限り、HMRではページ全体を読み込み直すようにしました。

再開を始めるタイミングにも注意が要りました。Gemini Nanoのダウンロード前に再開すると、モデルの呼び出しが失敗して実行が終わってしまうので、モデルが使えると分かってから再開しています。

## Gemini Nanoに任せる範囲

ブラウザだけで動かす以上、モデルは小さくなります。Prompt APIにはツール呼び出し専用の仕組みがないので、返答をJSON Schemaで「どれかのツールの呼び出し」か「回答」に縛り、返ってきたJSONをpi-ai（pi-durableが使うモデル呼び出しの層）のツール呼び出しに変換しました。

```ts
const session = await LanguageModel.create({ initialPrompts: [{ role: 'system', content: system }, ...history] })
const reply = await session.prompt(prompt, {
  responseConstraint: {
    anyOf: [
      { type: 'object', properties: { tool: { enum: ['execute_sql'] }, arguments: { type: 'object', properties: { sql: { type: 'string' } } } }, required: ['tool', 'arguments'] },
      { type: 'object', properties: { answer: { type: 'string' } }, required: ['answer'] },
    ],
  },
})
// {"tool": "execute_sql", "arguments": {...}} ならツール呼び出し、{"answer": "..."} なら回答として返す
```

実データで試すと、Gemini Nanoはいろいろな形で間違えました。`SELECT利用者 FROM …`のように空白を落とす、SQLを書かずにスキーマだけ見て答える、要約で`tanaka`を「田中さん」に書き換える、といった具合です。

そこで、モデルに任せるのは「SQLを1本書く」ことと「結果を文章にする」ことだけにしました。SQLの空白と引用符はコードで補い、`execute_sql`を実行するまでは上のスキーマから回答を外し、要約の数字と名前が結果にあるかを確かめています。

間違いと対策の一覧はデモの解説ページにまとめました。

https://530juz5j78.artifactshare.link/#how

## まだできないこと

モデルに渡す会話は最新の質問1つだけです。前の答えも渡すと、その値を次のSQLに持ち込むことがあったからで、そのかわり前の答えを受けた質問は苦手です。

1つの質問に答えるまでは15〜20秒かかります。ほとんどがGemini Nanoの推論で、DuckDBの集計は数十ミリ秒です。

pi-durableはパッケージに「Experimental」と書かれていて、APIは予告なく変わります。今回の組み合わせはpi-durable 1.3.0とChrome 155で確かめました。

ブラウザのタブの中だけで、閉じても次に開いたときに続きから動くデータ分析エージェントを作ってみた、という話でした。
