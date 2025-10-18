# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Zenn content repository containing Japanese technical articles and books. Zenn is a developer-focused blogging platform popular in the Japanese tech community. The repository contains markdown articles, multi-chapter books, and associated images.

## Common Commands

### Content Management

```bash
# Preview content locally with hot reload
zenn preview

# Create new article
zenn new:article

# Create new book
zenn new:book

# List all articles
zenn list:articles

# List all books  
zenn list:books
```

### Package Management

```bash
# Install dependencies (uses pnpm)
pnpm install

# Check for outdated packages
pnpm outdated
```

### Content Linting

```bash
# Run textlint on content (textlint is available as dependency)
npx textlint articles/
npx textlint books/
```

## Repository Structure

- `articles/` - Individual blog posts in markdown format with YAML frontmatter
- `books/` - Multi-chapter books organized with config.yaml and numbered chapters
- `images/` - Article images organized by article slug for easy association
- `books/conform-guide/` - 35-chapter comprehensive guide on Conform form library

## Content Architecture

### Articles

- Written in Japanese for Japanese developer audience
- Use YAML frontmatter with title, emoji, type, topics, published status
- Focus on modern web development, AI integration, and performance optimization
- Include practical code examples and benchmarks

### Books

- Multi-chapter comprehensive guides
- Each book has `config.yaml` defining structure and chapters
- Chapters are numbered markdown files (01-introduction.md, 02-setup.md, etc.)
- `conform-guide` is the main complete book with 35 chapters

### Content Topics

- React/Remix ecosystem (routing, forms, SPA development)
- AI/LLM development (Claude, Vercel AI SDK, Gemini)
- Backend technologies (FastAPI, Prisma, PostgreSQL, Supabase)
- UI libraries (shadcn/ui, Conform, XState)
- Database performance optimization

### Writing Style Guide

- **基本の文体**: 丁寧語（です・ます調）を必ず使用する。ただし、形式的になりすぎず、誠実で親しみやすい口調を心がける
  - ✅ 「〜してみました」「〜できます」「〜なります」
  - ❌ 「〜してみた」「〜できる」「〜なる」（常体は使わない）
  - ただし、文中で自然な流れを保つため、「〜なんですが」「〜けど」などの口語的な接続は可
- **体験ベースの語り口**: 「〜を作ってみました」「〜してみた結果です」など、実際の体験を共有する形式
- **能動的な表現**: 「〜により」→「〜で」、受動態より能動態を優先。ただし丁寧語は保つ
- **簡潔な構成**: 長い前置きを避け、すぐ本題に入る。各セクションの導入文は最小限に
- **自然な日本語**: 「〜について解説します」→「〜を作ってみました」、形式的すぎる表現を避ける
- **控えめで誠実な表現**: 大げさな表現や過度な強調を避け、技術的に正確で誠実な言葉を使う
  - ❌ 「完全理解」「爆速」「最強」「革命的」「圧倒的」などの誇張表現
  - ✅ 「理解する」「高速」「便利」「効果的」など控えめで正確な表現
  - 「必ず」「絶対」などの断定的表現は避け、「〜しておくと安心です」「〜がおすすめです」など柔らかい表現を使う
- **絵文字の使用は最低限に**: 記事タイトルのemojiフィールドのみ使用し、本文中の絵文字は効果的な場所で1-2個程度に絞る
  - コード例のコメント内の ✅ ❌ などの絵文字は使わず、「良い例」「悪い例」などテキストで表現
- **箇条書きは本当に必要な場所のみ**: 論理的な文章で説明することを優先し、チェックリストやまとめなど本当に必要な場所でのみ箇条書きを使う
  - 特徴や利点の説明は文章で記述する
  - 番号付きリストも可能な限り文章に統合する

### Article Structure Guidelines

- **冒頭に「これはなに？」セクション**: 記事の内容を簡潔に説明する導入部を設ける
- **デモやソースコードのリンク**: 動作するデモやGitHubリポジトリがある場合は冒頭に明記
- **コード量の適正化**: ポイントを絞って必要最小限のコードのみ掲載。冗長な例は避ける
- **実装内容の正確性**: 記事内のコードは実際のソースコードと一致させる。推測で書かない
- **検証記事の明確化**: 実装したものと検証・調査したものを明確に区別する

## Development Notes

- Content is primarily in Japanese
- Uses Zenn CLI for content management and preview
- Package manager is pnpm (not npm)
- No custom build process - relies on Zenn platform for publishing
- Images should be placed in `images/` directory organized by article slug
