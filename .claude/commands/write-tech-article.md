# write-tech-article

为 Hugo 博客撰写一篇深度技术文章，模仿作者写作风格，经过多轮自我 review 达到质量标准后自动构建部署。

## 使用方式

```
/write-tech-article <技术分析对象>
```

`技术分析对象` 可以是：URL、代码仓库地址、技术文档链接、或直接粘贴的技术内容。

---

## 阶段一：风格提取

读取以下三篇文章，提取作者写作风格（**不向用户输出提取过程**）：

- `content/posts/ai-agent/claude-code-architecture.md`
- `content/posts/ai-agent/google-adk-architecture.md`
- `content/posts/ai-agent/langchain-langgraph-architecture.md`

提取维度：

| 维度 | 关键特征 |
|------|----------|
| 行文节奏 | 长短句交替，段落密度高，信息推进从问题→机制→评价→延伸 |
| 叙述视角 | 避免第一人称，用"这意味着""这表明"等因果推断替代"我认为" |
| 概念引入 | 正文中先提问/设悬念再展开，不在小标题里问完就在正文直接给答案 |
| 类比来源 | 分布式系统、OS、数据库、编译器等 CS 基础领域，自然融入不生硬 |
| 技术深度 | ASCII 架构图 + 代码/伪代码/JSON 示例，引用具体文件名、行号、字段名 |
| 结构偏好 | `## 1`/`## 2` 数字编号大节，`### 1.1` 三级标题，散文为主不堆列表 |
| 禁忌套话 | 禁止"这正是……的体现""总的来说""综上所述""值得注意的是" |
| 权衡讨论 | 每章节内有明确标注为作者视角的评价段，不以客观陈述替代主观推断 |
| 收尾习惯 | 最后一章用立场性观点收尾，呼应开篇问题，不写"总结"章节 |

---

## 阶段二：技术内容分析

用 WebFetch 抓取技术分析对象，理解：
1. 整体设计意图与核心问题域
2. 关键抽象层与子系统划分
3. 值得深入的设计选择与权衡

---

## 阶段三：文章写作

### 文件路径规则
- 目录：`content/posts/<主题英文短名>/`
- 文件名：`<主题英文短名>-architecture.md`（或其他合适名称）
- Front matter：
  ```yaml
  ---
  title: "<中文标题>"
  date: <当前日期>T10:00:00+08:00
  draft: false
  categories: ["<分类>"]
  ---
  ```

### 写作规范
1. 开篇用 **30-50 字的具体失败案例或场景** 建立阅读动机，不以抽象概念开篇
2. 正文推进顺序：提出问题/动机 → 分析设计选择 → 揭示底层机制 → 给出评价或延伸
3. 每个大章节开头在**正文中**提问或设悬念，不在标题里问完就直接给答案
4. 关键数据结构/流程用 ASCII 图或代码块辅助，每 3 个连续文字段落之间至少有一个视觉元素
5. 章节数量控制在 **5-7 章**，优先合并相似子系统，拒绝并列堆砌
6. 文章长度 **3000-6000 字**，与技术对象复杂度匹配

---

## 阶段四：多轮自我 Review（迭代直到全部 ≥ 9 分）

每轮写完后，**并行**启动三个 subagent 分别从三个维度打分：

### Review Agent 1 — 风格还原度
```
参考：content/posts/ai-agent/google-adk-architecture.md
评估维度（各 1-10 分）：
1. 问题驱动节奏（正文提问 vs 直接给答案）
2. 类比自然度（来源领域、融入方式）
3. 权衡讨论处理方式
4. 过渡句与前后引用
5. 套话禁忌（"这正是……的体现"等）
输出：整体相似度（1-10）+ 最需改进的 1-2 处
```

### Review Agent 2 — 章节结构逻辑
```
评估维度（1-10 分）：
1. 逻辑递进（因果链 vs 并列堆砌）
2. 概念重叠（哪些章节/段落重复）
3. 信息量分布（是否均衡）
4. 结构完整性（是否遗漏重要维度）
输出：结构评分（1-10）+ 最需改进的 1-2 处
```

### Review Agent 3 — 可读性
```
评估维度（1-10 分）：
1. 开篇吸引力（具体案例 vs 抽象陈述）
2. 概念密度节奏（连续纯文字段落数量）
3. 图表与文字搭配（视觉缓冲是否到位）
4. 冗余表达（重复内容）
5. 结尾力度（首尾呼应是否完成）
输出：可读性评分（1-10）+ 最需改进的 1-2 处
```

### 迭代规则
- **三项均 ≥ 9 分**：进入阶段五
- **任意一项 < 9 分**：根据 review 反馈修改，重新启动三路 review，直到达标
- 每轮修改聚焦在 review 指出的最高优先级问题，不做无谓的全文重写

---

## 阶段五：构建与部署

### Hugo 构建
```bash
hugo --minify
```

确认输出页数合理（对比上一次构建）。

### Git 提交与推送（两个仓库）

**SSH 注意事项**：
- 当前环境 VPN 可能劫持 DNS，导致 `github.com` 解析到内网 IP
- 绕过方案：使用 `ssh.github.com:443` 通道
- 个人账号 `chr1sc2y` 使用 key：`~/.ssh/id_ed25519`
- SSH config 已配置 `Host ssh.github.com` 条目（port 443 + id_ed25519）

**推送顺序**：

1. **public submodule**（`prov1dence.github.io`）：
   ```bash
   cd public
   # 确认 remote 使用 ssh.github.com
   git remote set-url origin git@ssh.github.com:chr1sc2y/prov1dence.github.io.git
   git add .
   git commit -m "update: <文章主题简述>"
   git push origin master
   ```

2. **主仓库**（`hugo-site`）：
   ```bash
   # 确认 remote 使用 ssh.github.com
   git remote set-url origin git@ssh.github.com:chr1sc2y/hugo-site.git
   git add content/posts/<新文章路径> public
   git commit -m "update: <文章主题简述>"
   git push origin master
   ```

如果 SSH 连接仍失败，使用显式参数：
```bash
GIT_SSH_COMMAND="ssh -p 443 -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes" git push origin master
```

---

## 快速参考：作者风格核心特征

```
✓ 开篇：具体失败案例（30-50字）→ 问题归因 → 本文解析对象
✓ 正文：先问题/悬念 → 机制拆解 → 评价/权衡
✓ 图表：ASCII 架构图（章节总览）+ 代码块（关键实现）
✓ 类比：OS/数据库/分布式系统，命名模式并解释来源
✓ 收尾：立场性判断，回扣开篇，不写"总结"章节
✗ 禁止：列表替代散文、套话、无评价的纯描述、"总的来说"
```
