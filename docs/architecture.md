# Site Architecture

parthac.me is a static site built with Jekyll on the academicpages theme, deployed to GitHub Pages from the `master` branch.

---

## Directory map

```
partha117.github.io/
├── _config.yml              # Site config, author, SEO, analytics
├── _pages/                  # Content pages (rendered as HTML routes)
│   ├── about.md             # Homepage — the main content file
│   ├── year-archive.md      # Blog archive at /year-archive/
│   ├── sitemap.md           # Human-readable sitemap at /sitemap/
│   └── 404.md               # 404 error page
├── _posts/                  # Blog posts
│   └── YYYY-MM-DD-title.md  # One post per file
├── _includes/               # Reusable HTML partials (theme)
│   ├── seo.html             # Meta tags and JSON-LD — modified
│   ├── head/custom.html     # Favicons, MathJax, custom CSS
│   ├── masthead.html        # Top nav bar
│   ├── sidebar.html         # Author sidebar
│   ├── author-profile.html  # Avatar, bio, social links
│   ├── footer.html          # Footer
│   ├── scripts.html         # JS bundles
│   └── head.html            # <head> assembly (calls seo.html + custom.html)
├── _layouts/                # Page templates (theme)
│   ├── default.html         # Root layout: head + masthead + content + footer
│   ├── single.html          # Standard content page (used by _pages and _posts)
│   ├── archive.html         # List layout (used by year-archive)
│   └── compress.html        # HTML minification wrapper around default
├── _sass/                   # Stylesheets (theme, do not edit)
├── _data/
│   ├── navigation.yml       # Top nav items and their URL targets
│   └── ui-text.yml          # Theme UI strings (multi-language)
├── assets/
│   ├── css/main.scss        # Stylesheet entry point (imports _sass/)
│   └── js/                  # Theme JavaScript
├── images/                  # All images used in content
├── llms.txt                 # AI-readable site summary
├── CLAUDE.md                # Instructions for Claude Code
├── docs/
│   └── architecture.md      # This file
├── Gemfile                  # Ruby dependencies
└── CNAME                    # Custom domain (parthac.me)
```

---

## Request flow

```
Browser request
  → GitHub Pages CDN
    → Jekyll-built _site/ (static HTML)
      → _layouts/default.html (root template)
          → _includes/head.html
              → _includes/seo.html        (meta, og:, JSON-LD)
              → _includes/head/custom.html (favicons, MathJax)
          → _includes/masthead.html       (nav bar from _data/navigation.yml)
          → page content (from _pages/ or _posts/)
          → _includes/sidebar.html
              → _includes/author-profile.html
          → _includes/footer.html
          → _includes/scripts.html
```

---

## How content flows to pages

### Homepage (`/`)
Source: `_pages/about.md`  
Layout: `single` (via defaults in `_config.yml`)  
Structure: one long Markdown+HTML file with named `<div id="...">` sections. Navigation anchors in `_data/navigation.yml` point to these IDs directly (`/#work_experience`, `/#projects`, etc.).

The `#notes` section uses a Liquid `{% for post in site.posts %}` loop to list blog posts automatically.

### Blog posts (`/posts/YYYY/title/`)
Source: `_posts/YYYY-MM-DD-title.md`  
Layout: `single`  
Front matter requires: `title`, `date`, `permalink`  
Also listed at `/year-archive/` (source: `_pages/year-archive.md`)

### Blog archive (`/year-archive/`)
Source: `_pages/year-archive.md`  
Layout: `archive`  
Lists all posts grouped by year using a Liquid loop. No data file — reads `site.posts` directly.

---

## SEO layer

All SEO logic is in `_includes/seo.html`. It runs on every page via `_includes/head.html`.

| Tag | Source |
|-----|--------|
| `<title>` | `page.title` + `site.title` |
| `meta name="description"` | `page.description` → `page.excerpt` → `site.description` |
| `og:title` | `page.title` |
| `og:description` | same cascade as description |
| `og:image` | `page.header.image` → `page.header.overlay_image` → `site.og_image` |
| `og:url` | `site.url` + `page.url` |
| JSON-LD Person | `site.social.type/name/links` |
| JSON-LD Organization logo | `site.og_image` |

Two modifications were made to the default theme `seo.html`:
1. `og:description` condition changed to `{% if seo_description %}` (default was `{% if page.excerpt %}`, which silently dropped page-level `description` front matter)
2. `og:image` falls back to `site.og_image` when no page-level header image is set

---

## Configuration reference (`_config.yml`)

Key sections and what they control:

| Section | Controls |
|---------|---------|
| `title`, `name`, `description` | Site title and SEO description |
| `url` | Canonical base URL (`https://parthac.me`) |
| `author` | Sidebar: avatar, bio, location, social links |
| `og_image` | Default Open Graph and Twitter card image |
| `social` | JSON-LD Person schema (type, name, sameAs links) |
| `analytics.google.tracking_id` | Google Analytics 4 |
| `google_site_verification` | Google Search Console verification |
| `defaults` | Layout and sidebar defaults for `posts` and `pages` |
| `plugins` | jekyll-feed, jekyll-sitemap, jekyll-redirect-from, jemoji, jekyll-gist |

---

## Adding content

### New blog post
1. Create `_posts/YYYY-MM-DD-title.md`
2. Add front matter: `title`, `date`, `permalink: /posts/YYYY/title/`, `tags`
3. Write content in Markdown below the front matter
4. Push — the `#notes` section and `/year-archive/` update automatically

### Updating the homepage
Edit `_pages/about.md`. Sections are HTML `<div>` blocks with IDs that match `_data/navigation.yml` anchors. Adding a new section requires a matching nav entry.

### Adding an image
Drop the file into `images/` and reference it in `about.md` as `src="images/filename.png"`.
