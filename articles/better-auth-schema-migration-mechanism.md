---
title: "Better Authのスキーママイグレーション管理の仕組み"
emoji: "🔐"
type: "tech"
topics: ["betterauth", "typescript", "prisma", "drizzle", "kysely"]
published: true
---

# Better Auth はどうやってDBアダプタごとのスキーママイグレーションを扱っているのか？

この記事では、[Better Auth](https://better-auth.com/)というTypeScript向け認証・認可ライブラリのスキーママイグレーション管理について解説します。開発者[@beakcru](https://x.com/beakcru)さんが実装した仕組みを見ていきましょう。

Better Authは、TypeScript向けの包括的な認証・認可ライブラリで、様々なデータベースやフレームワークに対応しています。今回は、Better Authがどのようにして多様なDBアダプタ（Kysely、Prisma、Drizzleなど）に対応し、スキーママイグレーションを管理しているのかを解説します。

## なぜスキーマ管理が重要なのか

認証ライブラリにとって、ユーザー情報やセッションデータなどを格納するデータベーススキーマの管理は非常に重要です。特にBetter Authのように**プラグイン**によって必要なスキーマが動的に変わる場合、その管理はさらに複雑になります。

## Better Authのスキーマ管理の基本構造

Better Authのスキーマ管理は、以下の要素を組み合わせて実現しています：

### 1. スキーマ定義の集約

Better Authでは、様々な場所からスキーマ定義を集約しています：

- **コア機能**: `user`、`session`、`account`、`verification`といった基本テーブルのスキーマ定義
- **プラグイン**: 有効化されたプラグイン（`twoFactor`、`organization`、`passkey`、`apiKey`など）が追加するスキーマ定義
- **ユーザー設定**: デフォルトのテーブル名やフィールド名のカスタマイズ、独自フィールドの追加

これらすべてのスキーマ情報はBetter Auth内部で**マージ**され、最終的に必要となる完全なスキーマを構築します。

### 2. CLIによる処理 (`@better-auth/cli`)

`@better-auth/cli`は次のような流れで動作します：

1. ユーザーのプロジェクトからBetter Authの設定ファイル（`auth.ts`など）を読み込み
2. 使用中のDBアダプタとプラグイン、カスタマイズ情報を解析
3. 解析結果と内部スキーマ定義を元に、選択されたアダプタに応じた処理を実行

## アダプタ別のアプローチ

主要なアダプタごとに、具体的にどのようにスキーマ管理を行っているのかを見ていきましょう。

### 1. Kysely（ビルトインアダプタ）

Better Authは内部的に[Kysely](https://kysely.dev/)を使用して、SQLite、PostgreSQL、MySQL、MSSQLなどのRDBMSに対応しています。

#### `generate`コマンドの動作

- 集約されたスキーマ情報を元に、**SQLマイグレーションファイル（`.sql`）**を生成
- `CREATE TABLE`文や`ALTER TABLE`文を含む適切なSQLを出力
- 主に`packages/better-auth/src/db/get-migration.ts`と`packages/cli/src/generators/kysely.ts`で実装

#### `migrate`コマンドの動作

- 生成されたSQLを**データベースに直接適用**
- ユーザーは追加ツールなしでスキーマを最新状態に保てる

#### メリット・デメリット

- **メリット**: CLIだけでスキーマ生成から適用まで完結して手軽
- **デメリット**: ORM固有の機能（Prismaのリレーション構文など）は利用できない

### 2. Prisma

[Prisma](https://www.prisma.io/)は型安全なデータベースアクセスを提供する人気のORMです。

#### `generate`コマンドの動作

- 既存の`schema.prisma`ファイルを解析し、必要なモデルやフィールドを追記・更新
- `@mrleebo/prisma-ast`などを使用してPrismaスキーマをASTとして安全に編集
- ユーザー設定は`@map`属性を使って反映
- 主に`packages/cli/src/generators/prisma.ts`で実装

#### `migrate`コマンドの動作

- **サポートされていません**
- ユーザー自身がPrisma CLI（`prisma migrate dev`や`prisma db push`）を実行する必要あり

#### メリット・デメリット

- **メリット**: Prismaの型安全性やマイグレーション機能を最大限活用可能
- **デメリット**: スキーマ適用はPrisma CLIに依存するため、Better Auth CLIだけでは完結しない

### 3. Drizzle

[Drizzle ORM](https://orm.drizzle.team/)は軽量で高速、型安全なTypeScript ORMです。

#### `generate`コマンドの動作

- **Drizzle ORMのスキーマ定義ファイル（`.ts`）を生成・更新**
- データベース種類に応じて`pgTable`、`mysqlTable`、`sqliteTable`などを使い分け
- フィールドの型、NULL許容、ユニーク制約なども適切に出力
- 主に`packages/cli/src/generators/drizzle.ts`で実装

#### `migrate`コマンドの動作

- **サポートされていません**
- ユーザー自身がDrizzle Kit（`drizzle-kit generate`など）を実行する必要あり

#### メリット・デメリット

- **メリット**: Drizzleの型安全性やDrizzle Kitによるマイグレーション管理を活用可能
- **デメリット**: スキーマ適用はDrizzle Kitに依存するため、Better Auth CLIだけでは完結しない

## まとめ：アダプタ別のスキーマ管理方法

Better Authのスキーママイグレーション生成は、使用するDBアダプタに応じて最適なアプローチを提供します。

| アダプタ | `generate`コマンドの出力 | `migrate`コマンドの機能 | 適用方法 |
|:---------|:------------------------|:----------------------|:---------|
| **Kysely (ビルトイン)** | SQLマイグレーションファイル (`.sql`) | SQLをDBに**直接適用** | `npx @better-auth/cli migrate` |
| **Prisma** | Prismaスキーマ (`schema.prisma`) の更新 | **なし** | `prisma migrate dev`または`prisma db push` |
| **Drizzle** | Drizzleスキーマ (`.ts`) の更新 | **なし** | `drizzle-kit generate`と`drizzle-kit migrate` |

このように、Better Authは各データベース/ORMのエコシステムを尊重しつつ、プラグインによるスキーマ拡張にも対応できる柔軟な仕組みを提供しています。これにより、私たち開発者は自身が選択した技術スタックでスムーズにBetter Authを導入・運用できます。

より詳しい情報は[Better Authの公式ドキュメント](https://better-auth.com/docs)をご覧ください。この記事が皆さんのお役に立てば幸いです！
