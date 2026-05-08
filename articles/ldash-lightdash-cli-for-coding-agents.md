---
title: "Lightdash の BI データを Claude Code から触りたい ― エージェント前提の薄い CLI"
emoji: "📊"
type: "tech"
topics: ["lightdash", "claudecode", "cli", "ai", "typescript"]
published: true
---

## これはなに？

[Lightdash](https://www.lightdash.com/) という BI ツールのデータを Claude Code などのコーディングエージェントから触れるようにする CLI、ldash を作りました。

- npm: https://www.npmjs.com/package/@coji/ldash-cli
- GitHub: https://github.com/coji/ldash-cli

`npx @coji/ldash-cli setup` でブラウザ OAuth から始められます。

## なぜ作ったか

業務で Lightdash を使っているのですが、Claude Code から「先月の orders ステータスごとの件数を見せて」みたいに頼みたい場面が増えてきました。Lightdash は BI ツールなので、こういうクエリは UI で組み立てるのが基本ですが、[`@lightdash/cli`](https://www.npmjs.com/package/@lightdash/cli) という公式 CLI もあります。

ただ公式 CLI は **dbt 開発ワークフロー** （compile / deploy / preview / generate-schema）が主で、データを照会する用途には向いていません。SQL や explore を叩くのは API 経由になります。

最初は Claude Code に curl を直接書かせていましたが、PAT の管理、エラー文字列の解釈、入れ子 JSON をシェルから渡すときのクォートまわり、レスポンスの肥大化など、Lightdash 固有の作法が積み上がってきて見ていられなくなりました。「エージェントからデータを触る」だけのために、薄く包む CLI が欲しい、というのが出発点です。

## 設計のヒント

ちょうど書きはじめのタイミングで、[@nyosegawa](https://x.com/nyosegawa) さんの [Notion CLI for Coding Agent](https://nyosegawa.com/posts/notion-cli-for-coding-agent/) という記事を読みました。Notion Remote MCP をラップした CLI をエージェント前提で設計した話で、考え方の骨子をそのまま引き継がせてもらっています。

要点は 4 つでした。

1. デフォルト出力をエージェントが読みやすい形にする
2. コマンドの存在を伝える経路は `--help` とエラーヒントに寄せる
3. エラーは What + Why + Hint で構造化する
4. 生 API への抜け道を残しておく

ldash もこの 4 つは満たしたうえで、Lightdash の API 形状に合わせていくつか手を加えています。

## クイックツアー

セットアップはブラウザ OAuth が一番楽です。

```bash
npx @coji/ldash-cli setup
```

エージェントや CI 環境だとブラウザが開けないので、環境変数を直接渡す経路も用意しています。

```bash
export LIGHTDASH_API_URL=https://app.lightdash.cloud
export LIGHTDASH_API_KEY=<personal-access-token>
export LIGHTDASH_PROJECT_UUID=<project-uuid>
```

設定が揃っているかは `ldash doctor` で URL からプロジェクトアクセスまで通しで確認できます。

```bash
$ ldash doctor
Doctor: ✓ all checks passed

  ✓ apiUrl   https://app.lightdash.cloud (source: env)
  ✓ apiKey   present (source: env, env: LIGHTDASH_API_KEY)
  ✓ auth     token accepted by https://app.lightdash.cloud — 2 projects visible
  ✓ project  Sales (xxxxxxx-xxxx-...) — accessible
```

データを探すときは `ldash search` を最初に使います。

```bash
$ ldash search "orders" --kind chart,dashboard --compact --json | jq '.[0]'
{
  "kind": "chart",
  "uuid": "abc-123-...",
  "name": "Orders by Status",
  "description": "Daily breakdown of order statuses",
  "parent": "Sales",
  "nextCommand": "ldash chart get abc-123-..."
}
```

各ヒットに `nextCommand` が入っていて、エージェントが次に叩くべきコマンドをそのまま実行できます。`ldash explore get`、`ldash chart get`、`ldash dashboard get` のように、対象に応じた深掘りコマンドが返ります。

実際にクエリを投げるときはこんな感じです。

```bash
ldash query run orders \
  --dimensions '["orders_status"]' \
  --metrics '["orders_count"]' \
  --filters @./filters.json \
  --limit 10
```

`--filters` のような JSON フラグは入れ子が深くなるので、`@path/to/file.json` かハイフン `-`（標準入力）でファイル／パイプから読めるようにしています。シェルでのクォート地獄から逃げられて、エージェントもクォートのつけ忘れで事故を起こしません。

## エージェント向けに気をつけたところ

エージェント側から見て嬉しそうなところを 4 点だけ紹介します。

### 安定したエラーコード

`--json` を指定したとき、すべてのエラーは構造化された形で返ります。

```json
{
  "ok": false,
  "error": {
    "code": "EXPLORE_NOT_FOUND",
    "what": "Explore not found",
    "why": "...",
    "hint": "Run \"ldash explore list\" to see valid identifiers."
  }
}
```

code は安定した文字列の列挙で、AUTH_INVALID、EXPLORE_NOT_FOUND、RATE_LIMITED、NETWORK など 23 種類あります。エージェントは code を見て分岐すれば、メッセージ文字列を解析する必要がありません。

hint は具体的な次のコマンドを指します。たとえば explore が見つからないときは `ldash explore list`、フィールド名がおかしいときは `ldash explore get <id>` を実行するように促します。

### `--fields` / `--compact`

一覧系の API レスポンスは項目が多くて、エージェントの読み込み量を圧迫しがちです。`--fields` で必要なキーだけ抜き出し、`--compact` でコマンドごとに用意した妥当なデフォルト集合を使えるようにしました。

```bash
$ ldash chart list --compact --json
[
  { "uuid": "abc...", "name": "Orders by Status", "description": "..." },
  { "uuid": "def...", "name": "Daily Revenue", "description": "..." }
]
```

jq でパイプしても同じことはできるんですが、CLI 側で削っておくほうがエージェント経由のときには手数が少なくて済みます。

### `setup --check` と `doctor`

エージェントがセットアップをユーザーに案内するとき、「いま自分の環境で OAuth が走るのか」「PAT は有効なのか」を **副作用なしで** 知りたくなります。`ldash setup --check` は TTY や環境変数の状態を JSON で返すだけの下調べ用コマンドで、`ldash doctor` はそれより一歩踏み込んで API トークンとプロジェクトアクセスまで確認します。どちらも失敗時に終了コード 1 を返すので、`ldash doctor && deploy` のように CI のゲートとしても使えます。

### 生 API への抜け道

CLI で薄く包んでいるとはいえ、すべてのエンドポイントを CLI 側で覆うつもりはありません。CLI 側に用意がない API は `ldash api GET <path>` / `ldash api POST <path> --body @file.json` で直叩きできます。

```bash
ldash api GET /api/v1/org/projects
ldash api POST /api/v1/projects/{uuid}/sqlQuery --body @body.json
```

エージェントが「この機能は CLI に用意がないけど API はある」と判断したら、こちらで逃げられます。CLI が機能の縛りにならないようにする保険です。

## 公式 Lightdash CLI との関係

公式 [`@lightdash/cli`](https://www.npmjs.com/package/@lightdash/cli) は dbt 開発（compile / deploy / preview）に特化しています。ldash は逆で、データアクセス（query / explore / chart / dashboard 閲覧）専用です。

dbt 開発は公式 CLI、データ照会は ldash、という棲み分けで両立できます。

## 開発の感想

ほぼ全部 Claude Code に書いてもらいました。Notion CLI の記事を読み込ませて「これに沿った形で Lightdash 用に作って」と頼むところから始めて、`/simplify`（コードレビュー）と `/review`（自己レビュー）を回しながら詰めています。

途中で CodeRabbit の自動レビューも入って、「JSON のエラー構造を入れ子にしたのは破壊的変更です」のような誤検知も含めて 7〜8 件指摘をもらい、対応や反論のやりとりも Claude Code が代行してくれました。最終的に `pnpm validate` でフォーマット／lint／typecheck／114 件のテストが通る状態でリリースしています。

CLI 自体は薄い（コアは `src/api.ts` の OpenAPI 経由のクライアントと `src/cli.ts` の振り分けだけ）ので、似たようなエージェント向け CLI を別の SaaS 用に書くときのひな型にもなりそうです。

## まとめ

Lightdash の BI データを Claude Code などのエージェントから触りたいなら、`npx @coji/ldash-cli` で試せます。設計の骨格は [Notion CLI for Coding Agent](https://nyosegawa.com/posts/notion-cli-for-coding-agent/) に倣いました。安定したエラーコード、`nextCommand` 付きの search、`@file` で読める JSON フラグ、`--fields` での項目選別あたりが、エージェント視点での主な工夫です。

不具合や「こういう機能が欲しい」があれば issue / PR で歓迎します。
https://github.com/coji/ldash-cli
