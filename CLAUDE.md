# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A [Zenn](https://zenn.dev) content repository managed via `zenn-cli`, connected to Zenn through its GitHub repository linking feature. Pushing to `main` deploys published content to Zenn automatically.

## Commands

- `npx zenn new:article` — scaffold a new article under `articles/` with a generated slug
- `npx zenn new:book` — scaffold a new book under `books/`
- `npx zenn preview` — run a local preview server to check rendering before publishing

There is no build/lint/test pipeline; this repo is content only.

## Structure

- `articles/*.md` — one Markdown file per article, filename is the slug. Frontmatter controls publication:
  - `title`, `emoji`, `type` (`tech` or `idea`), `topics`, `published` (`false` until ready to go live)
- `books/*/` — multi-chapter books, each with its own config and chapter files
- Only content with `published: true` on `main` is visible on Zenn — draft by setting `published: false` and flipping it when ready.
