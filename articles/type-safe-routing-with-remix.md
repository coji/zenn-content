---
title: "Remix で型安全なルーティングを簡単に。"
emoji: "🐬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["remix", "typescript"]
published: false
---

# これはなに？

最近 Tanstack Router などで、型安全なルーティングが便利！という話を聞きます。いつも使ってる Remix にも、そういうコンセプトのライブラリがあって、型安全にできるらしい、というのは知ってたんですが、それって必要かなー？と思っていたのです。

で、ちょっと思い立ったので、習作として作って、実験につかってる Remix SPA での「しずかなインターネット」の真似アプリに入れてみたのでした。

結論、とても簡単に設定できて、下のようにエディタで補完も効くのでなかなか快適でした。

![](/images/type-safe-routing-with-remix//image.png)

あ、これはいい、と思ったので記事にしてみます。

# 型安全なルーティングを実現する remix-routes

今回いれるのはこのモジュールです。

https://github.com/yesmeck/remix-routes

設定はとても簡単で、 vite.config.ts に以下のようにプラグインを追加するだけです。

```diff ts:vite.config.ts
import { defineConfig } from 'vite'
import { vitePlugin as remix } from '@remix-run/dev'
+import { remixRoutes } from 'remix-routes/vite'

export default defineConfig({
  plugins: [
    remix(),
+   remixRoutes(),
  ],
})
```

これで vite を起動すると node_modules 下に以下のような型定義ファイルが生成されます。

```ts:remix-routes.d.ts
declare module "remix-routes" {
  // ...中略...
  export interface Routes {
    "/": {
      params: {},
      query: ExportedQuery<import('../app/root').SearchParams>,
    };
    "/:handle": {
      params: {
        handle: string | number;
      },
      query: ExportedQuery<import('../app/routes/$handle+/_index').SearchParams>,
    };
    "/:handle/posts/:id": {
      params: {
        handle: string | number;
        id: string | number;
      },
      query: ExportedQuery<import('../app/routes/$handle+/posts.$id._index').SearchParams>,
    };
    // ...以下省略...
  }
}
```

これを使って型安全にしてくれるわけですね。

## どんな感じになる？

remix-routes は $path という型安全にパス文字列を生成するヘルパー関数を用意してくれています。これを使い、Link や redirect などのパスを以下のように書きかえます。

### loader / action

オリジナル

```ts:_index.tsx
export const loader = async ({request}: LoaderFunctionArgs) => {
  authenticator.authenticate({ failureRedirect("/login") })
  return json({})
}

export const action = async ({request}: ActionFunctionArgs) => {
  return redirect("/")
}
```

書き換え後 ($path を使う)

```ts:_index.tsx
import { $path } from 'remix-routes'

export const loader = async ({request}: LoaderFunctionArgs) => {
  authenticator.authenticate({ failureRedirect($path("/login")) })
  return json({})
}

export const action = async ({request}: ActionFunctionArgs) => {
  return redirect($path("/"))
}
```

### コンポーネント

オリジナル

```ts:_index.tsx
export default function IndexPage = () => {
  const handle = 'coji',
  const postId = 'V1StGXR8_Z5jdHi6B-myT'

  return (
    <Link to={`/${handle}/posts/${postId}`}>こんにちは！</Link>
    <Link to="/help">ヘルプページ</Link>
  )
}
```

書き換え後 ($path を使う)

```ts:_index.tsx
import { $path } from 'remix-routes'

export default function IndexPage = () => {
  const handle = 'coji',
  const postId = 'V1StGXR8_Z5jdHi6B-myT'

  return (
    <Link to={$path("/:handle/posts/:id", { handle, id: postId })}>
      こんにちは！
    </Link>
    <Link to="/help">ヘルプページ</Link>
  )
}
```

ちょっと `$` 記号が「うっ...」と思いますが、ともかくこれで、エディタで補完されますし、存在しないルートやパラメータを指定するとエラーが出るようになります。まあ快適ですね！

# 実際の適用例

Remix SPA モードで作ったこのアプリに導入してみました。ソースコードも公開してるので、ご参考まで！

https://zenn.dev/articles/474d9b54d34962/edit

# まとめ

というわけで、Remix で型安全なルーティングを実現する remix-routes パッケージでした。
Remix はいいぞ！
