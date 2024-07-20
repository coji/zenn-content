---
title: "shadcn-uiとconformによるCheckboxの実装ガイド"
emoji: "📝"
type: "tech"
topics: ["conform", "shadcnui", "zod"]
published: true
---

## 概要

この記事では、shadcn-uiとconformを使用して基本的なCheckboxを実装する方法を解説します。通常の実装方法と、カスタムhelper関数を使用した実装方法の両方を紹介します。

## セットアップ

まず、必要なライブラリをインポートします。

```typescript
import { useForm } from '@conform-to/react'
import { getZodConstraint, parseWithZod } from '@conform-to/zod'
import { Form } from '@remix-run/react'
import { z } from 'zod'
import { Checkbox } from '~/components/ui/checkbox'
import { Label } from '~/components/ui/label'
import { getCheckboxProps } from './helper'
```

基本的なフォーム構造は以下のようになります：

```typescript
export default function CheckboxForm() {
  const [form, fields] = useForm({
    onValidate: ({ formData }) => parseWithZod(formData, { schema }),
    constraint: getZodConstraint(schema),
    shouldValidate: 'onBlur',
    shouldRevalidate: 'onInput',
  })

  return (
    <Form {...getFormProps(form)} method="post">
      {/* Checkbox fields will be placed here */}
    </Form>
  )
}
```

## スキーマ定義

```typescript
const schema = z.object({
  agreeTerms: z.boolean({
    required_error: '利用規約に同意してください',
    invalid_type_error: '無効な選択です',
  }).refine((val) => val === true, {
    message: '利用規約に同意する必要があります',
  }),
})
```

## 実装例

### 1. 基本的なCheckbox実装

```typescript
<div className="flex items-center space-x-2">
  <Checkbox
    id={fields.agreeTerms.id}
    name={fields.agreeTerms.name}
    required={fields.agreeTerms.required}
    defaultChecked={fields.agreeTerms.initialValue === 'on'}
    aria-invalid={!fields.agreeTerms.valid || undefined}
    aria-describedby={!fields.agreeTerms.valid ? fields.agreeTerms.errorId : undefined}
    onCheckedChange={(checked) => {
      form.update({
        name: fields.agreeTerms.name,
        value: checked ? 'on' : '',
      })
    }}
  />
  <Label
    htmlFor={fields.agreeTerms.id}
    className="text-sm font-medium leading-none peer-disabled:cursor-not-allowed peer-disabled:opacity-70"
  >
    利用規約に同意する
  </Label>
</div>
<div id={fields.agreeTerms.errorId} className="text-destructive">
  {fields.agreeTerms.errors}
</div>
```

### 2. Helper関数を使用したCheckbox実装

helper.tsファイルに定義されたgetCheckboxProps関数を使用して、より簡潔に実装できます。

```typescript
<div className="flex items-center space-x-2">
  <Checkbox
    {...getCheckboxProps(fields.agreeTerms)}
    onCheckedChange={(checked) => {
      form.update({
        name: fields.agreeTerms.name,
        value: checked ? 'on' : '',
      })
    }}
  />
  <Label
    htmlFor={fields.agreeTerms.id}
    className="text-sm font-medium leading-none peer-disabled:cursor-not-allowed peer-disabled:opacity-70"
  >
    利用規約に同意する
  </Label>
</div>
<div id={fields.agreeTerms.errorId} className="text-destructive">
  {fields.agreeTerms.errors}
</div>
```

## Helper関数の説明

helper.tsファイルには、Checkboxの実装を簡略化するための`getCheckboxProps`関数が定義されています：

```ts:helper.ts
import type { FieldMetadata } from '@conform-to/react';

/**
 * Cleanup `undefined` from the result.
 * To minimize conflicts when merging with user defined props
 */
function simplify<Props>(props: Props): Props {
  for (const key in props) {
    if (props[key] === undefined) {
      delete props[key];
    }
  }
  return props;
}

export const getCheckboxProps = <Schema>(
  metadata: FieldMetadata<Schema>,
  options:
    | {
        ariaAttributes?: true;
        ariaInvalid?: 'errors' | 'allErrors';
        ariaDescribedBy?: string;
        value?: boolean;
      }
    | {
        ariaAttributes: false;
        value?: boolean;
      } = {
    ariaAttributes: true,
  }
) => {
  const props: {
    id: string;
    key?: string;
    required?: boolean;
    name: string;
    defaultChecked?: boolean;
    'aria-invalid'?: boolean;
    'aria-describedby'?: string;
  } = {
    id: metadata.id,
    key: metadata.key,
    required: metadata.required,
    name: metadata.name,
  };

  if (typeof options.value === 'undefined' || options.value) {
    props.defaultChecked = metadata.initialValue === 'on';
  }

  if (options.ariaAttributes) {
    const invalid =
      options.ariaInvalid === 'allErrors'
        ? !metadata.valid
        : typeof metadata.errors !== 'undefined';
    const ariaDescribedBy = options.ariaDescribedBy;
    props['aria-invalid'] = invalid || undefined;
    props['aria-describedby'] = invalid
      ? `${metadata.errorId} ${ariaDescribedBy ?? ''}`.trim()
      : ariaDescribedBy;
  }

  return simplify(props);
};
```

この関数は以下のプロパティを生成します：

- `id`: CheckboxのID
- `key`: Reactのkey属性
- `required`: 必須フィールドかどうか
- `name`: フィールド名
- `defaultChecked`: 初期チェック状態
- `aria-invalid`: 無効な入力かどうか
- `aria-describedby`: エラーメッセージを関連付ける

これらのプロパティを使用することで、アクセシビリティを考慮しつつ、一貫性のある実装を維持できます。

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

## まとめ

shadcn-uiとconformを組み合わせることで、基本的なCheckboxを簡単に実装できます。さらに、カスタムhelper関数を使用することで、コードをより簡潔にし、保守性を高めることができます。適切なバリデーションとエラーハンドリングを行うことで、ユーザーフレンドリーなチェックボックスフォームを作成することができます。

次回は、Switchの実装について詳しく解説します。お楽しみに！
