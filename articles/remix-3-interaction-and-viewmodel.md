---
title: 'Remix 3 続編 - セマンティックイベントと ViewModel でフレームワーク依存を減らす'
emoji: '🎯'
type: 'tech'
topics: ['remix', 'react', 'typescript', 'javascript']
published: true
---

## これはなに？

:::message
この記事は「[Remix 3 の新コンポーネントライブラリを試してみた](https://zenn.dev/coji/articles/remix-3-component-library-trial)」の続編です。前回の記事を先に読むことをおすすめします。
:::

前回は `@remix-run/component` の基本を紹介しました。今回はさらに一歩進めて、`@remix-run/interaction` によるセマンティックなイベント処理と、ViewModel パターンによるロジック分離に取り組んでみます。

https://github.com/coji/remix-task-manager

デモはこちらです。

https://remix-task-manager-eight.vercel.app/

## なぜこれをやるのか

### @remix-run/interaction のメリット

`click` や `keydown` といった低レベルイベントには課題があります。ボタンは click だけでなく、キーボードの Enter/Space でも押せるべきですし、長押しを実装しようとするとタイマー管理が必要です。モバイルではタッチイベントも考慮しなければなりません。

`@remix-run/interaction` はこれらをセマンティックなインタラクションとして抽象化します。`press` と書くだけで、クリック・タップ・Enter/Space すべてに対応できます。

### ViewModel パターンのメリット

前回の実装では、状態とロジックがコンポーネント内に混在していました。

```tsx
function App(this: Handle) {
  let tasks: Task[] = []

  const addTask = (title: string) => {
    tasks.push({ id: nextId++, title, completed: false })
    this.update()  // ← UI への通知がロジックに混入
  }
  // ...
}
```

この書き方だと、テストにはコンポーネントのコンテキストが必要になりますし、別の UI で同じロジックを使いたいときに困ります。状態変更のたびに `this.update()` が散らばって見通しも悪くなります。

ViewModel パターンでロジックを分離すれば、純粋な TypeScript クラスとしてテスト・再利用できます。

## @remix-run/interaction とは

`@remix-run/interaction` は EventTarget ベースのイベントハンドリングライブラリです。`click` や `keydown` といった低レベルなイベントを、`press`（クリック/タップ/Enter）や `longPress`（長押し）といったセマンティックなインタラクションに抽象化します。

### インストール

```bash
pnpm add @remix-run/interaction
```

### 基本的な使い方

従来のイベントハンドリングと比較してみます。

```tsx
// 従来：低レベルイベント
<button on={{ click: handleSubmit }}>Add</button>

// @remix-run/interaction：セマンティック
import { press } from '@remix-run/interaction/press'

<button on={{ [press]: handleSubmit }}>Add</button>
```

`press` はマウスクリック、タッチ、キーボード Enter/Space を統合したインタラクションです。

## 実装例：タスク管理アプリへの適用

前回作成したタスク管理アプリに `@remix-run/interaction` を導入します。

### TaskInput - press インタラクション

ボタンのクリック/キーボード操作を `press` で統一します。

```tsx
import type { Handle } from '@remix-run/component'
import { press } from '@remix-run/interaction/press'

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
      <button type="button" on={{ [press]: handleSubmit }}>
        Add
      </button>
    </div>
  )
}
```

### TaskItem - longPress で編集モード

タスク名を長押し（500ms）で編集モードに入ります。

```tsx
import type { Handle } from '@remix-run/component'
import { longPress, press } from '@remix-run/interaction/press'
import type { Task } from '../types'

interface Props {
  task: Task
  onToggle: (id: number) => void
  onDelete: (id: number) => void
  onEdit: (id: number, title: string) => void
}

export default function TaskItem(this: Handle) {
  let isEditing = false
  let editValue = ''

  const startEdit = (title: string) => {
    isEditing = true
    editValue = title
    this.update()
  }

  const cancelEdit = () => {
    isEditing = false
    this.update()
  }

  const confirmEdit = (id: number, onEdit: Props['onEdit']) => {
    if (editValue.trim()) {
      onEdit(id, editValue.trim())
    }
    isEditing = false
    this.update()
  }

  return ({ task, onToggle, onDelete, onEdit }: Props) => (
    <li tabindex={0} class="flex items-center gap-3 py-3">
      <input
        type="checkbox"
        checked={task.completed}
        on={{ change: () => onToggle(task.id) }}
      />

      {isEditing ? (
        <input
          type="text"
          value={editValue}
          on={{
            input: (e: Event) => {
              editValue = (e.target as HTMLInputElement).value
            },
            keydown: (e: KeyboardEvent) => {
              if (e.key === 'Enter') confirmEdit(task.id, onEdit)
              if (e.key === 'Escape') cancelEdit()
            },
            blur: cancelEdit,
          }}
        />
      ) : (
        <span
          on={{
            [longPress]: (e: Event) => {
              e.preventDefault()
              startEdit(task.title)
            },
          }}
        >
          {task.title}
        </span>
      )}

      <button type="button" on={{ [press]: () => onDelete(task.id) }}>
        Delete
      </button>
    </li>
  )
}
```

`longPress` は 500ms の長押しを検出します。タップやクリックでは発火しません。

### TaskList - 矢印キーナビゲーション

上下キーでタスク間をフォーカス移動できるようにします。

```tsx
import type { Handle } from '@remix-run/component'
import { arrowDown, arrowUp } from '@remix-run/interaction/keys'
import type { Task } from '../types'
import TaskItem from './TaskItem'

interface Props {
  tasks: Task[]
  onToggle: (id: number) => void
  onDelete: (id: number) => void
  onEdit: (id: number, title: string) => void
}

export default function TaskList(this: Handle) {
  let listEl: HTMLUListElement | null = null

  const handleArrowUp = () => {
    if (!listEl) return
    const items = listEl.querySelectorAll<HTMLLIElement>('li[tabindex]')
    const currentIndex = Array.from(items).indexOf(
      document.activeElement as HTMLLIElement,
    )
    if (currentIndex > 0) {
      items[currentIndex - 1]?.focus()
    }
  }

  const handleArrowDown = () => {
    if (!listEl) return
    const items = listEl.querySelectorAll<HTMLLIElement>('li[tabindex]')
    const currentIndex = Array.from(items).indexOf(
      document.activeElement as HTMLLIElement,
    )
    if (currentIndex < items.length - 1) {
      items[currentIndex + 1]?.focus()
    }
  }

  return ({ tasks, onToggle, onDelete, onEdit }: Props) => (
    <ul
      connect={(el: HTMLUListElement) => {
        listEl = el
      }}
      on={{
        [arrowUp]: handleArrowUp,
        [arrowDown]: handleArrowDown,
      }}
    >
      {tasks.map((task) => (
        <TaskItem
          key={task.id}
          task={task}
          onToggle={onToggle}
          onDelete={onDelete}
          onEdit={onEdit}
        />
      ))}
    </ul>
  )
}
```

`tabindex={0}` を持つ `<li>` 要素間をキーボードでナビゲートできます。

## ViewModel パターンへのリファクタリング

ここまでの実装では、状態とロジックがコンポーネント内に混在しています。これを ViewModel パターンで分離します。

### 現状の問題

```tsx
function App(this: Handle) {
  // 状態
  let tasks: Task[] = []
  let nextId = 1
  let filter: FilterType = 'all'
  let isLoading = true

  // ロジック（状態変更 + this.update()）
  const addTask = (title: string) => {
    tasks.push({ id: nextId++, title, completed: false })
    this.update()  // ← UIへの通知がロジックに混入
  }

  // 派生状態
  const getFilteredTasks = () => { ... }
  const getActiveCount = () => { ... }

  return () => ( ... )
}
```

ロジックに `this.update()` が散らばっており、単体テストが困難（コンポーネントのコンテキストが必要）で、再利用もしにくい状態です。

### TaskViewModel の実装

前回の記事で紹介した ThemeStore と同じパターンで、EventTarget を継承した ViewModel を作ります。

```tsx
// src/stores/TaskViewModel.ts
import type { FilterType, Task } from '../types'

export class TaskViewModel extends EventTarget {
  tasks: Task[] = []
  nextId = 1
  filter: FilterType = 'all'
  isLoading = true

  async load() {
    const res = await fetch('/initial-tasks.json')
    const data = (await res.json()) as { tasks: Task[]; nextId: number }
    this.tasks = data.tasks
    this.nextId = data.nextId
    this.isLoading = false
    this.emit()
  }

  addTask(title: string) {
    this.tasks.push({ id: this.nextId++, title, completed: false })
    this.emit()
  }

  toggleTask(id: number) {
    const task = this.tasks.find((t) => t.id === id)
    if (task) {
      task.completed = !task.completed
      this.emit()
    }
  }

  deleteTask(id: number) {
    this.tasks = this.tasks.filter((t) => t.id !== id)
    this.emit()
  }

  editTask(id: number, title: string) {
    const task = this.tasks.find((t) => t.id === id)
    if (task) {
      task.title = title
      this.emit()
    }
  }

  setFilter(filter: FilterType) {
    this.filter = filter
    this.emit()
  }

  clearCompleted() {
    this.tasks = this.tasks.filter((t) => !t.completed)
    this.emit()
  }

  get filteredTasks(): Task[] {
    switch (this.filter) {
      case 'active':
        return this.tasks.filter((t) => !t.completed)
      case 'completed':
        return this.tasks.filter((t) => t.completed)
      default:
        return this.tasks
    }
  }

  get activeCount(): number {
    return this.tasks.filter((t) => !t.completed).length
  }

  get hasCompleted(): boolean {
    return this.tasks.some((t) => t.completed)
  }

  private emit() {
    this.dispatchEvent(new Event('change'))
  }
}
```

### リファクタリング後のコンポーネント

```tsx
// src/entry.tsx
import { createRoot, type Handle } from '@remix-run/component'
import { TaskViewModel } from './stores/TaskViewModel'

function App(this: Handle) {
  const vm = new TaskViewModel()

  // 変更を購読
  this.on(vm, { change: () => this.update() })

  // 初期ロード
  vm.load()

  return () => (
    <ThemeProvider>
      <div class="mx-auto max-w-lg p-5 font-sans">
        {vm.isLoading ? (
          <p>読み込み中...</p>
        ) : (
          <>
            <TaskInput onAdd={(title) => vm.addTask(title)} />
            <TaskFilter current={vm.filter} onChange={(f) => vm.setFilter(f)} />
            <TaskList
              tasks={vm.filteredTasks}
              onToggle={(id) => vm.toggleTask(id)}
              onDelete={(id) => vm.deleteTask(id)}
              onEdit={(id, title) => vm.editTask(id, title)}
            />

            {vm.tasks.length > 0 && (
              <TaskFooter
                activeCount={vm.activeCount}
                hasCompleted={vm.hasCompleted}
                onClearCompleted={() => vm.clearCompleted()}
              />
            )}
          </>
        )}
      </div>
    </ThemeProvider>
  )
}

createRoot(document.getElementById('root')!).render(<App />)
```

### メリット

ViewModel は純粋な TypeScript なので vitest でテストできます。コンポーネントは表示のみ、ロジックは ViewModel と関心が分離され、別の UI でも同じ ViewModel を使えます。getter で派生状態を定義すれば型推論も効きます。

### MVVM との対応

```
┌─────────────────────────────────────────┐
│  View（コンポーネント）                  │
│  - 表示のみ                             │
│  - this.on(vm, { change }) で購読       │
├─────────────────────────────────────────┤
│  ViewModel（EventTarget継承クラス）      │
│  - 状態 + ロジック                       │
│  - dispatchEvent で変更通知             │
├─────────────────────────────────────────┤
│  Model（型定義 + API）                   │
│  - Task, FilterType                     │
│  - fetch('/initial-tasks.json')         │
└─────────────────────────────────────────┘
```

## 一休レストランの記事との接点

一休レストランが公開した記事「[React/Remix への依存を最小にするフロントエンド設計](https://user-first.ikyu.co.jp/entry/2024/08/05/142626)」では、フレームワーク依存を最小化する三層アーキテクチャを提唱しています。

```
┌─────────────────────────────────────────┐
│  コンポーネント層（React）              │  ← 表示責務のみ
├─────────────────────────────────────────┤
│  カスタムフック層                       │  ← アダプター
├─────────────────────────────────────────┤
│  Vanilla JS ロジック層                  │  ← 可搬性を確保
└─────────────────────────────────────────┘
```

@remix-run/component では、この設計がアーキテクチャ的な工夫なしに自然に達成できます。

| 観点 | React（一休方式） | @remix-run/component |
|------|------------------|---------------------|
| ロジック層 | Vanilla JS Store | TaskViewModel (EventTarget) |
| アダプター | カスタムフック | 不要（this.on で直接購読） |
| 状態管理 | useReducer + Context | 普通の変数 |
| ボイラープレート | 多い | 少ない |

一休の記事が目指している「フレームワーク依存の最小化」は、`@remix-run/component` + ViewModel パターンで自然に実現できます。

### React で同じ ViewModel を使うなら

ちなみに、今回作った `TaskViewModel` を React で使うとしたらこうなります。

```tsx
import { useSyncExternalStore } from 'react'
import { TaskViewModel } from './stores/TaskViewModel'

const vm = new TaskViewModel()

function useTaskViewModel() {
  return useSyncExternalStore(
    (callback) => {
      vm.addEventListener('change', callback)
      return () => vm.removeEventListener('change', callback)
    },
    () => vm
  )
}

function App() {
  const vm = useTaskViewModel()
  return <TaskList tasks={vm.filteredTasks} />
}
```

EventTarget ベースの ViewModel は React でもそのまま使えます。`useSyncExternalStore` がアダプター役になるだけで、ロジック自体は共通です。まさに一休方式ですね。

## 将来の展望

### ドメインイベントの拡張

今回の TaskViewModel では単純な `change` イベントだけを使いましたが、より複雑なドメインモデルでは、セマンティックなイベントを定義することもできます。

`@remix-run/interaction` には `TypedEventTarget` というヘルパーがあり、型安全なイベント定義ができます。

```tsx
import { TypedEventTarget } from '@remix-run/interaction'

// イベントの型を定義
type ReservationEvents = {
  'reservation:confirmed': CustomEvent<{ id: string; confirmedAt: Date }>
  'reservation:cancelled': CustomEvent<{ id: string; reason: string }>
}

// 予約システムの例
class ReservationViewModel extends TypedEventTarget<ReservationEvents> {
  confirm(id: string) {
    // 確定処理
    this.dispatchEvent(new CustomEvent('reservation:confirmed', {
      detail: { id, confirmedAt: new Date() }
    }))
  }

  cancel(id: string, reason: string) {
    // キャンセル処理
    this.dispatchEvent(new CustomEvent('reservation:cancelled', {
      detail: { id, reason }
    }))
  }
}

// 購読側 - addEventListener の型が効く
vm.addEventListener('reservation:confirmed', (e) => {
  console.log(e.detail.id, e.detail.confirmedAt)  // 型補完が効く
})
```

`TypedEventTarget` も `EventTarget` も Web 標準 API（またはそれを拡張したもの）なので、React でも Remix でも、あるいは将来別のフレームワークが出てきても同じように使えます。ドメインロジックがフレームワークに依存しないというのは、こういうことです。

タスク管理アプリでここまでやるのはオーバーテクノロジーですが、予約システムや決済フローのような複雑なドメインでは、こうしたセマンティックなイベント設計が活きてきます。

### API バックエンドとの結合

Repository パターンを導入すれば、API との結合も ViewModel に閉じ込められます。

```tsx
export interface TaskRepository {
  getAll(): Promise<{ tasks: Task[]; nextId: number }>
  create(title: string): Promise<Task>
  update(id: number, patch: Partial<Task>): Promise<Task>
  delete(id: number): Promise<void>
}

export class TaskViewModel extends EventTarget {
  constructor(private repo: TaskRepository) {
    super()
  }

  async addTask(title: string) {
    // 楽観的更新
    const tempId = this.nextId++
    this.tasks.push({ id: tempId, title, completed: false })
    this.emit()

    const created = await this.repo.create(title)
    // サーバーからの ID で置き換え
    // ...
  }
}
```

ここまでで、`@remix-run/interaction` によるセマンティックなイベント処理と、ViewModel パターンによるロジック分離を紹介しました。一休レストランの記事が目指していたフレームワーク依存の最小化が、`@remix-run/component` では自然に達成できることも確認できました。

ここからは、この設計思想をもう少し広い視点で考えてみます。

### モバイルへの可能性

これは私見ですが、`@remix-run/component` の設計（普通の変数、EventTarget ベースの状態管理、セマンティックなインタラクション）は、SwiftUI や Jetpack Compose といったモバイル UI フレームワークと共通するものがあるように感じます。

ビジネスロジックを純粋な TypeScript（EventTarget 継承の ViewModel）で実装しておけば、将来「Remix Native」のようなモバイル向けフレームワークを作ることもできるのでは？そうなれば同一のロジックを Web とモバイルで共有できたりするのでは？と妄想しています。

## React と比べて

正直なところ、自分も当面は `@remix-run/component` を仕事で使う予定はありません。React Router v7 を使うつもりです。

ただ、Remix でのやり方を知ることで、React に完全依存する部分を減らす書き方のヒントが得られると感じています。

### class について

自分も正直、class には抵抗があります。React の関数コンポーネント + Hooks に慣れていると、class ベースの ViewModel は古臭く感じます。

ただ、EventTarget 自体が class を前提とした API なので、継承して使うのが自然な形になります。そういう書き方もありかもしれない、という程度に受け止めてもらえれば。

### AI によるコード生成について

AI によるコード生成が当たり前になった今、「どのフレームワークでも AI が書いてくれるから大丈夫」と思うかもしれません。

ただ、これも結局プロダクトの寿命の話に帰着します。AI で大量生産した画面群を、5年後10年後に仮に React が本流でなくなったとき、自分ならどうするだろう？AI を使うにしたって全部書き換えるのか？と考えてしまいます。

ロジックがフレームワークに依存していなければ、UI 層だけの書き換えで済みます。AI 時代だからこそ、「何を生成させるか」の設計判断は人間がすべきで、その判断軸として「フレームワーク依存の最小化」は有効だと思います。

### 考えてほしいこと

React は素晴らしいライブラリで、エコシステムも成熟しています。今すぐ乗り換える必要はありません。

ただ、フレームワークの寿命よりプロダクトの寿命が長くなることは珍しくありません。一休レストランは2006年から続いています。そのとき、ロジックがフレームワークに依存していない設計は価値を持ちます。

この記事が「React の当たり前を相対化する」きっかけになれば嬉しいです。次のセクションで告知するイベントでも、このテーマで話す予定です。

## まとめ

`@remix-run/interaction` と ViewModel パターンを組み合わせることで、press や longPress、arrowUp/arrowDown といったセマンティックなインタラクションを実現し、EventTarget 継承の ViewModel でロジックを分離できました。一休方式が目指すフレームワーク依存の最小化も自然に達成できます。

@remix-run/component のエコシステムはまだ発展途上ですが、Web 標準 API を活用した設計思想は、長期的なメンテナビリティを考える上で参考になります。

## 告知

2025年1月14日(水) 19:00 から「Offers フロントエンドカンファレンス」で、この記事の内容について LT とディスカッションをします。興味がある方はぜひご覧ください。

https://offers-jp.connpass.com/event/378636/
