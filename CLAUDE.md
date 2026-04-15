# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A personal Hexo (v3.9.0) blog using the NexT (v7.0.1) theme, deployed to GitHub Pages. The blog is written in Chinese (zh-CN) and covers technical topics. Site URL: https://hiram711.github.io/blog

## Common Commands

```bash
# Local development server (runs on http://localhost:4000/blog)
hexo server          # or: hexo s

# Generate static files
hexo generate        # or: hexo g

# Clean generated files and cache
hexo clean

# Deploy to GitHub Pages (gh-pages branch)
hexo deploy          # or: hexo d

# Generate then deploy
hexo g -d

# Create a new post
hexo new "Post Title"

# Create a new draft
hexo new draft "Draft Title"

# Publish a draft to _posts
hexo publish "Draft Title"
```

## Architecture

- **Branch model**: `hexo` branch holds source files; `gh-pages` branch holds the built site
- **Config**: `_config.yml` (site config), `themes/next/_config.yml` (theme config)
- **Content**: `source/_posts/` (published), `source/_drafts/` (drafts), `source/_discarded/` (archived)
- **Pages**: `source/about/`, `source/categories/`, `source/tags/`, `source/404/`
- **Images**: `source/images/` (PNG assets referenced by posts)
- **Scaffolds**: `scaffolds/` contains templates for `post`, `draft`, and `page`
- **Theme**: `themes/next/` — NexT theme with Swig templates in `layout/`, Stylus CSS in `source/css/`
- **Build output**: `public/` (gitignored)

## Post Front Matter

Posts use this front matter format (see `scaffolds/post.md`):

```yaml
---
title: Post Title
date: YYYY-MM-DD HH:mm:ss
tags:
  - tag1
  - tag2
---
```

## Key Configuration

- Permalink pattern: `:year/:month/:day/:title/`
- Root path: `/blog` (all URLs are prefixed with this)
- Deployment: git-based to `git@github.com:Hiram711/blog.git` on `gh-pages` branch
- Search: hexo-generator-searchdb enabled
- Word count / reading time: hexo-symbols-count-time enabled
