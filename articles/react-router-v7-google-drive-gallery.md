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

## Google Drive API を叩く

メインページの loader で Google Drive API を呼び出します：

```typescript
export async function loader({ request }: Route.LoaderArgs) {
  const session = await getSession(request.headers.get('Cookie'))
  const googleTokens = getSessionTokens(session)

  if (!googleTokens) {
    return data({ isAuthenticated: false, files: [] })
  }

  // Drive API で画像一覧を取得
  const filesData = await fetchDriveFilesWithAuth(session, folderId, pageToken, pageSize)

  return data({
    ...filesData,
    isAuthenticated: true,
    googleUser: session.get('google_user'),
  })
}
```

### アクセストークンの自動リフレッシュ

アクセストークンは1時間で期限切れので、リフレッシュトークンを使って自動更新する仕組みを作りました。

`withGoogleAccess` という関数でラップして、401 エラーが来たら自動でリフレッシュ → リトライ、という感じです。

## 画像プロキシの実装

Google Drive に保存した画像を直接表示しようとすると CORS エラーになったり、アクセストークンが露出しないように注意する必要があります。

その対応として、`proxy.$fileId.ts` という Resource Route を作って、画像をプロキシすることにしました：

```typescript
export async function loader({ params }: Route.LoaderArgs) {
  const { fileId } = params

  // Google Drive から画像を取得
  const imageResponse = await withGoogleAccess(
    session,
    (accessToken) => fetchDriveImage(accessToken, fileId)
  )

  // セキュリティヘッダーを追加して返す
  return new Response(imageResponse.body, {
    headers: {
      'Content-Type': 'image/jpeg',
      'Cache-Control': 'private, max-age=3600',
      'X-Content-Type-Options': 'nosniff',
      // などなど...
    }
  })
}
```

これで画像の URL は `/demo/google-drive/proxy/[fileId]` になって、アクセストークンも隠せるし、CDNキャッシュも効かせられます。

## セキュリティ対策

### トークンの暗号化

アクセストークンとリフレッシュトークンは、そのままセッションに保存するのは怖いので、AES-256-GCM で暗号化してます。

### CSRF 対策

OAuth の state パラメータを使って、リクエストの正当性を確認。セッションに保存した値と照合して、一致しなければエラーにします。

### パス・トラバーサル対策

ファイル ID は正規表現でチェック。変な値が来たら 400 Bad Request を返すようにしてます。

## 感想

React Router v7 の Framework mode、なかなか使い勝手がいいです。特に Resource Routes が便利で、API エンドポイントをルート定義の中で完結できるのがいいですね。

Google Drive API も、最初は OAuth とか面倒そうだなと思ってたんですが、ちゃんと抽象化すれば意外とシンプルに実装できました。トークンのリフレッシュとか、エラーハンドリングとか、一度作っちゃえば使い回せますし。

React Router v7は、こういう外部 API 連携が必要なアプリには向いてると思います。Cloudflare Workers で動かせるのも魅力ですね。

詳しい実装はGitHubのコードを見てもらえればと思います！
https://github.com/techtalkjp/techtalk.jp/tree/main/app/routes/demo%2B/google-drive%2B
