---
title: "PostgreSQL + Prismaでタグ管理：正規化 vs JSONB vs 配列 パフォーマンス徹底比較"
emoji: "🏷️"
type: "tech"
topics: ["postgresql", "prisma", "typescript", "database", "performance", "benchmark"]
published: true
slug: "postgresql-prisma-tag-management-benchmark"
---

こんにちは、cojiです。今回は**PostgreSQL + Prisma**でタグ機能を実装する時のパフォーマンスについて、気になったので本格的にベンチマークを取ってみました。

正規化、JSONB、配列型の3つのアプローチで、どれが一番速いのか？データ量が増えたらどうなるのか？実際に10,000件と100,000件のデータでPrismaを使って検証してみたので、結果をシェアします。

ソースコードはGitHubで公開してるので、気になる方はこちらもチェックしてみてください。
https://github.com/coji/tags_benchmark

## 先に結論：どのアプローチがおすすめ？

いろいろ測定した結果、**配列(TEXT[])型またはJSONB型が圧倒的におすすめ**です。特に書き込み性能と、データが増えた時の検索性能の安定感が素晴らしいです。

### パフォーマンス比較結果

**検索性能**：
- 10,000件：配列型とJSONB型がほぼ同じ速度で、正規化型より10倍以上高速
- 100,000件：配列型が最速、JSONB型も十分高速。正規化型は時間がかかりすぎて測定を断念...

**書き込み性能**：
- 単件処理：配列型とJSONB型が正規化型より3-4倍高速
- バッチ処理：配列型とJSONB型が正規化型より圧倒的に高速

### 実用的な選び方

- **シンプル重視なら**：**配列(TEXT[])型**が第一候補。GINインデックスとの相性抜群です
- **柔軟性も欲しいなら**：**JSONB型**。タグに属性を持たせたい場合などに便利
- **リレーショナルにこだわるなら**：正規化型も選択肢ですが、性能面では覚悟が必要

以下、この結論に至った経緯を詳しく説明していきます。

## なぜこのベンチマークをやったのか

タグ機能って、ブログサイトとかSNSとか、いろんなアプリで使いますよね。最近はPostgreSQL + Prismaの組み合わせで開発することが多いんですが、タグの実装方法がいくつかあって、どれが一番いいのかよくわからない...

特にPrismaを使う場合、スキーマの書き方やクエリの方法が変わってくるので、実際のパフォーマンスがどうなるのか気になってました。

そこで今回は、以下の3つのアプローチで実際にPrismaでスキーマを定義してデータを作って測定してみることにしました：

1. **正規化（Normalized）**：王道のリレーショナル設計。`Person`、`Tag`、`PersonTag`の3テーブル構成
2. **JSONB**：`Person`テーブルに`tags`カラム（JSONB型）でタグ配列を格納
3. **配列（Array）**：`Person`テーブルに`tags`カラム（TEXT[]型）でタグ配列を格納

## テスト環境とデータ仕様

- **データ量**：10,000人と100,000人
- **タグの種類**：15種類（"engineer"、"remote"など）
- **1人あたりのタグ数**：ランダムで5〜15個
- **検索テスト**：単一タグ、複数タグのAND/OR検索を各1,000回
- **書き込みテスト**：単件挿入、バッチ挿入、更新を各10,000件

## スキーマ設計

Prismaで各アプローチを実装しました。PostgreSQLの機能をPrismaでどう表現するかがポイントですね。コードで見ると違いがわかりやすいです。

### 正規化アプローチ

```prisma
model Person {
  id         Int         @id @default(autoincrement())
  name       String
  personTags PersonTag[]
  @@map("persons")
}

model Tag {
  id         Int         @id @default(autoincrement())
  name       String      @unique
  personTags PersonTag[]
  @@map("tags")
}

model PersonTag {
  personId Int    @map("person_id")
  tagId    Int    @map("tag_id")
  person   Person @relation(fields: [personId], references: [id], onDelete: Cascade)
  tag      Tag    @relation(fields: [tagId], references: [id], onDelete: Cascade)

  @@id([personId, tagId])
  @@index([tagId])
  @@map("person_tags")
}
```

### JSONB アプローチ

```prisma
model PersonJsonb {
  id   Int    @id @default(autoincrement())
  name String
  tags Json   // ['tag1', 'tag2', ...] 形式

  @@index([tags], type: Gin)
  @@map("persons_jsonb")
}
```

### 配列アプローチ

```prisma
model PersonArray {
  id   Int      @id @default(autoincrement())
  name String
  tags String[] // TEXT[]型

  @@index([tags], type: Gin)
  @@map("persons_array")
}
```

GINインデックス（`@@index([tags], type: Gin)`）がポイントです。これがあるからJSONBと配列型が高速になります。Prismaでは`type: Gin`でPostgreSQLのGINインデックスを指定できるのが便利ですね。

## ベンチマーク結果

### 10,000件データでの結果

**検索性能（平均応答時間）**

| クエリタイプ | 正規化 | JSONB | 配列 |
|-------------|--------|-------|------|
| 単一タグ検索 | 573.5ms | 39.5ms | 39.1ms |
| AND検索 | 605.6ms | 35.2ms | 34.2ms |
| OR検索 | 672.0ms | 47.0ms | 46.8ms |

**書き込み性能（1件あたり平均処理時間）**

| 操作 | 正規化 | JSONB | 配列 |
|-----|--------|-------|------|
| 単件挿入 | 3.7ms | 0.9ms | 0.8ms |
| バッチ挿入 | 0.1ms | 0.04ms | 0.04ms |
| 更新 | 3.5ms | 1.7ms | 1.0ms |

10,000件では、検索・書き込みともにJSONBと配列が正規化を圧倒しました。特に検索で10倍以上の差は衝撃的でした。

### 100,000件データでの結果

正規化アプローチは時間がかかりすぎたため、JSONBと配列のみで測定しました。

**検索性能（平均応答時間）**

| クエリタイプ | JSONB | 配列 |
|-------------|-------|------|
| 単一タグ検索 | 355.1ms | 304.3ms |
| AND検索 | 320.6ms | 262.9ms |
| OR検索 | 488.0ms | 376.8ms |

**書き込み性能（1件あたり平均処理時間）**

| 操作 | JSONB | 配列 |
|-----|-------|------|
| 単件挿入 | 0.8ms | 0.6ms |
| バッチ挿入 | 0.03ms | 0.04ms |
| 更新 | 0.9ms | 0.6ms |

100,000件でも、配列型とJSONB型は安定したパフォーマンスを維持してます。配列型がわずかに優位な結果に。

## 実装のポイント

PrismaでPostgreSQLのそれぞれの機能をどう使うかが重要なので、実際のクエリコードを見てみましょう。

### 検索クエリの書き方

#### 配列型の場合

PostgreSQLの配列演算子をPrismaで使う時は、`has`、`hasEvery`、`hasSome`が使えます。これがかなり直感的で使いやすい！

```typescript
// 単一タグ検索
await prisma.personArray.findMany({
  where: { tags: { has: tagName } }
});

// AND検索
await prisma.personArray.findMany({
  where: { tags: { hasEvery: tagNames } }
});

// OR検索
await prisma.personArray.findMany({
  where: { tags: { hasSome: tagNames } }
});
```

#### JSONB型の場合

PrismaのJSONB操作では`array_contains`を使います。PostgreSQLのJSONB演算子をいい感じにラップしてくれてます。

```typescript
// 単一タグ検索
await prisma.personJsonb.findMany({
  where: { tags: { array_contains: [tagName] } }
});

// AND検索
await prisma.personJsonb.findMany({
  where: { tags: { array_contains: tagNames } }
});

// OR検索
const conditions = tagNames.map(tag => ({
  tags: { array_contains: [tag] }
}));
await prisma.personJsonb.findMany({
  where: { OR: conditions }
});
```

#### 正規化の場合

Prismaのリレーション機能を使いますが、JOINを意識したクエリになります。

```typescript
// 単一タグ検索
await prisma.person.findMany({
  where: {
    personTags: { some: { tag: { name: tagName } } }
  },
  include: {
    personTags: { include: { tag: true } }
  }
});
```

正規化だと`include`が必要で、クエリが複雑になりがちですね。Prismaを使っていても、背後のSQLの複雑さは隠しきれない感じです。

### 書き込み処理の違い

Prismaの書き込み処理でも、アプローチによって複雑さが全然違います。

**配列・JSONB型**：Prismaの`createManyAndReturn`がシンプルで最高！

```typescript
// バッチ挿入
await prisma.personArray.createManyAndReturn({
  data: people.map(p => ({
    name: p.name,
    tags: p.tags  // そのまま配列を渡すだけ
  }))
});
```

**正規化型**：Prismaのトランザクション機能を使っても複雑...

```typescript
// バッチ挿入（一部抜粋）
await prisma.$transaction(async (tx) => {
  // 1. 既存タグをチェック
  // 2. 新規タグを作成
  // 3. Personを一括作成
  // 4. PersonTagを一括作成
  // ...複数ステップが必要
});
```

## なぜこんなに差が出るのか？

### 検索性能

- **配列・JSONB**：PostgreSQLのGINインデックスが効果的に働く。PrismaのクエリもPostgreSQLの最適化された配列・JSONB演算子に直接マッピングされる
- **正規化**：複数テーブルのJOINが必要。Prismaが生成するSQLも複雑になり、データが増えるほどコストが跳ね上がる

### 書き込み性能

- **配列・JSONB**：1つのテーブルの1レコードを操作するだけ。Prismaの`createMany`系メソッドも効率的
- **正規化**：3つのテーブルを整合性を保ちながら操作。Prismaのトランザクション処理でも、根本的な複雑さは変わらない

## 実際に使うならどうする？

個人的な使い分けの指針：

### 配列(TEXT[])型がおすすめな場面

- シンプルなタグ機能（文字列のリストだけでOK）
- 高いパフォーマンスが必要
- 実装をサクッと済ませたい

### JSONB型がおすすめな場面

- タグに属性情報を持たせたい（色、優先度など）
- 将来的に構造が変わる可能性がある
- PostgreSQLのJSON機能を活用したい

### 正規化がおすすめな場面

- タグ自体が複雑なエンティティ（作成者、作成日時など）
- 厳密なデータ整合性が必要
- リレーショナルな設計にこだわりたい
- 検索頻度がそれほど高くない

## まとめ

今回のベンチマークで、**PostgreSQL + Prisma**でのタグ管理においては**配列型とJSONB型が圧倒的に有利**だということがわかりました。

特に印象的だったのは：
- 検索性能で10倍以上の差
- データ量が増えても性能が安定
- Prismaでの実装のシンプルさ
- PostgreSQLの高度な機能をPrismaで簡単に使える

もちろん、すべてのケースで配列・JSONB型がベストというわけではありませんが、PostgreSQL + Prismaの組み合わせを使っているなら、多くの場面で有力な選択肢になりそうです。

PostgreSQL + Prismaを使ってる方は、ぜひ一度試してみてください。思った以上に簡単で高速ですよ！

---

この検証結果が、みなさんのDB設計の参考になれば嬉しいです。

何か質問や、「うちではこんな結果だった」みたいな情報があれば、ぜひコメントで教えてください！

**GitHub リポジトリ**：https://github.com/coji/tags_benchmark
