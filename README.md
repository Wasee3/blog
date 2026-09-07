# Wasee's Blog

A [Jekyll](https://jekyllrb.com/) blog hosted on GitHub Pages at
<https://wasee3.github.io/blog/>.

## Add a post

Add a Markdown file to `_posts/` named `YYYY-MM-DD-title.md`:

```markdown
---
layout: post
title: "Post title"
date: 2026-09-10 12:00:00 +0000
---

Body in Markdown.
```

Commit and push to `main`; GitHub Pages rebuilds automatically.

## Run locally (optional)

Requires Ruby. Then:

```bash
bundle install
bundle exec jekyll serve
```

Open <http://localhost:4000/blog/>.

## Configuration

Site title, description, and nav live in `_config.yml`. Changes to
`_config.yml` require a restart of the local server (GitHub rebuilds on push).
