---
title: "Better Authのスキーマ管理はどう動いているのか - Kysely・Prisma・Drizzleへのコード生成の仕組み"
emoji: "🔐"
type: "tech"
topics: ["betterauth", "typescript", "prisma", "drizzle", "kysely"]
published: true
---

## これはなに？

TypeScript向けの認証ライブラリ [Better Auth](https://better-auth.com/) は、Kysely、Prisma、Drizzle など複数のDBアダプタに対応しています。さらにプラグインを追加すると、二要素認証や組織管理に必要なテーブルが動的に増える設計になっています。

アダプタごとにスキーマの書き方もマイグレーションの仕組みも異なる中で、CLIがどうやって各ツールの定義ファイルへ反映しているのか、ソースコードを追って調べてみました。

## スキーマ定義が合成される流れ

Better Auth のスキーマ情報は、単一の静的ファイルではなく3系統の定義をメモリ上で合成して組み立てられます。

1. **コアテーブル**: `user`、`session`、`account`、`verification` といった認証の基本スキーマ
2. **プラグイン**: `twoFactor` や `organization` などのプラグインがそれぞれ要求する拡張スキーマ
3. **ユーザー設定**: `auth.ts` でユーザーが指定したテーブル名・フィールド名の変更やカスタムフィールド

CLI（`@better-auth/cli`）を実行すると、まずユーザーのプロジェクト内にある `auth.ts` を評価し、有効なプラグインとDB設定を読み出します。そのうえで内部スキーマを1つに統合し、選択されたアダプタ向けの生成処理へ渡す仕組みです。

ここから先のアプローチが、アダプタの性質ごとに大きく分かれています。

## アダプタごとに異なるアプローチ

### 1. Kysely: 自前でSQLを出力して適用まで行う

Better Auth の標準（ビルトイン）構成では内部で Kysely を使っています。

Kysely 向けの場合、CLI の `generate` コマンドは統合済みスキーマから直接 `.sql` ファイル（`CREATE TABLE` や `ALTER TABLE`）を組み立てます。処理の本体は [`packages/better-auth/src/db/get-migration.ts`](https://github.com/better-auth/better-auth/blob/main/packages/better-auth/src/db/get-migration.ts) と [`packages/cli/src/generators/kysely.ts`](https://github.com/better-auth/better-auth/blob/main/packages/cli/src/generators/kysely.ts) です。

さらに Kysely 向けに限り、`@better-auth/cli migrate` コマンドで生成した SQL をそのままデータベースへ直接流し込めます。別途マイグレーションツールを用意せずに完結できる手軽さがある一方、ORM 固有のリレーション定義などは扱えません。

### 2. Prisma: schema.prisma を AST で解析して追記する

Prisma を使っているプロジェクトでは、Better Auth が勝手にマイグレーションを実行することはありません。代わりに既存の `schema.prisma` を書き換えるアプローチをとっています。

CLI は [`packages/cli/src/generators/prisma.ts`](https://github.com/better-auth/better-auth/blob/main/packages/cli/src/generators/prisma.ts) 内で `@mrleebo/prisma-ast` を使い、既存の `schema.prisma` を構文木として読み込みます。必要なモデルやフィールドが存在しなければ差分を追加し、テーブル名のカスタマイズがあれば `@map` 属性を挿入してファイルを上書きします。

スキーマファイルを更新した後のマイグレーション（`prisma migrate dev` や `prisma db push`）は、開発者が普段のワークフローどおりに実行します。Better Auth 側が無理にマイグレーションへ介入せず、Prisma のライフサイクルに委ねる設計です。

### 3. Drizzle: TypeScript のテーブル定義コードを生成する

Drizzle を使う場合は、TypeScript のスキーマファイル（`auth-schema.ts` など）をコード生成で出力します。

実装は [`packages/cli/src/generators/drizzle.ts`](https://github.com/better-auth/better-auth/blob/main/packages/cli/src/generators/drizzle.ts) にあり、対象データベースが PostgreSQL か MySQL か SQLite かに応じて `pgTable`、`mysqlTable`、`sqliteTable` の定義コードを書き分けます。型注釈や制約、外部キー設定も TypeScript のコードとして組み立てられます。

Prisma と同様に、DBへの反映は `drizzle-kit generate` や `drizzle-kit migrate` に任せる形です。

## 各アダプタの対応方針の整理

各アダプタの違いを整理すると次の表のようになります。

| アダプタ | `generate` コマンドの出力 | `migrate` コマンドの対応 | 実際の反映手順 |
| :--- | :--- | :--- | :--- |
| **Kysely** | SQLファイル (`.sql`) | 対応（直接実行） | `npx @better-auth/cli migrate` |
| **Prisma** | `schema.prisma` のAST更新 | 非対応 | `npx prisma migrate dev` |
| **Drizzle** | TypeScript定義コード | 非対応 | `npx drizzle-kit generate` など |

外部ORMを使うときはスキーマファイルの更新やコード生成にとどめ、実際のマイグレーション実行は各エコシステムの公式CLIに委ねる、という役割分担が徹底されています。ライブラリ側でDBの接続管理やロールバックを無理に抱え込まない、現実的で扱いやすい設計だと感じました。
