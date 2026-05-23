# Tech Stack

## Overview

Static technical blog hosted on GitHub Pages, built with Hugo. No backend, no database — all content is compiled at deploy time from Markdown.

**Live URL:** https://prov1dence.top
**Repository:** https://github.com/chr1sc2y/hugo-site
**Maintained by:** Prov1dence (chr1sc2y), with Claude (AI) assistance via Claude Code

---

## Stack

### Site Generator

| Component | Details |
|-----------|---------|
| **Hugo** | v0.143.1 extended |
| **Theme** | [PaperMod](https://github.com/adityatelange/hugo-PaperMod) (git submodule at `themes/PaperMod`) |

Hugo compiles `content/` into a static site. PaperMod handles layout, dark/light theme, table of contents, archives, tag/category pages, and search (Fuse.js, client-side).

Hugo 0.143.1 builds the current repo cleanly with no deprecation warnings — no need to pin to an older version.

### Hosting & Deployment

| Component | Details |
|-----------|---------|
| **Host** | GitHub Pages |
| **Deploy branch** | `gh-pages` (auto-managed by Actions) |
| **CI/CD** | GitHub Actions (`.github/workflows/deploy.yml`) |
| **Trigger** | Every push to `master` |
| **Custom domain** | `prov1dence.top` (apex) |
| **DNS** | A/ALIAS records → GitHub Pages IPs |
| **TLS** | Auto-provisioned by GitHub Pages |

Workflow: push to `master` → Actions builds Hugo → pushes compiled HTML/assets to `gh-pages` → GitHub Pages serves it.

This replaced an older flow (pre-2026-05) where `public/` was a submodule pointing at `chr1sc2y/prov1dence.github.io` and `hugo` was run by hand on the maintainer's laptop. See [sop-site-edit.md](sop-site-edit.md) for the post-migration flow.

### Content Structure

```
content/
├── about.md              # /about page
├── archives.md           # /archives page
└── posts/                # All blog posts, grouped by topic
    ├── ai-agent/
    ├── Cloud/
    ├── computer-science/
    ├── cpp/
    ├── data/
    ├── harness-engineering/
    ├── parallel-computing/
    ├── python/
    ├── serialization/
    ├── service-governance/
    ├── LeetCode/         # Code solutions (style-skip)
    ├── Machine-Learning/ # Code solutions (style-skip)
    ├── Code-Jam/         # Code solutions
    ├── Kick-Start/       # Code solutions
    └── ...               # 25+ topic directories
```

Each post is a single `.md` file under a topic directory. Frontmatter is YAML with `title`, `date`, `draft`, `categories`. No page bundles, no per-post images directory — assets, if any, are referenced from `static/`.

```yaml
---
title: "Python 源码学习（5）：协程"
date: 2021-08-04T17:38:52+08:00
draft: false
categories: ["python"]
---
```

### Static Assets

```
static/
└── attachments/          # Resume PDFs (Chinese / English)
```

Anything under `static/` is copied verbatim into the build output at the same path.

### Config

Hugo config is `config.yml` at repo root. Key settings:

- `baseURL: https://prov1dence.top/`
- `theme: PaperMod`
- `params.defaultTheme: auto` (respects system preference)
- `params.profileMode.enabled: true` (home page is a profile card)
- `outputs.home: [HTML, RSS, JSON]` (JSON output feeds PaperMod's search)
- `menu.main`: `Posts` (`/archives/`), `About` (`/about/`)

There is a benign duplicate `theme: PaperMod` line in `config.yml` — Hugo accepts it; left alone for now.

---

## Dependencies

| Dependency | Version | Purpose | Update policy |
|------------|---------|---------|--------------|
| Hugo | 0.143.1 | Site generator | Pin in CI; bump deliberately |
| PaperMod | latest `master` (submodule) | Theme | Pin via submodule SHA; update manually |
| peaceiris/actions-hugo | v3 | CI Hugo setup | Auto minor |
| peaceiris/actions-gh-pages | v4 | CI deploy | Auto minor |

---

## Local Development

Requirements: Hugo extended ≥ 0.143.1, Git.

```bash
git clone --recurse-submodules git@github.com:chr1sc2y/hugo-site.git
cd hugo-site
hugo server -D       # -D = include drafts
```

Site available at `http://localhost:1313`.

To produce a one-off build into `public/` (gitignored, not deployed):

```bash
hugo --minify
```

You never need to run this for deployment — the GitHub Action does it.
