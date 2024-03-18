---
title: " Claude3 API で Haiku を使って Google 検索結果から各ページの企業業種を特定する CLI ツールを作ってみた"
emoji: "🔍"
type: "tech"
topics: ["typescript", "anthropic", "claude3", "haiku"]
published: true
---

# はじめに

こんにちは。今回は Claude3 API で Haiku を使って Google 検索結果から業種を特定する CLI ツールの作り方を紹介します。

TypeScript で実装していきますが、anthropic の SDK を使うので、比較的簡単に実装することができます。

また、cleye というライブラリを使って CLI ツールを作成します。

なお、今回作成したもののソースコードは以下で公開しています。

https://github.com/coji/haiku-example

# 準備

まずは必要なライブラリをインストールします。

```bash
npm install @anthropic-ai/sdk jsdom tsx cleye dotenv
```

[@anthropic-ai/sdk](https://github.com/anthropics/anthropic-sdk-typescript): anthropic の SDK
[jsdom](https://github.com/jsdom/jsdom): Node.js で HTML を解釈し、DOM 操作を行うためのライブラリ
[tsx](https://github.com/privatenumber/tsx): Typescript で作成されたスクリプトを直接実行するコマンド
[cleye](https://github.com/privatenumber/cleye): CLI ツールを作成するためのライブラリ
[dotenv](https://github.com/motdotla/dotenv): .env ファイルを読み込むためのライブラリ

# 実装

.env ファイルの作成
anthropic の API Key を .env ファイルに記載します。

```bash:.env
ANTHROPIC_API_KEY=YOUR_API_KEY
```

Anthropic でのアカウントの作成と、API キーの発行は、以下の記事が参考になります。

https://qiita.com/moritalous/items/8266f0c4bea6514713f7

## Google 検索結果から URL を取得する

listupUrlByGoogle 関数で Google 検索を行い、検索結果の URL を取得します。

```ts:commands/industry.ts
const listupUrlByGoogle = async (query: string) => {
  // Google検索を行い、検索結果ページのDOMを取得する
  const url = `https://www.google.com/search?q=${encodeURIComponent(query)}`;
  const window = await fetchWebDOM(url);

  // DOMから検索結果のURLリストを取得する
  const results = Array.from(window.document.querySelectorAll("h3"))
    .filter((h3) => h3.parentElement?.tagName === "A")
    .map((h3) => h3.parentElement?.getAttribute("href"))
    .filter(
      (text) =>
        text?.startsWith("http") && !text.startsWith("https://www.google.com/")
    );
  return results as string[];
};
```

fetchWebDOM 関数は jsdom を使って URL から Web ページの DOM を取得する関数です。
今回は Google の検索結果ページをパースして jsdom で解釈して textContent をコンテンツとして使っています。

Google 検索を頻繁にアクセスさせると、Bot 対策でしばらくアクセスができなくなくなりますのでご注意ください。

本格的に Google 検索をプログラムから使いたい場合は Custom Search API というものが公式にありますので、そちらのご利用を検討するとよいでしょう (100 件まで無料で、それ以上は 1000 クエリあたり$5。1 日あたり 1 万クエリまで)

https://developers.google.com/custom-search/v1/overview?hl=ja

## Web ページの内容を取得し、業種を判別する

extructAndDetectUrl 関数で Web ページの内容を取得し、anthropic の Claude に投げて業種を判別します。

```ts
const extructAndDetectUrl = async (url: string) => {
  const content = await fetchWebContent(url);

  const stream = await anthropic.messages.create({
    max_tokens: 1000,
    messages: [
      {
        role: "user",
        content: `Webページの内容を解析して、その会社の業種を判別し結果だけを出力してください:

出力結果: CSVフォーマット。項目は以下の4つ。
- URL
- 会社名
- 業種
- 会社概要要約

対象URL: ${url}
Webページの内容: ${content}
`,
      },
    ],
    model: "claude-3-haiku-20240307",
    stream: true,
  });

  for await (const message of stream) {
    if (message.type === "content_block_delta") {
      process.stdout.write(message.delta.text);
    }
  }
  console.log();
};
```

fetchWebContent 関数は jsdom を使って URL から Web ページのテキストを取得する関数です。

anthropic.messages.create で Claude に Web ページの内容を解析してもらい、業種を判別してもらいます。

stream: true オプションを指定することで、結果をストリーミングで取得できます。続く for await で取得した部分的なテキストを逐次・リアルタイムにコンソールに表示しています。

## CLI ツールの作成

[cleye](https://github.com/privatenumber/cleye) を使って CLI ツールを作成します。

```ts
import { cli } from "cleye";
import { commands } from "./commands";

const args = cli({
  name: "haiku",
  help: { description: "Claude3 Haiku API を使ってあれこれします" },
  commands: commands,
});
if (args.command === undefined) {
  args.showHelp();
}
```

commands には CLI ツールのコマンドを定義します。

```ts
export const commands = [
  command(
    {
      name: "industry",
      help: {
        description:
          "Google検索を行い1ページ目の結果からそれぞれの業種を特定します",
        examples: ["haiku industry-identification 業種の特定 会社概要"],
      },
      parameters: ["<query...>"],
    },
    ({ _ }) => industryCommand(_.query.join(" "))
  ),
  // ...
];
```

industryCommand 関数では、listupUrlByGoogle 関数で URL を取得し、extructAndDetectUrl 関数で業種を判別します。

```ts
export const industryCommand = async (query: string) => {
  const results = await listupUrlByGoogle(query);
  for (const result of results) {
    await extructAndDetectUrl(result);
    console.log();
  }
};
```

## 実行

CLI ツールを実行してみます。

```bash
$ npx tsx haiku.ts industry 業種の特定
https://example.com/,株式会社○○,IT・通信,株式会社○○は、IT・通信業界に属する企業です。主にソフトウェア開発やシステムインテグレーションを手掛けています。
https://example.net/,株式会社△△,製造業,株式会社△△は、製造業に属する企業です。主に自動車部品の製造を行っています。
```

Google 検索結果から取得した URL ごとに、業種が判別されて CSV 形式で出力されました。

# Haiku のコスト

今回のスクリプトを書くにあたって、何度も Web ページの内容を Claude API に渡して推論していました。どれくらいのコストがかかったか記録しておきます。

コスト: 0.3 ドル (45 円)
![](/images/claude3-haiku-crawl-company-detect-industry/billing.png)

入力トークン数: 100 万トークン
![](/images/claude3-haiku-crawl-company-detect-industry/input_tokens.png)

出力トークン数: 2 万トークン
![](/images/claude3-haiku-crawl-company-detect-industry/output_tokens.png)

Claude3 の価格はこんなかんじです。Haiku の安さが光りますね！

- Opus：100 万入力トークン当たり 15 ドル、100 万出力トークン当たり 75 ドル
- Sonnet：100 万入力トークン当たり 3 ドル、100 万出力トークン当たり 15 ドル
- Haiku：100 万入力トークン当たり 0.25 ドル、100 万出力トークン当たり 1.25 ドル

OpenAI だとこんなかんじ。

- GPT-4: 100 万入力トークンあたり 30 ドル、100 万出力トークン当たり 60 ドル
- GPT-4 Turbo: 100 万入力トークンあたり 10 ドル、100 万出力トークン当たり 30 ドル
- GPT-3.5 Turbo: 100 万入力トークンあたり 0.5 ドル、100 万出力トークン当たり 1.5 ドル

Haiku が最安ですが、性能は GPT-3.5 Turbo 並か、ちょっと上ぐらいに感じました。

# おわりに

今回は anthropic の Claude を使って Google 検索結果から業種を特定する CLI ツールの作り方を紹介しました。anthropic の SDK を使うことで、比較的簡単に Claude を使ったツールを作ることができます。

特に Haiku はコストも安く、プロンプトも最大 20 万トークンまで入れられますし、出力もとても速いので、あまり難しい工夫をすることなく適当に突っ込んで、抽出・要約するのにちょうど良いとおもいます。

皆さんも面白い使い方を考えて、色々なツールを作ってみてください。
