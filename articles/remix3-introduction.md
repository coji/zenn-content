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

完全なドラムマシンアプリを構築し、**Context API**、**EventTarget**、**qTask** など Remix 3 の核心機能を実演します。

**構築する機能：**

- Play/Stop ボタン
- テンポ調整（BPM）
- リアルタイムビジュアライザー（kick、snare、hi-hat の音量表示）
- キーボードショートカット（Space: 再生/停止、Arrow Up/Down: テンポ変更）

#### ステップ1: Drummer クラスを AI に生成させる

> 「Cursor に『キック、スネア、ハイハットを持ったドラマーを作って』って頼んだら、こいつが吐き出してくれた」- Ryan Florence [00:46:15]

```javascript
// AI が生成した Drummer クラス（EventTarget を継承）
class Drummer extends EventTarget {
  #bpm = 90
  #playing = false
  #kick = 0
  #snare = 0
  #hihat = 0

  play(bpm) {
    this.#bpm = bpm
    this.#playing = true
    // ドラムループを開始...
    this.dispatchEvent(new CustomEvent("change"))
  }

  stop() {
    this.#playing = false
    this.dispatchEvent(new CustomEvent("change"))
  }

  toggle() {
    if (this.#playing) {
      this.stop()
    } else {
      this.play(this.#bpm)
    }
  }

  set bpm(value) {
    this.#bpm = value
    this.dispatchEvent(new CustomEvent("change"))
  }

  get bpm() {
    return this.#bpm
  }

  get volumes() {
    return {
      kick: this.#kick,
      snare: this.#snare,
      hihat: this.#hihat
    }
  }
}
```

**重要なポイント:**

1. `EventTarget` を継承 → 標準的な DOM イベントモデルを利用
2. `CustomEvent` で変更を通知 → どんなコンポーネントでもリッスンできる
3. **特別な Remix 用の型は不要** → 普通の JavaScript クラス

> 「重要なのは、これが特別な型である必要がないってこと。Cursor に頼めば吐き出してくれる。動けば使う。動かなければもう一回試す」- Ryan Florence [00:50:04]

#### ステップ2: Context API でアプリ全体に Drummer を共有

> 💡 [動画で確認する (4:06:54~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=14814s)
>
> **前半で学んだ [Context API](#context-api-再レンダリングを引き起こさない) の実践例です！**

```javascript
function App(this: Remix.Handle<{ drummer: Drummer }>) {
  // セットアップスコープで Drummer を作成
  const drummer = new Drummer()

  // Context に設定（再レンダリングは不要）
  this.context.set(drummer)

  // レンダー関数を返す
  return () => (
    <div>
      <DrumControls />
      <Equalizer />
    </div>
  )
}
```

**Context API の利点:**

```javascript
function DrumControls(this: Remix.Handle) {
  // App コンポーネントから Context を取得
  const drummer = this.context.get(App)

  // drummer の変更を監視
  drummer.addEventListener("change", () => this.update())

  // DOM 更新後に Stop ボタンにフォーカスを移動（後述）
  let stopButton = null

  return () => (
    <div>
      <button
        on={pressDown(() => {
          drummer.play(drummer.bpm)
          // qTask: DOM 更新後の処理
          this.qTask(() => stopButton?.focus())
        })}
        disabled={drummer.playing}
      >
        Play
      </button>
      <button
        ref={(el) => stopButton = el}
        on={pressDown(() => drummer.stop())}
        disabled={!drummer.playing}
      >
        Stop
      </button>
    </div>
  )
}
```

> 「Context の取得で `this.context.get(App)` を使うと、どのコンポーネントがそれを提供しているか一目瞭然。Go to Definition で飛べる。型も完全に安全」- Ryan Florence [00:54:12]

**React の Context との違い:**

| React Context | Remix Context |
|---|---|
| Provider コンポーネントが必要 | `this.context.set()` だけ |
| Context 変更 = 再レンダリング | Context 変更しても再レンダリングなし |
| Provider を探すのが大変 | Go to Definition で即座に見つかる |

#### ステップ3: 型安全なイベントを作る（createEventType）

> 💡 [動画で確認する (4:04:31~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=14671s)

カスタムイベントを型安全にするため、`createEventType` を使います：

```javascript
import { createEventType } from "@remix/events"

// 型安全な "change" イベントを作成
const [change, createChange] = createEventType<void>("change")

class Drummer extends EventTarget {
  // ... 他のメソッド

  set bpm(value) {
    this.#bpm = value
    // 型安全な方法で dispatch
    this.dispatchEvent(createChange())
  }

  // 静的メソッドとして公開（推奨パターン）
  static change = change
}

// 使用例
function TempoDisplay(this: Remix.Handle) {
  const drummer = this.context.get(App)

  return () => (
    <div
      on={[
        // 型安全なイベントリスナー
        Drummer.change(() => this.update())
      ]}
    >
      BPM: {drummer.bpm}
    </div>
  )
}
```

> 「カスタムイベントを文字列で管理するのは型安全じゃない。`createEventType` を使えば、イベント名も detail の型も完全に安全になる」- Ryan Florence [00:48:55]

#### ステップ4: qTask で DOM 更新後の処理

> 💡 [動画で確認する (4:13:15~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=15195s)

Play ボタンを押すと、Stop ボタンに自動的にフォーカスを移動したい：

```javascript
function DrumControls(this: Remix.Handle) {
  const drummer = this.context.get(App)
  let stopButtonRef = null

  return () => (
    <>
      <button
        on={pressDown(() => {
          drummer.play(drummer.bpm)

          // ❌ ここで focus() してもまだ DOM が更新されていない
          // stopButtonRef.focus() // エラー: disabled 状態のボタンにフォーカスできない

          // ✅ qTask: DOM 更新が完了してから実行
          this.qTask(() => {
            stopButtonRef?.focus()
          })
        })}
        disabled={drummer.playing}
      >
        Play
      </button>
      <button
        ref={(el) => stopButtonRef = el}
        on={pressDown(() => drummer.stop())}
        disabled={!drummer.playing}
      >
        Stop
      </button>
    </>
  )
}
```

**qTask の仕組み:**

> 「Remix は microtask でレンダリングをバッチ処理する。`qTask` は DOM 更新が完了した後に実行されるキューだ。リスナーじゃない。次のレンダリングで一度だけ実行される」- Ryan Florence [00:57:35]

```text
1. drummer.play() → 状態変更
2. this.update() → レンダリングをキューに追加
3. [microtask] レンダリング実行 → DOM 更新
4. [qTask] stopButtonRef.focus() 実行 ← DOM が更新された後！
```

#### ステップ5: キーボードイベントの統合

> 💡 [動画で確認する (4:21:06~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=15666s)

**前半で学んだ [Remix Events](#remix-events-イベントを第一級市民に) の実践例です！**

```javascript
import { space, arrowUp, arrowDown, arrowLeft, arrowRight } from "@remix/events"
import { win } from "@remix/events"

function App(this: Remix.Handle<{ drummer: Drummer }>) {
  const drummer = new Drummer()
  this.context.set(drummer)

  return () => (
    <div
      on={win([
        // Space: 再生/停止
        [space, () => drummer.toggle()],
        // Arrow Up/Left: テンポアップ
        [arrowUp, () => { drummer.bpm += 5 }],
        [arrowLeft, () => { drummer.bpm += 5 }],
        // Arrow Down/Right: テンポダウン
        [arrowDown, () => { drummer.bpm -= 5 }],
        [arrowRight, () => { drummer.bpm -= 5 }],
      ])}
    >
      <DrumControls />
      <Equalizer />
    </div>
  )
}
```

> 「window にイベントを追加しているのに、コンポーネント内のコードと変わらない。`on` プロップは、カスタムインタラクションもDOM要素もwindowも、全部同じように扱える」- Ryan Florence [01:07:26]

**セマンティックなキーイベント:**

- `space` → スペースキー
- `arrowUp` / `arrowDown` → 上下矢印キー
- 内部的には `keydown` をラップしているだけだが、**意図が明確**

![キーボードショートカット](/images/remix3-introduction/demo2-keyboard-shortcuts.png)
*図: Space、Arrow キーでドラムマシンを操作*

#### ステップ6: Equalizer でビジュアライザー

> 💡 [動画で確認する (3:57:48~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=14268s)

リアルタイムで音量を視覚化するコンポーネント：

```javascript
function Equalizer(this: Remix.Handle) {
  const drummer = this.context.get(App)

  drummer.addEventListener("change", () => this.update())

  return () => {
    const { kick, snare, hihat } = drummer.volumes

    return (
      <div class="equalizer">
        {/* kick: 4本のバー */}
        <Bar height={kick} />
        <Bar height={kick} />
        <Bar height={kick} />
        <Bar height={kick} />

        {/* snare: 3本のバー */}
        <Bar height={snare} />
        <Bar height={snare} />
        <Bar height={snare} />

        {/* hihat: 2本のバー */}
        <Bar height={hihat} />
        <Bar height={hihat} />
      </div>
    )
  }
}

function Bar(props: { height: number }) {
  return (
    <div
      class="bar"
      style={{ height: `${props.height * 100}%` }}
    />
  )
}
```

> 「kick が4本のバー、snare が3本、hihat が2本持ってる。状態をどうレンダリングするかはもう決めた。あとはアニメーションするだけ」- Ryan Florence [00:42:09]

![ドラムマシンのコンテキストAPI](/images/remix3-introduction/demo2-context-api.png)
*図: Context API で Drummer を全コンポーネントに共有*

**このデモで学んだこと:**

1. ✅ **EventTarget の活用**: 標準的な DOM イベントモデルで状態を管理
2. ✅ **Context API**: 再レンダリングなしでアプリ全体に値を共有
3. ✅ **型安全なイベント**: `createEventType` でカスタムイベントを型安全に
4. ✅ **qTask**: DOM 更新後の処理を安全に実行
5. ✅ **セマンティックなキーイベント**: `space`、`arrowUp` などで意図を明確に
6. ✅ **AI フレンドリー**: Drummer クラスは AI が生成できる普通の JavaScript

### デモ3: フォームと非同期処理（Signal によるレースコンディション解決）

> 💡 [動画で確認する (4:37:24~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=16644s)
>
> **前半で学んだ [Signal: 非同期処理の管理](#signal-非同期処理の管理) の実践例です！**

州を選択すると、その州の都市リストを fetch する典型的な UI を構築します。これは、**非同期処理のレースコンディション**という古典的な問題を扱います。

#### 問題: レースコンディション

```javascript
function CitySelector(this: Remix.Handle) {
  let state = "idle" // "idle" | "loading" | "loaded"
  let cities = []

  return () => (
    <form>
      <select
        on={DOM.change(async (event) => {
          // ローディング開始
          state = "loading"
          this.update()

          // データ取得
          const response = await fetch(
            `/api/cities?state=${event.target.value}`
          )
          cities = await response.json()

          // ローディング完了
          state = "loaded"
          this.update()
        })}
      >
        <option value="AL">Alabama</option>
        <option value="AK">Alaska</option>
        <option value="AZ">Arizona</option>
        <option value="IL">Illinois</option>
        <option value="KY">Kentucky</option>
        <option value="KS">Kansas</option>
      </select>

      <select disabled={state === "loading"}>
        {cities.map(city => (
          <option key={city}>{city}</option>
        ))}
      </select>
    </form>
  )
}
```

> 「イベントから考え始める。それが僕のやり方だ。ユーザーが最初のセレクトボックスを変更した。それで機能が始まる。ローディング状態にする → データ取得 → ロード完了。これが一番自然じゃない？」- Ryan Florence [01:12:49]

**問題の再現:**

Ryan は、デモ用に各州の fetch に異なる遅延を設定しています：

- Alabama: 300ms
- Alaska: 500ms
- Kansas: 5000ms（意図的に遅い）

ユーザーが素早く選択を変更すると：

1. Kentucky を選択 → fetch 開始（500ms）
2. Illinois を選択 → fetch 開始（1000ms）
3. Arizona を選択 → fetch 開始（800ms）

**結果:** どの fetch が最後に完了するかによって、表示される都市リストが変わってしまう！

> 「Louisville（Kentucky）、Illinois、Phoenix（Arizona）って表示された。問題だよね？」- Ryan Florence [01:16:00]

![レースコンディション問題](/images/remix3-introduction/demo3-race-condition-problem.png)
*図: 連続して選択を変更すると、最後に完了した fetch の結果が表示される*

#### 解決策: Signal を使う

> 💡 [動画で確認する (4:42:50~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=16970s)

Remix 3 の原則：

> 「Remix 3 の原則として、あなたが関数を渡したら、僕らはあなたに signal を渡す。あなたは非同期関数の中で好きなことができるべきだから、レースコンディションから自分を守る方法を提供する必要がある」- Ryan Florence [01:16:50]

**Signal を使った修正版:**

```javascript
function CitySelector(this: Remix.Handle) {
  let state = "idle"
  let cities = []

  return () => (
    <form>
      <select
        on={DOM.change(async (event, signal) => {
          //                            ^^^^^^ Remix が渡す AbortSignal
          state = "loading"
          this.update()

          // ✅ fetch に signal を渡す
          const response = await fetch(
            `/api/cities?state=${event.target.value}`,
            { signal } // <- これが重要！
          )

          // ✅ JSON パース中に abort されるかもチェック
          if (signal.aborted) return

          cities = await response.json()
          state = "loaded"
          this.update()
        })}
      >
        <option value="AL">Alabama</option>
        <option value="AK">Alaska</option>
        <option value="AZ">Arizona</option>
        <option value="IL">Illinois</option>
        <option value="KY">Kentucky</option>
        <option value="KS">Kansas</option>
      </select>

      <select disabled={state === "loading"}>
        {cities.map(city => (
          <option key={city}>{city}</option>
        ))}
      </select>
    </form>
  )
}
```

**Signal の仕組み:**

```text
1. Kentucky 選択 → fetch 開始（関数A実行中）
2. Illinois 選択 → 関数Aの signal を abort
                  → fetch 開始（関数B実行中）
3. Arizona 選択 → 関数Bの signal を abort
                 → fetch 開始（関数C実行中）
```

> 「この関数は1つだけだが、ユーザーがセレクトボックスをクリックするたびに、複数の呼び出しが同時に進行してる。非同期だからね。1回選択したら関数を呼んで待ってる。もう一回クリックしたら、また関数を呼んで待ってる。関数が再度呼ばれた時、Remix は前の signal を abort する」- Ryan Florence [01:17:56]

![Signal でレースコンディション解決](/images/remix3-introduction/demo3-race-condition-solved.png)
*図: Signal を使うと、最新のリクエストだけが完了する*

#### Signal の2つの使い方

**1. fetch API に渡す（推奨）**

```javascript
const response = await fetch(url, { signal })
```

fetch API は、signal が abort されると自動的に `AbortError` を throw します。

**2. 手動でチェック**

```javascript
if (signal.aborted) return
```

JSON のパースなど、時間がかかる処理の後にチェックします。

> 「abort controller を fetch に渡すと、throw する。だから、それ以降のコードは実行されない」- Ryan Florence [01:19:35]

> 「2番目のチェックは実は不要だった。fetch が throw するから。でも、JSON のパースが巨大だったら、そこでもレースコンディションになりうる。だから、非同期処理の後は signal をチェックする癖をつけるといい」- Ryan Florence [01:19:35]

#### Remix 3 のシンプルな原則

> 「手動でやる必要がある。`this.update()` を呼ぶのと同じように、手動で `signal` を使う。でも、**いつも abort させたいわけじゃない**。投票システムみたいに、全部通したいこともある。重要な時だけ signal を使えばいい」- Ryan Florence [01:20:30]

**重要な設計思想:**

- **自動的な依存関係追跡はしない** → 明示的に `this.update()` を呼ぶ
- **自動的な abort もしない** → 明示的に `signal` を使う
- **シンプルで予測可能** → コードを読めば何が起こるか分かる

#### 利用可能な Signal の種類

Remix 3 では、3種類の signal が提供されます：

1. **`this.signal`**: コンポーネントがマウント/アンマウントされた時に abort
2. **イベントコールバックの `signal`**: 関数が再度呼ばれた時、または、コンポーネントがアンマウントされた時に abort
3. **レンダー中の `signal`**: 再レンダリングされた時に abort（通常は使わない）

> 「関数を渡したら、signal をあげる。これがルール。あなたがその中で何をするか分からないからね」- Ryan Florence [01:21:25]

**このデモで学んだこと:**

1. ✅ **レースコンディションの理解**: 複数の非同期処理が同時進行する問題
2. ✅ **Signal の基本**: `AbortSignal` を使って古い処理をキャンセル
3. ✅ **fetch API との統合**: `{ signal }` を渡すだけで自動キャンセル
4. ✅ **手動チェック**: 長時間処理の後は `signal.aborted` をチェック
5. ✅ **明示的な制御**: 必要な時だけ abort する設計
6. ✅ **Web 標準**: `AbortController` は Web 標準 API

### デモ4: ListBox - Web標準の統合デモ

> 💡 [動画で確認する (4:48:56~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=17336s)
>
> **これまで学んだ複数の概念（[Remix Events](#remix-events-イベントを第一級市民に)、Web標準API）を統合した実例です！**

Ryan は、Remix 3 と並行して **コンポーネントライブラリ** を開発しており、その中核となる **ListBox** コンポーネントを通じて、Web 標準との統合方法を示します。

> 「UIフレームワークとして relevantであるためには、簡単に組み合わせられるコンポーネントが必要だ。フルスタック体験を目指している」- Ryan Florence [01:23:07]

#### ステップ1: 基礎 - ネストされたドロップダウンメニュー

> 💡 [動画で確認する (4:49:11~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=17351s)
>
> 「コンポーネントモデルが動くようになった瞬間、最も難しいネストされたドロップダウンメニューを作り始めた」- Ryan Florence [01:24:11]

まず、最も複雑なコンポーネントから開始します：

**実装されている機能:**

- **ホバーインテント**: マウスが境界を横切っても意図を理解して消えない
- **3階層のネスト**: サブメニューのサブメニューまで対応
- **キーボードナビゲーション**: 完全なアクセシビリティ対応
- **Remix Events**: カスタムイベントで駆動

**レイアウトとテーマシステム:**

コンポーネントライブラリには、`Stack`（縦）と `Row`（横）のレイアウトシステムも含まれています：

```javascript
import { Stack, Row } from "@remix/ui"

function ComponentShowcase(this: Remix.Handle) {
  return () => (
    <Stack size="xxl">
      <Stack size="medium">
        <MenuExample />
      </Stack>
      <Row>
        <Button>Primary</Button>
        <Button>Secondary</Button>
      </Row>
    </Stack>
  )
}
```

- **CSS カスタムプロパティベース**: サーバーレンダリングと相性が良い
- **型安全なサイズ指定**: `"xxl"`, `"medium"` などが型チェックされる

> 「Tim（デザイナー）のデザインが素晴らしすぎて、それに見合うものを作らなきゃという気持ちになる」- Ryan Florence [01:26:49]

![コンポーネントライブラリ](/images/remix3-introduction/demo4-component-library.png)
*図: Remix UI コンポーネントライブラリのプレビュー*

#### ステップ2: ListBox の構築 - Popover API とフォーム統合

> 💡 [動画で確認する (4:43:07~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=16987s)

ここからが本題です。ネイティブの `<select>` 要素を超える **ListBox** を構築します。

**Popover API との統合:**

Web 標準の **Popover API** を使って、ドロップダウンリストを実装します：

```javascript
function ListBox(this: Remix.Handle, props: { options: string[] }) {
  let selectedValue = props.defaultValue || null
  let isOpen = false

  return () => (
    <>
      <button
        type="button"
        popovertarget="listbox-popover"
        on={[
          // Popover の開閉を検知
          DOM.toggle(() => {
            isOpen = !isOpen
            this.update()
          })
        ]}
      >
        {selectedValue || "Select..."}
      </button>

      <div id="listbox-popover" popover>
        {/* このdivはbuttonの中にあるが、top layerに表示される */}
        <ul role="listbox">
          {props.options.map(option => (
            <li
              role="option"
              on={pressDown(() => {
                selectedValue = option
                // カスタムイベントを dispatch
                this.dispatchEvent(new CustomEvent("listbox:change", {
                  detail: { value: option },
                  bubbles: true // ← バブリングを有効化
                }))
                this.update()
              })}
            >
              {option}
            </li>
          ))}
        </ul>
      </div>
    </>
  )
}
```

**Popover API のポイント:**

1. **`popover` 属性** → 自動的にトップレイヤーに配置
2. **`popovertarget`** → ボタンとポップオーバーを接続
3. **`toggle` イベント** → 開閉を検知できる

> 「Popover API は素晴らしい。トップレイヤーに行く。イベントもある。`popoverTargetToggle` を使えば、ボタンが所有するポップオーバーがいつ開くかリッスンできる。カスタムイベントを使えば、通常は接続されていないものを接続できるんだ」- Ryan Florence [01:27:48]

**リアルなフォーム要素として動作:**

> 💡 [動画で確認する (4:44:18~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=17058s)

ListBox は、内部に実際の `<input>` を持ち、フォームの一部として動作します：

```javascript
function ListBox(this: Remix.Handle, props: { name: string, options: string[] }) {
  let selectedValue = props.defaultValue || null

  return () => (
    <>
      {/* 隠しinput: フォーム送信時に値を送る */}
      <input type="hidden" name={props.name} value={selectedValue} />

      <button type="button" popovertarget="listbox-popover">
        {selectedValue || "Select..."}
      </button>

      <div id="listbox-popover" popover>
        {/* ... オプションリスト ... */}
      </div>
    </>
  )
}

// 使用例
function FruitForm(this: Remix.Handle) {
  let formData = null

  return () => (
    <form
      on={DOM.submit((event, signal) => {
        event.preventDefault()
        formData = new FormData(event.target)
        console.log("Selected:", formData.get("fruit"))
        this.update()
      })}
    >
      <ListBox name="fruit" options={["Apple", "Banana", "Orange"]} />
      <button type="submit">Submit</button>
      <button type="reset">Reset</button>
    </form>
  )
}
```

> 「これらは本物のフォーム要素なんだ。submit すると、実際の input が入ってる。リセットボタンを押すと、デフォルト状態に戻る。なぜなら、所属するフォームの submit イベントをリッスンしてるからだ。これが通常のフォーム要素がやることだよね」- Ryan Florence [01:28:42]

**フォームのリセットへの対応:**

```javascript
function ListBox(this: Remix.Handle, props: { options: string[], defaultValue?: string }) {
  let selectedValue = props.defaultValue || null

  return () => (
    <div
      on={[
        // 親フォームのresetイベントをリッスン
        DOM.reset(() => {
          selectedValue = props.defaultValue || null
          this.update()
        })
      ]}
    >
      {/* ... ListBox UI ... */}
    </div>
  )
}
```

> 「リセットボタンを押すと、watch this（これ見て）... デフォルト状態に戻る。なぜなら、所属するフォームの reset イベントをリッスンしているからだ」- Ryan Florence [01:28:42]

#### ステップ3: イベントのバブリング

> 💡 [動画で確認する (4:46:03~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=17163s)

Remix のカスタムイベントは、DOM標準のイベントと同様に **バブリング** します。

**親要素でのイベント処理:**

```javascript
function FormWithListBox(this: Remix.Handle) {
  let selectedFruit = null

  return () => (
    <form
      on={[
        // ★ フォーム要素で ListBox の変更を検知
        ListBox.change((event) => {
          selectedFruit = event.detail.value
          console.log("ListBox changed:", selectedFruit)
          this.update()
        })
      ]}
    >
      {/* ListBox 自体には on プロップを付けない */}
      <ListBox options={["Apple", "Banana", "Orange"]} />

      <p>Selected: {selectedFruit}</p>
    </form>
  )
}
```

**バブリングの仕組み:**

```text
<form>  ← イベントがバブリングして到達
  <ListBox>  ← ここで dispatch
    <button />
    <div popover>
      <li onClick>  ← ここでクリック
```

> 「div の中に画像があったら、div に `onLoad` を付けられるよね？div 自体は何もロードしないけど、load イベントはバブリングする。同じことだ。面白いパターンが生まれるはずだよ。ListBox が本当のイベントを dispatch して、親にバブリングする。だから、イベントを上の方で処理することも、下の方で処理することも、好きなところに置ける」- Ryan Florence [01:32:44]

**実用例: 複数の ListBox を1つのハンドラで処理**

```javascript
function MultiSelectForm(this: Remix.Handle) {
  let selections = { fruit: null, vegetable: null }

  return () => (
    <form
      on={[
        // ★ すべての ListBox の変更を1つのハンドラで処理
        ListBox.change((event) => {
          const name = event.target.getAttribute("name")
          selections[name] = event.detail.value
          this.update()
        })
      ]}
    >
      <ListBox name="fruit" options={["Apple", "Banana"]} />
      <ListBox name="vegetable" options={["Carrot", "Broccoli"]} />

      <pre>{JSON.stringify(selections, null, 2)}</pre>
    </form>
  )
}
```

#### ステップ4: Web Components との互換性

> 💡 [動画で確認する (4:50:55~)](https://www.youtube.com/watch?v=xt_iEOn2a6Y&t=17455s)

セッションのクライマックス。Ryan は、Remix コンポーネントを **Web Components** として公開できることを実演します。

> 「僕らのイベントシステム全体は、ただのカスタムイベントなんだ。通常のDOMを通してバブリングする。だから、Web Componentsを含む世界の他のすべてと、すぐに互換性がある」- Ryan Florence [01:35:41]

**カスタム要素としての使用:**

```html
<!-- 普通のHTMLファイル -->
<!DOCTYPE html>
<html>
  <head>
    <script type="module" src="/remix-components.js"></script>
  </head>
  <body>
    <!-- ★ カスタム要素として使用 -->
    <rmx-disclosure>
      <disclosure-button>Toggle Content</disclosure-button>
      <disclosure-content>
        Hidden content here
      </disclosure-content>
    </rmx-disclosure>

    <script type="module">
      // カスタム要素の定義
      class RmxDisclosure extends HTMLElement {
        connectedCallback() {
          // 既存のHTMLを取得
          const button = this.querySelector('disclosure-button')
          const content = this.querySelector('disclosure-content')

          // innerHTML を消去して Remix コンポーネントをレンダリング
          this.innerHTML = ""
          const root = createRoot(this)
          root.render(
            <Disclosure>
              <Disclosure.Button>{button.innerHTML}</Disclosure.Button>
              <Disclosure.Content>{content.innerHTML}</Disclosure.Content>
            </Disclosure>
          )
        }
      }

      customElements.define('rmx-disclosure', RmxDisclosure)

      // ★ 通常のDOM APIでイベントをリッスン
      document.querySelector('rmx-disclosure')
        .addEventListener('disclosure:toggle', (e) => {
          console.log('Disclosure toggled!', e.detail)
        })
    </script>
  </body>
</html>
```

> 「これは証明のためのコンセプトだ。ハイドレーションとかやるべきだけど、これは単なる HTML ファイル。`rmx-disclosure` と `disclosure-button` があって、これらはただの Web Components だ。`addEventListener` で `disclosure:toggle` をリッスンできる」- Ryan Florence [01:36:39]

![Web Components デモ](/images/remix3-introduction/demo5-web-components.png)
*図: HTMLファイル内でカスタム要素として使用される Remix コンポーネント*

**マイクロフロントエンドへの応用:**

> 「Remix で完全なアプリを作れるだけじゃない。Web Components の中に隠すこともできる。そうすれば、世界の他の部分と簡単に互換性を持たせられる。レガシーシステムや、AI チャットアプリに埋め込むとか、そういう新しいユースケースにも対応できる」- Ryan Florence [01:37:34]

**この設計の意義:**

1. **既存システムへの段階的導入**: レガシーアプリに Remix コンポーネントを少しずつ追加
2. **フレームワーク間の相互運用**: React、Vue、Angular などと共存
3. **AI エージェントへの埋め込み**: チャットボットや AI インターフェースにコンポーネントを提供
4. **標準への準拠**: Web 標準に基づいているため、将来性がある

**このデモで学んだこと:**

1. ✅ **Popover API**: Web標準のAPIとの統合
2. ✅ **フォーム統合**: submit/resetイベントへの自動対応
3. ✅ **イベントのバブリング**: 親要素での一括処理
4. ✅ **Web Components**: 標準技術との完全な互換性
5. ✅ **型安全なコンポーネント**: TypeScript でのDX向上
6. ✅ **AI フレンドリー**: 標準技術ベースで LLM が理解しやすい

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
