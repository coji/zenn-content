# CLAUDE.md

This file provides guidance to Claude Code when working with this Zenn content repository.

## Project Overview

Zenn向けの技術記事・書籍を管理するリポジトリです。日本語の技術記事（articles/）と複数章で構成される書籍（books/）を含みます。

### Repository Structure

- `articles/` - 個別の技術記事（YAML frontmatter付きmarkdown）
- `books/` - 複数章で構成される書籍（config.yaml + 番号付き章）
- `images/` - 記事スラッグごとに整理された画像ファイル

### Content Topics

React/Remix、AI/LLM（Claude, Vercel AI SDK）、FastAPI、Prisma、PostgreSQL、Supabase、shadcn/ui、Conform、XStateなど、モダンWeb開発とAI統合が中心です。

## Common Commands

```bash
# プレビュー（ホットリロード対応）
zenn preview

# 新規作成
zenn new:article
zenn new:book

# パッケージ管理（pnpm使用）
pnpm install
pnpm outdated

# textlint実行
npx textlint articles/
npx textlint books/
```

## Writing Guidelines

### 文体と表現

**基本スタイル**: 丁寧語（です・ます調）で、誠実で親しみやすい口調。体験を共有する形式で書く。

- ✅ 「〜してみました」「〜できます」「〜なんですが」「〜けど」
- ❌ 「〜してみた」「〜できる」（常体は使わない）
- ❌ 「完全理解」「爆速」「最強」「革命的」「圧倒的」（誇張表現）
- ❌ 「必ず」「絶対」（断定的表現）→「〜しておくと安心です」

**文章の原則**:
- 簡潔でクリアな論旨。文章は短く明快に
- 長い前置きを避け、すぐ本題に入る
- 能動的な表現を優先（「〜により」→「〜で」）
- 形式的すぎる表現を避ける（「〜について解説します」→「〜を作ってみました」）
- 文末コロンは使わない（「以下のようになります：」→「以下のようになります。」）

### 記事構成の要素

**最小限の使用**:
- 箇条書き: チェックリストやまとめなど本当に必要な箇所のみ。説明は文章で
- 表組: どうしても必要な場合のみ
- 絵文字: frontmatterのemojiフィールドのみ。本文中は使わない
- ソースコード: ポイント理解に必要な部分のみ抜粋。完全なコードはGitHubリンクで代替

### 記事構成

**必須要素**:
- 冒頭に「これはなに？」セクション: 記事内容を簡潔に説明
- デモ/GitHubリンク: ある場合は冒頭に明記
- コードは実際のソースと一致させる（推測で書かない）
- 実装と検証・調査を明確に区別

### タイトル命名

**基本原則**:
- 記事本文・見出しに登場するキーワードを使う
- サブタイトルを積極的に設ける: 「メインタイトル - サブタイトル」
  - メインタイトル: 記事の主題を簡潔に
  - サブタイトル: キャッチコピーor 3つ程度の列挙で魅力を打ち出す

**避けるべき表現**:
- ❌ 「の話」「について」「の件」（汎用的すぎる）
- ❌ メインとサブでキーワード重複（「AのB - AがC」）
- ❌ メインとサブ両方で同じ品詞の用言止め（「Aで遊ぶ - Bを構築する」）
- ✅ 体言止めはメインとサブ両方で使用可能
