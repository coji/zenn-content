---
title: "Remix のドキュメントを Gemini 1.5 Flash で日本語化しました"
emoji: "💿"
type: "tech"
topics: ["remix", "react", "typescript"]
published: true
---
# Remix ドキュメント日本語版を公開しました

## はじめに

この度、Remix の公式ドキュメントを日本語に翻訳し、「Remix ドキュメント日本語版」というサイトで公開しました。

https://remix-docs-ja.techtalk.jp

本記事では、Remix ドキュメントの翻訳を Remix を使って開発した内容について技術的な側面から説明します。また、これから Remix を使った開発を始めたい方に向けて、日本語版ドキュメントを活用いただく方法についてもご紹介します。

## 日本語版ドキュメントの構成

今回翻訳したRemixドキュメントは、以下のようなカテゴリに分かれています。

- start: 5分でできるチュートリアルなど、Remixを始めるためのガイド。
- discussion: Remixの設計思想や概念の解説。
- file-conventions: ファイルベースルーティングや特別なファイルの規約。
- route: ルートモジュールの仕様。
- components: Remixが提供する React コンポーネントのリファレンス。
- hooks: Remixが提供する React Hooks のリファレンス。
- utils: Remixが提供するユーティリティ関数のリファレンス。
- styling: Tailwindcss や CSS Modules等でのスタイリングの方法。
- other-api: その他の Remix が提供するAPI。
- guides: 各機能の詳細な使い方。SPAモードや、Vite の設定などもこちら。

これらを網羅的に翻訳することで、日本の開発者がRemixを理解し、活用するための情報を提供しています。

## 翻訳の手法

今回のドキュメント翻訳では、Google が開発した大規模言語モデル Gemini 1.5 Flash を活用しました。翻訳処理では以下のようなシンプルなプロンプトを使用しています。

```txt
Translate the following text to Japanese. Markdowns should be left intact:
```

このプロンプトにより、Gemini 1.5 Flash はマークダウンの記法を維持しつつ、英語のドキュメントを日本語に翻訳してくれます。

翻訳のフローは以下のようになっています。

1. プロジェクトごとにドキュメントファイルを収集
2. 各ファイルの内容をGemini APIに渡して翻訳
3. 翻訳結果をデータベースに保存
4. 必要に応じて再利用

Gemini 1.5 Flash の利用により、手作業のみでは難しい大量のドキュメント翻訳を効率的に進めることができました。

翻訳に使ったコードを以下で公開しています。
これも remix で作ったもので、手元PCで実行して翻訳を行いました。

https://github.com/coji/oss-translation

## Remixによる日本語版サイトの開発

翻訳したドキュメントは、Remixを使って構築したWebサイトで公開しています。Remixの特徴を活かし、以下のような工夫をしました。

### マークダウンの動的ルーティング

`app/routes/_root.$/route.tsx` では、マークダウンファイルのパスをもとに動的なルーティングを行なっています。これにより、マークダウンファイルを追加するだけで自動的にページが生成されます。

### サイドメニューの自動生成

`app/routes/_root.$/functions/build-menu.ts` では、マークダウンファイルの構成をもとにサイドメニューのツリー構造を自動生成しています。カテゴリ分けや並び順は、マークダウンのYAMLヘッダーで制御できます。

### 言語切り替えリンクの追加

各ページのヘッダーには、英語版の該当ページへのリンクを配置しました。これにより、日本語と英語を行き来しながらドキュメントを読むことができます。

このサイトのソースコードは以下で公開しています。
日本語に翻訳した markdown ファイルも同じリポジトリに含めました。

https://github.com/coji/remix-docs-ja

## Remixで開発を始めるには

本サイトを参考に、ぜひRemixでの開発にチャレンジしてみてください。おすすめの学習ステップは以下のとおりです。

1. `start` で基本的な使い方をつかむ
2. `discussion` で設計思想や概念を深く知る
3. `hooks` や `components` を活用してコーディングする
4. `file-conventions` や `other-api` で発展的なトピックを学ぶ
5. `guides` で各機能の詳細を理解する
6. `other-api` で発展的なトピックを学ぶ

特に、チュートリアルが終わったタイミングで、`discussion`にある以下の記事を読むのをおすすめします。

1. [フルスタックデータフロー](https://remix-docs-ja.techtalk.jp/discussion/data-flow)
2. [プログレッシブエンハンスメント](https://remix-docs-ja.techtalk.jp/discussion/progressive-enhancement)
3. [状態管理](https://remix-docs-ja.techtalk.jp/discussion/state-management)
4. [フォーム vs. フェッチャー](https://remix-docs-ja.techtalk.jp/discussion/form-vs-fetcher)

Remix の流儀を理解することで、シンプルに機能を実装できるようになるでしょう。

また、実際のコードを読むのも理解が深まるでしょう。
本サイトのソースコードもGitHubで公開しているので、参考にしてみてください。

https://github.com/coji/remix-docs-ja

## さいごに

Remix ドキュメントの日本語化により、日本国内でのRemixの普及が進むことを願っています。翻訳の改善やアップデートも継続的に行なっていく予定です。

フィードバックやコントリビューションもお待ちしています。みなさまのRemixライフに、本ドキュメントが役立つことを祈っています。
