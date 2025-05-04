---
title: "neverthrow: ResultAsync の使い方 (和訳)"
emoji: "🔄"
type: "tech"
topics: ["neverthrow", "typescript", "prisma"]
published: true
---

:::message
この文書は neverthrow のリポジトリにある wiki [https://github.com/supermacro/neverthrow/wiki/Working-with-ResultAsync](Working with ResultAsync)を和訳したものです。
:::

# ResultAsync の使い方

ResultAsync は、非同期の結果を型安全な方法で扱うために使用できます。
失敗する可能性のある同期操作の結果を含む `Result<T,E>` と同様に、`ResultAsync<T,E>` を使って非同期操作の結果を表すことができます。

## `fromPromise` で Promise をラップする

Neverthrow は、`Promise` を `ResultAsync` にラップするユーティリティを公開しています。
これの非常に典型的なユースケースの1つは、`Prisma`（または他のORM）がデータベースから返す結果を処理することです。

```typescript
const userPromise = prisma.user.findUnique(...)

// userPromise は ResultAsync<User | null, {err: unknown, message: string}> 型になります
const result = fromPromise(
  userPromise,
  err => ({err, message: "データベース読み取り中にエラーが発生しました"})
)
```

`fromPromise` の第二引数として、`(error: unknown)` を受け取り、私たちが作成した新しいエラーオブジェクトを返す関数を渡していることに注意してください。
これにより、`ResultAsync<T, Error>` における `Error` の型を制御できます。

### Promise.catch()

Promise ネイティブの `.catch` 関数を使ってエラーをキャッチすることはありません。これはまさに `ResultAsync` を使うことで担う役割です。

### async - await と try - catch

`Promise.catch()` と同様に、Promise を `await` したり、エラーを `try-catch` しようとしたりすることはありません。
`ResultAsync` を扱う関数を `async` としてマークしたいと思った場合、おそらくその関数が何をすべきか、そして非同期アクションをどのように処理すべきかについて誤解があるでしょう。

### 例

以下は3つの関数です。それぞれが同じエラーを異なる方法で処理しています。

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

通常、`ResultAsync` をラップした直後にその結果を読み取ることはしないでしょう。なぜなら、それは関数を少し冗長にする以外の効果があまりないからです。

このパラダイムを使用する利点は、異なる `Result` と `ResultAsync` を `.map` 関数や `.andThen` 関数で連鎖させ始めたときに真価を発揮します。これにより、(`match` や `map`、`mapErr` を使って) 結果を一箇所でまとめてアンラップし、エラーと成功に対応する責任を外部委託することができます。
