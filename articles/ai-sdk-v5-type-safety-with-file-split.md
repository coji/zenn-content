---
title: "AI SDK v5 でコンポーネント分割しても型安全を保つ方法"
emoji: "🔐"
type: "tech"
topics: ["typescript", "vercelaisdk", "react"]
published: true
---

## これはなに?

AI SDK v5 でチャットアプリを作る際、ツール呼び出しのあるメッセージをコンポーネント分割すると型の扱いに悩むことがあります。[公式のチャットボットガイド](https://ai-sdk.dev/docs/ai-sdk-ui/chatbot-tool-usage)では `useChat` でのツール使用方法が紹介されていますが、1ファイル内での実装例のため、実際にコンポーネントをファイル分割したときにどう型定義を共有するかまでは示されていません。

この記事では、実際にチャットアプリを作ってみて見つけた、コンポーネント分割しても型安全性を保つための実装パターンを紹介します。

## v4 での困りごと

AI SDK v4 を使っていたとき、ツールの入出力型が `unknown` として扱われることが多く、各コンポーネントで型アサーションを書く必要がありました。

```typescript
// v4 でよく書いていたコード
const output = part.output as { title: string; content: string } | undefined
```

これだと、ツールの定義を変更しても各コンポーネントの型アサーションは自動で更新されないので、実行時エラーが起きるリスクがありました。保守性の面で課題がある実装です。

## v5 で型定義を中央管理する

v5 では [`InferUITools`](https://ai-sdk.dev/docs/reference/ai-sdk-ui/infer-ui-tools) を使うことで、ツールセットから UI 用の型を自動推論できるようになりました。ただ、コンポーネントをファイル分割する際は、型定義ファイルを1箇所にまとめて、すべてのコンポーネントがそこから型をインポートする設計にしておくと管理しやすくなります。

## 型定義ファイルを1箇所にまとめる

まず、すべてのコンポーネントから参照する型定義ファイルを作ります。

```typescript
// types/chat-tools.ts
import type { openai } from '@ai-sdk/openai'
import type { InferUITools, UIDataTypes, UIMessage } from 'ai'
import type { documentTool } from '../api/tools'

// アプリで使うツールをまとめた型
export type ChatToolSet = {
  documentTool: typeof documentTool
  web_search: ReturnType<typeof openai.tools.webSearch>
}

// InferUITools でツールセットから UI 用の型を推論
export type ChatUITools = InferUITools<ChatToolSet>

// チャットメッセージの型
export type ChatUIMessage = UIMessage<never, UIDataTypes, ChatUITools>

// ツール呼び出し部分の型
export type ChatToolPart = Extract<
  NonNullable<ChatUIMessage['parts']>[number],
  { type: `tool-${string}` }
>
```

この型定義ファイルがアプリ全体の型の「真実の情報源」になります。ツールの定義を変更すると、ここで推論される型が自動的に更新されて、それを使うすべてのコンポーネントに変更が伝わります。

## ツールごとのコンポーネントで型を使う

各ツール用のコンポーネントでは、中央の型定義から `Extract` を使って特定のツールの型だけを取り出します。

```typescript
// components/ToolWebSearchResult.tsx
import type { ChatToolPart } from '../types/chat-tools'

type WebSearchToolPart = Extract<ChatToolPart, { type: 'tool-web_search' }>

interface Props {
  part: WebSearchToolPart
}

export const ToolWebSearchResult = ({ part }: Props) => {
  // part.output は web_search の出力型として自動で推論される
  const output = part.output

  if (!output?.sources) return null

  return (
    <div>
      {output.sources.map((source) => (
        <a key={source.url} href={source.url}>
          {source.url}
        </a>
      ))}
    </div>
  )
}
```

ポイントは、各コンポーネントで型アサーションを書く必要がないことです。`Extract` で型を絞り込むだけで、`part.output` の型が自動で推論されます。

## メッセージを表示するコンポーネント

メッセージを表示するコンポーネントでは、`part.type` で分岐して各ツール用のコンポーネントを呼び出します。

```typescript
// components/MessageList.tsx
import { match } from 'ts-pattern'
import type { ChatUIMessage, ChatToolPart } from '../types/chat-tools'
import { ToolWebSearchResult } from './ToolWebSearchResult'
import { ToolDocumentResult } from './ToolDocumentResult'

const ToolCallRenderer = ({ part }: { part: ChatToolPart }) => {
  return match(part)
    .with({ type: 'tool-web_search', state: 'output-available' }, (part) => {
      return <ToolWebSearchResult part={part} />
    })
    .with({ type: 'tool-documentTool', state: 'output-available' }, (part) => {
      return <ToolDocumentResult part={part} />
    })
    .otherwise(() => null)
}
```

ここでは [ts-pattern](https://github.com/gvergnaud/ts-pattern) の `match` を使ってパターンマッチングしています。`part.type` と `state` の組み合わせで条件分岐することで、TypeScript が各分岐で型を自動的に絞り込んでくれます。各 `.with()` ブロック内では `part` の型がそのツール固有の型になっているので、型安全にプロパティにアクセスできます。

通常の `if` 文や `switch` 文でも同じことはできますが、ts-pattern を使うと複数の条件を組み合わせた分岐が書きやすく、型の絞り込みも確実に効くのでおすすめです。

## ファイル分割するときに気をつけること

この型安全な設計を維持するには、いくつかポイントがあります。

中央の型定義ファイルを必ず作って、すべてのコンポーネントが同じ型定義を参照するようにします。これで一貫性が保たれます。各コンポーネントで型を再定義するのではなく、`Extract` を使って中央の型から必要な部分だけを取り出します。型アサーションも避けて、`Extract` と TypeScript の型推論に任せることで、ツール定義の変更が自動的に伝わるようになります。

公式ドキュメントには `InferUITools` の使い方は書いてありますが、実際のアプリでは「どこに型定義を置くか」「各コンポーネントでどう参照するか」という設計が大事になってきます。

## ストリーミング中の型の扱い方

ストリーミング中は、ツールの入力が部分的にしか利用できないことがあります。そういうときは型ガードを使います。

```typescript
// components/ToolWebSearchStreaming.tsx
import type { ChatToolPart } from '../types/chat-tools'

type WebSearchToolPart = Extract<ChatToolPart, { type: 'tool-web_search' }>

interface Props {
  input: WebSearchToolPart['input']
}

export const ToolWebSearchStreaming = ({ input }: Props) => {
  const query =
    input &&
    typeof input === 'object' &&
    'query' in input &&
    typeof input.query === 'string'
      ? input.query
      : undefined

  return <div>{query && `検索中: ${query}`}</div>
}
```

ストリーミング中は `input` が空オブジェクトのこともあるので、型ガードで安全にアクセスしておくと安心です。

## カスタムフックでも同じ型を使う

カスタムフックでも同じメッセージ型を使うことで、アプリ全体で型の一貫性を保てます。

```typescript
// hooks/useChatMessages.ts
import { useChat } from '@ai-sdk/react'
import type { ChatUIMessage } from '../types/chat-tools'

export const useChatMessages = ({
  chatId,
  initialMessages,
}: {
  chatId: string
  initialMessages: ChatUIMessage[]
}) => {
  const chat = useChat<ChatUIMessage>({
    id: chatId,
    messages: initialMessages,
    // ...
  })

  return { chat }
}
```

`useChat<ChatUIMessage>` と型を指定することで、`chat.messages` の型も自動的に `ChatUIMessage[]` になって、コンポーネント全体で型が一致します。

## v4 から v5 での変更点

v4 から v5 に移行するときに、いくつか構造が変わっているので参考までにまとめておきます。

メッセージパートの構造が変わりました。v4 では `part.toolInvocation.toolName`、`part.toolInvocation.state`、`part.toolInvocation.args` のようにネストしていましたが、v5 では `part.type`、`part.state`、`part.input`、`part.output` とフラットになっています。

状態名も変わっていて、v4 の `'partial-call'` と `'result'` が、v5 では `'input-streaming'`、`'input-available'`、`'output-available'` になりました。

その他にも、v4 で `part.source.url` と `part.source.title` だったものが、v5 では `part.type === 'source-url'` で判定して `part.url` と `part.title` でアクセスするようになっています。推論パートも v4 の `part.reasoning` から、v5 では `part.type === 'reasoning'` で判定して `part.text` でアクセスする形に変わりました。

## まとめ

AI SDK v5 の型システムを使うことで、ツール定義から UI コンポーネントまで一貫した型安全性を保てるようになります。型アサーションも最小限で済むので、保守性の高いコードになります。TypeScript の型推論のおかげで、開発中の補完やエラー検出も効きやすくなります。

鍵になるのは、中央に配置した型定義ファイルと `InferUITools` によるツールセットからの型推論、そして `Extract` による各ツール固有の型の抽出です。この組み合わせで、コンポーネントをファイル分割しても型安全性を保ちながら、ツールの定義を変更したときに、それを使うすべてのコンポーネントの型が自動で更新されるようになります。
