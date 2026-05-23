# blog

Zhengyu Chen 的个人技术博客，地址 [prov1dence.top](https://prov1dence.top/)。

## 技术栈

- **生成器**：Hugo v0.143.1（extended）
- **主题**：PaperMod（`themes/PaperMod`，git submodule）
- **部署**：GitHub Pages，构建产物推到本仓库的 `gh-pages` 分支（由 GitHub Actions 管理，见 `.github/workflows/deploy.yml`）
- **配置**：`config.yml`

## 目录结构

```
content/posts/          # 博客文章，按主题分目录（_index.md 是 /posts/ 列表页）
content/about.md        # /about
static/attachments/     # 简历 PDF 等静态文件
themes/PaperMod/        # 主题 submodule
.github/workflows/      # CI: push to master → build → deploy to gh-pages
docs/                   # 运维文档（tech-stack / maintenance / SOPs）
.claude/commands/       # Claude Code skills
```

文章类别：`ai-agent/`、`python/`、`cpp/`、`computer-science/`、`serialization/`、`parallel-computing/`、`service-governance/`、`data/`、`Cloud/`、`harness-engineering/` 为散文式技术文章；`LeetCode/`、`Machine-Learning/`、`Code-Jam/`、`Kick-Start/` 为代码题解，风格学习时跳过。

## 常用命令

```bash
hugo server -D      # 本地预览（含草稿）
hugo --minify       # 构建到 public/（仅本地预览用，CI 会自己构建）
```

## 部署

推到 `master` 即可，GitHub Actions 自动构建并发布到 `gh-pages`，约 60 秒。**不要手动 `hugo` 后提交 `public/`**，也不要碰 `gh-pages` 分支。详见 `docs/sop-site-edit.md`。

VPN 环境下 SSH 22 端口不通，走 443：

```bash
GIT_SSH_COMMAND="ssh -p 443 -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes" git push origin master
```

Remote: `git@ssh.github.com:chr1sc2y/blog.git`。

## Skill

`/write-tech-article <URL>` — 模仿作者风格写深度技术文章，多轮 review 达标后自动部署。详见 `.claude/commands/write-tech-article.md`。
