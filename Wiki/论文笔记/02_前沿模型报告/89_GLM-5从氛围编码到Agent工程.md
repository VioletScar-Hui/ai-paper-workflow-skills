---
tags: [论文笔记, GLM-5, MoE, DSA, Agent工程, 开源前沿, ZhipuAI]
笔记层级: 参考
paper_id: "89"
filename: "89 - GLM-5 - from Vibe Coding to Agentic Engineering.pdf"
authors: "GLM-5 Team (ZhipuAI & Tsinghua)"
year: 2026
成熟度: 🏭
---

# GLM-5：从氛围编码到 Agent 工程

📄 **原文 PDF**：[[RAW/89 - GLM-5 - from Vibe Coding to Agentic Engineering.pdf]]

## PM 速判（30秒）

> **2026 年初开源权重中第一个在 Intelligence Index 达到 50 分的模型，整体媲美 Claude Opus 4.5 和 GPT-5.2。** ZhipuAI 发布 GLM-5（744B 参数，40B 激活），引入 DSA+MLA-256+异步 RL，基于 28.5T tokens 训练；在 LMArena 文本和代码双榜单位列开源第一；Vending Bench 2（长视野 Agent 任务）成本仅 $1,034，是其他模型的 1/3-1/5；完整适配 7 个国内 GPU 平台。

| 年份 | 成熟度 | PM 相关性 |
|------|--------|---------|
| 2026-02 | 🏭 | **最高**（国产开源最强 Agent 模型，国内 GPU 部署首选）|

## 核心机制

```
架构升级（相比 GLM-4.5）：
  规模：355B(32B active) → 744B(40B active)
  注意力：DSA（DeepSeek Sparse Attention）大幅降低推理成本
  KV 压缩：MLA-256（调整注意力头维度 128→256，头数减少 1/3，保持效率）
  MTP：多 Token 预测 + 参数共享（GLM-5 accept length > DeepSeek-V3.2）

训练流水线：
  预训练：27T tokens（代码和推理优先）
  中间训练：4K → 200K 上下文扩展（专注 agentic 数据）
  后训练（串行 RL）：
    ① Reasoning RL（基础推理）
    ② Agentic RL（长视野交互）
    ③ General RL（通用对齐）
  贯穿全程：On-Policy Cross-Stage Distillation（防止灾难性遗忘）

异步 Agent RL 算法：
  将生成与训练解耦（基于"slime"框架）
  大规模探索 Agent 轨迹，无同步瓶颈
  支持长视野、动态环境中的连续学习

国产 GPU 支持：
  华为昇腾/摩尔线程/海光/寒武纪/昆仑芯/MetaX/壁仞
```

## 关键数据 📅 2026-02

| 指标 | GLM-5 | Claude Opus 4.5 | GPT-5.2(xhigh) | Gemini 3 Pro |
|------|-------|----------------|----------------|--------------|
| Intelligence Index v4.0 | **50** | 竞争 | 竞争 | 低 |
| Vending Bench 2 成本 | **$1,034** | $4,967 | $5,478 | $4,432 |
| LMArena 文本/代码 | **开源#1** | — | — | — |
| 对 GLM-4.7 提升 | **+20%** | — | — | — |

SWE-bench Verified、HLE、Terminal-Bench 2.0 等指标整体媲美 Claude Opus 4.5

## PM 结论

- 📅 2026 年 2 月，GLM-5 是开源 Agent 模型新天花板
- **Vending Bench 2 成本优势**：长视野任务只需竞品 1/5 成本（$1034 vs $4000+），对 AI 产品成本核算有巨大意义
- **国产 GPU 全栈适配**是核心差异化：国内企业合规部署 AI Agent 的最优解
- 训练范式创新：串行 RL（推理→Agent→通用）+ 异步解耦 = Agent 能力边际递增不衰减

## PM 决策框架

| 场景 | 推荐理由 |
|------|---------|
| 需要开源 Agent 模型 | ✅ 媲美 Claude Opus 4.5，开源可控 |
| 国内 GPU 部署（合规） | ✅ 7 大国产平台全适配 |
| 长视野复杂任务（成本敏感）| ✅ Vending Bench 成本最低 |
| 代码生成/工程任务 | ✅ LMArena Code #1 开源 |

## 论文精读卡片

**一句话**：GLM-5（744B/40B 激活）在 Intelligence Index 首达 50 分，Vending Bench 2 长视野 Agent 任务成本仅 $1,034（竞品 $4,000-5,500），同时完整适配 7 个国产 GPU 平台。

**问题**：现有顶级模型（Claude Opus 4.5、GPT-5.2）在复杂 Agent 任务上成本极高，且无法在国内 GPU（昇腾/摩尔线程等）上部署，如何构建一个同时满足顶级能力、低 Agent 成本和国产硬件适配三个维度的开源模型？

**核心方法**：
- DSA + MLA-256 架构——DeepSeek 稀疏注意力 + 调整 KV 头维度（128→256，头数减少 1/3），降低推理成本同时保持表达能力
- 串行 RL 三阶段后训练——① Reasoning RL（基础推理能力）→ ② Agentic RL（长视野交互能力）→ ③ General RL（通用对齐），加 On-Policy Cross-Stage Distillation 防止遗忘
- 异步 Agent RL（slime 框架）——生成与训练解耦，大规模并行探索 Agent 轨迹，消除同步瓶颈

**关键图/公式**：Vending Bench 2 成本对比：GLM-5 $1,034 vs Claude Opus 4.5 $4,967 vs GPT-5.2 $5,478 vs Gemini 3 Pro $4,432。同等任务完成质量下，GLM-5 的长视野 Agent 成本约为竞品的 1/4-1/5，对 AI 产品毛利率影响巨大。

**实验设置**：
- 规模/数据：744B 参数（40B 激活），28.5T tokens 预训练（代码和推理优先）+ 4K→200K 上下文扩展中间训练 + 三阶段串行 RL 后训练
- 对比：Claude Opus 4.5、GPT-5.2（xhigh）、Gemini 3 Pro，Intelligence Index v4.0、Vending Bench 2、LMArena

**最强证据**：Intelligence Index v4.0 首达 50 分（开源第一）；Vending Bench 2 成本 $1,034（最低，竞品 4-5x）；LMArena 文本和代码双榜开源第一；对 GLM-4.7 整体提升 +20%。

**最弱证据**：Intelligence Index 是综合评分，具体构成未完全公开；对比竞品时采用 GLM-5 的"最高配置"（xhigh），而竞品可能有更低成本版本未纳入对比；国产 GPU 适配的性能损失（vs A100/H100）未量化。

**可复用点**：串行 RL 三阶段范式（推理→Agent→通用）+ On-Policy Cross-Stage Distillation——解决了多阶段 RL 中的"灾难性遗忘"问题，可推广至任何需要按难度递增训练 Agent 能力的场景。

**和哪些论文相关**：
- [[Wiki/论文笔记/02_前沿模型报告/83_DeepSeek-V3.2开放大模型新前沿]] — GLM-5 采用 DSA 与 V3.2 同源，但在 Agent RL 框架上有独特创新
- [[Wiki/论文笔记/02_前沿模型报告/87_GLM-5V-Turbo多模态Agent原生基础模型]] — GLM-5 的多模态 Agent 扩展版
- [[Wiki/概念/01_架构技术/稀疏注意力与长上下文效率]] — DSA + MLA-256 是该方向的最新实现
- [[Wiki/概念/02_训练方法/GRPO优化算法]] — 异步 Agent RL 框架与 GRPO 类优化算法的关系
- [[Wiki/概念/04_Agent框架/动态智能体脚手架]] — slime 框架异步解耦是 Agent 训练基础设施的创新

**我能拿它做什么**：
- 国内 GPU 合规部署场景：GLM-5 是唯一适配 7 大国产 GPU 平台且能力媲美 Claude Opus 的选择
- 长视野复杂 Agent 任务：Vending Bench 2 成本优势（1/4-1/5）可直接转化为产品毛利率提升
- 构建多阶段 Agent RL 训练时，参考串行 RL + On-Policy Distillation 的灾难性遗忘解决方案

**3天后要回忆的问题**：
1. GLM-5 的 Vending Bench 2 成本是多少？与 Claude Opus 4.5 相比？
2. 串行 RL 三阶段分别是什么？为什么要串行而不是并行？
3. MLA-256 相对于标准 MLA 做了什么调整？目的是什么？
4. 异步 Agent RL（slime 框架）解决了什么具体问题？
5. GLM-5 适配哪 7 个国产 GPU 平台？

## 原子概念索引
- [[Wiki/概念/01_架构技术/稀疏注意力与长上下文效率]] — GLM-5 的 DSA + MLA-256 是稀疏注意力在超大 MoE 模型上的工程实践
- [[Wiki/概念/01_架构技术/MoE混合专家架构]] — GLM-5（744B/40B 激活）的 MoE 设计
- [[Wiki/概念/02_训练方法/GRPO优化算法]] — 异步 Agent RL 训练框架的算法基础
- [[Wiki/概念/01_架构技术/MTP多Token预测]] — GLM-5 使用 MTP 多 Token 预测 + 参数共享，accept length 超过 DeepSeek-V3.2
- [[Wiki/概念/05_记忆与检索/上下文压缩与KV缓存]] — DSA + MLA-256 的组合架构体现了上下文压缩技术在超大 MoE 模型中的工程集成实践
