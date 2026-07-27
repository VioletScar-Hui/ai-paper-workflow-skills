---
tags: [论文笔记, 技能优化, 自进化Agent, 文本优化, Microsoft, 上交]
笔记层级: 标准
paper_id: "41"
filename: "41 - SkillOpt - Executive Strategy for Self-Evolving Agent Skills.pdf"
authors: "Yifan Yang et al. (Microsoft / SJTU / Tongji / Fudan)"
year: 2026
成熟度: 🔧
---

# SkillOpt：Agent 技能自进化的文本空间优化器

📄 **原文 PDF**：[[RAW/41 - SkillOpt - Executive Strategy for Self-Evolving Agent Skills.pdf]]

## PM 速判（3 分钟）

> **这篇论文给 PM 的一句话**：把深度学习的优化纪律（梯度 → 有界编辑、学习率 → 编辑预算、验证集门控 → held-out gate）迁移到技能文档优化——SkillOpt 用独立的优化模型把轨迹转化为对技能文档的"增/删/改"编辑，且只有验证集提升的编辑才被接受；在 52 个（模型×基准×执行框架）评测格中**全部第一或并列第一**，GPT-5.5 平均提升 **+23.5 点**。

| 项目 | 评估 |
|------|------|
| **核心创新** | 文本空间优化器 + 有界 add/delete/replace 编辑 + held-out gate |
| **GPT-5.5 提升** | direct chat +23.5 pp；Codex 框架 +24.8 pp；Claude Code +19.1 pp |
| **迁移性** | GPT-5.4 训练的技能 → 更小 GPT 变体 / Claude Code 均有效（最高 +59.7 pp）|
| **部署成本** | 零推理时模型调用增量（技能是文档，不是模型权重）|
| **成熟度** | 🔧 开源实验，Microsoft / SJTU，2026 |

---

## 核心机制

```
类比：把 DL 优化纪律映射到技能优化

  权重 → 技能文档 S
  梯度方向 → 轨迹驱动的编辑方向
  学习率 → 编辑预算 L（每步允许的最大编辑数）
  验证集门控 → held-out 选择门（只接受提升验证集分数的候选技能）
  正则化 / 动量 → rejected-edit buffer（拒绝的编辑成为下一轮的负反馈）
  epoch + slow update → epoch 级慢更新（跨 epoch 纵向比较，防止漂移）

具体流程（每步）：
  1. 目标 Agent 用当前技能执行一批 rollout
  2. 优化模型（独立 frontier 模型）将轨迹分 minibatch 反思（成功 vs 失败）
  3. 提出 add/delete/replace 编辑候选（受 L_t 预算约束）
  4. 在验证集上评估：只有提升 val 分数的编辑才被接受
  5. 被拒绝的编辑存入 rejected-edit buffer → 下次迭代作为负反馈
  6. epoch 结束：slow/meta 更新（从纵向视角修正漂移和回退）
  7. 导出最佳 best_skill.md → 可复用文档
```

## 关键数据（带时间锚）📅 2026

| 基准（GPT-5.5 direct chat）| 无技能 | SkillOpt | 提升 |
|--------------------------|--------|----------|------|
| SpreadsheetBench | 41.8 | 80.7 | +38.9 |
| OfficeQA | 33.1 | 72.1 | +39.0 |
| LiveMathematician | 37.6 | 66.9 | +29.3 |
| SearchQA | 77.7 | 87.3 | +9.6 |
| ALFWorld | 83.6 | 95.5 | +11.9 |
| 平均 | — | — | **+23.5** |

---

## 对 AI PM 的启示

| 场景 | 洞察 |
|------|------|
| **技能库建设** | SkillOpt 的产出是可审计的 Markdown 技能文档，可以部署成技能库，按需检索 |
| **模型无关性** | 技能文档可以跨模型/框架迁移（GPT → Claude，Codex → Claude Code）；一次优化多处复用 |
| **vs 人工编写技能** | SkillOpt 在所有评测格上超过人工编写技能 —— 自动化优化质量已超过人工 |
| **与 Self-Harness 对比** | Self-Harness（Paper 22）改进整个 harness；SkillOpt 专注于技能文档优化，粒度更细，控制更精确 |

---

## 权威学习资源

- 📄 论文：Yang et al.（Microsoft / 上交大 / 同济 / 复旦），2026

## 论文精读卡片

**一句话**：将深度学习优化纪律（梯度/学习率/验证门控）迁移到技能文档文本空间，GPT-5.5 平均提升 +23.5 点，在所有 52 个评测格中全部第一，且优化产出的技能文档可零成本跨模型迁移。

**问题**：Agent 技能文档（skill docs）通常靠人工编写或简单反思更新，缺乏类似梯度下降的系统性优化机制——如何在无法对文本做微分的情况下实现高质量的技能文档自动优化？

**核心方法**：
- 用独立优化模型（frontier LLM）将执行轨迹分 minibatch 反思，提出 add/delete/replace 三类有界编辑（受预算 L 约束）
- Held-out gate：只有通过验证集评估分数提升的编辑才被接受；被拒绝的编辑存入 rejected-edit buffer 作为下轮负反馈
- Epoch 级 slow update：跨 epoch 纵向比较，防止技能文档漂移和回退

**关键图/公式**：优化类比表：权重→技能文档，梯度→轨迹驱动编辑方向，学习率→编辑预算 L，验证集门控→held-out gate，正则化→rejected-edit buffer。

**实验设置**：
- 规模/数据：5 个基准（SpreadsheetBench/OfficeQA/LiveMathematician/SearchQA/ALFWorld），模型覆盖 GPT-5.5/GPT-5.4/Claude Code，52 个评测格（模型×基准×框架）
- 对比：无技能基线、人工编写技能、ExpeL 等现有方法

**最强证据**：GPT-5.5 在 SpreadsheetBench +38.9、OfficeQA +39.0、LiveMathematician +29.3，平均 +23.5；GPT-5.4 优化的技能迁移到更小 GPT 变体/Claude Code 最高 +59.7 pp；52 个评测格全部第一。

**最弱证据**：实验局限于特定任务类型（办公/数学/搜索），未在开放式创意或多步骤复杂规划任务上验证；优化器本身使用的是 frontier 模型，小型部署场景（如本地 7B 模型做优化器）未测试。

**可复用点**：Held-out gate（验证集门控）+ rejected-edit buffer（负反馈缓冲区）的两件套可直接用于任何基于反思的技能/提示优化流水线；"技能=可审计 Markdown 文档"的架构决策使得优化产出人可读、可版本控制。

**和哪些论文相关**：
- [[Wiki/概念/04_Agent框架/Harness设计模式]] — SkillOpt 产出的技能文档是 harness 的核心组件
- [[Wiki/概念/04_Agent框架/动态智能体脚手架]] — 技能文档自进化是动态脚手架能力提升的关键机制
- [[Wiki/概念/04_Agent框架/学习型编排器]] — SkillOpt 将优化内化到技能层，与学习型编排器思路互补

**我能拿它做什么**：
- 对现有 Agent 产品的技能文档引入 held-out gate：只有在验证集提升的技能更新才部署
- 用 SkillOpt 的跨模型迁移特性：在大模型上优化技能文档，部署到小/低成本模型
- 把技能文档纳入版本控制（Git），每次优化产出一个带评分的新版本，建立技能文档发布流程

**3天后要回忆的问题**：
1. SkillOpt 中的 held-out gate 起什么作用？没有它会发生什么？
2. rejected-edit buffer 对应 DL 中的哪个概念？它如何改进下一轮优化？
3. 为什么技能文档能跨模型迁移？这个迁移能力的边界在哪里？
4. SkillOpt vs 人工编写技能：哪类任务上自动优化已经超过人工？
5. 如果你要把 SkillOpt 思路应用到你的产品，第一步你会优化哪个技能文档？

## 原子概念索引

- [[Wiki/概念/04_Agent框架/Harness设计模式]] — SkillOpt 产出的技能文档是 harness 的核心可优化组件
- [[Wiki/概念/04_Agent框架/动态智能体脚手架]] — 技能自进化是动态脚手架持续改进的核心机制
