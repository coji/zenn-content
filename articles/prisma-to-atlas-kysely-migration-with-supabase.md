---
title: 'Prisma から Atlas + Kysely への移行 - Supabase 環境での実践記録'
emoji: '🔄'
type: 'tech'
topics: ['prisma', 'kysely', 'atlas', 'supabase', 'postgresql']
published: true
---

## これはなに？

本番環境で約2年間 Prisma を使っていたプロジェクトを、Atlas + Kysely に移行した記録です。Supabase の PostgreSQL を使っている環境だったので、いくつか注意点がありました。この記事では、移行の動機から具体的な手順、そして移行リスクを抑えるために追加した E2E テストまで、一連の流れを共有します。

## プロジェクトの背景

このプロジェクトは 2021年8月に Next.js で開発を開始し、約4年半運用しています。

技術スタックは何度か変わっていて、最初は Next.js と Supabase SDK を使っていました。認証も Supabase Auth を使っていたのですが、2023年7月に Remix へ移行したタイミングで Supabase SDK を外し、Prisma と自前の認証に切り替えました。2024年11月には React Router v7 に移行して、現在に至ります。

累計コミット数は 1,400 以上になりました。一人で開発・保守しながら継続的に機能追加・改善を行っている、いわゆる「生きているプロダクト」です。

## 移行の動機

直接のきっかけは Prisma v7 へのアップグレードでした。

Prisma v7 では設定周りが大きく変わりました。`prisma.config.ts` での設定が必要になり、ジェネレーター名が `prisma-client-js` から `prisma-client` に変更され、出力先も `node_modules/.prisma/client` から `generated/prisma` に変わりました。

さらに気になったのが、公式ブログで「3倍高速」と謳われていたにもかかわらず、GitHub の Issue で報告されているベンチマークでは逆の結果が出ていたことです。読み取り操作で 35-40% 遅い、書き込み操作で 26-27% 遅い、スループットで 57% 低下という報告がありました。

「v7 にアップグレードして設定を全部書き直すなら、いっそのこと Prisma から離れよう」と決断しました。

## 移行のハードルが低かった理由

実は、プロジェクトでは既に Kysely を併用していました。

管理画面にダッシュボードがあり、グラフ表示のために集計クエリを書く必要がありました。GROUP BY や集計関数、サブクエリなどを使う複雑なクエリは Prisma では書きづらく、`prisma-kysely` というジェネレータを使って Prisma スキーマから Kysely の型を生成し、集計系のクエリは Kysely で書いていたのです。

```typescript
// 管理画面の集計クエリは元々 Kysely で書いていた
const monthlyStats = await db
  .selectFrom('activityLogs')
  .select([sql<string>`date_trunc('month', created_at)`.as('month'), (eb) => eb.fn.count('id').as('count')])
  .groupBy(sql`date_trunc('month', created_at)`)
  .orderBy('month', 'desc')
  .execute()
```

この状態だったので、Prisma Client を全て Kysely に置き換える作業量は想定より少なく済みました。

## 移行後の構成

移行前は Prisma CLI でマイグレーションを管理し、Prisma Client と Kysely を併用していました。移行後は Atlas でマイグレーションを管理し、クエリは全て Kysely で書くようになりました。型生成も prisma-kysely から kysely-codegen に変わり、DB スキーマから直接 TypeScript 型を生成するようになりました。

ディレクトリ構成はこのようになっています。

```plaintext
atlas.hcl                    # Atlas 設定
atlas/
├── schema.sql              # スキーマソース（Single Source of Truth）
└── migrations/
    └── 20260112054950_baseline.sql

app/services/kysely/
├── index.ts                # Kysely クライアント
├── types.ts                # kysely-codegen で自動生成
└── models.ts               # 型エイリアス（手動管理）
```

## 移行手順

### kysely-codegen の導入

まず、prisma-kysely から kysely-codegen に切り替えます。

```bash
pnpm add -D kysely-codegen
```

package.json にスクリプトを追加します。`--camel-case` オプションをつけると、DB の `snake_case` カラムを TypeScript では `camelCase` で扱えます。Kysely の `CamelCasePlugin` と組み合わせて使います。

```json
{
  "scripts": {
    "db:codegen": "kysely-codegen --out-file app/services/kysely/types.ts --camel-case"
  }
}
```

### Prisma Client から Kysely への移行

ファイルごとに Prisma Client を Kysely に置き換えていきます。Kysely は ORM ではなくクエリビルダなので、JOIN は自分で書く必要があります。慣れると SQL を書く感覚に近くて気持ちいいです。

```typescript
// Before: Prisma
const user = await prisma.user.findUnique({
  where: { email },
  include: { profile: true },
})

// After: Kysely
const user = await db
  .selectFrom('users')
  .innerJoin('profiles', 'profiles.id', 'users.id')
  .selectAll()
  .where('users.email', '=', email)
  .executeTakeFirst()
```

### Atlas の導入

Prisma のマイグレーション機能を Atlas に置き換えます。

```bash
# Atlas インストール
curl -sSf https://atlasgo.sh | sh

# 既存スキーマをエクスポート
atlas schema inspect --url "$DATABASE_URL" --format '{{ sql . }}' > atlas/schema.sql

# ベースラインマイグレーション作成
atlas migrate diff baseline --env local
```

`atlas.hcl` で環境を定義します。`local` は開発環境用で、`dev` に Docker イメージを指定してマイグレーションの検証ができます。

```hcl
variable "database_url" {
  type    = string
  default = getenv("DATABASE_URL")
}

env "local" {
  src = "file://atlas/schema.sql"
  url = var.database_url
  dev = "docker://postgres/16/dev?search_path=public"
  migration {
    dir = "file://atlas/migrations"
  }
}
```

## Supabase との統合

Supabase を使っている場合、これを知らないと確実にハマります。

Supabase は内部で多くのスキーマ（`auth`, `storage`, `realtime` など）と拡張機能を管理しています。これらを Atlas の管理対象から除外しないと、マイグレーション時に衝突します。

```hcl
env "production" {
  url = var.database_url
  migration {
    dir = "file://atlas/migrations"
  }
  exclude = [
    # Supabase 内部スキーマ
    "auth.*",
    "storage.*",
    "realtime.*",
    "graphql.*",
    "graphql_public.*",
    "pgbouncer.*",
    "pgsodium.*",
    "pgsodium_masks.*",
    "vault.*",
    "extensions.*",

    # Supabase が管理する拡張機能
    "pg_graphql",
    "pg_stat_statements",
    "pgcrypto",
    "pgjwt",
    "pgsodium",
    "supabase_vault",
    "uuid-ossp",
    "hypopg",
    "index_advisor",

    # Prisma のレガシーテーブル（移行後も残る）
    "public._prisma_migrations",
  ]
}
```

## E2E テストの追加

移行リスクを抑えるため、Playwright で E2E テストを追加しました。

### テストユーザの作成

Kysely を使ってテスト用ユーザを作成し、Playwright でログインしてセッションを保存します。

```typescript
// e2e/global-setup.ts
import bcrypt from 'bcrypt'
import { Kysely, PostgresDialect, CamelCasePlugin } from 'kysely'
import { chromium } from 'playwright'
import type { DB } from '~/services/kysely/types'

export default async function globalSetup() {
  const db = new Kysely<DB>({
    dialect: new PostgresDialect({ pool }),
    plugins: [new CamelCasePlugin()],
  })

  const encryptedPassword = await bcrypt.hash('Password123!', 10)

  await db
    .insertInto('users')
    .values({
      id: '00000000-0000-0000-0000-000000000000',
      email: 'e2e@example.com',
      encryptedPassword,
      emailConfirmedAt: new Date(),
    })
    .onConflict((oc) => oc.column('id').doUpdateSet({ encryptedPassword }))
    .execute()

  const browser = await chromium.launch()
  const page = await browser.newPage()

  await page.goto('http://localhost:3000/signin')
  await page.getByLabel('メールアドレス').fill('e2e@example.com')
  await page.getByLabel('パスワード').fill('Password123!')
  await page.getByRole('button', { name: 'サインイン' }).click()

  await page.context().storageState({ path: 'e2e/.auth/user.json' })
  await browser.close()
}
```

### DB アサーションを含むテスト

E2E テストで UI だけでなく、DB の状態も検証できます。これが移行の検証に役立ちました。

```typescript
// e2e/helpers/db.ts
export const getActivityLogCount = async (email: string) => {
  const db = getDb()
  const result = await db
    .selectFrom('users')
    .innerJoin('activityLogs', 'activityLogs.userId', 'users.id')
    .select((eb) => eb.fn.countAll<string>().as('count'))
    .where('users.email', '=', email)
    .executeTakeFirst()

  return Number(result?.count ?? 0)
}

// テスト内で使用
test('検索するとログが記録される', async ({ page }) => {
  await page.goto('/search')
  await page.fill('input', 'テスト')
  await page.click('button[type="submit"]')

  expect(await getActivityLogCount('e2e@example.com')).toBe(1)
})
```

### CI での注意点

CI 環境では Vite dev server の HMR が不安定だったため、本番ビルドを使うようにしました。

```typescript
// playwright.config.ts
const isCI = !!process.env.CI
const baseURL = isCI ? 'http://localhost:3000' : 'http://localhost:5173'

export default defineConfig({
  webServer: {
    command: isCI ? 'react-router-serve ./build/server/index.js' : 'pnpm dev',
    url: baseURL,
    reuseExistingServer: !isCI,
  },
})
```

## Kysely の便利な機能

### プラグインの活用

CamelCasePlugin は snake_case のカラム名を camelCase に自動変換してくれます。ParseJSONResultsPlugin は JSON カラムを自動でパースしてくれます。

```typescript
const db = new Kysely<DB>({
  dialect: new PostgresDialect({ pool }),
  plugins: [new CamelCasePlugin(), new ParseJSONResultsPlugin()],
})
```

### 型エイリアスの整理

kysely-codegen が生成する型はテーブル名そのままなので、エイリアスを定義しておくと使いやすいです。

```typescript
// app/services/kysely/models.ts
import type { Selectable, Insertable, Updateable } from 'kysely'
import type { Users, Profiles, Posts } from './types'

export type User = Selectable<Users>
export type CreateUser = Insertable<Users>
export type UpdateUser = Updateable<Users>

export type Profile = Selectable<Profiles>
export type Post = Selectable<Posts>
```

### トランザクション

トランザクションは `db.transaction().execute()` で書けます。コールバック内で `trx` を使ってクエリを実行します。

```typescript
await db.transaction().execute(async (trx) => {
  const user = await trx
    .insertInto('users')
    .values({ ... })
    .returningAll()
    .executeTakeFirstOrThrow()

  await trx
    .insertInto('profiles')
    .values({ id: user.id, ... })
    .execute()
})
```

## 振り返り

Atlas は Go 製で軽量です。Prisma CLI と比べるとかなり軽く、`schema.sql` がスキーマの Single Source of Truth になるのでわかりやすいです。Kysely の型安全性も良く、コンパイル時にクエリのエラーを検出できます。ランタイムに Prisma Client が不要になったことで、バンドルサイズも削減できました。

一方で、Atlas の日本語ドキュメントはまだ少ないです。Supabase 環境では exclude 設定が必須で、これを知らないとハマります。Kysely は ORM より低レベルなので、JOIN を自分で書く必要があります。SQL に慣れていれば問題ないですが、ORM に慣れている人は最初戸惑うかもしれません。

## まとめ

Prisma v7 へのアップグレードを機に、Atlas + Kysely に移行しました。既に集計クエリで Kysely を使っていたこともあり、移行のハードルは想定より低かったです。

Supabase 環境では exclude 設定が必須なので、これから移行する方は忘れずに設定してください。E2E テストを先に整備しておくと、移行中の動作確認がスムーズになります。

Prisma のバージョンアップ対応に疲弊している方、集計クエリなど SQL を直接書きたい場面が増えてきた方には、Atlas + Kysely への移行は検討する価値があると思います。
