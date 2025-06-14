---
title: "Vercel AI SDK用のClaude Code providerを作ってみた"
emoji: "🛠️"
type: "tech"
topics: ["claude", "ai", "vercel", "typescript", "provider"]
published: true
---

こんにちは、cojiです。

Vercel AI SDKのcommunity providerとして、Claude Code用のproviderを作ってみました。

https://github.com/coji/claude-code-ai-provider

せっかくClaude MAXに入っているので、高性能なモデルをフルに活用して自分用のLLMアプリを作りたくて。

最近普段遣いにしてる [Claude Code](https://claude.ai/code)、「これをVercel AI SDKで使えたら便利そう」と思ったのがきっかけです。

## 先に結論

Claude Code AI providerを作ることで、いつものVercel AI SDKの書き方そのままで、ローカルのClaude Codeが使えるようになりました。

実際に使ってみて感じたメリット：

- **Claude MAXを最大活用**: 高性能モデルを思う存分使える
- **開発効率**: いつものAI SDK開発体験そのまま

## なぜ作ったのか

**Claude MAXの恩恵を最大化したい**
せっかく月額サブスクに入っているので、制限を気にせず高性能モデルを使い倒したい

**自分用アプリを気軽に作りたい**
プロトタイプや個人用ツールを、コストを気にせず思い切り作ってみたい

**いつもの開発体験を維持したい**
新しいAPIを覚えるのではなく、慣れ親しんだVercel AI SDKをそのまま使いたい

こんな要望を満たすために、Claude Code用のproviderを作ってみました。

## Claude Code AI provider

[Claude Code](https://claude.ai/code)をVercel AI SDK providerとして実装したものです。これで、いつものAI SDK開発フローをそのまま活用できます。

実は、このprovider自体もClaude Codeに書いてもらいました。qwen-ai-provider、Vercel AI SDK、Claude Code SDKのドキュメントを参考資料として渡して、「こんな感じのproviderを作って」とお願いしたら、4時間程度でちゃんと動くものができあがりました。

Claude Codeを使ってClaude Code用のproviderを作る、というメタな感じが面白かったです。

### 使い方

インストールして使うだけです：

```bash
# インストール
npm install claude-code-ai-provider
```

```typescript
// 使用例
import { claudeCode } from 'claude-code-ai-provider'
import { generateText } from 'ai'

const result = await generateText({
  model: claudeCode('sonnet'),
  prompt: 'TypeScriptでTODOアプリを作って'
})
```

### できること

現在実装済みの機能：

- **テキスト生成**: generateTextとstreamTextに対応
- **オブジェクト生成**: generateObjectとstreamObjectに対応
- **ストリーミング対応**: リアルタイムでのテキスト・オブジェクト生成

今後検討中：

- **ツール呼び出し**: Claude Codeの対応状況を調査して実装検討
- **MCP統合**: Model Context Protocolサポートの可能性を調査

## 実際に使ってみて良かった点

### 1. 開発体験がそのまま

これが一番のポイントですね。いつものVercel AI SDKの書き方がそのまま使えます：

```typescript
// 普段のAI SDKと全く同じ書き方
const { text } = await generateText({
  model: claudeCode('sonnet'),
  prompt: 'Hello!'
})

// generateObjectも同様に使える
const { object } = await generateObject({
  model: claudeCode('sonnet'),
  schema: z.object({
    title: z.string(),
    content: z.string(),
  }),
  prompt: 'ブログ記事のタイトルと内容を生成して'
})
```

新しいAPIを覚える必要がないので、既存プロジェクトにもすぐ導入できました。

### 2. プロセス管理が簡単

Claude Code SDKを使ったので、プロセス管理周りは簡単に実装できました。SDKが面倒な部分を隠してくれるので、provider側ではシンプルに書けて助かりました。

## 実際の使用例

### Next.jsでのチャットボット

```typescript
import { claudeCode } from 'claude-code-ai-provider'
import { streamText } from 'ai'

export async function POST(request: Request) {
  const { messages } = await request.json()
  
  const result = streamText({
    model: claudeCode('sonnet'),
    messages,
  })
  
  return result.toDataStreamResponse()
}
```

API Routeでストリーミングレスポンスも問題なく動作します。

## 今後やりたいこと

### ツール呼び出し対応の検討

Claude Codeがツール呼び出し機能をサポートしているかまだ調査中ですが、もしサポートしていれば、将来的にはこんな使い方もできるかもしれません：

```typescript
const result = await generateText({
  model: claudeCode('sonnet'),
  prompt: 'ファイルの内容を確認して',
  tools: {
    readFile: tool({
      description: 'ファイルを読み取る',
      parameters: z.object({
        path: z.string()
      }),
      execute: async ({ path }) => {
        return await fs.readFile(path, 'utf-8')
      }
    })
  }
})
```

まずはClaude Codeの機能を詳しく調査してみる必要がありますね。

### その他の機能拡張

- エラーハンドリングの改善
- 設定オプションの追加
- パフォーマンス最適化

まだまだ改善の余地があるので、継続的にアップデートしていく予定です。

## まとめ

Vercel AI SDK用のClaude Code providerを作ってみて、**Claude MAXユーザーにとって理想的な開発環境**が整ったと感じています。

特にこんな場面で重宝しそうです：

**個人プロジェクト**
Claude MAXの高性能モデルを制限なく使って、思い切りアイデアを形にできる

**プロトタイピング**
コストを気にせず何度でも試行錯誤して、最適解を見つけられる

**学習・実験**
新しいAI技術やパターンを、制約なく思う存分試せる

community providerとして作ったので、Vercel AI SDKエコシステムの一部として自然に使えるのも気に入っています。

Claude MAXユーザーで、ローカルでLLMアプリを作ってみたい方は、ぜひ試してみてください！

## コントリビュート歓迎

まだできたばかりなので、機能が足りない部分や変な挙動があるかもしれません。バグ報告や機能追加のPR、大歓迎です！

特に以下の改善アイデアがあれば、ぜひコントリビュートしてください：

- エラーハンドリングの改善
- パフォーマンス最適化
- 新機能の追加（ツール呼び出し対応など）
- ドキュメントの充実
- テストケースの追加

みんなで育てていけたらいいなと思っています。

気軽にGitHubのIssueやPRを送ってもらえるほか、[@techtalkjp](https://x.com/techtalkjp) まで連絡してもらってもOKです！

---

## 参考リンク

- [Claude Code AI Provider](https://github.com/coji/claude-code-ai-provider)
- [Claude Code](https://claude.ai/code)
- [Vercel AI SDK](https://sdk.vercel.ai/)
- [作者Twitter](https://x.com/techtalkjp)
