---
title: "React Router v7 の末尾スラッシュ問題 ── Hono で解決"
emoji: "🔄"
type: "tech"
topics: ["reactrouter", "hono", "seo", "prerender", "redirect"]
published: true
---

## これはなに？

React Router v7 の prerender 機能を使って静的生成したページで、末尾スラッシュへの 301 リダイレクトが発生していました。SEO に悪影響があったので、React Router のソースコードを調べて、Hono サーバーへの移行で解決した話です。

https://github.com/rphlmr/react-router-hono-server

## 問題の発見 ── 301 リダイレクトが発生

`/ja/area/marunouchi/cafe/rating` にアクセスすると、自動的に `/ja/area/marunouchi/cafe/rating/` へ 301 リダイレクトされていました。これには canonical URL（末尾スラッシュなし）と実際の URL が不一致になるという問題がありました。不要な HTTP リダイレクトでパフォーマンスが低下しますし、Google がどちらの URL を正規とすべきか混乱する可能性もあります。

環境は React Router v7 で、SSR + prerender による静的生成を使っていて、`@react-router/serve` でサーバー配信していました。

## 原因の調査 ── Express の仕様だった

まず prerender によって生成されるファイル構造を確認しました。各ルートに対して `[path]/index.html` という形式でファイルが生成されていました。たとえば `/rating` のルートは `rating/index.html` として出力されます。

`@react-router/serve` は内部で Express の `express.static` を使っています。`express.static` のデフォルト動作として、ディレクトリへのアクセス時に末尾スラッシュがない場合、まず `/rating` へのリクエストを受信して、`rating` ディレクトリが存在することを確認します。そして 301 リダイレクトを `/rating/` に発行してから、`/rating/index.html` を配信します。この `redirect: true` がデフォルト設定で、変更できません。

さらに React Router のソースコードを確認したところ、`prerenderRoute` 関数で出力ファイル名が `index.html` として**ハードコードされていました**。つまり React Router v7 では必ず `[path]/index.html` として出力されて、`[path].html` のような形式に変更する設定は存在しません。

## 解決策 ── Hono サーバーへの移行

いくつか選択肢を検討しました。prerender を諦めて SSR のみにするとパフォーマンスが犠牲になります。Express の設定を変更したいんですが `@react-router/serve` では不可能です。カスタムサーバーに切り替える手もありますが実装コストが高いです。

そこで **Hono サーバーに切り替える**ことにしました。GitHub の Discussion で紹介されていた `react-router-hono-server` は、Hono の `serveStatic` がリダイレクトを発行しないという特徴があります。React Router v7 に特化した設計で、移行が容易（プラグインとして提供）です。Express より高速でモダンなのも魅力でした。

## 移行の手順

まず必要なパッケージをインストールします。

```bash
pnpm add react-router-hono-server hono @hono/node-server
```

`vite.config.ts` に `reactRouterHonoServer()` を追加します。`reactRouter()` の前に配置するのがポイントです。

```typescript
import { reactRouter } from '@react-router/dev/vite'
import { reactRouterHonoServer } from 'react-router-hono-server/dev'
import { defineConfig } from 'vite'

export default defineConfig({
  plugins: [
    reactRouterHonoServer(), // reactRouter() の前に配置
    reactRouter(),
  ],
})
```

`react-router.config.ts` で `serverBuildFile` を指定します。Hono では必須の設定です。

```typescript
import type { Config } from '@react-router/dev/config'

export default {
  ssr: true,
  serverBuildFile: 'server/index.js', // Hono では必須
  prerender: () => {
    // 既存の prerender 設定
  },
} satisfies Config
```

`app/server.ts` を作成して、Hono サーバーのエントリーポイントにします。

```typescript
import { createHonoServer } from 'react-router-hono-server/node'

export default await createHonoServer()
```

`package.json` のスタートスクリプトを変更して、`react-router-serve` の代わりに直接 Node.js でサーバーファイルを実行するようにします。`@react-router/serve` への依存は削除できます。

## Hono の動作原理 ── リダイレクトなしで配信

Express と Hono の違いを説明します。Express `express.static` は `/rating` にアクセスすると、rating ディレクトリを検出して、301 リダイレクトを `/rating/` に発行してから `/rating/index.html` を配信します。

一方、Hono `serveStatic` は `/rating` にアクセスすると、rating ディレクトリを検出して、直接 `/rating/index.html` の内容を返します。リダイレクトなしです。

Hono の内部実装を見ると、ディレクトリ判定時にパスを `index.html` に変換してから、`getContent()` でファイル内容を直接返しています。リダイレクトレスポンスを返すのではなく、ファイル内容を直接返すので、クライアントからはリダイレクトが発生していることに気づきません。

## 結果 ── リダイレクトの解消に成功

ビルドは問題なく完了して、Hono サーバーとして動作するファイルが生成されました。リダイレクトが発生せず、`/ja/area/marunouchi/cafe/rating` で直接コンテンツを配信できるようになりました。canonical URL と実際の URL が一致して、不要な HTTP リダイレクトが削除されました。

リダイレクトがなくなったことで、HTTP ラウンドトリップの削減、ページロード時間の短縮、SEO シグナルの向上といった改善が期待できます。

## 注意点

`react-router-hono-server` への移行時にいくつか注意が必要です。ESM モードのみサポートされていて、CommonJS では動作しません。`@react-router/serve` からの移行が必要で、カスタムミドルウェアを使っている場合は書き換えが必要です。React Router v7 専用なので、他のバージョンでは動作保証がありません。

移行後は必ず `pnpm run validate` で lint、format、typecheck、test を実行して、`pnpm run build` でビルドが成功するか確認しておくと安心です。

## まとめ

React Router v7 の prerender による末尾スラッシュリダイレクト問題を、Hono サーバーへの移行で解決しました。

今回わかったことをまとめます。prerender の出力形式は変更できません。React Router のソースコードで `index.html` がハードコードされていて、設定で変更する方法は存在しません。サーバー実装が挙動を左右します。Express の `express.static` がリダイレクトを発行するのに対して、Hono の `serveStatic` はリダイレクトなしで配信します。ソースコード調査が重要で、ドキュメントに記載がない挙動も、ソースを見れば理解できます。GitHub の Discussion や Issue も貴重な情報源です。

Hono への移行で、SEO フレンドリーな URL 構造、パフォーマンス向上、モダンで型安全な実装が実現できました。同じ問題に直面している方の参考になれば嬉しいです。
