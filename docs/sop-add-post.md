# SOP: Adding a New Post

The unit of content on this blog is a single Markdown file under `content/posts/<topic>/`. This SOP covers the manual flow. For style-mimicking longer articles, the `/write-tech-article` skill automates much of step 2 onward — see [.claude/commands/write-tech-article.md](../.claude/commands/write-tech-article.md).

For non-content edits (config, theme, menus), use [sop-site-edit.md](sop-site-edit.md) instead.

---

## Step 1: Pick the topic directory

Posts are grouped by topic under `content/posts/`. Pick the existing directory that fits, or create a new one.

| Type | Directory examples |
|------|--------------------|
| Prose-style technical articles | `ai-agent/`, `python/`, `cpp/`, `computer-science/`, `serialization/`, `parallel-computing/`, `service-governance/`, `data/`, `Cloud/`, `harness-engineering/` |
| Code-solution series (style-skip) | `LeetCode/`, `Machine-Learning/`, `Code-Jam/`, `Kick-Start/` |

To create a new topic, just `mkdir content/posts/<topic>/` — no `_index.md` is required.

---

## Step 2: Create the post file

Filename → slug. Use lowercase kebab-case, no Chinese characters in the filename. Example: `content/posts/python/python-source-code-coroutine.md`.

Frontmatter is YAML:

```yaml
---
title: "Python 源码学习（5）：协程"
date: 2021-08-04T17:38:52+08:00
draft: false
categories: ["python"]
---
```

| Field | Notes |
|-------|-------|
| `title` | The display title. Chinese is fine. |
| `date` | RFC 3339 timestamp. Controls sort order on `/archives/`. |
| `draft` | `true` while writing — drafts are excluded from production builds. Flip to `false` to publish. |
| `categories` | Array. Usually matches the topic directory name. PaperMod renders these as taxonomy pages. |

Optional: `tags: [...]` if you want a more granular slice than `categories`.

---

## Step 3: Write the content

- Use standard CommonMark + Goldmark extensions. Raw HTML is enabled (`markup.goldmark.renderer.unsafe: true`).
- Code blocks use fenced syntax with a language tag — Chroma handles highlighting.
- Math: PaperMod doesn't ship MathJax by default; if needed, drop a `<script>` tag in the post (raw HTML is enabled).
- Images / PDFs: drop the file in `static/attachments/` and link to it as `/attachments/<file>`.

---

## Step 4: Preview locally

```bash
hugo server -D    # -D = include drafts
```

Open `http://localhost:1313` and find the post under `/archives/` or its topic page.

For trivial posts you can skip preview, but for anything with code blocks, math, or raw HTML, preview first.

---

## Step 5: Publish

Flip `draft: false` if you haven't already, then:

```bash
git add content/posts/<topic>/<slug>.md
git commit -m "add: <short description>"
git push origin master
```

Commit-message convention (per `git log`): prefix with `add:`, `update:`, or `fix:` followed by a short imperative description.

GitHub Actions then runs `hugo --minify` and pushes the build to `gh-pages`. Takes ~60 s. Watch progress at https://github.com/chr1sc2y/hugo-site/actions.

---

## Step 6: Verify

Open https://prov1dence.top in a fresh tab (or hard-refresh) and confirm the post appears in `/archives/` and at its slug URL.

---

## Adding attachments (resume, PDFs, images)

Drop the file in `static/attachments/`:

```bash
cp ~/Downloads/foo.pdf static/attachments/foo.pdf
```

Reference it from a post as `/attachments/foo.pdf` (the `static/` prefix is stripped at build time).

---

## Checklist

```
[ ] Topic directory chosen / created under content/posts/
[ ] Filename is lowercase kebab-case
[ ] Frontmatter has title, date, draft, categories
[ ] Previewed with `hugo server -D` (skip for trivial posts)
[ ] draft: false set before publish
[ ] git commit with add:/update:/fix: prefix
[ ] git push origin master
[ ] GitHub Action succeeded (Actions tab)
[ ] Verified live on prov1dence.top
```
