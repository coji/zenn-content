---
title: 'Cloudflareで追加料金をかけずに動かすマルチテナントSaaSの日本語全文検索'
emoji: '🔎'
type: 'tech'
topics: ['cloudflare', 'durableobjects', 'sqlite', 'fts5', 'workers']
published: false
---

## これはなに？

ひとりで開発している共有サービス（Artifact Share）に、ファイルの本文まで検索できる全文検索機能を追加しました。テナントごとに Cloudflare の Durable Objects を1つ割り当て、その中の SQLite FTS5 で検索インデックスを構築しています。外部の検索サービスは使っていません。

本番環境で動かした直近の使用量を1か月に換算すると、次のようになります。いずれも Workers Paid プラン（月 $5）の無料枠内に収まっており、検索機能のための追加料金は発生していません。

| 項目 | 1か月の使用量（概算） | Workers Paid に含まれる量 |
|---|---|---|
| Durable Objects のリクエスト | 50万回 | 100万回 |
| Durable Objects の実行時間 | 3万 GB-s | 40万 GB-s |
| SQLite の書き込み | 250万行 | 5,000万行 |
| SQLite の保存容量 | 0.4GB | 5GB |

この記事では、実現のために行った工夫を6つに分けて、コードと数値を交えながら紹介します。最後に、AI にそのまま渡せるチェックリストも載せています。

## 1. テナントごとに Durable Object を1つ持たせる

検索インデックスは、ワークスペース（テナント）ごとに1つの Durable Object に配置します。こうすることで、検索クエリにテナントで絞り込む条件を書く必要がなくなり、他社の文書が検索結果に混ざる心配もなくなります。

```html-embed
<div class="zd-wrap">
  <div class="zd-head">
    <span class="zd-tag">アーキテクチャ</span>
    <span class="zd-title">テナントごとに独立した Durable Object（SQLite FTS5）</span>
  </div>
  <div class="zd-grid-3">
    <div class="zd-card zd-do">
      <div class="zd-card-h">Tenant A 用 DO</div>
      <div class="zd-card-b">
        <div class="zd-pill">SQLite FTS5</div>
        <ul class="zd-list">
          <li>A社の文書のみ格納</li>
          <li>A社専用の BM25 統計</li>
        </ul>
      </div>
    </div>
    <div class="zd-card zd-do">
      <div class="zd-card-h">Tenant B 用 DO</div>
      <div class="zd-card-b">
        <div class="zd-pill">SQLite FTS5</div>
        <ul class="zd-list">
          <li>B社の文書のみ格納</li>
          <li>B社専用の BM25 統計</li>
        </ul>
      </div>
    </div>
    <div class="zd-card zd-warn">
      <div class="zd-card-h">D1（正本・バックアップ対象）</div>
      <div class="zd-card-b">
        <div class="zd-pill zd-pill-warn">FTS5 は置かない</div>
        <ul class="zd-list">
          <li>FTS5 仮想テーブルがあると <code>wrangler d1 export</code> 不可</li>
        </ul>
      </div>
    </div>
  </div>
  <div class="zd-foot">
    1つの Durable Object 上限は <strong>10GB</strong>（PoC 10万件で約 <strong>5GB</strong>）
  </div>
</div>
<style>
  .zd-wrap {
    --zd-bg: #f8fafc;
    --zd-surface: #ffffff;
    --zd-border: #cbd5e1;
    --zd-text: #0f172a;
    --zd-muted: #475569;
    --zd-accent: #0284c7;
    --zd-accent-soft: #e0f2fe;
    --zd-warn: #d97706;
    --zd-warn-soft: #fef3c7;
    font-family: "Hiragino Kaku Gothic ProN", "Hiragino Sans", "Yu Gothic UI", "Meiryo", system-ui, sans-serif;
    font-feature-settings: "palt";
    font-variant-numeric: tabular-nums;
    line-height: 1.65;
    letter-spacing: 0.02em;
    word-break: normal;
    overflow-wrap: anywhere;
    background: var(--zd-bg);
    color: var(--zd-text);
    border: 1px solid var(--zd-border);
    border-radius: 10px;
    padding: 16px;
  }
  [data-theme="dark"] .zd-wrap {
    --zd-bg: #0f172a;
    --zd-surface: #1e293b;
    --zd-border: #334155;
    --zd-text: #f1f5f9;
    --zd-muted: #94a3b8;
    --zd-accent: #38bdf8;
    --zd-accent-soft: #0c4a6e;
    --zd-warn: #fbbf24;
    --zd-warn-soft: #451a03;
  }
  .zd-head { display: flex; align-items: center; gap: 8px; margin-bottom: 12px; flex-wrap: wrap; }
  .zd-tag {
    font-size: 11px; font-weight: 700; padding: 2px 8px; border-radius: 999px;
    background: var(--zd-accent-soft); color: var(--zd-accent);
  }
  .zd-title { font-size: 14px; font-weight: 700; }
  .zd-grid-3 { display: grid; grid-template-columns: repeat(3, 1fr); gap: 10px; }
  .zd-card {
    background: var(--zd-surface); border: 1.5px solid var(--zd-border);
    border-radius: 8px; padding: 12px;
  }
  .zd-do { border-color: var(--zd-accent); }
  .zd-warn { border-color: var(--zd-warn); }
  .zd-card-h { font-size: 13px; font-weight: 700; margin-bottom: 6px; }
  .zd-pill {
    display: inline-block; font-size: 11px; font-weight: 700; padding: 1px 7px;
    border-radius: 4px; background: var(--zd-accent-soft); color: var(--zd-accent); margin-bottom: 8px;
  }
  .zd-pill-warn { background: var(--zd-warn-soft); color: var(--zd-warn); }
  .zd-list { margin: 0; padding-left: 16px; font-size: 12px; color: var(--zd-muted); }
  .zd-list li + li { margin-top: 4px; }
  .zd-list code { font-size: 11px; padding: 1px 4px; border-radius: 4px; background: rgba(128,128,128,0.15); }
  .zd-foot { margin-top: 12px; font-size: 12px; color: var(--zd-muted); text-align: right; }
  @media (max-width: 640px) {
    .zd-grid-3 { grid-template-columns: 1fr; }
    .zd-foot { text-align: left; }
  }
</style>
```

BM25 のスコア計算に使う単語の統計情報も、そのテナントの文書だけで計算されます。1つの Durable Object は 10GB まで利用でき、PoC で10万件のデータを登録した際は約 5GB でした。

なお、D1 も SQLite ベースなので FTS5 は使えますが、FTS5 の仮想テーブルがあると `wrangler d1 export` が使えなくなります。バックアップを export で取得している場合は、D1 に FTS5 テーブルを置かないほうが安全です。

## 2. 配置リージョンを固定し、オブジェクト名に含める

Durable Object は最初に作成された場所で配置リージョンが決まり、あとから変更できません。利用者の多いリージョンに固定し、そのリージョン名をオブジェクト名に含めておけば、どこに配置したかを D1 に記録して毎回参照する処理が不要になります。

```html-embed
<div class="zd-wrap">
  <div class="zd-head">
    <span class="zd-tag">命名規則</span>
    <span class="zd-title">オブジェクト名に配置リージョンを埋め込む</span>
  </div>
  <div class="zd-anatomy">
    <div class="zd-seg zd-seg-a">
      <div class="zd-seg-code">${workspaceId}</div>
      <div class="zd-seg-desc">テナント識別子</div>
    </div>
    <div class="zd-sep">:</div>
    <div class="zd-seg zd-seg-b">
      <div class="zd-seg-code">0</div>
      <div class="zd-seg-desc">将来の分割番号</div>
    </div>
    <div class="zd-sep">@</div>
    <div class="zd-seg zd-seg-c">
      <div class="zd-seg-code">apac-ne</div>
      <div class="zd-seg-desc"><code>locationHint</code> と一致</div>
    </div>
  </div>
  <div class="zd-foot">D1 への場所記録・毎回参照が不要になり、リージョン変更時は名前を変えて作り直す</div>
</div>
<style>
  .zd-wrap {
    --zd-bg: #f8fafc; --zd-surface: #ffffff; --zd-border: #cbd5e1;
    --zd-text: #0f172a; --zd-muted: #475569; --zd-accent: #0284c7; --zd-accent-soft: #e0f2fe;
    font-family: "Hiragino Kaku Gothic ProN", "Hiragino Sans", "Yu Gothic UI", "Meiryo", system-ui, sans-serif;
    font-feature-settings: "palt"; line-height: 1.65; letter-spacing: 0.02em;
    word-break: normal; overflow-wrap: anywhere;
    background: var(--zd-bg); color: var(--zd-text);
    border: 1px solid var(--zd-border); border-radius: 10px; padding: 16px;
  }
  [data-theme="dark"] .zd-wrap {
    --zd-bg: #0f172a; --zd-surface: #1e293b; --zd-border: #334155;
    --zd-text: #f1f5f9; --zd-muted: #94a3b8; --zd-accent: #38bdf8; --zd-accent-soft: #0c4a6e;
  }
  .zd-head { display: flex; align-items: center; gap: 8px; margin-bottom: 12px; flex-wrap: wrap; }
  .zd-tag {
    font-size: 11px; font-weight: 700; padding: 2px 8px; border-radius: 999px;
    background: var(--zd-accent-soft); color: var(--zd-accent);
  }
  .zd-title { font-size: 14px; font-weight: 700; }
  .zd-anatomy { display: flex; align-items: center; justify-content: center; gap: 6px; flex-wrap: wrap; }
  .zd-seg {
    background: var(--zd-surface); border: 1.5px solid var(--zd-border);
    border-radius: 8px; padding: 10px 14px; text-align: center; min-width: 120px;
  }
  .zd-seg-c { border-color: var(--zd-accent); background: var(--zd-accent-soft); }
  .zd-seg-code { font-family: ui-monospace, SFMono-Regular, Menlo, monospace; font-size: 14px; font-weight: 700; }
  .zd-seg-desc { font-size: 12px; color: var(--zd-muted); margin-top: 4px; }
  .zd-seg-desc code { font-size: 11px; padding: 1px 4px; border-radius: 4px; background: rgba(128,128,128,0.15); }
  .zd-sep { font-family: ui-monospace, monospace; font-size: 18px; font-weight: 700; color: var(--zd-muted); }
  .zd-foot { margin-top: 12px; font-size: 12px; color: var(--zd-muted); text-align: center; }
  @media (max-width: 640px) {
    .zd-anatomy { flex-direction: column; align-items: stretch; }
    .zd-sep { text-align: center; }
  }
</style>
```

```ts
export const SEARCH_INDEX_LOCATION = 'apac-ne'

export function searchIndexName(workspaceId: string): string {
  return `${workspaceId}:0@${SEARCH_INDEX_LOCATION}`
}

export function workspaceSearchIndex(namespace, workspaceId) {
  return namespace.getByName(searchIndexName(workspaceId), {
    locationHint: SEARCH_INDEX_LOCATION,
  })
}
```

`:0` は将来インデックスを分割するときのための番号です。リージョンを変更する場合はオブジェクト名が変わるため、新しい名前で作り直してから古いオブジェクトを削除します。

## 3. テーブルは初回の同期時に作成する

検索リクエストは、まだ一度もデータを同期していないテナントの Durable Object にも届きます。そのタイミングでテーブルを作成してしまうと、検索されただけのテナントにもストレージ費用がかかります。そのため、コンストラクタではテーブルの有無だけを確認し、実際の作成は同期処理のときに行うようにしました。

```html-embed
<div class="zd-wrap">
  <div class="zd-head">
    <span class="zd-tag">遅延初期化</span>
    <span class="zd-title">未同期テナントにストレージ費用を発生させない</span>
  </div>
  <div class="zd-grid-3">
    <div class="zd-card">
      <div class="zd-card-h">検索のみ（未同期）</div>
      <div class="zd-card-b">テーブルを作らず空配列を返す。<strong>ストレージ費用 0</strong>。</div>
    </div>
    <div class="zd-card zd-do">
      <div class="zd-card-h">初回 <code>sync()</code> 実行</div>
      <div class="zd-card-b">ここで初めて SQLite テーブルを作成して索引を構築。</div>
    </div>
    <div class="zd-card zd-warn">
      <div class="zd-card-h">名前とテナント不一致</div>
      <div class="zd-card-b"><code>deleteAll()</code> で中身を削除し、古いオブジェクトを自動掃除。</div>
    </div>
  </div>
</div>
<style>
  .zd-wrap {
    --zd-bg: #f8fafc; --zd-surface: #ffffff; --zd-border: #cbd5e1;
    --zd-text: #0f172a; --zd-muted: #475569; --zd-accent: #0284c7; --zd-accent-soft: #e0f2fe;
    --zd-warn: #d97706;
    font-family: "Hiragino Kaku Gothic ProN", "Hiragino Sans", "Yu Gothic UI", "Meiryo", system-ui, sans-serif;
    font-feature-settings: "palt"; line-height: 1.65; letter-spacing: 0.02em;
    word-break: normal; overflow-wrap: anywhere;
    background: var(--zd-bg); color: var(--zd-text);
    border: 1px solid var(--zd-border); border-radius: 10px; padding: 16px;
  }
  [data-theme="dark"] .zd-wrap {
    --zd-bg: #0f172a; --zd-surface: #1e293b; --zd-border: #334155;
    --zd-text: #f1f5f9; --zd-muted: #94a3b8; --zd-accent: #38bdf8; --zd-accent-soft: #0c4a6e;
    --zd-warn: #fbbf24;
  }
  .zd-head { display: flex; align-items: center; gap: 8px; margin-bottom: 12px; flex-wrap: wrap; }
  .zd-tag {
    font-size: 11px; font-weight: 700; padding: 2px 8px; border-radius: 999px;
    background: var(--zd-accent-soft); color: var(--zd-accent);
  }
  .zd-title { font-size: 14px; font-weight: 700; }
  .zd-grid-3 { display: grid; grid-template-columns: repeat(3, 1fr); gap: 10px; }
  .zd-card { background: var(--zd-surface); border: 1.5px solid var(--zd-border); border-radius: 8px; padding: 12px; }
  .zd-do { border-color: var(--zd-accent); }
  .zd-warn { border-color: var(--zd-warn); }
  .zd-card-h { font-size: 13px; font-weight: 700; margin-bottom: 6px; }
  .zd-card-b { font-size: 12px; color: var(--zd-muted); }
  .zd-card-h code, .zd-card-b code { font-size: 11px; padding: 1px 4px; border-radius: 4px; background: rgba(128,128,128,0.15); }
  @media (max-width: 640px) { .zd-grid-3 { grid-template-columns: 1fr; } }
</style>
```

```ts
constructor(ctx, env) {
  super(ctx, env)
  const exists = ctx.storage.sql
    .exec("SELECT 1 FROM sqlite_master WHERE type='table' AND name='state'")
    .toArray().length
  if (exists) this.#initSchema()
}

async search(query) {
  if (!this.#schemaReady) return { items: [], nextCursor: null }
  // ...
}
```

あわせて、同期を受け取ったオブジェクトが自身の名前と記録しているテナント ID を照合し、一致しなければ `deleteAll()` で中身を削除するようにしています。命名規則を変更したあとに古いオブジェクトが残っていても、次の同期タイミングで自動的に掃除されます。

## 4. 同期は変更行のみを読み込み、全件照合は12時間に1回にする

運用コストで最も大きかったのは、D1 の全データとインデックスを照合する処理を頻繁に回していたことでした。この構成のままでは月 $27〜93 かかる見積もりになっていましたが、変更された行だけを読み込む形に改めたことで、1時間あたりの負荷は次のように減りました。

| 項目 | 改善前 | 改善後 |
|---|---|---|
| Durable Objects の呼び出し | 2.1万回 | 約630回 |
| SQLite の書き込み | 16万行 | 約1,500行 |
| D1 の読み込み | 7,800万行 | 50万〜200万行 |

ファイルが更新されると、アプリは Durable Object の `sync()` を呼び出します。`sync()` は通知を受け取った回数をカウントして alarm を設定するだけで、具体的な変更内容は渡しません。実際の同期は alarm の中で動き、前回読み込んだ時刻以降に更新された行（`updated_at` が新しい行）だけを D1 から取得します。

```ts
const recent = await inventory(db, workspace)
  .where((eb) =>
    eb.or([
      eb('s.updated_at', '>', time),
      eb.and([eb('s.updated_at', '=', time), eb('s.id', '>', id)]),
    ]),
  )
  .orderBy('s.updated_at', 'asc')
  .orderBy('s.id', 'asc')
  .limit(50)
  .execute()
```

次に読み始める時刻は、今回読み始めた時刻の5分前に設定しています。書き込み中のトランザクションが、古い `updated_at` のまま後から確定するケースを取りこぼさないためです。

また、読み込みを始めるタイミングで、それまでに溜まっていた通知回数を0にリセットします。読み込み中に新しい通知が届いていた場合は、処理を終えたあとにもう一度差分を読み込みます。

```html-embed
<div class="sync-diagram">
  <div class="sync-title">差分同期の流れ（1時間あたり 7,800万行 → 50万〜200万行）</div>
  <div class="sync-flow">
    <div class="node app">
      <div class="node-label">1. ファイル更新</div>
      <div class="node-sub">アプリが <code>sync()</code> を呼ぶ</div>
    </div>
    <div class="arrow">
      <span class="pulse-dot"></span>
      <span class="arrow-text">通知回数 +1</span>
    </div>
    <div class="node do">
      <div class="node-label">2. Durable Object</div>
      <div class="node-sub">alarm をセットして即返却</div>
    </div>
    <div class="arrow">
      <span class="pulse-dot delay"></span>
      <span class="arrow-text">alarm 発火</span>
    </div>
    <div class="node d1">
      <div class="node-label">3. D1 から差分取得</div>
      <div class="node-sub"><code>updated_at</code> 以降を 50件ずつ</div>
    </div>
  </div>
  <div class="sync-notes">
    <div class="note-item">
      <span class="badge">取りこぼし防止</span>
      次回は「読み始め時刻の <strong>5分前</strong>」から読む
    </div>
    <div class="note-item">
      <span class="badge">保険</span>
      <strong>12時間ごと</strong>に Cron 経由で全件照合（50件ずつ）
    </div>
  </div>
</div>
<style>
  .sync-diagram {
    --bg: #f8fafc;
    --card: #ffffff;
    --border: #cbd5e1;
    --text: #0f172a;
    --muted: #475569;
    --accent: #0284c7;
    --accent-bg: #e0f2fe;
    --badge-bg: #fef3c7;
    --badge-text: #92400e;
    font-family: system-ui, -apple-system, sans-serif;
    background: var(--bg);
    color: var(--text);
    padding: 16px;
    border-radius: 10px;
    border: 1px solid var(--border);
  }
  [data-theme="dark"] .sync-diagram {
    --bg: #0f172a;
    --card: #1e293b;
    --border: #334155;
    --text: #f1f5f9;
    --muted: #94a3b8;
    --accent: #38bdf8;
    --accent-bg: #0c4a6e;
    --badge-bg: #451a03;
    --badge-text: #fcd34d;
  }
  .sync-title {
    font-size: 13px;
    font-weight: 700;
    margin-bottom: 14px;
    color: var(--text);
  }
  .sync-flow {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }
  .node {
    flex: 1;
    background: var(--card);
    border: 1.5px solid var(--border);
    border-radius: 8px;
    padding: 10px 12px;
    text-align: center;
  }
  .node.do {
    border-color: var(--accent);
    background: var(--accent-bg);
  }
  .node-label {
    font-size: 13px;
    font-weight: 700;
    margin-bottom: 4px;
  }
  .node-sub {
    font-size: 12px;
    color: var(--muted);
    line-height: 1.4;
  }
  .node-sub code {
    font-size: 11px;
    padding: 1px 4px;
    border-radius: 4px;
    background: rgba(128, 128, 128, 0.15);
  }
  .arrow {
    position: relative;
    display: flex;
    flex-direction: column;
    align-items: center;
    min-width: 72px;
  }
  .arrow::before {
    content: "";
    display: block;
    width: 100%;
    height: 2px;
    background: var(--border);
    margin-bottom: 4px;
  }
  .pulse-dot {
    position: absolute;
    top: -3px;
    left: 0;
    width: 8px;
    height: 8px;
    border-radius: 50%;
    background: var(--accent);
    animation: move-right 2s linear infinite;
  }
  .pulse-dot.delay {
    animation-delay: 1s;
  }
  @keyframes move-right {
    0% { left: 0; opacity: 0; }
    15% { opacity: 1; }
    85% { opacity: 1; }
    100% { left: calc(100% - 8px); opacity: 0; }
  }
  .arrow-text {
    font-size: 11px;
    color: var(--muted);
    white-space: nowrap;
  }
  .sync-notes {
    margin-top: 14px;
    display: flex;
    gap: 10px;
    flex-wrap: wrap;
  }
  .note-item {
    flex: 1;
    min-width: 220px;
    font-size: 12px;
    background: var(--card);
    border: 1px solid var(--border);
    border-radius: 6px;
    padding: 8px 10px;
    display: flex;
    align-items: center;
    gap: 8px;
  }
  .badge {
    font-size: 11px;
    font-weight: 700;
    padding: 2px 6px;
    border-radius: 4px;
    background: var(--badge-bg);
    color: var(--badge-text);
    white-space: nowrap;
  }
  @media (max-width: 640px) {
    .sync-flow {
      flex-direction: column;
      align-items: stretch;
    }
    .arrow {
      min-width: auto;
      padding: 6px 0;
    }
    .arrow::before {
      width: 2px;
      height: 18px;
      margin-bottom: 2px;
    }
    .pulse-dot {
      display: none;
    }
  }
  @media (prefers-reduced-motion: reduce) {
    .pulse-dot {
      animation: none;
      display: none;
    }
  }
</style>
```

通知を取りこぼした変更については、12時間に1回の全件照合で拾い上げます。この全件照合も50件ずつ alarm で進める仕組みです。5分ごとの Cron は、前回の送信から時間が経過したテナントから順に50件へ `sync()` を送るだけにしています。

## 5. 権限情報はインデックスに入れず、候補を D1 で判定する

ファイルを閲覧できるかどうかは、所有者、個別の共有設定、プロジェクトのメンバー、社内全員への公開、リンクの有効期限などで決まります。これらの条件をインデックス側に複製すると、同期が遅れているあいだに本来見えてはいけないファイルが検索結果に出てしまいます。そこでインデックスには権限情報を持たせず、権限の判定はすべて D1 に任せることにしました。

PoC で10万件のテナントを使い、3つの方式を比較した結果がこちらです。

| 方式 | p95 | 1回の D1 読み込み |
|---|---|---|
| D1 が閲覧可能なファイルの ID をすべて取得してインデックスに渡す | 568ms | 24万行 |
| インデックスが関連性の高い順に候補を2,000件出し、D1 が候補だけ判定する | 171ms | 1.2万行 |
| D1 が所有者・共有・参加プロジェクトから ID を取得してインデックスに渡す | 117ms | 1,228行 |

実際の運用では、テナントのメンバーには2つ目の方式、他社から共有を受けた外部ユーザーには3つ目の方式を使っています。メンバーは社内全体に公開されたファイルも閲覧できるため、2つ目の方式なら候補がほぼ1回で埋まります。一方、外部ユーザーは閲覧できる範囲が狭いため、先に D1 で ID を絞り込んでから渡したほうが高速です。

```html-embed
<div class="auth-tabs">
  <input type="radio" name="auth-mode" id="tab-member" checked>
  <input type="radio" name="auth-mode" id="tab-guest">
  <div class="tab-bar">
    <label for="tab-member" class="tab-btn">テナントメンバー向け（171ms）</label>
    <label for="tab-guest" class="tab-btn">外部共有ユーザー向け（117ms）</label>
  </div>

  <div class="panel panel-member">
    <div class="panel-desc">社内公開ファイルが多く対象が広いため、先に DO で候補を絞ってから D1 で権限判定します。</div>
    <div class="steps">
      <div class="step do">
        <div class="step-num">Step 1</div>
        <div class="step-title">Durable Object（FTS5）</div>
        <div class="step-body">関連性の高い順に候補 <strong>2,000件</strong> の ID を取得</div>
      </div>
      <div class="step-arrow">→</div>
      <div class="step d1">
        <div class="step-num">Step 2</div>
        <div class="step-title">D1（権限の正本）</div>
        <div class="step-body"><code>json_each(?1)</code> と <code>+workspace_id</code> で閲覧可否を判定（約1.2万行読込）</div>
      </div>
    </div>
  </div>

  <div class="panel panel-guest">
    <div class="panel-desc">閲覧できるファイルが限られているため、先に D1 で許可 ID を引いてから DO で検索します。</div>
    <div class="steps">
      <div class="step d1">
        <div class="step-num">Step 1</div>
        <div class="step-title">D1（権限の正本）</div>
        <div class="step-body">所有者・個別共有・参加プロジェクトから許可 ID を取得（1,228行読込）</div>
      </div>
      <div class="step-arrow">→</div>
      <div class="step do">
        <div class="step-num">Step 2</div>
        <div class="step-title">Durable Object（FTS5）</div>
        <div class="step-body">受け取った ID の範囲内だけで全文検索を実行</div>
      </div>
    </div>
  </div>
</div>
<style>
  .auth-tabs {
    --bg: #f8fafc;
    --card: #ffffff;
    --border: #cbd5e1;
    --text: #0f172a;
    --muted: #475569;
    --do-border: #0284c7;
    --do-bg: #e0f2fe;
    --d1-border: #f59e0b;
    --d1-bg: #fef3c7;
    font-family: system-ui, -apple-system, sans-serif;
    background: var(--bg);
    color: var(--text);
    padding: 16px;
    border-radius: 10px;
    border: 1px solid var(--border);
  }
  [data-theme="dark"] .auth-tabs {
    --bg: #0f172a;
    --card: #1e293b;
    --border: #334155;
    --text: #f1f5f9;
    --muted: #94a3b8;
    --do-border: #38bdf8;
    --do-bg: #0c4a6e;
    --d1-border: #fbbf24;
    --d1-bg: #451a03;
  }
  .auth-tabs input[type="radio"] {
    display: none;
  }
  .tab-bar {
    display: flex;
    gap: 8px;
    margin-bottom: 14px;
  }
  .tab-btn {
    flex: 1;
    text-align: center;
    padding: 8px 10px;
    font-size: 13px;
    font-weight: 700;
    border-radius: 8px;
    border: 1.5px solid var(--border);
    background: var(--card);
    color: var(--muted);
    cursor: pointer;
    transition: all 0.15s ease;
  }
  #tab-member:checked ~ .tab-bar label[for="tab-member"],
  #tab-guest:checked ~ .tab-bar label[for="tab-guest"] {
    border-color: var(--do-border);
    background: var(--do-bg);
    color: var(--text);
  }
  .panel {
    display: none;
  }
  #tab-member:checked ~ .panel-member,
  #tab-guest:checked ~ .panel-guest {
    display: block;
  }
  .panel-desc {
    font-size: 12px;
    color: var(--muted);
    margin-bottom: 12px;
    line-height: 1.5;
  }
  .steps {
    display: flex;
    align-items: center;
    gap: 10px;
  }
  .step {
    flex: 1;
    background: var(--card);
    border: 1.5px solid var(--border);
    border-radius: 8px;
    padding: 12px;
  }
  .step.do {
    border-color: var(--do-border);
  }
  .step.d1 {
    border-color: var(--d1-border);
  }
  .step-num {
    font-size: 11px;
    font-weight: 700;
    color: var(--muted);
    margin-bottom: 2px;
  }
  .step-title {
    font-size: 13px;
    font-weight: 700;
    margin-bottom: 6px;
  }
  .step-body {
    font-size: 12px;
    line-height: 1.5;
  }
  .step-body code {
    font-size: 11px;
    padding: 1px 4px;
    border-radius: 4px;
    background: rgba(128, 128, 128, 0.15);
  }
  .step-arrow {
    font-size: 18px;
    font-weight: 700;
    color: var(--muted);
  }
  @media (max-width: 640px) {
    .tab-bar {
      flex-direction: column;
    }
    .steps {
      flex-direction: column;
      align-items: stretch;
    }
    .step-arrow {
      text-align: center;
      transform: rotate(90deg);
    }
  }
</style>
```

候補の判定では、ID の配列を1つの JSON 文字列にまとめて D1 に渡します。D1 は1クエリあたりのバインド引数が100個までに制限されているため、ID をそのまま引数に並べる方法は使えません。

```sql
SELECT s.id FROM shareables s
WHERE s.id IN (SELECT value FROM json_each(?1))
  AND +s.workspace_id = ?2   -- workspace_id の索引を使わせない
  AND (/* 所有者、共有、プロジェクト、社内全員への公開の条件 */)
```

`+` を付けないと、SQLite は `workspace_id` のインデックスを使ってテナントの全行をスキャンする実行計画を選んでしまい、2,000件の判定に13秒かかりました。`+` を付けて主キーで検索させることで、候補1件あたり約6行の読み込みで済むようになります。

## 6. 日本語は Intl.Segmenter で分割し、カタカナは1語にまとめる

インデックスに登録するテキストと検索クエリの両方に、同じ前処理をかけます。NFKC で正規化して小文字に変換したあと、`Intl.Segmenter` で単語に分割してスペースでつなぎます。また、3文字以上のカタカナについては末尾の長音記号を削り、「サーバー」と「サーバ」を同じ語として扱います。

```html-embed
<div class="zd-wrap">
  <div class="zd-head">
    <span class="zd-tag">日本語前処理</span>
    <span class="zd-title">検索クエリとインデックスの両方に同じ正規化を適用</span>
  </div>
  <div class="zd-pipe">
    <div class="zd-pbox">
      <div class="zd-pstep">入力</div>
      <div class="zd-pval">「サーバーのレビュー」</div>
    </div>
    <div class="zd-parr">→</div>
    <div class="zd-pbox">
      <div class="zd-pstep">NFKC・小文字化・長音除去</div>
      <div class="zd-pval">「サーバのレビュー」</div>
    </div>
    <div class="zd-parr">→</div>
    <div class="zd-pbox zd-pbox-accent">
      <div class="zd-pstep">2つの索引へ</div>
      <div class="zd-pval">
        <div>通常：<code>サーバ / の / レ / ビュ</code></div>
        <div>カタカナ1語：<code>サーバー</code> <code>レビュー</code>（前方一致）</div>
      </div>
    </div>
  </div>
</div>
<style>
  .zd-wrap {
    --zd-bg: #f8fafc; --zd-surface: #ffffff; --zd-border: #cbd5e1;
    --zd-text: #0f172a; --zd-muted: #475569; --zd-accent: #0284c7; --zd-accent-soft: #e0f2fe;
    font-family: "Hiragino Kaku Gothic ProN", "Hiragino Sans", "Yu Gothic UI", "Meiryo", system-ui, sans-serif;
    font-feature-settings: "palt"; line-height: 1.65; letter-spacing: 0.02em;
    word-break: normal; overflow-wrap: anywhere;
    background: var(--zd-bg); color: var(--zd-text);
    border: 1px solid var(--zd-border); border-radius: 10px; padding: 16px;
  }
  [data-theme="dark"] .zd-wrap {
    --zd-bg: #0f172a; --zd-surface: #1e293b; --zd-border: #334155;
    --zd-text: #f1f5f9; --zd-muted: #94a3b8; --zd-accent: #38bdf8; --zd-accent-soft: #0c4a6e;
  }
  .zd-head { display: flex; align-items: center; gap: 8px; margin-bottom: 12px; flex-wrap: wrap; }
  .zd-tag {
    font-size: 11px; font-weight: 700; padding: 2px 8px; border-radius: 999px;
    background: var(--zd-accent-soft); color: var(--zd-accent);
  }
  .zd-title { font-size: 14px; font-weight: 700; }
  .zd-pipe { display: flex; align-items: center; gap: 8px; }
  .zd-pbox {
    flex: 1; background: var(--zd-surface); border: 1.5px solid var(--zd-border);
    border-radius: 8px; padding: 10px 12px;
  }
  .zd-pbox-accent { border-color: var(--zd-accent); background: var(--zd-accent-soft); }
  .zd-pstep { font-size: 11px; font-weight: 700; color: var(--zd-muted); margin-bottom: 4px; }
  .zd-pval { font-size: 12px; }
  .zd-pval code { font-size: 11px; padding: 1px 4px; border-radius: 4px; background: rgba(128,128,128,0.18); }
  .zd-parr { font-size: 16px; font-weight: 700; color: var(--zd-muted); }
  @media (max-width: 640px) {
    .zd-pipe { flex-direction: column; align-items: stretch; }
    .zd-parr { text-align: center; transform: rotate(90deg); }
  }
</style>
```

なお、`Intl.Segmenter` で日本語を分かち書きし、Cloudflare D1 の SQLite FTS5 で検索する基本的な実装については、以前の記事でコード付きで紹介しています。[個人開発に無料の日本語全文検索を - Cloudflare D1 と Web Component でつくる検索 API](https://zenn.dev/coji/articles/cloudflare-d1-fts5-japanese-search-api)

今回はさらに、検索順位の調整と取りこぼし対策として次の3点を加えました。

- 連続するカタカナは1語にまとめて別の FTS5 テーブルに入れ、前方一致で検索する（`Intl.Segmenter` が「レビュ」を「レ|ビュ」のように分割してしまうため）
- タイトルに入力文字列をそのまま含む文書を優先して上位に表示する
- 検索語が隣接して出現する文書を、離れて出現する文書より上位に表示する

本文から切り出した文字列を検索クエリとし、その文字列を含む文書が正しくヒットするかを計測した結果が次の表です。

| クエリの種類 | 取りこぼし率（対策前 → 対策後） |
|---|---|
| 長音を省いた形（「サーバー」を「サーバ」で検索） | 40% → 8〜11% |
| タイトルの単語の途中まで | 26〜28% → 11〜13% |
| 本文のカタカナ | 10〜11% → 4〜7% |
| 本文の英数字 | 3% → 0.2% |

本文のテキストは HTMLRewriter で抽出しており、`alt` などの属性値や、script タグ内の日本語文字列もインデックスに含めています。JavaScript で描画するページでは、表示される文字列が script タグの中に書かれていることがあるためです。

## 限界

1つの Durable Object はリクエストを1つずつ順番に処理するため、PoC の10万件規模では検索スループットが毎秒16〜17件で頭打ちになりました。人間の手動検索であれば十分な余裕がありますが、AI エージェントが連続して検索を叩くような使い方では上限に近づく可能性があります。

また、単語の統計情報をテナント全体で計算しているため、閲覧権限のない文書に含まれる単語の有無が、検索結果の並び順からわずかに推測できてしまいます[^stress]。これについては、権限のない文書のタイトルや抜粋を表示しないこと、および前方一致などの検索演算子をユーザーに開放しないことで、実運用上は許容することにしました。

[^stress]: Liang Wang ほか「Side-Channel Attacks on Shared Search Indexes」(IEEE S&P 2017)。共有インデックスの TF-IDF スコアから、他人の文書に特定の語が含まれるかを推測する手法について論じている。https://ieeexplore.ieee.org/document/7958604/

## AI に渡すチェックリスト

同じ仕組みを実装させるときは、以下のチェックリストをそのまま渡せます。

```text
- 索引はテナントごとに1つの Durable Object(SQLite FTS5)に置く。D1 に FTS5 は置かない
- オブジェクト名に地域を入れ(`${tenant}:0@apac-ne`)、getByName に同じ locationHint を渡す。置き場所を D1 に記録しない
- テーブルは最初の sync で作る。コンストラクタは sqlite_master を見るだけ、未同期なら検索は空を返す
- sync() は通知回数を数えて alarm を仕掛けるだけ。差分は alarm で updated_at, id の順に50件ずつ読む
- 次の読み始めは読み始めた時刻の5分前。読み始めに通知回数を0に戻す
- 全件の照合は12時間に1回、50件ずつ。Cron は最後に送ってから時間のたったテナントから順に sync を送るだけ
- 名前と記録したテナントが合わないオブジェクトは deleteAll() で消す
- 権限は索引に入れない。メンバーは索引の候補2,000件を D1 で判定、外部の人は D1 で id を出して索引に渡す
- 候補の判定は json_each で id を1引数にし、テナント列に + を付けて主キーで探させる
- 日本語は NFKC と小文字化のあと Intl.Segmenter で分ける。3文字以上のカタカナの末尾の長音は削る
- カタカナの連続は1語にして別テーブルで前方一致。入力をそのまま含むタイトルと、語が並んで出る文書を上に並べる
```

実装は公開リポジトリにあります。

https://github.com/artifactshare/artifactshare/blob/main/apps/web/workers/workspace-search-index.ts
