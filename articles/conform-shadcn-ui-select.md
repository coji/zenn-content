---
title: "shadcn-uiとconformによるSelectの実装ガイド"
emoji: "📝"
type: "tech"
topics: ["conform", "shadcnui", "zod"]
published: true
---

## 概要

この記事では、shadcn-uiとconformを使用して基本的なSelectを実装する方法を解説します。通常の実装方法と、カスタムhelper関数を使用した実装方法の両方を紹介します。

## セットアップ

まず、必要なライブラリをインポートします。

```typescript
import { useForm } from '@conform-to/react'
import { getZodConstraint, parseWithZod } from '@conform-to/zod'
import { Form } from '@remix-run/react'
import { z } from 'zod'
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '~/components/ui/select'
import { Label } from '~/components/ui/label'
import { getSelectProps, getSelectTriggerProps } from './helper'
```

基本的なフォーム構造は以下のようになります：

```typescript
export default function SelectForm() {
  const [form, fields] = useForm({
    onValidate: ({ formData }) => parseWithZod(formData, { schema }),
    constraint: getZodConstraint(schema),
    shouldValidate: 'onBlur',
    shouldRevalidate: 'onInput',
  })

  return (
    <Form {...getFormProps(form)} method="post">
      {/* Select fields will be placed here */}
    </Form>
  )
}
```

## スキーマ定義

```typescript
const schema = z.object({
  basicSelect: z.enum(['apple', 'banana', 'orange'], {
    required_error: '選択してください',
    invalid_type_error: '無効な選択です',
  }),
})
```

## 実装例

### 1. 基本的なSelect実装

```typescript
<div>
  <Label htmlFor={fields.basicSelect.id}>果物を選択</Label>
  <Select
    name={fields.basicSelect.name}
    defaultValue={fields.basicSelect.initialValue}
    onValueChange={(value) => {
      form.update({
        name: fields.basicSelect.name,
        value: value,
      })
    }}
  >
    <SelectTrigger 
      id={fields.basicSelect.id}
      aria-invalid={!fields.basicSelect.valid || undefined}
      aria-describedby={!fields.basicSelect.valid ? fields.basicSelect.errorId : undefined}
    >
      <SelectValue placeholder="選択してください" />
    </SelectTrigger>
    <SelectContent>
      <SelectItem value="apple">りんご</SelectItem>
      <SelectItem value="banana">バナナ</SelectItem>
      <SelectItem value="orange">オレンジ</SelectItem>
    </SelectContent>
  </Select>
  <div id={fields.basicSelect.errorId} className="text-destructive">
    {fields.basicSelect.errors}
  </div>
</div>
```

### 2. Helper関数を使用したSelect実装

helper.tsファイルに定義されたgetSelectPropsとgetSelectTriggerProps関数を使用して、より簡潔に実装できます。

```typescript
<div>
  <Label htmlFor={fields.basicSelect.id}>果物を選択 (Helper使用)</Label>
  <Select
    {...getSelectProps(fields.basicSelect)}
    onValueChange={(value) => {
      form.update({
        name: fields.basicSelect.name,
        value: value,
      })
    }}
  >
    <SelectTrigger {...getSelectTriggerProps(fields.basicSelect)}>
      <SelectValue placeholder="選択してください" />
    </SelectTrigger>
    <SelectContent>
      <SelectItem value="apple">りんご</SelectItem>
      <SelectItem value="banana">バナナ</SelectItem>
      <SelectItem value="orange">オレンジ</SelectItem>
    </SelectContent>
  </Select>
  <div id={fields.basicSelect.errorId} className="text-destructive">
    {fields.basicSelect.errors}
  </div>
</div>
```

## Helper関数の説明

helper.tsファイルには、Selectの実装を簡略化するための2つの主要な関数が定義されています：

1. `getSelectProps`: Selectに必要なプロパティを生成します。
2. `getSelectTriggerProps`: SelectTriggerに必要なプロパティを生成します。

これらの関数を使用することで、コードの冗長性を減らし、一貫性のある実装を維持できます。

helper.ts の実装は以下のとおりです。

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

export const getSelectProps = <Schema>(
  metadata: FieldMetadata<Schema>,
  options: {
    value?: boolean;
  } = {}
) => {
  const props: {
    key?: string;
    required?: boolean;
    name: string;
    defaultValue?: string;
  } = {
    key: metadata.key,
    required: metadata.required,
    name: metadata.name,
  };

  if (typeof options.value === 'undefined' || options.value) {
    props.defaultValue = metadata.initialValue?.toString();
  }

  return simplify(props);
};

export const getSelectTriggerProps = <Schema>(
  metadata: FieldMetadata<Schema>,
  options:
    | {
        ariaAttributes?: true;
        ariaInvalid?: 'errors' | 'allErrors';
        ariaDescribedBy?: string;
      }
    | {
        ariaAttributes: false;
      } = {
    ariaAttributes: true,
  }
) => {
  const props: {
    id: string;
    'aria-invalid'?: boolean;
    'aria-describedby'?: string;
  } = {
    id: metadata.id,
  };

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

shadcn-uiとconformを組み合わせることで、基本的なSelectを簡単に実装できます。さらに、カスタムhelper関数を使用することで、コードをより簡潔にし、保守性を高めることができます。適切なバリデーションとエラーハンドリングを行うことで、ユーザーフレンドリーな選択フォームを作成することができます。

次回は、Checkboxの実装について詳しく解説します。お楽しみに！
