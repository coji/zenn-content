---
title: "Cloudflare Workers + better-auth で全リクエストが無応答になる - hanging promise の罠"
emoji: "🧊"
type: "tech"
topics: ["cloudflare", "cloudflareworkers", "betterauth", "d1"]
published: true
---

## これはなに？

Cloudflare Workers + D1 + better-auth で運用している Web サービスで、「特定のユーザーだけ、サイトへの全リクエストが応答待ちのまま固まり、ブラウザを再起動しないと直らない」という障害に繰り返し遭遇しました。丸一日かけて原因を特定して解決したので、同じ構成でハマった人が解決策にたどり着けるように、調査の過程と修正をまとめておきます。

原因は Cloudflare Workers の hanging promise でした。リクエスト処理中に生成された promise は、そのリクエストがクライアントに中断されると永遠に解決しなくなります。better-auth はモジュールやインスタンスに promise を遅延キャッシュする箇所があり、運悪く初期化を担いだリクエストが中断されると、その isolate の認証がすべて永久にハングします。同じクライアントの接続は同じ isolate に届き続けるため、そのユーザーには「サーバ全体が落ちた」ように見えるのがこの障害の厄介なところです。

修正は 3 つで、AsyncLocalStorage を静的 import で事前セットする、遅延初期化を `ctx.waitUntil` に係留する、`getSession` にハング検知と作り直しを入れる、です。upstream には [better-auth#10315](https://github.com/better-auth/better-auth/issues/10315) として報告済みです。対象バージョンは better-auth 1.6.20 です。

## 症状

利用者から「サービスが落ちています」と連絡が来ました。ところが手元では普通に動いています。本人にブラウザを再起動してもらうと直りました。数日後にまた同じ連絡が来て、今度は自分の環境でも起きました。ページを開いてコメントを 2 件投稿すると、ほぼ毎回、そのブラウザからの全リクエストが応答待ちのまま固まります。

DevTools の Network を見ると、リクエストは送信されたまま 15 秒 (アプリ側のタイムアウト) で `(canceled)` になり続けます。新しいタブで開き直すと動くことがあり、リロードでは直らず、ブラウザ再起動か `chrome://net-internals/#sockets` の Flush socket pools で一時的に回復します。この見え方から、最初は Chrome の接続プールやセキュリティソフト、ネットワーク経路を疑って、かなり遠回りしました。

## リクエストはサーバに届いていた

最初の突破口はサーバ側のログでした。Cloudflare の zone ログには、固まっている時間帯のリクエストが 499 (クライアント切断) として記録されていて、Workers の invocation も outcome `canceled` で残っていました。つまりリクエストは edge に届き、Worker は起動しているのに、15 秒たっても応答を作れていません。「クライアントが送れない」のではなく「サーバが返せない」障害でした。

次の決め手は `chrome://net-export` の NetLog です。固まっている最中のキャプチャを解析すると、HTTP/2 接続そのものは完全に健康でした。h2 の PING は数ミリ秒で ACK が返り続けていて、Worker を通らない edge 配信の静的アセットは同じ接続で正常に応答しています。応答が来ないのは Worker 行きのストリームだけです。接続、経路、ブラウザの仮説はここで全部消えました。

## ハング箇所の特定

再現手順が確立していたので、Worker の処理段階ごとに `console.info` のマークを入れた診断ビルドを本番にデプロイしました。Workers では、クライアントに中断された invocation のログも中断時にまとめて flush されるので、「最後に通過したマーク」がそのままハング箇所を教えてくれます。

結果はこうでした。better-auth のインスタンス初期化 (`auth.$context` の await) は通過するのに、`auth.api.getSession({ headers })` が最初の D1 クエリに到達しないまま止まっています。同じ isolate で認証を通さずに実行した `env.DB.prepare('SELECT 1')` は成功するので、D1 も isolate のイベントループも生きています。ハングは better-auth のディスパッチ層、DB アクセスより手前にありました。

## 原因は hanging promise だった

Cloudflare Workers には、あまり知られていない性質があります。リクエスト処理中に生成された promise はそのリクエストの IoContext に属していて、クライアントが先に切断すると、workerd はその promise を永遠に解決しません。reject もされず、ただ pending のまま残ります[^hanging]。

[^hanging]: `ctx.waitUntil` に渡した処理が切断後も走り続けるのは、waitUntil が IoContext の寿命を延ばしているからです。逆に言うと、waitUntil に載っていない promise は切断とともに宙吊りになります。

better-auth 1.6.20 は、この性質と相性の悪い遅延初期化を持っています。ひとつは `@better-auth/core` の AsyncLocalStorage 解決で、モジュールスコープに `import("node:async_hooks")` の promise をキャッシュし、全 API 呼び出しが通る `ensureAsyncStorage` がそれを await します。もうひとつは `betterAuth()` が返すインスタンスの `$context` で、初期化 promise がインスタンスに保持され、全 API 呼び出しが await します。

どちらも「最初に触ったリクエスト」が初期化を担う設計です。Workers ではインスタンスを isolate 単位でキャッシュする書き方が一般的なので、新しい isolate に最初に届いた認証リクエストが初期化を駆動します。そのリクエストが初期化の完了前にクライアントに中断されると、キャッシュされた promise は永久に pending になり、以後その isolate に届いた認証 API はすべてそこで止まります。

「特定のユーザーだけ全滅する」ように見えた理由もここにあります。同じクライアントの接続は同じ isolate にルーティングされ続けるので、汚染された isolate を掴んだユーザーだけがサイト全体の無応答を体験し、他のユーザーは別の isolate で平常運転です。ブラウザ再起動で直るのは、新しい接続が別の isolate に届くからで、接続が直ったわけではありませんでした。

引き金になるクライアント中断は、珍しいものではありません。タブを閉じる、ページを離れる、`AbortController` で古いリクエストを打ち切る、のどれでも起きます。私の場合は、直前のリリースで「新しい取得を始めるとき進行中の取得を abort する」最適化を入れたことで中断の頻度が跳ね上がり、「コメント 2 件でほぼ毎回」という再現性になっていました。

## 修正

3 層で防ぎました。1 つでも効きますが、better-auth の内部に遅延キャッシュが残っている可能性を考えて重ねています。

1 層目は、AsyncLocalStorage の事前セットです。グローバルスコープでの静的 import は特定のリクエストに属さないので、中断で汚染されようがありません。better-auth が遅延解決するストレージを先回りして埋めておくと、危険な動的 import の経路自体を通らなくなります。

```typescript
import { AsyncLocalStorage } from 'node:async_hooks'

const betterAuthGlobalSymbol = Symbol.for('better-auth:global')

function seedBetterAuthAsyncStorage(): void {
  const holder = globalThis as unknown as Record<
    symbol,
    | { version: string; epoch: number; context: Record<string, unknown> }
    | undefined
  >
  const shared = (holder[betterAuthGlobalSymbol] ??= {
    version: 'seed',
    epoch: 0,
    context: {},
  })
  shared.context.requestStateAsyncStorage ??= new AsyncLocalStorage()
  shared.context.endpointContextAsyncStorage ??= new AsyncLocalStorage()
  shared.context.adapterAsyncStorage ??= new AsyncLocalStorage()
}
seedBetterAuthAsyncStorage()
```

キー名は better-auth の内部実装なので、バージョンを上げるときは確認が必要です。名前が変わっても `??=` なので壊れはせず、事前セットが効かなくなるだけです。

2 層目は、残る遅延初期化の `ctx.waitUntil` への係留です。waitUntil はクライアント切断後もリクエストの寿命を延ばすので、初期化を担いだリクエストが中断されても初期化自体は完了します。isolate ごとに 1 回だけ実行すれば十分です。係留する await 自体にも上限を置いておかないと、すでに汚染済みのインスタンスを掴んだとき係留側が永久 pending になるので、そこも打ち切ります。

```typescript
let cachedAuth: ReturnType<typeof buildAuth> | undefined
let lazyInitAnchored = false

export function createAuth() {
  return (cachedAuth ??= buildAuth()) // buildAuth() が betterAuth({...}) を組む
}

export function anchorAuthInit(ctx: ExecutionContext): void {
  if (lazyInitAnchored) return
  lazyInitAnchored = true
  ctx.waitUntil(
    initAuthLazyState().catch(() => {
      lazyInitAnchored = false
    }),
  )
}

async function initAuthLazyState(): Promise<void> {
  const auth = createAuth()
  const ready = await raceHang(
    (auth as unknown as { $context: Promise<unknown> }).$context,
    10_000,
  )
  if (ready === HANG) {
    if (cachedAuth === auth) cachedAuth = undefined
    throw new Error('auth lazy init hang')
  }
}
```

これを Worker の fetch ハンドラの先頭で毎リクエスト呼びます (フラグで isolate ごと 1 回に間引かれます)。

3 層目は、`getSession` のハング検知と自己回復です。better-auth の API は abort シグナルを受け取れないので、キャンセルではなく `Promise.race` で「ハングの検知」だけを行い、検知したらインスタンスを捨てて作り直します。作り直しても駄目なら null (未認証扱い) を返して、ハングをリクエストの応答まで伝播させません。

```typescript
export const HANG = Symbol('promise hang')

export async function raceHang<T>(
  promise: Promise<T>,
  ms: number,
): Promise<T | typeof HANG> {
  let timer: ReturnType<typeof setTimeout> | undefined
  const timeout = new Promise<typeof HANG>((resolve) => {
    timer = setTimeout(() => resolve(HANG), ms)
  })
  try {
    return await Promise.race([promise, timeout])
  } finally {
    clearTimeout(timer)
    promise.catch(() => {})
  }
}

async function getSession(headers: Headers) {
  const auth = createAuth()
  const first = await raceHang(auth.api.getSession({ headers }), 3000)
  if (first !== HANG) return first

  if (cachedAuth === auth) {
    cachedAuth = undefined
    lazyInitAnchored = false
  }
  const fresh = createAuth()
  const second = await raceHang(fresh.api.getSession({ headers }), 5000)
  console.warn('auth_hang', { recovered: second !== HANG })
  if (second !== HANG) return second
  if (cachedAuth === fresh) {
    cachedAuth = undefined
    lazyInitAnchored = false
  }
  return null
}
```

細部にいくつか落とし穴がありました。インスタンスの破棄は「自分が使っていたインスタンスのままのとき」に限定しないと、同時にハングを検知した並行リクエストが互いの再構築を潰し合います。破棄したら係留のフラグも戻さないと、次のインスタンスが未係留になります。race に負けた promise には `catch` を付けておかないと、あとから reject したときに未処理として扱われる可能性があります。

デプロイ後、「コメント 2 件でほぼ毎回」だった再現手順で通信停止が起きなくなりました。まれに 3 秒ほど引っかかるのはハング検知と作り直しが動いた瞬間で、構造化ログに残るので発生頻度も追えます。

## まとめ

Cloudflare Workers で「特定のユーザーだけ全リクエストが無応答になり、ブラウザ再起動で直る」症状が出たら、接続やクライアント環境より先に、isolate 単位で汚染される遅延初期化を疑うのが近道です。判別のポイントは 2 つで、Workers の invocation が outcome `canceled` で残っているか (リクエストは届いている)、NetLog で h2 の PING が生きているか (接続は健康) です。両方当てはまるなら、サーバ側のどこかの await が返っていません。

より一般的な教訓としては、Workers ではリクエスト処理中に生成した promise をモジュールスコープや isolate 寿命のオブジェクトにキャッシュしてはいけない、ということになります。自分のコードで必要になった場合は、waitUntil への係留かハング検知を必ず併設してください。ライブラリが内部でやっている場合は、今回のように外側から事前セットと係留と検知で防げます。

better-auth には [#10315](https://github.com/better-auth/better-auth/issues/10315) として報告しました。同じ遅延初期化には並行初期化の競合 ([#10300](https://github.com/better-auth/better-auth/issues/10300)) も報告されていて、静的 import による eager 初期化ならどちらも解消するはずです。修正が入るまでは、この記事の 3 層の回避策が使えます。
