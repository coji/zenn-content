---
title: "React Router v7 SPA Mode でクライアントミドルウェアを使ってみた"
emoji: "🧩"
type: "tech"
topics: ["reactrouter", "react", "typescript"]
published: true
---

React Router v7.3.0 で導入された「クライアントミドルウェア」機能を使って認証ガードをリファクタリングしてみました。この記事では、Firebase 認証を使った SPA アプリ「[しずかな Remix SPA Example](https://github.com/coji/remix-spa-example)」での実際の使用例をもとに解説します。

## TL;DR

- React Router v7.3.0 から `unstable_clientMiddleware` がサポートされました（[リリースノート](https://github.com/remix-run/react-router/blob/main/CHANGELOG.md#middleware-unstable)）
- ミドルウェアを使うと、各ルートに散らばっていた認証チェックを一箇所にまとめられます
- コンテキストを使ってユーザー情報を共有できるので、ルートローダーでの重複した認証処理が不要になります
- まだ unstable 機能ですが、コードがすっきりして可読性が上がるのでおすすめです

## 従来の認証実装の問題点

Firebase 認証を使った SPA アプリでは、通常、保護されたルートごとに認証チェックを行う必要があります。例えば：

```tsx
export const clientLoader = async ({ request }: Route.ClientLoaderArgs) => {
  const user = await requireAuth(request, { failureRedirect: href('/') })
  if (user.handle) {
    return redirect(href('/:handle', { handle: user.handle }))
  }
  return null
}
```

これを **すべての保護されたルート** に書く必要があり、認証ロジックが散らばって管理が難しくなります。また、ユーザー情報を取得するだけの場合も同様のコードが必要でした。

## ミドルウェアを使った改善

React Router v7.3.0 から導入された `unstable_clientMiddleware` を使うと、これらの問題を解決できます。

### 1. セットアップ

まず `react-router.config.ts` でミドルウェア機能を有効にします：

```typescript
// react-router.config.ts
import type { Config } from '@react-router/dev/config'

export default {
  ssr: false,
  future: {
    unstable_middleware: true,
  },
} satisfies Config
```

:::message
unstable_middleware を有効にすると loader / action の引数である context の形が AppLoaderContext から unstable_RouterContextProvider に変更されます。

React Router v7.6.0 以降、このようにfuture フラッグを有効して型が変わる場合は、declare を使ったアンビエント宣言は不要となり、自動的に反映されるようになりました。
:::

### 2. コンテキストの作成

認証ユーザー情報を共有するためのコンテキストを作成します：

```typescript
// middlewares/user-context.ts
import { unstable_createContext } from 'react-router'
import type { AppUser } from '~/services/auth'

export const userContext = unstable_createContext<AppUser | null>()
```

### 3. 認証ミドルウェアの実装

2種類のミドルウェアを作成しました：

#### 必須認証ミドルウェア（オンボーディング用）

```typescript
// middlewares/on-boarding-auth-middleware.ts
import {
  href,
  redirect,
  type unstable_MiddlewareFunction as MiddlewareFunction,
} from 'react-router'
import { requireAuth } from '~/services/auth'
import { userContext } from './user-context'

export const onBoardingAuthMiddleware: MiddlewareFunction = async ({
  request,
  context,
}) => {
  // オンボーディング手続きはログインしていないとアクセスできない
  const user = await requireAuth(request, { failureRedirect: href('/') })
  context.set(userContext, user)

  // すでにオンボーディング済みの場合はプロフィールページにリダイレクト
  if (user.handle) {
    throw redirect(href('/:handle', { handle: user.handle }))
  }
}
```

#### オプショナル認証ミドルウェア（一般ページ用）

```typescript
// middlewares/optional-auth-middleware.ts
import type { unstable_MiddlewareFunction as MiddlewareFunction } from 'react-router'
import { isAuthenticated } from '~/services/auth'
import { userContext } from './user-context'

export const optionalAuthMiddleware: MiddlewareFunction = async ({
  request,
  context,
}) => {
  // ログインしている場合はユーザー情報をセット
  const user = await isAuthenticated(request)
  context.set(userContext, user)
}
```

### 4. ミドルウェアの適用

レイアウトルートにミドルウェアを設定することで、そのレイアウト配下のすべてのルートに適用されます：

```typescript
// routes/welcome+/_layout/route.ts
import { onBoardingAuthMiddleware } from '~/middlewares/on-boarding-auth-middleware'

// Middleware を設定
export const unstable_clientMiddleware = [onBoardingAuthMiddleware]
```

```typescript
// routes/$handle+/_layout/route.tsx
import { Outlet } from 'react-router'
import { AppFooter } from '~/components/AppFooter'
import { optionalAuthMiddleware } from '~/middlewares/optional-auth-middleware'

export const unstable_clientMiddleware = [optionalAuthMiddleware]

export default function UserPageLayout() {
  return (
    <>
      <Outlet />
      <AppFooter />
    </>
  )
}
```

### 5. ルートローダーでの使用

ミドルウェアを設定すると、ルートローダーでは `context` からユーザー情報を取得できるようになります：

```typescript
export const clientLoader = async ({
  params: { handle },
  context,
}: Route.ClientLoaderArgs) => {
  const isExist = await isAccountExistsByHandle(handle)
  if (!isExist) throw data(null, { status: 404 })

  // ミドルウェアからセットされたユーザー情報を取得
  const user = context.get(userContext)
  const posts = await listUserPosts(handle)

  return { handle, user, posts, isAuthor: handle === user?.handle }
}
```

## リファクタリング前後の比較

### Before：各ルートで認証チェック

```typescript
// 認証必須ルート
export const clientLoader = async ({ request }: Route.ClientLoaderArgs) => {
  const user = await requireAuth(request, { failureRedirect: href('/') })
  if (user.handle) {
    return redirect(href('/:handle', { handle: user.handle }))
  }
  return null
}

// ユーザー情報だけ必要なルート
export const clientLoader = async ({ request, params }: Route.ClientLoaderArgs) => {
  // ...
  const user = await isAuthenticated(request)
  // ...
  return { /* ... */ }
}
```

### After：ミドルウェアとコンテキスト

```typescript
// レイアウトルートにミドルウェアを設定
export const unstable_clientMiddleware = [onBoardingAuthMiddleware]

// ルートローダーでは context からユーザー情報を取得
export const clientLoader = async ({ context }: Route.ClientLoaderArgs) => {
  const user = context.get(userContext)
  // ...
  return { /* ... */ }
}
```

## メリット

1. **認証ロジックの集約**: 各ルートに散らばっていた認証ロジックを一箇所にまとめられる
2. **重複コードの削減**: 同じ認証チェックを何度も書く必要がなくなる
3. **ユーザー情報の共有**: `context` を通じてユーザー情報を簡単に共有できる
4. **関心の分離**: 認証とルートの本来の機能を分離できる

## 注意点

- `unstable_` プレフィックスがついているように、まだ安定版ではありません
- プロジェクトの構造によっては、複数のレイアウトルートにミドルウェアを設定する必要があるかもしれません
- 将来的に API が変更される可能性があるので、アップデート時には注意が必要です

## まとめ

React Router v7.3.0 のクライアントミドルウェア機能を使うことで、認証ロジックを一箇所にまとめ、コードの可読性と保守性を大幅に向上させることができました。まだ unstable 機能ですが、大きなメリットがあるので、小〜中規模の SPA プロジェクトでは積極的に活用していきたい機能です。

今回紹介したコードは「[しずかな Remix SPA Example](https://github.com/coji/remix-spa-example)」というFirebase認証を使ったReact Router v7のサンプルアプリで実装しています。実際の使い方を確認したい方はぜひ参照してみてください。

Firebase 認証を使った SPA アプリの開発において、React Router v7 のミドルウェア機能が皆さんのコード品質向上に役立てば幸いです。
