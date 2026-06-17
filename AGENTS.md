# hugo-site

Zhengyu Chen 的个人技术博客，地址 [prov1dence.top](https://prov1dence.top/)。

## 技术栈

- **生成器**：Hugo v0.143.1（extended）
- **主题**：PaperMod（`themes/PaperMod`，git submodule）
- **部署**：GitHub Pages，`public/` 指向 `chr1sc2y/prov1dence.github.io`（git submodule）
- **配置**：`config.yml`

## 目录结构

```
content/posts/          # 博客文章，按主题分目录
static/attachments/     # 简历 PDF 等静态文件
themes/PaperMod/        # 主题 submodule
public/                 # 构建输出（submodule）
.Codex/commands/       # Codex skills
```

文章类别：`ai-agent/`、`python/`、`cpp/`、`computer-science/`、`serialization/`、`parallel-computing/`、`service-governance/`、`data/`、`Cloud/`、`harness-engineering/` 为散文式技术文章；`LeetCode/`、`Machine-Learning/`、`Code-Jam/`、`Kick-Start/` 为代码题解，风格学习时跳过。

## 常用命令

```bash
hugo server -D      # 本地预览
hugo --minify       # 构建到 public/
```

## 部署

先推 `public` submodule，再推主仓库。VPN 环境下 SSH 22 端口不通，走 443：

```bash
# SSH fallback
GIT_SSH_COMMAND="ssh -p 443 -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes" git push origin master
```

两个仓库的 remote 均使用 `git@ssh.github.com:chr1sc2y/<repo>.git`。

## Skill

`/write-tech-article <URL>` — 模仿作者风格写深度技术文章，多轮 review 达标后自动部署。详见 `.Codex/commands/write-tech-article.md`。
