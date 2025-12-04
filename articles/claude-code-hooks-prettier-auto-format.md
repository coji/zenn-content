---
title: 'Claude Code hooks で prettier を自動実行する'
emoji: '🧹'
type: 'tech'
topics: ['claudecode', 'typescript', 'prettier', 'hooks']
published: true
---

:::message
2025-12-04: よりシンプルな書き方に更新しました。
:::

## これはなに？

Claude Code がファイルを編集するたびに prettier を自動実行する hooks の設定です。毎回忘れるのでコピペで反映できるようにしました。

## Claude Code に渡すプロンプト

以下をコピペして Claude Code に渡せば設定完了です。

````text
Claude Code の Hooks でファイル編集後に prettier を自動実行する設定をして。

1. cc-hooks-ts と tsx をインストール（tsx が既にあればスキップ）
pnpm add -D cc-hooks-ts tsx

2. .claude/hooks/format-on-edit.ts を作成
```typescript
import { defineHook, runHook } from 'cc-hooks-ts'
import { execSync } from 'node:child_process'

const FORMATTABLE_EXTENSIONS = [
  '.ts',
  '.tsx',
  '.js',
  '.jsx',
  '.json',
  '.md',
  '.css',
]

const formatHook = defineHook({
  trigger: {
    PostToolUse: {
      Write: true,
      Edit: true,
    },
  },
  run: (context) => {
    const toolInput = context.input.tool_input
    const filePath = toolInput.file_path

    if (
      filePath &&
      FORMATTABLE_EXTENSIONS.some((ext) => filePath.endsWith(ext))
    ) {
      try {
        execSync(`pnpm exec prettier --write "${filePath}"`, {
          stdio: 'inherit',
        })
      } catch {}
    }

    return context.success()
  },
})

await runHook(formatHook)
```

3. .claude/settings.local.json の hooks に追加（既存設定とマージして）
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [{ "type": "command", "command": "pnpm exec tsx .claude/hooks/format-on-edit.ts" }]
      }
    ]
  }
}
```
````

## なぜ cc-hooks-ts を使うのか

公式ドキュメントの jq を使う方法は読みづらいです。

```bash
jq -r '.tool_input.file_path | select(endswith(".ts") or endsWith(".tsx"))' | xargs -r prettier --write
```

cc-hooks-ts を使えば TypeScript で書けて型補完も効きます。trigger で特定のツールに絞れるので、コードもシンプルになります。

2025-12-04: cc-hooks-ts 作者の [@sushichan044](https://x.com/sushichan044) さんから「PostToolUse は特定のツールに絞って実行できる」と教えていただき、ts-pattern を使わないシンプルな書き方に更新しました。ありがとうございます。

https://x.com/sushichan044/status/1996255588754567650

## 参考

- [Claude Code Hooks 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code/hooks)
- [cc-hooks-ts](https://github.com/sushichan044/cc-hooks-ts)
