---
title: "bq コマンドは成功するのに Python で 403？GCP 認証の違いと注意点"
emoji: "🔐"
type: "tech"
topics: ["GCP", "BigQuery", "Python", "認証", "トラブルシューティング"]
published: true
---

# これはなに？

BigQuery を Python で使おうとしたら「403 Forbidden」エラーが出たんですが、`bq` コマンドでは問題なく動く。同じアカウントなのに挙動が違う、という状況に遭遇しました。

調べてみたところ、Google Cloud SDK（CLI）と Python SDK（ライブラリ）では使われる認証方式が異なるため、設定や権限の扱いに差が出ることがわかりました。同じような問題で困っている人向けに、仕組みと原因、解決策をまとめます。

---

# 問題の状況

```bash
# bq コマンドは成功
$ bq query "SELECT COUNT(*) FROM \`my-project.production.users\`"
+-------+
| count |
+-------+
| 12500 |
+-------+
```

```python
# Python は失敗
from google.cloud import bigquery

client = bigquery.Client(project='my-project')
df = client.query("SELECT COUNT(*) FROM production.users").to_dataframe()

# エラー内容
google.api_core.exceptions.Forbidden: 403 POST https://bigquery.googleapis.com/...
Caller does not have required permission to use project my-project.
Grant the caller the roles/serviceusage.serviceUsageConsumer role...
```

---

# 結論だけ知りたい人向け

## 原因の概要

CLI ツール（`gcloud` / `bq`）と Python SDK では、内部的に使う認証情報の種類が違います。
そのため、「どのプロジェクトの quota（使用枠）を消費するか」という設定がずれていると、API 側で 403 Forbidden が発生します。

| 認証方式                                     | 主な利用ツール                    | 認証情報の保存場所                                               | quota の指定方法                               |
| ---------------------------------------- | -------------------------- | ------------------------------------------------------- | ----------------------------------------- |
| **CLI認証**                                | `gcloud`, `bq`, `gsutil`   | `~/.config/gcloud/credentials.db`                       | `gcloud config set billing/quota_project` |
| **ADC（Application Default Credentials）** | Python SDK, Node.js SDK など | `~/.config/gcloud/application_default_credentials.json` | `quota_project_id` フィールド                  |

---

## 対応方法

まず ADC の quota project を明示的に設定します。

```bash
gcloud auth application-default set-quota-project my-project
```

このとき、`my-project` に対して `serviceusage.services.use` 権限（通常は `roles/serviceusage.serviceUsageConsumer`）が必要です。

それでも解決しない場合は、一時的な回避策として CLI トークンを使う方法があります。

```python
import subprocess
from google.cloud import bigquery
from google.oauth2 import credentials as oauth2_credentials

def get_bigquery_client(project):
    token = subprocess.run(
        ['gcloud', 'auth', 'print-access-token'],
        capture_output=True, text=True, check=True
    ).stdout.strip()
    creds = oauth2_credentials.Credentials(token=token)
    return bigquery.Client(project=project, credentials=creds)
```

---

# GCP の認証方式を整理する

## 1. CLI 認証（`gcloud auth login`）

CLI ツール（`bq` など）で利用される方式で、認証情報は `~/.config/gcloud/credentials.db` に保存されます。

```bash
gcloud auth login
```

この方式では、`gcloud config get-value billing/quota_project` の設定値が quota project として利用されます。
設定していない場合は Google が管理するデフォルトプロジェクトが使われることもあります。

---

## 2. ADC（Application Default Credentials）

Python や Node.js の SDK が使う認証方式です。

```bash
gcloud auth application-default login
```

認証情報は `~/.config/gcloud/application_default_credentials.json` に保存されます。

```json
{
  "client_id": "1234567890-...",
  "quota_project_id": "my-project",
  "refresh_token": "1//0e...",
  "type": "authorized_user"
}
```

`quota_project_id` が空のままだと、API 側で quota 消費先を判断できず、エラーや警告が出る場合があります。
Google Cloud の公式ドキュメントにも「クォータプロジェクトを設定していない場合、API リクエストが失敗する可能性がある」と記載があります。

---

# quota_project_id の意味

クォータプロジェクトとは、「この API 呼び出しでどのプロジェクトの使用枠（quota）を使うか」を指定する仕組みです。
課金先プロジェクトとは別の概念で、quota を利用するためにも権限が必要です。

```text
quota_project_id を設定するには
↓
serviceusage.services.use 権限が必要
↓
roles/serviceusage.serviceUsageConsumer ロールを付与
```

この権限がない場合、ADC の認証を通してリクエストすると 403 Forbidden が返ることがあります。

---

# CLI と SDK で挙動が異なる理由

CLI（`bq`）と SDK（`google-cloud-bigquery`）は同じ API を呼びますが、**使う認証情報のソースが異なる**ため、
結果的に quota project や権限チェックの対象が違うことがあります。

CLI では `gcloud` の設定が自動で quota project に反映され、問題が表面化しにくい一方、
ADC は `application_default_credentials.json` の設定をそのまま使うため、欠けているとエラーになります。

---

# 安定した運用方法

開発環境では CLI のトークンを使う方法（`gcloud auth print-access-token`）が手軽です。一方、CI/CD や本番環境ではサービスアカウントキーを利用し、環境変数 `GOOGLE_APPLICATION_CREDENTIALS` で指定することをおすすめします。また、クォータプロジェクトは明示しておくと安心です。

---

# まとめ

今回の 403 Forbidden エラーは「クォータプロジェクトの不整合」が原因でした。ADC では明示設定がないと厳密な権限チェックに引っかかります。

なお、403 エラーには API 未有効化、IAM 権限不足、クォータ超過など他にも複数の原因があるため、実際のトラブルシュートではそれらも合わせて確認してください。

---

# 参考資料

* [Application Default Credentials (ADC)](https://cloud.google.com/docs/authentication/application-default-credentials)
* [Set the quota project (Cloud Docs)](https://cloud.google.com/docs/quotas/set-quota-project)
* [Service Usage API Docs](https://cloud.google.com/service-usage/docs)
* [BigQuery Python Client Library](https://cloud.google.com/python/docs/reference/bigquery/latest)
* [Stack Overflow: Quota Project Warnings](https://stackoverflow.com/questions/72745805/warnings-because-of-user-credentials-without-quota-project)
