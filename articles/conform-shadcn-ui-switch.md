---
title: "shadcn/uiとconformによるSwitchの実装ガイド"
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

この記事では、shadcn/uiとconformを使用して基本的なSwitchを実装する方法を解説します。通常の実装方法と、カスタムhelper関数を使用した実装方法の両方を紹介します。

shadcn/ui の Switch コンポーネント自体の仕様や使い方については、公式サイトをご覧ください。

https://ui.shadcn.com/docs/components/switch

## セットアップ

まず、必要なライブラリをインポートします。

```typescript
import { useForm } from '@conform-to/react'
import { getZodConstraint, parseWithZod } from '@conform-to/zod'
import { z } from 'zod'
import { Switch, Label } from '~/components/ui'
import { getSwitchProps } from './helper'
```

基本的なフォーム構造は以下のようになります：

```jsx
export default function SwitchForm() {
  const [form, fields] = useForm({
    onValidate: ({ formData }) => parseWithZod(formData, { schema }),
    constraint: getZodConstraint(schema),
    shouldValidate: 'onBlur',
    shouldRevalidate: 'onInput',
  })

  return (
    <form {...getFormProps(form)} method="post">
      {/* Switch fields will be placed here */}
    </form>
  )
}
```

## スキーマ定義

```typescript
const schema = z.object({
  notifications: z.boolean({
    required_error: '選択してください',
    message: '無効な選択です',
  }),
})
```

## 実装例

### 1. 基本的なSwitch実装

```jsx
<div className="flex items-center space-x-2">
  <Switch
    key={fields.notifications.key}
    id={fields.notifications.id}
    name={fields.notifications.name}
    required={fields.notifications.required}
    defaultChecked={fields.notifications.initialValue === 'on'}
    aria-invalid={!fields.notifications.valid || undefined}
    aria-describedby={!fields.notifications.valid ? fields.notifications.errorId : undefined}
    onCheckedChange={(checked) => {
      form.update({
        name: fields.notifications.name,
        value: checked,
      })
    }}
  />
  <Label
    htmlFor={fields.notifications.id}
    className="text-sm font-medium leading-none peer-disabled:cursor-not-allowed peer-disabled:opacity-70"
  >
    通知を受け取る
  </Label>
</div>
<div id={fields.notifications.errorId} className="text-destructive">
  {fields.notifications.errors}
</div>
```

### 2. Helper関数を使用したSwitch実装

helper.tsファイルに定義されたgetSwitchProps関数を使用して、より簡潔に実装できます。

```jsx
<div className="flex items-center space-x-2">
  <Switch
    {...getSwitchProps(fields.notifications)}
    key={fields.notifications.key}
    onCheckedChange={(checked) => {
      form.update({
        name: fields.notifications.name,
        value: checked,
      })
    }}
  />
  <Label
    htmlFor={fields.notifications.id}
    className="text-sm font-medium leading-none peer-disabled:cursor-not-allowed peer-disabled:opacity-70"
  >
    通知を受け取る
  </Label>
</div>
<div id={fields.notifications.errorId} className="text-destructive">
  {fields.notifications.errors}
</div>
```

## Helper関数の説明

helper.tsファイルには、Switch要素の実装を簡略化するための`getSwitchProps`関数を定義しました。

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

export const getSwitchProps = <Schema>(
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

- `id`: SwitchのID
- `key`: Reactのkey属性
- `required`: 必須フィールドかどうか
- `name`: フィールド名
- `defaultChecked`: 初期チェック状態
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
  <pre>{JSON.stringify(form.allErrors, null, 2)}</pre>
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

shadcn/uiとconformを組み合わせることで、基本的なSwitchを簡単に実装できます。さらに、カスタムhelper関数を使用することで、コードをより簡潔にし、保守性を高めることができます。適切なバリデーションとエラーハンドリングを行うことで、ユーザーフレンドリーなスイッチフォームを作成することができます。

次回は、[RadioGroupの実装について](https://zenn.dev/coji/articles/conform-shadcn-ui-radiogroup)詳しく解説します。お楽しみに！
