---
title: "Vercel AI SDK useObject: リッチなストリーミングUIを簡単に"
emoji: "🔧"
type: "tech"
topics: ["remix", "nextjs", "typescript", "LLM"]
published: true
---

# Vercel AI SDK useObject: 効率的なLLMストリーミング処理の実現

Vercel AI SDKの`useObject`フックを使用した経験から、その有用性と開発効率の向上について共有します。

## useObjectの概要

[`useObject`](https://sdk.vercel.ai/docs/reference/ai-sdk-ui/use-object)は、[Vercel AI SDK](https://sdk.vercel.ai/docs/introduction)が提供するReactフックです。このフックを使用することで、AIからのストリーミングレスポンスをJSONオブジェクトとして簡単に扱うことができ、それを使ってリッチなストリーミングUIを簡単に作ることができます。

たとえば、以下のような UIを 10分で作ることができます。

![demo](/images/vercel-ai-sdk-use-object-is-nice/use-object-demo.gif)

## 主な特徴

1. **使用の簡便性**: APIエンドポイントとスキーマの指定だけで、AIレスポンスの処理が可能です。
2. **型安全性**: Zodスキーマを使用することで、型の整合性を確保できます。
3. **リアルタイム更新**: ストリーミングデータをUIにリアルタイムで反映できます。

## 実装例

以下は`useObject`の基本的な使用例です：

```tsx
const { submit, isLoading, object, stop } = useObject({
  api: "/api/use-object",
  schema: notificationSchema,
});
```

この実装により、`submit`でAIにリクエストを送信し、`object`でレスポンスを受信できます。また、`isLoading`と`stop`機能を使用することで、ユーザーインターフェースの制御が可能になります。

Remix での例ですが、サーバサイドは以下のように `streamObject` 関数を使うことで簡単に実装できます。

```ts:api.notification.ts
import { openai } from '@ai-sdk/openai'
import type { ActionFunctionArgs } from '@remix-run/cloudflare'
import { streamObject } from 'ai'
import { notificationSchema } from './schema'

export const action = async ({ request }: ActionFunctionArgs) => {
  const context = await request.text()

  const result = await streamObject({
    model: openai('gpt-4-turbo'),
    prompt: `Generate 3 notifications for a messages app in this context: ${context}`,
    schema: notificationSchema,
  })

  return result.toTextStreamResponse()
}
```

zodスキーマはクライアント・サーバ共通で使います

```ts:schema.ts
import { DeepPartial } from 'ai';
import { z } from 'zod';

// define a schema for the notifications
export const notificationSchema = z.object({
  notifications: z.array(
    z.object({
      name: z.string().describe('Name of a fictional person.'),
      message: z.string().describe('Message. Do not use emojis or links.'),
      minutesAgo: z.number(),
    }),
  ),
})

// define a type for the partial notifications during generation
export type PartialNotification = DeepPartial<typeof notificationSchema>
```

Nextjs App Router での実装例は　Vercel AI SDK 自体の Example にありますので参考にしてください。

https://github.com/vercel/ai/blob/main/examples/next-openai/app/use-object/page.tsx

## 開発サポートの実例と感想

`useObject`が含まれたバージョンがリリースされた直後に使用を始めたのですが当時は、`isLoading`状態と`stop`機能が欠けていることに気づきました。これらの機能はユーザー体験を向上させる上で重要だと考え、`Feature Request`のイシューとして開発要望を上げました。

https://github.com/vercel/ai/issues/2082

驚いたことに、開発チームは報告を受けてすぐに対応し、わずか数日でこれらの機能を実装してくれました。この迅速な対応と機能追加には本当に感銘を受けました。オープンソースプロジェクトでここまで迅速かつ丁寧なサポートを受けられるのは稀です。

その後、実装された`isLoading`と`stop`機能の使用中に軽微な問題を発見し、再度イシューとして報告しました。ここでも開発チームの対応速度に驚かされました。報告からわずか4時間以内に問題が修正されたのです。

https://github.com/vercel/ai/issues/2122

これらの経験を通じて、開発チームの熱意と専門性が強く伝わってきました。単なる機能追加やバグ修正以上に、開発者コミュニティの声に真摯に耳を傾け、迅速に行動する姿勢を感じることができました。この対応の速さと質の高さは、`useObject`を使い続ける上で大きな信頼感につながりました。

技術的な優位性に加えて、このような開発者サポートの質の高さも、`useObject`の大きな魅力の一つだと感じています。ライブラリ自体の機能性だけでなく、背後にある開発チームの姿勢も、ツール選択の重要な要素になると実感しました。

## フレームワークの互換性

`useObject`の特筆すべき点として、Next.js App RouterだけでなくRemixでも使用可能な点が挙げられます。これにより、開発者は好みのフレームワークでAI機能を実装できます。フレームワークの選択肢が広がることで、既存のプロジェクトへの導入もスムーズに行えるでしょう。

## 使用例

`useObject`は様々なシナリオで活用できます。例えば：

- チャットボットの応答生成
- AIを使用した文章要約
- リアルタイムでの自然言語処理タスク

これらのタスクでは、`useObject`を使用することで、ストリーミングデータの処理とUIの更新を効率的に行うことができます。

## 注意点

`useObject`は非常に強力なツールですが、適切に使用することが重要です。大量のデータを処理する場合は、パフォーマンスに注意を払う必要があります。また、AIモデルの選択と適切なプロンプト設計も、最終的な結果の質に大きく影響します。

## 結論

Vercel AI SDKの`useObject`は、AIを活用したアプリケーション開発を効率化する強力なツールです。使いやすさ、型安全性、そしてフレームワークの柔軟性を考慮すると、AIウェブアプリケーション開発において有力な選択肢となり得ます。

個人的な経験から、開発チームのサポート品質の高さも、このライブラリの大きな利点だと感じています。機能の迅速な追加や問題への素早い対応は、継続的な改善と信頼性を示しています。

AIを使用したウェブアプリケーションの開発を検討している場合、`useObject`の使用を強くお勧めします。その機能と利便性は、開発プロセスを大幅に改善する可能性があります。ぜひ一度試してみてください。
