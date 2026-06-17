# blog

Zhengyu Chen 的个人技术博客，地址 [prov1dence.top](https://prov1dence.top/)。

> Agent 维护须知：本文件描述当前生效的工作流，请勿按历史做法操作。详见下方「部署」一节。

## 技术栈

- **生成器**：Hugo v0.143.1（extended）
- **主题**：PaperMod（`themes/PaperMod`，唯一的 git submodule）
- **部署**：GitHub Pages。push 到 `master` 后由 GitHub Actions 自动构建并发布到本仓库的 `gh-pages` 分支（见 `.github/workflows/deploy.yml`）。`gh-pages` 绑定自定义域名 `prov1dence.top`。
- **配置**：`config.yml`

## 目录结构

```
content/posts/          # 博客文章，按主题分目录（_index.md 是 /posts/ 列表页）
content/about.md        # /about
static/attachments/     # 简历 PDF 等静态文件
themes/PaperMod/        # 主题 submodule（唯一的 submodule）
layouts/                # 项目级模板，覆盖主题 partial（如 footer.html）
.github/workflows/      # CI: push master → build → deploy gh-pages
.claude/commands/       # Claude Code skills
public/                 # 本地构建输出，已 gitignore，不提交
```

文章类别：`ai-agent/`、`python/`、`cpp/`、`computer-science/`、`serialization/`、`parallel-computing/`、`service-governance/`、`data/`、`Cloud/`、`harness-engineering/` 为散文式技术文章；`LeetCode/`、`Machine-Learning/`、`Code-Jam/`、`Kick-Start/` 为代码题解，风格学习时跳过。

## 常用命令

```bash
hugo server -D      # 本地预览（含草稿）
hugo --minify       # 仅本地验证构建，产物在 public/（已 gitignore）；CI 会自己构建
```

## 部署（重要）

**当前唯一正确的发布方式：把改动 commit 到 `master` 然后 push 主仓库即可。** GitHub Actions 会自动构建并发布到 `gh-pages`，约 60 秒。

```bash
# VPN 环境下 SSH 22 端口不通，走 443
GIT_SSH_COMMAND="ssh -p 443 -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes" git push origin master
```

Remote：`git@ssh.github.com:chr1sc2y/blog.git`。

### 不要这样做（历史遗留，已废弃）

- **不要**本地 `hugo` 构建后提交 `public/`。`public/` 已 gitignore，构建交给 CI。
- **不要**把构建产物 push 到 `chr1sc2y/prov1dence.github.io`。那是旧的发布仓库，**已于 2026-05-23 归档成只读**，push 会报 `repository was archived so it is read-only`。它不是 submodule，本仓库与它无任何关联。
- 本仓库**没有** `public` submodule，唯一 submodule 是主题 `themes/PaperMod`。

### 改主题样式的正确姿势

不要直接改 `themes/PaperMod/` 里的文件（那是 submodule，改动不被本仓库追踪、易丢失）。把要覆盖的 partial 复制到项目级 `layouts/`（如 `layouts/partials/footer.html`），Hugo 会优先用项目级模板。

## 草稿

front matter 里 `draft: true` 的文章 CI 不会发布（构建命令不带 `-D`），可放心保留作存档。

## Skill

`/write-tech-article <URL>` — 模仿作者风格写深度技术文章，多轮 review 达标后自动部署。详见 `.claude/commands/write-tech-article.md`。
