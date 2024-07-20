---
title: "shadcn-uiとconformによるTextareaの実装ガイド"
emoji: "📝"
type: "tech"
topics: ["conform", "shadcnui", "zod"]
published: true
---

## 関連記事

このシリーズの他の記事もご覧ください：

- [shadcn-uiとconformによるInputの実装](https://zenn.dev/coji/articles/conform-shadcn-ui-input)
- [shadcn-uiとconformによるTextareaの実装](https://zenn.dev/coji/articles/conform-shadcn-ui-textarea)
- [shadcn-uiとconformによるSelectの実装](https://zenn.dev/coji/articles/conform-shadcn-ui-select)
- [shadcn-uiとconformによるCheckboxの実装](https://zenn.dev/coji/articles/conform-shadcn-ui-checkbox)
- [shadcn-uiとconformによるSwitchの実装](https://zenn.dev/coji/articles/conform-shadcn-ui-switch)
- [shadcn-uiとconformによるRadioGroupの実装](https://zenn.dev/coji/articles/conform-shadcn-ui-radiogroup)

これらの記事を組み合わせることで、shadcn-uiとconformを使用した包括的なフォーム実装の知識を得ることができます。

## 概要

この記事では、shadcn-uiとconformを使用して基本的なTextareaを実装する方法を解説します。長文入力やマルチラインテキストの取り扱いについて、スキーマ定義と実装例を提供します。

## セットアップ

まず、必要なライブラリをインポートします。

```typescript
import { getTextareaProps, useForm } from '@conform-to/react'
import { getZodConstraint, parseWithZod } from '@conform-to/zod'
import { Form } from '@remix-run/react'
import { z } from 'zod'
import { Textarea } from '~/components/ui/textarea'
import { Label } from '~/components/ui/label'
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

以下、基本的なTextareaのスキーマ定義と実装例を示します。

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
  <pre>{JSON.stringify(form.allErrors, null, 2)}</pre>
</div>
```

これにより、フォームの状態を簡単に確認できます。

## デモと実装例

本記事で解説した内容の実際の動作を確認したい場合は、以下のデモページをご覧ください：

[shadcn-uiとconformを使用したフォーム実装デモ](https://www.techtalk.jp/demo/conform/shadcn-ui)

このデモページでは、shadcn-uiとconformを組み合わせた様々なフォーム要素の実装例を見ることができます。各要素の動作を確認でき、実際のユーザー体験を把握するのに役立ちます。

また、デモページからは実装のソースコードにもアクセスできます。ソースコードを参照することで、本記事で説明した実装方法がどのように適用されているかを詳細に確認できます。

デモとソースコードを併せて確認することで、理論と実践の両面から理解を深めることができます。ぜひ、デモページを訪れて、実際の動作とコードの詳細を確認してみてください。

## まとめ

shadcn-uiとconformを組み合わせることで、基本的なTextareaを簡単に実装できます。適切なバリデーションとエラーハンドリングを行うことで、ユーザーフレンドリーな長文入力フォームを作成することができます。

次回は、[Selectの実装について](https://zenn.dev/coji/articles/conform-shadcn-ui-select)詳しく解説します。お楽しみに！
