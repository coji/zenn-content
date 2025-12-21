---
title: 'Prisma 7 がつらくて Atlas + Kysely に移行した - Turso + better-auth 環境での実践'
emoji: '🗃️'
type: 'tech'
topics: ['prisma', 'kysely', 'atlas', 'turso', 'betterauth']
published: true
---

## これはなに？

[SlideCraft](https://github.com/techtalkjp/slidecraft) という、AI でスライドを修正・再構成するアプリを開発しています。React Router v7 + Turso + better-auth という構成で、認証基盤を Prisma 7 で管理していました。

しかし Prisma 7 へのアップグレードで色々とつらくなってきたので、思い切って Atlas + Kysely に移行したら想像以上に快適になりました。その経緯と手順を共有します。

https://github.com/techtalkjp/slidecraft

## Prisma 7 のつらみ

### ドライバーアダプター必須化

Prisma 7 から、SQLite 系のデータベース（Turso/LibSQL 含む）を使う場合はドライバーアダプターが必須になりました。

```typescript
import { PrismaClient } from '@prisma/client'
import { PrismaLibSQL } from '@prisma/adapter-libsql'
import { createClient } from '@libsql/client'

const libsql = createClient({
  url: process.env.DATABASE_URL!,
  authToken: process.env.DATABASE_AUTH_TOKEN,
})

const adapter = new PrismaLibSQL(libsql)
const prisma = new PrismaClient({ adapter })
```

これだけでなく、Prisma 7 では設定周りも大きく変わりました。`prisma.config.ts` での設定が必要になり、ジェネレーター名が `prisma-client-js` から `prisma-client` に変更され、出力先も `node_modules/.prisma/client` から `generated/prisma` に変わりました。さらにローカル開発では `@prisma/adapter-better-sqlite3`、本番では `@prisma/adapter-libsql` と使い分けが必要になります。

### Turso との接続でハマる

環境変数の扱いで混乱しました。`DATABASE_URL` にトークンをクエリパラメータとして含めていたら動きません。

```bash
# ダメな例
DATABASE_URL=libsql://xxx.turso.io?authToken=eyJ...

# 正解: 分離する
DATABASE_URL=libsql://xxx.turso.io
DATABASE_AUTH_TOKEN=eyJ...
```

Prisma の libsql アダプターは `url` と `authToken` を別引数で受け取ります。ドキュメントを読めばわかることですが、これに気づくまで時間を溶かしました。

### Prisma CLI が Turso 未サポート

`prisma migrate deploy` が Turso に直接使えません。仕方なくカスタムのマイグレーションスクリプトを書きました。`_prisma_migrations` テーブルで適用済みを管理する形式で、これを維持するのが地味につらいです。

### 生成コードが巨大

Prisma Client は型安全で便利なのですが、生成されるコードが巨大です。今回のプロジェクトでは約 11,000 行のコードが生成されていました。これがビルド時間に影響しますし、エディタの補完も重くなります。

### 設定の複雑さ

Prisma 7 では設定ファイルの構成が大きく変わり、`prisma.config.ts` の書き方やアダプターの使い分けなど、把握すべきことが増えました。ドキュメントを何度も行き来しながら設定を調整する必要があり、正直なところ疲弊しました。

### パフォーマンス問題

Prisma 7 は公式ブログで「3倍高速」と謳われていましたが、[実際のベンチマーク](https://github.com/prisma/prisma/issues/28845)では読み取り操作で 35-40% 遅い、書き込み操作で 26-27% 遅い、スループットで 57% 低下という逆の結果が報告されています。

原因は「クエリコンパイルステップが各クエリ実行前に発生する」ためで、小さなクエリが多数実行される環境では、このオーバーヘッドが顕著になります。サーバーレス/エッジ環境では特に影響が大きいです。

## 移行先の選定

### なぜ Atlas か

[Atlas](https://atlasgo.io/) は HashiCorp の Terraform に影響を受けた、宣言的なスキーマ管理ツールです。SQL ファーストでスキーマを書け、差分マイグレーションを自動生成してくれます。Turso (LibSQL) を公式サポートしていて、CI/CD との統合も簡単です。

### なぜ Kysely か

[Kysely](https://kysely.dev/) は TypeScript 製の型安全なクエリビルダーです。実は 2 年前から別プロジェクトで使っていて、とても気に入っています。

生成コード不要で、型は DB スキーマから生成されます。SQL に近い API なので学習コストが低く、依存も少なく軽量です。better-auth が Kysely インスタンスを直接受け取れるのも大きなポイントでした。

[ベンチマーク](https://izanami.dev/post/1e3fa298-252c-4f6e-8bcc-b225d53c95fb)によると Update は 31%、Upsert は 68% 高速で、集計クエリや CTE も書けるため管理画面を作るときにも便利です。

## 移行手順

### 1. スキーマの移行

Prisma Schema から SQL DDL を作成します。

```sql
-- db/schema.sql
CREATE TABLE user (
  id TEXT PRIMARY KEY NOT NULL,
  name TEXT,
  email TEXT UNIQUE,
  email_verified INTEGER NOT NULL DEFAULT 0,
  image TEXT,
  is_anonymous INTEGER NOT NULL DEFAULT 0,
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE session (
  id TEXT PRIMARY KEY NOT NULL,
  user_id TEXT NOT NULL REFERENCES user(id) ON DELETE CASCADE,
  token TEXT UNIQUE NOT NULL,
  expires_at TEXT NOT NULL,
  ip_address TEXT,
  user_agent TEXT,
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX session_user_id_idx ON session(user_id);
```

### 2. Atlas の設定

```hcl
// atlas.hcl
variable "database_url" {
  type    = string
  default = getenv("DATABASE_URL")
}

variable "database_auth_token" {
  type    = string
  default = getenv("DATABASE_AUTH_TOKEN")
}

env "local" {
  src = "file://db/schema.sql"
  url = "sqlite://file:./data/local.db"
  dev = "sqlite://file?mode=memory"
  migration {
    dir = "file://db/migrations"
  }
}

env "turso" {
  src     = "file://db/schema.sql"
  url     = "${var.database_url}?authToken=${var.database_auth_token}"
  dev     = "sqlite://file?mode=memory"
  exclude = ["_litestream*"]
}
```

既にデータがある DB に Atlas を導入する場合は、ベースラインを設定する必要があります。

```bash
# 初期マイグレーションを生成
atlas migrate diff init --env local

# ローカル DB にベースラインを設定（マイグレーション自体は適用せず、履歴だけ記録）
atlas migrate apply --env local --baseline 20251220054340
```

本番（Turso）は `atlas schema apply` が宣言的に差分を計算するので、ベースライン設定は不要です。

### 3. Kysely のセットアップ

```typescript
// app/lib/db/kysely.ts
import { createClient } from '@libsql/client'
import { LibsqlDialect } from '@libsql/kysely-libsql'
import { CamelCasePlugin, Kysely } from 'kysely'
import type { DB } from './types'

const LOCAL_DATABASE_URL = 'file:./data/local.db'

const databaseUrl = process.env.DATABASE_URL ?? LOCAL_DATABASE_URL
const isTurso = databaseUrl.startsWith('libsql://')

if (isTurso && !process.env.DATABASE_AUTH_TOKEN) {
  throw new Error('DATABASE_AUTH_TOKEN is required for Turso connection')
}

const client = createClient({
  url: databaseUrl,
  authToken: isTurso ? process.env.DATABASE_AUTH_TOKEN : undefined,
})

export const db = new Kysely<DB>({
  dialect: new LibsqlDialect({ client }),
  plugins: [new CamelCasePlugin()],
})
```

`CamelCasePlugin` を使うことで、DB の snake_case カラムを TypeScript では camelCase で扱えます。

### 4. npm scripts の整備

```json
{
  "scripts": {
    "db:migrate": "atlas migrate diff --env local",
    "db:apply": "atlas migrate apply --env local",
    "db:codegen": "kysely-codegen --dialect libsql --url 'file:./data/local.db' --camel-case --out-file app/lib/db/types.ts --exclude-pattern atlas_schema_revisions",
    "turso:apply": "dotenv -e .env.production -- atlas schema apply --env turso"
  }
}
```

| コマンド | 説明 |
| --- | --- |
| db:migrate | スキーマ変更からマイグレーションファイルを生成 |
| db:apply | ローカル DB にマイグレーションを適用 |
| db:codegen | DB スキーマから TypeScript 型を生成 |
| turso:apply | 本番 (Turso) にスキーマを適用 |

日常的なワークフローは、スキーマを編集して `pnpm db:migrate` → `pnpm db:apply` → `pnpm db:codegen` という流れになります。

### 5. better-auth の設定変更

Prisma アダプターから Kysely 直接渡しに変更します。

```typescript
// Before: Prisma アダプター
import { prismaAdapter } from 'better-auth/adapters/prisma'
import { prisma } from '~/lib/db/prisma'

export const auth = betterAuth({
  database: prismaAdapter(prisma, { provider: 'sqlite' }),
})

// After: Kysely 直接
import { db } from '~/lib/db/kysely'

export const auth = betterAuth({
  database: { db, type: 'sqlite' },
  user: {
    modelName: 'user',
    fields: {
      emailVerified: 'email_verified',
      createdAt: 'created_at',
      updatedAt: 'updated_at',
    },
  },
  session: {
    modelName: 'session',
    fields: {
      userId: 'user_id',
      expiresAt: 'expires_at',
    },
  },
})
```

better-auth は CamelCasePlugin を経由せず直接 SQL を発行するため、カラム名のマッピングを明示的に設定する必要があります。

### 6. クエリの書き換え

```typescript
// Before: Prisma
const result = await prisma.apiUsageLog.create({
  data: { userId, inputTokens, outputTokens, operation },
})

// After: Kysely
const result = await db
  .insertInto('apiUsageLog')
  .values({ id: crypto.randomUUID(), userId, inputTokens, outputTokens, operation })
  .returningAll()
  .executeTakeFirstOrThrow()
```

## ハマりポイント

### better-auth のカラム名マッピング

Kysely の CamelCasePlugin は better-auth には効きません。better-auth は内部で直接 SQL を組み立てるため、`fields` オプションで明示的にマッピングする必要があります。これを忘れると `table user has no column named emailVerified` のようなエラーが出ます。

### generateId: false の罠

Prisma 時代に設定していた `generateId: false` をそのまま残していたら、`NOT NULL constraint failed: user.id` エラーが発生しました。better-auth のデフォルトでは ID を自動生成するので、この設定は不要です。

### @libsql/client のバージョン

`@libsql/kysely-libsql` は `@libsql/client ^0.8.0` を要求します。最新の 0.14.x は使えないので注意が必要です。

## 移行後の変化

### コード量の削減

| 項目 | Before | After |
| --- | --- | --- |
| 生成コード | 11,000行 | 100行 |
| DB関連ファイル | 5ファイル | 3ファイル |
| 依存パッケージ | 4つ | 3つ |

### パフォーマンス

[ベンチマーク記事](https://izanami.dev/post/1e3fa298-252c-4f6e-8bcc-b225d53c95fb)によると、Kysely は多くの操作で Prisma より高速です。Find all はほぼ同等、Update は 31% 高速、Upsert は 68% 高速という結果が出ています。ただしネストされたクエリ（`jsonArrayFrom` など）を使うと逆に遅くなるケースがあるので、SQL を意識した書き方が重要です。

### 開発体験

スキーマ変更が楽になりました。SQL を直接編集して `atlas migrate diff` するだけです。型の更新も `kysely-codegen` は一瞬で完了します。巨大な生成ファイルがなくなったのでエディタも軽くなりました。Kysely のクエリは SQL に近いので、何が実行されるか分かりやすいのも気に入っています。

## まとめ

Prisma は素晴らしいツールですが、Turso + better-auth という構成では相性の問題が出てきました。Atlas + Kysely に移行したことで、SQL ファーストでスキーマを管理でき、軽量で高速な型生成、シンプルな依存関係、better-auth との良好な相性という恩恵を得られました。

## 参考

- [Atlas](https://atlasgo.io/)
- [Kysely](https://kysely.dev/)
- [better-auth](https://www.better-auth.com/)
- [Turso](https://turso.tech/)
- [kysely-codegen](https://github.com/RobinBlomberg/kysely-codegen)
