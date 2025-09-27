---
title: "React Router v7 で Google Drive ギャラリーアプリを作ってみた"
emoji: "📸"
type: "tech"
topics: ["reactrouter", "googleapi", "oauth", "typescript", "cloudflareworkers"]
published: true
---

# これはなに？

Google Drive API を使ったことがなかったので、試しに React Router v7 でフォルダの中の画像を表示するギャラリーアプリを作って、実装のポイントをまとめてみました。

デモは https://www.techtalk.jp/demo/google-drive で触れます。

コードは https://github.com/techtalkjp/techtalk.jp/tree/main/app/routes/demo%2B/google-drive%2B に置いてあります。

## 作ったもの

![スクリーンショット](/images/react-router-v7-google-drive-gallery/google-drive-gallery.jpg)

Google Drive の画像をギャラリー表示するアプリです。Google アカウントでログインすると、自分の Drive に保存してある画像が見られるという、シンプルなものです。

フォルダの中も移動できるし、ページネーションもあります。サムネイルをクリックすると元画像が見られる、よくあるギャラリーアプリです。

## React Router v7 の Framework mode が便利

React Router v7 はフルスタックフレームワークになっていて、SSR もできるし、API ルートも書けるし、セッション管理もできます。

今回使った機能はこんな感じ：
- **loader 関数** でサーバーサイドのデータ取得
- **Resource Routes** で OAuth のコールバックとか画像プロキシを実装
- **セッション管理** でトークンを安全に保存

## OAuth 認証の流れ

Google の OAuth 2.0 認証を実装しました。流れはこんな感じです：

1. ログインボタンを押す → `/demo/google-drive/auth` へ
2. Google の認証画面へ飛ばす
3. 認証したら `/demo/google-drive/callback` に戻ってくる
4. 認可コードをアクセストークンに交換
5. セッションに保存して元のページへ

### Resource Route が便利

認証用のエンドポイントを Resource Route (APIルート)として実装します。`auth.ts` はこんな感じ：

```typescript
// auth.ts - Google OAuth URLを作ってリダイレクト
export async function loader({ request }: Route.LoaderArgs) {
  const session = await getSession(request.headers.get('Cookie'))
  const state = crypto.randomUUID() // CSRF対策
  session.set('oauth_state', state)

  const authUrl = getGoogleOAuthURL(url.origin, state)

  return redirect(authUrl, {
    headers: { 'Set-Cookie': await commitSession(session) }
  })
}
```

コールバックも同じように Resource Route で処理します。Express や Hono など、別の API サーバー立てなくていいのが楽ですね。

## Google Drive API でファイル一覧を取得

Google Drive API でファイルを検索する際は、`q` パラメータでクエリを指定します。これが結構強力で、ファイルタイプやフォルダ、ゴミ箱の中身など、いろんな条件で絞り込めるんです。

### ファイル一覧取得の実装

```typescript
async function fetchDriveFiles(accessToken: string, folderId: string | null) {
  const params = new URLSearchParams({
    pageSize: '30',
    orderBy: 'createdTime desc',
    fields: 'nextPageToken,files(id,name,mimeType,thumbnailLink,size)',
  })

  // クエリの組み立て
  const query = folderId
    ? `'${folderId}' in parents and (mimeType contains 'image/' or mimeType = 'application/vnd.google-apps.folder') and trashed = false`
    : `(mimeType contains 'image/' or mimeType = 'application/vnd.google-apps.folder') and trashed = false and 'root' in parents`

  params.set('q', query)

  const response = await fetch(
    `https://www.googleapis.com/drive/v3/files?${params.toString()}`,
    {
      headers: { Authorization: `Bearer ${accessToken}` }
    }
  )
}
```

### クエリの詳細解説

Google Drive API のクエリはこんな感じで組み立てます：

- **`'folderId' in parents`** - 特定のフォルダの中のファイルを取得
- **`mimeType contains 'image/'`** - 画像ファイルだけ取得（jpeg, png, gif など全部含まれる）
- **`mimeType = 'application/vnd.google-apps.folder'`** - フォルダも含める
- **`trashed = false`** - ゴミ箱のファイルは除外
- **`'root' in parents`** - ルートフォルダの直下のファイル

これらを `and` や `or` でつなげて、欲しいファイルだけを絞り込むわけです。今回は「画像とフォルダだけ、ゴミ箱以外から」という条件にしました。

`fields` パラメータで取得するフィールドを指定するのも重要です。必要なフィールドだけを指定することで、レスポンスサイズを小さくできます。

詳しいクエリの書き方は [Google Drive API のファイル検索ガイド](https://developers.google.com/workspace/drive/api/guides/search-files?hl=ja) を参照してください。

### アクセストークンの自動リフレッシュ

アクセストークンは1時間で期限切れるので、リフレッシュトークンを使って自動更新する仕組みを作りました。

`withGoogleAccess` という関数でラップして、401 エラーが来たら自動でリフレッシュ → リトライ、という感じです。

## 画像プロキシで本体を取得

Google Drive の画像を直接表示しようとすると、CORS エラーになったり、アクセストークンが露出したりします。そこで、プロキシ用の Resource Route を作って、サーバーサイドで画像を取得して返すようにしました。

### 画像本体の取得方法

Google Drive API では、ファイルの内容を取得するには `alt=media` パラメータを使います：

```typescript
async function fetchDriveImage(accessToken: string, fileId: string, isThumb: boolean) {
  // サムネイルの場合は、まずメタデータを取得
  if (isThumb) {
    const metadataResponse = await fetch(
      `https://www.googleapis.com/drive/v3/files/${fileId}?fields=thumbnailLink`,
      {
        headers: { Authorization: `Bearer ${accessToken}` }
      }
    )

    const metadata = await metadataResponse.json()
    if (metadata.thumbnailLink) {
      // サムネイルURLから画像を取得
      return fetch(metadata.thumbnailLink, {
        headers: { Authorization: `Bearer ${accessToken}` }
      })
    }
  }

  // フルサイズ画像の取得 - alt=media がポイント！
  return fetch(
    `https://www.googleapis.com/drive/v3/files/${fileId}?alt=media`,
    {
      headers: { Authorization: `Bearer ${accessToken}` }
    }
  )
}
```

### プロキシ Route の実装

`proxy.$fileId.ts` では、取得した画像にセキュリティヘッダーを追加して返します：

```typescript
export async function loader({ params }: Route.LoaderArgs) {
  const { fileId } = params

  // Google Drive から画像を取得
  const imageResponse = await withGoogleAccess(
    session,
    (accessToken) => fetchDriveImage(accessToken, fileId, isThumb)
  )

  // セキュリティヘッダーを追加して返す
  const headers = new Headers()
  headers.set('Content-Type', 'image/jpeg')
  headers.set('Cache-Control', 'private, max-age=3600')
  headers.set('X-Content-Type-Options', 'nosniff')
  headers.set('X-Frame-Options', 'DENY')
  headers.set('Content-Security-Policy', "default-src 'none'; sandbox")

  return new Response(imageResponse.body, { status: 200, headers })
}
```

### なぜプロキシが必要か

1. **アクセストークンの隠蔽** - クライアントにトークンを露出させない
2. **CORS エラーの回避** - サーバーサイドで取得するので CORS の制約を受けない
3. **キャッシュ制御** - CDN でキャッシュできるようにヘッダーを調整
4. **セキュリティ強化** - Content-Type の検証やセキュリティヘッダーの追加

これで画像の URL は `/demo/google-drive/proxy/[fileId]` になって、普通の画像URLのように使えます。`<img>` タグの `src` に指定するだけで表示できるので、フロントエンド側はシンプルに実装できました。

## セキュリティ対策

### トークンの暗号化

アクセストークンとリフレッシュトークンは、そのままセッションに保存するのは怖いので、AES-256-GCM で暗号化してます。

### CSRF 対策

OAuth の state パラメータを使って、リクエストの正当性を確認。セッションに保存した値と照合して、一致しなければエラーにします。

### パス・トラバーサル対策

ファイル ID は正規表現でチェック。変な値が来たら 400 Bad Request を返すようにしてます。

## 感想

React Router v7 の Framework mode、なかなか使い勝手がいいです。特に Resource Routes が便利で、API エンドポイントをルート定義の中で完結できるのがいいですね。

Google Drive API については、最初は OAuth とか面倒そうだなと思ってたんですが、ちゃんと抽象化すれば意外とシンプルに実装できました。特に印象的だったのは：

- **クエリ言語が独特** - `'folderId' in parents` とか `mimeType contains 'image/'` みたいな書き方は Google Drive 特有で、最初は戸惑いました。ドキュメントをちゃんと読まないと正しいクエリが書けないですね
- **`alt=media` パラメータが便利** - ファイルのメタデータと内容を別々に取得できるのがいい。必要に応じて使い分けられるし、サムネイルだけ先に取得とかもできる
- **fields でレスポンスを絞れる** - 必要なフィールドだけ指定できるので、レスポンスサイズを最適化できました。特に大量のファイルを扱う時に重要ですね

あと、Google Drive API のレート制限も思ったより緩くて、普通に使う分には全然問題なかったです。トークンのリフレッシュとか、エラーハンドリングとか、一度作っちゃえば使い回せますしね。

React Router v7 と組み合わせると、OAuth のコールバックや画像プロキシを Resource Routes で実装できるので、外部 API 連携アプリがすごく作りやすくなったと感じました。Cloudflare Workers で動かせるのも魅力ですね。

詳しい実装はGitHubのコードを見てもらえればと思います！
https://github.com/techtalkjp/techtalk.jp/tree/main/app/routes/demo%2B/google-drive%2B
