---
title: "bq コマンドは成功するのに Python で 403？GCP 認証の仕組みを理解する"
emoji: "🔐"
type: "tech"
topics: ["GCP", "BigQuery", "Python", "認証", "トラブルシューティング"]
published: true
---

# これはなに？

BigQueryをPythonで使おうとしたら「403 Forbidden」エラーが出たんですが、`bq`コマンドだと問題なく動くという謎の現象に遭遇しました。調べてみたら、GCPの認証方式が2種類あって、それぞれ権限チェックの厳格さが違うことが原因でした。

同じような問題で困っている人向けに、原因と解決策をまとめます。

# 問題: 謎の 403 Forbidden エラー

こんな状況でした：

```bash
# bq コマンドは成功
$ bq query "SELECT COUNT(*) FROM \`my-project.production.users\`"
+-------+
| count |
+-------+
| 12500 |
+-------+
# 問題なく動作
```

```python
# Python は失敗
from google.cloud import bigquery

client = bigquery.Client(project='my-project')
df = client.query("SELECT COUNT(*) FROM production.users").to_dataframe()

# エラーが発生
google.api_core.exceptions.Forbidden: 403 POST https://bigquery.googleapis.com/...
Caller does not have required permission to use project my-project.
Grant the caller the roles/serviceusage.serviceUsageConsumer role...
```

**同じプロジェクト、同じアカウント、なのに片方だけエラーが出る…？**

---

# 結論だけ知りたい人向け

## 問題の本質

GCPには2つの認証方式があって、それぞれ異なる権限チェックが行われます：

| 方式 | 使用ツール | 保存場所 | 権限チェック |
|------|-----------|---------|------------|
| **CLI認証** | `gcloud`, `bq`, `gsutil` | `~/.config/gcloud/` 配下（例: `credentials.db`） | `gcloud config set billing/quota_project` が利用される |
| **ADC** | Python SDK | `~/.config/gcloud/application_default_credentials.json` | `quota_project_id` が必須 |

## 解決策

まず ADC の quota project を指定して権限を揃えます：

```bash
gcloud auth application-default set-quota-project my-project
```

それでも権限差異が残る場合の一時的な回避策として、Python から gcloud のトークンを借用します：

```python
import subprocess
from google.cloud import bigquery
from google.oauth2 import credentials as oauth2_credentials

def get_bigquery_client():
    # gcloud のトークンを取得
    result = subprocess.run(
        ['gcloud', 'auth', 'print-access-token'],
        capture_output=True,
        text=True,
        check=True
    )
    access_token = result.stdout.strip()

    # そのトークンで認証
    creds = oauth2_credentials.Credentials(token=access_token)
    return bigquery.Client(project='your-project', credentials=creds)

# 使用
client = get_bigquery_client()
df = client.query("SELECT 1").to_dataframe()
```

---

# GCP の 2 つの認証方式

## 1. CLI 認証（gcloud auth login）

コマンドラインツール用の認証方式です。

```bash
$ gcloud auth login
Your browser has been opened to visit: https://accounts.google.com/...

Credentials saved to file: [~/.config/gcloud/credentials.db]
```

### 特徴

この認証方式は `gcloud`, `bq`, `gsutil` などのCLIツールで使われます。開発者が直接コマンドを実行する際に利用され、認証情報は `~/.config/gcloud/` 配下に保存されます。

### 確認方法

```bash
$ gcloud auth list
               Credentialed Accounts
ACTIVE  ACCOUNT
*       your.email@example.com

$ gcloud config list
[core]
account = your.email@example.com
project = my-project
```

## 2. ADC (Application Default Credentials)

アプリケーション（Python等）用の認証方式です。

```bash
$ gcloud auth application-default login
Your browser has been opened to visit: https://accounts.google.com/...

Credentials saved to file: [~/.config/gcloud/application_default_credentials.json]
```

### 特徴

この認証方式は Google Cloud の Python、Node.js、Go などの SDK で使われます。プログラムが自動で実行する際に利用され、認証情報は `~/.config/gcloud/application_default_credentials.json` に保存されます。

### 確認方法

```bash
$ cat ~/.config/gcloud/application_default_credentials.json
{
  "client_id": "123456789012-...",
  "client_secret": "ABC123...",
  "quota_project_id": "my-project",  # ← これが重要
  "refresh_token": "1//0e...",
  "type": "authorized_user"
}
```

### Quota プロジェクトの設定

ADC で利用する quota project は `gcloud auth application-default set-quota-project <PROJECT_ID>` で更新できます。これを実行するには指定したプロジェクトに対する `serviceusage.services.use` 権限（例: `roles/serviceusage.serviceUsageConsumer`）が必要です。

---

# 鍵は quota_project_id にあった

## Quota とは何か

**Quota（クォータ）** はプロジェクトごとに割り当てられた API 使用量の上限のことです。

例えばBigQueryでは、1日あたり 100 TB までのスキャン、1秒あたり 100 リクエスト、同時実行クエリ 100 個までといった制限があります。これは有限のリソースなので、誰でも勝手に使えるわけではありません。

## quota_project_id の意味

```json
{
  "quota_project_id": "my-project"
}
```

これは「`my-project` プロジェクトの quota（使用枠）を消費します」という宣言です。

## なぜ権限が必要なのか

もし誰でも他人のプロジェクトの quota を使えてしまったら、こんなことが起きます：

```text
悪意あるユーザー
  ↓ quota_project_id 指定
my-project
  ↓ 大量リクエスト
Quotaを使い切る
  ↓
本当に使いたい人が使えなくなる
```

DoS攻撃が可能になってしまうわけです。

だから、GCPは厳格にチェックしています：

```text
quota_project_id を指定するには
↓
serviceusage.services.use 権限が必要
↓
roles/serviceusage.serviceUsageConsumer ロールを付与
```

---

# 課金 vs Quota の違い

ここ、混乱しやすいポイントなので整理します。

## 課金（Billing）

**データスキャン量に対する料金**

```python
client = bigquery.Client(project='my-project')  # ← これで決まる
df = client.query("SELECT * FROM users").to_dataframe()
# → my-project プロジェクトに課金
```

- `project=` パラメータで決まる
- BigQueryのデータがどのプロジェクトにあるかで決まる

## Quota（使用量制限）

**API呼び出し回数の制限**

```json
{
  "quota_project_id": "my-project"  # ← これで決まる
}
```

- `quota_project_id` で決まる
- 「誰の使用枠を消費するか」の指定

## 比較表

| 項目 | 課金 | Quota |
|------|------|-------|
| **対象** | データスキャン量 | API呼び出し回数 |
| **決定要因** | `project=` パラメータ | `quota_project_id` |
| **必要な権限** | BigQuery データ読み取り権限 | `serviceusage.services.use` |
| **目的** | 料金の支払先 | リソース枠の管理 |

:::message
課金先と quota 消費先は別の概念です。混同しやすいので注意してください。
:::

---

# なぜ CLI 認証は動いたのか

## CLI認証のquota管理

```bash
bq query "SELECT * FROM table"
```

CLIツールは `gcloud config get-value billing/quota_project` の設定値を quota project として送信します。設定していない場合は Google 管理プロジェクト（`764086051850`）が使われるため、実務では `gcloud config set billing/quota_project my-project` で明示的に指定しておくと安心です。

## ADCの厳格なチェック

```python
client = bigquery.Client(project='my-project')
```

一方、ADCを使う場合は `application_default_credentials.json` から `quota_project_id` を読み取り、その project を使う権限があるかを厳格にチェックします。権限がなければ 403 エラーになります。

## なぜ差があるのか

CLI認証でも quota project に対する `serviceusage.services.use` 権限が必要なのは同じです。一方 ADC では `application_default_credentials.json` の `quota_project_id` がそのまま使われるため、こちらを忘れると権限エラーが発生しやすいという違いがあります。

---

# 解決策: Python から gcloud 認証を使う

## 実装

```python
import subprocess
from google.cloud import bigquery
from google.oauth2 import credentials as oauth2_credentials


def get_bigquery_client(project: str):
    """
    gcloud CLI の認証を使って BigQuery クライアントを作成

    Args:
        project: BigQueryプロジェクトID

    Returns:
        bigquery.Client インスタンス
    """
    # gcloud からアクセストークンを取得
    result = subprocess.run(
        ['gcloud', 'auth', 'print-access-token'],
        capture_output=True,
        text=True,
        check=True
    )
    access_token = result.stdout.strip()

    # トークンから認証情報を作成
    creds = oauth2_credentials.Credentials(token=access_token)

    # BigQuery クライアントを作成
    return bigquery.Client(project=project, credentials=creds)


# 使用例
client = get_bigquery_client('my-project')
df = client.query("""
    SELECT COUNT(*) as count
    FROM `my-project.production.users`
""").to_dataframe()

print(df)
# ✅ 動きます！
```

## なぜこれで動くのか

```text
Python
  ↓ subprocess
gcloud auth print-access-token
  ↓ CLI認証のトークン
oauth2_credentials.Credentials
  ↓ quota_project不要
BigQuery Client
  ↓ クエリ実行
成功
```

Pythonが「CLI認証を借りている」状態になるわけです。

:::message alert
このアクセストークンは数分〜1時間程度で失効します。長時間のバッチやCI/CDでは、サービスアカウントまたは ADC + `set-quota-project` を使った認証に切り替えてください。
:::

---

# 課金はどこに行く？

重要な疑問：この方法で実行した時、課金はどこに行くのか？

## 答え: project= で指定したプロジェクト

```python
client = bigquery.Client(project='my-project', credentials=creds)
# ↑ my-project に課金されます
```

個人アカウントには課金されないので安心してください。

## 確認方法

```bash
# 最近のジョブ履歴を確認
$ bq ls -j -n 5 my-project

jobId                      Job Type    State      Start Time
ab1234cd-...               query       SUCCESS    18 Oct 19:34
bqjob_r1e5226698...        query       SUCCESS    18 Oct 19:33
```

→ `my-project` プロジェクトに記録されている = 課金先も同じ

---

# 本番環境での推奨構成

## 開発環境（今回の方法）

```python
# 開発者の gcloud 認証を使用
client = get_bigquery_client('project-id')
```

この方法は開発者個人のマシンで、権限管理が複雑でない場合に適しています。

## 本番環境（推奨）

```python
# サービスアカウントを使用
from google.cloud import bigquery

# 環境変数でサービスアカウントキーを指定
# export GOOGLE_APPLICATION_CREDENTIALS="path/to/service-account-key.json"
client = bigquery.Client(project='project-id')
```

本番環境ではサービスアカウントを使うことをおすすめします。アプリケーション専用の権限を設定でき、個人アカウントと分離されるためセキュリティ面でも安全です。また、誰が何をしたか追跡しやすく、GitHub Actions などの CI/CD 環境でも使いやすいというメリットがあります。

---

# まとめ

GCPには CLI認証（gcloud auth login）と ADC（gcloud auth application-default login）という2つの認証方式があり、それぞれ権限チェックの厳格さが異なります。今回の 403 エラーは、ADC の `quota_project_id` に対する厳格な権限チェックが原因でした。

quota_project_id は課金先とは別の概念で、「誰の quota 枠を使うか」を指定するものです。そのため、指定したプロジェクトを使う権限が厳格にチェックされます。

開発環境では `gcloud auth print-access-token` を使って CLI 認証を Python で利用することで回避できます。ただし、本番環境ではサービスアカウントを使うことをおすすめします。

## トラブルシューティングチェックリスト

403 エラーが出たら、この順番で確認してみてください：

- [ ] `bq` コマンドは動くか？
- [ ] `gcloud auth list` で正しいアカウントがアクティブか？
- [ ] `application_default_credentials.json` に `quota_project_id` があるか？
- [ ] その project を使う権限があるか？

## 参考リンク

- [Application Default Credentials](https://cloud.google.com/docs/authentication/application-default-credentials)
- [Service Usage API](https://cloud.google.com/service-usage/docs)
- [BigQuery Python Client](https://cloud.google.com/python/docs/reference/bigquery/latest)
