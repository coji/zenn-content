---
title: "Cloudflare Workers/D1 と React Router v7 で作る Slack 感情ログアプリ「grr」"
emoji: "😠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cloudflare", "react", "reactrouter", "slack", "typescript", "kysely", "d1", "vite"]
published: true
---

## はじめに

こんにちは！この記事では、私が開発したSlackアプリ「grr」（グルル）を紹介します。日々のちょっとしたイライラをSlack上で簡単に記録・共有できるWebアプリケーションです。

![grr](/images/grr-slack-app-on-cloudflare/grr.png)

このアプリは、以下の技術スタックを採用して開発しました。

- **ランタイム:** Cloudflare Workers
- **データベース:** Cloudflare D1
- **フレームワーク:** React Router v7
- **DB クライアント:** Kysely (D1 Dialect)
- **Slack 連携:** slack-edge / slack-cloudflare-workers
- **UI:** React v19 / Tailwind CSS v4
- **その他:** TypeScript, pnpm, Biome, prettier

この記事では、「grr」の概要や機能を紹介するとともに、上記の技術スタックを選定した理由、特に **Cloudflare Workers/D1** と **React Router v7** を組み合わせた開発で得られた知見や実装のポイントについて解説します。

Cloudflare Workers や D1、React Router v7 を使ったアプリケーション開発に興味がある方の参考になれば幸いです。

## アプリ「grr」について

### 開発の背景・動機

私たちは日々の業務や生活の中で、大小さまざまな「イラッ」とする瞬間に遭遇します。多くの場合、それはすぐに忘れてしまったり、誰かに愚痴ってスッキリしたりしますが、言語化して記録することで、自分の感情の傾向を客観視したり、チーム内で共有することでストレスの原因を特定したりするきっかけになるのではないかと考えました。

そこで、「もっと気軽に、Slack上でイライラを記録・共有できる仕組み」として「grr」を開発しました。名前の由来は、もちろん「grr...」という唸り声です 😠

### 主な機能

「grr」には、主に以下の機能があります。

1. **Slack から簡単記録:**
    - `/grr [イラっとしたこと]` のスラッシュコマンドで記録を開始できます。
    - 既存のSlackメッセージに対して、メッセージショートカット（メッセージメニューから「grr」を選択）で記録を開始できます。

    ![スラッシュコマンド](/images/grr-slack-app-on-cloudflare/slash-command.png)

    ![登録完了メッセージ](/images/grr-slack-app-on-cloudflare/message.png)

2. **イライラ度の設定:**
    - 記録時にモーダルが開き、1〜5段階でイライラ度を設定できます。
    - メッセージ内容も編集できます。

    ![grr モーダル](/images/grr-slack-app-on-cloudflare/modal.png)

3. **記録の一覧表示 (Web UI):**
    - 記録されたイライラは、Cloudflare Workers でホストされた Web ページで一覧表示されます。（現在は基本的な表示のみです）

    ![grr 一覧画面](/images/grr-slack-app-on-cloudflare/list.png)

4. **Slack 通知:**
    - イライラが記録されると、記録を開始したチャンネルまたはDMに通知メッセージが投稿されます。

### リポジトリ

ソースコードはこちらで公開しています。
https://github.com/coji/grr

## 技術スタック紹介

今回「grr」を開発するにあたり、以下の技術スタックを選定しました。

- **フレームワーク:** [React Router](https://reactrouter.com/) (v7)
- **UI:** [React](https://react.dev/) (v19), [Tailwind CSS](https://tailwindcss.com/) (v4)
- **ビルドツール:** [Vite](https://vitejs.dev/)
- **言語:** [TypeScript](https://www.typescriptlang.org/)
- **ランタイム:** [Cloudflare Workers](https://workers.cloudflare.com/)
- **データベース:** [Cloudflare D1](https://developers.cloudflare.com/d1/)
- **DB クライアント:** [Kysely](https://kysely.dev/) with [kysely-d1](https://github.com/aidenwallis/kysely-d1)
- **Slack 連携:** [slack-cloudflare-workers](https://github.com/slackapi/slack-cloudflare-workers), [slack-edge](https://github.com/slackapi/slack-edge)
- **パッケージマネージャー:** [pnpm](https://pnpm.io/)
- **リンター/フォーマッター:** [Biome](https://biomejs.dev/)

### なぜこの技術を選んだか？

- **Cloudflare Workers & D1:**
  - **サーバレス & エッジ:** 管理の手間が少なく、ユーザーに近い場所で高速に動作させたい。
  - **低コスト:** 無料枠が充実しており、個人開発や小規模サービスに適している。
  - **D1 の魅力:** Workers との親和性が高く、SQLite ベースで手軽に始められる SQL データベース。マイグレーション機能も Wrangler CLI で提供されている。
- **React Router v7:**
  - **Vite との親和性:** v7 から Vite Plugin が提供され、Vite プロジェクトへの統合が非常にスムーズになった。`@react-router/dev` により、開発サーバーの起動やビルド、型生成などが統一されたコマンド (`react-router dev`, `react-router build`) で行える。
  - **SSR サポート:** Cloudflare Workers 環境での SSR (Server-Side Rendering) を公式にサポートしている。`createRequestHandler` を使うことで、Workers 上でのリクエストハンドリングが容易。
  - **ファイルベースルーティング (風):** `remix-flat-routes` とアダプタ (`@react-router/remix-routes-option-adapter`) を組み合わせることで、ファイルシステムに基づいたルーティング定義が可能。
- **Kysely & kysely-d1:**
  - **型安全:** TypeScript との相性が抜群で、コンパイル時に SQL クエリの間違いを発見できる。
  - **D1 Dialect:** `kysely-d1` を使うことで、Cloudflare D1 への接続やマイグレーションが容易になる。
  - **優れた DX:** 直感的な API で SQL クエリを構築できる。`CamelCasePlugin` でスネークケースのカラム名とキャメルケースのプロパティ名を自動変換できるのも便利。
- **slack-edge & slack-cloudflare-workers:**
  - **Cloudflare Workers 特化:** Workers 環境で Slack アプリを開発するために最適化されている。リクエスト検証や各種イベントハンドリングが簡単に行える。
  - **TypeScript ファースト:** 型定義が充実しており、安全に開発を進められる。
  - **Block Kit Helper:** `slack-edge` は Block Kit の型定義やユーティリティを提供しており、モーダルやメッセージの構築がしやすい。
- **Vite:**
  - **高速な開発体験:** 開発サーバーの起動や HMR (Hot Module Replacement) が非常に高速。
  - **React Router との連携:** 上述の通り、React Router v7 との連携が強化された。
- **Tailwind CSS v4 (Lightning CSS):**
  - **DX の向上:** v4 から JIT エンジンが Rust 製 (Lightning CSS ベース) になり、さらに高速化。設定も簡素化された。
  - **Vite Plugin:** `@tailwindcss/vite` を利用して Vite と簡単に連携できる。
- **Biome:**
  - **オールインワン:** リンターとフォーマッターが統合されており、設定や管理がシンプル。
  - **高速:** Rust 製で非常に高速に動作する。Prettier や ESLint からの移行も比較的容易。

## 実装解説

ここでは、いくつかの特徴的な部分の実装について解説します。

### 1. Cloudflare Workers + React Router v7 SSR

React Router v7 を Cloudflare Workers 上で SSR させるための設定とコードです。

**設定ファイル:**

- `vite.config.ts`: `@cloudflare/vite-plugin` と `@react-router/dev/vite` を導入します。`cloudflare()` プラグインで `viteEnvironment: { name: 'ssr' }` を指定するのがポイントです。

    ```typescript:vite.config.ts
    import { cloudflare } from '@cloudflare/vite-plugin'
    import { reactRouter } from '@react-router/dev/vite'
    import tailwindcss from '@tailwindcss/vite'
    import { defineConfig } from 'vite'
    import tsconfigPaths from 'vite-tsconfig-paths'

    export default defineConfig({
      plugins: [
        // Cloudflare Plugin: SSR環境を指定
        cloudflare({ viteEnvironment: { name: 'ssr' } }),
        tailwindcss(),
        // React Router Plugin
        reactRouter(),
        tsconfigPaths(),
      ],
      // ...
    })
    ```

- `react-router.config.ts`: SSR を有効にし、Vite 環境 API を使う設定を行います。

    ```typescript:react-router.config.ts
    import type { Config } from "@react-router/dev/config";

    export default {
      ssr: true,
      future: {
        unstable_viteEnvironmentApi: true,
      },
    } satisfies Config;
    ```

- `wrangler.jsonc`: Worker のエントリーポイントを指定します。

    ```jsonc:wrangler.jsonc
    {
      // ...
      "main": "./workers/app.ts",
      // ...
    }
    ```

**Worker エントリーポイント (`workers/app.ts`):**

`createRequestHandler` を使用して、React Router のビルド成果物 (`virtual:react-router/server-build`) を読み込み、リクエストハンドラを作成します。`AppLoadContext` を拡張して、Cloudflare の `env` と `ctx` を `loader` や `action` に渡せるようにしています。

```typescript:workers/app.ts
import { createRequestHandler } from "react-router";

// AppLoadContext を拡張して Cloudflare の環境情報を渡せるようにする
declare module "react-router" {
  export interface AppLoadContext {
    cloudflare: {
      env: Env; // wrangler.jsonc や .dev.vars で定義した Bindings や Variables
      ctx: ExecutionContext; // waitUntil などの実行コンテキスト
    };
  }
}

// ビルド成果物を動的に import
const requestHandler = createRequestHandler(
  () => import("virtual:react-router/server-build"),
  import.meta.env.MODE
);

export default {
  async fetch(request, env, ctx) {
    // リクエストハンドラに request と context を渡す
    return requestHandler(request, {
      cloudflare: { env, ctx },
    });
  },
} satisfies ExportedHandler<Env>;
```

**サーバーエントリー (`app/entry.server.tsx`):**

`renderToReadableStream` を使って React コンポーネントをストリーミングレンダリングします。

```typescript:app/entry.server.tsx
import { renderToReadableStream } from 'react-dom/server'
import type { AppLoadContext, EntryContext } from 'react-router'
import { ServerRouter } from 'react-router'
// isbot などのユーティリティも利用可能

export default async function handleRequest(
  request: Request,
  responseStatusCode: number,
  responseHeaders: Headers,
  routerContext: EntryContext,
  loadContext: AppLoadContext, // ここで Cloudflare の env や ctx を受け取れる
) {
  const body = await renderToReadableStream(
    <ServerRouter context={routerContext} url={request.url} />,
    {
      // エラーハンドリングなど
    },
  )

  responseHeaders.set('Content-Type', 'text/html')
  return new Response(body, {
    headers: responseHeaders,
    status: responseStatusCode,
  })
}
```

**Loader/Action での Context 利用:**

ルートファイル (`app/routes/some/route.tsx` など) の `loader` や `action` の引数から `context` を経由して Cloudflare の `env` や `ctx` にアクセスできます。

```typescript
import type { ActionArgs, LoaderArgs } from 'react-router'

export const loader = async ({ context }: LoaderArgs) => {
  const { env, ctx } = context.cloudflare
  // env.DB や env.SLACK_BOT_TOKEN などにアクセス可能
  // ctx.waitUntil(...) なども利用可能
  // ...
}

export const action = async ({ request, context }: ActionArgs) => {
  const { env, ctx } = context.cloudflare
  // ...
}
```

### 2. Slack 連携 (slack-edge / slack-cloudflare-workers)

Slack からのリクエスト (スラッシュコマンド、インタラクション) を処理する部分です。

**Webhook ルート (`app/routes/webhook.slack/route.tsx`):**

Slack からのすべてのリクエスト (コマンド、インタラクション、イベント) を受け付けるエンドポイントです。`action` 関数内で `slack-cloudflare-workers` の `SlackApp` インスタンスを生成し、リクエストを処理させます。`context.cloudflare` を `SlackApp` の `run` メソッドに渡すことで、ハンドラ内で `env` や `ctx` を利用できるようにします。

```typescript:app/routes/webhook.slack/route.tsx
import { createSlackApp } from '~/slack-app/app'
import type { Route } from './+types/route'

// POST リクエストを処理する action 関数
export const action = ({ request, context }: Route.ActionArgs) => {
  const slackApp = createSlackApp()
  // SlackApp に request と Cloudflare の context を渡して実行
  // 検証やハンドラの呼び出しは SlackApp が行う
  return slackApp.run(request, context.cloudflare.ctx)
}
```

**SlackApp 初期化 (`app/slack-app/app.ts`):**

`slack-cloudflare-workers` の `SlackApp` を初期化し、環境変数 (特に `SLACK_SIGNING_SECRET`, `SLACK_BOT_TOKEN`) を設定します。ここで各種イベントハンドラを登録します。

```typescript:app/slack-app/app.ts
import { env } from 'cloudflare:workers' // Cloudflare 環境変数へのアクセス
import {
  SlackApp,
  type SlackAppLogLevel,
  type SlackEdgeAppEnv,
} from 'slack-cloudflare-workers'
import { registerGrrHandlers } from './handlers/grr' // ハンドラを別ファイルから import

// SlackApp インスタンスを作成する関数
export function createSlackApp() {
  const app = new SlackApp<SlackEdgeAppEnv>({
    env: {
      // スプレッド構文で Cloudflare の env をすべて渡す
      ...env,
      // SLACK_LOGGING_LEVEL は型を明示的に指定
      SLACK_LOGGING_LEVEL: env.SLACK_LOGGING_LEVEL as SlackAppLogLevel,
    },
  })

  // ハンドラを登録
  registerHandlers(app)
  return app
}

// ハンドラ登録関数
function registerHandlers(app: SlackApp<SlackEdgeAppEnv>) {
  registerGrrHandlers(app)
  // 他のハンドラがあればここに追加
}
```

**イベントハンドラ (`app/slack-app/handlers/grr.ts`):**

スラッシュコマンド (`/grr`)、メッセージショートカット (`grr_shortcut`)、モーダル送信 (`grr_modal`) のハンドラを定義します。`context.client` を使って Slack API (例: `views.open`, `chat.postMessage`) を呼び出します。

```typescript:app/slack-app/handlers/grr.ts
import { nanoid } from 'nanoid'
import type { SlackApp, SlackEdgeAppEnv } from 'slack-cloudflare-workers'
import { db } from '~/services/db' // DB サービスを import
import dayjs from '~/utils/dayjs' // 日付ユーティリティを import
import { buildGrrModal } from './views/grr-modal' // モーダル構築関数を import

export const registerGrrHandlers = (app: SlackApp<SlackEdgeAppEnv>) => {
  // /grr コマンドハンドラ
  app.command(
    '/grr',
    async (req) => {}, // ack (必須だがここでは何もしない)
    async ({ context, payload }) => {
      // モーダルを開く
      await context.client.views.open({
        trigger_id: payload.trigger_id,
        view: buildGrrModal(payload.channel_id, payload.text), // モーダルを構築して渡す
      })
    },
  )

  // メッセージショートカットハンドラ
  app.shortcut(
    'grr_shortcut',
    async (req) => {}, // ack
    async ({ context, payload }) => {
      const message =
        payload.type === 'message_action' ? payload.message.text : undefined
      await context.client.views.open({
        trigger_id: payload.trigger_id,
        view: buildGrrModal(context.channelId, message),
      })
    },
  )

  // モーダル (View) 送信ハンドラ
  app.view(
    'grr_modal',
    async (req) => { // ack (送信ボタン押下時に即時応答)
      return // ack のみ
    },
    async ({ context, payload: { view, user }, body }) => {
      // view.state.values から入力値を取得
      const level = view.state.values.level_block.level.selected_option?.value
      const text = view.state.values.text_block.text.value
      // private_metadata から初期値を復元
      const meta = JSON.parse(view.private_metadata ?? '{}')
      const channelId = meta.channelId

      const score = Number(level ?? '3') // デフォルト値

      // D1 にデータを保存 (Kysely を使用)
      const ret = await db
        .insertInto('irritations')
        .values({
          id: nanoid(),
          userId: user.id,
          channelId: channelId ?? null,
          rawText: text ?? '',
          score,
          createdAt: dayjs().utc().toISOString(),
          updatedAt: dayjs().utc().toISOString(),
          isPublic: 0, // デフォルトは非公開 (例)
        })
        .returningAll()
        .executeTakeFirstOrThrow()

      // Slack に通知メッセージを投稿
      await context.client.chat.postMessage({
        channel: channelId ?? user.id, // チャンネルIDがあればそこへ、なければDMへ
        text: `😤 ${user.name} さんがイライラ "${text}"を記録しました (イラ度: ${score})`,
      })
    },
  )
}
```

**Block Kit モーダル (`app/slack-app/handlers/views/grr-modal.ts`):**

`slack-edge` の型定義を利用して、モーダルの Block Kit JSON を構築します。`private_metadata` を使って、モーダルを開いた際のチャンネルIDなどの情報を後続の `view` イベントハンドラに渡すことができます。

```typescript:app/slack-app/handlers/views/grr-modal.ts
import type { ModalView, PlainTextOption } from 'slack-edge'

// 選択肢の定義
export const ILA_LEVEL_OPTIONS: PlainTextOption[] = [
  { text: { type: 'plain_text', text: '1 - まあいいか', emoji: true }, value: '1' },
  { text: { type: 'plain_text', text: '2 - ちょっとイラッ', emoji: true }, value: '2' },
  // ... (省略) ...
] as const

// モーダル構築関数
export const buildGrrModal = (
  channelId?: string,
  defaultMessage?: string,
): ModalView => {
  return {
    type: 'modal',
    // private_metadata にチャンネルIDなどの情報を JSON 文字列として埋め込む
    private_metadata: JSON.stringify({ channelId }),
    callback_id: 'grr_modal', // view ハンドラと紐付ける ID
    title: { type: 'plain_text', text: 'イライラを記録する' },
    submit: { type: 'plain_text', text: '保存' },
    close: { type: 'plain_text', text: 'キャンセル' },
    blocks: [
      // Input Block (イライラ度選択)
      {
        type: 'input',
        block_id: 'level_block',
        label: { type: 'plain_text', text: 'イライラ度 (1〜5)' },
        element: {
          type: 'static_select',
          action_id: 'level',
          options: ILA_LEVEL_OPTIONS,
          initial_option: ILA_LEVEL_OPTIONS[2], // デフォルト選択肢
        },
      },
      // Input Block (内容入力)
      {
        type: 'input',
        block_id: 'text_block',
        label: { type: 'plain_text', text: '内容' },
        element: {
          type: 'plain_text_input',
          action_id: 'text',
          multiline: true,
          initial_value: defaultMessage, // ショートカットやコマンドからの初期値
        },
      },
    ],
  }
}
```

### 3. Cloudflare D1 + Kysely

Cloudflare D1 データベースを Kysely を使って操作する部分です。

**DB サービス (`app/services/db.ts`):**

Kysely のインスタンスを生成し、D1 Dialect と CamelCasePlugin を設定します。`Database` インターフェースでテーブルスキーマと対応する TypeScript の型を定義します。

```typescript:app/services/db.ts
import { env } from 'cloudflare:workers' // D1 Binding へのアクセス
import { CamelCasePlugin, Kysely } from 'kysely'
import { D1Dialect } from 'kysely-d1'

// データベーススキーマのインターフェース定義
export interface Database {
  irritations: {
    id: string
    userId: string // SlackユーザーID (user_id)
    channelId: string | null // SlackチャンネルID (channel_id)
    rawText: string // 元の投稿内容 (raw_text)
    score: number // イライラ度の累積 (score)
    createdAt: string // 初記録日時 (created_at)
    updatedAt: string // 最新記録日時 (updated_at)
    isPublic: number // 公開可否 (is_public)
  }
  // 他のテーブルがあればここに追加
}

// Kysely インスタンスの作成
export const db = new Kysely<Database>({
  // D1 Dialect を使用し、Cloudflare 環境変数から D1 Binding を渡す
  dialect: new D1Dialect({ database: env.DB }),
  plugins: [
    // スネークケース (DB) <-> キャメルケース (TS) を自動変換
    new CamelCasePlugin(),
  ],
})
```

**マイグレーション (`migrations/0001_init.sql`):**

`wrangler d1 migration create` で生成された SQL ファイルにテーブル定義を記述します。

```sql:migrations/0001_init.sql
-- Migration number: 0001
-- イライラ記録
CREATE TABLE IF NOT EXISTS irritations (
  id          TEXT    NOT NULL PRIMARY KEY,
  user_id     TEXT    NOT NULL,
  channel_id  TEXT,
  raw_text    TEXT    NOT NULL,
  score       INTEGER NOT NULL DEFAULT 0,
  created_at  TEXT    NOT NULL,  -- 記録日時 (ISO8601)
  updated_at  TEXT    NOT NULL,  -- 更新日時 (ISO8601)
  is_public   INTEGER NOT NULL DEFAULT 1 -- 0: false, 1: true
);
```

マイグレーションの適用は `wrangler d1 migration apply grr-db` コマンドで行います (`grr-db` は `wrangler.jsonc` で定義した `database_name`)。

**DB 操作 (例: `_index.tsx` loader):**

`loader` や `action` 内で `db` サービスを使って D1 を操作します。Kysely のおかげで型安全にクエリを記述できます。

```typescript:app/routes/_index.tsx
import { db } from '~/services/db'
import dayjs from '~/utils/dayjs' // dayjs も活用
import type { Route } from './+types/_index'

// loader で D1 からデータを取得
export const loader = async () => {
  const irritations = await db
    .selectFrom('irritations') // テーブル選択 (型補完が効く)
    .selectAll()              // 全カラム選択
    .orderBy('createdAt', 'desc') // createdAt カラムで降順ソート (キャメルケースで指定)
    .limit(100)               // 100件取得
    .execute()                // クエリ実行
  return {
    irritations, // 結果はキャメルケースのプロパティを持つオブジェクト配列
  }
}

// コンポーネントで loaderData を利用
export default function Home({ loaderData }: Route.ComponentProps) {
  // ... loaderData.irritations を使ってリスト表示 ...
}
```

### 4. その他

- **ルーティング:** `remix-flat-routes` を使うことで、`app/routes` ディレクトリ以下のファイル構造に基づいてルートが自動生成されます。`app/routes.ts` で設定しています。

    ```typescript:app/routes.ts
    import { remixRoutesOptionAdapter } from '@react-router/remix-routes-option-adapter';
    import { flatRoutes } from 'remix-flat-routes';

    export default remixRoutesOptionAdapter((defineRotue) =>
      flatRoutes('routes', defineRotue, {
        // 特定のファイルを除外するなどの設定も可能
        ignoredRouteFiles: ['**/index.ts', '**/_shared/**'],
      })
    ) as ReturnType<typeof remixRoutesOptionAdapter>;
    ```

- **日付処理:** `dayjs` を UTC プラグインと共に利用し、DB への保存や表示時のフォーマットを行っています (`app/utils/dayjs.ts`)。
- **コード品質:** `Biome` を使って、`pnpm biome check --apply .` や `pnpm biome format --write .` でコードのチェックとフォーマットを統一しています。

## 開発・デプロイフロー

README にも記載がありますが、開発からデプロイまでの流れは以下のようになります。

1. **セットアップ:**
    - リポジトリをクローンし、`pnpm install` で依存関係をインストール。
    - Cloudflare D1 データベースを作成 (`wrangler d1 create`) し、`wrangler.jsonc` に ID を設定。
    - マイグレーションを実行 (`wrangler d1 migration apply`)。
    - Slack アプリを作成し、マニフェスト (`slack-app-manifest.example.json`) を参考に設定。Bot Token と Signing Secret を取得。
    - `.dev.vars` ファイルを作成し、Slack の認証情報などを設定。
    - 型定義を生成 (`pnpm typecheck`)。
2. **ローカル開発:**
    - `pnpm dev` を実行。Vite 開発サーバーと `wrangler dev` が起動し、HMR が有効な状態で開発できます。D1 など Cloudflare リソースもローカルでエミュレートされます。
    - `http://localhost:5173` などでアプリにアクセスできます。
    - Slack からの Webhook をローカルで受け取るには、`cloudflared tunnel` や ngrok などのトンネリングツールが必要です。トンネル URL を Slack アプリ設定の Request URL に設定します。
3. **ビルド:**
    - `pnpm run build` を実行。`@react-router/dev` が Cloudflare Workers 用のビルドを行います。
4. **デプロイ:**
    - `pnpm deploy` (内部で `wrangler deploy` を実行) で Cloudflare Workers にデプロイします。
    - デプロイ後に表示される Worker の URL を、Slack アプリ設定の Request URL (Interactivity & Shortcuts, Event Subscriptions, Slash Commands) に **本番用 URL** として設定します。
    - 本番環境用の環境変数 (Slack Token など) は `wrangler secret put` コマンドで設定します。

## 今後の展望

まだ基本的な機能しか実装できていないため、今後は以下のような機能を追加していきたいと考えています。

- イライラランキング表示 (日次、週次、ユーザー別など)
- 記録の編集・削除機能
- 簡単な認証機能 (特定のユーザーのみ一覧を見れるようにするなど)
- グラフなどを用いた分析機能
- Slack 通知のカスタマイズ

## まとめ

この記事では、Cloudflare Workers/D1 と React Router v7 を中心とした技術スタックで開発した Slack アプリ「grr」について紹介しました。

Cloudflare Workers/D1 は、手軽に始められるサーバレス環境として非常に魅力的であり、特に D1 と Kysely の組み合わせによる型安全なデータベースアクセスは開発体験を大きく向上させました。React Router v7 は Vite との統合が進み、Cloudflare Workers 上での SSR もスムーズに行えるため、モダンなフロントエンド開発の選択肢として有力だと感じています。slack-edge/slack-cloudflare-workers も Workers 環境での Slack アプリ開発を強力にサポートしてくれます。

まだまだ荒削りなアプリですが、この構成に興味を持った方にとって、何かしらの参考になれば嬉しいです。ぜひ GitHub リポジトリも覗いてみてください！

https://github.com/coji/grr
