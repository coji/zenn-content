---
title: "Conformとdnd kitで作る！ソート可能なネスト配列動的フォームの実装ガイド (React Router v7"
emoji: "📝"
type: "tech"
topics: ["conform", "dndkit", "zod", "react-router", "shadcnui"]
published: false
---

## はじめに

Webアプリケーション開発において、複雑なフォームの実装はしばしば頭痛の種となります。特に、ユーザーが自由に行を追加・削除したり、順番を入れ替えたりできる「動的なリスト」をフォーム内に含める場合、状態管理やバリデーション、UIの整合性を保つのが難しくなります。ネストされた構造（リストの中にさらにリストがある）になると、その複雑さはさらに増します。

この記事では、React Router v7 を利用したWebアプリケーションを例に取り、以下の技術スタックを用いて、**動的に増減可能で、かつドラッグ＆ドロップで並び替えも可能なネストされたオブジェクト配列**を扱うフォームの実装方法を詳細に解説します。

* **Conform**: 宣言的なフォームバリデーションと状態管理ライブラリ
* **dnd kit**: 高機能で柔軟なドラッグ＆ドロップライブラリ
* **Zod**: 型安全なスキーマ定義・バリデーションライブラリ
* **React Router v7**: データローディング、ミューテーション、ルーティング機能を提供

具体的には、「チーム」を複数管理し、各チーム内に所属する「メンバー」をリスト形式で表示・編集できるフォームを作成します。チーム自体も追加・削除・上下移動が可能で、さらに各チーム内のメンバーリストも同様に追加・削除・ドラッグ＆ドロップでの並び替えができる、という実践的な要件に挑戦します。

**実際に動作するデモはこちらで確認できます:**
▶ [**デモを見る**](https://www.techtalk.jp/demo/conform/nested-array)

![完成形フォームの動作イメージ](/images/conform-dndkit-nested-sortable-form/conform-dndkit-form.gif)

Conformで動的に増減するネスト配列を扱い、さらにdnd kitでその要素を並び替える、という組み合わせの実装例はまだ少ないため、同様の実装を目指す方の具体的なガイドとなることを目指します。特に、Reactの`key` propの適切な設定が、この種のUIを実現する上でいかに重要かについても深掘りします。

**対象読者:**

* ReactおよびReact Router v7 (またはそれを利用するRemixなどのフレームワーク) の基本的な知識がある方
* Conform, Zod, dnd kitを使ったことがある、または興味がある方
* 動的なリストを含む複雑なフォームの実装方法を探している方

## なぜこの組み合わせが難しいのか？

Conformはフォームの状態管理、特にZodのようなスキーマと連携したバリデーションやエラーハンドリングを、非常にシンプルかつ宣言的に記述できる強力なライブラリです。`useForm`, `useField`, `getFieldList` といったフックやユーティリティを使えば、リスト形式の入力も比較的容易に扱えます。

一方、dnd kitはアクセシビリティやパフォーマンスにも配慮された、柔軟性の高いドラッグ＆ドロップ機能を提供します。`useSortable` フックなどを使えば、リストの並び替えUIも効率的に実装できます。

しかし、これらを組み合わせ、特に「ネストされた」動的リスト（チームリストの中にメンバーリストがある構造）で並び替えを実現しようとすると、いくつかの特有の課題が生じます。

1. **状態の同期:** Conformが管理するフォームの状態（配列の順序や各要素の値、エラー状態など）と、dnd kitによるUI上の並び替え操作（DOM要素の物理的な移動）を正確に同期させる必要があります。ユーザーが要素をドロップした瞬間に、Conformの内部状態も更新され、その結果がUIに正しく反映されなければなりません。
2. **Reactのレンダリングと`key` prop:** Reactがリスト要素を効率的に、かつ正確に更新するためには、各要素に**安定した一意な`key` prop**を割り当てる必要があります。動的なリスト、特に並び替えが発生する場合、この`key`の管理が非常に重要になります。不適切な`key`（例えば配列の`index`）を使うと、並び替え後に表示が崩れたり、入力内容が意図しない要素に紐づいてしまう（例えば、Aさんの行を移動したら中身がBさんのものになってしまう）といった問題が発生します。
3. **Conformの内部状態との整合性:** Conformは各フォームフィールドの状態（値、エラー、ダーティ状態、バリデーションメッセージIDなど）を内部で厳密に管理しています。ドラッグ＆ドロップによるDOM要素の並び替え後も、これらの内部状態と対応するDOM要素が正しく紐付き続けている必要があります。`key` propの問題とも密接に関連します。

この記事では、これらの課題をReact Router v7のアーキテクチャの上で、Conformとdnd kitを用いてどのように解決していくかを、具体的なコードと共にステップバイステップで解説します。

## 実装の全体像

まず、今回のデモアプリケーションのファイル構成と、主要なコンポーネントの役割を見てみましょう。ファイル構成はRemixの規約に従っていますが、React Router v7 を利用するアプリケーションであれば同様の構成が可能です。

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
                ├── route.tsx              # ルート定義ファイル (Loader, Action, Component) ★最重要
                └── schema.ts              # Zodによるフォームスキーマ定義 ★重要
```

**ソースコードはこちらで確認できます:**
▶ [**GitHubでソースを見る**](https://github.com/techtalkjp/techtalk.jp/blob/main/app/routes/demo+/conform.nested-array/route.tsx)

**データの流れと処理の概要:**

1. **`loader` (`route.tsx`)**: ユーザーが `/demo/conform/nested-array` にアクセスすると、まずサーバーサイドで `loader` 関数が実行されます（React Router v7 のデータローディング機能）。ここでは初期表示用のダミーのチーム・メンバーデータを生成し、JSONとしてクライアント（ブラウザ）に渡します。
2. **`schema.ts`**: Zodを使って、フォームで扱うデータの構造（チーム配列、各チームが持つメンバー配列）と、それぞれのフィールドに対するバリデーションルールを定義します。
3. **Component (`route.tsx`)**: ブラウザ側でページのUIをレンダリングするメインコンポーネントです。`useLoaderData` で初期データを、`useActionData` でフォーム送信結果を取得し、`useForm` (Conform) でフォーム状態を管理します。`getFieldList` を使ってチームリストを描画します。
4. **`TeamCard.tsx`**: 各チームのUIを担当し、チーム名入力やメンバーリストを表示します。内部で`useField` (Conform) を使い、メンバーリスト (`getFieldList`) を `MemberListItem` として描画します。dnd kit (`DndContext`, `SortableContext`) を設定し、メンバーの並び替え機能を実装します。
5. **`MemberListItem.tsx`**: 各メンバー行のUIを担当します。`useField` (Conform) でメンバーのフィールド状態を取得し、`getInputProps` で入力要素にバインドします。`SortableItem` でラップされ、ドラッグハンドルを提供します。
6. **`SortableItem.tsx`**: `useSortable` (dnd kit) を使った汎用ラッパーで、要素をドラッグ可能にするスタイルや属性を提供します。
7. **`action` (`route.tsx`)**: フォーム送信時にサーバーサイドで実行されます（React Router v7 のデータミューテーション機能）。`parseWithZod` (Conform) でバリデーションを行い、結果をクライアントに返します。

## コアとなる実装の解説

それでは、この複雑なフォームを実現するための核心部分となるコードを、ファイルごとに詳しく見ていきましょう。

### 1. スキーマ定義 (`schema.ts`)

Zodを使って、フォームで扱うデータの構造とルールを定義します。

```typescript
// app/routes/demo+/conform.nested-array/schema.ts
import { z } from 'zod'

const memberSchema = z.object({
  id: z.string().optional(),
  name: z.string({ required_error: '必須' }),
  gender: z.enum(['male', 'female', 'non-binary'], {
    required_error: '必須',
    message: '性別を選択してください',
  }),
  zip: z
    .string({ required_error: '必須' })
    .regex(/^\d{3}-\d{4}$/, { message: '000-0000形式' }),
  tel: z
    .string({ required_error: '必須' })
    .regex(/^\d{3}-\d{4}-\d{4}$/, { message: '000-0000-0000形式' }),
  email: z
    .string({ required_error: '必須' })
    .email({ message: 'メールアドレス' }),
})

export const teamSchema = z.object({
  id: z.string().optional(),
  name: z.string({ required_error: '必須' }),
  members: z
    .array(memberSchema)
    .min(1, { message: 'メンバーを追加してください' }),
})

export const formSchema = z.object({
  teams: z.array(teamSchema).min(1, { message: 'チームを追加してください' }),
})

export type MemberSchema = z.infer<typeof memberSchema>
export type TeamSchema = z.infer<typeof teamSchema>
export type FormSchema = z.infer<typeof formSchema>
```

*ネストされた `memberSchema` と `teamSchema` を定義し、それぞれにバリデーションルールを設定しています。`id` は `optional` とし、新規・既存の両方を扱えるようにしています。*

### 2. ルート定義ファイル (`route.tsx`)

React Router v7 の `loader`, `action`, およびUIコンポーネントを定義します。

```typescript
// app/routes/demo+/conform.nested-array/route.tsx
import { FormProvider, getFormProps, useForm } from '@conform-to/react'
import { parseWithZod } from '@conform-to/zod'
import { EllipsisVerticalIcon } from 'lucide-react'
import { Form } from 'react-router'
import { dataWithSuccess } from 'remix-toast'
import {
  Button,
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
  Stack,
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from '~/components/ui'
import type { Route } from './+types/route'
import { TeamCard } from './components'
import {
  fakeEmail,
  fakeGender,
  fakeId,
  fakeName,
  fakeTeamName,
  fakeTel,
  fakeZip,
} from './faker.server'
import { formSchema } from './schema'

export const loader = ({ request }: Route.LoaderArgs) => {
  // ブラウザバンドルが巨大になってしまうので、faker はサーバサイドでのみ使用
  const fakeMembers = Array.from({ length: 30 }, () => ({
    id: fakeId(),
    name: fakeName(),
    gender: fakeGender(),
    zip: fakeZip(),
    tel: fakeTel(),
    email: fakeEmail(),
  }))

  const teams = [
    {
      id: fakeId(),
      name: fakeTeamName(),
      members: fakeMembers.slice(0, 5),
    },
    {
      id: fakeId(),
      name: fakeTeamName(),
      members: fakeMembers.slice(5, 10),
    },
  ]
  return { teams, fakeMembers }
}

export const action = async ({ request }: Route.ActionArgs) => {
  const submission = parseWithZod(await request.formData(), {
    schema: formSchema,
  })
  if (submission.status !== 'success') {
    return { lastResult: submission.reply(), result: null }
  }

  return dataWithSuccess(
    {
      lastResult: submission.reply({ resetForm: true }),
      result: submission.value.teams.map((team, teamIndex) => ({
        id: team.id ?? fakeId(),
        ...team,
        members: team.members.map((member, memberIndex) => ({
          ...member,
          id: member.id ?? fakeId(),
        })),
      })),
    },
    {
      message: '登録しました',
      description: '登録されたデータを確認してください',
    },
  )
}

export default function ConformNestedArrayDemo({
  loaderData: { teams: defaultTeams, fakeMembers },
  actionData,
}: Route.ComponentProps) {
  const [form, fields] = useForm({
    lastResult: actionData?.lastResult,
    defaultValue: { teams: defaultTeams },
    onValidate: ({ formData }) =>
      parseWithZod(formData, { schema: formSchema }),
  })

  const teams = fields.teams.getFieldList()

  return (
    <Stack>
      <FormProvider context={form.context}>
        <Form method="POST" {...getFormProps(form)}>
          <Stack>
            {teams.map((team, index) => (
              <TeamCard
                key={team.key}
                formId={form.id}
                name={team.name}
                menu={
                  <DropdownMenu>
                    <DropdownMenuTrigger asChild>
                      <Button type="button" variant="ghost" size="icon">
                        <EllipsisVerticalIcon />
                      </Button>
                    </DropdownMenuTrigger>
                    <DropdownMenuContent>
                      <DropdownMenuItem
                        disabled={index === 0}
                        onClick={() => {
                          form.reorder({
                            name: fields.teams.name,
                            from: index,
                            to: index - 1,
                          })
                        }}
                      >
                        Move Up
                      </DropdownMenuItem>
                      <DropdownMenuItem
                        disabled={index === teams.length - 1}
                        onClick={() => {
                          form.reorder({
                            name: fields.teams.name,
                            from: index,
                            to: index + 1,
                          })
                        }}
                      >
                        Move Down
                      </DropdownMenuItem>
                      <DropdownMenuItem
                        onClick={(e) => {
                          form.remove({ name: fields.teams.name, index })
                        }}
                        className="text-destructive"
                      >
                        Remove
                      </DropdownMenuItem>
                    </DropdownMenuContent>
                  </DropdownMenu>
                }
              />
            ))}

            <Button
              variant="outline"
              {...form.insert.getButtonProps({ name: fields.teams.name })}
            >
              Add Team
            </Button>

            <Button type="submit">Submit</Button>
          </Stack>
        </Form>
      </FormProvider>


      {/* 登録結果テーブル */}
      {actionData?.result && (
        <Stack className="text-xs">
          <div>登録されました。</div>
          <Table>{/* TableHeader, TableBody ... */}</Table>
        </Stack>
      )}
    </Stack>
  )
}
```

React Router v7 の `loader` と `action` でデータの読み書きを行い、UIコンポーネント側では `useLoaderData`, `useActionData` でデータを取得します。Conform の `useForm` で状態を初期化し、`getFieldList` でチーム配列を取得して `TeamCard` をマッピングします。チームの追加・削除・並び替えは Conform の `insert`, `remove`, `reorder` API を使用します。こちらのリストの `key` には Conform が提供する `team.key` を使っています。

### 3. チームカード (`TeamCard.tsx`)

各チームの表示と、内部のメンバーリストの並び替えを担当します。

```typescript
// app/routes/demo+/conform.nested-array/components/team-card.tsx
import { getInputProps, useField, type FieldName } from '@conform-to/react'
import { DndContext, PointerSensor, useSensor, useSensors, type DragEndEvent, closestCenter } from '@dnd-kit/core'
import { restrictToVerticalAxis, restrictToParentElement } from '@dnd-kit/modifiers'
import { SortableContext, verticalListSortingStrategy } from '@dnd-kit/sortable'
import { TrashIcon } from 'lucide-react'
import { useId } from 'react'
import { Badge, Button, Card, CardContent, CardHeader, CardTitle, HStack, Input, Label, Stack } from '~/components/ui'
import { MemberList, MemberListHeader, MemberListItem } from './'
import type { FormSchema, TeamSchema } from '../schema'

interface TeamCardProps extends React.ComponentProps<typeof Card> {
  formId: string
  name: FieldName<TeamSchema, FormSchema>
  menu: React.ReactNode
}

export const TeamCard = ({ formId, name, menu, className }: TeamCardProps) => {
  const [field, form] = useField(name, { formId })
  const teamFields = field.getFieldset()
  const teamMembers = teamFields.members.getFieldList()

  const dndContextId = useId()
  const sensors = useSensors(useSensor(PointerSensor))

  // ドラッグ終了時の処理
  const handleDragEnd = (event: DragEndEvent) => {
    const { active, over } = event
    if (over && active.id !== over.id) {
      // ConformのフィールドIDを使ってインデックスを検索
      const from = teamMembers.findIndex((m) => m.id === active.id)
      const to = teamMembers.findIndex((m) => m.id === over.id)
      if (from === -1 || to === -1) return
      // Conformのreorder APIで並び替え
      form.reorder({ name: teamFields.members.name, from, to })
    }
  }

  return (
    <Card className={className}>
      <CardHeader>{/* チームタイトルとメニュー */}</CardHeader>
      <CardContent>
        <Stack>
          <input {...getInputProps(teamFields.id, { type: 'hidden' })} />
          {/* チーム名入力 */}
          <Stack gap="sm">
            <HStack>
              <Label htmlFor={teamFields.name.id}>チーム名</Label>
              <Input {...getInputProps(teamFields.name, { type: 'text' })} />
            </HStack>
            <div
              id={teamFields.name.errorId}
              className="text-destructive text-xs"
            >
              {teamFields.name.errors}
            </div>
          </Stack>

          {/* メンバーリストとD&D設定 */}
          <MemberList>
            <MemberListHeader />
            <DndContext
              id={dndContextId}
              sensors={sensors}
              collisionDetection={closestCenter}
              modifiers={[restrictToVerticalAxis, restrictToParentElement]}
              onDragEnd={handleDragEnd}
            >
              <SortableContext
                items={teamMembers.map((m) => m.id)}
                strategy={verticalListSortingStrategy}
              >
                {/* メンバーリストの描画 */}
                {teamMembers.map((teamMember, index) => {
                  const memberFields = teamMember.getFieldset();
                  return (
                    <MemberListItem
                      key={memberFields.id.value}
                      formId={form.id}
                      name={teamMember.name}
                      className="group"
                      removeButton={/* 削除ボタン */}
                        <Button
                          variant="ghost"
                          size="sm"
                          className="opacity-0 group-hover:opacity-100"
                          {...form.remove.getButtonProps({ name: teamFields.members.name, index })}
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
          <div
            id={teamFields.members.errorId}
            className="text-destructive text-xs">
            {teamFields.members.errors}
          </div>
          {/* メンバー追加ボタン */}
          <Button
            variant="outline"
            size="sm"
            {...form.insert.getButtonProps({ name: teamFields.members.name })}
          >
            Add a member
          </Button>
        </Stack>
      </CardContent>
    </Card>
  )
}
```

`useField` でチームの情報を取得し、`getFieldList` でメンバーリストを取得します。dnd kit の `DndContext` と `SortableContext` を使い、メンバーリストを並び替え可能にします。`SortableContext` の `items` には Conform のフィールドID (`m.id`) の配列を渡します。`handleDragEnd` では、イベントから取得したID (`active.id`, `over.id`) を使ってリスト内のインデックスを特定し、Conform の `reorder` API を呼び出して状態を更新します。**そして最も重要なのが、`MemberListItem` の `key` prop です。ここには、メンバーデータに紐づく安定した一意な識別子（DBのIDである `memberFields.id.value`）を設定します。これにより、並び替え後もReactとConformの状態が正しく同期され、UIの不整合を防ぎます。配列のインデックスを `key` に使ってはいけません。

### 4. メンバーリストアイテム (`MemberListItem.tsx`)

各メンバー行のUIと入力フィールドを定義します。

```typescript
// app/routes/demo+/conform.nested-array/components/member-list-item.tsx
import { getInputProps, useField, type FieldName } from '@conform-to/react'
import { DragHandleDots2Icon } from '@radix-ui/react-icons'
import type React from 'react'
import { Input, Select, SelectContent, SelectItem, SelectTrigger, SelectValue, Tooltip, TooltipContent, TooltipTrigger } from '~/components/ui'
import { cn } from '~/libs/utils'
import { ZipInput } from '~/routes/demo+/resources.zip-input/route'
import type { FormSchema, MemberSchema } from '../schema'
import { SortableItem } from './sortable-item'

interface MemberProps extends React.ComponentProps<'div'> {
  formId: string
  name: FieldName<MemberSchema, FormSchema>
  removeButton: React.ReactNode
}
export const MemberListItem = ({ formId, name, removeButton, className }: MemberProps) => {
  const [meta, form] = useField(name, { formId })
  const fields = meta.getFieldset()

  return (
    // SortableItemでラップし、ConformのIDを渡す
    <SortableItem id={meta.id} className={cn('col-span-full grid grid-cols-subgrid', className)}>
      {({ attributes, listeners }) => ( // ドラッグハンドル用の属性とリスナーを受け取る
        <>
          {/* ドラッグハンドル */}
          <DragHandleDots2Icon {...attributes} {...listeners} className="text-muted-foreground cursor-grab self-center rounded-md" />
          {/* 各入力フィールド */}
          <input {...getInputProps(fields.id, { type: 'hidden' })} />
          <div><Input {...getInputProps(fields.name, { type: 'text' })} /><div id={fields.name.errorId}>{fields.name.errors}</div></div>
          <div><Select name={fields.gender.name} defaultValue={fields.gender.initialValue} onValueChange={(v) => form.update({ name: fields.gender.name, value: v })}>{/* SelectTrigger/Content */}</Select><div id={fields.gender.errorId}>{fields.gender.errors}</div></div>
          <div><input {...getInputProps(fields.zip, { type: 'hidden' })} /><ZipInput defaultValue={fields.zip.value} onChange={(v) => form.update({ name: fields.zip.name, value: v })} /><div id={fields.zip.errorId}>{fields.zip.errors}</div></div>
          <div><Input {...getInputProps(fields.tel, { type: 'text' })} /><div id={fields.tel.errorId}>{fields.tel.errors}</div></div>
          <div><Input {...getInputProps(fields.email, { type: 'text' })} /><div id={fields.email.errorId}>{fields.email.errors}</div></div>
          {/* 削除ボタン (Tooltip付き) */}
          <div><Tooltip><TooltipTrigger asChild>{removeButton}</TooltipTrigger><TooltipContent>Remove a member</TooltipContent></Tooltip></div>
        </>
      )}
    </SortableItem>
  )
}
```

*`useField` でメンバー個別の情報を取得し、`getFieldset` で各フィールドにアクセスします。`getInputProps` を使って標準入力要素を Conform の状態に接続します。カスタムコンポーネント (`Select`, `ZipInput`) では、値の変更時に `form.update` を呼び出して Conform の状態を更新します。全体を `SortableItem` でラップし、`id` に Conform のフィールドID (`meta.id`) を渡します。ドラッグハンドル要素には `SortableItem` から受け取った `attributes` と `listeners` を展開します。*

### 5. ソート可能アイテム (`SortableItem.tsx`)

dnd kit の `useSortable` を使った汎用的なラッパーコンポーネントです。

```typescript
// app/routes/demo+/conform.nested-array/components/sortable-item.tsx
import type { DraggableAttributes, DraggableSyntheticListeners } from '@dnd-kit/core'
import { useSortable } from '@dnd-kit/sortable'
import { CSS } from '@dnd-kit/utilities'
import type React from 'react'

interface SortableItemProps extends Omit<React.ComponentProps<'div'>, 'id' | 'children'> {
  id: string // 要素の一意なID (ConformのフィールドID)
  children: // 子要素 or 関数
    | React.ReactNode
    | ((props: { attributes: DraggableAttributes, listeners: DraggableSyntheticListeners }) => React.ReactNode)
}

export const SortableItem = ({ id, children, ...rest }: SortableItemProps) => {
  // useSortableフックでD&Dに必要な情報を取得
  const { attributes, listeners, setNodeRef, transform, transition } = useSortable({ id })

  // ドラッグ中のスタイル適用
  const style = {
    transform: CSS.Transform.toString(transform ? { ...transform, scaleY: 1 } : null),
    transition,
  }

  // 子が関数の場合、属性とリスナーを渡して実行
  const content = typeof children === 'function' ? children({ attributes, listeners }) : children

  return (
    // refとstyleを適用したdivでラップ
    <div ref={setNodeRef} style={style} {...rest}>
      {content}
    </div>
  )
}
```

*`useSortable` フックに要素の一意な `id` (ConformのフィールドID) を渡して初期化します。返される `setNodeRef` を要素の `ref` に設定し、`transform` と `transition` を使ってスタイルを適用します。ドラッグハンドル用の `attributes` と `listeners` は、`children` prop が関数であれば引数として渡します。*

## まとめ

この記事では、React Router v7 環境で Conform と dnd kit を組み合わせ、動的に増減・並び替え可能なネスト配列フォームを実装する方法を解説しました。

**実装の重要なポイント:**

1. **React Router v7 と Conform の連携:** `loader`/`action` でデータを扱い、`useForm` で状態を管理します。
2. **スキーマ定義:** Zod でデータ構造とバリデーションルールを定義します。
3. **Conform API の活用:** `getFieldList`, `getInputProps`, `insert`, `remove`, `reorder` を適切に使います。
4. **dnd kit との連携:** `DndContext`, `SortableContext`, `useSortable` を使い、並び替え対象のIDには Conform のフィールドID (`meta.id`) を使用します。`onDragEnd` で `form.reorder` を呼び出します。
5. **`key` prop の重要性:** **リストアイテムの `key` には、データに紐づく安定した一意な識別子 (`field.id.value ?? field.key`) を必ず指定します。** これがUIと状態の整合性を保つ鍵です。

この組み合わせは、リッチでインタラクティブなフォームUIを効率的に構築するための強力なパターンとなり得ます。

**ぜひ、実際のデモを触って、ソースコードを読み解いてみてください:**

* **デモ:** [https://www.techtalk.jp/demo/conform/nested-array](https://www.techtalk.jp/demo/conform/nested-array)
* **ソースコード:** [https://github.com/techtalkjp/techtalk.jp/blob/main/app/routes/demo+/conform.nested-array/route.tsx](https://github.com/techtalkjp/techtalk.jp/blob/main/app/routes/demo+/conform.nested-array/route.tsx)

## 補足

* **UIライブラリ:** デモでは Shadcn UI を使用していますが、基本的な考え方は他のUIライブラリでも同様です。
* **さらなる改善点:** エラーハンドリング、ローディング状態、アクセシビリティ、楽観的更新などを考慮すると、より実践的なアプリケーションになります。
* **チーム自体の並び替え:** チームカード自体をD&Dで並び替える場合は、`route.tsx` の `teams.map` 周辺で同様に dnd kit を設定します。

この記事が、Conform と dnd kit を使った複雑なフォーム実装の一助となれば幸いです。
