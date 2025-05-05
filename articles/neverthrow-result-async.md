---
title: "neverthrow: ResultAsync の使い方 (和訳)"
emoji: "🔄"
type: "tech"
topics: ["neverthrow", "typescript", "prisma"]
published: true
---

:::message
この文書は [neverthrow のリポジトリ](https://github.com/supermacro/neverthrow)にある [Working with ResultAsync](https://github.com/supermacro/neverthrow/wiki/Working-with-ResultAsync) を和訳したものです。
:::

# ResultAsync の使い方

`ResultAsync` は、非同期処理の結果を型安全に扱うための型です。
失敗する可能性のある同期操作の結果を扱う `Result<T,E>` と同様に、`ResultAsync<T,E>` は非同期操作の結果を表すために使用できます。

## `fromPromise` で Promise をラップする

Neverthrow は、`Promise` を `ResultAsync` に変換するユーティリティ関数 `fromPromise` を提供しています。
この関数の典型的なユースケースの1つは、`Prisma`（または他のORM）がデータベースから返す `Promise` を処理することです。

```typescript
const userPromise = prisma.user.findUnique(...)

// result は ResultAsync<User | null, {err: unknown, message: string}> 型になります
const result = fromPromise(
  userPromise,
  err => ({err, message: "データベース読み取り中にエラーが発生しました"})
)
```

`fromPromise` の第二引数には、`Promise` が `reject` されたときに受け取るエラー (`unknown` 型) を引数とし、独自のエラーオブジェクトを返す関数を渡している点に注意してください。
これにより、`ResultAsync<T, Error>` における `Error` の型を自由に定義できます。

### Promise.catch() について

`ResultAsync` を使う場合、`Promise` ネイティブの `.catch` メソッドでエラーを処理する必要はありません。エラー処理は `ResultAsync` が担います。

### async/await と try-catch について

`Promise.catch()` と同様に、`Promise` を `await` したり、`try-catch` でエラーを捕捉したりする必要もありません。
もし `ResultAsync` を扱う関数を `async` にしたいと感じた場合、それはおそらく関数の役割や非同期処理の扱い方について誤解がある可能性があります。

### 例

以下に、同じ非同期処理を3つの異なる方法でエラーハンドリングする例を示します。

```typescript
// コールバックベース
const callbackUser = () => {
    const userPromise = prisma.user.findUnique(...)

    userPromise
    .then(user => console.log("ユーザーが見つかりました", user))
    .catch(err => console.error("ユーザー読み取り中のエラー", err))
}

// async/await ベース
const asyncAwaiter = async () => {
    try {
        const user = await prisma.user.findUnique(...)
        console.log("ユーザーが見つかりました", user)
    } catch (err) {
        console.error("ユーザー読み取り中のエラー", err)
    }
}

// Neverthrow ベース
const neverthrower = () => {
    const userPromise = prisma.user.findUnique(...)

    // エラー型を制御するために fromPromise を使用
    const result = fromPromise(
      userPromise,
      originalError => ({originalError, message: "ユーザー読み取り中のエラー"})
    )

    // match で成功と失敗の両方を処理
    result.match(
        user => console.log("ユーザーが見つかりました", user),
        err => console.log(err.message, err.originalError)
    )
}
```

通常、`ResultAsync` でラップした直後に `match` などで結果を処理することは少ないでしょう。なぜなら、それだけではコードが少し冗長になるだけで、`ResultAsync` の利点を十分に活かせないからです。

`ResultAsync` の真価は、複数の `Result` や `ResultAsync` を `.map` や `.andThen` といったメソッドで連鎖させるときに発揮されます。これにより、最終的な結果を一箇所でまとめてアンラップし (`match`、`map`、`mapErr` などを使用)、成功とエラーの処理をその箇所に集約できます。
