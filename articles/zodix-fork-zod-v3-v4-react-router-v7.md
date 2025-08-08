---
title: "React Router v7 のバリデーションライブラリ Zodix を Zod v3/v4 両対応にしてみた"
emoji: "🚀"
type: "tech"
topics: ["reactrouter", "zod", "typescript", "opensource", "remix"]
published: true
---

## Zodix を fork して改良してみました

React Router v7（旧 Remix）でフォームやクエリパラメータのバリデーションを行うライブラリ「[Zodix](https://github.com/rileytomasek/zodix)」を fork して、Zod v3/v4 両対応と React Router v7 対応にしてみました。NPM パッケージとして `@coji/zodix` で公開しています。

https://github.com/coji/zodix

https://www.npmjs.com/package/@coji/zodix

## なぜ fork したのか

オリジナルの Zodix は便利なライブラリなんですが、使っていて困ったことがいくつかありました。

### 1. Zod v3 から v4 への移行問題

プロジェクトによって Zod のバージョンが違うんですよね。大規模プロジェクトだと、依存関係の都合で簡単にアップグレードできないこともあります。Zod v3 と v4 は互換性がないので、両方使えるようにしたかったんです。

### 2. React Router v7 への対応

Remix が React Router v7 に統合されたことに伴い、`@remix-run` パッケージへの依存を削除しました。これにより Remix でも React Router v7 でも、どちらでも使えるようになりました。

### 3. メンテナンス状況

オリジナルのリポジトリは更新が止まっていて、Issue や PR への対応も滞っている状態でした。実際に使っているプロジェクトがあるので、自分でメンテナンスできるようにしたかったんです。

## 実装の工夫

### Zod v3/v4 両対応の仕組み

Zod v3 と v4 の非互換性が一番の課題でした。両バージョンで型定義が違うので、単純に条件分岐では対応できないんですよね。

結局、**別々のインポートパス**を提供することにしました。

```typescript
// Zod v3 を使う場合（デフォルト）
import { zx } from '@coji/zodix'

// Zod v4 を使う場合
import { zx } from '@coji/zodix/v4'
```

`package.json` の exports フィールドで、それぞれのパスに対応するファイルを指定：

```json
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "require": "./dist/index.cjs"
    },
    "./v4": {
      "types": "./dist/v4.d.ts",
      "import": "./dist/v4.js",
      "require": "./dist/v4.cjs"
    }
  }
}
```

内部では、v3 用と v4 用で別々の実装ファイルを用意：

```
src/
├── parsers.v3.ts    # Zod v3 用のパーサー実装
├── parsers.v4.ts    # Zod v4 用のパーサー実装
├── schemas.v3.ts    # Zod v3 用のヘルパースキーマ
├── schemas.v4.ts    # Zod v4 用のヘルパースキーマ
├── v3.ts           # v3 のエクスポート（デフォルト）
└── v4.ts           # v4 のエクスポート
```

これだと以下のメリットがあります：
- **完全な型安全性** - それぞれのバージョンで正しい型が適用される
- **ビルド時の最適化** - 使わないバージョンのコードは含まれない
- **移行が簡単** - インポートパスを変えるだけで v3 から v4 に移行可能

### React Router v7 / Remix 両対応

`@remix-run` パッケージへの依存を削除し、どちらでも使えるようにしました：

```typescript
// Remix でも
import type { LoaderFunctionArgs } from '@remix-run/node'

export async function loader({ params }: LoaderFunctionArgs) {
  const { postId } = zx.parseParams(params, {
    postId: zx.NumAsString
  })
  // ...
}

// React Router v7 でも
import type { Route } from './+types.posts.$postId'

export async function loader({ params }: Route.LoaderArgs) {
  const { postId } = zx.parseParams(params, {
    postId: zx.NumAsString
  })
  // ...
}
```

## 使い方

### インストール

```bash
npm install @coji/zodix zod
```

### 基本的な使い方

React Router v7 のルートファイルで：

```typescript
import { z } from 'zod'
import { zx } from '@coji/zodix' // Zod v3
// import { zx } from '@coji/zodix/v4' // Zod v4

// パラメータのバリデーション
export async function loader({ params }: Route.LoaderArgs) {
  const { userId, postId } = zx.parseParams(params, {
    userId: z.string(),
    postId: zx.NumAsString // "123" → 123
  })
  
  // userIdとpostIdは型安全に使える
  const post = await getPost(userId, postId)
  return { post }
}

// フォームデータのバリデーション
export async function action({ request }: Route.ActionArgs) {
  const data = await zx.parseForm(request, {
    title: z.string().min(1),
    content: z.string().min(10),
    published: zx.CheckboxAsString // "on" → true
  })
  
  // dataは完全に型付けされている
  await createPost(data)
  return { success: true }
}
```

### ヘルパースキーマ

HTML フォームや URL パラメータは全て文字列なので、数値やブール値への変換が必要ですよね。Zodix には便利なヘルパースキーマがあります：

```typescript
// 数値として扱う
zx.NumAsString    // "3.14" → 3.14
zx.IntAsString    // "42" → 42 (整数のみ)

// ブール値として扱う
zx.BoolAsString      // "true" → true, "false" → false
zx.CheckboxAsString  // "on" → true, undefined → false
```

### エラーハンドリング

バリデーションエラーは自動的に 400 エラーとして処理されます：

```typescript
export async function loader({ params }: Route.LoaderArgs) {
  // postIdが数値でない場合、400エラーをスロー
  const { postId } = zx.parseParams(
    params,
    { postId: zx.NumAsString },
    { message: "Invalid post ID" }
  )
}

// カスタムエラーハンドリングが必要な場合
export async function action({ request }: Route.ActionArgs) {
  const result = await zx.parseFormSafe(request, {
    email: z.string().email()
  })
  
  if (!result.success) {
    return { error: result.error.flatten() }
  }
  
  // result.data を使って処理
}
```

## Zod v3 から v4 への移行

プロジェクトを Zod v4 にアップグレードする際の手順：

1. Zod をアップグレード
```bash
npm install zod@^4.0.0
```

2. Zodix のインポートパスを変更
```typescript
// Before
import { zx } from '@coji/zodix'

// After
import { zx } from '@coji/zodix/v4'
```

3. 必要に応じて Zod スキーマを更新（[Zod v4 の変更点](https://zod.dev/v4/changelog)を参照）

これだけで移行完了です！

## 実際のプロジェクトでの活用例

React Router v7 の SPA モードでも問題なく動作します。実際のプロジェクトで使用している例：

```typescript
// routes/posts.$postId.edit.tsx
import { zx } from '@coji/zodix/v4'
import type { Route } from './+types.posts.$postId.edit'

export async function action({ params, request }: Route.ActionArgs) {
  const { postId } = zx.parseParams(params, {
    postId: zx.NumAsString
  })
  
  const formData = await zx.parseForm(request, {
    title: z.string().min(1, "タイトルは必須です"),
    content: z.string().min(10, "本文は10文字以上必要です"),
    tags: z.array(z.string()).optional()
  })
  
  await updatePost(postId, formData)
  return redirect(`/posts/${postId}`)
}
```

## まとめ

Zodix を fork して、Zod v3/v4 の両対応と React Router v7 対応を実現できました。別々のインポートパスで型安全性を維持しつつ、簡単に移行できるようになっています。

React Router v7 でフォームバリデーションを使いたい方は、`@coji/zodix` を試してみてください。フィードバックもお待ちしています！

## 参考記事

Zodix の基本的な使い方については、以前書いたこちらの記事も参考にしてください：

https://zenn.dev/coji/articles/zodix-introduction

## リンク

- [GitHub リポジトリ](https://github.com/coji/zodix)
- [NPM パッケージ](https://www.npmjs.com/package/@coji/zodix)
- [オリジナルの Zodix](https://github.com/rileytomasek/zodix)（Thanks to Riley Tomasek! 🙏）
