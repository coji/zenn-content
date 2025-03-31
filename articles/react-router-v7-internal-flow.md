---
title: "React Router v7 の内部構造を探る：リクエストからレンダリングまでの道のり"
emoji: "🗺️"
type: "tech"
topics: ["react", "reactrouter", "vite", "frontend", "javascript"]
published: true
---

## はじめに

React Router は、React アプリケーションにおけるルーティングライブラリのデファクトスタンダードとして長年利用されてきました。v6 で Data API が導入され、フルスタックフレームワークとしての側面が強化されましたが、v7 ではさらに進化し、Vite との統合、Single Fetch、Lazy Loading といったモダンな機能がデフォルトで組み込まれ、より洗練された開発体験とパフォーマンスを提供します。

しかし、これらの機能がどのように連携し、ブラウザのリクエストがどのように処理され、最終的にページが表示されるのか、その内部構造は少し複雑に見えるかもしれません。

この記事では、React Router v7 で構築されたアプリケーションの動作フローを、主要なパッケージやコンポーネントの役割、データ取得の仕組み、レンダリングプロセスなどに焦点を当てて、内部構造の観点から解説します。具体的なコード例よりも、全体の流れと重要な概念の理解を目的とし、適宜 GitHub のソースコードへのリンクを付記します。

**対象読者:**

* React Router v6/v7 をある程度利用したことがある開発者
* React Router の内部的な仕組みやデータフローに興味がある方
* SSR/CSR、Single Fetch、Lazy Loading といった概念がどのように実装されているか知りたい方

## パッケージ構成の概観

React Router v7 はモノレポ構成を採用しており、機能ごとに複数のパッケージに分割されています。主要なパッケージとその役割を理解することが、全体の流れを把握する第一歩となります。

![](/images/react-router-v7-internal-flow/packages.png)

* **`react-router`**: ([packages/react-router](https://github.com/remix-run/react-router/tree/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router))
  * React Router のコアロジックを提供します。
  * `<Routes>`, `<Route>`, `<Outlet>`, `<Navigate>` といった基本的なコンポーネント。
  * `useLocation`, `useParams`, `useNavigate`, `useRoutes`, `useLoaderData`, `useActionData` などのコアフック。
  * プラットフォーム非依存のルーターカーネル (`@remix-run/router` から内部化)。
  * SSRやSingle Fetchに関連するコンポーネント (`<Meta>`, `<Links>`, `<Scripts>` など) もv7からこのパッケージに含まれます。
* **`react-router-dom`**: ([packages/react-router-dom](https://github.com/remix-run/react-router/tree/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router-dom))
  * DOM 環境（Web ブラウザ）向けの API を提供します。
  * `<BrowserRouter>`, `<HashRouter>`, `<Link>`, `<NavLink>` など。
  * v7 では、その多くの機能が `react-router` 本体に統合され、主に後方互換性やDOM特化のAPI（`<RouterProvider>` の `ReactDOM.flushSync` 利用など）を提供します。実質的には `react-router` を再エクスポートする部分が大きいです。
* **`@react-router/dev`**: ([packages/react-router-dev](https://github.com/remix-run/react-router/tree/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router-dev))
  * 開発体験を向上させるためのツールと CLI を提供します。
  * Vite プラグイン (`reactRouter()`): HMR、コード分割、SSR設定などを Vite と統合します。 (関連コード: [packages/react-router-dev/vite/plugin.ts](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router-dev/vite/plugin.ts))
  * CLI コマンド (`react-router dev`, `react-router build`): 開発サーバーの起動や本番ビルドを実行します。 (関連コード: [packages/react-router-dev/cli/run.ts](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router-dev/cli/run.ts))
  * 型生成 (`typegen`): ルート定義に基づいて型定義を自動生成します。 (関連コード: [packages/react-router-dev/typegen/index.ts](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router-dev/typegen/index.ts))
* **`@react-router/create-react-router`**: ([packages/create-react-router](https://github.com/remix-run/react-router/tree/252d928179e54e3bf0e99493fb6affb8ad294935/packages/create-react-router))
  * `npm create react-router` コマンドの実体で、プロジェクトの雛形を生成します。
* **`@react-router/node`**, **`@react-router/express`**, **`@react-router/cloudflare`** etc.:
  * 各サーバープラットフォーム用の **アダプター** を提供します。
  * プラットフォーム固有の Request/Response オブジェクトを標準の Web API に変換したり、プラットフォーム固有の機能（セッションストレージなど）を提供したりします。
  * `createRequestHandler` をエクスポートし、サーバーフレームワークとの連携を担います。 (例: [packages/react-router-express/server.ts#createRequestHandler](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router-express/server.ts#L44))
* **`@react-router/serve`**: ([packages/react-router-serve](https://github.com/remix-run/react-router/tree/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router-serve))
  * 本番環境用のシンプルな Node.js アプリケーションサーバー。`@react-router/express` を内部で使用しています。
* **`@react-router/fs-routes`**: ([packages/react-router-fs-routes](https://github.com/remix-run/react-router/tree/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router-fs-routes))
  * ファイルシステムベースのルーティング規約（Remix v2 形式）を提供します。`routes.ts` 内で使用できます。
* **`@react-router/remix-routes-option-adapter`**: ([packages/react-router-remix-routes-option-adapter](https://github.com/remix-run/react-router/tree/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router-remix-routes-option-adapter))
  * 従来の Remix の `remix.config.js` 内の `routes` オプション形式を `routes.ts` で使用するためのアダプター。

## 起動から初回表示までの道のり (SSRフロー)

ユーザーが最初にページにアクセスした際の、サーバーサイドでの処理の流れを見ていきましょう。

![](/images/react-router-v7-internal-flow/ssr.png)

1. **[サーバー] リクエスト受信とアダプター**
    * ブラウザからのリクエストが Express などのサーバーに到着します。
    * `@react-router/express` の `createRequestHandler` で生成されたハンドラーが呼び出されます。
    * アダプターは Express の `req` オブジェクトなどから標準の `Request` オブジェクトを作成します。
        * (関連コード: [packages/react-router-express/server.ts#createRemixRequest](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router-express/server.ts#L95))

2. **[サーバー] ルートマッチングとデータ取得 (Core Handler)**
    * アダプターから `react-router` のコアハンドラー (`createRequestHandler` で作成されたもの) が呼び出されます。
        * (関連コード: [packages/react-router/lib/server-runtime/server.ts#createRequestHandler](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/server-runtime/server.ts#L81))
    * 内部で `createStaticHandler` が利用され、URL パス名に基づいてルート定義 (`build.routes`) と照合し、マッチするルート (`matches`) を特定します。
        * (関連コード: [packages/react-router/lib/router/router.ts#createStaticHandler](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/router/router.ts#L431))
    * `staticHandler.query()` が呼び出され、マッチした各ルートの `loader` 関数が **並列** で実行されます。
        * (関連コード: [packages/react-router/lib/router/router.ts#queryImpl](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/router/router.ts#L774))
    * すべての `loader` が完了すると、結果 (データ、リダイレクト、エラー) を含む `StaticHandlerContext` が生成されます。

3. **[サーバー] サーバーサイドレンダリング (SSR)**
    * サーバーエントリーポイント (`entry.server.tsx` の `default` export) が呼び出されます。
    * React の `renderToPipeableStream` (または環境に応じたAPI) が `<ServerRouter>` コンポーネントをレンダリングします。
        * (関連コード: [packages/react-router/lib/dom/ssr/server.tsx#ServerRouter](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/dom/ssr/server.tsx#L24))
    * `<ServerRouter>` は内部で `createStaticRouter` を使い、`StaticHandlerContext` から受け取ったデータで初期化されたルーターインスタンスを作成します。
        * (関連コード: [packages/react-router/lib/dom/server.tsx#createStaticRouter](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/dom/server.tsx#L270))
    * React コンポーネントツリー (`root.tsx` から `<Outlet>` を通じて) がレンダリングされます。
    * `<Meta />`, `<Links />`, `<Scripts />` が適切なタグをHTMLに挿入します。
        * `<Scripts />` は、ハイドレーションに必要なデータ (`loaderData`, `actionData`, `errors`) を `window.__reactRouterContext` としてインライン `<script>` にシリアライズし、クライアントJSのエントリーポイントを読み込む `<script type="module">` も生成します。
        * (関連コード: [packages/react-router/lib/dom/ssr/components.tsx#Scripts](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/dom/ssr/components.tsx#L645))

4. **[サーバー] レスポンス送信**
    * レンダリングされたHTMLストリームと、収集されたヘッダー（`Set-Cookie` など）を含む `Response` オブジェクトが構築されます。
    * プラットフォームアダプターがこの `Response` をサーバーフレームワークのレスポンスに変換してクライアントに送信します。
        * (関連コード: [packages/react-router-express/server.ts#sendRemixResponse](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router-express/server.ts#L138))

## クライアントサイドの魔法: ハイドレーション

サーバーから送られてきたHTMLを、ブラウザ上でインタラクティブなReactアプリケーションとして復元するプロセスです。

![](/images/react-router-v7-internal-flow/hydration.png)

1. **[ブラウザ] アセット受信と解析:**
    * ブラウザはHTMLを受信し、DOMツリーを構築します。`<link>` タグによりCSSやJSモジュールのダウンロードが始まります。
2. **[ブラウザ] クライアントエントリー実行:**
    * `<Scripts />` が出力した `<script type="module">` により `entry.client.tsx` が実行されます。
3. **[ブラウザ/React] ハイドレーション:**
    * `hydrateRoot` が呼び出されます。
    * `<HydratedRouter>` (または `<RouterProvider>`) がレンダリングされます。
        * (関連コード: [packages/react-router/lib/dom-export/hydrated-router.tsx#HydratedRouter](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/dom-export/hydrated-router.tsx#L241))
    * 内部で `createHydratedRouter` が呼び出され、`window.__reactRouterContext` 等からサーバーの状態 (`loaderData`, `errors` など) を読み取ります。Single Fetch のストリームデータもこのタイミングでデコードされ始めます。
        * (関連コード: [packages/react-router/lib/dom-export/hydrated-router.tsx#createHydratedRouter](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/dom-export/hydrated-router.tsx#L64))
    * `createBrowserRouter` (または `createHashRouter`) を使ってクライアントルーターが初期化されます。
    * React はサーバー生成のHTMLとクライアント生成のコンポーネントツリーを比較し、イベントリスナーをアタッチしてDOMをインタラクティブにします。
4. **[ブラウザ] 完了:**
    * ハイドレーションが完了し、アプリケーションが操作可能になります。`useEffect` などが実行され、必要に応じて `<ScrollRestoration>` がスクロール位置を調整します。

## ページ遷移の裏側 (クライアントサイドナビゲーション)

ハイドレーション後、ユーザーが `<Link>` をクリックした際の動作です。

![](/images/react-router-v7-internal-flow/csr.png)

1. **`<Link>` クリックとイベント抑制:**
    * クリックイベントが発生しますが、`useLinkClickHandler` が `event.preventDefault()` を呼び、ブラウザのデフォルト動作をキャンセルします。
        * (関連コード: [packages/react-router/lib/dom/lib.tsx#useLinkClickHandler](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/dom/lib.tsx#L249))
2. **`router.navigate()`:**
    * `<Link>` は `router.navigate()` を呼び出し、React Routerにナビゲーションの開始を伝えます。
3. **状態更新とHistory API:**
    * ルーターは `state.navigation` を `'loading'` に更新します。
    * History API (`pushState` または `replaceState`) を操作してブラウザのURLを更新します。
4. **ルートマッチングとLazy Loading:**
    * 新しいURLに対して `matchRoutes` が実行されます。
    * マッチしたルートに `lazy` があれば、モジュール読み込みが開始されます。
        * (関連コード: [packages/react-router/lib/router/router.ts#matchLoaderMatches](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/router/router.ts#L1663) 付近)
        * (関連コード: [packages/react-router/lib/dom/ssr/routes.tsx#createClientRoutes](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/dom/ssr/routes.tsx#L224) の `lazy` プロパティ)
5. **Single Fetch (Loader):**
    * 再検証が必要な `loader` (または `clientLoader`) を特定します (`shouldRevalidate` 考慮)。
    * 必要なルートIDを含む `.data` エンドポイント (例: `/page-b.data?_routes=id1,id2`) に対して **単一の GET fetchリクエスト** を送信します。
        * (関連コード: [packages/react-router/lib/dom/ssr/single-fetch.tsx#getSingleFetchDataStrategy](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/dom/ssr/single-fetch.tsx#L140))
    * サーバー側 (`singleFetchLoaders`) は対応する `loader` を実行し、結果をエンコードして返します。
        * (関連コード: [packages/react-router/lib/server-runtime/single-fetch.ts#singleFetchLoaders](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/server-runtime/single-fetch.ts#L149))
    * クライアントはレスポンスをデコード (`decodeViaTurboStream`) します。
        * (関連コード: [packages/react-router/lib/dom/ssr/single-fetch.tsx#decodeViaTurboStream](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/dom/ssr/single-fetch.tsx#L587))
6. **状態更新とUIレンダリング:**
    * Lazy Loadingとデータ取得が完了したら、`state.loaderData` 等が更新されます。
    * `<RouterProvider>` がトリガーされ、`useRoutesImpl` が新しいコンポーネントツリーを計算し、`<Outlet>` などが更新されます。
7. **スクロール復元:**
    * `<ScrollRestoration>` が遷移前ページのスクロール位置を保存し、遷移先ページのスクロール位置を復元（またはリセット）します。
8. **ナビゲーション完了:**
    * `state.navigation` が `'idle'` に戻ります。

## データの変更 (Form Submission / Action)

`<Form>` 送信時の流れはナビゲーションと似ていますが、Actionの実行とそれに続くRevalidationが含まれます。

![](/images/react-router-v7-internal-flow/form-submission.png)

1. **`<Form>` サブミットとイベント抑制:**
    * サブミットイベントが発生し、`event.preventDefault()` でデフォルト動作をキャンセルします。
2. **`router.navigate()`:**
    * フォームの情報（action, method, formDataなど）を使って `router.navigate()` が呼び出されます。
3. **状態更新 (Submitting):**
    * `state.navigation` が `'submitting'` になり、フォームデータなどが格納されます。
4. **ルートマッチングとLazy Loading (Action):**
    * Actionに対応するルートを特定し、必要なら `lazy()` を実行します。
5. **Single Fetch (Action):**
    * `.data` エンドポイントに **単一の POST (等) fetchリクエスト** を送信します。
        * (関連コード: [packages/react-router/lib/dom/ssr/single-fetch.tsx#getSingleFetchDataStrategy](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/dom/ssr/single-fetch.tsx#L140))
    * サーバー側 (`singleFetchAction`) は対応する `action` 関数を実行します。
        * (関連コード: [packages/react-router/lib/server-runtime/single-fetch.ts#singleFetchAction](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/server-runtime/single-fetch.ts#L49))
    * クライアントはレスポンス (Action Data または Redirect) を受け取ります。
6. **状態更新とリダイレクト/Revalidation:**
    * `state.actionData` が設定されます。
    * Actionがリダイレクトを返した場合、新しいナビゲーションが `replace: true` で開始されます。
    * リダイレクトがない場合、**Revalidation** がトリガーされます。関連する `loader` が再度 Single Fetch (GET) で呼び出されます。
        * (関連コード: [packages/react-router/lib/router/router.ts#handleActionResult](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/router/router.ts#L1361) 付近)
7. **UIレンダリング、スクロール処理、完了:**
    * ナビゲーションと同様に、UIが更新され、スクロールが処理され、`state.navigation` が `'idle'` に戻ります。

## キーとなる概念の深掘り

* **Vite統合 (`@react-router/dev`):** ビルド時の最適化（コード分割）、開発時の高速な HMR、サーバーリクエストハンドリングの仲介など、React Router をフレームワークとして機能させるための重要な役割を担います。
  * (関連コード: [packages/react-router-dev/vite/plugin.ts](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router-dev/vite/plugin.ts))
* **Single Fetch:** ナビゲーションやサブミッション時に発生する複数のデータ取得/更新リクエストを1つにまとめる仕組みです。これにより、ネットワークのオーバーヘッドを削減し、ウォーターフォール問題を緩和します。サーバー側とクライアント側で協調して動作します。
  * (関連コード: [packages/react-router/lib/dom/ssr/single-fetch.tsx](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/dom/ssr/single-fetch.tsx))
  * (関連コード: [packages/react-router/lib/server-runtime/single-fetch.ts](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/server-runtime/single-fetch.ts))
* **Lazy Loading:** ルート定義時に `lazy()` 関数を指定することで、そのルートが実際に必要になるまで関連モジュールの読み込みを遅延させます。初期バンドルサイズを削減し、アプリケーションの起動時間を短縮します。
  * (関連コード: [packages/react-router/lib/router/utils.ts#LazyRouteFunction](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/router/utils.ts#L428))
  * (関連コード: [packages/react-router/lib/dom/ssr/routes.tsx#createClientRoutes](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/dom/ssr/routes.tsx#L224))
* **`<RouterProvider>` と状態管理:** ルーターの現在の状態（location, loaderData, navigation stateなど）を一元管理し、コンテキストを通じて配下のコンポーネントに提供します。これにより、フック (`useLocation`, `useLoaderData`, `useNavigation`など) を使って状態にアクセスできます。
  * (関連コード: [packages/react-router/lib/components.tsx#RouterProvider](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/components.tsx#L230))
  * (関連コード: [packages/react-router/lib/context.ts](https://github.com/remix-run/react-router/blob/252d928179e54e3bf0e99493fb6affb8ad294935/packages/react-router/lib/context.ts))
* **アダプター:** Express, Cloudflare Workers, Node.js httpサーバーなど、様々な実行環境の差異を吸収し、共通のインターフェースでReact Routerを利用可能にします。

## まとめ

React Router v7 は、単なるルーティングライブラリを超え、データ取得、ミューテーション、レンダリング、開発ツールを統合したフルスタックに近いフレームワークへと進化しました。Vite との緊密な連携、Single Fetch による効率的なデータ通信、Lazy Loading によるパフォーマンス最適化などがその核となる特徴です。

この記事で解説した内部構造の流れを理解することで、アプリケーションの動作をより深く把握し、デバッグやパフォーマンスチューニングに役立てることができるでしょう。

**参考資料:**

* React Router 公式ドキュメント: [https://reactrouter.com/](https://reactrouter.com/)
* React Router GitHub リポジトリ: [https://github.com/remix-run/react-router](https://github.com/remix-run/react-router)

---
**免責事項:** この記事は React Router v7.4.1 時点の情報に基づいており、内部実装は将来のバージョンで変更される可能性があります。GitHub のリンクは特定のコミットハッシュ (`252d928...`) を指していますが、最新の情報はリポジトリの最新状態を確認してください。
