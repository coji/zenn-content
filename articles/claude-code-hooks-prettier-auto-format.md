---
title: 'Claude Code hooks で prettier を自動実行する'
emoji: '🧹'
type: 'tech'
topics: ['claudecode', 'typescript', 'prettier', 'hooks']
published: true
---

## これはなに？

Claude Code がファイルを編集するたびに prettier を自動実行する hooks の設定です。毎回忘れるのでコピペで反映できるようにしました。

## Claude Code に渡すプロンプト

以下をコピペして Claude Code に渡せば設定完了です。

````text
Claude Code の Hooks でファイル編集後に prettier を自動実行する設定をして。

1. cc-hooks-ts と ts-pattern をインストール（ts-pattern が既にあればスキップ）
pnpm add -D cc-hooks-ts ts-pattern

2. .claude/hooks/format-on-edit.ts を作成
```typescript
import { defineHook, runHook } from 'cc-hooks-ts'
import { execSync } from 'node:child_process'
import { match, P } from 'ts-pattern'

const EDIT_TOOLS = P.union(
  'Write' as const,
  'Edit' as const,
  'MultiEdit' as string,
)

const formatOnEditHook = defineHook({
  trigger: { PostToolUse: true },
  run: (context) =>
    match(context.input)
      .with(
        { tool_name: EDIT_TOOLS, tool_input: { file_path: P.string.endsWith('.ts') } },
        { tool_name: EDIT_TOOLS, tool_input: { file_path: P.string.endsWith('.tsx') } },
        { tool_name: EDIT_TOOLS, tool_input: { file_path: P.string.endsWith('.js') } },
        { tool_name: EDIT_TOOLS, tool_input: { file_path: P.string.endsWith('.jsx') } },
        { tool_name: EDIT_TOOLS, tool_input: { file_path: P.string.endsWith('.json') } },
        { tool_name: EDIT_TOOLS, tool_input: { file_path: P.string.endsWith('.md') } },
        { tool_name: EDIT_TOOLS, tool_input: { file_path: P.string.endsWith('.css') } },
        ({ tool_input }) => {
          try {
            execSync(`pnpm exec prettier --write "${tool_input.file_path}"`, { stdio: 'inherit' })
          } catch {}
          return context.success()
        },
      )
      .otherwise(() => context.success()),
})

await runHook(formatOnEditHook)
```

3. .claude/settings.local.json の hooks に追加（既存設定とマージして）
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [{ "type": "command", "command": "pnpm exec tsx .claude/hooks/format-on-edit.ts" }]
      }
    ]
  }
}
```
````

## なぜ cc-hooks-ts と ts-pattern を使うのか

公式ドキュメントの jq を使う方法は読みづらいです。

```bash
jq -r '.tool_input.file_path | select(endswith(".ts") or endswith(".tsx"))' | xargs -r prettier --write
```

cc-hooks-ts を使えば TypeScript で書けて型補完も効きます。ts-pattern を使うとパターンマッチで拡張子ごとの分岐がすっきり書けます。

## 参考

- [Claude Code Hooks 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code/hooks)
- [cc-hooks-ts](https://github.com/sushichan044/cc-hooks-ts)
- [ts-pattern](https://github.com/gvergnaud/ts-pattern)
