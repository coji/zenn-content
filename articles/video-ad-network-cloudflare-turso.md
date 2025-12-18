---
title: 'Cloudflare Workers + Turso で動画広告配信システムを作ってみた - VAST準拠、エッジで動くOSS'
emoji: '📺'
type: 'tech'
topics: ['cloudflare', 'turso', 'hono', 'betterauth', 'reactrouter']
published: true
---

## これはなに？

趣味で動画・音声広告を配信するアドネットワークを OSS としてフルスクラッチで構築してみました。技術スタックの選定理由と実装のポイントを共有します。

https://github.com/coji/video-ad-network

LT 用に作ったスライドもあるので、概要を把握したい方はこちらもどうぞ。

@[docswell](https://www.docswell.com/s/8723156/KN9DEE-2025-12-18-150333)

## システム構成

![システム構成概要](/images/video-ad-network-cloudflare-turso/architecture.png)

4つのパッケージで構成しています。

| パッケージ | 役割 | 技術 |
| --- | --- | --- |
| apps/ad-server | 広告配信 API | Hono + Cloudflare Workers |
| apps/ui | 管理画面 | React Router v7 + Cloudflare Workers |
| packages/ad-sdk | クライアント SDK | TypeScript + Vite |
| packages/db | データベース層 | Kysely + Atlas |

pnpm ワークスペースと Turborepo でモノレポ管理しています。

## 技術選定のポイント

### エッジ実行を前提にした構成

広告配信は低遅延が求められます。Cloudflare Workers と Turso の組み合わせで、ユーザーに近いエッジで処理を完結できるようにしました。

Turso は libsql ベースの分散 SQLite です。エッジからの読み取りが高速で、広告の選択やトラッキングの記録に適しています。

### 型安全なデータベースアクセス

Kysely をクエリビルダーに採用しました。kysely-codegen でスキーマから型を自動生成し、CamelCasePlugin で SQL の snake_case と TypeScript の camelCase を自動変換しています。

```typescript
const ads = await db
  .selectFrom('ads')
  .innerJoin('adGroups', 'adGroups.id', 'ads.adGroupId')
  .innerJoin('campaigns', 'campaigns.id', 'adGroups.campaignId')
  .where('campaigns.status', '=', 'ACTIVE')
  .select(['ads.id', 'ads.url', 'adGroups.bidPriceCpm'])
  .execute()
```

スキーマ管理には Atlas を使っています。schema.sql に現在のスキーマを書き、`atlas migrate diff` で差分マイグレーションを生成するスタイルです。Prisma のような自動生成ではなく、SQL を直接書けるのが気に入っています。

### VAST 4.1 準拠

VAST は IAB 標準の動画広告フォーマットです。準拠することで既存の動画プレイヤーや DSP との連携が容易になります。

```xml
<VAST version="4.1">
  <Ad id="ad123">
    <InLine>
      <Creatives>
        <Creative>
          <Linear>
            <Duration>00:00:15</Duration>
            <MediaFiles>...</MediaFiles>
            <TrackingEvents>
              <Tracking event="start"><![CDATA[https://.../progress?p=0]]></Tracking>
              <Tracking event="firstQuartile"><![CDATA[https://.../progress?p=25]]></Tracking>
              <Tracking event="complete"><![CDATA[https://.../progress?p=100]]></Tracking>
            </TrackingEvents>
          </Linear>
        </Creative>
      </Creatives>
    </InLine>
  </Ad>
</VAST>
```

## 広告配信の流れ

### 1. VAST リクエスト

メディアサイトが SDK を読み込み、`initializeAdSDK` を呼び出します。SDK は広告サーバーに VAST リクエストを送ります。

```typescript
const state = await initializeAdSDK({
  mediaId: 'media123',
  adSlotId: 'slot456',
  containerElement: document.getElementById('ad-player'),
})
```

### 2. 広告選択

広告サーバーは配信可能なキャンペーンから広告を選択します。このときフリークエンシーキャップを判定し、同じユーザーへの過剰な広告表示を防いでいます。

```typescript
const candidates = await db
  .selectFrom('ads')
  .innerJoin('adGroups', 'adGroups.id', 'ads.adGroupId')
  .innerJoin('campaigns', 'campaigns.id', 'adGroups.campaignId')
  .where('campaigns.status', '=', 'ACTIVE')
  .where('ads.type', '=', mediaType)
  .execute()

const eligible = candidates.filter((ad) => canSelectAd(ad, frequencyData))
const selected = eligible[Math.floor(Math.random() * eligible.length)]
```

### 3. VAST 生成と返却

選択した広告の VAST XML を生成して返却します。XML にはメディアファイルの URL、クリック先 URL、各種トラッキング URL が含まれます。

### 4. 再生とトラッキング

SDK は VAST をパースして video 要素を生成し、再生を開始します。再生開始時にインプレッショントラッカーを発火し、25%、50%、75%、完了時点でそれぞれ進捗トラッカーを発火します。

```typescript
mediaElement.addEventListener('timeupdate', () => {
  const progress = (mediaElement.currentTime / mediaElement.duration) * 100

  if (progress >= 25 && !sent.firstQuartile) {
    sendTracker(trackers.firstQuartile)
    sent.firstQuartile = true
  }
  // ...
})
```

## フリークエンシーキャップの実装

同じユーザーに同じ広告を何度も見せないための仕組みを Cookie で実装しました。

```typescript
interface FrequencyData {
  [adGroupId: string]: {
    count: number
    windowStartTime: number
  }
}

function canSelectAd(ad: Ad, data: FrequencyData): boolean {
  const entry = data[ad.adGroupId]
  if (!entry) return true

  const windowMs = getWindowMs(ad.frequencyCapUnit, ad.frequencyCapWindow)
  const elapsed = Date.now() - entry.windowStartTime

  if (elapsed > windowMs) return true

  return entry.count < ad.frequencyCapImpressions
}
```

広告グループごとに表示回数とウィンドウ開始時刻を保存し、VAST リクエスト時に判定しています。ウィンドウ単位は分、時間、日から選べます。

## トラッキングエンドポイント

広告サーバーは4種類のトラッキングエンドポイントを持ちます。

```typescript
app.get('/v1/vast', handleVastRequest)
app.get('/v1/impression', handleImpression)
app.get('/v1/progress', handleProgress)
app.get('/v1/click', handleClick)
```

`/v1/impression` は広告表示完了時に呼ばれます。イベントを記録し、1x1 の透明 GIF を返します。

```typescript
async function handleImpression(c: Context) {
  const { mediaId, adId, impressionId } = c.req.query()

  await saveAdEvent({
    eventType: 'impression',
    mediaId,
    adId,
    impressionId,
    uid: c.get('uid'),
  })

  return c.body(TRANSPARENT_GIF, 200, {
    'Content-Type': 'image/gif',
    'Cache-Control': 'no-store',
  })
}
```

`/v1/progress` は再生進捗を記録します。progress パラメータで 0/25/50/75/100 を受け取ります。

`/v1/click` はクリックを記録した後、広告主のランディングページにリダイレクトします。

```typescript
async function handleClick(c: Context) {
  const { adId, isCompanion, companionId } = c.req.query()

  const clickThroughUrl = isCompanion
    ? await getCompanionClickUrl(companionId)
    : await getAdClickUrl(adId)

  await db.insertInto('clicks').values({ ... }).execute()

  return c.redirect(clickThroughUrl)
}
```

## 管理画面

React Router v7 を Cloudflare Workers 上で動かしています。認証には better-auth を使い、組織ベースのマルチテナントを実現しました。

### better-auth による マルチテナント

better-auth の organization plugin を使って、複数の広告主・メディアを1つのシステムで管理できるようにしています。

```typescript
import { betterAuth } from 'better-auth'
import { admin } from 'better-auth/plugins/admin'
import { organization } from 'better-auth/plugins/organization'

export const createAuth = (env: AuthEnv) => {
  return betterAuth({
    baseURL: env.BETTER_AUTH_URL,
    secret: env.BETTER_AUTH_SECRET,
    database: { dialect, type: 'sqlite' },
    emailAndPassword: { enabled: true },
    plugins: [
      admin(),
      organization({
        async sendInvitationEmail(data) {
          // メール送信処理
        },
      }),
    ],
  })
}
```

organization plugin を入れると、ユーザーは複数の組織に所属でき、セッションに `activeOrganizationId` が追加されます。これを使って現在操作中の組織を判別します。

```typescript
export const requireOrgUser = async (args: LoaderFunctionArgs) => {
  const session = await requireUser(args)
  if (!session.session.activeOrganizationId) {
    throw redirect('/login')
  }
  return session
}

export const loader = async (args: LoaderFunctionArgs) => {
  const { session } = await requireOrgUser(args)
  const orgId = session.activeOrganizationId

  const campaigns = await db
    .selectFrom('campaigns')
    .innerJoin('advertisers', 'advertisers.id', 'campaigns.advertiserId')
    .where('advertisers.organizationId', '=', orgId)
    .selectAll()
    .execute()

  return { campaigns }
}
```

データの分離は各テーブルの `organizationId` カラムで行っています。広告主やメディアは必ず組織に紐づき、クエリ時に組織 ID でフィルタすることで他の組織のデータにアクセスできないようにしています。管理者は admin plugin でユーザーの ban/impersonate 機能を使えます。

### 広告主・メディア向け機能

広告主向けには、キャンペーン（配信期間、予算、配信ペース）、広告グループ（入札価格、フリークエンシーキャップ）、広告（動画/音声ファイル、クリック先URL）、コンパニオンバナー（付属の画像広告）を階層的に管理できる機能を用意しています。

メディア向けには、メディア（コンテンツサイトの登録）、広告枠（動画/音声の広告枠）、コンパニオンスロット（バナーサイズの指定）を設定できます。

## まとめ

Cloudflare Workers、Turso、Hono、React Router v7 というモダンなエッジスタックで広告配信システムを構築しました。フリークエンシーキャップや VAST といった広告業界特有の要件も Cookie とトラッキングエンドポイントで実装できました。better-auth の organization plugin によるマルチテナント、Atlas による宣言的マイグレーション、Kysely による型安全なクエリで、開発体験も良好です。
