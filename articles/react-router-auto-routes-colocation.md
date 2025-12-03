---
title: "remix-flat-routes から react-router-auto-routes へ移行する"
emoji: "📁"
type: "tech"
topics: ["react", "reactrouter", "設計", "typescript"]
published: false
---

## これはなに？

[機能的凝集とコロケーションで保守しやすい React Router v7 コンポーネント設計](https://zenn.dev/coji/articles/react-router-v7-functional-cohesion-colocation)の続きです。

これまで remix-flat-routes を使っていたのですが、[react-router-auto-routes](https://github.com/kenn/react-router-auto-routes) のほうがスッキリしていて良さそうなので、乗り換えにあたって構成をまとめました。

## remix-flat-routes との違い

主な違いは以下のとおりです。

| 項目 | remix-flat-routes | react-router-auto-routes |
|------|-------------------|-------------------------|
| ルートエントリ | `route.tsx` | `index.tsx` |
| レイアウト | 親フォルダの `route.tsx` または同フォルダの `_layout.tsx` | `_layout.tsx` |
| pathless layout | `_auth+/` (サフィックス) | `_auth/` (プレフィックスのみ) |
| コロケーション | `$id+/` (サフィックス) | `+/`, `+components/` (プレフィックス) |
| 除外設定 | `routes.ts` で `ignoredRouteFiles` 指定 | 不要（`+` プレフィックスで自動除外） |
| 共有フォルダ | `_shared/` + 除外設定 | `+_shared/`（自動除外） |

移行は CLI ツールでできます。

```bash
npx migrate-auto-routes
```

Git の未コミット変更がないことを確認した上で実行すると、ファイル名やフォルダ構造を自動で変換してくれます。変換前後で `npx react-router routes` の出力を比較し、差異があれば元に戻してくれるので安心です。

## react-router-auto-routes の特徴

- `index.tsx` がルートのエントリポイント
- `_layout.tsx` がレイアウト
- `+` **プレフィックス**でコロケーション（`+/`、`+components/`）
- 設定不要で `+` 付きファイルは自動的にルートから除外

```ts
// routes.ts
import { autoRoutes } from "react-router-auto-routes"

export default autoRoutes()
```

## ディレクトリ構成

### 基本構成

```txt
app/routes/
├── products/
│   ├── index.tsx              # /products
│   ├── $productId/
│   │   ├── index.tsx          # /products/:productId
│   │   └── +/
│   │       ├── queries.ts     # データ取得
│   │       ├── mutations.ts   # データ更新
│   │       └── components/
│   │           └── product-info.tsx
│   └── new/
│       └── index.tsx          # /products/new
```

`+/` フォルダ内のファイルはルートとして認識されません。ヘルパー、コンポーネント、型定義などを自由に配置できます。

### ロール別の分離

```txt
app/routes/
├── _buyer/                    # pathless layout（URLに含まれない）
│   ├── _layout.tsx            # 購入者用レイアウト
│   └── products/
│       └── $productId/
│           ├── index.tsx      # /products/:productId（購入者向け）
│           └── +/
│               └── queries.ts
│
├── _seller/
│   ├── _layout.tsx            # 出品者用レイアウト
│   └── products/
│       └── $productId/
│           ├── index.tsx      # /products/:productId（出品者向け）
│           └── +/
│               ├── queries.ts
│               └── mutations.ts
```

`_` プレフィックスのフォルダは pathless layout になります。URL には含まれませんが、レイアウトやデータ取得を共有できます。

### 親子間での共有

```txt
app/routes/products/
├── +_shared/                  # 親子ルート間で共有
│   └── components/
│       └── product-form.tsx
├── new/
│   └── index.tsx
└── $productId/
    └── edit/
        └── index.tsx
```

```tsx
// app/routes/products/new/index.tsx
import { ProductForm } from "../+_shared/components/product-form"
```

`+_shared/` のように `+` プレフィックスを付ければ、どこに置いてもルートから除外されます。

### ネストしたレイアウト

```txt
app/routes/products/$productId/
├── _layout.tsx                # 共通レイアウト + loader
├── index.tsx                  # /products/:productId
├── reviews/
│   └── index.tsx              # /products/:productId/reviews
└── edit/
    └── index.tsx              # /products/:productId/edit
```

`_layout.tsx` で商品情報を取得し、子ルートで `<Outlet />` を通じて表示します。

## コード例

### ルートファイル

```tsx
// app/routes/products/$productId/index.tsx
import type { Route } from "./+types/index"
import { getProduct } from "./+/queries"
import { ProductInfo } from "./+/components/product-info"

export async function loader({ params }: Route.LoaderArgs) {
  return { product: await getProduct(params.productId) }
}

export default function ProductPage({ loaderData }: Route.ComponentProps) {
  return <ProductInfo product={loaderData.product} />
}
```

### コロケーションファイル

```ts
// app/routes/products/$productId/+/queries.ts
export async function getProduct(id: string) {
  // ...
}
```

```tsx
// app/routes/products/$productId/+/components/product-info.tsx
export function ProductInfo({ product }: { product: Product }) {
  // ...
}
```

## 配置場所の判断基準

| 共有範囲 | 配置場所 |
|---------|---------|
| 同一ルート内 | `+/components/` |
| 親子ルート間 | 親の `+_shared/` |
| 3つ以上のルート | `app/features/` |

3つ以上のルートで使われるまでは `features/` への切り出しを避けます。早すぎる共通化は、結局あとで分離し直すことになりがちです。

## まとめ

react-router-auto-routes を使うと、以下のパターンで機能的凝集とコロケーションを実現できます。

- **ルートエントリ**: `index.tsx`
- **レイアウト**: `_layout.tsx`
- **コロケーション**: `+/` フォルダ（自動除外）
- **pathless layout**: `_` プレフィックスのフォルダ
- **親子共有**: `+_shared/` フォルダ

詳細な設計思想については[元記事](https://zenn.dev/coji/articles/react-router-v7-functional-cohesion-colocation)を参照してください。

## 参考

- [react-router-auto-routes](https://github.com/kenn/react-router-auto-routes)
- [機能的凝集とコロケーションで保守しやすい React Router v7 コンポーネント設計](https://zenn.dev/coji/articles/react-router-v7-functional-cohesion-colocation)

本記事では基本的な構成パターンに絞りました。オプショナルセグメント `(segment)`、スプラットルート `$.tsx`、リテラルドットのエスケープ `[.]`、モノレポ対応など、詳細な命名規則は README を参照してください。
