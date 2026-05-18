# write-tech-article

为 Hugo 博客撰写一篇深度技术文章，模仿作者写作风格，经过多轮自我 review 达到质量标准后自动构建部署。

## 使用方式

```
/write-tech-article <技术分析对象>
```

`技术分析对象` 由用户在调用时提供，可以是 URL、代码仓库地址、技术文档链接，或直接粘贴的技术内容。

---

## 阶段一：风格提取（读取全部散文式技术文章）

**不向用户输出提取过程**，内部建立风格模型。

逐篇读取以下所有文章（按主题分组，覆盖作者不同时期和不同领域的写作）：

### AI Agent 系列（最新风格，权重最高）
- `content/posts/ai-agent/claude-code-architecture.md`
- `content/posts/ai-agent/google-adk-architecture.md`
- `content/posts/ai-agent/langchain-langgraph-architecture.md`
- `content/posts/harness-engineering/harness-engineering-architecture.md`

### Python 源码系列（散文叙述风格参考）
- `content/posts/python/source-code-1.md`
- `content/posts/python/source-code-2.md`
- `content/posts/python/source-code-3-list-and-dict.md`
- `content/posts/python/python-source-code-interpreter.md`
- `content/posts/python/python-source-code-coroutine.md`

### 序列化与协议
- `content/posts/serialization/protocol-buffer.md`
- `content/posts/serialization/protocol-buffer-en.md`

### 系统与架构
- `content/posts/parallel-computing/parallel-computing.md`
- `content/posts/cpp/concurrency/introduction-to-concurrency.md`
- `content/posts/cpp/closure-and-anonymous-function.md`
- `content/posts/service-governance/load-balancing.md`
- `content/posts/computer-science/Database.md`
- `content/posts/network/rpc.md`

### 云与基础设施
- `content/posts/Cloud/intro-to-Prometheus.md`
- `content/posts/Disaster Recovery Architecture on AWS/disaster-recovery-architecture-on-AWS 9b3dc135d29c47acbfe19084c57035d5.md`
- `content/posts/data/Redis 持久化机制 RDB 和 AOF 72a88c70207a4518824d1fe4be7ebb9e.md`

### C++ 深度技术
- `content/posts/cpp/basics/effective-cpp.md`
- `content/posts/cpp/smart pointer/C++-Smart-Pointer-1.md`
- `content/posts/cpp/smart pointer/C++-Smart-Pointer-2.md`

**注意**：以下类别不读取，风格学习价值低（题解/手册/算法实现）：
LeetCode/、Code-Jam/、Kick-Start/、compilation/、Machine-Learning/、LeetCode-Archiver/

### 风格提取维度

读取时重点归纳以下维度，AI Agent 系列的权重高于其他系列：

| 维度 | 需要归纳的内容 |
|------|----------------|
| 行文节奏 | 句子长短搭配规律、段落密度、信息推进节奏 |
| 叙述视角 | 是否使用第一人称、个人判断的标识方式 |
| 概念引入 | 先问题/悬念再拆解 vs 先结论再论证，节奏分布 |
| 类比来源 | 类比的领域、频率、融入方式是否自然 |
| 技术深度 | 代码块/ASCII图/伪代码的使用时机与密度 |
| 结构偏好 | 标题层级、章节数量、小节命名风格 |
| 禁忌表达 | 归纳作者从不使用的套话和句式 |
| 权衡风格 | 设计权衡的讨论方式与出现位置 |
| 收尾习惯 | 文章最后一章的惯用模式 |

---

## 阶段二：技术内容分析

用 WebFetch 抓取用户提供的技术分析对象，理解：
1. 整体设计意图与核心问题域
2. 关键抽象层与子系统划分
3. 值得深入的设计选择与权衡
4. 与现有技术方案的对比与分歧

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

### 写作规范（基于风格提取结果执行）

1. **开篇**：用 30-50 字的具体失败案例或场景建立阅读动机，不以抽象概念开篇
2. **推进节奏**：提出问题/动机 → 分析设计选择 → 揭示底层机制 → 给出评价或延伸
3. **悬念设置**：每个大章节在**正文中**先提问或设悬念，不在标题里问完就在正文直接给答案
4. **视觉节奏**：每 3 个连续文字段落之间至少穿插一个视觉元素（ASCII 图/代码块/表格）
5. **章节控制**：5-7 章，优先合并相似子系统，拒绝并列堆砌
6. **长度**：3000-6000 字，与技术对象复杂度匹配
7. **个人评价**：文章中的判断和评价明确标识为作者视角，不以客观陈述替代主观推断

---

## 阶段四：多轮自我 Review（迭代直到全部 ≥ 9 分）

每轮写完后，**并行**启动三个 subagent（`subagent_type: Explore`，`run_in_background: true`）：

### Review Agent 1 — 风格还原度
参考文件：`content/posts/ai-agent/google-adk-architecture.md`

评估维度（各 1-10 分）：
1. 问题驱动节奏（正文提问 vs 直接给答案）
2. 类比自然度（来源领域、融入方式）
3. 权衡讨论处理方式
4. 过渡句与前后引用
5. 套话禁忌

输出：整体相似度（1-10）+ 最需改进的 1-2 处（附原文对比）。400 字以内。

### Review Agent 2 — 章节结构逻辑
评估维度（1-10 分）：
1. 逻辑递进（因果链 vs 并列堆砌）
2. 概念重叠（具体章节/段落）
3. 信息量分布均衡性
4. 主题完整性（是否遗漏重要维度）

输出：结构评分（1-10）+ 最需改进的 1-2 处。400 字以内。

### Review Agent 3 — 可读性
评估维度（1-10 分）：
1. 开篇吸引力（具体案例 vs 抽象陈述）
2. 概念密度节奏（连续纯文字段落区域）
3. 图表与文字搭配（视觉缓冲是否到位）
4. 冗余表达（重复内容）
5. 结尾力度（首尾呼应）

输出：可读性评分（1-10）+ 最需改进的 1-2 处。400 字以内。

### 迭代规则
- **三项均 ≥ 9 分**：进入阶段五
- **任意一项 < 9 分**：根据 review 反馈做针对性修改，重新并行启动三路 review
- 每轮修改聚焦 review 指出的最高优先级问题，不做无谓的全文重写
- 向用户展示每轮的评分变化

---

## 阶段五：构建与部署

### Hugo 构建
```bash
hugo --minify
```

确认输出页数合理（对比上一次）。

### SSH 说明

当前环境 VPN 可能劫持 DNS，导致 `github.com` 被解析到内网 IP，SSH 22 端口不通。
绕过方案：走 `ssh.github.com:443` 通道，已在 `~/.ssh/config` 中配置。

个人账号 `chr1sc2y`：
- key：`~/.ssh/id_ed25519`
- SSH config host：`ssh.github.com`（port 443）

### 提交推送顺序

**1. public submodule**（`prov1dence.github.io`）：
```bash
cd public
git remote set-url origin git@ssh.github.com:chr1sc2y/prov1dence.github.io.git
git add .
git commit -m "update: <文章主题简述>"
git push origin master
```

**2. 主仓库**（`hugo-site`）：
```bash
git remote set-url origin git@ssh.github.com:chr1sc2y/hugo-site.git
git add content/posts/<新文章路径> public
git commit -m "update: <文章主题简述>"
git push origin master
```

**SSH fallback**（config 不生效时）：
```bash
GIT_SSH_COMMAND="ssh -p 443 -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes" git push origin master
```

---

## 快速参考：作者风格核心特征

```
✓ 开篇：具体失败案例/场景（30-50字）→ 归因 → 本文解析对象
✓ 推进：先悬念/问题 → 机制拆解 → 评价/权衡
✓ 图表：ASCII 架构图（整体总览）+ 代码块（关键实现细节）
✓ 类比：OS/数据库/分布式/编译器等 CS 基础，命名模式后解释来源
✓ 收尾：立场性判断，回扣开篇，不写"总结"章节
✗ 禁止：列表替代散文、"这正是……的体现"、"总的来说"、"综上所述"
✗ 禁止：无评价的纯描述、连续 3 段以上纯文字无视觉元素
```
