---
title: "React Router v7でURLSearchParamsの状態を保持したまま「戻る」を実装するカスタムフック"
emoji: "🔄"
type: "tech"
topics: ["reactrouter", "react", "typescript"]
published: true
---
# React Router v7でURLSearchParamsの状態を保持したまま「戻る」を実装するカスタムフック

## これはなに？

React Routerを使ってて、一覧ページのURLSearchParams（クエリパラメータ）で管理しているフィルタ設定が詳細ページから戻ったときにリセットされる問題、ありがちですよね。

そこで `useSmartNavigation` というカスタムフックを作りました。これを使うと：

- 一覧ページ：`useSmartNavigation({ autoSave: true, baseUrl: '/tasks' })`
- 詳細ページ：`const { goBack } = useSmartNavigation({ baseUrl: '/tasks' })`

たったこれだけで、フィルタ状態を保持したまま「戻る」ができます。sessionStorageを使ってURLを自動保存してくれるので、開発者はURL管理を気にしなくて済みます。

## 実際に動いてるデモ

百聞は一見にしかず。まずは実際のデモを触ってみてください：

1. **フィルタされた一覧ページ**：[https://shadcn-admin-react-router.vercel.app/tasks?priority=high&priority=medium&status=backlog](https://shadcn-admin-react-router.vercel.app/tasks?priority=high&priority=medium&status=backlog)
   - Priority: High, Medium、Status: Backlogでフィルタされた状態

2. **詳細ページへ移動**：任意のタスクをクリックして詳細ページへ
   - 例：[https://shadcn-admin-react-router.vercel.app/tasks/TASK-7878](https://shadcn-admin-react-router.vercel.app/tasks/TASK-7878)

3. **「戻る」ボタンをクリック**：フィルタ状態が保持されたまま一覧に戻ります

実装は[GitHubリポジトリ](https://github.com/coji/shadcn-admin-react-router/)で公開してます。

## なんでこんなの作ったの？

正直言うと、毎回同じようなコードを書くのが面倒だったからです。

一覧ページで「ステータス：未完了、担当者：自分、2ページ目」みたいに設定したのに、詳細ページから戻ったら全部リセット。ユーザーからしたら「また設定し直しかよ...」ってなりますよね。

かといって、`useSearchParams`でゴリゴリURL組み立てるのも大変だし、Props経由で遷移元情報を渡すのもめんどくさい。

そんなわけで、「いい感じに自動でやってくれるやつ」を作りました。

## 使い方

### まずはフックを用意

```typescript
// app/hooks/use-smart-navigation.ts
import { useEffect } from 'react'
import { useLocation, useNavigate } from 'react-router'

export function useSmartNavigation(options?: {
  autoSave?: boolean
  baseUrl?: string
}) {
  const location = useLocation()
  const navigate = useNavigate()
  const { autoSave = false, baseUrl = '/' } = options || {}

  // 自動保存機能
  useEffect(() => {
    if (autoSave && location.pathname === baseUrl) {
      sessionStorage.setItem(
        `nav_${baseUrl}`,
        location.pathname + location.search,
      )
    }
  }, [location, baseUrl, autoSave])

  // SSR対応
  const getBackUrl = () => {
    if (typeof window === 'undefined') return baseUrl
    const savedUrl = sessionStorage.getItem(`nav_${baseUrl}`)
    return savedUrl || baseUrl
  }

  const goBack = () => {
    navigate(getBackUrl())
  }

  return {
    goBack,
    backUrl: getBackUrl(),
  }
}
```

### 一覧ページで使う（保存する側）

```tsx
// タスク一覧ページ
export default function TasksPage() {
  // これだけ！現在のURL（クエリパラメータ込み）を自動保存
  useSmartNavigation({
    autoSave: true,
    baseUrl: '/tasks',
  })

  return (
    <div>
      {/* フィルタとかページネーションとか */}
      {/* タスクリスト */}
    </div>
  )
}
```

### 詳細ページで使う（戻る側）

```tsx
// タスク詳細ページ
export default function TaskDetailPage() {
  const { goBack, backUrl } = useSmartNavigation({
    baseUrl: '/tasks', // 一覧ページと同じbaseUrlを指定
  })

  return (
    <div>
      <h1>タスク詳細</h1>
      {/* 詳細情報 */}
      
      {/* 関数で戻る */}
      <button onClick={goBack}>一覧へ戻る</button>
      
      {/* Linkでも戻れる */}
      <Link to={backUrl}>一覧へ戻る</Link>
    </div>
  )
}
```

## 実際の動作例

1. タスク一覧で「未完了のタスク、2ページ目」を表示中
   - URL: `/tasks?status=pending&page=2`
   - この状態がsessionStorageに自動保存される

2. あるタスクの詳細ページへ移動
   - URL: `/tasks/123`

3. 「一覧へ戻る」をクリック
   - `/tasks?status=pending&page=2` に戻る（フィルタ状態維持！）

保存されたURLがない場合（詳細ページのURLを直接シェアされて来たときなど）は、baseUrl（`/tasks`）にフォールバックするので安心です。

## オプションの説明

- `autoSave`: trueにすると現在のURLを自動保存（一覧ページで使う）
- `baseUrl`: 保存のキーとフォールバック先URL（複数の一覧がある場合は個別に指定）

例えば商品一覧（`/products`）と注文一覧（`/orders`）がある場合、それぞれ別のbaseUrlを指定すれば独立して状態管理できます。

## こんな場面で便利

- **ECサイトの商品一覧**: カテゴリ・価格帯・並び順を設定して詳細を見た後、同じ条件の一覧に戻りたい
- **管理画面のデータ一覧**: 検索条件やページネーションを維持したまま編集ページから戻りたい  
- **タスク管理アプリ**: フィルタした状態から新規作成・編集して、同じフィルタ状態に戻りたい

## 技術的なポイント

- sessionStorageを使用（タブを閉じると消える、タブ間では共有されない）
- SSR対応済み（`window`オブジェクトがない環境でも動く）
- React Router v7で動作確認済み

## さいごに

「また同じコード書いてる...」って思ったときに作ったものですが、意外と汎用的に使えて重宝してます。

URLパラメータの管理って地味に面倒だけど、ユーザー体験には大きく影響する部分なので、こういうの使って楽できるところは楽しちゃいましょう。

何か改善案があったら教えてください！
