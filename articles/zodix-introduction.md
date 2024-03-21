---
title: "Remix × Zodix: パラメータやクエリのバリデーションをシンプルに書く"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["remix", "zod", "typescript"]
published: true
---

# これは何？

Remix でアプリケーションを開発する際、route でのパラメータやクエリのバリデーションは必須です。通常、このバリデーションは手動で行う必要がありますが、`zod` と `zodix` というモジュールを使うことでとてもシンプルに書くことができます。

たとえば、URL パラメータのバリデーションと値の取り出しでは以下のように定型句が多く煩雑になりがちですが、`zodix` はそこで威力を発揮します。

例:
`https://remix-app/query?count=100&page=1` のバリデーションを行う方法

zodix がない場合

```typescript
export async function loader({ request }: LoaderFunctionArgs) {
  const { searchParams } = new URL(request.url);
  const count = Number(searchParams.get("count"));
  const page = Number(searchParams.get("page"));
  if(isNaN(count) || isNaN(page)) {
    throw new Response("invalid query", { status: 400 })
  }
```

zodix を使う場合

```typescript
export async function loader({ request }: LoaderArgs) {
  const { count, page } = zx.parseQuery(request, {
    count: zx.NumAsString, // "100"
    page: zx.NumAsString, // "1"
  });
}
```

この記事では、`zodix` の基本的な使い方と、Remix アプリケーションでの活用方法について解説します。

# zodix とは

`zodix` は、Remix の loader や action でパラメータやクエリのバリデーションを簡単に行うための Zod ユーティリティのコレクションです。`FormData` や `URLSearchParams` のパースとバリデーションをシンプルにし、loader や action をクリーンで型安全に保ちます。

https://github.com/rileytomasek/zodix

## セットアップ

まず、`zodix` と `zod` をインストールします。

```bash
pnpm install zodix zod
```

次に、基本になる zx オブジェクトをインポートします。

```tsx
import { zx } from "zodix";
```

## 使い方

### route パラメータのバリデーション

`zx.parseParams` を使って、`LoaderFunctionArgs['params']` や `ActionFunctionArgs['params']` から `params` オブジェクトをパースしバリデーションできます。

```tsx
// route: users.$userId.notes.$noteId.tsx
// URL: https://remix-app/users/coji/notes/y8XparQV
export async function loader({ params }: LoaderArgs) {
  const { userId, noteId } = zx.parseParams(params, {
    userId: z.string(), // "coji"
    noteId: z.string(), // "y8XparQV"
  });
}
```

### クエリ文字列 URLSearchParams のバリデーション

`zx.parseQuery` を使って、`Request` のクエリ文字列（検索パラメータ）をパースしバリデーションできます。

```ts
// URL https://remix-app/users?count=100&page=1
export async function loader({ request }: LoaderArgs) {
  const { count, page } = zx.parseQuery(request, {
    count: zx.NumAsString, // "100"
    page: zx.NumAsString, // "1"
  });
}
```

`zx.NumAsString` は zodix が提供するヘルパー zod スキーマです。詳細は後述します。

### フォームデータのバリデーション

:::message
FormData を含む、フォームのバリデーションは [Conform](https://zenn.dev/topics/conform) というモジュールが非常に便利なので、そちらを使うのをオススメします。

`zodix` で可能な`loader`や`action`でのバリデーションだけでなく、クライアントサイドでのリアルタイムなバリデーションや、submission.reply を使ったエラー処理など非常に便利な機能をシンプルに書くことができます。
:::

`zx.parseForm` を使って、Remix の `action` で `FormData` をパースしバリデーションできます。

```ts
export async function action({ request }: ActionFunctionArgs) {
  const { email, password, saveSession } = await zx.parseForm(request, {
    email: z.string().email(),
    password: z.string().min(6),
    saveSession: zx.CheckboxAsString,
  });
}
```

### ヘルパー Zod スキーマ

`FormData` と `URLSearchParams` は全ての値を文字列にシリアライズするため、"5"、"on"、"true" のような値を扱うことがよくあります。`zodix` には、これらの文字列を適切な型にパースしバリデーションするためのヘルパースキーマが用意されています。

- `zx.BoolAsString`: "true" を `true` に、"false" を `false` にパースします。
- `zx.CheckboxAsString`: "on" を `true` に、`undefined` を `false` にパースします。
- `zx.IntAsString`: "3" を `3` にパースします。小数点以下は切り捨てられます。
- `zx.NumAsString`: "3" を `3` に、"3.14" を `3.14` にパースします。

これらのヘルパースキーマを使用すると、次のようにフォームデータやクエリ文字列をパースできます。

```typescript
const Schema = z.object({
  isAdmin: zx.BoolAsString,
  agreedToTerms: zx.CheckboxAsString,
  age: zx.IntAsString,
  cost: zx.NumAsString,
});

export async function action({ request }: ActionArgs) {
  const { isAdmin, agreedToTerms, age, cost } = await zx.parseForm(
    request,
    Schema
  );
  // isAdmin: boolean
  // agreedToTerms: boolean
  // age: number
  // cost: number
}

export async function loader({ request }: LoaderArgs) {
  const { isAdmin, agreedToTerms, age, cost } = zx.parseQuery(request, Schema);
  // isAdmin: boolean
  // agreedToTerms: boolean
  // age: number
  // cost: number
}
```

この例では、`zx.BoolAsString`、`zx.CheckboxAsString`、`zx.IntAsString`、`zx.NumAsString` を使用して、フォームデータとクエリ文字列の値を適切な型にパースしています。これにより、`action` 関数と `loader` 関数内で、変換された値を型安全に使用できます。

### エラーハンドリング

`parseParams`、`parseForm`、`parseQuery` は、パースに失敗した場合に 400 Response をスローします。これは Remix の error boundary とうまく連動し、カスタムエラーハンドリングを必要としない場合に使用するのに適しています。

```ts
export async function loader({ params }: LoaderArgs) {
  const { postId } = zx.parseParams(
    params,
    { postId: zx.NumAsString },
    { message: "Invalid postId parameter", status: 400 }
  );
  const post = await getPost(postId);
  return { post };
}

export function ErrorBoundary() {
  const error = useRouteError();
  return (
    <div>
      <h1>{String(error)}</h1>
    </div>
  );
}
```

# 実際の使用例 (SPA Mode)

zodix は 通常の SSR を行う Remix だけでなく、SPA mode でも使うことができます。
たとえば以下のように、Remix SPA のアプリで実際に使っています。

https://github.com/coji/remix-spa-example/blob/main/app/routes/%24handle%2B/posts.%24id.edit.tsx#L36-L52

上記コードを含むリポジトリはこちらです。参考になるとうれしいです。

https://github.com/coji/remix-spa-example

# まとめ

`zodix` を使うことで、Remix アプリケーションでのパラメータやクエリのバリデーションを簡単に実装できます。`zodix` は、定義済みの Zod スキーマや独自のスキーマを使用でき、エラーハンドリングも容易です。ぜひ `zodix` を活用して、Remix アプリケーションの開発を効率化しましょう。
