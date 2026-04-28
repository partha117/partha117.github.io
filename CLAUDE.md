# CLAUDE.md

## What this repo is

Personal portfolio site for Partha Chakraborty (parthac.me). Built on the academicpages Jekyll theme, deployed via GitHub Pages. All real content lives in two files: `_pages/about.md` and `_posts/`.

## Key files

| File | Purpose |
|------|---------|
| `_pages/about.md` | Homepage: bio, research interests, work experience, projects, publications, education, services, skills, notes |
| `_posts/YYYY-MM-DD-title.md` | Blog posts (currently one post) |
| `_config.yml` | Site config, author info, SEO (description, og_image, social schema) |
| `_data/navigation.yml` | Top nav links |
| `_includes/seo.html` | SEO meta tags and JSON-LD — modified from theme default |
| `_includes/head/custom.html` | Favicons, MathJax, custom CSS |
| `images/` | All images referenced from about.md |
| `llms.txt` | AI-readable summary of the site owner |
| `docs/architecture.md` | Full site architecture reference |

## Running locally

```bash
bundle install
bundle exec jekyll serve --livereload
```

Opens at http://localhost:4000.

## Deployment

Push to `master`. GitHub Actions deploys to GitHub Pages automatically. Live at parthac.me.

## How content is structured

The homepage (`_pages/about.md`) is one long Markdown/HTML hybrid file with anchor-linked sections: `#work_experience`, `#projects`, `#publications`, `#education`, `#services`, `#skills`, `#notes`. Navigation in `_data/navigation.yml` points to these anchors.

Blog posts appear in the `#notes` section via a Liquid loop and also at `/year-archive/`.

## Things NOT to do

- Do not add back the empty collections (`_talks`, `_teaching`, `_publications`, `_portfolio`) — they were cleaned out intentionally; real content is in `about.md`.
- Do not edit files in `_site/` — that directory is gitignored and rebuilt on every deploy.
- Do not add em dashes to content — preference is for plain connective language.
- Do not add comments to template/theme files unless modifying behavior.

## SEO notes

`_includes/seo.html` has two non-default modifications:
1. `og:description` condition changed from `page.excerpt` to `seo_description` so pages with `description` in front matter get a meta description tag.
2. `og:image` falls back to `site.og_image` when no `page.header.image` is set.
