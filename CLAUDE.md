# hugo-site

Zhengyu Chen 的个人技术博客，部署在 [prov1dence.top](https://prov1dence.top/)。

## 技术栈

| 组件 | 说明 |
|------|------|
| 静态站点生成器 | [Hugo](https://gohugo.io/) v0.143.1（extended + withdeploy） |
| 主题 | [PaperMod](https://github.com/adityatelange/hugo-PaperMod)（git submodule，路径 `themes/PaperMod`） |
| 部署目标 | GitHub Pages，仓库 `chr1sc2y/prov1dence.github.io`（git submodule，路径 `public`） |
| 配置文件 | `config.yml` |

## 仓库结构

```
hugo-site/
├── config.yml              # Hugo 站点配置（baseURL、主题、菜单、搜索等）
├── content/
│   ├── about.md            # About 页面
│   ├── archives.md         # 归档页
│   └── posts/              # 所有博客文章，按主题分目录
│       ├── ai-agent/       # AI Agent 架构分析（最新系列）
│       ├── python/         # Python 源码解析系列
│       ├── cpp/            # C++ 技术深度文章
│       ├── computer-science/
│       ├── serialization/
│       ├── parallel-computing/
│       ├── service-governance/
│       ├── data/
│       ├── Cloud/
│       ├── harness-engineering/
│       └── ...             # 其他主题目录
├── static/
│   └── attachments/        # 简历 PDF 等静态文件
├── themes/
│   └── PaperMod/           # 主题 submodule
├── public/                 # 构建输出，指向 prov1dence.github.io submodule
├── archetypes/
│   └── default.md          # 新文章模板
└── .claude/
    └── commands/
        └── write-tech-article.md  # 技术文章写作 skill
```

## 常用命令

```bash
# 本地预览（默认 http://localhost:1313）
hugo server -D

# 构建生产版本（输出到 public/）
hugo --minify

# 新建文章
hugo new posts/<主题>/<文章名>.md
```

## 文章 Front Matter 模板

```yaml
---
title: "<文章标题>"
date: 2026-01-01T10:00:00+08:00
draft: false
categories: ["<分类>"]
---
```

`draft: true` 时本地预览可见，`hugo --minify` 构建时不输出。

## 部署流程

构建产物在 `public/`（submodule 指向 `chr1sc2y/prov1dence.github.io`），需要先提交 `public`，再提交主仓库。

**SSH 注意**：VPN 环境下 `github.com` DNS 可能被劫持（解析到 `198.18.0.147`），SSH 22 端口不通。绕过方式是走 `ssh.github.com:443`，`~/.ssh/config` 已配置对应条目，使用 `id_ed25519`（`chr1sc2y` 个人账号）。

```bash
# 1. 构建
hugo --minify

# 2. 提交 public submodule
cd public
git remote set-url origin git@ssh.github.com:chr1sc2y/prov1dence.github.io.git
git add . && git commit -m "update: ..." && git push origin master

# 3. 提交主仓库
cd ..
git remote set-url origin git@ssh.github.com:chr1sc2y/hugo-site.git
git add content/posts/<新文章> public
git commit -m "update: ..." && git push origin master

# SSH fallback（config 不生效时）
GIT_SSH_COMMAND="ssh -p 443 -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes" git push origin master
```

## 文章分类说明

| 目录 | 内容类型 |
|------|----------|
| `ai-agent/` | AI Agent 系统架构分析（散文式深度技术文章） |
| `python/` | CPython 源码解析系列 |
| `cpp/` | C++ 语言特性、并发、智能指针等 |
| `computer-science/` | 计算机基础（OS、网络、数据库） |
| `serialization/` | 序列化协议（protobuf 等） |
| `parallel-computing/` | 并行计算 |
| `service-governance/` | 服务治理（负载均衡等） |
| `data/` | 数据存储（Redis 等） |
| `Cloud/` | 云服务与基础设施 |
| `harness-engineering/` | AI Agent 工程化方法论 |
| `LeetCode/` | 算法题解（代码为主） |
| `Machine-Learning/` | ML 算法实现（代码为主） |
| `Code-Jam/` `Kick-Start/` | 竞赛题解 |

## Skill

`/write-tech-article <URL 或技术内容>` — 模仿作者风格撰写深度技术文章，多轮 review 达标后自动构建部署。详见 `.claude/commands/write-tech-article.md`。
