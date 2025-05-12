---
title: "Conformとdnd kitで作る！ソート可能なネスト配列動的フォームの実装ガイド (React Router v7"
emoji: "📝"
type: "tech"
topics: ["conform", "dndkit", "zod", "react-router", "shadcnui"]
published: false
---

# Conformとdnd kitで作る！ソート可能なネスト配列動的フォームの実装ガイド (React Router v7)

## はじめに

Webアプリケーション開発において、複雑なフォームの実装はしばしば頭痛の種となります。特に、ユーザーが自由に行を追加・削除したり、順番を入れ替えたりできる「動的なリスト」をフォーム内に含める場合、状態管理やバリデーション、UIの整合性を保つのが難しくなります。ネストされた構造（リストの中にさらにリストがある）になると、その複雑さはさらに増します。

この記事では、Reactベースのフルスタックフレームワークである **Remix** (内部で **React Router v7** を使用) を例に取り、以下の技術スタックを用いて、**動的に増減可能で、かつドラッグ＆ドロップで並び替えも可能なネストされたオブジェクト配列**を扱うフォームの実装方法を詳細に解説します。

* **Conform**: 宣言的なフォームバリデーションと状態管理ライブラリ
* **dnd kit**: 高機能で柔軟なドラッグ＆ドロップライブラリ
* **Zod**: 型安全なスキーマ定義・バリデーションライブラリ
* **Remix (React Router v7)**: フルスタックWebフレームワーク

具体的には、「チーム」を複数管理し、各チーム内に所属する「メンバー」をリスト形式で表示・編集できるフォームを作成します。チーム自体も追加・削除・上下移動が可能で、さらに各チーム内のメンバーリストも同様に追加・削除・ドラッグ＆ドロップでの並び替えができる、という実践的な要件に挑戦します。

**実際に動作するデモはこちらで確認できます:**
▶ [**デモを見る**](https://www.techtalk.jp/demo/conform/nested-array)

![完成形フォームのイメージ図](https://user-images.githubusercontent.com/11521994/282627094-05623100-6320-4b8f-a05a-a9d7c4113a89.png)
*チームカードが並び、各カード内でメンバーをドラッグ＆ドロップで並び替えたり、追加・削除できるUI*

Conformで動的に増減するネスト配列を扱い、さらにdnd kitでその要素を並び替える、という組み合わせの実装例はまだ少ないため、同様の実装を目指す方の具体的なガイドとなることを目指します。特に、Reactの`key` propの適切な設定が、この種のUIを実現する上でいかに重要かについても深掘りします。

**対象読者:**

* ReactおよびRemix (またはNext.jsなどReact Routerベースのフレームワーク) の基本的な知識がある方
* Conform, Zod, dnd kitを使ったことがある、または興味がある方
* 動的なリストを含む複雑なフォームの実装方法を探している方

## なぜこの組み合わせが難しいのか？

Conformはフォームの状態管理、特にZodのようなスキーマと連携したバリデーションやエラーハンドリングを、非常にシンプルかつ宣言的に記述できる強力なライブラリです。`useForm`, `useField`, `getFieldList` といったフックやユーティリティを使えば、リスト形式の入力も比較的容易に扱えます。

一方、dnd kitはアクセシビリティやパフォーマンスにも配慮された、柔軟性の高いドラッグ＆ドロップ機能を提供します。`useSortable` フックなどを使えば、リストの並び替えUIも効率的に実装できます。

しかし、これらを組み合わせ、特に「ネストされた」動的リスト（チームリストの中にメンバーリストがある構造）で並び替えを実現しようとすると、いくつかの特有の課題が生じます。

1. **状態の同期:** Conformが管理するフォームの状態（配列の順序や各要素の値、エラー状態など）と、dnd kitによるUI上の並び替え操作（DOM要素の物理的な移動）を正確に同期させる必要があります。ユーザーが要素をドロップした瞬間に、Conformの内部状態も更新され、その結果がUIに正しく反映されなければなりません。
2. **Reactのレンダリングと`key` prop:** Reactがリスト要素を効率的に、かつ正確に更新するためには、各要素に**安定した一意な`key` prop**を割り当てる必要があります。動的なリスト、特に並び替えが発生する場合、この`key`の管理が非常に重要になります。不適切な`key`（例えば配列の`index`）を使うと、並び替え後に表示が崩れたり、入力内容が意図しない要素に紐づいてしまう（例えば、Aさんの行を移動したら中身がBさんのものになってしまう）といった問題が発生します。
3. **Conformの内部状態との整合性:** Conformは各フォームフィールドの状態（値、エラー、ダーティ状態、バリデーションメッセージIDなど）を内部で厳密に管理しています。ドラッグ＆ドロップによるDOM要素の並び替え後も、これらの内部状態と対応するDOM要素が正しく紐付き続けている必要があります。`key` propの問題とも密接に関連します。

この記事では、これらの課題をRemix (React Router v7) のアーキテクチャの上で、Conformとdnd kitを用いてどのように解決していくかを、具体的なコードと共にステップバイステップで解説します。

## 実装の全体像

まず、今回のデモアプリケーションのファイル構成と、主要なコンポーネントの役割を見てみましょう。Remixの規約に基づいたルーティング (`routes/demo+/conform.nested-array`) を使用しています。

```plaintext
└── app
    └── routes
        └── demo+
            └── conform.nested-array  # このデモページのルートディレクトリ
                ├── components          # UIコンポーネント群
                │   ├── index.ts             # コンポーネントのエクスポート集約
                │   ├── member-list-header.tsx # メンバーリストの列ヘッダー
                │   ├── member-list-item.tsx # 各メンバー行のコンポーネント ★重要
                │   ├── member-list.tsx      # メンバーリスト全体のレイアウトコンテナ
                │   ├── sortable-item.tsx    # D&Dのための汎用ラッパー ★重要
                │   └── team-card.tsx        # 各チームを表示・編集するカード ★重要
                ├── faker.server.ts        # ダミーデータ生成ロジック (サーバーサイド限定)
                ├── route.tsx              # Remixのルートファイル (Loader, Action, Component) ★最重要
                └── schema.ts              # Zodによるフォームスキーマ定義 ★重要
```

**ソースコードはこちらで確認できます:**
▶ [**GitHubでソースを見る**](https://github.com/techtalkjp/techtalk.jp/blob/main/app/routes/demo+/conform.nested-array/route.tsx)

**データの流れと処理の概要:**

1. **`loader` (`route.tsx`)**: ユーザーが `/demo/conform/nested-array` にアクセスすると、まずサーバーサイドで `loader` 関数が実行されます。ここでは `faker.server.ts` を使用して、初期表示用のダミーのチーム・メンバーデータを生成し、JSONとしてクライアント（ブラウザ）に渡します。（Remix/React Router v7 のデータローディング機能）
2. **`schema.ts`**: Zodを使って、フォームで扱うデータの構造（チーム配列、各チームが持つメンバー配列）と、それぞれのフィールドに対するバリデーションルール（必須入力、文字形式など）を定義します。このスキーマはサーバーサイドのActionとクライアントサイドのバリデーションの両方で使用されます。
3. **Component (`route.tsx`)**: ブラウザ側でページのUIをレンダリングするメインコンポーネントです。
    * `useLoaderData` (Remix/React Router) で `loader` から渡された初期データを取得します。
    * `useActionData` (Remix/React Router) でフォーム送信後の `action` からの結果（バリデーションエラーや成功メッセージなど）を取得します。
    * `useForm` (Conform) フックを使ってフォームの状態管理を初期化します。`loader` からのデータを `defaultValue` として設定し、`action` からの結果を `lastResult` に渡します。また、クライアントサイドバリデーション用に `onValidate` を設定します。
    * Conformの `fields.teams.getFieldList()` を使って、Conformが管理するチームのリストを取得し、`map` 関数で各チームに対応する `TeamCard` コンポーネントを描画します。
    * ユーザーがフォームを送信 (`<Form method="POST">`) すると、Remix/React Routerの機能により、同じルートの `action` 関数がサーバーサイドで呼び出されます。
4. **`TeamCard.tsx`**:
    * 親コンポーネントから渡されたチームデータ (`name` prop、例: `'teams[0]'`) に基づき、そのチームの詳細（チーム名入力欄やメンバーリスト）を表示します。
    * `useField` (Conform) で、担当するチームフィールドの状態を取得します。
    * `teamFields.members.getFieldList()` で、そのチームに属するメンバーリストの情報をConformから取得し、`map` で `MemberListItem` コンポーネントを描画します。
    * `DndContext` と `SortableContext` (dnd kit) を設定し、このカード内のメンバーリストに対してドラッグ＆ドロップによる並び替え機能を有効化します。
    * メンバーがドラッグ＆ドロップで並び替えられた際 (`onDragEnd` イベント)、`form.reorder` (Conform) アクションを発行し、Conformが管理するメンバー配列の状態（順序）を更新します。
5. **`MemberListItem.tsx`**:
    * 各メンバーの入力フィールド（名前、性別、郵便番号、電話番号、Email）を表示します。
    * `useField` (Conform) を使って、担当するメンバーフィールド（例: `'teams[0].members[1]'`）の状態（値、エラー、IDなど）を取得し、各入力コンポーネント (`<Input>`, `<Select>` など) にバインドします。
    * `SortableItem` コンポーネントでラップされ、dnd kitによるドラッグ＆ドロップの対象となります。ドラッグ操作のためのハンドル（アイコン）もここに含みます。
    * **このコンポーネントに設定するReactの `key` prop が、並び替え時のUIの安定性を保つ上で極めて重要になります。**
6. **`SortableItem.tsx`**: `useSortable` (dnd kit) フックを内部で使用し、ラップした要素をドラッグ可能にするためのスタイルやイベントリスナーを提供する汎用的なラッパーコンポーネントです。
7. **`action` (`route.tsx`)**: フォームが送信されるとサーバーサイドで実行されます。
    * `request.formData()` で送信されたフォームデータを取得します。
    * `parseWithZod` (Conform) を使って、受け取ったデータを `schema.ts` で定義したスキーマに基づきバリデーションします。
    * バリデーションエラーがあれば、エラー情報を含んだ `submission.reply()` を返却します。これがクライアント側の `useForm` の `lastResult` に渡され、フォームにエラーメッセージが表示されます。
    * バリデーションが成功すれば、データの登録処理（このデモでは受け取ったデータを整形してそのまま返す）を行い、成功を示すデータやメッセージと共に `submission.reply()` を返却します。これも `lastResult` に渡され、成功時のUI表示（例: 登録結果テーブルの表示、トースト通知）に使用されます。

## コアとなる実装の解説

それでは、この複雑なフォームを実現するための核心部分となるコードを、ファイルごとに詳しく見ていきましょう。

### 1. スキーマ定義 (`schema.ts`)

まず、フォームで扱うデータの構造とルールをZodで定義します。ネスト構造を明確に表現できるのがZodの強みです。

```typescript
// app/routes/demo+/conform.nested-array/schema.ts
import { z } from 'zod'

// 各メンバーのデータ構造とバリデーションルール
const memberSchema = z.object({
  id: z.string().optional(), // 既存メンバーのID（UUID想定）。新規追加時はundefined
  name: z.string({ required_error: '必須' }).min(1, '必須'), // 名前は必須
  gender: z.enum(['male', 'female', 'non-binary'], { // 性別は選択肢から
    required_error: '必須',
    invalid_type_error: '性別を選択してください',
  }),
  zip: z
    .string({ required_error: '必須' })
    .regex(/^\d{3}-\d{4}$/, { message: '000-0000形式で入力してください' }), // 郵便番号形式
  tel: z
    .string({ required_error: '必須' })
    .regex(/^\d{3}-\d{4}-\d{4}$/, { message: '000-0000-0000形式で入力してください' }), // 電話番号形式
  email: z
    .string({ required_error: '必須' })
    .email({ message: '有効なメールアドレスを入力してください' }), // Email形式
})

// チームのデータ構造とバリデーションルール
export const teamSchema = z.object({
  id: z.string().optional(), // 既存チームのID
  name: z.string({ required_error: '必須' }).min(1, '必須'), // チーム名は必須
  members: z
    .array(memberSchema) // ↑で定義したmemberSchemaの配列
    .min(1, { message: 'メンバーを少なくとも1人追加してください' }), // メンバーは最低1人必要
})

// フォーム全体のデータ構造とバリデーションルール
export const formSchema = z.object({
  teams: z.array(teamSchema) // ↑で定義したteamSchemaの配列
             .min(1, { message: 'チームを少なくとも1つ追加してください' }), // チームは最低1つ必要
})

// TypeScriptの型をスキーマから生成
export type MemberSchema = z.infer<typeof memberSchema>
export type TeamSchema = z.infer<typeof teamSchema>
export type FormSchema = z.infer<typeof formSchema>
```

ここでは、`memberSchema`, `teamSchema`, `formSchema` という3つのスキーマを定義し、`array()` を使ってネスト構造を表現しています。各フィールドには `required_error` や `regex`, `email` などで詳細なバリデーションルールとカスタムエラーメッセージを設定しています。`id` フィールドを `optional()` とすることで、新規作成（IDなし）と更新（IDあり）の両方のケースに対応できるようにしています。最後に `z.infer` を使って、これらのスキーマからTypeScriptの型を自動生成しています。これにより、コード全体で型安全性を保つことができます。

### 2. Conform の設定とリスト操作 (`route.tsx`)

Remixのルートコンポーネント (`route.tsx`) で、フォーム全体の初期化、チームリストの表示、チームの追加・削除・並び替え、そしてフォーム送信時のデータハンドリング（LoaderとAction）を行います。

```typescript
// app/routes/demo+/conform.nested-array/route.tsx
import { FormProvider, getFormProps, useForm } from '@conform-to/react'
import { parseWithZod } from '@conform-to/zod'
import { EllipsisVerticalIcon } from 'lucide-react'
import { Form, useActionData, useLoaderData } from '@remix-run/react' // Remix/React Routerのフック
import { json, type ActionFunctionArgs, type LoaderFunctionArgs } from '@remix-run/node' // Remixのサーバーサイド関数
import { dataWithSuccess } from 'remix-toast' // トースト通知用 (オプション)
// ... UIコンポーネントのimport
import { TeamCard } from './components'
import { fakeId, fakeName, fakeGender, fakeZip, fakeTel, fakeEmail } from './faker.server' // ダミーデータ生成
import { formSchema } from './schema' // Zodスキーマ

// サーバーサイド: ページ読み込み時のデータ取得
export const loader = async ({ request }: LoaderFunctionArgs) => {
  // fakerはブラウザバンドルに含めたくないのでサーバーサイドでのみ使用
  const fakeMembers = Array.from({ length: 2 }, () => ({ // 初期メンバー数を減らす
    id: fakeId(), name: fakeName(), gender: fakeGender(),
    zip: fakeZip(), tel: fakeTel(), email: fakeEmail(),
  }))
  const teams = [
    { id: 't1', name: 'Alpha Team', members: fakeMembers.slice(0, 1) },
    { id: 't2', name: 'Bravo Team', members: fakeMembers.slice(1, 2) },
  ]
  // 初期データをJSONで返す
  return json({ teams })
}

// サーバーサイド: フォーム送信時の処理
export const action = async ({ request }: ActionFunctionArgs) => {
  const formData = await request.formData()
  const submission = parseWithZod(formData, { schema: formSchema }) // Conform + Zodでバリデーション

  if (submission.status !== 'success') {
    // バリデーション失敗時: エラー情報を含む結果を返す
    return json({ lastResult: submission.reply(), result: null })
  }

  // バリデーション成功時: 送信された値 (submission.value) を使って処理
  // このデモでは、IDがない場合に仮のIDを付与してそのまま返す
  const processedResult = submission.value.teams.map((team, teamIndex) => ({
    ...team,
    id: team.id ?? `new-team-${teamIndex + 1}`,
    members: team.members.map((member, memberIndex) => ({
      ...member,
      id: member.id ?? `new-member-${teamIndex + 1}-${memberIndex + 1}`,
    })),
  }))

  // 成功メッセージと共に、処理結果とリセット指示を含む結果を返す
  return json(dataWithSuccess(
    { lastResult: submission.reply({ resetForm: true }), result: processedResult },
    { message: '登録しました', description: '登録されたデータを確認してください' }
  ))
}

// クライアントサイド: UIコンポーネント
export default function ConformNestedArrayDemo() {
  const { teams: defaultTeams } = useLoaderData<typeof loader>() // Loaderからの初期値
  const actionData = useActionData<typeof action>() // Actionからの結果

  const [form, fields] = useForm({
    // Actionの結果をConformに渡す (エラー表示やフォームリセットのため)
    lastResult: actionData?.lastResult,
    // Loaderからのデータを初期値として設定
    defaultValue: { teams: defaultTeams },
    // クライアントサイドでも同じZodスキーマでバリデーション
    onValidate: ({ formData }) => parseWithZod(formData, { schema: formSchema }),
    // フォームに一意なIDを付与 (複数のフォームがある場合に重要)
    id: 'nested-array-form',
    // shouldValidate: 'onBlur', // 必要ならバリデーションタイミングを変更
  })

  // Conformからチームリストのメタ情報を取得
  const teams = fields.teams.getFieldList()

  return (
    <Stack>
      {/* Conformのコンテキストを提供 */}
      <FormProvider context={form.context}>
        {/* Remix/React RouterのFormコンポーネントを使用 */}
        {/* getFormPropsでConformに必要な属性(id, onSubmit等)を展開 */}
        <Form method="POST" {...getFormProps(form)}>
          <Stack>
            {/* チームリストをループで描画 */}
            {teams.map((team, index) => (
              <TeamCard
                key={team.key} // ★ Conformが提供する安定したリストキーを使用
                formId={form.id} // 子コンポーネントがフォームを特定できるようにIDを渡す
                name={team.name} // このチームへのパス (e.g., 'teams[0]')
                menu={ // チーム操作メニュー (上下移動、削除)
                  <DropdownMenu>
                    <DropdownMenuTrigger asChild>
                      <Button type="button" variant="ghost" size="icon"><EllipsisVerticalIcon /></Button>
                    </DropdownMenuTrigger>
                    <DropdownMenuContent>
                      {/* 上へ移動ボタン */}
                      <DropdownMenuItem
                        disabled={index === 0}
                        onClick={() => form.reorder({ name: fields.teams.name, from: index, to: index - 1 })}
                      >Move Up</DropdownMenuItem>
                      {/* 下へ移動ボタン */}
                      <DropdownMenuItem
                        disabled={index === teams.length - 1}
                        onClick={() => form.reorder({ name: fields.teams.name, from: index, to: index + 1 })}
                      >Move Down</DropdownMenuItem>
                      {/* 削除ボタン */}
                      <DropdownMenuItem
                        className="text-destructive"
                        onClick={() => form.remove({ name: fields.teams.name, index })}
                      >Remove</DropdownMenuItem>
                    </DropdownMenuContent>
                  </DropdownMenu>
                }
              />
            ))}

            {/* チーム追加ボタン */}
            <Button
              variant="outline"
              // insertアクションに必要な属性をConformから取得して展開
              {...form.insert.getButtonProps({ name: fields.teams.name })}
            >
              Add Team
            </Button>

            {/* 送信ボタン */}
            <Button type="submit">Submit</Button>
          </Stack>
        </Form>
      </FormProvider>

      {/* Actionで成功結果(result)が返ってきた場合に登録内容を表示 */}
      {actionData?.result && (
        <Stack className="mt-8 text-xs">
          <div className="font-semibold">登録データ:</div>
          {/* ... 登録結果テーブルの表示 (省略) ... */}
        </Stack>
      )}
    </Stack>
  )
}
```

* **Loader/Action (Remix/React Router v7)**: Remixの規約に従い、`loader` で初期データを、`action` でフォーム送信処理をサーバーサイドで記述します。これにより、UIロジックとビジネスロジックが分離されます。
* **`useForm` (Conform)**: クライアントサイドでフォームの状態管理を初期化します。`useLoaderData` からの初期値、`useActionData` からのサーバーバリデーション結果を連携させ、`onValidate` でクライアントサイドバリデーションも設定します。
* **`fields.teams.getFieldList()`**: Conformが管理する `teams` 配列の各要素に対応するメタ情報（`key`, `name`, `defaultValue` など）を持つオブジェクトのリストを取得します。
* **リストのレンダリングと `key`**: `teams.map` を使う際、Reactの `key` prop には **`team.key`** を指定します。これはConformが内部でリスト要素ごとに生成・管理する安定した一意なキーであり、要素の追加・削除・並び替え（`form.reorder`）が行われても、Reactが要素を正しく追跡し、効率的かつ正確にDOMを更新するために不可欠です。配列の `index` を `key` に使うと、並び替え時に問題が発生します。
* **リスト操作**:
  * チームの追加には `form.insert.getButtonProps({ name: fields.teams.name })` を使います。これにより、クリック時に新しいチーム要素を `teams` 配列に追加するための情報がボタンに自動的に設定されます。
  * チームの削除には `form.remove({ name: fields.teams.name, index })` を使います。
  * チームの並び替えには `form.reorder({ name: fields.teams.name, from: index, to: index +/- 1 })` を使います。
    これらはConformが提供するAPIで、内部の状態配列を安全に操作します。

### 3. チームカードと dnd kit の設定 (`TeamCard.tsx`)

`TeamCard` コンポーネントは、1つのチームの情報（チーム名）と、そのチームに属するメンバーのリストを表示・編集する役割を担います。メンバーリストのドラッグ＆ドロップによる並び替え機能は、このコンポーネント内で dnd kit を使って実装されます。

```typescript
// app/routes/demo+/conform.nested-array/components/team-card.tsx
import { getInputProps, useField, type FieldName } from '@conform-to/react'
import { DndContext, PointerSensor, useSensor, useSensors, type DragEndEvent, closestCenter } from '@dnd-kit/core'
import { restrictToVerticalAxis, restrictToParentElement } from '@dnd-kit/modifiers'
import { SortableContext, verticalListSortingStrategy } from '@dnd-kit/sortable'
import { TrashIcon } from 'lucide-react'
import { useId } from 'react'
// ... UIコンポーネントのimport
import { MemberList, MemberListHeader, MemberListItem } from './' // components/index.tsからインポート
import type { FormSchema, TeamSchema } from '../schema'

interface TeamCardProps extends React.ComponentProps<typeof Card> {
  formId: string // 親から渡されるフォームID
  name: FieldName<TeamSchema, FormSchema> // このチームへのパス (e.g., 'teams[0]')
  menu: React.ReactNode // チーム操作メニュー
}

export const TeamCard = ({ formId, name, menu, className }: TeamCardProps) => {
  // このカードが担当するチームフィールドの情報をConformから取得
  const [field, form] = useField(name, { formId })
  // チーム内の各フィールド (id, name, members) の情報にアクセスしやすくする
  const teamFields = field.getFieldset()
  // このチームに属するメンバーリストのメタ情報をConformから取得
  const teamMembers = teamFields.members.getFieldList()

  // DndContext用の一意なID (useIdで生成)
  const dndContextId = useId()
  // dnd kit の入力センサー (ここではポインターイベントのみ使用)
  const sensors = useSensors(useSensor(PointerSensor))

  // ★ ドラッグ終了時の処理
  const handleDragEnd = (event: DragEndEvent) => {
    const { active, over } = event
    // ドラッグ元(active)とドロップ先(over)が異なる場合のみ処理
    if (over && active.id !== over.id) {
      // active.id と over.id には、SortableContextのitemsに渡したID
      // (ここではMemberListItemのConformフィールドID) が入る
      const fromIndex = teamMembers.findIndex((m) => m.id === active.id)
      const toIndex = teamMembers.findIndex((m) => m.id === over.id)

      // findIndexで見つからないケースは基本的にないはずだが念のためチェック
      if (fromIndex === -1 || toIndex === -1) {
        console.warn('Could not find dragged item index in Conform list.')
        return
      }

      // Conformのreorderアクションを発行して内部状態を更新
      form.reorder({
        name: teamFields.members.name, // 並び替えるリストのname ('teams[0].members')
        from: fromIndex,
        to: toIndex,
      })
    }
  }

  return (
    <Card className={className}>
      <CardHeader>
        {/* ... チーム名の表示とメニュー (省略) ... */}
         <HStack>
           <Label htmlFor={teamFields.name.id} className="text-muted-foreground shrink-0">チーム名</Label>
           <Input {...getInputProps(teamFields.name, { type: 'text' })} />
           {/* チーム名のエラー表示 */}
           <div id={teamFields.name.errorId} className="text-destructive text-xs">{teamFields.name.errors}</div>
         </HStack>
      </CardHeader>
      <CardContent>
        <Stack>
          {/* チームID (あればhiddenで送信) */}
          <input {...getInputProps(teamFields.id, { type: 'hidden' })} />

          {/* === メンバーリストとD&D設定 === */}
          <MemberList> {/* グリッドレイアウトコンテナ */}
            <MemberListHeader /> {/* 列名ヘッダー */}
            {/* D&Dの全体設定 */}
            <DndContext
              id={dndContextId} // コンテキストの一意なID
              sensors={sensors} // 使用するセンサー
              collisionDetection={closestCenter} // 衝突検出アルゴリズム
              modifiers={[restrictToVerticalAxis, restrictToParentElement]} // 移動制限 (垂直方向のみ、親要素内のみ)
              onDragEnd={handleDragEnd} // ★ ドラッグ終了ハンドラ
            >
              {/* 並び替え可能な要素のコンテキスト */}
              <SortableContext
                // ★ 並び替え対象要素のIDリスト (ConformのフィールドIDを使う)
                items={teamMembers.map((m) => m.id)}
                strategy={verticalListSortingStrategy} // 並び替え戦略 (垂直リスト)
              >
                {/* メンバーリストをループで描画 */}
                {teamMembers.map((teamMember, index) => {
                  // 各メンバーフィールドのメタ情報を取得 (idフィールドの値取得のため)
                  const memberFields = teamMember.getFieldset();
                  return (
                    <MemberListItem
                      // ★★★★★ 最重要ポイント: Reactのkey prop ★★★★★
                      // 安定した一意な識別子として、メンバーのID (あれば)、なければConformのキーを使う
                      key={memberFields.id.value ?? teamMember.key}
                      formId={form.id} // フォームIDを渡す
                      name={teamMember.name} // メンバーへのパス (e.g., 'teams[0].members[0]')
                      removeButton={ // メンバー削除ボタン
                        <Button
                          variant="ghost" size="sm"
                          {...form.remove.getButtonProps({ // removeアクションに必要な属性を展開
                            name: teamFields.members.name, // 削除対象リストのname
                            index, // 削除する要素のインデックス
                          })}
                        >
                          <TrashIcon className="h-4 w-4" />
                        </Button>
                      }
                    />
                  )
                })}
              </SortableContext>
            </DndContext>
          </MemberList>
          {/* メンバーリスト全体のエラー表示 (例: 最低1人必要) */}
           <div id={teamFields.members.errorId} className="text-destructive text-xs">{teamFields.members.errors}</div>

          {/* メンバー追加ボタン */}
          <Button
            variant="outline" size="sm"
            {...form.insert.getButtonProps({ // insertアクションに必要な属性を展開
              name: teamFields.members.name, // 追加対象リストのname
              // defaultValue: { name: 'New Member', ... } // 必要なら初期値も設定可
            })}
          >
            Add a member
          </Button>
        </Stack>
      </CardContent>
    </Card>
  )
}
```

* **`useField(name, { formId })`**: 親から渡された `name` (例: `'teams[0]'`) を使って、この `TeamCard` が担当するチームのフィールド情報を Conform から取得します。
* **`teamFields.members.getFieldList()`**: そのチームに属するメンバーリストの情報（各メンバーの `key`, `name`, `id`, `getFieldset` など）を Conform から取得します。
* **dnd kit の設定**:
  * `DndContext`: ドラッグ＆ドロップ操作の全体的な設定を行います。`sensors` で入力を検知し、`collisionDetection` でドロップ先を判断し、`modifiers` で動きを制限し、`onDragEnd` で操作完了時の処理 `handleDragEnd` を指定します。
  * `SortableContext`: このコンテキスト内にある要素が並び替え可能であることを示します。`items` prop には、**並び替え対象となる各メンバーの一意なIDの配列**を渡す必要があります。ここでは `teamMembers.map((m) => m.id)` を使っています。この `m.id` は、Conform が各フィールド（この場合はメンバーリスト内の各メンバーを表すフィールド）に内部的に割り当てる一意なIDです。dnd kit はこのIDを使って、ドラッグ中の要素 (`active.id`) やドロップ先の要素 (`over.id`) を識別します。
* **`handleDragEnd` の実装**:
  * dnd kit からドラッグ元 (`active.id`) とドロップ先 (`over.id`) の要素のID（＝ConformのフィールドID）が渡されます。
  * `teamMembers.findIndex((m) => m.id === active.id)` のようにして、Conform が現在管理しているメンバーリスト (`teamMembers`) における、ドラッグ元と先の要素のインデックス (`fromIndex`, `toIndex`) を特定します。
  * `form.reorder({ name: teamFields.members.name, from: fromIndex, to: toIndex })` を呼び出します。これにより、Conform に対して、指定したリスト (`teamFields.members.name`) の `fromIndex` 番目の要素を `toIndex` 番目に移動するように指示します。Conform の内部状態が更新され、次回のReactレンダリングでUIが正しい順序で表示されます。
* **`MemberListItem` への `key` prop (最重要)**: ここで `key={memberFields.id.value ?? teamMember.key}` としている点が、この実装の安定性の鍵です。
  * `memberFields.id.value` は、そのメンバーがデータベース等から取得した際に持つ永続的なID（例: UUID）です。`loader` で読み込んだデータにはこの値が存在します。
  * `?? teamMember.key` は、もし `id` フィールドの値が存在しない場合（例えば、ユーザーが「Add a member」ボタンでクライアント側で新規追加したばかりで、まだサーバー側でIDが採番されていない状態）に、Conform が提供する一時的なキー (例: `teams[0].members[2]`) をフォールバックとして使用します。
  * **なぜこの `key` がクリティカルなのか？** React は `key` を使って、リスト内の各要素と実際のDOMノード、そしてコンポーネントインスタンスを関連付けます。並び替えが発生すると、React は `key` を頼りに要素の移動や再利用を判断します。Conform は `useField` や `getInputProps` を通じて、各フォーム要素の状態（入力値、エラーメッセージ、ダーティ状態など）を管理しています。もし `key` が不安定だったり、データの実体と一致しないもの（例えば配列の `index`）を使っていると、dnd kit が DOM 上で要素を物理的に並び替えた後、React が再レンダリングする際に、**DOM要素（見た目）と、Conformが管理する内部状態（入力値やエラー）との間の紐付けがズレてしまう**可能性があります。結果として、見た目は並び替わったのに、入力フィールドの中身が元のままだったり、他の行の内容が表示されてしまう、といった不整合が発生します。
  * `memberFields.id.value` (またはフォールバックの `teamMember.key`) は、**そのメンバーデータ自身に紐づく、安定した（並び替えによって変化しない）一意な識別子**です。これを `key` に使用することで、dnd kit による並び替え後も React は各 `MemberListItem` がどのメンバーデータに対応するのかを正確に識別し続けることができます。これにより、Conform が持つ正しい状態（入力値、エラーなど）と紐付いたUI要素が常に正しくレンダリングされ、ドラッグ＆ドロップ後の表示崩れや状態の不整合を防ぐことができるのです。

### 4. メンバーリストアイテム (`MemberListItem.tsx`)

各メンバー行の具体的なUI（入力フィールド、ドラッグハンドル、削除ボタンなど）を定義するコンポーネントです。Conform と dnd kit の両方と連携します。

```typescript
// app/routes/demo+/conform.nested-array/components/member-list-item.tsx
import { getInputProps, useField, type FieldName } from '@conform-to/react'
import { DragHandleDots2Icon } from '@radix-ui/react-icons'
import type React from 'react'
// ... UIコンポーネントのimport
import { cn } from '~/libs/utils' // className結合ユーティリティ
import { ZipInput } from '~/routes/demo+/resources.zip-input/route' // 郵便番号入力コンポーネント (カスタム)
import type { FormSchema, MemberSchema } from '../schema'
import { SortableItem } from './sortable-item' // D&Dラッパー

interface MemberProps extends React.ComponentProps<'div'> {
  formId: string // フォームID
  name: FieldName<MemberSchema, FormSchema> // このメンバーへのパス (e.g., 'teams[0].members[1]')
  removeButton: React.ReactNode // 削除ボタン要素
}
export const MemberListItem = ({ formId, name, removeButton, className }: MemberProps) => {
  // このメンバーフィールドの情報をConformから取得
  const [meta, form] = useField(name, { formId })
  // メンバー内の各フィールド (id, name, gender, ...) の情報にアクセス
  const fields = meta.getFieldset()

  return (
    // dnd kit 用のラッパーコンポーネントで全体を囲む
    <SortableItem
      id={meta.id} // ★ dnd kit がこの要素を識別するためのID (ConformのフィールドID)
      className={cn('col-span-full grid grid-cols-subgrid group', className)} // スタイルとグループ化用クラス
    >
      {/* SortableItemからドラッグハンドル用の属性とリスナーを受け取る */}
      {({ attributes, listeners }) => (
        <>
          {/* ドラッグハンドル (アイコン) */}
          <div className="flex items-center justify-center"> {/* 中央寄せのための div */}
            <DragHandleDots2Icon
              {...attributes} // dnd kit用属性 (draggableなど)
              {...listeners} // dnd kit用イベントリスナー (onPointerDownなど)
              className="text-muted-foreground cursor-grab rounded-md opacity-50 group-hover:opacity-100" // ホバーで表示
            />
          </div>

          {/* 各入力フィールドとエラー表示 */}
          {/* ID (hidden) */}
          <input {...getInputProps(fields.id, { type: 'hidden', valueType: 'string' })} />

          {/* 名前 */}
          <div>
            <Input {...getInputProps(fields.name, { type: 'text', valueType: 'string' })} />
            <div id={fields.name.errorId} className="text-destructive text-xs">{fields.name.errors}</div>
          </div>

          {/* 性別 (Select) */}
          <div>
            <Select name={fields.gender.name} defaultValue={fields.gender.initialValue}>
              <SelectTrigger id={fields.gender.id}>
                <SelectValue />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="male">男性</SelectItem>
                <SelectItem value="female">女性</SelectItem>
                <SelectItem value="non-binary">その他</SelectItem>
              </SelectContent>
            </Select>
            <div id={fields.gender.errorId} className="text-destructive text-xs">{fields.gender.errors}</div>
          </div>

          {/* 郵便番号 (カスタムInput + hidden) */}
          <div>
            {/* 実際の値はhidden inputで管理 */}
            <input {...getInputProps(fields.zip, { type: 'hidden', valueType: 'string' })} />
            {/* 見た目と入力用のカスタムコンポーネント */}
            <ZipInput
              defaultValue={fields.zip.value} // Conformからの現在の値
              onChange={(value) => form.update({ name: fields.zip.name, value })} // 変更時にConformの状態を更新
              aria-describedby={fields.zip.errorId} // エラーメッセージとの関連付け
            />
            <div id={fields.zip.errorId} className="text-destructive text-xs">{fields.zip.errors}</div>
          </div>

          {/* 電話番号 */}
          <div>
            <Input {...getInputProps(fields.tel, { type: 'tel', valueType: 'string' })} />
            <div id={fields.tel.errorId} className="text-destructive text-xs">{fields.tel.errors}</div>
          </div>

          {/* Email */}
          <div>
            <Input {...getInputProps(fields.email, { type: 'email', valueType: 'string' })} />
            <div id={fields.email.errorId} className="text-destructive text-xs">{fields.email.errors}</div>
          </div>

          {/* 削除ボタン (親から渡されたもの) */}
          <div className="flex items-center justify-center">
            {removeButton}
          </div>
        </>
      )}
    </SortableItem>
  )
}
```

* **`useField(name, { formId })`**: 親から渡された `name` (例: `'teams[0].members[1]'`) を使って、このメンバーに対応する Conform のフィールドメタ情報 (`meta`) とフォームインスタンス (`form`) を取得します。
* **`meta.getFieldset()`**: メンバー内の各フィールド (`id`, `name`, `gender` など) のプロパティ（`name`, `value`, `initialValue`, `errorId`, `errors`, `id` など）にアクセスするためのオブジェクトを取得します。
* **`getInputProps(fields.xxx, { ... })`**: 各標準HTML入力要素 (`<input>`, `<select>`) に必要な属性（`name`, `id`, `value`/`defaultValue`, `aria-invalid`, `aria-describedby` など）を Conform から取得し、展開します。これにより、入力要素が Conform の状態と自動的に同期します。`valueType` を指定することで、Conformが値を正しく扱うのを助けます。
* **カスタム入力コンポーネントとの連携 (`ZipInput`)**: 標準でない入力コンポーネントを使う場合は、`getInputProps` から必要な情報（`name`, `value`, `errorId` など）を取得し、コンポーネントの `props` に渡します。値が変更された際には `onChange` コールバック内で `form.update()` を呼び出し、Conform の状態を手動で更新する必要があります。隠し `input` (`type="hidden"`) を併用して、実際のフォーム送信値を管理するのも一般的なパターンです。
* **`SortableItem` でラップ**: コンポーネント全体を `SortableItem` でラップし、`id` prop に `meta.id` (ConformのフィールドID) を渡します。これが dnd kit がこの要素を識別するためのキーとなります。
* **ドラッグハンドルの実装**: `SortableItem` は子要素として関数を受け取り、その関数にドラッグハンドルに必要な `attributes` と `listeners` を引数として渡します。これらをドラッグ操作の起点としたい要素（ここでは `DragHandleDots2Icon`）に展開 (`{...attributes} {...listeners}`) することで、その要素を掴んでドラッグできるようになります。CSSでホバー時に表示を変化させるなど、UI的な工夫も加えています。

### 5. ソート可能アイテム (`SortableItem.tsx`)

これは dnd kit の `useSortable` フックをラップし、リストアイテムをドラッグ可能にするためのスタイルや属性を提供する、再利用可能な汎用コンポーネントです。

```typescript
// app/routes/demo+/conform.nested-array/components/sortable-item.tsx
import type { DraggableAttributes, DraggableSyntheticListeners } from '@dnd-kit/core'
import { useSortable } from '@dnd-kit/sortable'
import { CSS } from '@dnd-kit/utilities' // スタイルユーティリティ
import type React from 'react'

// SortableItemが受け取るPropsの型定義
interface SortableItemProps extends Omit<React.ComponentProps<'div'>, 'id' | 'children'> {
  id: string // ★ 要素を一意に識別するID (ConformのフィールドIDを受け取る)
  children: // 子要素。Reactノードか、属性とリスナーを受け取る関数
    | React.ReactNode
    | ((props: {
        attributes: DraggableAttributes // ドラッグハンドル用属性
        listeners: DraggableSyntheticListeners // ドラッグハンドル用リスナー
      }) => React.ReactNode)
}

// SortableItemコンポーネント本体
export const SortableItem = ({ id, children, ...rest }: SortableItemProps) => {
  // dnd kit の useSortable フックを使用
  const {
    attributes, // ドラッグハンドルに適用する属性
    listeners, // ドラッグハンドルに適用するイベントリスナー
    setNodeRef, // この要素のDOMノードをdnd kitに登録するためのref
    transform, // ドラッグ中の座標移動情報
    transition, // ドラッグ中のアニメーション情報
    isDragging, // ドラッグ中かどうかのフラグ (必要ならスタイル変更などに使う)
  } = useSortable({ id }) // ★ 受け取った一意なIDで初期化

  // ドラッグ中の要素移動を滑らかに見せるためのスタイル
  const style: React.CSSProperties = {
    // transform をCSS文字列に変換 (scaleYを常に1にすることで潰れないように)
    transform: CSS.Transform.toString(transform ? { ...transform, scaleY: 1 } : null),
    transition, // アニメーションのtransition
    opacity: isDragging ? 0.8 : 1, // ドラッグ中は少し透過させる (オプション)
    zIndex: isDragging ? 10 : undefined, // ドラッグ中は手前に表示 (オプション)
  };

  // childrenが関数の場合は、attributesとlistenersを渡して実行し、その結果を内容とする
  // そうでなければ、childrenをそのまま内容とする
  const content = typeof children === 'function' ? children({ attributes, listeners }) : children

  return (
    // ルート要素に dnd kit から取得した ref と style を適用
    // {...rest} で他のdiv属性 (classNameなど) も渡せるようにする
    <div ref={setNodeRef} style={style} {...rest}>
      {content} {/* 描画内容 */}
    </div>
  )
}
```

* **`useSortable({ id })`**: dnd kit のコアとなるフックの一つです。親の `SortableContext` と連携し、渡された `id` に基づいて、この要素を並び替え可能にするために必要な情報（`attributes`, `listeners`, `setNodeRef`, `transform`, `transition`, `isDragging` など）を返します。`id` は `MemberListItem` から渡された Conform のフィールドIDです。
* **`setNodeRef`**: この `div` 要素のDOMノードへの参照を dnd kit に伝えるために `ref` propに設定します。これにより、dnd kit はこの要素の位置やサイズを把握し、ドラッグ操作を正しく処理できます。
* **`style`**: `transform` と `transition` は、dnd kit が計算した要素の移動量とアニメーション情報です。`CSS.Transform.toString()` ユーティリティを使って CSS の `transform` プロパティ値に変換し、`style` propに適用します。これにより、要素がドラッグされたり、他の要素の移動によって押し出されたりする際に、滑らかなアニメーションで表示位置が変わります。`isDragging` フラグを使って、ドラッグ中の要素の見た目を少し変える（例: 半透明にする、z-indexを上げる）ことも可能です。
* **`attributes` と `listeners` の提供**: `useSortable` から返される `attributes` (例: `role`, `aria-pressed`, `tabIndex`) と `listeners` (例: `onPointerDown`) は、ユーザーが実際にドラッグ操作を開始する要素（通常はドラッグハンドル）に適用する必要があります。`SortableItem` では、これらの値を `children` propが関数である場合に引数として渡し、呼び出し元 (`MemberListItem`) が適切な要素に設定できるようにしています。これにより、コンポーネントのどの部分をドラッグハンドルにするかを柔軟に決定できます。

## まとめ

この記事では、Remix (React Router v7), Conform, dnd kit, Zod を組み合わせて、動的に増減・並び替え可能なネスト配列フォームを実装する方法を、具体的なコードと共に詳細に解説しました。

**実装の重要なポイントをおさらいしましょう:**

1. **データフローの理解:** Remix/React Routerの `loader`/`action` モデル、Conformのフォーム状態管理、dnd kitのUI操作がどのように連携するかを把握することが重要です。
2. **スキーマ駆動開発:** Zodでデータ構造とバリデーションルールを最初に定義することで、サーバー・クライアント間での一貫性と型安全性を確保します。
3. **Conform のフル活用:** `useForm`, `getFieldList`, `getInputProps`, `insert`, `remove`, `reorder` といったConformのAPIを適切に使い分けることで、複雑なフォームの状態管理と操作を宣言的に記述できます。
4. **dnd kit と Conform の連携:**
    * `DndContext`, `SortableContext`, `useSortable` を設定し、ドラッグ＆ドロップ機能を実装します。
    * 並び替え対象要素の識別には、**Conform が管理する安定したフィールドID (`meta.id`)** を `SortableContext` の `items` や `useSortable` の `id` に一貫して使用します。
    * `onDragEnd` で `form.reorder()` を呼び出し、UI上の並び替え操作を Conform の内部状態に正確に反映させます。
5. **`key` prop の最重要性:** Reactで動的リスト、特に並び替えを伴うリストをレンダリングする際は、**各リストアイテムの `key` prop に、そのデータ自体に紐づく、安定した（並び替えによって変化しない）一意な識別子**（今回の例では `memberFields.id.value ?? teamMember.key`）を指定することが、UIの表示崩れや内部状態との不整合を防ぐ上で絶対に不可欠です。配列のインデックスを `key` に使用してはいけません。

この組み合わせは初期設定が少し複雑に感じるかもしれませんが、一度パターンを理解してしまえば、非常にリッチでユーザーフレンドリーなフォーム UI を、状態管理の破綻を心配することなく効率的に構築できるようになります。チーム・メンバー管理だけでなく、タスクリスト、階層的な設定項目、商品オプションの並び替えなど、様々な応用が考えられます。

**ぜひ、実際のデモを触って、ソースコードを読み解いてみてください:**

* **デモ:** [https://www.techtalk.jp/demo/conform/nested-array](https://www.techtalk.jp/demo/conform/nested-array)
* **ソースコード:** [https://github.com/techtalkjp/techtalk.jp/blob/main/app/routes/demo+/conform.nested-array/route.tsx](https://github.com/techtalkjp/techtalk.jp/blob/main/app/routes/demo+/conform.nested-array/route.tsx)

## 補足

* **UIライブラリ:** このデモでは UI コンポーネントに [Shadcn UI](https://ui.shadcn.com/) を使用していますが、基本的な考え方（Conform, dnd kit, `key` prop の扱い）は、Material UI, Chakra UI, Mantine UI など、他のUIライブラリを使用する場合でも同様に適用できます。
* **さらなる改善点:** 実際のアプリケーションでは、より詳細なエラーハンドリング、ローディング状態の表示（特に非同期処理中）、アクセシビリティの向上（キーボード操作での並び替えなど）、楽観的更新 (Optimistic UI) の導入などを検討すると、さらにユーザー体験が向上します。
* **チーム自体の並び替え:** このデモではチームの並び替えはメニューの「Move Up/Down」ボタンで行っていますが、もしチームカード自体をドラッグ＆ドロップで並び替えたい場合は、`route.tsx` の `teams.map` をレンダリングしている箇所で、`TeamCard` をラップする形で `DndContext` や `SortableContext` を設定することになります。考え方はメンバーリストの並び替えと同様です。

この記事が、Conform と dnd kit を使ったインタラクティブで複雑なフォーム実装に挑戦する皆さんの一助となれば幸いです。
