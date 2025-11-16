---
title: "Claude Code で SEO 問題を解決 ── prerender と localhost URL の罠"
emoji: "🔍"
type: "tech"
topics: ["seo", "claudecode", "ai", "reactrouter", "troubleshooting"]
published: true
---

## これはなに？

Google Search Console を見たら、数百ページあるサイトのうち10ページくらいしかインデックスされていなくて焦りました。原因がわからず困っていたんですが、Claude Code で調べてもらったら解決できたので、その体験を書いてみました。

## 問題の発見 ── localhost URL が本番に

Claude Code に本番の URL をチェックしてもらったところ、JSON-LD 構造化データに localhost の URL が含まれていることがわかりました。本番環境でこんなコードが配信されていたんです。

```json
{
  "@context": "http://schema.org",
  "@type": "LocalBusiness",
  "url": "http://localhost:3000/ja/area/marunouchi/cafe/rating"
}
```

これは Google がページを信頼できないと判断するので、かなりまずいです。他にも meta description が設定されていないページや、canonical URL の末尾スラッシュが統一されていないページも見つかりました。

なぜ localhost になったのか説明します。最初は prerender を使わず SSR だけで運用していて、meta タグや JSON-LD は `request.url` を使って実装していました。これは問題なく動いていたんです。ところが Fly.io + Cloudflare の構成でページ遷移が遅くなることがあって、パフォーマンス改善のために prerender を導入しました。

ここに罠がありました。**prerender 時のリクエストは localhost として処理されます**。ビルド時に生成される静的 HTML には localhost の URL が埋め込まれてしまっていたんです。SSR では実際のリクエスト URL が使われるので問題なかったんですが、prerender に切り替えた時にこの違いに気づきませんでした。

## Claude Code で原因を特定

次に Claude Code にコードベースを調べてもらいました。「JSON-LD の URL を確認したい」と頼んだら、関連するファイルを見つけてくれました。

```typescript
// apps/web/app/routes/_public.($lang)+/area.$area.$category.$rank/route.tsx
export const meta: Route.MetaFunction = ({ data, location }) => {
  return [
    {
      'script:ld+json': {
        '@context': 'http://schema.org',
        '@type': 'LocalBusiness',
        url: new URL(
          `${data.lang.path}area/${data.area.areaId}/${data.category.id}/${data.rankingType}`,
          data.url  // ← ここが問題！request.url をそのまま使用
        ).toString(),
      },
    },
  ]
}
```

Claude Code はすぐに「`data.url` が `request.url` から来ているので、開発環境では localhost になりますね」と教えてくれました。話していくうちに、loader 関数で `request.url` をそのまま返していて、meta 関数でその URL を使って JSON-LD を生成しているから、prerender 時に localhost の URL が含まれてしまうことがわかりました。

さらに Claude Code が既存の `generateCanonicalUrl` という関数を見つけてくれて、これを使う方法を提案してくれました。

## 修正の実装 ── 環境非依存の URL 生成

まず `generateCanonicalUrl` 関数を改善しました。環境に依存せず、常に正しい本番 URL を生成するようにしています。

```typescript
// apps/web/app/features/seo/canonical-url.ts
const CANONICAL_BASE_URL = 'https://tokyo.hyper-local.app'

export const generateCanonicalUrl = (pathname: string): string => {
  const cleanPath = pathname === '/' ? pathname : pathname.replace(/\/$/, '')
  return `${CANONICAL_BASE_URL}${cleanPath}`
}
```

そして JSON-LD の生成部分で、`request.url` の代わりに `generateCanonicalUrl` を使うように変更しました。これで環境に関係なく正しい URL が生成されるようになりました。

```typescript
export const meta: Route.MetaFunction = ({ data, location }) => {
  return [
    {
      'script:ld+json': {
        '@context': 'http://schema.org',
        '@type': 'LocalBusiness',
        url: generateCanonicalUrl(
          `${data.lang.path}area/${data.area.areaId}/${data.category.id}/${data.rankingType}`
        ),
      },
    },
  ]
}
```

あとはエリアとカテゴリーの組み合わせごとに meta description を動的生成する関数を追加したり、エリアページへのリンクで city を含む正しいドメインが生成されるように直しました。

## Claude Code の効果 ── 3つの助け

「JSON-LD を使っている箇所を探して」みたいな曖昧な頼み方でも、関連するファイルを全部見つけてくれました。大きなコードベースでも grep や IDE の検索を使うより効率的です。一つのファイルを直す時に影響を受けそうな他のファイルも教えてくれたので、修正漏れを防げました。

一度に全部直すんじゃなくて、優先度の高い問題から順番に対処する方法を提案してくれたのもよかったです。まず localhost URL の問題を修正して、次に meta description を追加して、最後に canonical URL を最適化する、という流れで進めました。各修正の効果を確認しながら進められました。

修正後は自動的に `pnpm run validate` でテストを実行して、lint、format、typecheck、test が全部成功することを確認してから次に進めてくれました。

## 結果と今後

修正をデプロイした後、Google Search Console でサイトマップを再送信して、重要なページで個別にインデックス登録をリクエストしました。1-2 週間待って再クロールされたら、インデックス状況の改善を確認する予定です。

もし改善が見られなかったら、レストラン個別ページの構造化データ追加や Open Graph タグの最適化、サイトマップの最適化なども試してみようかなと思っています。

## まとめ

Claude Code のおかげで、複雑な SEO 問題を体系的にデバッグして解決できました。大規模コードベースでも関連箇所を素早く特定してくれて、問題の根本原因をわかりやすく説明してくれます。段階的で実践的な解決策を提案してくれるだけでなく、修正後の動作確認まで含めてサポートしてくれました。

AI ペアプログラミングツールは、単なるコード補完ツールじゃなくて、問題解決のパートナーとして使えるんだなと実感しました。同じような SEO 問題で困っている方の参考になれば嬉しいです。
