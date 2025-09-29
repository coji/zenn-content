---
title: "React Router v7のデータモードを使ってみた+Conformのfuture機能"
emoji: "🧩"
type: "tech"
topics: ["react", "reactrouter", "conform", "typescript"]
published: true
---

## これはなに？

React Router v7 のデータモード（loader/action）を実際に触ってみた検証記事です。将来自作のブラウザ拡張で使うことを想定して Memory Router での実装を試し、ついでに Conform の future 機能も試してみました。データモードを使ったことがなかったので、まずは基本的な動作を確認したかったんです。

実際に動作するデモは以下URLに公開しています。
https://data-router-test.vercel.app/

ソースコードは以下リポジトリで公開しています。
https://github.com/coji/data-router-test

## 動機：データモードを触ってみたかった

ブラウザ拡張を作る予定があって、React Router v7 を使いたいなと思っていました。でも、そもそもデータモード（loader/action）を使ったことがなかったので、まずは基本的な動作を理解したくて検証プロジェクトを作ってみました。

ブラウザ拡張では URL バーに依存できないので、Memory Router での実装になるはずです。そこで今回は Memory Router でデータモードがちゃんと動くか確認してみることにしました。

## 検証用プロジェクトのセットアップ

ブラウザ拡張での利用を想定して、検証用のプロジェクトを作成しました。

```bash
pnpm create vite@latest data-router-test --template react-ts
cd data-router-test
pnpm install
```

必要なパッケージをインストールします。

```bash
# React Router v7
pnpm add react-router@7

# Conform + Zod (future機能を使用)
pnpm add @conform-to/react@1.11.0 @conform-to/zod@1.11.0 zod@4

# UI関連
pnpm add sonner tailwindcss@4 @tailwindcss/vite
```

## Memory Routerを使った実装

ブラウザ拡張では通常のブラウザのURLとは独立したルーティングが必要なので、`createMemoryRouter`を使います。

### ルーター設定（routes.tsx）

```tsx
import { createMemoryRouter } from 'react-router'
import App, { loader } from './routes/_index/route'
import Layout from './routes/_layout'
import Form, { action } from './routes/form/route'

export const router = createMemoryRouter([
  {
    Component: Layout,
    children: [
      {
        path: '/',
        loader,
        Component: App,
      },
      {
        path: '/form',
        Component: Form,
        action,
      },
    ],
  },
])
```

### エントリーポイント（main.tsx）

```tsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { RouterProvider } from 'react-router'
import { Toaster } from '~/components/ui/sonner'
import './index.css'
import { router } from './routes'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <Toaster richColors closeButton />
    <RouterProvider router={router} />
  </StrictMode>,
)
```

## データモードの実装

### Loaderを使ったデータ取得

データモードの肝は、コンポーネントレンダリング前にデータを取得できることです。

```tsx
export const loader = async ({ request }: LoaderFunctionArgs) => {
  // APIコールやストレージからのデータ取得
  const data = await fetchSomeData()
  return { data }
}

export default function HomePage() {
  const { data } = useLoaderData<typeof loader>()
  // dataが保証された状態でレンダリング
  return <div>{data.message}</div>
}
```

## Conformのfuture機能を使ったフォーム実装

Conform v1.11から導入されたfuture機能を使って、よりシンプルなフォーム実装を試してみました。

### future機能のポイント

future機能では、新しいAPIが使えます。特に`parseSubmission`と`report`の組み合わせで、エラー処理が劇的にシンプルになりました。

```tsx
// futureパスからインポート
import { parseSubmission, report, useForm } from '@conform-to/react/future'
import { coerceFormValue } from '@conform-to/zod/v4/future'

// スキーマ定義（coerceFormValueで型強制）
const schema = coerceFormValue(
  z.object({
    name: z.string({ error: '名前は必須です' }),
  }),
)

// Action関数でフォーム送信を処理
export const action = async ({ request }: ActionFunctionArgs) => {
  const submission = parseSubmission(await request.formData())
  const result = schema.safeParse(submission.payload)

  if (!result.success) {
    // reportでエラーを返す
    return { lastResult: report(submission, { error: result.error }) }
  }

  // 成功時はresetでフォームをクリア
  return { lastResult: report(submission, { reset: true }) }
}

// コンポーネント側
export default function FormPage() {
  const actionData = useActionData<typeof action>()
  const { form, fields, intent } = useForm({
    lastResult: actionData?.lastResult,
    schema,
  })

  return (
    <Form method="POST" {...form.props}>
      <input name={fields.name.name} />
      <div>{fields.name.errors}</div>
      <button type="submit">送信</button>
      <button onClick={() => intent.reset()}>リセット</button>
    </Form>
  )
}
```

## 実装のポイント

### Memory Router × データモード

Memory Routerとデータモードの組み合わせは、ブラウザ拡張開発に最適です。URLバーに依存しないルーティングでありながら、loader/actionの恩恵を受けられます。

### Conformのfuture機能の威力

従来のConformと比べて、future機能は以下の点で優れています。

- **parseSubmission**: FormDataの解析が1行で完了
- **report関数**: エラーとリセットの状態管理を統一
- **coerceFormValue**: 型変換を自動化（"123" → 123など）
- **intent.reset()**: フォームリセットがメソッド1つで実現

## ブラウザ拡張での活用イメージ

実際にブラウザ拡張を作る際は、`chrome.storage` APIとの組み合わせが強力になりそうです。

```tsx
// loaderでストレージからデータ取得
export const loader = async () => {
  const { settings } = await chrome.storage.local.get(['settings'])
  return { settings: settings || {} }
}

// actionでストレージに保存
export const action = async ({ request }: ActionFunctionArgs) => {
  const formData = await request.formData()
  await chrome.storage.local.set({
    settings: Object.fromEntries(formData)
  })
  return { success: true }
}
```

## まとめ

今回の検証で、React Router v7のデータモードがブラウザ拡張開発でも問題なく使えることが確認できました。Memory Routerを使えば、URLバーに依存せずにいつものReact Routerのスタイルで開発できます。

また、Conformのfuture機能は想像以上にシンプルでした。特に`parseSubmission`と`report`の組み合わせは、エラー処理のボイラープレートを大幅に削減してくれます。

次回は実際にブラウザ拡張を作ってみたいと思います。普段使い慣れたツールがそのまま使えるのは本当に便利ですね。

## 参考リンク

- [React Router v7 Documentation](https://reactrouter.com/)
- [Conform](https://conform.guide/)
- [デモサイト](https://data-router-test.vercel.app/)
- [ソースコード（GitHub）](https://github.com/coji/data-router-test)
