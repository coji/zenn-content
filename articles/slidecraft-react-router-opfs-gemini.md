---
title: "SlideCraft - AI生成スライドを部分的に直せるツールを作りました"
emoji: "🎨"
type: "tech"
topics: ["react", "reactrouter", "gemini", "typescript", "opfs"]
published: true
---

## これはなに？

PDFスライドをアップロードして、直したいスライドだけをAIで再構成できるWebアプリ「SlideCraft」を作りました。React Router v7、Google Gemini API、Origin Private File System（OPFS）を組み合わせた、完全クライアントサイドで動作するアプリケーションです。

https://www.slidecraft.work

https://github.com/coji/slidecraft

## 作った背景

Nano Banana ProやGoogle Notebook LMでプレゼン資料を生成すると、20枚のスライドのうち17枚は良い感じなのに、3枚だけ微妙、みたいなことがあります。

ここで問題が起きます。プロンプトを調整して再生成すると、問題のない17枚まで変わってしまいます。かといってPowerPointで手作業するのは時間がかかります。「この3枚だけAIで直したい」という、よくある要望に応えるツールがなかったのです。

この記事では、SlideCraftを開発する中で面白かった技術的なポイントを紹介します。

## Origin Private File System（OPFS）によるデータ永続化

### なぜOPFSを選んだか

SlideCraftは完全にクライアントサイドで動作します。バックエンドを持たない設計にしたかったため、ユーザーのプロジェクトデータをどこに保存するかが課題でした。

選択肢としてはlocalStorage、IndexedDB、そしてOPFSがあります。localStorageは容量制限が厳しく、画像を扱うには不向きです。IndexedDBは十分な容量がありますが、Key-Value的なAPIでファイルシステムの概念を表現するには抽象化が必要になります。

OPFSを選んだ理由は、ファイルシステムそのものの概念をそのまま使えるからです。プロジェクトをディレクトリとして、スライド画像やメタデータをファイルとして扱えます。開発者にとって直感的ですし、将来的にFile System Access APIと連携する拡張も視野に入れやすい設計になります。

### 実装のポイント

```typescript
// storage.client.ts より抜粋
export async function writeFile(path: string, content: string | Blob): Promise<void> {
  const root = await navigator.storage.getDirectory()
  const parts = path.split('/').filter(Boolean)
  const fileName = parts.pop()
  if (!fileName) throw new Error('Invalid path')

  let current = root
  for (const part of parts) {
    current = await current.getDirectoryHandle(part, { create: true })
  }

  const fileHandle = await current.getFileHandle(fileName, { create: true })
  const writable = await fileHandle.createWritable()
  await writable.write(content)
  await writable.close()
}
```

パスを`/`で分割してディレクトリを再帰的に作成し、最後にファイルを書き込みます。`{ create: true }`オプションにより、存在しないディレクトリやファイルは自動的に作成されます。

バイナリ（Blob）とテキスト（JSON文字列）を同じインターフェースで扱えるのも便利なところです。メタデータはJSONで、スライド画像はBlobで保存しています。

OPFSはWeb Workerからも同期的にアクセスできる`createSyncAccessHandle()`も提供しているため、将来的に重い処理をWorkerに移すことも可能です。

## React Router v7のclientLoader/clientActionパターン

### サーバーなしのデータフェッチ

React Router v7では、従来の`loader`/`action`に加えて、`clientLoader`/`clientAction`が使えます。これはクライアントサイドでのみ実行されるデータフェッチ・ミューテーション関数です。

SlideCraftではバックエンドがないため、すべてのデータ操作をクライアントで行います。`clientLoader`でOPFSからプロジェクトデータを読み込み、`clientAction`でスライドの選択や画像の更新を行います。

```typescript
// routes/editor.tsx より
export const clientLoader = async ({ params }: Route.ClientLoaderArgs) => {
  const projectId = params.projectId
  const slideIndex = Number(params.slideIndex)

  const project = await loadProject(projectId)
  const slide = project.slides[slideIndex]
  const originalImagePath = slide.originalImagePath

  return { project, slide, slideIndex, originalImagePath }
}

export const clientAction = async ({ request, params }: Route.ClientActionArgs) => {
  const formData = await request.formData()
  const intent = formData.get('intent')

  switch (intent) {
    case 'selectCandidate':
      // 生成された候補画像を選択
      return await selectCandidateAction(params, formData)
    case 'resetToOriginal':
      // 元の画像にリセット
      return await resetToOriginalAction(params)
    // ...
  }
}
```

### なぜuseStateではなくclientActionか

スライドエディタでは「生成された候補画像を選択する」「元の画像にリセットする」といった操作があります。これらを`useState`で管理すると、状態の更新ロジックがコンポーネント内に散らばりがちです。

`clientAction`を使うと、状態の変更が明示的なフォームのsubmitとして表現されます。React Router v7の`useFetcher`と組み合わせることで、非同期処理中のpending状態も自動的に追跡できます。

```typescript
const fetcher = useFetcher()
const isUpdating = fetcher.state !== 'idle'

// フォーム送信
<fetcher.Form method="post">
  <input type="hidden" name="intent" value="selectCandidate" />
  <button disabled={isUpdating}>選択</button>
</fetcher.Form>
```

読み取り（clientLoader）と書き込み（clientAction）が明確に分離され、コードの見通しがよくなりました。

## Gemini APIを使った並列画像生成とコスト計算

### 複数バリエーションの並列生成

スライドの再構成では、1枚のスライドに対して複数のバリエーションを生成します。ユーザーが「もっとシンプルに」とプロンプトを入力すると、3枚の候補画像を同時に生成します。

```typescript
// gemini-api.client.ts より
export async function generateSlideImages(
  apiKey: string,
  imageBlob: Blob,
  prompt: string,
  count: number = 3
): Promise<GenerationResult[]> {
  const base64Image = await blobToBase64(imageBlob)

  const promises = Array.from({ length: count }, () =>
    callGeminiAPI(apiKey, base64Image, prompt)
  )

  const results = await Promise.all(promises)
  return results
}
```

`Promise.all`で並列実行することで、3枚の画像生成を待つ時間が単純に3倍になることを避けています。Gemini APIのレート制限に引っかからない範囲で、できるだけ高速にレスポンスを返すことを意識しました。

### コストの可視化

AI画像生成はコストがかかります。ユーザーが安心して使えるよう、生成前にコストの見積もりを表示しています。

```typescript
// cost-calculator.ts より
const GEMINI_PRICING = {
  image_2k: 0.134,  // 2K画像1枚あたりのUSドル
  image_4k: 0.24,
}

export function calculateCost(slideCount: number, resolution: '2k' | '4k'): number {
  const pricePerImage = resolution === '2k'
    ? GEMINI_PRICING.image_2k
    : GEMINI_PRICING.image_4k
  return slideCount * pricePerImage
}
```

さらに、exchangerate-apiから為替レートを取得して日本円での表示も行っています。レートは24時間キャッシュして、APIコールを最小限に抑えています。

小さな機能ですが、AIツールでは見落とされがちなコスト透明性を意識しました。「このボタンを押したらいくらかかるのか」を常に把握できる状態にしておくと、安心して使ってもらえるのではないかと考えています。

## おわりに

SlideCraftを開発する中で、ブラウザの進化を実感しました。OPFSによるファイルシステム抽象化、React Router v7のクライアントサイドデータフェッチ、そしてGemini APIへの直接アクセス。これらを組み合わせることで、バックエンドなしでもリッチなAIアプリケーションが作れます。

ぜひ試してみてください。質問やフィードバックがあれば、GitHubのIssueでお待ちしています。
