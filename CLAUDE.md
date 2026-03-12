# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A GitHub-based blog publishing system that automatically syncs Markdown files to [dev.to](https://dev.to) via the [`sinedied/publish-devto`](https://github.com/sinedied/publish-devto) GitHub Action. Pushing to `main` publishes or updates articles. Opening a PR triggers a dry run and posts results as a PR comment.

## Publishing Flow

- **Publish/update**: push to `main` — the workflow runs `publish-devto` against `posts/**/*.md`
- **Preview**: open a PR — dry run runs and comments the diff summary on the PR
- **Conventional commits** are enabled; commit message prefixes affect how changes are reported

## Article Format

Articles live in `posts/` as Markdown files with required YAML frontmatter:

```markdown
---
title: 'Article Title'
description: Short description
tags: 'tag1, tag2, tag3'
cover_image: ./assets/image.png
canonical_url: null
published: true
id: <assigned by dev.to after first publish>
date: '<ISO timestamp assigned by dev.to>'
---
```

- `id` and `date` are written back to the file automatically by the publish action after first publication — do not set them manually
- Always start new posts with `published: false` — never set `published: true` unless explicitly told the post is ready to go live
- Use `./assets/basic_header_1000x420.png` as the default `cover_image` for new posts unless told otherwise
- Cover images go in `posts/assets/`
- Write prose as single continuous lines — no hard line breaks within paragraphs
- First reference to any external product or tool should be hyperlinked

## Repo Structure

- `posts/` — published and in-progress articles
- `posts/assets/` — images referenced by articles
- `working_docs/` — drafts, outlines, and scratch notes (not published)
- `.github/workflows/publish.yml` — the publish action configuration
- `.env` — local dev.to API token (never commit this)

## Working Conventions

- Never commit or push without being explicitly asked to do so

## Secrets Required

- `DEVTO_TOKEN` — dev.to API key, stored as a GitHub Actions secret
- `GITHUB_TOKEN` — provided automatically by GitHub Actions (used to commit `id`/`date` writebacks)
