---
title: "FastAPI + Prisma で REST API を作る"
emoji: "🐍"
type: "tech"
topics: ["python", "fastapi", "prisma"]
published: true
---

## FastAPI + Prisma で REST API を作る

Python で REST API を作りたい場合、FastAPI と Prisma を組み合わせると非常に効率的に開発を進められます。本記事では、FastAPI と Prisma を使って簡単な REST API を作成する方法を解説します。

## 環境構築

まず、`rye` を使ってプロジェクトを初期化し、必要なパッケージをインストールします。

```bash
rye init fastapi-prisma-test
cd fastapi-prisma-test
rye add fastapi "uvicorn[standard]" prisma
rye sync
rye shell
```

## データベース設定

`.env` ファイルを作成し、データベースの接続 URL を設定します。今回は SQLite を使用します。

```sh:.env
DATABASE_URL=file:data/database.db
```

次に、Prisma のスキーマファイルを `prisma/schema.prisma` に作成します。

```prisma:prisma/schema.prisma
generator client {
  provider = "prisma-client-py"
  recursive_type_depth = 5
}

datasource db {
  provider = "sqlite"
  url = env("DATABASE_URL")
}

model Message {
  id        String   @id @default(uuid())
  text      String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

スキーマファイルを作成したら、マイグレーションを実行します。

```bash
prisma migrate dev
```

## FastAPI アプリケーションの作成

`server.py` ファイルを作成し、FastAPI アプリケーションを実装します。

```python:server.py
from fastapi import FastAPI
from prisma import Prisma

app = FastAPI()
prisma = Prisma()

@app.on_event("startup")
async def startup():
    await prisma.connect()

@app.on_event("shutdown")
async def shutdown():
    await prisma.disconnect()

@app.get("/")
def index():
    return {"message": "Hello, World!"}

@app.post("/message")
async def add_message(message: str):
    return await prisma.message.create(data={"text": message})

@app.get("/message")
async def get_messages(take: int = 10):
    message_count = await prisma.message.count()
    messages = await prisma.message.find_many(take=take)
    return {"message_count": message_count, "messages": messages}

@app.get("/message/{id}")
async def get_message(id: str):
    return await prisma.message.find_first(where={"id": id})
```

ここでは、以下のエンドポイントを実装しています。

- `GET /`: "Hello, World!" メッセージを返します。
- `POST /message`: 新しいメッセージを作成します。
- `GET /message`: メッセージの一覧を取得します。
- `GET /message/{id}`: 特定のメッセージを取得します。

また、FastAPI の `startup` および `shutdown` イベントを使用して、アプリケーションの起動時と終了時に Prisma クライアントの接続と切断を行っています。

## アプリケーションの起動

`pyproject.toml` ファイルに以下のスクリプトを追加します。

```toml:pyproject.toml
[tool.rye.scripts]
start = { cmd = 'uvicorn server:app --reload' }
```

これで、以下のコマンドでアプリケーションを起動できます。

```bash
rye run start
```

`http://localhost:8000/docs` にアクセスすると、Swagger UI で API のドキュメントを確認できます。

## API の使用例

以下は、API の使用例です。

```bash
# メッセージの作成
$ curl -X POST -H "Content-Type: application/json" -d '{"message": "Hello, World!"}' http://localhost:8000/message

# メッセージの一覧取得
$ curl http://localhost:8000/message

# 特定のメッセージの取得
$ curl http://localhost:8000/message/{id}
```

## まとめ

FastAPI と Prisma を使うことで、Python で REST API を簡単に作成できます。Prisma の型安全性と FastAPI の優れたパフォーマンスにより、効率的に開発を進められます。

上記コードを含むリポジトリはこちらです。参考になるとうれしいです。
https://github.com/coji/fastapi-prisma-test

本記事で紹介した方法を参考に、ぜひ自分だけの REST API を作ってみてください。
