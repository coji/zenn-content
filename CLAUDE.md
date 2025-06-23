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

- **体験ベースの語り口**: 「〜を作ってみました」「〜してみた結果です」など、実際の体験を共有する形式
- **読者との距離感**: 「〜ですね」「〜じゃないですか」「〜と感じています」など、親しみやすい口調
- **能動的な表現**: 「〜できます」→「〜できる」、「〜により」→「〜で」、受動態より能動態を優先
- **簡潔な構成**: 長い前置きを避け、すぐ本題に入る。各セクションの導入文は最小限に
- **自然な日本語**: 「〜について解説します」→「〜を作ってみました」、形式的すぎる表現を避ける

## Development Notes

- Content is primarily in Japanese
- Uses Zenn CLI for content management and preview
- Package manager is pnpm (not npm)
- No custom build process - relies on Zenn platform for publishing
- Images should be placed in `images/` directory organized by article slug