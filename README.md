# parthac.me

Personal site for [Partha Chakraborty](https://parthac.me). Built with Jekyll and the [academicpages](https://github.com/academicpages/academicpages.github.io) theme, hosted on GitHub Pages.

## Prerequisites

- Ruby >= 3.0
- Bundler (`gem install bundler`)

## Local development

```bash
bundle install
bundle exec jekyll serve --livereload
```

Open [http://localhost:4000](http://localhost:4000). The `--livereload` flag refreshes the browser on file save.

## Build

```bash
bundle exec jekyll build
```

Output goes to `_site/` (gitignored).

## Deploy

Push to `master`. GitHub Actions picks it up and deploys to GitHub Pages automatically. The live site is at [parthac.me](https://parthac.me).

## Content

| What | Where |
|------|-------|
| Homepage (about, experience, projects, skills) | `_pages/about.md` |
| Blog posts | `_posts/YYYY-MM-DD-title.md` |
| Site config, SEO, author info | `_config.yml` |
| Navigation | `_data/navigation.yml` |
| Images | `images/` |

### Adding a blog post

Create a file in `_posts/` named `YYYY-MM-DD-your-title.md` with this front matter:

```yaml
---
title: "Your Post Title"
date: YYYY-MM-DD
permalink: /posts/YYYY/your-title/
tags:
  - tag1
  - tag2
---
```
