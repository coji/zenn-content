---
title: 'Claude Code hooks で prettier を自動実行する'
emoji: '🧹'
type: 'tech'
topics: ['claudecode', 'typescript', 'prettier', 'hooks']
published: true
---

## これはなに？

Claude Code がファイルを編集するたびに prettier を自動実行する hooks の設定です。

人間がコードを書いていた頃はエディタの自動フォーマット機能がありましたが、Claude Code がコードを書く場合はフォーマット実行が LLM の判断次第になります。hooks を使えば確実に実行されます。

## 設定方法

### 1. cc-hooks-ts をインストール

```bash
pnpm add -D cc-hooks-ts
```

[cc-hooks-ts](https://github.com/sushichan044/cc-hooks-ts) を使うと hooks を TypeScript で書けます。公式ドキュメントの jq を使う方法より読みやすく、型補完も効きます。

### 2. フックファイルを作成

`.claude/hooks/format-on-edit.ts`:

```typescript
import { defineHook, runHook } from 'cc-hooks-ts'
import { execSync } from 'node:child_process'
import { match, P } from 'ts-pattern'

const EDIT_TOOLS = P.union(
  'Write' as const,
  'Edit' as const,
  'MultiEdit' as string, // cc-hooks-ts の型定義にないため string でキャスト
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
          } catch {
            // prettier が失敗してもワークフローは止めない
          }
          return context.success()
        },
      )
      .otherwise(() => context.success()),
})

await runHook(formatOnEditHook)
```

### 3. settings.local.json に追加

`.claude/settings.local.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "pnpm exec tsx .claude/hooks/format-on-edit.ts"
          }
        ]
      }
    ]
  }
}
```

### 4. Claude Code を再起動

設定を反映するために再起動が必要です。

## 参考

- [Claude Code Hooks 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code/hooks)
- [cc-hooks-ts](https://github.com/sushichan044/cc-hooks-ts)
- [ts-pattern](https://github.com/gvergnaud/ts-pattern)
