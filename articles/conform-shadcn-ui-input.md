---
title: "shadcn-uiとconformによるInput要素の包括的実装ガイド"
emoji: "📝"
type: "tech"
topics: ["conform", "shadcnui"]
published: true
---

## 概要

この記事では、shadcn-uiとconformを使用して、様々なタイプのInput要素を実装する方法を詳しく解説します。各Input要素タイプについて、スキーマ定義と実装例を提供します。

## セットアップ

まず、必要なライブラリをインポートします。

```typescript
import { getInputProps, useForm } from '@conform-to/react'
import { getZodConstraint, parseWithZod } from '@conform-to/zod'
import { Form } from '@remix-run/react'
import { z } from 'zod'
import { Input, Label } from '~/components/ui'
```

基本的なフォーム構造は以下のようになります：

```typescript
export default function InputForm() {
  const [form, fields] = useForm({
    onValidate: ({ formData }) => parseWithZod(formData, { schema }),
    constraint: getZodConstraint(schema),
    shouldValidate: 'onBlur',
    shouldRevalidate: 'onInput',
  })

  return (
    <Form {...getFormProps(form)} method="post">
      {/* Input fields will be placed here */}
    </Form>
  )
}
```

## 実装例

以下、各Input要素のスキーマ定義と実装例を示します。

### テキスト入力 (text)

スキーマ定義:

```typescript
const schema = z.object({
  text: z
    .string({ required_error: '必須' })
    .max(100, { message: '100文字以内で入力してください' }),
})
```

実装例:

```typescript
<div>
  <Label htmlFor={fields.text.id}>テキスト入力</Label>
  <Input
    placeholder="テキスト"
    {...getInputProps(fields.text, { type: 'text' })}
    key={fields.text.key}
  />
  <div id={fields.text.errorId} className="text-destructive">
    {fields.text.errors}
  </div>
</div>
```

### メール入力 (email)

スキーマ定義:

```typescript
const schema = z.object({
  email: z
    .string({ required_error: '必須' })
    .email({ message: 'メールアドレスの形式で入力してください' })
    .max(500, { message: '500文字以内で入力してください' }),
})
```

実装例:

```typescript
<div>
  <Label htmlFor={fields.email.id}>メールアドレス</Label>
  <Input
    placeholder="example@example.com"
    {...getInputProps(fields.email, { type: 'email' })}
    key={fields.email.key}
  />
  <div id={fields.email.errorId} className="text-destructive">
    {fields.email.errors}
  </div>
</div>
```

### 検索入力 (search)

スキーマ定義:

```typescript
const schema = z.object({
  search: z
    .string({ required_error: '必須' })
    .max(100, { message: '100文字以内で入力してください' }),
})
```

実装例:

```typescript
<div>
  <Label htmlFor={fields.search.id}>検索</Label>
  <Input
    placeholder="検索キーワード"
    {...getInputProps(fields.search, { type: 'search' })}
    key={fields.search.key}
  />
  <div id={fields.search.errorId} className="text-destructive">
    {fields.search.errors}
  </div>
</div>
```

### URL入力 (url)

スキーマ定義:

```typescript
const schema = z.object({
  url: z
    .string({ required_error: '必須' })
    .url({ message: '有効なURLを入力してください' }),
})
```

実装例:

```typescript
<div>
  <Label htmlFor={fields.url.id}>URL</Label>
  <Input
    placeholder="https://example.com"
    {...getInputProps(fields.url, { type: 'url' })}
    key={fields.url.key}
  />
  <div id={fields.url.errorId} className="text-destructive">
    {fields.url.errors}
  </div>
</div>
```

### 電話番号入力 (tel)

スキーマ定義:

```typescript
const schema = z.object({
  tel: z
    .string({ required_error: '必須' })
    .regex(/^\d{3}-\d{4}-\d{4}$/, { message: '000-0000-0000の形式で入力してください' }),
})
```

実装例:

```typescript
<div>
  <Label htmlFor={fields.tel.id}>電話番号</Label>
  <Input
    placeholder="000-0000-0000"
    {...getInputProps(fields.tel, { type: 'tel' })}
    key={fields.tel.key}
  />
  <div id={fields.tel.errorId} className="text-destructive">
    {fields.tel.errors}
  </div>
</div>
```

### 数値範囲入力 (range)

スキーマ定義:

```typescript
const schema = z.object({
  range: z
    .number({ required_error: '必須' })
    .min(0, { message: '0以上の値を選択してください' })
    .max(100, { message: '100以下の値を選択してください' }),
})
```

実装例:

```typescript
<div>
  <Label htmlFor={fields.range.id}>範囲</Label>
  <Input
    {...getInputProps(fields.range, { type: 'range' })}
    min="0"
    max="100"
    key={fields.range.key}
  />
  <div id={fields.range.errorId} className="text-destructive">
    {fields.range.errors}
  </div>
</div>
```

### 日時入力 (datetime-local)

スキーマ定義:

```typescript
const schema = z.object({
  datetime: z
    .string({ required_error: '必須' })
    .refine((val) => !isNaN(Date.parse(val)), { message: '有効な日時を入力してください' }),
})
```

実装例:

```typescript
<div>
  <Label htmlFor={fields.datetime.id}>日時</Label>
  <Input
    {...getInputProps(fields.datetime, { type: 'datetime-local' })}
    key={fields.datetime.key}
  />
  <div id={fields.datetime.errorId} className="text-destructive">
    {fields.datetime.errors}
  </div>
</div>
```

### 時間入力 (time)

スキーマ定義:

```typescript
const schema = z.object({
  time: z
    .string({ required_error: '必須' })
    .regex(/^([01]\d|2[0-3]):([0-5]\d)$/, { message: '有効な時間を入力してください' }),
})
```

実装例:

```typescript
<div>
  <Label htmlFor={fields.time.id}>時間</Label>
  <Input
    {...getInputProps(fields.time, { type: 'time' })}
    key={fields.time.key}
  />
  <div id={fields.time.errorId} className="text-destructive">
    {fields.time.errors}
  </div>
</div>
```

### 月入力 (month)

スキーマ定義:

```typescript
const schema = z.object({
  month: z
    .string({ required_error: '必須' })
    .regex(/^\d{4}-([0][1-9]|[1][0-2])$/, { message: '有効な月を選択してください' }),
})
```

実装例:

```typescript
<div>
  <Label htmlFor={fields.month.id}>月</Label>
  <Input
    {...getInputProps(fields.month, { type: 'month' })}
    key={fields.month.key}
  />
  <div id={fields.month.errorId} className="text-destructive">
    {fields.month.errors}
  </div>
</div>
```

### 週入力 (week)

スキーマ定義:

```typescript
const schema = z.object({
  week: z
    .string({ required_error: '必須' })
    .regex(/^\d{4}-W(0[1-9]|[1-4][0-9]|5[0-3])$/, { message: '有効な週を選択してください' }),
})
```

実装例:

```typescript
<div>
  <Label htmlFor={fields.week.id}>週</Label>
  <Input
    {...getInputProps(fields.week, { type: 'week' })}
    key={fields.week.key}
  />
  <div id={fields.week.errorId} className="text-destructive">
    {fields.week.errors}
  </div>
</div>
```

### ファイル入力 (file)

スキーマ定義:

```typescript
const schema = z.object({
  file: z
    .instanceof(File)
    .refine((file) => file.size <= 5000000, { message: 'ファイルサイズは5MB以下にしてください' })
    .refine((file) => ['image/jpeg', 'image/png'].includes(file.type), { message: 'JPEGまたはPNG形式のファイルをアップロードしてください' }),
})
```

実装例:

```typescript
<div>
  <Label htmlFor={fields.file.id}>ファイル</Label>
  <Input
    {...getInputProps(fields.file, { type: 'file' })}
    key={fields.file.key}
    accept="image/jpeg, image/png"
  />
  <div id={fields.file.errorId} className="text-destructive">
    {fields.file.errors}
  </div>
</div>
```

### 色選択 (color)

スキーマ定義:

```typescript
const schema = z.object({
  color: z
    .string({ required_error: '必須' })
    .regex(/^#[0-9A-F]{6}$/i, { message: '有効な色を選択してください' }),
})
```

実装例:

```typescript
<div>
  <Label htmlFor={fields.color.id}>色</Label>
  <Input
    {...getInputProps(fields.color, { type: 'color' })}
    key={fields.color.key}
  />
  <div id={fields.color.errorId} className="text-destructive">
    {fields.color.errors}
  </div>
</div>
```

## デバッグとテスト

フォームの値とエラーを確認するためのデバッグセクションを追加することができます：

```typescript
<div>
  <h3>フォームの値</h3>
  <pre>{JSON.stringify(form.value, null, 2)}</pre>
</div>
<div>
  <h3>フォームのエラー</h3>
  <pre>{JSON.stringify(form.errors, null, 2)}</pre>
</div>
```

これにより、フォームの状態を簡単に確認できます。

## まとめ

shadcn-uiとconformを組み合わせることで、様々なタイプのInput要素を美しく機能的に実装できます。適切なバリデーションとエラーハンドリングを行うことで、ユーザーフレンドリーなフォームを作成することができます。

次回は、Textarea要素の実装について詳しく解説します。お楽しみに！
