---
title: "Remix 3 続報 - v3.0.0-alpha.2 の新アニメーションAPI を試す"
emoji: "🎬"
type: "tech"
topics: ["remix", "react", "animation", "typescript"]
published: true
---

## これはなに？

:::message
この記事は [Remix 3 の新コンポーネントライブラリを試してみた](https://zenn.dev/coji/articles/remix-3-component-library-trial) の続報です。基本的な使い方はそちらを参照してください。
:::

2025年1月28日にリリースされた [Remix v3.0.0-alpha.2](https://github.com/remix-run/remix/releases/tag/remix%403.0.0-alpha.2) で追加された3つのアニメーションAPI（`animate`、`spring`、`tween`）を、以前作成したタスク管理アプリに組み込んでみました。

公式ドキュメントは [@remix-run/component の README](https://github.com/remix-run/remix/tree/main/packages/component) を参照してください。

https://github.com/coji/remix-task-manager

![タスク追加のデモ](/images/remix-3-component-animation-api/01-task-add.gif)

## Web標準ベースの設計思想

このライブラリのアニメーションAPIは、**Web標準技術の上に薄いラッパーを被せる**という設計思想で作られています。独自のアニメーションエンジンを持たず、ブラウザのネイティブ機能を最大限活用しています。

| API | 内部で使用しているWeb標準 |
|-----|--------------------------|
| `animate` (enter/exit) | [Web Animations API](https://developer.mozilla.org/ja/docs/Web/API/Web_Animations_API) |
| `animate` (layout) | FLIP technique + `requestAnimationFrame` |
| `spring` | CSS Transitions |
| `tween` | `requestAnimationFrame` + Generator |

軽量で余計なランタイムがなく、ブラウザ最適化の恩恵を受けられます。DevToolsで普通に確認できるのもうれしいポイントです。

---

## animate - 宣言的アニメーション

要素の出入りやレイアウト変更を宣言的に記述できるAPIです。`enter` は要素がDOMに追加されたとき、`exit` は削除されるとき、`layout` は位置やサイズが変わったときのアニメーションを定義します。

```tsx
<li
  animate={{
    enter: {
      opacity: 0,
      transform: 'translateX(-20px)',
      duration: 150,
      easing: 'ease-out',
    },
  }}
>
  {task.title}
</li>
```

### enter / exit を使ったアイコン切り替え

テーマ切り替えボタンでは、`enter` と `exit` を組み合わせてアイコンのスワップアニメーションを実現しています。

```tsx
const iconAnimation = {
  enter: {
    transform: 'translateY(-12px) scale(0.7)',
    filter: 'blur(3px)',
    opacity: 0,
    duration: 250,
    easing: 'ease-out',
  },
  exit: {
    transform: 'translateY(12px) scale(0.7)',
    filter: 'blur(3px)',
    opacity: 0,
    duration: 200,
    easing: 'ease-in',
  },
}

{theme.value === 'light' ? (
  <span key="moon" class="absolute" animate={iconAnimation}>🌙</span>
) : (
  <span key="sun" class="absolute" animate={iconAnimation}>☀️</span>
)}
```

`key` を付けることで、条件分岐による要素の切り替え時にアニメーションがトリガーされます。

![テーマ切り替え](/images/remix-3-component-animation-api/05-theme-toggle.gif)

### layout アニメーション

`layout` を指定すると、要素の位置やサイズが変わったときにFLIP技法でスムーズにアニメーションします。

```tsx
<div
  key="highlight"
  class="absolute inset-y-1 rounded-md bg-white"
  style={{
    width: 'calc((100% - 0.5rem) / 3)',
    left: `calc(${FILTERS.indexOf(filter)} * (100% - 0.5rem) / 3 + 0.25rem)`,
  }}
  animate={{ layout: { ...spring('snappy') } }}
/>
```

フィルターボタンの背景ハイライトが、選択に応じてスライドします。

![フィルター切り替え](/images/remix-3-component-animation-api/03-filter-switch.gif)

---

## spring - 物理ベースのアニメーション

バネの物理法則に基づいたCSS transitionを生成します。自然で心地よい動きが特徴です。

3つのプリセットが用意されています。`smooth` はゆったりした動きでページ遷移やモーダル向け、`snappy` はキビキビした動きでボタンやタブ切り替え向け、`bouncy` は弾むような動きで強調や成功フィードバック向けです。

`spring.transition()` を使うと、指定したプロパティに対してスプリングベースのCSS transition文字列を生成できます。

```tsx
<input
  style={{
    transition: spring.transition(['transform', 'box-shadow'], 'snappy'),
    transform: isFocused ? 'scale(1.02)' : 'scale(1)',
    boxShadow: isFocused
      ? '0 4px 12px rgba(59, 130, 246, 0.25)'
      : '0 0 0 transparent',
  }}
/>
```

![入力フォーカス](/images/remix-3-component-animation-api/06-input-focus.gif)

チェックボックスには `bouncy` プリセットを適用しています。

```tsx
<input
  type="checkbox"
  style={{
    transition: spring.transition('transform', 'bouncy'),
    transform: task.completed ? 'scale(1.2)' : 'scale(1)',
  }}
/>
```

![チェックボックス](/images/remix-3-component-animation-api/04-checkbox-toggle.gif)

---

## tween - 命令的アニメーション

Generatorベースの命令的アニメーションAPIです。より細かい制御が必要な場合に使います。

```tsx
import { tween, easings } from '@remix-run/component'
```

### カウンターアニメーション

タスクの残り件数が変わったとき、数字がアニメーションします。

```tsx
const animateCount = (from: number, to: number) => {
  const animation = tween({
    from,
    to,
    duration: 200,
    curve: easings.easeOut,
  })

  animation.next()

  const tick = (timestamp: number) => {
    if (handle.signal.aborted) return

    const result = animation.next(timestamp)
    displayCount = Math.round(result.value)
    handle.update()

    if (!result.done) {
      requestAnimationFrame(tick)
    }
  }

  requestAnimationFrame(tick)
}
```

`handle.signal.aborted` をチェックすることで、コンポーネントがアンマウントされた場合に安全に中断できます。

### 削除アニメーション

タスク削除時は、アニメーション完了後に実際の削除処理を呼び出しています。

```tsx
const handleDelete = (id: number, onDelete: Props['onDelete']) => {
  const animation = tween({
    from: 0,
    to: 1,
    duration: 150,
    curve: easings.easeIn,
  })

  animation.next()

  const tick = (timestamp: number) => {
    if (handle.signal.aborted) return

    const result = animation.next(timestamp)
    rowEl.style.opacity = String(1 - result.value)
    rowEl.style.transform = `translateX(${result.value * 30}px)`

    if (!result.done) {
      requestAnimationFrame(tick)
    } else {
      onDelete(id)
    }
  }

  requestAnimationFrame(tick)
}
```

![タスク削除](/images/remix-3-component-animation-api/02-task-delete.gif)

`tween` はボイラープレートが増えがちなので、ヘルパー関数にまとめておくと使いやすくなります。詳しくは [リポジトリの実装](https://github.com/coji/remix-task-manager) を参照してください。

---

## 使い分け

| API | 用途 | 例 |
|-----|------|-----|
| `animate` | 要素の出入り、レイアウト変更 | リストアイテムの追加/削除、タブ切り替え |
| `spring` | CSS transitionの強化 | ホバー効果、フォーカス状態、スケール変化 |
| `tween` | 値のアニメーション、複雑な制御 | カウンター、プログレスバー、カスタム削除演出 |

基本は `animate` と `spring` で宣言的に書き、細かい制御が必要なときだけ `tween` を使うのがおすすめです。

---

## 内部実装の解説

各APIがどのようにWeb標準を活用しているか、もう少し詳しく見てみましょう。

### animate (enter/exit) の仕組み

内部では **Web Animations API** を使用しています。

```tsx
// ライブラリ内部のイメージ
function animateEnter(element: HTMLElement, config: EnterConfig) {
  const { opacity, transform, duration, easing } = config

  // 初期状態を設定
  const keyframes = [
    { opacity, transform },           // from（指定した値）
    { opacity: 1, transform: 'none' } // to（通常状態）
  ]

  // Web Animations API を呼び出し
  element.animate(keyframes, {
    duration,
    easing,
    fill: 'forwards',
  })
}
```

`exit` の場合は逆方向のアニメーションを実行し、完了後にDOMから要素を削除します。Web Animations APIを使うことで、CSSアニメーションと同等のパフォーマンスを得られます。

### animate (layout) の仕組み - FLIP技法

レイアウトアニメーションには **FLIP (First, Last, Invert, Play)** 技法を使用しています。

```tsx
// FLIP技法の流れ
function animateLayout(element: HTMLElement, springConfig: SpringConfig) {
  // 1. First: 現在の位置を記録
  const first = element.getBoundingClientRect()

  // 2. Last: DOM更新後の位置を取得（実際はレンダリング後）
  const last = element.getBoundingClientRect()

  // 3. Invert: 差分を計算して逆変換を適用
  const deltaX = first.left - last.left
  const deltaY = first.top - last.top
  element.style.transform = `translate(${deltaX}px, ${deltaY}px)`

  // 4. Play: transformを解除してアニメーション
  requestAnimationFrame(() => {
    element.style.transition = generateSpringTransition(springConfig)
    element.style.transform = 'none'
  })
}
```

これにより、CSSの `left` や `top` を直接アニメーションするよりも高パフォーマンスなレイアウトアニメーションが実現できます。

### spring の仕組み

`spring()` は物理パラメータから **CSS cubic-bezier** を生成します。

```tsx
// プリセットの内部定義（イメージ）
const presets = {
  smooth: { tension: 120, friction: 14 },
  snappy: { tension: 300, friction: 20 },
  bouncy: { tension: 400, friction: 10 },
}

function spring(preset: keyof typeof presets) {
  const { tension, friction } = presets[preset]

  // 物理パラメータからベジェ曲線を近似計算
  const bezier = approximateSpringCurve(tension, friction)
  const duration = calculateSettlingTime(tension, friction)

  return {
    duration,
    easing: `cubic-bezier(${bezier.join(', ')})`,
  }
}

// spring.transition() は上記を CSS transition 文字列に変換
spring.transition = (properties, preset) => {
  const { duration, easing } = spring(preset)
  const props = Array.isArray(properties) ? properties : [properties]
  return props.map(p => `${p} ${duration}ms ${easing}`).join(', ')
}
```

真のスプリング物理シミュレーションではなくベジェ曲線近似ですが、CSSネイティブで動作するため非常に軽量です。

### tween の仕組み

`tween` は **JavaScript Generator** を使った設計です。

```tsx
function* tween(config: TweenConfig): Generator<number, number, number> {
  const { from, to, duration, curve } = config

  // 最初の next() で初期化、startTime を受け取る準備
  let startTime: number | null = null

  while (true) {
    const currentTime: number = yield from  // timestamp を受け取る

    if (startTime === null) {
      startTime = currentTime
    }

    const elapsed = currentTime - startTime
    const progress = Math.min(elapsed / duration, 1)

    // イージング関数を適用
    const easedProgress = curve(progress)
    const value = from + (to - from) * easedProgress

    if (progress >= 1) {
      return to  // 完了
    }
  }
}
```

Generatorを使うことで、状態を内包しつつ外部から `next(timestamp)` でタイムスタンプを注入できます。ループを抜ければキャンセルも簡単で、必要なときだけ値を生成するのでメモリ効率も良いです。

これを `requestAnimationFrame` と組み合わせることで、フレームごとに正確なタイミングでアニメーション値を取得できます。

---

## まとめ

Remix v3.0.0-alpha.2 のアニメーションAPIは、宣言的な `animate`、物理ベースの `spring`、命令的な `tween` と、用途に応じて使い分けられる設計になっています。

すべてのAPIがWeb標準（Web Animations API、CSS Transitions、requestAnimationFrame）の上に構築されているため、軽量で高速に動作し、既存のWeb知識がそのまま活かせます。ブラウザのDevToolsでそのまま確認できるのも開発体験として優れています。

特に `animate` の `layout` オプションと `spring` の組み合わせは、少ないコードでリッチなUIを実現できるので試してみてください。
