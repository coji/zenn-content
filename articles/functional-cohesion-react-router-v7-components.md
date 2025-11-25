---
title: "機能的凝集とコロケーションで実現する React Router v7 コンポーネント設計"
emoji: "🗂️"
type: "tech"
topics: ["react", "reactrouter", "remix", "typescript", "frontend"]
published: false
---

# 機能的凝集とコロケーションで実現する React Router v7 コンポーネント設計

## これはなに？

複数のユーザーロールや類似機能を含むフロントエンドで、コンポーネントをどう分割すべきか悩んだことはありませんか？本記事では「機能的凝集」という考え方と、remix-flat-routes のコロケーション構成を組み合わせることで、条件分岐が散らばりにくく保守しやすい設計を紹介します。

本記事では以下の技術スタックを前提としています。

- React Router v7（framework モード、loader / action 使用）
- remix-flat-routes（Nested folders with flat-files convention）
- route.tsx をエントリポイントとしたコロケーション構成

本記事のガイドラインを Claude Code などの AI アシスタント向けに CLAUDE.md へ記載する例を付録に用意しています。

## 凝集度とは

まず「凝集度」という言葉について簡単に説明します。凝集度とは、モジュール内の要素がどれだけ密接に関連しているかを表す指標です。高い凝集度を持つモジュールは、単一の目的に集中しているため、理解しやすく変更にも強くなります。

凝集度にはいくつかの段階がありますが、フロントエンドのコンポーネント設計で特に意識したいのは「論理的凝集」と「機能的凝集」の2つです。論理的凝集は避けるべきパターン、機能的凝集は目指すべきパターンです。

## 論理的凝集の問題：条件分岐が散らばる

開発初期には「購入者と出品者で商品詳細のUIはほとんど同じだから共通化しよう」といった判断がなされがちです。

```tsx
// 避けるべき例：論理的凝集
function ProductDetailPage({ role }: { role: "buyer" | "seller" | "admin" }) {
  return (
    <div>
      {role === "buyer" && <PurchaseButton />}
      {role === "seller" && <EditButton />}
      {role === "admin" && <DeleteButton />}
      {role !== "buyer" && <StockInfo />}
      {/* さらに条件分岐が増えていく... */}
    </div>
  )
}
```

時間が経つにつれて「購入者のモバイルではレビューを非表示」「出品者はドラフト時だけ公開ボタン」「管理者用にも同じコンポーネントを」といった要件が追加され、条件分岐がコード中に点在していきます。これが**論理的凝集**の問題です。

## 機能的凝集とは

機能的凝集とは、モジュール全体が単一の明確な目的のために構成されている状態です。購入者向けの商品詳細ページには購入者に必要な機能だけがあり、出品者向けには出品者に必要な機能だけがある、という設計です。

機能的凝集のメリットは2つあります。

1つ目は**変更に強い**こと。購入者向け機能の修正が出品者向け機能に影響しません。関心のあるコードだけを読めばよく、追加するコードも自然と機能的凝集しやすくなります。

2つ目は**要件との対応が明確**なこと。要件定義書は通常機能単位で書かれているため、コードとの対応づけが容易です。別の機能を開発しているメンバーやデザイナーとの意思疎通も取りやすくなります。

## コロケーション構成との相乗効果

React Router v7 + remix-flat-routes のコロケーション構成では、ルートごとにディレクトリを作成し、関連ファイルを同じ場所に配置します。

```txt
app/routes/
├── _buyer+/                           # 購入者向け
│   └── products+/
│       └── $productId+/
│           ├── route.tsx              # loader / コンポーネント
│           ├── components/
│           │   └── purchase-section.tsx
│           └── services/
│               └── queries.server.ts
│
├── _seller+/                          # 出品者向け
│   └── products+/
│       └── $productId+/
│           ├── route.tsx              # loader / action / コンポーネント
│           ├── components/
│           │   └── stock-management.tsx
│           └── services/
│               ├── queries.server.ts
│               └── mutations.server.ts
```

この構造と機能的凝集を組み合わせると、以下の効果が得られます。

**ディレクトリ = 機能境界**
ルートディレクトリがそのまま機能単位になります。「購入者向け商品詳細」と「出品者向け商品詳細」が別ディレクトリなので、どこに何があるか一目瞭然です。

**データフローが閉じる**
loader（データ取得）→ コンポーネント（表示）→ action（更新）が同じディレクトリ内で完結します。機能の全体像を把握するのにディレクトリ外を見る必要がありません。

**変更の影響範囲が明確**
購入者向け機能を修正するとき、`_buyer+/` 以下だけを見ればよいです。出品者向け機能に影響しないことがディレクトリ構造から保証されます。

**共通化の判断が容易**
共通コンポーネントを作りたくなったら、配置場所を考えます。同一ルート内なら `components/`、親子間なら親の `_shared/`、複数機能で共通なら `app/features/`。配置場所が決まらないなら、それは共通化すべきでないサインかもしれません。

## 実践パターン

### ロールごとに分ける

購入者・出品者・管理者でUIが異なる場合、レイアウトルート（`_buyer+/`、`_seller+/`、`_admin+/`）で分岐します。各ルート内にはそのロールに必要な機能だけを実装します。

```tsx
// app/routes/_buyer+/products+/$productId+/route.tsx
// 購入者に必要な機能だけ
export async function loader({ params }: Route.LoaderArgs) {
  return { product: await getProductForBuyer(params.productId) }
}

export default function BuyerProductPage({ loaderData }: Route.ComponentProps) {
  return (
    <div>
      <ProductInfo product={loaderData.product} />
      <PurchaseButton />
      <ReviewSection />
    </div>
  )
}
```

```tsx
// app/routes/_seller+/products+/$productId+/route.tsx
// 出品者に必要な機能だけ（action もある）
export async function loader({ params }: Route.LoaderArgs) {
  return { product: await getProductForSeller(params.productId) }
}

export async function action({ params, request }: Route.ActionArgs) {
  const formData = await request.formData()
  await updateStock(params.productId, formData)
  return { success: true }
}

export default function SellerProductPage({ loaderData }: Route.ComponentProps) {
  return (
    <div>
      <ProductInfo product={loaderData.product} />
      <StockManagement />
      <EditProductButton />
    </div>
  )
}
```

購入者向けには action がなく、出品者向けには在庫更新の action がある、という違いが明確です。

### 作成と編集を分ける

新規作成と編集でフォームが同じでも、ルートは分けます。共通のフォームコンポーネントは親ディレクトリの `_shared/` に配置し、各ルートから利用します。

```txt
app/routes/products+/
├── _shared/
│   └── components/
│       └── product-form.tsx   # 共通フォーム
├── new+/
│   └── route.tsx              # 作成用 action
└── $productId+/
    └── edit+/
        └── route.tsx          # 編集用 loader / action
```

```tsx
// app/routes/products+/new+/route.tsx
import { ProductForm } from "../_shared/components/product-form"

export async function action({ request }: Route.ActionArgs) {
  const formData = await request.formData()
  const product = await createProduct(formData)
  return redirect(`/products/${product.id}`)
}

export default function NewProductPage() {
  return <ProductForm submitLabel="登録する" />
}
```

```tsx
// app/routes/products+/$productId+/edit+/route.tsx
import { ProductForm } from "../../_shared/components/product-form"

export async function loader({ params }: Route.LoaderArgs) {
  return { product: await getProduct(params.productId) }
}

export async function action({ params, request }: Route.ActionArgs) {
  const formData = await request.formData()
  await updateProduct(params.productId, formData)
  return redirect(`/products/${params.productId}`)
}

export default function EditProductPage({ loaderData }: Route.ComponentProps) {
  return <ProductForm defaultValues={loaderData.product} submitLabel="更新する" />
}
```

新規作成には loader がなく、編集には既存データを取得する loader がある、という違いがルート単位で分離されています。

### Outlet でネストする

商品詳細の中にレビュー一覧や編集フォームがある場合、親ルートで商品情報を取得し、子ルートで各機能を実装します。

```txt
app/routes/products+/$productId+/
├── route.tsx           # 親：商品情報を取得、共通レイアウト
├── _index+/
│   └── route.tsx       # 子：商品詳細
├── reviews+/
│   └── route.tsx       # 子：レビュー一覧
└── edit+/
    └── route.tsx       # 子：編集フォーム
```

```tsx
// app/routes/products+/$productId+/route.tsx（親）
export async function loader({ params }: Route.LoaderArgs) {
  return { product: await getProduct(params.productId) }
}

export default function ProductLayout({ loaderData }: Route.ComponentProps) {
  return (
    <div>
      <h1>{loaderData.product.name}</h1>
      <nav>
        <Link to=".">詳細</Link>
        <Link to="reviews">レビュー</Link>
        <Link to="edit">編集</Link>
      </nav>
      <Outlet />
    </div>
  )
}
```

```tsx
// app/routes/products+/$productId+/reviews+/route.tsx（子）
export async function loader({ params }: Route.LoaderArgs) {
  return { reviews: await getReviews(params.productId) }
}

export default function ReviewsPage({ loaderData }: Route.ComponentProps) {
  // 親のデータが必要なら useRouteLoaderData で取得
  return <ReviewList reviews={loaderData.reviews} />
}
```

親で共通のデータ取得とレイアウトを行い、子は自分の機能に集中できます。

### ts-pattern で型による出し分け

ルートで分岐できない場合もあります。たとえば通知一覧のように、同じリスト内で種類によって表示を変える必要があるケースです。

このような場合、ts-pattern を使うと機能的凝集に近い形で実装できます。

```tsx
// app/routes/notifications+/route.tsx
import { match } from "ts-pattern"

type Notification =
  | { id: string; type: "order_completed"; orderId: string; productName: string }
  | { id: string; type: "review_posted"; productId: string; reviewerName: string }
  | { id: string; type: "stock_alert"; productId: string; currentStock: number }

export default function NotificationsPage({ loaderData }: Route.ComponentProps) {
  return (
    <ul>
      {loaderData.notifications.map((n) => (
        <NotificationItem key={n.id} notification={n} />
      ))}
    </ul>
  )
}

function NotificationItem({ notification }: { notification: Notification }) {
  return match(notification)
    .with({ type: "order_completed" }, (n) => (
      <li>
        <a href={`/orders/${n.orderId}`}>「{n.productName}」の注文が完了しました</a>
      </li>
    ))
    .with({ type: "review_posted" }, (n) => (
      <li>
        <a href={`/products/${n.productId}/reviews`}>{n.reviewerName}さんがレビューを投稿</a>
      </li>
    ))
    .with({ type: "stock_alert" }, (n) => (
      <li>
        <a href={`/products/${n.productId}/stock`}>在庫が残り{n.currentStock}個です</a>
      </li>
    ))
    .exhaustive()
}
```

`match().with().exhaustive()` のパターンには以下のメリットがあります。

- 各通知タイプの処理が `.with()` ブロック内に閉じている（機能的凝集）
- `exhaustive()` により、新しい通知タイプを追加したときにコンパイルエラーで漏れを検知できる
- `if-else` や `switch` より、各ケースの独立性が視覚的に明確

ルートで分岐できない場合でも、ts-pattern を使えば条件分岐の散らばりを防げます。

## 共通化の判断基準

「似ている」と感じても、すぐに共通化しないことが重要です。

共通化してよいのは、**APIスキーマやDBスキーマで同じ型**を参照している場合です。購入者と出品者で同じ `User` 型を使っているなら、`UserInfo` コンポーネントや `getUserById` のようなクエリは共通化できます。

共通化を避けるべきなのは、**意味的に異なる**場合です。購入者視点の「購入履歴画面」と管理者視点の「注文管理画面」は、見た目が似ていても責務が異なります。別々に実装した方が、将来の変更に耐えやすくなります。

### 配置場所の目安

共通化する場合、どこに置くかも重要です。

- **同一ルート内で共通** → そのルートの `components/` に配置
- **親子ルート間で共通** → 親ルートの `_shared/` に配置
- **3つ以上のルートで共通** → `app/features/` に切り出す

```txt
app/
├── routes/
│   ├── _buyer+/...
│   ├── _seller+/...
│   ├── _admin+/...
│   └── products+/
│       ├── _shared/                # 親子ルート間で共通
│       │   ├── components/
│       │   │   └── product-card.tsx
│       │   └── services/
│       │       └── queries.server.ts
│       ├── $productId+/
│       │   └── route.tsx
│       └── new+/
│           └── route.tsx
└── features/                       # 3つ以上のルートで共通
    └── user/
        ├── components/
        │   └── user-avatar.tsx
        └── services/
            └── queries.server.ts
```

`_shared/` ディレクトリは `routes.ts` で除外設定しておけば、ルートとして認識されません。

```ts
// routes.ts
import { flatRoutes } from "remix-flat-routes"

export default flatRoutes("routes", defineRoutes, {
  ignoredRouteFiles: ["**/_shared/**", "**/index.ts", "**/index.tsx"],
})
```

`index.ts` / `index.tsx` も除外しておくと、`_shared/components/index.ts` で re-export をまとめられるので import が楽になります。

```ts
// _shared/components/index.ts
export { ProductCard } from "./product-card"
export { ProductForm } from "./product-form"
```

```tsx
// 利用側
import { ProductCard, ProductForm } from "../_shared/components"
```

`features/` には、複数のルートから参照される機能単位のコードをまとめます。ただし、最初から `features/` に置くのではなく、3つ以上のルートで使われるようになってから切り出すくらいがちょうどよいです。

なぜ「3つ以上」なのか。2つのルートで共通化すると、片方の要件変更でもう片方に影響が出やすくなります。3つ以上で使われているなら、それは本当に共通の概念である可能性が高く、切り出す価値があります。早すぎる共通化は、結局あとで分離し直すことになりがちです。

## まとめ

機能的凝集の考え方とコロケーション構成を組み合わせると、以下が実現できます。

- 条件分岐がコード中に散らばらない
- ディレクトリ構造を見るだけで機能の境界がわかる
- 変更の影響範囲が明確
- オンボーディングやバグ対応が速くなる

共通化できる塊は思っているより少ないです。「似ている」からといってすぐに共通化せず、機能単位でディレクトリを分けることを優先してみてください。

## 参考

- [React Router v7](https://reactrouter.com/) - 本記事で前提としている framework モードのドキュメント
- [remix-flat-routes](https://github.com/kiliman/remix-flat-routes) - Nested folders with flat-files convention の詳細
- [ts-pattern](https://github.com/gvergnaud/ts-pattern) - 型安全なパターンマッチングライブラリ
- TSKaigi 2025 Noritaka Ikeda氏（ROUTE06）の発表「機能的凝集の概念を用いて、複数ロール・類似機能を多く含むシステムのフロントエンドのコンポーネントを適切に分割する」

また、kennさんが開発している [react-router-auto-routes](https://github.com/kenn/react-router-auto-routes) も便利です。`+` プレフィックスによるコロケーションや、フォルダベースとドット区切りを混在できる柔軟なファイル構成など、React Router v7 向けにモダンな設計がされています。

## 付録: CLAUDE.md 記載例

Claude Code などの AI アシスタント向けに、本記事のガイドラインを CLAUDE.md に記載する例です。

```markdown
## React Router v7 + remix-flat-routes Guidelines

### Route Structure
- Use colocation: `route.tsx` + `components/` + `services/` in each route directory
- Split by role: `_buyer+/`, `_seller+/`, `_admin+/` as layout routes
- Separate create/edit: `new+/route.tsx` and `$id+/edit+/route.tsx`
- Use `services/queries.server.ts` for loaders, `mutations.server.ts` for actions

### Functional Cohesion
- Avoid logical cohesion (role-based conditionals scattered in one component)
- Each route should contain only features needed for that specific function
- Buyer route: no action needed. Seller route: has stock update action

### Shared Code Placement
- Same route: `components/`
- Parent-child routes: `_shared/` in parent (excluded in routes.ts)
- 3+ routes: `app/features/`
- Don't share until used by 3+ routes

### routes.ts Config
ignoredRouteFiles: ["**/_shared/**", "**/index.ts", "**/index.tsx"]
```
