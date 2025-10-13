---
title: "Remix 3 発表まとめ - React を捨て、Web標準で新しい世界へ"
emoji: "💿️"
type: "tech"
topics: ["remix", "react", "typescript", "web", "javascript"]
published: true
---

## はじめに

2025年10月10日、カナダのトロントで開催されたイベント "Remix Jam 2025" で Ryan Florence と Michael Jackson が **Remix 3** を発表しました。このセッションは、React Router の生みの親たちが、なぜ React から離れ、独自のフレームワークを作ることにしたのか、その理由と新しいビジョンを語った歴史的な発表です。

https://www.youtube.com/live/xt_iEOn2a6Y?t=11764s

本記事では、1時間47分に及ぶセッションの内容を詳しく解説します。

:::message alert
**注意事項**

- この記事は、セッション動画の音声を AI で文字起こしし、その内容をもとに AI を活用して執筆しています
- 記事内のコード例は、セッションでの説明をもとに再構成したものですが、実際の動作確認はまだ行えていません
- Remix 3 は現在プロトタイプ段階のため、実際の API や仕様は変更される可能性があります
- コードの正確性については、追って確認・修正する予定です
- より正確な情報については、上記の動画を直接ご確認ください
:::

:::message
この記事は、セッションの前半部分（Part 1）のみを扱います。バックエンド関連の内容は Part 2 で語られました。
:::

## なぜ Remix 3 を作るのか

> 💡 [動画で確認する (3:17:30~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=11850s)

### React への感謝と決別

Michael Jackson と Ryan Florence は、React に対して深い敬意を持っています。React は彼らのキャリアを変え、Web 開発の考え方を一変させました。React Router を10年以上メンテナンスし、Shopify のような大企業がそれに依存しています。

しかし、ここ1〜2年、彼らは React の方向性に違和感を感じるようになりました。

> 「僕らはもう、React がどこに向かっているのか分からなくなってきた」- Michael Jackson

### 現代のフロントエンド開発の複雑さ

Ryan は、フロントエンドエコシステムの複雑さについて率直に語ります：

> 「正直言って、全体像を把握できなくなってきた。フロントエンド開発者として、自分でも何が起きているのか分からない時がある」- Ryan Florence

彼らは、この状況を「山を登る」比喩で表現しています：

> 「僕らはこの山を登ってきて、頂上でキャンプしようとしている。でも、登ってきたおかげで視野が広がり、別の山が見えてきた。もっとシンプルな山が。だから、この山を下りて、あっちの山に登り直すことにした」- Ryan Florence

### Web プラットフォームの進化

Node.js は16歳、React は12〜13歳です。その間に Web プラットフォームは大きく進化しました：

- **ES Modules**: ブラウザでモジュールをロードできる
- **TypeScript**: 型による開発体験の向上
- **Service Workers**: バックエンド機能をブラウザで
- **Web Streams**: Node.js にも標準ストリームが
- **Fetch API**: Node.js でも使える
- **Web Crypto**: 暗号化機能が標準に

> 💡 [動画で確認する (3:22:45~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=12165s)

### AI 時代のフレームワーク

Ryan は、AI 時代のフレームワークに必要な要素について語ります：

- **安定した URL**: LLM がアクションを実行するため、URL は常に同じである必要がある
- **シンプルなコード**: AI が生成・理解しやすいコード
- **バンドラーへの依存を減らす**: ランタイムセマンティクスがバンドラーに依存しない

React の `use server` では、RPC 関数の URL がビルドごとに変わってしまうため、AI がそれを利用することが困難です。

> 💡 [動画で確認する (3:19:34~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=11974s)

## Remix 3 の核心アイデア

### セットアップスコープ (Setup Scope)

> 💡 [動画で確認する (3:50:03~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=13803s)

Remix 3 の最も革新的な概念が **Setup Scope（セットアップスコープ）** です。

```javascript
import { events } from "@remix-run/events"
import { tempo } from "./01-intro/tempo"
import { createRoot, type Remix } from "@remix-run/dom"

function App(this: Remix.Handle) {
  // このスコープは1回だけ実行される（セットアップスコープ）
  let bpm = 60

  // レンダー関数を返す
  return () => (
    <button
      on={tempo((event) => {
        bpm = event.detail
        this.update()
      })}
    >
      BPM: {bpm}
    </button>
  )
}

createRoot(document.body).render(<App />)
```

重要なポイント：

1. **セットアップコードは1回だけ実行される**
2. **状態は JavaScript のクロージャに保存される**（Remix の特別な機能ではない）
3. **再レンダリングは `this.update()` を明示的に呼ぶ**

> 「ボタンはどうやって BPM が変わったことを知るの？ **知らない**。それが Remix 3 の素晴らしいところ。これはただの JavaScript スコープ。君が `update()` を呼んだ時だけ、レンダー関数を再実行する」- Ryan Florence

### Remix Events: イベントを第一級市民に

> 💡 [動画で確認する (3:34:50~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=12890s)

Remix 3 では、**イベントをコンポーネントと同じレベルの抽象化**として扱います。

#### `click` イベントの複雑さ

Ryan は、`click` イベントの複雑さを説明します：

- マウスダウン + マウスアップ（同じ要素上）
- キーボードの Space ダウン + Space アップ（Escape なし）
- キーボードの Enter ダウン（即座にクリック + リピート）
- タッチスタート + タッチアップ（スワイプなし）

これらすべてが `click` イベントです。

#### カスタムインタラクションの作成

Remix Events を使うと、独自のインタラクションを作成できます：

```javascript
import { createInteraction, events } from "@remix-run/events"
import { pressDown } from "@remix-run/events/press"

export const tempo = createInteraction<HTMLElement, number>(
  "rmx:tempo",
  ({ target, dispatch }) => {
    let taps: number[] = []
    let resetTimer: number = 0

    function handleTap() {
      clearTimeout(resetTimer)
      taps.push(Date.now())
      taps = taps.filter((tap) => Date.now() - tap < 4000)
      if (taps.length >= 4) {
        let intervals = [];
        for (let i = 1; i < taps.length; i++) {
          intervals.push(taps[i] - taps[i - 1])
        }
        let bpm = intervals.map(
          (interval) => 60000 / interval
        )
        let avgTempo = Math.round(
          bpm.reduce((sum, value) => sum + value, 0) /
            bpm.length
        )
        dispatch({ detail: avgTempo })
      }
      resetTimer = window.setTimeout(() => {
        taps = []
      }, 4000)
    }

    return events(target, [pressDown(handleTap)])
  }
)
```

使い方：

```javascript
<button on={tempo((event) => {
  bpm = event.detail
  this.update()
})}>
  BPM: {bpm}
</button>
```

> 「コンポーネントが要素に対する抽象化であるように、カスタムインタラクションはイベントに対する抽象化だ」- Ryan Florence

![Remix Events の概念図](/images/remix3-introduction/demo1-step2-events-concept.png)
*図: Components are to elements as custom interactions are to events*

### Context API: 再レンダリングを引き起こさない

> 💡 [動画で確認する (4:07:36~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=14856s)

Remix 3 の Context API は、React とは根本的に異なります。

```typescript
function App(this: Remix.Handle<Drummer>) {
  const drummer = new Drummer(120)
  // コンテキストをセット（再レンダリングは発生しない）
  this.context.set(drummer)

  return () => (
    <Layout>
      <DrumControls />
    </Layout>
  )
}

function DrumControls(this: Remix.Handle) {
  // コンテキストを型安全に取得
  let drummer = this.context.get(App)
  // drummer の変更を購読
  events(drummer, [Drummer.change(() => this.update())])

  return () => (
    <ControlGroup>
      <Button on={dom.pointerdown(() => drummer.play())}>
        PLAY
      </Button>
      <Button on={dom.pointerdown(() => drummer.stop())}>
        STOP
      </Button>
    </ControlGroup>
  )
}
```

重要なポイント：

1. **`context.set()` は再レンダリングを引き起こさない**
2. **`context.get(Component)` でプロバイダーを直接参照**（"Go to Definition" が効く！）
3. **型安全**: プロバイダーコンポーネントの型から自動推論

![Context API の実装](/images/remix3-introduction/demo2-context-api.png)
*図: Go to Definition でプロバイダーに直接ジャンプできる*

### Signal: 非同期処理の管理

> 💡 [動画で確認する (4:42:39~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=16959s)

Remix 3 には重要な原則があります：

> 「関数を渡したら、signal を返す」

イベントハンドラーには自動的に `signal` が渡されます（`AbortController` の signal）：

```javascript
<select
  id="state"
  on={dom.change(async (event, signal) => {
    fetchState = "loading"
    this.update()

    const response = await fetch(
      `/api/cities?state=${event.target.value}`, 
      { signal } // signal を fetch に渡す
    )
    cities = await response.json()
    if (signal.aborted) return // 古いリクエストは自動的に中断される

    fetchState = "loaded"
    this.update()
  })}
>
```

ユーザーが連続してセレクトボックスを変更すると：

1. 古いハンドラーの signal が abort される
2. `fetch()` が自動的にキャンセルされる
3. `signal.aborted` チェックで古い処理をスキップ

![Signal でレースコンディションを解決](/images/remix3-introduction/demo3-race-condition-solved.png)
*図: ネットワークタブで古いリクエストがキャンセルされている様子*

これにより、**レースコンディションを手動で、しかしシンプルに解決**できます。

## 実際のデモから学ぶ

### デモ1: カウンターからテンポタッパーへ

> 💡 [動画で確認する (3:29:03~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=12543s)

Ryan は最もシンプルな例から始めます。

#### ステップ1: プレーンJSでカウンター → テンポタッパー

まずは、プレーンな JavaScript でシンプルなカウンターを作ります：

```javascript
// プレーンな DOM API から始める
let button = document.createElement("button")
let count = 0

button.addEventListener("click", () => {
  count++
  update()
})

function update() {
  button.textContent = `Count: ${count}`
}

update()
document.body.appendChild(button)
```

> 「山を下りているんだ。プラットフォームには何がある？」- Ryan Florence

![シンプルなカウンター](/images/remix3-introduction/demo1-step1-counter.png)
*図: プレーンJavaScriptで実装したカウンター*

**退屈だから、もっと面白いものへ**

> 💡 [動画で確認する (3:32:13~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=12733s)

> 「退屈だな。Remix Jam なのに、何でくだらないカウンターの話をしてるんだ？もっとエキサイティングなものを作ろう」- Ryan Florence

ここで Ryan は、クリックの**速さ（BPM）**を測定するテンポタッパーに変更します：

```javascript
let button = document.createElement("button")
let tempo = 60
let taps = []
let resetTimer = 0

function handleTap() {
  clearTimeout(resetTimer)
  taps.push(Date.now())
  taps = taps.filter((tap) => Date.now() - tap < 4000)

  if (taps.length >= 4) {
    let intervals = []
    for (let i = 1; i < taps.length; i++) {
      intervals.push(taps[i] - taps[i - 1])
    }
    let bpm = intervals.map((interval) => 60000 / interval)
    tempo = Math.round(
      bpm.reduce((sum, value) => sum + value, 0) / bpm.length
    )
    update()
  }

  resetTimer = window.setTimeout(() => {
    taps = []
  }, 4000)
}

button.addEventListener("pointerdown", handleTap)
button.addEventListener("keydown", (event) => {
  if (event.repeat) return
  if (event.key === "Enter" || event.key === " ") {
    handleTap()
  }
})

function update() {
  button.textContent = `${tempo} BPM`
}

update()
document.body.appendChild(button)
```

このコードは、タップの間隔を計算して平均 BPM を算出しています：

1. 直近4秒間のタップを配列に保存
2. タップ間の間隔（ミリ秒）を計算
3. 各間隔から BPM を計算（60000 / interval）
4. すべての BPM を平均して表示

![BPM計算ロジック](/images/remix3-introduction/demo1-step1-bpm-logic.png)
*図: タップ間隔を計算して平均BPMを算出*

#### ステップ2: Remix Events でイベントを抽象化

> 💡 [動画で確認する (3:34:50~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=12890s)

Ryan は `click` イベントの複雑さを説明します：

> 「みんな、`click` イベントって本当に知ってる？`click` は実は複雑なんだ」- Ryan Florence

**`click` イベントの内部動作:**
- マウスダウン + マウスアップ（同じ要素上）
- キーボードの Space ダウン + Space アップ（Escape なし）
- キーボードの Enter ダウン（即座にクリック + リピート）
- タッチスタート + タッチアップ（スワイブなし）

これらすべてが `click` として発火します。

そこで、Remix Events を使って**カスタムインタラクション**を作成します：

```javascript
import { createInteraction, events } from "@remix-run/events"
import { pressDown } from "@remix-run/events/press"

export const tempo = createInteraction<HTMLElement, number>(
  "rmx:tempo",
  ({ target, dispatch }) => {
    let taps = []
    let resetTimer = 0

    function handleTap() {
      clearTimeout(resetTimer)
      taps.push(Date.now())
      taps = taps.filter((tap) => Date.now() - tap < 4000)

      if (taps.length >= 4) {
        let intervals = []
        for (let i = 1; i < taps.length; i++) {
          intervals.push(taps[i] - taps[i - 1])
        }
        let bpm = intervals.map((interval) => 60000 / interval)
        let avgTempo = Math.round(
          bpm.reduce((sum, value) => sum + value, 0) / bpm.length
        )
        dispatch({ detail: avgTempo })
      }

      resetTimer = window.setTimeout(() => {
        taps = []
      }, 4000)
    }

    return events(target, [pressDown(handleTap)])
  }
)
```

> 「コンポーネントが要素に対する抽象化であるように、カスタムインタラクションはイベントに対する抽象化だ」- Ryan Florence

![Remix Events の概念図](/images/remix3-introduction/demo1-step2-events-concept.png)
*図: Components are to elements as custom interactions are to events*

**重要なポイント:**

1. **状態とイベントをカプセル化**: `taps` 配列や `resetTimer` は `tempo` インタラクション内部に隠蔽
2. **型安全**: `createInteraction<HTMLElement, number>` で型を定義
3. **再利用可能**: どこでも `tempo` インタラクションを使える
4. **合成可能**: `pressDown` は内部で `pointerdown` と `keydown` を統合

#### ステップ3: Remix 3 のコンポーネント化

> 💡 [動画で確認する (3:50:03~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=13803s)

> 「みんな、コンポーネントを見せろって言ってる。よし、コンポーネントにしよう」- Ryan Florence

ここで、プレーンな JavaScript から Remix 3 のコンポーネントに変換します：

```javascript
import { events } from "@remix-run/events"
import { tempo } from "./01-intro/tempo"
import { createRoot, type Remix } from "@remix-run/dom"

function App(this: Remix.Handle) {
  // セットアップスコープ（1回だけ実行される）
  let bpm = 60

  // レンダー関数を返す
  return () => (
    <button
      on={tempo((event) => {
        bpm = event.detail
        this.update()
      })}
    >
      BPM: {bpm}
    </button>
  )
}

createRoot(document.body).render(<App />)
```

ここで Ryan が強調する重要なポイント：

> 「ボタンはどうやって BPM が変わったことを知るの？ **知らない**。それが Remix 3 の素晴らしいところ。これはただの JavaScript スコープ。君が `update()` を呼んだ時だけ、レンダー関数を再実行する」- Ryan Florence

**セットアップスコープの特徴：**

1. **1回だけ実行される**: コンポーネントの初期化時のみ
2. **状態は JavaScript のクロージャに保存**: 特別な機能ではなく、普通の JavaScript
3. **`this.update()` で明示的に再レンダリング**: 自動的な依存性追跡はなし

`tempo` カスタムインタラクションが、先ほどの複雑なタップ計算ロジックをすべてカプセル化しています。コンポーネントは結果を受け取って表示するだけです。

![コンポーネント化されたテンポタッパー](/images/remix3-introduction/demo1-step3-component.png)
*図: Remix 3 コンポーネントとして実装されたテンポタッパー*

### デモ2: ドラムマシン

:::message alert
- ここ以降に出てくるソースコードは、文字起こしで話されている内容から、AI がコードを推測したものなので、正しくない可能性が高いです。
- 今後、動画を見直してソースコードを確認して修正してきます。
:::

> 💡 [動画で確認する (3:56:02~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=14162s)

完全なドラムマシンアプリを構築します。

主な機能：

- Play/Stop ボタン
- テンポ調整（BPM）
- ビジュアライザー（音量表示）
- キーボードショートカット（Space: 再生/停止、Arrow Up/Down: テンポ変更）

#### Drummer クラス（AI が生成）

```javascript
class Drummer extends EventTarget {
  #bpm = 90

  play(bpm) {
    this.#bpm = bpm
    // ドラムサウンドを再生...
    this.dispatchEvent(new CustomEvent("change"))
  }

  stop() {
    // 再生を停止...
    this.dispatchEvent(new CustomEvent("change"))
  }

  set bpm(value) {
    this.#bpm = value
    this.dispatchEvent(new CustomEvent("change"))
  }

  get bpm() {
    return this.#bpm
  }
}
```

> 「Cursor に『キック、スネア、ハイハットを持ったドラマーを作って』って頼んだら、こいつが吐き出してくれた。最高だろ？」- Ryan Florence

#### キーボードイベントの統合

```javascript
import { space, arrowUp, arrowDown } from "@remix/events"

function App(this: RemixHandle) {
  const drummer = new Drummer()

  return function render() {
    return (
      <div
        on:window={[
          [space, () => drummer.toggle()],
          [arrowUp, () => drummer.bpm += 5],
          [arrowDown, () => drummer.bpm -= 5],
        ]}
      >
        <DrumMachine />
      </div>
    )
  }
}
```

`window` にイベントを追加しているのに、コンポーネント内のコードと変わりません。

![キーボードショートカット](/images/remix3-introduction/demo2-keyboard-shortcuts.png)
*図: Space、Arrow Up/Down でドラムマシンを操作*

### デモ3: フォームと非同期処理

> 💡 [動画で確認する (4:37:24~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=16644s)

州を選択すると、その州の都市リストを fetch する典型的な UI です。

```javascript
function CitySelector(this: RemixHandle) {
  let state = "idle"
  let cities = []

  return function render() {
    return (
      <form>
        <select
          on:change={async (event, signal) => {
            state = "loading"
            this.update()

            const response = await fetch(
              `/api/cities?state=${event.target.value}`,
              { signal }
            )

            if (signal.aborted) return

            cities = await response.json()
            state = "loaded"
            this.update()
          }}
        >
          <option>Alabama</option>
          <option>Alaska</option>
          {/* ... */}
        </select>

        <select disabled={state === "loading"}>
          {cities.map(city => (
            <option>{city}</option>
          ))}
        </select>
      </form>
    )
  }
}
```

> 「イベントから考え始める。それが僕のやり方。ユーザーが最初のセレクトボックスを変更した → ローディング状態にする → データを取得 → ロード完了。これが一番自然な考え方だと思わない？」- Ryan Florence

![レースコンディション](/images/remix3-introduction/demo3-race-condition-problem.png)
*図: 連続して選択を変更した場合の問題*

## Remix 3 の設計思想

### 抽象化は最小限に

> 「抽象化は、本当に必要だと感じるまで導入しない。イベントには型安全性と合成のために必要だった。でも、他の部分は？」- Ryan Florence

Remix 3 のコンポーネントは、特別な状態管理ライブラリを使いません：

```javascript
let bpm = 60 // ただの変数
```

更新も明示的：

```javascript
this.update() // これだけ
```

### Web 標準を最大限活用

- `EventTarget` と `CustomEvent`
- `AbortController` と `signal`
- `PointerEvent` でマウス・タッチ・ペンを統一
- DOM API をそのまま利用

### TypeScript ファーストの開発体験

> 「Remix 1 と 2 では TypeScript はサイドクエストみたいなものだった。でも今は、TypeScript が開発体験の中心だ」- Michael Jackson

すべての API が型安全に設計されています：

- イベントの detail 型
- Context の型推論
- コンポーネントの props 型

### LLM で生成しやすいコード

Ryan は、AI が Drummer クラスを生成したことを何度も強調します。Remix 3 のコードは：

- シンプルで予測可能
- 特殊な規則が少ない
- Web 標準に基づいている

そのため、LLM が理解・生成しやすいのです。

## React Router は継続される

> 💡 [動画で確認する (3:19:04~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=11944s)

重要なポイント：

- **React Router は継続されます**
- Shopify など多くの企業が React Router に依存
- Remix チームが React Router V7 を開発中
- Remix 3 は別の選択肢として提供

> 「React Router はどこにも行かない。それだけは明確にしておきたい」- Ryan Florence

## 現在のステータス

> 💡 [動画で確認する (3:40:32~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=13232s)

- **プロトタイプ段階**
- ブログ投稿の3ヶ月後に開発開始
- 個別パッケージとして公開中（`@remix/events`、`@remix/ui` など）
- 最終的には統合されたフレームワークとして提供予定
- **コンポーネントライブラリも開発中**（ドロップダウンメニュー、テーマシステムなど）

![ドラムマシンアプリ](/images/remix3-introduction/demo2-drum-machine.png)
*図: Remix 3 で構築した完全なドラムマシンアプリ*

## まとめ

Remix 3 は、フロントエンド開発の複雑さに対するアンチテーゼです。

**主要な特徴：**

1. **Setup Scope**: JavaScript のクロージャを活用した状態管理
2. **Remix Events**: イベントを第一級市民として扱う
3. **明示的な再レンダリング**: `this.update()` で制御
4. **型安全な Context API**: 再レンダリングを引き起こさない
5. **Signal による非同期管理**: レースコンディションをシンプルに解決
6. **Web 標準ベース**: バンドラーへの依存を最小化
7. **TypeScript ファースト**: すべての API が型安全
8. **AI フレンドリー**: LLM が理解・生成しやすいコード

**Ryan と Michael のメッセージ：**

> 「3ヶ月間、日の光を見ていない。でも、これはワクワクする。僕らは正しい山を見つけたと思う」- Ryan Florence

Remix 3 は、Web 開発の未来を再定義しようとしています。シンプルさ、Web 標準、型安全性、そして AI との親和性。これらすべてを兼ね備えた新しいフレームワークの登場を、期待して待ちましょう。

---

## 参考リンク

- [Remix 公式サイト](https://remix.run/)
- [React Router](https://reactrouter.com/)
- [Remix Jam 2025 セッション動画](https://www.youtube.com/watch?v=xt_iEOn2a6Y)

---

この記事が役に立ったら、ぜひ実際のセッション動画もご覧ください。Ryan のライブコーディングと軽妙なトークは、文字では伝えきれない魅力があります！
