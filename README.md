# Technical Notes

Jekyll-based technical blog for GitHub Pages.

## Local preview

```sh
bundle install
bundle exec jekyll serve
```

Then open <http://localhost:4000>.

## Add a post

Create a Markdown file in `_posts/`:

```text
YYYY-MM-DD-title.md
```

Each post needs front matter:

```yaml
---
layout: post
title: "Post Title"
date: 2026-05-14 00:00:00 +0800
categories: category-name
---
```

## Deployment

Pushing to `main` runs the GitHub Actions workflow in `.github/workflows/pages.yml`
and deploys the generated site to GitHub Pages.
