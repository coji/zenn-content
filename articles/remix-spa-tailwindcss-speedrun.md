---
title: "Remix SPA Mode に TailwindCSS を入れて Cloudflare Pages にデプロイするスピードラン"
emoji: "👉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["remix", "tailwindcss", "cloudflarepages"]
published: false
---

# これは何？

たまに、Remix SPA Mode で、TailwindCSS がうまく入らない、というつぶやきを拝見することがあります。

確かに Remix ドキュメントが現状 Vite モードで、TailwindCSS の記述を見てそのまま設定すると動かない、ということがおきているようです。Remix の Vite のほうには書いてあるんですがね。。

というわけで、Remix SPA モードに モードで、TailwindCSS を入れて、ついでに Vercel でデプロイするのを最小手でやってみましょう。

# 手順

それではやっていきましょう。

## 1. SPA モードテンプレートを使って Remix アプリを初期設定

Remix SPA mode のテンプレートを使って初期設定します。
ディレクトリ名は適宜ご自身で命名してください。他の質問にはすべて Yes で OK です。

```sh
$ pnpm create remix@latest --template remix-run/remix/templates/spa
.../Library/pnpm/store/v3/tmp/dlx-57864  | Progress: resolved 1, reused 0, downloaded 0,.../Library/pnpm/store/v3/tmp/dlx-57864  | Progress: resolved 12, reused 12, downloaded .../Library/pnpm/store/v3/tmp/dlx-57864  | Progress: resolved 13, reused 12, downloaded .../Library/pnpm/store/v3/tmp/dlx-57864  | Progress: resolved 14, reused 14, downloaded .../Library/pnpm/store/v3/tmp/dlx-57864  | Progress: resolved 15, reused 14, downloaded .../Library/pnpm/store/v3/tmp/dlx-57864  | Progress: resolved 53, reused 52, downloaded .../Library/pnpm/store/v3/tmp/dlx-57864  | Progress: resolved 100, reused 99, downloaded.../Library/pnpm/store/v3/tmp/dlx-57864  | Progress: resolved 117, reused 117, downloade.../Library/pnpm/store/v3/tmp/dlx-57864  | Progress: resolved 138, reused 138, downloade.../Library/pnpm/store/v3/tmp/dlx-57864  | Progress: resolved 159, reused 158, downloade.../Library/pnpm/store/v3/tmp/dlx-57864  | Progress: resolved 170, reused 170, downloade.../Library/pnpm/store/v3/tmp/dlx-57864  | Progress: resolved 176, reused 176, downloade.../Library/pnpm/store/v3/tmp/dlx-57864  | Progress: resolved 188, reused 188, downloade.../Library/pnpm/store/v3/tmp/dlx-57864  | +200 ++++++++++++++++++++
.../Library/pnpm/store/v3/tmp/dlx-57864  | Progress: resolved 188, reused 188, downloade.../Library/pnpm/store/v3/tmp/dlx-57864  | Progress: resolved 200, reused 200, downloade.../Library/pnpm/store/v3/tmp/dlx-57864  | Progress: resolved 200, reused 200, downloaded 0, added 200, done

 remix   v2.8.1 💿 Let's build a better website...

   dir   Where should we create your new project?
         remix-spa-tailwindcss-speedrun

      ◼  Template: Using remix-run/remix/templates/spa...
      ✔  Template copied

   git   Initialize a new git repository?
         Yes

  deps   Install dependencies with pnpm?
         Yes

      ✔  Dependencies installed

      ✔  Git initialized

  done   That's it!

         Enter your project directory using cd ./remix-spa-tailwindcss-speedrun
         Check out README.md for development and deploy instructions.

         Join the community at https://rmx.as/discord
```

以下、ここで作成したプロジェクトのディレクトリ `remix-spa-tailwindcss-speedrun` 内で作業を行います。

## 2. TailwindCSS / PostCSS をインストール・設定

### 2-1. TailwindCSS / PostCSS npm modules をインストール

npm modules として tailwdincss と postcss を入れます。

```sh
$ pnpm add -D tailwindcss postcss
```

### 2-2. TailwindCSS の設定ファイルを作成・編集

tailwindcss の設定ファイル雛形を tailwindcss コマンドを使って作りましょう。

```sh
$ pnpm tailwindcss init --ts

Created Tailwind CSS config file: tailwind.config.ts
```

これでプロジェクトルートディレクトリに `tailwind.config.ts` が生成されます。これを以下のように編集します。1 行だけの変更です。

```diff ts:tailwind.config.ts
import type { Config } from "tailwindcss";

export default {
-  content: [],
+  content: ["./app/**/*.{js,jsx,ts,tsx}"],
  theme: {
    extend: {},
  },
  plugins: [],
} satisfies Config;
```

### 2-3. PostCSS の設定ファイルを追加

tailwindcss を動かすには postcss を入れて設定する必要があります。以下の内容で `postcss.config.js` ファイルを、プロジェクトルートディレクトリに作成します。

```js:postcss.config.js
export default {
  plugins: {
    tailwindcss: {},
  },
};
```

### 2-4. app/routes/tailwind.css ファイルを作成

実際にブラウザが読み込む CSS を作らねばなりません。
以下の内容で CSS ファイルを作成します。

```css:app/routes/tailwind.css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### 2-5. app/routes/root.tsx を編集し tailwind.css をインポート

`app/routes/root.tsx` ファイルを編集し、先程作成した `tailwind.css` ファイルを `import` する文を 1 行追加します。

`import ~ from` ではなく `import` だけなのがポイントです。これで vite がよろしくやってくれます。

```diff tsx:app/routes/root.tsx
import {
  Links,
  Meta,
  Outlet,
  Scripts,
  ScrollRestoration,
} from "@remix-run/react";
+import "./tailwind.css";

export function Layout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <head>
        <meta charSet="utf-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1" />
        <Meta />
        <Links />
      </head>
      <body>
        {children}
        <ScrollRestoration />
        <Scripts />
      </body>
    </html>
  );
}

export default function App() {
  return <Outlet />;
}

export function HydrateFallback() {
  return <p>Loading...</p>;
}
```

以上で TailwindCSS の CSS クラスを使うことができるようになりました。

## 3. index ページに TailwindCSS のクラスを適用して確認

index ページの JSX を編集して TailwindCSS のクラスを使うように変更します。
今回は 3 行だけです。区別のために見出しを青くしてみましょう。

```diff tsx:app/routes/_index.tsx
import type { MetaFunction } from "@remix-run/node";

export const meta: MetaFunction = () => {
  return [
    { title: "New Remix SPA" },
    { name: "description", content: "Welcome to Remix (SPA Mode)!" },
  ];
};

export default function Index() {
  return (
-    <div style={{ fontFamily: "system-ui, sans-serif", lineHeight: "1.8" }}>
+    <div className="font-[system-ui,sans-serif] leading-8 mx-4 my-2">
-      <h1>Welcome to Remix (SPA Mode)</h1>
+      <h1 className="text-2xl font-bold text-blue-500">Welcome to Remix (SPA Mode)</h1>
-      <ul>
+      <ul className="list-disc list-inside">
        <li>
          <a
            target="_blank"
            href="https://remix.run/future/spa-mode"
            rel="noreferrer"
          >
            SPA Mode Guide
          </a>
        </li>
        <li>
          <a target="_blank" href="https://remix.run/docs" rel="noreferrer">
            Remix Docs
          </a>
        </li>
      </ul>
    </div>
  );
}
```

手元で起動して、ちゃんと適用されているか確認しましょう。

```sh
$ pnpm dev

> remix-spa-tailwindcss-speedrun@ dev /Users/coji/progs/spike/remix/remix-spa-tailwindcss-speedrun
> remix vite:dev

  ➜  Local:   http://localhost:5173/
  ➜  Network: use --host to expose
  ➜  press h + enter to show help
```

![](/images/remix-spa-tailwindcss-speedrun//local-check.png)

できましたね！

## 4. Cloudflare Pages にデプロイ

### 4-1. vite でビルドする

vite でデプロイするファイルをビルドします。build/client 以下に生成されます。

```sh
$ pnpm build

> remix-spa-tailwindcss-speedrun@ build /Users/coji/progs/spike/remix/remix-spa-tailwindcss-speedrun
> remix vite:build

vite v5.1.5 building for production...
✓ 81 modules transformed.
build/client/.vite/manifest.json                0.95 kB │ gzip:  0.28 kB
build/client/assets/root-D9o5_-Jr.css           4.78 kB │ gzip:  1.49 kB
build/client/assets/_index-y6CHwiUj.js          0.70 kB │ gzip:  0.39 kB
build/client/assets/root-DQS6S_dm.js            1.51 kB │ gzip:  0.87 kB
build/client/assets/jsx-runtime-BlSqMCxk.js     8.09 kB │ gzip:  3.04 kB
build/client/assets/entry.client-Cnfe980b.js   14.06 kB │ gzip:  5.00 kB
build/client/assets/components-uA9UXrr3.js    212.90 kB │ gzip: 68.90 kB
✓ built in 533ms
vite v5.1.5 building SSR bundle for production...
✓ 6 modules transformed.
build/server/.vite/manifest.json               0.19 kB
build/server/assets/server-build-D9o5_-Jr.css  4.78 kB
build/server/index.js                          4.44 kB
SPA Mode: index.html has been written to your build/client directory
✓ built in 50ms
```

### 4-2. wrangler のインストール

cloudflare pages にデプロイするために、wrangler をインストールします。

```sh
$ pnpm add -D wrangler
 WARN  2 deprecated subdependencies found: rollup-plugin-inject@3.0.2, sourcemap-codec@1.4.8
Packages: +41 -1
+++++++++++++++++++++++++++++++++++++++++-
Progress: resolved 865, reused 785, downloaded 0, added 41, done

devDependencies:
+ wrangler 3.30.1

```

### 4-3. Cloudflare Page のプロジェクトを作成

wrangler を使って cloudflare pages のプロジェクトを新規作成します。

```
$ pnpm wrangler pages project create remix-spa-tailwindcss-speedrun
✔ Enter the production branch name: … main
✨ Successfully created the 'remix-spa-tailwindcss-speedrun' project. It will be available at https://remix-spa-tailwindcss-speedrun.pages.dev/ once you create your first deployment.
To deploy a folder of assets, run 'wrangler pages deploy [directory]'.
```

### 4-4. デプロイ

wrangler を使って cloudflare pages にデプロイします。

```sh
$ pnpm wrangler pages deploy build/client
🌍  Uploading... (9/9)

✨ Success! Uploaded 9 files (3.95 sec)

✨ Deployment complete! Take a peek over at https://bea2e17b.remix-spa-tailwindcss-speedrun.pages.dev
```

しばらくしてからプロジェクト作成時に表示された公開 URL にアクセスします。

![](/images/remix-spa-tailwindcss-speedrun//deploy-check.png)

できました！

以下でアクセス可能です。
https://remix-spa-tailwindcss-speedrun.pages.dev/

# まとめ

以上、Remix SPA Mode に TailwindCSS を入れて、Cloudflare Pages にデプロイするスピードランでした。

今回のソースコード一式は以下でご覧いただけます。
https://github.com/coji/remix-spa-tailwindcss-speedrun

参考になると幸いです 😆
Remix はいいね！
