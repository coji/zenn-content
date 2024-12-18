---
title: "shadcn/uiとconformによるRadioGroupの実装ガイド"
emoji: "📝"
type: "tech"
topics: ["conform", "shadcnui", "zod"]
published: true
---

## 関連記事

このシリーズの他の記事もご覧ください：

- [shadcn/uiとconformによるInputの実装ガイド](https://zenn.dev/coji/articles/conform-shadcn-ui-input)
- [shadcn/uiとconformによるTextareaの実装ガイド](https://zenn.dev/coji/articles/conform-shadcn-ui-textarea)
- [shadcn/uiとconformによるSelectの実装ガイド](https://zenn.dev/coji/articles/conform-shadcn-ui-select)
- [shadcn/uiとconformによるCheckboxの実装ガイド](https://zenn.dev/coji/articles/conform-shadcn-ui-checkbox)
- [shadcn/uiとconformによるSwitchの実装ガイド](https://zenn.dev/coji/articles/conform-shadcn-ui-switch)
- [shadcn/uiとconformによるRadioGroupの実装ガイド](https://zenn.dev/coji/articles/conform-shadcn-ui-radiogroup)

これらの記事を組み合わせることで、shadcn/uiとconformを使用した包括的なフォーム実装の知識を得ることができます。

## 概要

この記事では、shadcn/uiとconformを使用して基本的なRadioGroupを実装する方法を解説します。通常の実装方法と、カスタムhelper関数を使用した実装方法の両方を紹介します。

shadcn/ui の RadioGroup コンポーネント自体の仕様や使い方については、公式サイトをご覧ください。

https://ui.shadcn.com/docs/components/radio-group

## セットアップ

まず、必要なライブラリをインポートします。

```typescript
import { useForm } from '@conform-to/react'
import { getZodConstraint, parseWithZod } from '@conform-to/zod'
import { z } from 'zod'
import { RadioGroup, RadioGroupItem } from '~/components/ui/radio-group'
import { Label } from '~/components/ui/label'
import { getRadioGroupProps } from './helper'
```

基本的なフォーム構造は以下のようになります：

```jsx
export default function RadioGroupForm() {
  const [form, fields] = useForm({
    onValidate: ({ formData }) => parseWithZod(formData, { schema }),
    constraint: getZodConstraint(schema),
    shouldValidate: 'onBlur',
    shouldRevalidate: 'onInput',
  })

  return (
    <form {...getFormProps(form)} method="post">
      {/* RadioGroup fields will be placed here */}
    </form>
  )
}
```

## スキーマ定義

```typescript
const schema = z.object({
  plan: z.enum(['basic', 'pro', 'enterprise'], {
    required_error: 'プランを選択してください',
    message: '無効な選択です',
  }),
})
```

## 実装例

### 1. 基本的なRadioGroup実装

```jsx
<div>
  <Label>プランを選択</Label>
  <RadioGroup
    key={fields.plan.key}
    name={fields.plan.name}
    defaultValue={fields.plan.initialValue}
    onValueChange={(value) => {
      form.update({
        name: fields.plan.name,
        value,
      })
    }}
    aria-invalid={!fields.plan.valid || undefined}
    aria-describedby={!fields.plan.valid ? fields.plan.errorId : undefined}
  >
    <div className="flex items-center space-x-2">
      <RadioGroupItem value="basic" id="basic" />
      <Label htmlFor="basic">ベーシック</Label>
    </div>
    <div className="flex items-center space-x-2">
      <RadioGroupItem value="pro" id="pro" />
      <Label htmlFor="pro">プロ</Label>
    </div>
    <div className="flex items-center space-x-2">
      <RadioGroupItem value="enterprise" id="enterprise" />
      <Label htmlFor="enterprise">エンタープライズ</Label>
    </div>
  </RadioGroup>
  <div id={fields.plan.errorId} className="text-destructive">
    {fields.plan.errors}
  </div>
</div>
```

### 2. Helper関数を使用したRadioGroup実装

helper.tsファイルに定義されたgetRadioGroupProps関数を使用して、より簡潔に実装できます。

```jsx
<div>
  <Label>プランを選択</Label>
  <RadioGroup
    {...getRadioGroupProps(fields.plan)}
    key={fields.plan.key}
    onValueChange={(value) => {
      form.update({
        name: fields.plan.name,
        value,
      })
    }}
  >
    <div className="flex items-center space-x-2">
      <RadioGroupItem value="basic" id="basic" />
      <Label htmlFor="basic">ベーシック</Label>
    </div>
    <div className="flex items-center space-x-2">
      <RadioGroupItem value="pro" id="pro" />
      <Label htmlFor="pro">プロ</Label>
    </div>
    <div className="flex items-center space-x-2">
      <RadioGroupItem value="enterprise" id="enterprise" />
      <Label htmlFor="enterprise">エンタープライズ</Label>
    </div>
  </RadioGroup>
  <div id={fields.plan.errorId} className="text-destructive">
    {fields.plan.errors}
  </div>
</div>
```

## Helper関数の説明

helper.tsファイルには、RadioGroupの実装を簡略化するための`getRadioGroupProps`関数を定義しました：

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

export const getRadioGroupProps = <Schema>(
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
    key?: string;
    required?: boolean;
    name: string;
    defaultValue?: string;
    'aria-invalid'?: boolean;
    'aria-describedby'?: string;
  } = {
    key: metadata.key,
    required: metadata.required,
    name: metadata.name,
  };

  if (typeof options.value === 'undefined' || options.value) {
    props.defaultValue = metadata.initialValue?.toString();
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

- `key`: Reactのkey属性
- `name`: フィールド名
- `defaultValue`: 初期選択値
- `aria-invalid`: 無効な入力かどうか
- `aria-describedby`: エラーメッセージを関連付ける

これらのプロパティを使用することで、アクセシビリティを考慮しつつ、一貫性のある実装を維持できます。

## デバッグとテスト

フォームの値とエラーを確認するためのデバッグセクションを追加することができます：

```jsx
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

## デモと実装例

本記事で解説した内容の実際の動作を確認したい場合は、以下のデモページをご覧ください：

[shadcn-uiとconformを使用したフォーム実装デモ](https://www.techtalk.jp/demo/conform/shadcn-ui)

このデモページでは、shadcn/uiとconformを組み合わせた様々なフォーム要素の実装例を見ることができます。各要素の動作を確認でき、実際のユーザー体験を把握するのに役立ちます。

また、デモページからは実装のソースコードにもアクセスできます。ソースコードを参照することで、本記事で説明した実装方法がどのように適用されているかを詳細に確認できます。

デモとソースコードを併せて確認することで、理論と実践の両面から理解を深めることができます。ぜひ、デモページを訪れて、実際の動作とコードの詳細を確認してみてください。

## まとめ

shadcn/uiとconformを組み合わせることで、基本的なRadioGroupを簡単に実装できます。さらに、カスタムhelper関数を使用することで、コードをより簡潔にし、保守性を高めることができます。適切なバリデーションとエラーハンドリングを行うことで、ユーザーフレンドリーな選択フォームを作成することができます。

これで、主要なフォーム要素（Input、Textarea、Select、Checkbox、Switch、RadioGroup）の実装方法を網羅しました。これらの要素を組み合わせることで、複雑なフォームも簡単に作成できるようになります。
