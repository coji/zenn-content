---
title: "dbt初心者が手を動かして学ぶ！DuckDBで作るデータパイプライン入門"
emoji: "🔨"
type: "tech"
topics: ["dbt", "duckdb", "dataengineering", "sql"]
published: true
---

## これはなに?

dbtとDuckDBを使ってデータパイプラインを作ってみました。CSVから顧客別の購入サマリーを集計するまでの手順を、初心者向けに解説します。

## はじめに

dbt（data build tool）を触ったことがなかったため、Claude Codeの学習モードを使って一からチュートリアルをやってみました。

この記事では、dbtの基本的な概念を**実際に手を動かしながら**学んでいく過程を紹介します。

### この記事で学べること

- dbtプロジェクトの基本構造
- Seeds（CSVデータのロード）
- Staging層とMart層の作り方
- `{{ source() }}` と `{{ ref() }}` の使い方
- 依存関係の自動解決

### 環境

- dbt-core: 1.10.13
- dbt-duckdb: 1.9.6
- DuckDB: 1.4.0
- uv（Pythonパッケージマネージャー）

## 環境構築

### 1. uvのインストール

まず、高速なPythonパッケージマネージャー `uv` をインストールします：

```bash
# macOS/Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# またはHomebrewで
brew install uv
```

Windowsの場合は[公式サイト](https://github.com/astral-sh/uv)を参照してください。

### 2. プロジェクトのセットアップ

プロジェクトディレクトリを作成し、必要なパッケージをインストール：

```bash
# プロジェクトディレクトリ作成
mkdir hello-dbt
cd hello-dbt

# pyproject.tomlを作成
cat > pyproject.toml << 'EOF'
[project]
name = "hello-dbt"
version = "0.1.0"
dependencies = [
    "dbt-core>=1.10.0",
    "dbt-duckdb>=1.9.6",
]
requires-python = ">= 3.11"
EOF

# 依存関係をインストール
uv sync
```

これで `dbt-core` と `dbt-duckdb` がインストールされ、すぐに使える状態になります。

### 3. dbtコマンドの実行方法

この記事では `uv run dbt` という形式でdbtコマンドを実行します：

```bash
uv run dbt --version
# dbt-core: 1.10.13
# dbt-duckdb: 1.9.6
```

`uv run` を付けることで、仮想環境を自動的に有効化してコマンドを実行できます。

## dbtプロジェクトのセットアップ

まず、プロジェクト構造を作成します。

```bash
mkdir -p myproject/{models/{staging,marts},seeds,tests,macros,snapshots,analyses}
```

### dbt_project.yml

プロジェクトの設定ファイルを作成：

```yaml
name: 'myproject'
version: '1.0.0'
config-version: 2

profile: 'myproject'

model-paths: ["models"]
seed-paths: ["seeds"]
test-paths: ["tests"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

clean-targets:
  - "target"
  - "dbt_packages"
```

### ~/.dbt/profiles.yml

データベース接続設定：

```yaml
myproject:
  outputs:
    dev:
      type: duckdb
      path: dev.duckdb
      threads: 1

  target: dev
```

動作確認

```bash
cd myproject
uv run dbt debug
```

`All checks passed!` と表示されればOKです。

## ステップ1: Seedsでサンプルデータを作成

dbtでは `seeds/` ディレクトリにCSVファイルを置くことで、開発用のデータをロードできます。

### seeds/raw_customers.csv

```csv
customer_id,first_name,last_name,email
1,John,Doe,john@example.com
2,Jane,Smith,jane@example.com
3,Bob,Johnson,bob@example.com
4,Alice,Williams,alice@example.com
```

### seeds/raw_orders.csv

```csv
order_id,customer_id,order_date,amount
101,1,2025-01-15,250
102,2,2025-01-16,450
103,3,2025-01-17,300
104,4,2025-01-18,150
105,2,2025-01-19,500
106,1,2025-01-20,700
107,3,2025-01-21,200
108,4,2025-01-22,350
109,1,2025-01-23,400
```

CSVをDuckDBにロード：

```bash
uv run dbt seed
```

```
Found 2 seeds, 444 macros
1 of 2 OK loaded seed file main.raw_customers ... [INSERT 4 in 0.04s]
2 of 2 OK loaded seed file main.raw_orders ...... [INSERT 9 in 0.01s]
Done. PASS=2 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=2
```

データが入ったか確認：

```bash
duckdb dev.duckdb
```

```sql
SELECT * FROM raw_customers;
```

### 💡 学んだこと

- `dbt seed` はCSVをテーブルとして作成する
- スキーマ変更時は `--full-refresh` で再作成が必要
- 本番環境では使わず、開発用の参照データに便利

## ステップ2: Sourceの定義

`{{ source() }}` を使う前に、ソーステーブルをdbtに教える必要があります。

### models/schema.yml

```yaml
version: 2

sources:
  - name: main
    tables:
      - name: raw_customers
      - name: raw_orders
```

`name: main` はDuckDBのスキーマ名です。

### 💡 学んだこと

- `schema.yml` でソースを定義しないと `{{ source() }}` が使えない
- dbtはこのファイルを読んで、データの依存関係を理解する

## ステップ3: Stagingモデルの作成

Staging層では、生データをクリーンアップします。

### models/staging/stg_customers.sql

```sql
-- ステージング: 顧客データ
SELECT
  customer_id,
  first_name,
  last_name,
  email
FROM
  {{ source('main', 'raw_customers') }}
```

### models/staging/stg_orders.sql

```sql
-- ステージング: 注文データ
SELECT
  order_id,
  customer_id,
  order_date,
  amount
FROM
  {{ source('main', 'raw_orders') }}
```

モデルを実行：

```bash
uv run dbt run
```

```
Concurrency: 1 threads (target='dev')

1 of 2 START sql view model main.stg_customers ... [RUN]
1 of 2 OK created sql view model main.stg_customers [OK in 0.06s]
2 of 2 START sql view model main.stg_orders ....... [RUN]
2 of 2 OK created sql view model main.stg_orders ... [OK in 0.01s]

Done. PASS=2 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=2
```

DuckDBでスキーマ確認：

```bash
duckdb dev.duckdb -c ".schema"
```

```sql
CREATE TABLE raw_customers(...);
CREATE TABLE raw_orders(...);
CREATE VIEW stg_customers AS SELECT ... FROM dev.main.raw_customers;
CREATE VIEW stg_orders AS SELECT ... FROM dev.main.raw_orders;
```

### 💡 学んだこと

- dbtモデルはSELECT文を書くだけ
- デフォルトでVIEWとして作成される
- `{{ source('schema', 'table') }}` で生データを参照
- Staging層でカラム名の統一や型変換を行う

## ステップ4: Martモデルの作成

Mart層では、複数のステージングモデルをJOINしてビジネス分析用のデータを作ります。

### models/marts/customer_summary.sql

```sql
-- マートモデル: customer_summary.sql
SELECT
  c.customer_id,
  c.first_name || ' ' || c.last_name as full_name,
  c.email,
  COUNT(o.order_id) as total_orders,
  SUM(o.amount) as total_spent
FROM
  {{ ref('stg_customers') }} as c
LEFT JOIN
  {{ ref('stg_orders') }} as o
ON
  c.customer_id = o.customer_id
GROUP BY
  c.customer_id, c.first_name, c.last_name, c.email
```

モデルを実行：

```bash
uv run dbt run
```

```
Found 3 models, 2 seeds, 2 sources, 444 macros

1 of 3 START sql view model main.stg_customers ..... [RUN]
1 of 3 OK created sql view model main.stg_customers  [OK in 0.04s]
2 of 3 START sql view model main.stg_orders ........ [RUN]
2 of 3 OK created sql view model main.stg_orders ... [OK in 0.01s]
3 of 3 START sql view model main.customer_summary .. [RUN]
3 of 3 OK created sql view model main.customer_summary [OK in 0.01s]

Done. PASS=3 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=3
```

結果を確認：

```bash
duckdb dev.duckdb -c "SELECT * FROM customer_summary ORDER BY customer_id;"
```

```
┌─────────────┬────────────────┬──────────────┬─────────────┐
│ customer_id │   full_name    │ total_orders │ total_spent │
├─────────────┼────────────────┼──────────────┼─────────────┤
│           1 │ John Doe       │            3 │        1350 │
│           2 │ Jane Smith     │            2 │         950 │
│           3 │ Bob Johnson    │            2 │         500 │
│           4 │ Alice Williams │            2 │         500 │
└─────────────┴────────────────┴──────────────┴─────────────┘
```

### 💡 学んだこと

- `{{ ref('model_name') }}` で他のdbtモデルを参照
- dbtが依存関係を自動解決して正しい順序で実行
- JOINする時はテーブルエイリアスを使う（`c.customer_id = o.customer_id`）
- Mart層でビジネスロジック（集計・JOIN）を実装

## dbtの依存関係の仕組み

実行ログを見ると、dbtが依存関係を理解していることがわかります：

```
1 of 3: stg_customers  ← まず顧客データ
2 of 3: stg_orders     ← 次に注文データ
3 of 3: customer_summary ← 最後に集計（両方を使うから）
```

`{{ ref() }}` を解析して、自動的にDAG（有向非巡回グラフ）を構築しています。

## データパイプラインの全体像

```
[Seeds - Bronze層]
├── raw_customers.csv  → raw_customers (TABLE)
└── raw_orders.csv     → raw_orders (TABLE)
          ↓ {{ source() }}
[Staging - Silver層]
├── stg_customers.sql  → stg_customers (VIEW)
└── stg_orders.sql     → stg_orders (VIEW)
          ↓ {{ ref() }}
[Marts - Gold層]
└── customer_summary.sql → customer_summary (VIEW)
```

これは「メダリオンアーキテクチャ」と呼ばれる設計パターンです：

- **Bronze層（Seeds）**: 生データをそのまま保存
- **Silver層（Staging）**: データをクリーンアップ・標準化
- **Gold層（Marts）**: ビジネス分析用に加工

## よくあるトラブルと解決法

### 1. ソースが見つからない

```
Error: depends on a source named 'main.raw_customers' which was not found
```

→ `models/schema.yml` でソースを定義する

### 2. カラムが見つからない

```
Error: Referenced column "order_date" not found!
Candidate bindings: "order_data", ...
```

→ CSVのカラム名とSQLが一致しているか確認
→ スキーマ変更時は `dbt seed --full-refresh` で再作成

### 3. JOIN条件が曖昧

```sql
-- ❌ NG: どちらのcustomer_idか不明
ON customer_id = customer_id

-- ✅ OK: テーブルエイリアスで明示
ON c.customer_id = o.customer_id
```

## まとめ

dbtを使うことで得られるメリット：

1. **SQLだけでデータパイプラインが作れる** - Pythonコード不要（ランタイムは必要）
2. **依存関係の自動解決** - 実行順序を気にしなくて良い
3. **モジュール化** - 再利用可能な小さいモデルに分割
4. **テスト・ドキュメント機能** - データ品質の担保

### 使ってみた感想

宣言的にSQLを書くだけで、`dbt seed`や`dbt run`で依存関係を解決しながら整ってくれるのはかなり楽でした。

一方で、Jinjaテンプレート記法でSQLを手で書くのは少し大変に感じました。ただ、Claude Codeのようなコーディングエージェントがあれば、この辺りの煩雑さは問題にならないかもしれません。

### 次のステップ

- `dbt test` でデータ品質テストを追加
- `dbt docs generate` でドキュメント自動生成
- `{{ config(materialized='table') }}` でテーブル化してパフォーマンス向上
- incrementalモデルで大規模データに対応

## Claude Codeの学習モードについて

この記事は、Claude Codeの「学習モード」を使って作成しました。

学習モードの特徴：
- **手を動かす学習**: AIが全部やるのではなく、要所で自分でコードを書く
- **段階的な説明**: 一つずつ概念を理解しながら進める
- **実践的なガイド**: なぜそうするのか、どう書けば良いかの指針

特に「`{{ source() }}` を使う前にschema.ymlが必要」「seedの再実行には--full-refreshが必要」など、ドキュメントだけでは気づきにくいポイントを、実際にエラーに遭遇しながら学べたのが良かったです。

## 参考リンク

- [dbt公式ドキュメント](https://docs.getdbt.com/)
- [DuckDB](https://duckdb.org/)
- [uv - Python package manager](https://github.com/astral-sh/uv)
