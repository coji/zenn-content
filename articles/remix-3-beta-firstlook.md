---
title: "Remix v3 beta を触ってみる - React 経験者からみたフルスタックの新しい選択肢"
emoji: "🧩"
type: "tech"
topics: ["remix", "react", "typescript", "fullstack"]
published: true
---

## これはなに？

Remix 開発チームがついに `Remix 3 Beta Preview` を公開しました。

https://remix.run/blog/remix-3-beta-preview

Remix 3 は、これまで React と React Router の上に乗るメタフレームワークだった位置づけから方向転換して、独立したフルスタックフレームワークとして作り直されています。UI ランタイム・コンポーネントからルーティング・認証・フォーム・データ層・テスト・CLI まで、Web アプリ作りに必要なものを `remix` という単一パッケージに揃えた構成です。

公式ブログの言葉を借りれば、目指しているのは「Remix を入れたら、もう作り始められる」状態です。

実際に触ってみないと感覚がつかめないので、以前作ったタスクマネージャーアプリを v3 beta で書き直してみました。その作業を通じて見えてきた Remix v3 の姿を、React 経験者の目線で紹介します。

- デモ: https://remix-task-manager-eight.vercel.app/
- リポジトリ: https://github.com/coji/remix-task-manager
- 移行 PR: https://github.com/coji/remix-task-manager/pull/8

## Remix v3 はどういう立ち位置のもの？

仕組みとしては React に近い「JSX を書いて関数コンポーネントに props を渡す」スタイルですが、内部的には React も React Router も使っていません。代わりに、ブラウザの Web 標準 API のうえに薄いランタイムを敷いています。たとえば、

- 要素の出入りのアニメーションは Web Animations API
- ポップオーバーの開閉は HTML の `popover` 属性
- サーバとの通信は `fetch` API、ルータは Web の `Request` / `Response`

といった具合に、「ブラウザにある機能はブラウザに任せる」設計です。独自の仮想 DOM と言うよりは、Web プラットフォームを薄い JSX レイヤから操作する、という感覚に近いです。

`npm install remix` を 1 つ入れると、その下から `remix/ui` (UI ランタイム), `remix/ui/button` (Button コンポーネント), `remix/data-schema` (バリデーション), `remix/fetch-router` (ルータ), `remix/auth` (認証) のように、機能別のサブパスがずらっと使えるようになります。

## まずはコンポーネントを書いてみる

雰囲気をつかむには、シンプルなカウンターを書いてみるのが早いです。

React で書くとこんな感じですよね。

```tsx
function Counter() {
  const [count, setCount] = useState(0)
  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  )
}
```

Remix v3 だとこう書きます。

```tsx
import { type Handle, on } from 'remix/ui'

function Counter(handle: Handle) {
  let count = 0
  const inc = () => {
    count++
    handle.update()
  }
  return () => (
    <button mix={on('click', inc)}>Count: {count}</button>
  )
}
```

ここに Remix v3 のコンポーネントモデルが詰まっているので、ひとつずつ見ていきます。

### コンポーネントは「セットアップ + 描画関数」

Remix v3 のコンポーネントは 2 段構えです。一度しか走らない外側の関数 (セットアップ) と、再描画のたびに呼ばれる内側の関数 (描画) に分かれています。

外側の関数は最初のマウント時に 1 回だけ実行されます。`let count = 0` のような状態の宣言や、外部のストアの購読といった「初期化」はここに書きます。React で言えば `useState` の初期値や `useEffect(..., [])` の中身に相当します。

そしてこの関数が描画関数を返します。再描画はこの内側の関数だけが繰り返し呼ばれます。外側のセットアップで宣言したローカル変数や関数の参照は、再描画しても変わりません。React で `useMemo` や `useCallback` を予防的に挟んでいた場面の多くは、そのまま書くだけで済みます。

### 状態は「ふつうのローカル変数」

React の `useState` を覚えるとき少し戸惑うのが、「ローカル変数に見えるけど、再レンダリングで再生成されないように特別なフックで包んでいる」という二重の世界観でした。

Remix v3 では、状態は本当にただのローカル変数です。`let count = 0` と書けば、それがそのコンポーネントインスタンスの状態になります。クロージャに閉じ込められていて、再描画しても消えません。

その代わり、変えたあとに明示的に `handle.update()` を呼ぶ必要があります。React の `setState` 相当の役割ですね。「変えた」と「再描画したい」を別の動作として書く分、ちょっと手数は増えますが、何が起きているかは追いやすいです。

### 外側のストアと繋ぐ

複数のコンポーネントから同じ状態を見たいときは、`EventTarget` を継承したただのクラスを書いてしまえば済みます。今回のタスクマネージャーでも、タスクの一覧を持つ `TaskViewModel` をクラスとして書いて、変更があれば `change` イベントを `dispatchEvent` するようにしました。

コンポーネント側はそれを購読すれば、変化のたびに再描画されます。

```tsx
function App(handle: Handle) {
  const vm = new TaskViewModel()
  addEventListeners(vm, handle.signal, {
    change: () => handle.update(),
  })
  vm.load()
  return () => /* ... */
}
```

`handle.signal` は、コンポーネントがアンマウントされたときに自動で `abort()` される `AbortSignal` です。`addEventListeners` にこれを渡しておけば、後片付けは Remix がやってくれます。React の `useEffect` のクリーンアップ関数を書く感覚に近いですが、`AbortController` というブラウザ標準の仕組みに乗っかっている点で素直です。

## 要素の振る舞いを「混ぜる」 mix prop

React では、ボタンに「クリックで何かする」「マウント時に DOM 参照を取る」「focus 時にスタイルを変える」のような振る舞いを足したいとき、JSX の prop で書いたり (`onClick`, `ref`)、カスタムフックを呼んだり (`useFocus`, `useHotkeys`)、`forwardRef` でラップしたりと、混ぜ方が複数のレイヤーに分かれていました。

Remix v3 では、要素の振る舞いは `mix` という 1 本の prop に配列で混ぜて渡す設計に統一されています。

```tsx
<button
  mix={[
    css({ padding: 8, borderRadius: 8 }),
    on('click', handleClick),
    on('keydown', (e) => {
      if (e.key === 'Escape') handleEscape()
    }),
    ref((el) => {
      buttonEl = el
    }),
  ]}
>
  保存
</button>
```

ここで使っている `css()`, `on()`, `ref()` はどれも mixin 関数と呼ばれるもので、要素に振る舞いを足すための関数です。`mix` の配列に好きな順番で並べられます。配列の中身は `false` や `null` でも構わないので、条件によって混ぜたり混ぜなかったりも自然に書けます。

```tsx
mix={[
  baseCss,
  isActive && activeCss,
  on('click', toggle),
]}
```

React で `useEffect` / `useRef` / `className` / `onClick` のように複数のレイヤに分かれていた要素の振る舞いを、ぜんぶ 1 つの配列に平らに並べ直す感覚です。スタイル・イベント・ref・属性のデフォルト値、どれもこの仕組みに乗ります。

## アニメーションも標準でついてくる

入退場アニメーションも、`remix/ui/animation` から `mix` で混ぜ込めます。

```tsx
import { animateEntrance, animateExit, spring } from 'remix/ui/animation'

const slideIn = animateEntrance({
  opacity: 0,
  transform: 'translateX(-20px)',
  duration: 150,
  easing: 'ease-out',
})

<li mix={[slideIn]}>
  {task.title}
</li>
```

`animateEntrance` / `animateExit` の中身は Web Animations API の薄いラッパーで、ブラウザの DevTools のアニメーションパネルにそのまま見えます。`spring()` のほうは少し別の作りで、バネ物理の curve を CSS の `linear()` 関数として書き出して CSS Transition で動かす仕組みです。どちらにせよ、専用のアニメーションエンジンを抱え込まず、ブラウザの実装に乗っかっています。

タスクのカウンタが「2 から 5 にぬるっと変わる」みたいな数値補間も、`tween()` というジェネレータベースのヘルパで素直に書けます。React なら framer-motion を入れるところで、Remix v3 はこのへんも素のランタイムで賄います。

## 使えるコンポーネントが揃ってきた

v3 beta の特徴的なところで、UI コンポーネントが結構な数、最初から同梱されています。タスクマネージャーで実際に使ったのは Button, Menu, Popover, Glyph (アイコン) の 4 つですが、ほかにも Accordion / Combobox / Listbox / Select / Tabs / Breadcrumbs / Anchor / Separator が `remix/ui/...` から import できます。

たとえば「⋯ メニュー」を開いて「編集 / 削除」を選ぶ UI は、こんな具合です。

```tsx
import { Menu, MenuItem, onMenuSelect } from 'remix/ui/menu'
import { Glyph } from 'remix/ui/glyph'

<Menu
  label={<Glyph name="menu" />}
  mix={[
    onMenuSelect((event) => {
      if (event.item.name === 'edit') startEdit()
      if (event.item.name === 'delete') requestDelete()
    }),
  ]}
>
  <MenuItem name="edit">編集</MenuItem>
  <MenuItem name="delete">削除</MenuItem>
</Menu>
```

これだけで、キーボード操作 (矢印キーで項目移動・Enter で選択・Esc で閉じる) と ARIA 属性が一通り揃ったメニューが手に入ります。React で同じことをやろうとすると、Radix UI や React Aria を使うことになるあのあたりが、`remix/ui/menu` で標準提供されているイメージです。

これらのコンポーネントは、最近の HTML に入った `popover` 属性 (要素を最前面のレイヤーに持ち上げて開閉できる仕組み) を内部で使っています。開閉時のレイヤー管理、いわゆる z-index 戦争もブラウザに任せられます。

## テーマもセットで提供される

Button や Menu のような既製コンポーネントが揃うと「色や角丸はどう揃えるんだろう」が気になります。

Remix v3 ではこれも `remix/ui/theme` で用意されています。`RMX_01` という標準テーマがあって、コンポーネントツリーのどこかでこれを描画しておくと、内部の `Button` などはこのトークンを参照して色が決まります。

```tsx
import { RMX_01, RMX_01_GLYPHS } from 'remix/ui/theme'

function App() {
  return () => (
    <>
      <RMX_01 />
      <RMX_01_GLYPHS />
      {/* 以下、Button や Menu はこのテーマで描画される */}
    </>
  )
}
```

ダークテーマは、`createTheme` でトークン値を上書きしたバリアントを別のセレクタ (たとえば `.dark`) にスコープして並べておくと作れます。今回は html 要素に `dark` クラスをトグルするだけのシンプルな実装にしました。

トークンへのアクセスは `theme.colors.text.primary`, `theme.space.lg` のような型付きの参照で、`css()` の mixin 関数の中から自然に使えます。

```tsx
const cardCss = css({
  padding: theme.space.lg,
  border: `1px solid ${theme.colors.border.default}`,
  borderRadius: theme.radius.lg,
})
```

## バリデーションも、テストも、CLI も

UI 周り以外も一通り揃っています。フォーム入力のバリデーションは `remix/data-schema` で、Zod 風の API です。

```tsx
import { string } from 'remix/data-schema'
import { minLength, maxLength } from 'remix/data-schema/checks'

const taskTitleSchema = string().pipe(
  minLength(1),
  maxLength(100),
)
```

テストは `remix test` というコマンドで Vitest 互換のランナーが走ります。`remix new` で雛形を作って、`remix routes` でルートツリーを確認、`remix doctor` でプロジェクトの健康診断、というように CLI も付いています。

サーバ側に踏み込むと、`remix/fetch-router` の Web Fetch API ベースのルータや、`remix/auth` の OAuth/OIDC プリミティブ、`remix/data-table` の型付き ORM、11 種類のミドルウェア (CORS / CSRF / セッション / 静的ファイル配信など) があります。今回のタスクマネージャーはクライアント単独の SPA に絞ったので使っていませんが、SSR にしたい・ログインを足したい・DB に永続化したい、というときに同じパッケージの中で広げていける構成です。

## 触ってみての評価

良かった点と困った点を分けて並べます。

### 良かった点

- 1 つのパッケージから UI・アニメーション・バリデーション・ルータ・認証・DB・テストまでひととおり揃うので、最初に何を入れるかで迷う時間が短くて済みます。
- 状態がふつうのローカル変数なので、`useEffect` の依存配列や `useMemo` / `useCallback` の予防的なメモ化を考える場面が減ります。
- アニメーション・ポップオーバー・メニューといった「外部ライブラリを足して始めるのが定番」だった部分が同梱されています。
- `Request` / `Response` / `popover` / `AbortSignal` といった Web 標準 API がそのまま顔を出すので、フレームワーク固有の知識を覚える前に Web の知識でだいたい読めます。

### 困った点

- ドキュメントがまだ薄いです。型定義やテストファイルを読みに行かないと挙動が分からない場面がそこそこありました。
- 既存の React エコシステム (Radix UI / shadcn-ui / React Hook Form / TanStack Query / framer-motion など) がそのままは効きません。同等機能が `remix/...` 側にあるとはいえ、StackOverflow や記事で問題を引ければすぐ済む類いの救済が、Remix v3 ではほぼ得られません。
- ベータなので API はまだ仕様変更されます。alpha 時代の主要な prop が一通り入れ替わっていることからも、beta から RC にかけてさらに変わる可能性は十分あります。
- vdom の reconcile 周りで再現条件の絞り込みに苦労する不具合に当たりました。タスクのタイトルを編集して Enter を押した直後に、編集前の `<span>` が DOM に残ったまま編集後の `<span>` と並んで表示される現象です。最終的な原因は、Enter で確定した直後にブラウザが続けて `blur` を発火し、blur ハンドラの `cancelEdit` がもう一度走って二重に再描画がスケジュールされる、というロジック起因のものでした。初動ではフレームワーク側のバグを疑い続けることになり、調査コストはそれなりにかかりました。
- `mix={[...]}` の配列に css・event・ref・animation・属性のデフォルトをぜんぶ詰めていく書き方は、柔軟ではある反面、要素の責務が見えにくくなる側面があります。React の「props を別の概念で分ける」設計より見通しが良いとは限らないです。
- AI コーディングエージェントの訓練データに含まれる Remix v3 のコード量が現時点では極端に少なく、素で聞くと alpha 時代の API がそのまま返ってきます。ただし opensrc などで Remix v3 のソースコードをサブエージェントに読ませる前提なら、実装そのものを参照するので訓練データの古さはほぼ問題になりません。むしろブログ記事や StackOverflow の古い断片を踏むより正確です。「素のモデルだけで戦う」と苦しいが、「ソースを読ませる運用を組める」なら新しい OSS でも十分扱える、という温度感です。

### Next.js / React Router / TanStack Start と比べてどうか

React で書く現代的なフルスタックフレームワークの選択肢としては、Next.js (App Router)、React Router v7 (旧 Remix が合流したもの)、TanStack Start あたりが現実的なところです。Remix v3 を含めて並べると、こういう住み分けになります。

- Next.js は機能・ホスティング (Vercel) の統合・エコシステムの厚みが頭ひとつ抜けています。RSC やファイルベース規約が好き嫌いの分かれ目で、規約に乗せたら書く量はいちばん少ないです。一方で、自分の体感としては App Router はフレームワーク側のコードが厚く、サーバの実行メモリ・cold start・ビルド時間がいずれも軽くはないと感じています。Remix v3 や TanStack Start のほうが runtime そのものは薄く感じます。
- React Router v7 は「旧 Remix の loader / action 思想を React Router の上で素直に書ける」立ち位置です。Remix v2 から触っていたチームなら違和感なく移行できますし、Remix v3 とは思想を共有していますが API は別物です。
- TanStack Start は型安全なルーティングや TanStack 系 (Query / Form / Table) との親和性が魅力で、最近採用例が増えてきている印象です。React の自由度を保ちつつ、ルーティング・データフェッチ周りに強い味付けがされています。
- Remix v3 はそれらと違って、React 自体に乗っていません。UI ランタイムから自前で持っているぶん概念的にはきれいですが、その代償として React エコシステム (shadcn-ui / Radix UI / React Hook Form など) が一切使えなくなります。

機能の網羅性で言うと、Next.js / React Router / TanStack Start で書けて Remix v3 で書けないものはほとんどありません。けれどエコシステムの厚み・プロダクション実績・周辺ツール (ホスティング、モニタリング、デプロイ)・チームに馴染んだ React 知識の流用しやすさを含めた総合点で見ると、現時点で Remix v3 を選ぶ理由はかなり限定的です。

### AI が書く時代にフレームワークを選ぶ意味

最近はコードもレビューも AI が回すので、フロントエンドフレームワークの選定は「人間にとっての DX」よりも「AI と runtime にとっての適性」で見たほうが実態に合っているのかもしれない、とよく考えます。自分なりに気にしている軸を並べてみます。

- AI の成功率: 素のモデルに聞くだけだと、訓練データの量で Next.js が圧倒的に有利だと感じます。一方、opensrc などでサブエージェントにソースを直接読ませる運用を組めるなら、Remix v3 のように「単一パッケージで参照範囲が狭い・Web 標準に直結している」設計はむしろ調査しやすく、訓練データの少なさは思ったほど痛くないように感じています。
- AI が詰まったときの診断容易性: コンパイラ的な変換が薄く、Web 標準にそのまま落ちるフレームワークのほうが、AI 自身が一次資料 (MDN・仕様) に降りて考えやすい印象があります。この観点では Remix v3 が個人的には書きやすかったです。
- runtime の軽さ: AI がどれだけ綺麗に書いても、フレームワーク側のコード量・メモリ消費・cold start は変わりません。ここは Remix v3 や TanStack Start のほうが薄く感じます。

こうして並べると、「単一パッケージで完結する」「Web 標準に近い」「規約より明示」という Remix v3 の設計は、AI と一緒に書く運用とは相性が良い方向に振れていると感じています。

### Rails 的な「ひとつで何でも作れる」への期待

ここまでは厳しめの評価が続きましたが、もう一つ正直に書いておくと、Remix v3 の「ひとつのフレームワークでだいたい作れる」という方向性そのものは、自分は面白いと感じています。TypeScript のエコシステムでも Rails 的な全部入りを目指したフレームワークは過去にいくつかありましたが、UI・ルータ・認証・DB・スキーマ検証・テストまで同じパッケージから提供する構成で、ここまで真面目に揃えてきている例はあまり思い当たりません。

「最初に何を組み合わせるか」を考えずに済む、トラブルが出たときの参照先が一箇所に集中している、というのは、書きながら想像する未来としてはかなり気持ちが良いです。完成度よりも野心が先行している段階だとは思いますが、API が落ち着いて事例とドキュメントが揃ったときに、すごく使いやすくて作りやすいものに育っていてほしい、という期待を持って見ています。

production に乗せるのはまだ早めかなと感じていて、現時点で自分が薦められる現実的な使い方は「設計思想に触れるためのプロトタイプを 1 つ書いてみる」くらいです。

## これから試したいこと

タスクマネージャーは現状クライアントオンリーなので、次に伸ばしたい方向として 3 つ Issue を立てました。

- SSR 化 — `remix/ui/server` の `renderToStream` と `<Frame>` で、必要な部分だけ `clientEntry` でハイドレートする構成を試す
- ログイン機能 — `remix/auth` + `remix/session` で OAuth / クッキーセッション
- DB 永続化 — `remix/data-table` で SQLite や Postgres にタスクを保存

それぞれ Remix v3 の同じパッケージの別サブパスを足すだけで進められるはずなので、書ける段階で続報を出したいと思います。

雛形は `npx remix@next new my-remix-app` で作れます。

## 付録: alpha 時代を触っていた人向けの変更点

すでに `@remix-run/component` などの alpha 版を触っていた方向けに、目につく変化だけまとめておきます。

- パッケージが 1 つに統合されました。これまで `@remix-run/component` と `@remix-run/interaction` に分かれていたものを含めて、全部 `remix` の subpath (`remix/ui`, `remix/ui/animation`, `remix/data-schema` 等) から import します。
- ホスト props が `mix` に統一されました。`on={{ click: fn }}`, `connect={el => ...}`, `animate={...}`, `css={...}` のようなプロパティは廃止で、`mix={[on('click', fn), ref(el => ...), animateEntrance(...), css({...})]}` のように mixin 関数を配列で渡す形に揃いました。
- `interaction` のシンボル系イベントキー (`press`, `longPress`, `arrowUp` など) は廃止されました。ふつうの `on('click', ...)` や `on('keydown', e => { if (e.key === 'ArrowUp') ... })` で書きます。長押しのような複合ジェスチャは自前で `createMixin` する形になっています。
- `setup` prop も廃止されて、通常の prop を `handle.props` から読む形に統一されました。
- UI コンポーネントとテーマが同梱されています。Button, Menu, Popover, Glyph, Combobox, Listbox などと、それらが参照する theme トークン (`RMX_01`) が標準で入っています。
- CLI とテストランナー、サーバ周りも揃っています。`remix new`, `remix test`, `remix routes`, `remix doctor` といったコマンドや、`fetch-router` / 各種ミドルウェア / `auth` / `data-table` がパッケージ内に入っています。
