---
title: 'Remix 3 の新コンポーネントライブラリを試してみた - Hooks なし、クロージャで状態管理'
emoji: '🎸'
type: 'tech'
topics: ['remix', 'react', 'typescript', 'javascript']
published: true
---

## これはなに？

:::message
この記事で紹介する `@remix-run/component` は開発中のライブラリです。仕様は今後変更される可能性が高いです。
:::

2025年1月14日(水) 19:00 から「Offers フロントエンドカンファレンス」で、この記事の内容について LT とディスカッションをします。興味がある方はぜひご覧ください。

https://offers-jp.connpass.com/event/378636/

Remix チームが開発中の `@remix-run/component` を試してみました。React とは異なるコンポーネントモデルで、タスク管理アプリを作りながら違いを理解したのでまとめます。

https://github.com/coji/remix-task-manager

デモはこちらです。

https://remix-task-manager-eight.vercel.app/

## 経緯

2025年10月に Remix チームが「Remix 3」のデモを公開しました。React Router v7 とは別のアプローチで、React に依存しない独自のコンポーネントモデルを採用していて興味深かったのですが、当時はイベント用のデモという位置づけでした。

それから約2ヶ月、2025年12月19日に `@remix-run/component` v0.2.1 として npm に公開されたので、実際に触って試してみることにしました。

## @remix-run/component とは

Remix チームが実験的に開発しているコンポーネントライブラリです。React のような仮想 DOM ベースの UI ライブラリですが、Hooks を使わず、クロージャベースで状態を管理するのが特徴です。現時点では開発中で、SSR や async コンポーネントには対応していません。

## 基本的な考え方の違い

React と @remix-run/component では、コンポーネントの実行モデルが根本的に異なります。

React ではコンポーネント関数は再レンダリングのたびに全体が実行され、状態は Hooks で保持します。一方 @remix-run/component では、コンポーネント関数は初期化時に1回だけ実行されます。関数を return し、その関数が再レンダリング時に実行される形式で、状態は普通の変数（クロージャ）で保持します。

## コンポーネントの基本構造

まず React の場合です。

```tsx
function Counter() {
  const [count, setCount] = useState(0)
  return <div>{count}</div>
}
```

`useState` で状態を宣言し、コンポーネント関数全体が毎回実行されます。

@remix-run/component では構造が異なります。

```tsx
function Counter(this: Handle) {
  let count = 0
  return () => <div>{count}</div>
}
```

外側の関数は初期化時に1回だけ実行され、return で返した関数が再レンダリング時に実行されます。`count` は普通の変数で、クロージャとしてキャプチャされます。

外側の関数（初期化フェーズ）は1回だけ実行され、return で返す関数（render フェーズ）は `update()` のたびに実行されます。状態は普通の変数として保持されます。

## 状態の更新と再レンダリング

状態を更新して画面に反映させる方法も異なります。

React では `useState` の setter を呼ぶと、自動的に再レンダリングがスケジュールされます。

```tsx
function Counter() {
  const [count, setCount] = useState(0)

  const increment = () => {
    setCount(count + 1)
  }

  return <button onClick={increment}>{count}</button>
}
```

@remix-run/component では、変数を直接変更し、`this.update()` を呼んで明示的に再レンダリングを要求します。

```tsx
function Counter(this: Handle) {
  let count = 0

  const increment = () => {
    count++
    this.update()
  }

  return () => <button on={{ click: increment }}>{count}</button>
}
```

イベントハンドラの書き方も違います。React は `onClick` ですが、@remix-run/component は `on={{ click: ... }}` という形式です。

## Props の受け取り方

@remix-run/component では、props を受け取るタイミングが2つあります。

| フェーズ | 実行タイミング | 用途 |
| --- | --- | --- |
| Setup | コンポーネント生成時（1回のみ） | 状態の初期化、イベント購読、リソース確保 |
| Render | 毎回のレンダリング時 | UI の構築、変化する props の反映 |

React では毎回すべての props を受け取りますが、@remix-run/component ではフェーズごとに使い分けます。

```tsx
// React
function Counter({ initial, label }: Props) {
  const [count, setCount] = useState(initial)
  return (
    <div>
      {label}: {count}
    </div>
  )
}

// @remix-run/component
function Counter(this: Handle, { initial }: { initial: number }) {
  let count = initial

  return ({ label }: { label: string }) => (
    <div>
      {label}: {count}
    </div>
  )
}
```

親から渡される props は同じでも、用途に応じてどちらで受け取るか決める設計です。`initial` は初期値として1回使うだけなので Setup Props、`label` は親が変更したら追従したいので Render Props で受け取ります。

### 実践的なガイドライン

| 用途 | どちらで受け取る？ |
| --- | --- |
| 初期値（`initial`, `defaultValue`） | Setup Props |
| 表示用の値（`title`, `count`, `items`） | Render Props |
| 状態を示す値（`disabled`, `selected`） | Render Props |
| コールバック関数（`onClick`, `onChange`） | どちらでも OK（Setup が多い） |
| children | Render Props |

迷ったら Render Props で受け取れば安全です。Setup で受け取るのは、明確に「初期化用」とわかる props だけにしておくと安心です。

## 実行タイミングの比較

| フェーズ | React | @remix-run/component |
| --- | --- | --- |
| 初期化 | 毎レンダリングで関数全体実行 | 1回だけ実行 |
| 再レンダリング | 関数全体を再実行 | return 内の関数のみ実行 |
| 状態の保持 | Hooks | クロージャ |
| 再レンダリングのトリガー | setter 呼び出しで自動 | `this.update()` で手動 |

## 副作用の扱い方

タイマーやイベントリスナーなど、レンダリング以外の処理（副作用）の扱い方も異なります。

React では `useEffect` で副作用を扱います。

```tsx
function Timer() {
  const [count, setCount] = useState(0)

  useEffect(() => {
    const id = setInterval(() => {
      setCount((c) => c + 1)
    }, 1000)

    return () => clearInterval(id)
  }, [])

  return <div>{count}</div>
}
```

@remix-run/component では、初期化フェーズで直接副作用を実行します。クリーンアップは `this.signal` の abort イベントで行います。

```tsx
function Timer(this: Handle) {
  let count = 0

  const id = setInterval(() => {
    count++
    this.update()
  }, 1000)

  this.signal.addEventListener('abort', () => clearInterval(id))

  return () => <div>{count}</div>
}
```

`this.signal` はコンポーネントが削除されると abort される `AbortSignal` です。React の `useEffect` のクリーンアップ関数に相当します。ただし、依存配列に相当する機能はないため、props の変更に応じた副作用の再実行は自分で制御する必要があります。

## 実践：タスク管理アプリ

ここまでの知識を使って、実際にタスク管理アプリを作ってみました。

### App コンポーネント

メインのコンポーネントです。タスクの状態管理と、サーバーからの初期データ取得を行います。

```tsx
import { createRoot, type Handle } from '@remix-run/component'

export default function App(this: Handle) {
  let tasks: Task[] = []
  let nextId = 1
  let filter: FilterType = 'all'
  let isLoading = true

  fetch('/initial-tasks.json')
    .then((res) => res.json())
    .then((data) => {
      tasks = data.tasks
      nextId = data.nextId
      isLoading = false
      this.update()
    })

  const addTask = (title: string) => {
    tasks.push({ id: nextId++, title, completed: false })
    this.update()
  }

  const toggleTask = (id: number) => {
    const task = tasks.find((t) => t.id === id)
    if (task) {
      task.completed = !task.completed
      this.update()
    }
  }

  return () => (
    <div class="mx-auto max-w-lg p-5">
      {isLoading ? (
        <p>読み込み中...</p>
      ) : (
        <>
          <TaskInput onAdd={addTask} />
          <TaskFilter current={filter} onChange={setFilter} />
          <TaskList tasks={getFilteredTasks()} onToggle={toggleTask} />
        </>
      )}
    </div>
  )
}

createRoot(document.getElementById('root')!).render(<App />)
```

### TaskInput コンポーネント

`connect` prop で DOM 要素への参照を取得し、直接値を操作しています。`onAdd` はコールバック関数なので初期化時に受け取っています。

```tsx
import type { Handle } from '@remix-run/component'

interface Props {
  onAdd: (title: string) => void
}

export default function TaskInput(this: Handle, { onAdd }: Props) {
  let inputEl: HTMLInputElement | null = null

  const handleSubmit = () => {
    if (inputEl?.value.trim()) {
      onAdd(inputEl.value.trim())
      inputEl.value = ''
    }
  }

  return () => (
    <div class="flex gap-2">
      <input
        connect={(el: HTMLInputElement) => {
          inputEl = el
        }}
        type="text"
        on={{
          keydown: (e: KeyboardEvent) => {
            if (e.key === 'Enter' && !e.isComposing) handleSubmit()
          },
        }}
      />
      <button on={{ click: handleSubmit }}>Add</button>
    </div>
  )
}
```

## データフェッチの制限

async/await でデータを取得したいところですが、現在の @remix-run/component は async コンポーネントに対応していません。

```tsx
// これは動かない
export default async function App(this: Handle) {
  const res = await fetch('/initial-tasks.json')
  const data = await res.json()
}
```

Promise チェーンを使うのが現状のパターンです。

## その他の特徴

README にはいくつか興味深い機能が紹介されています。

### イベントハンドラの AbortSignal

イベントハンドラには第2引数として `signal` が渡されます。検索ボックスで入力のたびにリクエストを送る場合、前のリクエストを自動キャンセルできます。

```tsx
<input
  on={{
    input: (event, signal) => {
      const query = event.currentTarget.value
      fetch(`/search?q=${query}`, { signal })
        .then((res) => res.json())
        .then((results) => {
          if (signal.aborted) return
        })
    },
  }}
/>
```

### css prop

疑似セレクタやネスト、メディアクエリに対応した CSS-in-JS も組み込まれています。

```tsx
<button
  css={{
    color: 'white',
    backgroundColor: 'blue',
    '&:hover': {
      backgroundColor: 'darkblue',
    },
  }}
>
  Click me
</button>
```

### connect prop（ref 相当）

React の `ref` に相当する機能です。DOM ノードへの参照を取得でき、クリーンアップ用の signal も受け取れます。

```tsx
<div
  connect={(node, signal) => {
    const observer = new ResizeObserver(() => {})
    observer.observe(node)

    signal.addEventListener('abort', () => {
      observer.disconnect()
    })
  }}
>
  Content
</div>
```

### Context API

React の Context に相当する機能です。

```tsx
function App(this: Handle<{ theme: string }>) {
  this.context.set({ theme: 'dark' })
  return () => <Header />
}

function Header(this: Handle) {
  const { theme } = this.context.get(App)
  return <header>Theme: {theme}</header>
}
```

## Web 標準 API の活用

@remix-run/component の特徴として、Web 標準 API を積極的に活用している点が挙げられます。React が独自の抽象化（合成イベント、Hooks など）を作ったのに対し、こちらは「すでにブラウザにある機能をそのまま使う」というアプローチです。

| 機能 | Web 標準 API | 用途 |
| --- | --- | --- |
| イベント購読 | `EventTarget` | Context の変更通知、カスタムイベント |
| ライフサイクル管理 | `AbortSignal` / `AbortController` | コンポーネント削除時のクリーンアップ |
| イベント発火 | `Event` | `dispatchEvent(new Event('change'))` |

この設計思想は Remix 本体とも一貫しています。Remix は Web Fetch API、FormData、Request/Response など Web 標準を重視しており、@remix-run/component もその延長線上にあります。

## 比較まとめ

| 観点 | React | @remix-run/component |
| --- | --- | --- |
| メンタルモデル | 関数が毎回実行される | 初期化とレンダリングが分離 |
| 状態管理 | Hooks (useState, useReducer) | 普通の変数 + this.update() |
| Props | 自動で最新値を受け取る | render 関数の引数で受け取る |
| 再レンダリング | 自動 | 手動 (this.update()) |
| 副作用 | useEffect + 依存配列 | 初期化フェーズで直接実行 |
| エコシステム | 成熟している | 開発中 |

どちらが良いというものではなく、トレードオフがあります。React は Hooks のルールを覚える必要がありますが、自動的な再レンダリングや props の伝播が楽です。@remix-run/component はクロージャベースで素の JavaScript に近いですが、props を render 関数で受け取る必要があります。

## まとめ

@remix-run/component を試してみました。React とは異なるアプローチで、クロージャベースの状態管理と Web 標準 API（`AbortSignal`、`EventTarget`）を中心としたクリーンアップ・イベント機構が特徴です。Hooks のルールを覚える必要がない一方で、props の受け取り方など独自の注意点があります。

現時点での制限として、SSR 未対応、async コンポーネント未対応、エコシステムが未成熟という点があります。React からの移行を考えるというより、Remix チームが提案する新しいコンポーネントモデルとして今後の発展を見守る段階です。
