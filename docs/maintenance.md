# Maintenance Status

## Ownership

| Role | Owner |
|------|-------|
| **Content owner** | Prov1dence (chr1sc2y) — writes all posts |
| **Technical maintainer** | Prov1dence, with Claude (AI) assistance via Claude Code |
| **Hosting account** | chr1sc2y (GitHub) |

The owner writes the posts. Claude assists with infrastructure, theme tweaks, and the `write-tech-article` skill workflow.

---

## Status

**Site status:** Active
**Deployment:** Automated (push-to-deploy on `master`)
**Last infrastructure migration:** 2026-05-23 — replaced `public/` submodule + manual `hugo` deploy with GitHub Actions building to a `gh-pages` branch on this repo. The old `chr1sc2y/prov1dence.github.io` repo was retired.

---

## Operational Health

### What's automated

- Build and deploy on every push to `master` (GitHub Actions, `.github/workflows/deploy.yml`)
- TLS certificate renewal (GitHub Pages)
- RSS / sitemap / JSON search index generation at build time

### What requires manual action

| Task | Trigger | Owner |
|------|---------|-------|
| Write a new post | Content owner decision | Owner (optionally with Claude via `/write-tech-article`) |
| Update Hugo version | Security note or theme requirement | Owner / Claude |
| Update PaperMod theme | Theme release | Owner / Claude (`git submodule update --remote themes/PaperMod`) |
| DNS changes | Domain renewal or provider change | Owner |
| GitHub Pages config | Domain change | Owner (browser) |

### Maintenance windows

No scheduled windows. GitHub Pages serves static content with no downtime; deploys take ~60 s and swap atomically.

---

## Known Constraints

- **GitHub Pages free tier:** 1 GB soft storage limit, 100 GB/month bandwidth. Current repo is well within limits.
- **Apex domain:** `prov1dence.top` is configured as an apex (not `www.`). DNS uses A records pointing at GitHub Pages IPs — changing this requires owner action at the registrar.
- **Single repo:** there is no longer a `public/` submodule. The compiled site lives only on the `gh-pages` branch and is owned by CI — never edited by hand.
- **VPN / SSH 22 blocked:** the maintainer's environment sometimes blocks SSH on port 22. Push via `ssh.github.com:443` (see CLAUDE.md).

---

## Dependency Update Policy

| Dependency | Policy |
|------------|--------|
| Hugo | Pinned in `deploy.yml`. Test locally before bumping; PaperMod compatibility must hold. |
| PaperMod | Submodule. Run `git submodule update --remote themes/PaperMod`, build locally, commit the new pointer. |
| GitHub Actions | `@v3` / `@v4` floating tags. Watch for breaking changes on major bumps. |

---

## Incident Response

**Site down (DNS / Pages issue):**
1. Check `https://www.githubstatus.com/`.
2. Verify DNS: `dig prov1dence.top A` — should return GitHub Pages IPs (185.199.108.153 / .109.153 / .110.153 / .111.153).
3. Verify Pages config: GitHub → `chr1sc2y/blog` → Settings → Pages → custom domain = `prov1dence.top`, source = `gh-pages` branch.

**Deployment failing:**
1. Check the Actions tab on `chr1sc2y/blog` for error logs.
2. Common causes: Hugo version mismatch, malformed frontmatter, missing PaperMod submodule (Actions does `submodules: recursive`, so this would mean the submodule URL is broken).

**Broken page or 404 after deploy:**
1. Confirm the page builds locally with `hugo server -D`.
2. Check the post's `draft:` flag — `draft: true` posts are excluded from production builds.
3. Check the slug — Hugo derives URLs from filename + frontmatter; renames change URLs.

**Push blocked by SSH:**
Use the port-443 fallback documented in `CLAUDE.md`:

```bash
GIT_SSH_COMMAND="ssh -p 443 -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes" git push origin master
```
