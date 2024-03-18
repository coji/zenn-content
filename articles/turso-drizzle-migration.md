---
title: "Turso の DB を drizzle でマイグレーションやってみる。"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["turso", "drizzle", "db"]
published: true
---

:::message
この記事は Claude3 API で Haiku を使って、Zenn のスクラップ記事を元に LLM によって生成された記事になります。[元のスクラップはこちら](https://zenn.dev/coji/scraps/4ee5385c9dbfc7)です。[生成に使ったコードやプロンプトはこちら](https://github.com/coji/haiku-example/blob/main/commands/zenn-scrap-to-article.ts)
:::

# Turso の DB を drizzle でマイグレーションやってみる。

## はじめに

Turso を使ったデータベースに対して、drizzle というツールを使ってマイグレーションを行う方法を解説します。

## 環境設定

1. まずは、Turso のアカウントを作成し、データベースを作成します。データベースの情報を控えておきましょう。
2. 次に、プロジェクトディレクトリを作成し、npm/pnpm を初期化します。

```shell
mkdir turso-drizzle-migration-example
cd turso-drizzle-migration-example/
pnpm init
```

3. .env ファイルを作成し、Turso の接続情報を記載します。

```
TURSO_CONNECTION_URL=libsql://xxx.turso.io
TURSO_AUTH_TOKEN=xxxxxxxxxx
```

4. drizzle 関連のライブラリをインストールします。

```
pnpm add drizzle-orm @libsql/client
pnpm add -D drizzle-kit
```

## drizzle の設定

1. drizzle のインスタンスを作成するヘルパーファイルを作成します。

```typescript
// src/db/db.ts
import "dotenv/config";
import { drizzle } from "drizzle-orm/libsql";
import { createClient } from "@libsql/client";

const client = createClient({
  url: process.env.TURSO_CONNECTION_URL!,
  authToken: process.env.TURSO_AUTH_TOKEN!,
});

export const db = drizzle(client);
```

2. drizzle.config.ts を作成し、マイグレーションの設定を行います。

```typescript
// drizzle.config.ts
import "dotenv/config";
import type { Config } from "drizzle-kit";
export default {
  schema: "./src/db/schema.ts",
  out: "./migrations",
  driver: "turso",
  dbCredentials: {
    url: process.env.TURSO_CONNECTION_URL!,
    authToken: process.env.TURSO_AUTH_TOKEN!,
  },
} satisfies Config;
```

3. データベースのスキーマを定義します。

```typescript
// src/db/schema.ts
import { sql } from "drizzle-orm";
import { integer, sqliteTable, text } from "drizzle-orm/sqlite-core";

export const users = sqliteTable("users", {
  id: integer("id").primaryKey(),
  name: text("name").notNull(),
  email: text("email").unique().notNull(),
});

export const posts = sqliteTable("posts", {
  id: integer("id").primaryKey(),
  title: text("title").notNull(),
  content: text("content").notNull(),
  userId: integer("user_id")
    .notNull()
    .references(() => users.id, { onDelete: "cascade" }),
  createdAt: text("created_at")
    .default(sql`CURRENT_TIMESTAMP`)
    .notNull(),
});

export type InsertUser = typeof users.$inferInsert;
export type SelectUser = typeof users.$inferSelect;

export type InsertPost = typeof posts.$inferInsert;
export type SelectPost = typeof posts.$inferSelect;
```

## マイグレーションの適用

1. まずはマイグレーションファイルを生成します。

```
npx drizzle-kit generate:sqlite
```

これにより、`migrations/0000_optimal_overlord.sql` というファイルが生成されます。

2. 次に、このマイグレーションを Turso のデータベースに適用するコードを書きます。

```typescript
// src/index.ts
import "dotenv/config";
import { resolve, dirname } from "node:path";
import { db } from "./db/db";
import { migrate } from "drizzle-orm/libsql/migrator";

const __dirname = dirname(new URL(import.meta.url).pathname);

await migrate(db, { migrationsFolder: resolve(__dirname, "../migrations") });
```

3. 最後に、この index.ts ファイルを実行します。

```
pnpm tsx src/index.ts
```

これによりマイグレーションが適用されます。

## データベースの確認

`turso db shell drizzle-turso-db` コマンドでデータベースにアクセスし、`users` と `posts` テーブルが作成されていることを確認できます。

## schema の変更

データベースのスキーマを変更したい場合は、`src/db/schema.ts` を編集して、`npx drizzle-kit push:sqlite` コマンドを実行すると、変更が即座に反映されます。

## まとめ

今回は、Turso のデータベースに対して、drizzle を使ってマイグレーションを行う方法を解説しました。drizzle は手動で SQL を書く必要がなく、型安全なクエリ生成が行えるため、開発効率が高まります。一方で、CLI ツールがないのが少し不便な点です。
