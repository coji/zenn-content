---
title: "shadcn-uiとconformによるTextarea要素の実装"
emoji: "📝"
type: "tech"
topics: ["conform", "shadcnui"]
published: true
---

## 概要

この記事では、shadcn-uiとconformを使用して基本的なTextarea要素を実装する方法を解説します。長文入力やマルチラインテキストの取り扱いについて、スキーマ定義と実装例を提供します。

## セットアップ

まず、必要なライブラリをインポートします。

```typescript
import { getTextareaProps, useForm } from '@conform-to/react'
import { getZodConstraint, parseWithZod } from '@conform-to/zod'
import { Form } from '@remix-run/react'
import { z } from 'zod'
import { Textarea, Label } from '~/components/ui'
```

基本的なフォーム構造は以下のようになります：

```typescript
export default function TextareaForm() {
  const [form, fields] = useForm({
    onValidate: ({ formData }) => parseWithZod(formData, { schema }),
    constraint: getZodConstraint(schema),
    shouldValidate: 'onBlur',
    shouldRevalidate: 'onInput',
  })

  return (
    <Form {...getFormProps(form)} method="post">
      {/* Textarea field will be placed here */}
    </Form>
  )
}
```

## 実装例

以下、基本的なTextarea要素のスキーマ定義と実装例を示します。

### 基本的なTextarea

スキーマ定義:

```typescript
const schema = z.object({
  basicTextarea: z
    .string({ required_error: '必須' })
    .min(10, { message: '10文字以上入力してください' })
    .max(1000, { message: '1000文字以内で入力してください' }),
})
```

実装例:

```typescript
<div>
  <Label htmlFor={fields.basicTextarea.id}>基本的なTextarea</Label>
  <Textarea
    placeholder="ここに長文を入力してください"
    {...getTextareaProps(fields.basicTextarea)}
    key={fields.basicTextarea.key}
  />
  <div id={fields.basicTextarea.errorId} className="text-destructive">
    {fields.basicTextarea.errors}
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

shadcn-uiとconformを組み合わせることで、基本的なTextarea要素を簡単に実装できます。適切なバリデーションとエラーハンドリングを行うことで、ユーザーフレンドリーな長文入力フォームを作成することができます。

次回は、Select要素の実装について詳しく解説します。お楽しみに！
