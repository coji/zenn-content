---
title: '個人開発に無料の日本語全文検索を - Cloudflare D1 と Web Component でつくる検索 API'
emoji: '🔍'
type: 'tech'
topics: ['cloudflare', 'd1', 'sqlite', 'fts5', 'webcomponents']
published: true
---

## これはなに？

Cloudflare D1（SQLite ベース）の全文検索モジュール FTS5 と `Intl.Segmenter` を組み合わせて、日本語全文検索を実装してみました。さらに Web Component として切り出して、任意のサイトに 2 行で埋め込めるようにしています。

:::message
動くデモを公開しています。実際に検索を試せます。
https://www.techtalk.jp/demo/db/fts
:::

## 2 テーブル構成で原文と検索インデックスを分離する

FTS5 仮想テーブルにはトークン化済みのテキストを格納し、原文は別テーブルに保持します。`fts_index` の `rowid` と `fts_contents` の `id` を一致させることで、検索結果から原文を JOIN で取得する仕組みです。

```sql
CREATE TABLE IF NOT EXISTS fts_contents (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  title TEXT NOT NULL,
  body TEXT NOT NULL,
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE VIRTUAL TABLE IF NOT EXISTS fts_index USING fts5(title, body);
```

登録時は、原文を `fts_contents` に INSERT したあと、`Intl.Segmenter` でトークン分割したテキストを `fts_index` に INSERT します。検索時は、クエリをトークン化して `fts_index` を MATCH 検索し、`rowid` で原文を JOIN して返します。

## 8 行で日本語トークン化 - Intl.Segmenter の実装

`Intl.Segmenter` は V8 エンジンに組み込まれた API で、Cloudflare Workers でもそのまま使えます。外部ライブラリは不要です。

```typescript
const segmenter = new Intl.Segmenter('ja', { granularity: 'word' })

export function tokenize(text: string): string {
  return [...segmenter.segment(text)]
    .filter((s) => s.isWordLike)
    .map((s) => s.segment)
    .join(' ')
}
```

たとえば「Cloudflare Workers はエッジで動作するサーバーレスプラットフォームです」を入力すると、以下のようにトークン化されます。

```
Cloudflare | Workers | は | エッジ | で | 動作 | する | サーバー | レス | プラットフォーム | です
```

`isWordLike` フィルタで助詞（は、で）なども含まれますが、格納側と検索側で同じトークン化をするので問題ありません。

## FTS5 MATCH クエリと BM25 ランキング

検索クエリもトークン化し、各トークンをダブルクオートで囲んで完全一致にします。FTS5 の `MATCH` 演算子はデフォルトで AND 検索になるので、複数トークンすべてを含む結果だけが返ります。`rank` は BM25 スコア（低いほど関連度が高い）です。

```typescript
const matchQuery = tokens
  .split(' ')
  .filter(Boolean)
  .map((t) => `"${t.replaceAll('"', '""')}"`)
  .join(' ')

const result = await sql`
  SELECT c.id, c.title, c.body, c.created_at as "createdAt", f.rank
  FROM fts_index f
  JOIN fts_contents c ON c.id = f.rowid
  WHERE fts_index MATCH ${matchQuery}
  ORDER BY f.rank
  LIMIT 20
`.execute(db)
```

登録・削除の実装もシンプルですが、D1 は 2026 年 3 月時点でトランザクションをサポートしていないため、`fts_contents` と `fts_index` の整合性はアプリケーション側で担保する必要があります。

## 「サーバレス」で検索できない理由 - 表記揺れの壁

`Intl.Segmenter` は格納時と検索時で同じトークン化をするので、「ベストプラクティス」のような複合語がバラバラに分割されても AND 検索で正しくヒットします。「推し活」のように 1 文字ずつに分解されるケースでも同様です。

一方、カタカナの長音記号（ー）の有無が異なると検索に失敗します。「サーバレス」で検索しても格納側の「サーバーレス」とトークンが一致しません。「プロパティー」と「プロパティ」、「インタフェース」と「インターフェース」も同じ問題です。

対策としては、格納時・検索時の両方で正規化の前処理を入れることで対応できます。

```typescript
function normalize(text: string): string {
  return text.replace(/([ァ-ヴ])ー+/g, '$1')
}
```

ただし「ラーメン」が「ラメン」になるような意図しない変換が起きないよう、正規化ルールの設計は慎重に行う必要があります。

## D1 のスケーラビリティとコスト

D1 の最大データベースサイズは 10 GB です。FTS5 のインデックスも SQLite ファイル内に格納されるため、原文の約 2〜3 倍のストレージを消費すると見積もると、ブログ記事（3,000 字）で約 50〜70 万件、短文（150 字）なら約 1,300 万件を格納できます。

D1 は **Workers Free プラン（$0/月）でも利用可能** です。Free プランでは 5 GB のストレージと 500 万行/日の読み取りが使えます。1 回の検索で 20 行読むとしても 1 日 25 万回検索できる計算なので、個人開発レベルなら十分です。

Paid プラン（$5/月〜）にスケールアップすると、250 億行/月の読み取りと 5,000 万行/月の書き込みが使えます。10 GB フル利用でも月額約 $8.75 程度です。

Algolia（Grow プランで 1,000 検索あたり $0.50）や Meilisearch Cloud（$30〜/月）、Elasticsearch Service（$95〜/月）と比べると、D1 FTS5 はコストの面で大きなアドバンテージがあります。検索品質（ファジーマッチ、タイポ耐性、ファセット検索など）では専用サービスに劣りますが、シンプルなキーワード検索を低コストで提供したいケースに向いています。

## 制限事項

FTS5 + `Intl.Segmenter` の組み合わせにはいくつか制限があります。カタカナ長音の表記揺れには正規化の前処理が必要です。D1 はトランザクション未サポートのため 2 テーブル間の整合性に注意が必要で、1 行あたりの最大サイズは 1 MB です。`Intl.Segmenter` のトークン化精度は V8 の辞書に依存するため、環境によって結果が異なる可能性があります。また FTS5 は完全一致ベースなので、タイポ耐性や類義語検索が必要な場合は別途実装が必要です。

## 2 行で埋め込み - Web Component として切り出す

ここまでの全文検索機能を Web Component として切り出しました。`<fts-search>` を読み込むだけで、任意のサイトに検索 UI を埋め込めます。

CORS 対応の検索 API（`GET /demo/api/fts?q=キーワード&limit=20`）が `Access-Control-Allow-Origin: *` を返すので、どのドメインからでもフェッチできます。

埋め込みコードはたった 2 行です。

```html
<script src="https://www.techtalk.jp/fts-search.js"></script>
<fts-search api-url="https://www.techtalk.jp/demo/api/fts"></fts-search>
```

Shadow DOM でスタイルを内包しているのでホストサイトの CSS に影響されず、フレームワークも不要です。入力後 300ms でデバウンス検索し、リクエストのキャンセル処理もついています。ビルド後 4.6 KB（gzip 1.8 KB）と軽量で、CSS Custom Properties で外部からスタイルをカスタマイズできます。

```html
<style>
  fts-search {
    --fts-font-size: 16px;
    --fts-border-color: #e5e7eb;
    --fts-focus-color: #6366f1;
    --fts-item-bg: #fafafa;
  }
</style>
```

`api-url`（必須）、`placeholder`（デフォルト「検索キーワードを入力...」）、`limit`（デフォルト 20）の 3 つの属性で設定できます。

### 1 つの Workers で複数サービスの検索をまかなう

このアプローチの面白いところは、自分の Cloudflare Workers にデプロイするだけで、複数の個人開発サービスに全文検索を提供できることです。

```
[サービスA] ──→ /api/fts?q=... ──→ [Cloudflare Workers + D1]
[サービスB] ──→ /api/fts?q=... ──→       (共通の検索基盤)
[ブログ]    ──→ /api/fts?q=... ──→
```

Workers Free プランでも D1 は 5 GB・500 万行読み取り/日まで使えるので、個人開発の規模なら無料で全文検索を提供できます。検索 UI は Web Component を `<script>` で読み込むだけ。バックエンドもフロントエンドもゼロコンフィグです。

## まとめ

Cloudflare D1 の FTS5 + `Intl.Segmenter` による日本語全文検索は、小〜中規模のユースケースであれば十分実用的です。Free プランなら $0/月で、`Intl.Segmenter` による日本語トークン化は 8 行で実装できます。Web Component で任意のサイトに埋め込めるので、個人開発サービスに全文検索を追加したい方にはおすすめです。

Algolia や Meilisearch のような高機能な検索が必要な場面には向きませんが、「自分のサービスにシンプルな検索をつけたい」「コストをかけたくない」というケースには良い選択肢だと思います。

デモで実際に試せます。Web Component の埋め込みプレビューもあります。
https://www.techtalk.jp/demo/db/fts

## 参考

- [Cloudflare D1 FTS（サイボウズ フロントエンド）](https://zenn.dev/cybozu_frontend/articles/cloudflare-d1-fts) - 本記事の元ネタ
- [SQLite FTS5 Extension](https://www.sqlite.org/fts5.html)
- [Intl.Segmenter - MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Intl/Segmenter)
- [Cloudflare D1 Pricing](https://developers.cloudflare.com/d1/pricing/)
