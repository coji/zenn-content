---
title: Remix + Conform で動的に増減する複数フィールドフォームのバリデーション
emoji: "↕️"
type: "tech"
topics: ["remix", "conform", "zod", "typescript"]
published: true
---

# はじめに

Web アプリケーションを開発していると、ユーザーから複数の連絡先情報を入力してもらうようなフォームを実装する場面がしばしばあります。例えば、以下のようにグループ登録の際に複数人の名前や電話番号、メールアドレスを一度に入力するケースなどです。

![](/images/remix-conform-nested-array/screencast.gif)

このようなフォームでは、入力フィールドの数が動的に増減できることが求められます。ユーザーがボタンをクリックすることで、新しい連絡先を追加したり、不要な連絡先を削除したりできる必要があるでしょう。さらに、それぞれの連絡先情報は名前、電話番号、メールアドレスなどの複数のフィールドを持つため、ネストしたデータ構造を扱う必要もあります。こういったフォームのバリデーションを正しく実装するのは、案外難しい問題です。動的に増減するフィールドに対して、それぞれ適切なバリデーションを行わなければなりません。

しかし、React のフレームワークである Remix と、フォームバリデーションライブラリの Conform を組み合わせることで、この問題を簡単に解決できます。Conform は、動的に増減するフィールドを Intent Button という機能でサポートしています。さらに、Zod などのスキーマバリデーションライブラリと組み合わせることで、ネストしたデータ構造のバリデーションも容易に実現できます。

本記事では、Remix と Conform を使って動的に増減する複数フィールドフォームを実装する方法を解説します。サンプルコードを交えながら、具体的な実装方法を見ていきましょう。

# 使用技術

本記事で紹介する実装では、以下の技術を使用しています。

## Remix

Remix は、React をベースとしたフルスタックの Web アプリケーションフレームワークです。サーバーサイドレンダリングや、ルーティング、データの取得や更新などの機能を提供しています。
Remix の特徴は、ネストされたルーティングと、それに対応したデータのローディングが簡単に実装できる点です。また、フォームの送信やバリデーションも、Remix の Action という機能を使うことで、サーバーサイドで行うことができます。

## Conform

Conform は、React 用のフォームバリデーションライブラリです。フォームの状態管理やバリデーション、エラー表示などの機能を提供しています。

Conform の大きな特徴は、Intent Button という機能です。これは、フォームの状態を変更するボタン（フィールドの追加・削除など）を簡単に実装できる機能で、動的に増減するフィールドを扱う場合に非常に便利です。

また、Conform は Zod などの外部のスキーマバリデーションライブラリと連携することができます。これにより、複雑なデータ構造のバリデーションを宣言的に記述できます。

## Zod

Zod は、TypeScript のためのスキーマバリデーションライブラリです。オブジェクトの形状や、プロパティの型、制約などを宣言的に記述できます。

フォームバリデーションの文脈では、Zod を使うことで、フォームデータの期待する形状を予め定義しておくことができます。そして、実際に入力されたデータが、その定義に沿っているかどうかを簡単にチェックできます。特に、オブジェクトの配列のようなネストしたデータ構造の場合、Zod の記述力が威力を発揮します。

本記事のサンプルコードでは、これらの技術を組み合わせることで、動的に増減する複数フィールドフォームのバリデーションを実装しています。次のセクションから、実際のコードを見ていきましょう。

# コードの解説

それでは、実際のコードを見ていきましょう。サンプルコードは、連絡先情報（名前、郵便番号、電話番号、メールアドレス）の配列を入力するフォームを実装しています。

## スキーマの定義（Zod）

まず、Zod を使ってフォームデータのスキーマを定義します。

```ts
const schema = z.object({
  persons: z.array(
    z.object({
      name: z.string({ required_error: "必須" }),
      zip: z
        .string({ required_error: "必須" })
        .regex(/^\d{3}-\d{4}$/, { message: "000-0000形式" }),
      tel: z
        .string({ required_error: "必須" })
        .regex(/^\d{3}-\d{4}-\d{4}$/, { message: "000-0000-0000形式" }),
      email: z
        .string({ required_error: "必須" })
        .email({ message: "メールアドレス" }),
    })
  ),
});
```

ここでは、`persons` という名前の配列を定義しています。配列の各要素は、`name`, `zip`, `tel`, `email` のプロパティを持つオブジェクトです。

それぞれのプロパティには、`required_error` でバリデーションエラー時のメッセージを指定しています。また、`zip` と `tel` には正規表現によるフォーマットのチェックを、`email` にはメールアドレスのフォーマットのチェックを行っています。

## フォームの実装（Conform）

次に、Conform を使ってフォームを実装します。

```ts
export default function ConformNestedArrayDemo() {
  const { defaultPersons, fakePersons } = useLoaderData<typeof loader>();
  const actionData = useActionData<typeof action>();
  const [form, fields] = useForm({
    lastResult: actionData?.lastResult,
    defaultValue: { persons: defaultPersons },
    onValidate: ({ formData }) => parseWithZod(formData, { schema }),
    shouldValidate: "onSubmit",
    shouldRevalidate: "onInput",
  });
  const persons = fields.persons.getFieldList();

  // ...
}
```

ここでは、`useForm` フックを使ってフォームを作成しています。`defaultValue` には、サーバーサイドで生成した初期値を設定しています。

`onValidate` には、`parseWithZod` を指定し、先ほど定義した Zod のスキーマを使ってバリデーションを行うようにしています。

`fields.persons.getFieldList()` で、`persons` フィールドの配列を取得しています。これをもとに、後ほどフィールドの動的な増減を実装します。

## フィールドの動的な増減（Conform の Intent Button）

Conform の Intent Button を使って、フィールドの動的な増減を実装します。

```ts
<Button
  size="xs"
  variant="outline"
  disabled={persons.length >= fakePersons.length}
  {...form.insert.getButtonProps({
    name: fields.persons.name,
    defaultValue: {
      ...fakePersons[persons.length],
    },
  })}
>
  Append
</Button>
```

ここでは、`form.insert.getButtonProps` を使って、新しいフィールドを追加するボタンを作成しています。`name` には、追加するフィールドの名前（ここでは `persons`）を指定します。`defaultValue` には、新しく追加するフィールドの初期値を指定します。

```ts
<Button
  variant="outline"
  size="xs"
  {...form.remove.getButtonProps({
    name: fields.persons.name,
    index: index,
  })}
>
  <TrashIcon className="h-4 w-4" />
</Button>
```

同様に、`form.remove.getButtonProps` を使って、フィールドを削除するボタンを作成しています。`name` には削除するフィールドの名前を、`index` には削除するフィールドのインデックスを指定します。

このように、Conform の Intent Button を使うことで、フィールドの動的な増減を簡単に実装できます。

## サーバーサイドのデータ処理（Remix の Action）

最後に、Remix の Action を使って、フォームの送信時のサーバーサイドの処理を実装します。

```ts
export const action = async ({ request, response }: ActionFunctionArgs) => {
  const submission = parseWithZod(await request.formData(), { schema });
  if (submission.status !== "success") {
    return { lastResult: submission.reply(), result: null };
  }

  return jsonWithSuccess(
    response,
    {
      lastResult: submission.reply({ resetForm: true }),
      result: submission.value.persons.map((person, idx) => ({
        id: idx + 1,
        ...person,
      })),
    },
    {
      message: "Success!",
      description: `${submission.value.persons.length} persons submitted`,
    }
  );
};
```

ここでは、`parseWithZod` を使ってフォームデータのバリデーションを行っています。バリデーションが成功した場合は、フォームの内容をリセットし、送信されたデータに ID を付与して返しています。バリデーションが失敗した場合は、エラー情報を含んだ `lastResult` を返し、フォームにエラーを表示します。

このように、Remix の Action を使うことで、フォームの送信処理とバリデーションをサーバーサイドで行うことができます。

以上が、コードの主要部分の解説です。Remix と Conform を組み合わせることで、動的に増減する複数フィールドフォームのバリデーションを、非常にシンプルに実装できることがわかります。

# まとめ

本記事では、Remix と Conform を組み合わせて、動的に増減する複数フィールドフォームのバリデーションを実装する方法を紹介しました。

Remix は、React をベースとしたフルスタックの Web アプリケーションフレームワークで、サーバーサイドでのフォーム処理や、データローディングが簡単に実装できます。

Conform は、React 用のフォームバリデーションライブラリで、特に動的に増減するフィールドを扱うのに便利な Intent Button という機能を提供しています。

また、Zod などのスキーマバリデーションライブラリと組み合わせることで、複雑なデータ構造のバリデーションも宣言的に記述できます。

サンプルコードでは、これらの技術を組み合わせて、以下のような実装を行いました。

1. Zod を使ってフォームデータのスキーマを定義
2. Conform を使ってフォームを作成し、Zod のスキーマでバリデーションを実行
3. Conform の Intent Button を使ってフィールドの動的な増減を実装
4. Remix の Action を使ってサーバーサイドでのフォーム処理とバリデーションを実装

これらの技術を組み合わせることで、複雑なフォームのバリデーションロジックを、非常にシンプルかつ宣言的に記述できます。特に、動的に増減するフィールドを扱う場合に、その威力を発揮します。

本記事で紹介した方法は、連絡先情報の入力フォームに限らず、様々な場面で応用できるでしょう。ぜひ、皆さんの開発現場でも試してみてください。

# 参考リンク

[Remix 公式サイト](https://remix.run/)
[Conform 公式サイト(日本語版)](https://ja.conform.guide/)
[Zod 公式サイト](https://zod.dev/)

また、本記事で使用したサンプルコードは、以下の GitHub リポジトリで公開しています。

[サンプルコードリポジトリ](https://github.com/techtalkjp/techtalk.jp/blob/main/app/routes/demo+/conform.nested-array/route.tsx)

動作するデモも以下の URL で公開してます。
https://www.techtalk.jp/demo/conform/nested-array

ぜひ、一度さわってみて、Remix と Conform の組み合わせの利点を体験してみてください。

以上で、技術ブログ記事「Remix + Conform で動的に増減する複数フィールドフォームのバリデーション」の解説は終わりです。

どのような形でフォームを実装する必要があるのか、悩むことも多いかと思います。本記事が、そのような場面での技術選定の一助となれば幸いです。
