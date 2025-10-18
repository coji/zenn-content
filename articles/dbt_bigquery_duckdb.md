---
title: "dbt で BigQuery → DuckDB パイプラインを構築してローカル分析環境を作る"
emoji: "🦆"
type: "tech"
topics: ["dbt", "BigQuery", "DuckDB", "Python", "データ分析"]
published: true
---

# これはなに?

BigQueryのデータをローカルにダウンロードして、課金を気にせず何度でもクエリを実行できる環境を作ってみた話です。dbtとDuckDBを使って、BigQuery → DuckDBのパイプラインを構築しました。

# 背景

BigQueryでデータ分析をしていると、試行錯誤でクエリを何度も実行するたびに課金が気になります。ダッシュボード開発中なんかは特に、同じデータを何度も確認したくなるんですが、毎回BigQueryを叩くのはもったいない。

そこで、BigQueryのデータを定期的にDuckDBにエクスポートして、ローカルで好きなだけ分析できる環境を作ってみました。

最終的にこうなります。

```bash
# BigQueryからデータ取得 → DuckDBに保存
$ dbt run

# ローカルで自由に分析（課金ゼロ！）
$ duckdb data/local_analytics.duckdb
D SELECT * FROM marts.mart_user_activity LIMIT 10;
```

---

# なぜ BigQuery → DuckDB なのか

BigQueryは便利ですが、クエリ実行ごとにスキャン量で課金されるのがネックです。開発中は試行錯誤でクエリを何度も実行するので、コストが気になります。

一方、DuckDBはローカルで動くので課金ゼロ。しかも分析クエリが高速で、SQLiteみたいに1ファイルで完結する軽量さも魅力です。

使い分けはこんな感じです。

```text
BigQuery: 本番データの保存・更新
   ↓ (dbtで定期実行)
DuckDB: ローカルでの分析・ダッシュボード開発
```

---

# 環境構築

## 前提条件

- Python 3.11+
- BigQueryへのアクセス権
- gcloud CLI（認証用）

## パッケージ管理

今回は高速なパッケージマネージャー `uv` を使いました。

```bash
# uvのインストール
curl -LsSf https://astral.sh/uv/install.sh | sh

# プロジェクト作成
mkdir bq_to_duckdb && cd bq_to_duckdb
uv init
```

## pyproject.toml

```toml
[project]
name = "bq-to-duckdb"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "dbt-core>=1.8.0",
    "dbt-duckdb>=1.8.0",
    "google-cloud-bigquery>=3.25.0",
    "duckdb>=1.0.0",
    "pyyaml>=6.0.0",
]

[tool.hatch.build.targets.wheel]
packages = ["models", "lib"]
```

```bash
# 依存関係インストール
uv sync
```

---

# dbt プロジェクト構造

## ディレクトリ構成

```
bq_to_duckdb/
├── dbt_project.yml       # dbt設定
├── profiles.yml          # DB接続設定
├── lib/
│   └── config.py         # 共通設定ライブラリ
├── models/
│   ├── staging/          # BigQueryからデータ取得（Python）
│   │   ├── schema.yml
│   │   ├── stg_users.py
│   │   └── stg_transactions.py
│   └── marts/            # 分析用テーブル（SQL）
│       ├── mart_user_activity.sql
│       └── mart_daily_metrics.sql
└── data/
    └── local_analytics.duckdb  # DuckDBファイル
```

## dbt_project.yml

```yaml
name: 'local_analytics'
version: '1.0.0'
config-version: 2
profile: 'local_analytics'

vars:
  start_date: '2025-10-10'  # データ取得開始日
  bq_project: 'my-project'

models:
  local_analytics:
    staging:
      +materialized: table
      +schema: staging
    marts:
      +materialized: table
      +schema: marts
```

## profiles.yml

```yaml
local_analytics:
  outputs:
    dev:
      type: duckdb
      path: 'data/local_analytics.duckdb'
      threads: 4
  target: dev
```

---

# 2層アーキテクチャ

データフローは2段階に分けました。

```text
BigQuery (クラウド)
   ↓ Python モデル
Staging層 (DuckDB)  -- 生データそのまま
   ↓ SQL モデル
Mart層 (DuckDB)     -- 分析用に加工
```

## なぜ2層に分けるのか

**Staging層**で生データを忠実に保存しておいて、**Mart層**でビジネスロジックを適用して加工します。こうしておくと、ロジックを変更したくなったときもStaging層から再計算できるので便利です。

---

# Staging層: Python モデルで BigQuery からデータ取得

## なぜ Python モデルが必要か

執筆当初に検証した v1.9.6 では、dbt-duckdb が BigQuery から直接読み込む手段を見つけられませんでした。そのため本稿では**dbt Python モデル**を使って、BigQueryからデータを取得し、pandas DataFrameを返してDuckDBに保存する構成を採用しています。

なお、DuckDB にはコミュニティ製の BigQuery 拡張が提供されており、dbt プロファイルで拡張を読み込む手順を追加すれば直接連携できそうな気配があります。ただし、まだ手元で再現検証できていないため、後日あらためて確認してから別途まとめます。

## 設定の一元管理: lib/config.py

まず、プロジェクト全体で使う設定を一元管理するモジュールを作りました。

```python
# lib/config.py
"""dbt_project.yml から設定を読み込む共通モジュール"""

import yaml
from pathlib import Path

# dbt_project.yml を読み込む
project_root = Path(__file__).parent.parent
config_path = project_root / 'dbt_project.yml'

with open(config_path, 'r', encoding='utf-8') as f:
    _project_config = yaml.safe_load(f)


def var(name: str, default=None):
    """
    dbt_project.yml の vars から値を取得

    dbt SQLの {{ var() }} と同じインターフェース
    """
    try:
        return _project_config['vars'][name]
    except KeyError:
        if default is None:
            available_vars = list(_project_config.get('vars', {}).keys())
            raise KeyError(
                f"Variable '{name}' not found in dbt_project.yml vars. "
                f"Available vars: {available_vars}"
            )
        return default
```

:::message
dbt SQL の `{{ var('start_date') }}` と同じAPIにしておくと、一貫性が保てて使いやすいです。
:::

## stg_users.py の実装

```python
# models/staging/stg_users.py
from google.cloud import bigquery
from lib.config import var


def model(dbt, session):
    """
    BigQueryからユーザーデータを取得してDuckDBに保存

    dbt Pythonモデル:
    - BigQueryからデータを取得
    - pandas DataFrameとして返す
    - dbtが自動的にDuckDBに保存
    """
    # dbt_project.yml の vars から取得
    start_date = var('start_date')
    bq_project = var('bq_project')

    # BigQueryクライアントを初期化
    bq_client = bigquery.Client(project=bq_project)

    # クエリを実行
    query = f"""
    SELECT
        id as user_id,
        created_at,
        updated_at,
        status
    FROM `{bq_project}.production.users`
    WHERE created_at >= '{start_date}'
    """

    # DataFrameとして取得
    df = bq_client.query(query).to_dataframe()

    return df  # dbtがDuckDBに保存
```

## 実行

```bash
$ dbt run --select stg_users

Running with dbt=1.10.13
1 of 1 START python table model main_staging.stg_users ...
1 of 1 OK created python table model main_staging.stg_users
```

これで、BigQueryのデータがDuckDBの `main_staging.stg_users` テーブルに保存されました！

---

# Mart層: SQL モデルで分析用テーブルを作成

Staging層のデータを使って、分析用のテーブルを作っていきます。

## mart_user_activity.sql

```sql
-- models/marts/mart_user_activity.sql

-- 存在するユーザーのみを対象としたアクティビティサマリー
with existing_users as (
  select distinct user_id
  from {{ ref('stg_users') }}
),

user_transactions as (
  select t.*
  from {{ ref('stg_transactions') }} t
  inner join existing_users u on t.user_id = u.user_id
),

transaction_summary as (
  select
    user_id,
    count(*) as total_transactions,
    sum(amount) as total_amount,
    sum(bonus_amount) as total_bonus,
    min(created_at) as first_transaction_at,
    max(created_at) as last_transaction_at
  from user_transactions
  group by user_id
)

select
  u.user_id,
  u.created_at as user_created_at,
  coalesce(t.total_transactions, 0) as total_transactions,
  coalesce(t.total_amount, 0) as total_amount,
  coalesce(t.total_bonus, 0) as total_bonus,
  t.first_transaction_at,
  t.last_transaction_at
from {{ ref('stg_users') }} u
left join transaction_summary t on u.user_id = t.user_id
```

## ref() 関数の役割

```sql
{{ ref('stg_users') }}
```

`ref()` 関数を使うことで、依存関係を自動解決してくれます（stg_users → mart_user_activity）。また、テーブル名も正しく解決してくれます（main_staging.stg_users）。

---

# データ品質テスト

dbtの強力な機能の一つが**テスト**です。実際、このテストのおかげでバグを発見できました。

## models/staging/schema.yml

```yaml
version: 2

models:
  - name: stg_users
    description: "2025-10-10以降に入会したユーザー"
    columns:
      - name: user_id
        description: "ユーザーID（プライマリキー）"
        tests:
          - unique
          - not_null

      - name: created_at
        description: "ユーザー登録日時"
        tests:
          - not_null

  - name: stg_transactions
    description: "トランザクション"
    columns:
      - name: user_id
        description: "ユーザーID"
        tests:
          - not_null
          - relationships:
              arguments:
                to: ref('stg_users')
                field: user_id
```

## テスト実行

```bash
$ dbt test

1 of 7 START test unique_stg_users_user_id ........ [RUN]
1 of 7 PASS unique_stg_users_user_id .............. [PASS]
2 of 7 START test not_null_stg_users_user_id ...... [RUN]
2 of 7 PASS not_null_stg_users_user_id ............ [PASS]
3 of 7 START test relationships_stg_charge_transactions_user_id__user_id__ref_stg_users_
3 of 7 PASS relationships .......................... [PASS]

Done. PASS=7 WARN=0 ERROR=0 SKIP=0 TOTAL=7
```

## テストで発見したバグ

実際の開発中、`relationships` テストでこんなエラーが出ました：

```text
FAIL 1234 relationships_stg_transactions_user_id__user_id__ref_stg_users_
Got 1234 results, configured to fail if != 0
```

原因を調べてみると、`stg_users` と `stg_transactions` でフィルタ条件が異なっていました：

```python
# stg_users.py
WHERE created_at >= '2025-10-10'  # ユーザーの登録日

# stg_transactions.py (間違い)
WHERE created_at >= '2025-10-10'  # トランザクションの日付
```

つまり、2025-10-09に登録したユーザーが2025-10-11にトランザクションを行った場合、トランザクションデータはあるのにユーザーデータがない状態になってしまいます。

修正後：

```python
# stg_transactions.py (正しい)
WHERE user_id IN (
  SELECT id
  FROM `{bq_project}.production.users`
  WHERE created_at >= '{start_date}'
)
```

:::message alert
テストがなければこのデータ不整合に気づかなかったはずです。テスト駆動開発の重要性を実感しました。
:::

---

# 実行スクリプト

全体を実行するスクリプトを用意しました。

## run.sh

```bash
#!/bin/bash
set -e

source .venv/bin/activate
export DBT_PROFILES_DIR=$(pwd)

echo "Step 1: Running staging models (fetch from BigQuery)..."
dbt run --select staging

echo "Step 2: Running mart models (materialize in DuckDB)..."
dbt run --select marts

echo "Step 3: Running tests..."
dbt test

echo "Export completed successfully!"
echo "DuckDB file location: data/local_analytics.duckdb"
```

```bash
bash run.sh
```

---

# 分析してみる

DuckDBに接続して分析してみます。

```bash
duckdb data/local_analytics.duckdb
```

```sql
-- ユーザーアクティビティサマリー
SELECT
  COUNT(*) as total_users,
  COUNT(first_transaction_at) as active_users,
  SUM(total_transactions) as total_transactions,
  SUM(total_amount) as total_revenue
FROM main_marts.mart_user_activity;

┌─────────────┬──────────────┬────────────────────┬───────────────┐
│ total_users │ active_users │ total_transactions │ total_revenue │
├─────────────┼──────────────┼────────────────────┼───────────────┤
│       12500 │          380 │                450 │       1250000 │
└─────────────┴──────────────┴────────────────────┴───────────────┘
```

課金を気にせず、何度でもクエリを実行できます！

---

# ベストプラクティス

## 1. 設定の一元管理

```python
# 悪い例: 各ファイルにハードコード
start_date = "2025-10-10"

# 良い例: dbt_project.yml で一元管理
start_date = var('start_date')
```

## 2. 2層アーキテクチャ

```text
Staging: 生データを忠実に保存
Mart: ビジネスロジックで加工
```

分離することで、ビジネスロジック変更時もStagingから再計算できます。

## 3. テストファースト

モデルを書いたらテストを追加しておくと安心です。

```yaml
tests:
  - unique
  - not_null
  - relationships
```

## 4. ドキュメント化

```yaml
description: "2025-10-10以降に入会したユーザー"
```

将来の自分・チームメンバーのために。

---

# まとめ

dbt と DuckDB を組み合わせることで、BigQueryのデータをローカルにエクスポートし、課金を気にせず何度でもクエリを実行できる開発環境を作ることができました。テストでデータ品質を保証し、設定を一元管理することで保守性も向上しています。

ローカルで自由にデータ分析できる環境はかなり便利で、ダッシュボード開発やデータ探索がはかどるようになりました。

## 参考リンク

- [dbt Documentation](https://docs.getdbt.com/)
- [DuckDB](https://duckdb.org/)
