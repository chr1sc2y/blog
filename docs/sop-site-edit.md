# SOP: Editing Site Config or Theme (non-post)

For any change that isn't adding a post — e.g. tweaking `config.yml`, editing the menu, updating the PaperMod theme, editing `about.md`, changing CI, updating docs.

For new posts, use [sop-add-post.md](sop-add-post.md) instead.

---

## Deploy model — read this first

This site is deployed by **GitHub Actions**, not by hand. There is **one repo** (`chr1sc2y/blog`); the built HTML lives on the `gh-pages` branch and is managed entirely by CI.

- You do **not** run `hugo` locally to deploy.
- You do **not** push to a second repo. (The old `public/` submodule pointing at `chr1sc2y/prov1dence.github.io` was retired in commit `f0f3726`, 2026-05-23.)
- You do **not** touch the `gh-pages` branch by hand.

The only thing you push is your source change on `master`. See [.github/workflows/deploy.yml](../.github/workflows/deploy.yml) and [tech-stack.md](tech-stack.md#hosting--deployment).

---

## Steps

### 1. Make the edit

Edit files at the repo root or under `content/`, `static/`, `themes/`, `docs/`, etc. as needed.

### 2. (Optional) Preview locally

```bash
hugo server -D
```

Open `http://localhost:1313` to verify. Skip this for trivial edits (typos, docs).

### 3. Commit

```bash
git add <changed files>
git commit -m "<add:/update:/fix: short description>"
```

Match the existing commit-message style: `add:`, `update:`, or `fix:` prefix.

### 4. Push to master

```bash
git push origin master
```

If SSH on port 22 is blocked (VPN), use the port-443 fallback documented in `CLAUDE.md`.

### 5. Wait for the Action

GitHub Actions runs `hugo --minify` and pushes the build to `gh-pages`. Takes ~60 s. Watch at https://github.com/chr1sc2y/blog/actions.

### 6. Verify

Open https://prov1dence.top in a fresh tab (or hard-refresh) and confirm the change is live.

---

## Updating the PaperMod theme

```bash
git submodule update --remote themes/PaperMod
hugo server -D    # confirm site still builds & looks right
git add themes/PaperMod
git commit -m "update: bump PaperMod to <sha>"
git push origin master
```

The submodule pointer is what gets committed — not the contents.

---

## What if I want to render locally anyway?

You can — `hugo --minify` produces output in `public/` (gitignored). It's useful for previewing the production-minified output, but it's never committed and never deployed. Only pushes to `master` trigger a real deploy.

---

## Checklist

```
[ ] Edit made
[ ] (optional) Previewed with `hugo server -D`
[ ] git commit with add:/update:/fix: prefix
[ ] git push origin master
[ ] GitHub Action succeeded (Actions tab)
[ ] Verified live on prov1dence.top
```
