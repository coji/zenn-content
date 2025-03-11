---
title: "Shadcn Admin Dashboard: Shadcn と React Router v7 で構築された管理ダッシュボード UI"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["shadcnui", "dashboard", "reactrouter", "remix", "conform", "react"]
published: true
---


![alt text](https://github.com/coji/shadcn-admin-react-router/blob/main/public/images/shadcn-admin.png?raw=true)

デモサイト
https://shadcn-admin-react-router.vercel.app

ソースコード
https://github.com/coji/shadcn-admin-react-router

## はじめに

Shadcn Admin Dashboard は、[ShadcnUI](https://ui.shadcn.com) と [React Router v7](https://reactrouter.com/) を使用して構築した、再利用可能な管理ダッシュボード UI のコレクションです。レスポンシブデザインとアクセシビリティに配慮して設計されています。

このプロジェクトは、[satnaing 氏](https://github.com/satnaing) による [shadcn-admin](https://github.com/satnaing/shadcn-admin) をフォークした上で、React Router v7 で動くように書き換えたものです。オリジナルの作者に感謝いたします。

## 特徴

- ライト/ダークモードの切り替え
- レスポンシブ対応
- アクセシビリティ対応
- 再利用可能なサイドバーコンポーネント
- グローバル検索コマンド
- 10 以上のページ
- 追加のカスタムコンポーネント

## 技術スタック

- **UI:** [ShadcnUI](https://ui.shadcn.com) (TailwindCSS + RadixUI)
- **ビルドツール:** [Vite](https://vitejs.dev/)
- **ルーティング:** [React Router v7](https://reactrouter.com/en/main) (フレームワーク)
- **型チェック:** [TypeScript](https://www.typescriptlang.org/)
- **リンティング/フォーマッティング:** [Biome](https://biomejs.dev/) & [Prettier](https://prettier.io/)
- **フォームバリデーション:** [Conform](https://conform.guide/)
- **アイコン:** [Tabler Icons](https://tabler.io/icons)

## ディレクトリ構造

プロジェクトのディレクトリ構造は以下の通りです。

```sh
├── .github
│   ├── FUNDING.yml
│   └── workflows
│       └── ci.yml
├── .gitignore
├── .prettierignore
├── CHANGELOG.md
├── CODE_OF_CONDUCT.md
├── LICENSE
├── README.md
├── app
│   ├── assets
│   │   └── vite.svg
│   ├── components
│   ├── context
│   ├── data
│   ├── hooks
│   ├── index.css
│   ├── lib
│   ├── root.tsx
│   ├── routes.ts
│   └── routes
├── biome.json
├── components.json
├── package.json
├── pnpm-lock.yaml
├── postcss.config.js
├── prettier.config.js
├── public
│   ├── avatars
│   ├── favicon.ico
│   └── images
├── react-router.config.ts
├── tailwind.config.js
├── tsconfig.json
└── vite.config.ts
```

## `routes` 下の主要ルート概説

`app/routes` ディレクトリには、アプリケーションのルーティング定義が含まれています。`route.tsx` ファイルがルートハンドラとなり、それを含むフォルダが主要ルートとなります。私が作成した主要なルートとその役割を以下に説明します。

### 認証済みユーザー用 (`_authenticated+`)

- **`_authenticated+/_layout/route.tsx`**: 認証済みユーザー用ページ全体のレイアウト
- **`_authenticated+/_index/route.tsx`**: ダッシュボードのホームページ
- **`_authenticated+/apps/route.tsx`**: アプリ連携ページ
- **`_authenticated+/chats/route.tsx`**: チャットページ
- **`_authenticated+/help-center/route.tsx`**: ヘルプセンターページ (Coming Soon)
- **`_authenticated+/settings+/_layout/route.tsx`**: 設定ページ全体のレイアウト
- **`_authenticated+/settings+/_index/route.tsx`**: プロフィール設定
- **`_authenticated+/settings+/account/route.tsx`**: アカウント設定ページ
- **`_authenticated+/settings+/appearance/route.tsx`**: 外観設定ページ
- **`_authenticated+/settings+/display/route.tsx`**: 表示設定ページ
- **`_authenticated+/settings+/notifications/route.tsx`**: 通知設定ページ
- **`_authenticated+/tasks/route.tsx`**: タスク管理ページ
- **`_authenticated+/users/route.tsx`**: ユーザー管理ページ

### 認証関連 (`_auth+`)

- **`_auth+/_layout/route.tsx`**: 認証関連ページ全体のレイアウト
- **`_auth+/sign-in/route.tsx`**: ログインページ
- **`_auth+/sign-in-2/route.tsx`**: ログインページ (2カラムレイアウト)
- **`_auth+/sign-up/route.tsx`**: 新規登録ページ
- **`_auth+/forgot-password/route.tsx`**: パスワード再設定ページ
- **`_auth+/otp/route.tsx`**: OTP (ワンタイムパスワード) 認証ページ

### エラーページ (`_errors+`)

- **`_errors+/401.tsx`**: 401 Unauthorized エラーページ
- **`_errors+/403.tsx`**: 403 Forbidden エラーページ
- **`_errors+/404.tsx`**: 404 Not Found エラーページ
- **`_errors+/500.tsx`**: 500 Internal Server Error エラーページ
- **`_errors+/503.tsx`**: 503 Service Unavailable エラーページ

## 使い方

1. リポジトリをクローンします。

```bash
git clone https://github.com/coji/shadcn-admin-react-router.git
```

2. プロジェクトディレクトリに移動します。

```bash
cd shadcn-admin-react-router
```

3. 依存関係をインストールします。

```bash
pnpm install
```

4. 開発サーバーを起動します。

```bash
pnpm run dev
```

## デプロイ

このプロジェクトは、デフォルトで [Vercel](https://vercel.com) にデプロイされるように設定しています。

React Router v7 は、サーバーサイドレンダリング (SSR) をサポートしており、[Cloudflare Workers](https://workers.cloudflare.com/) や [Fly.io](https://fly.io/)、[Cloud Run](https://cloud.google.com/run)、[AWS ECS](https://aws.amazon.com/ecs/) など、他のプラットフォームにもデプロイすることが可能です。

他のプラットフォームへのデプロイ方法については、[React Router v7 の公式サンプル](https://github.com/remix-run/react-router-templates) のコードを参考にしてください。

## ライセンス

このプロジェクトは [MIT ライセンス](https://choosealicense.com/licenses/mit/) のもとで公開しています。

## コントリビューション

コントリビューションは大歓迎です！このプロジェクトをより良くするために、ぜひあなたの力をお貸しください。詳細については、[コントリビューションガイド](https://github.com/coji/shadcn-admin-react-router/blob/main/CODE_OF_CONDUCT.md) をご覧ください。

## おわりに

Shadcn Admin Dashboard は、Shadcn と React Router v7 を使用して構築した、管理ダッシュボード UI です。レスポンシブデザインとアクセシビリティに配慮して設計されています。

このプロジェクトが、あなたの次の管理ダッシュボードの構築に役立つことを願っています！
