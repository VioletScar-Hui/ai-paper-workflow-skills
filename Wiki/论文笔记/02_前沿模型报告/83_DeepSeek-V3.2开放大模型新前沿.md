---
tags: [论文笔记, DeepSeek-V3.2, DSA, 稀疏注意力, 推理, 开源前沿, DeepSeek]
笔记层级: 参考
paper_id: "83"
filename: "83 - DeepSeek-V3.2 - Pushing the Frontier of Open Large Language Models.pdf"
authors: "DeepSeek-AI"
year: 2025
成熟度: 🏭
---

# DeepSeek-V3.2：开放大模型新前沿

📄 **原文 PDF**：[[RAW/83 - DeepSeek-V3.2 - Pushing the Frontier of Open Large Language Models.pdf]]

## PM 速判（30秒）

> **2025 年底最强开源模型，首次在 IMO/IOI 赛场拿金牌，代码能力超过 GPT-5。** DeepSeek 2025 年 12 月发布 V3.2，三大突破：① DSA（DeepSeek Sparse Attention）将注意力复杂度从 O(L²) 降为 O(Lk)；② 可扩展 RL 框架使推理能力媲美 GPT-5；③ 大规模 Agentic 任务合成流水线提升工具使用能力。V3.2-Speciale 变体在 Codeforces（3000评分）、TerminalBench 2.0（77.2%）上超过 GPT-5。

| 年份 | 成熟度 | PM 相关性 |
|------|--------|---------|
| 2025-12 | 🏭 | **最高**（代码/推理/Agent 任务的顶级开源选择）|

## 核心机制

```
三大技术突破：

1. DSA（DeepSeek Sparse Attention）
   原理：Lightning Indexer 计算 query-token 与 preceding token 的相关分数
         只对 Top-k 个高分 token 做完整 Attention
   复杂度：O(L²) → O(Lk)，k << L
   训练过程：Dense 预热（1000 steps）→ Sparse 训练
   结果：长序列推理速度显著提升，性能无退化

2. 可扩展 RL 框架
   扩大 post-training 计算量 → Thinking 模式推理能力跳跃
   V3.2-Speciale = 推理模式最大算力变体
   
3. Agentic 任务合成流水线
   自动生成大规模工具使用训练数据
   显著提升 Agent 场景泛化能力
   
训练基础：在 DeepSeek-V3.1-Terminus 上持续训练
```

## 关键数据 📅 2025-12

| 基准 | V3.2-Speciale | V3.2-Thinking | GPT-5-High | Claude-4.5-Sonnet |
|------|-------------|--------------|-----------|------------------|
| AIME 2025 | **99.2%** | 96.0% | 93.1% | 94.6% |
| HMMT 2025 Feb | 95.0% | **97.5%** | 90.2% | — |
| HLE | 42.8% | 37.7% | **54.2%** | — |
| Codeforces | **3000** | 2708 | 2537 | — |
| SWE-Verified | 80.3% | 80.2% | 84.7% | **85.4%** |
| TerminalBench 2.0 | **77.2%** | 76.2% | 73.1% | — |
| Tool-Decathlon | **38.6%** | 36.4% | — | 35.2% |

**特殊成就**：2025 年 IMO 和 IOI 金牌表现

## PM 结论

- 📅 2025 年 12 月发布，开源模型首次在复赛级数学/编程竞赛超过 GPT-5
- **DSA 是架构级突破**：不用换模型，通过持续训练给 V3 增加稀疏注意力，长序列效率大幅提升
- 适合场景：数学/代码推理、长上下文任务（V3.2 长文本比 V3.1 高 4 分）、Agent 工具使用

## 论文精读卡片

**一句话**：DeepSeek-V3.2-Speciale 在 Codeforces 达到 3000 评分（超 GPT-5 的 2537），AIME 2025 达到 99.2%，首次让开源模型在竞赛级编程和数学上全面超越顶级闭源模型。

**问题**：如何在不改变基础模型架构的前提下，通过持续训练大幅提升长序列推理效率，同时让模型在数学/代码竞赛级任务上超越 GPT-5？

**核心方法**：
- DSA（DeepSeek Sparse Attention）——Lightning Indexer 计算每个 query 与历史 token 的相关分数，只对 Top-k 个高分 token 做完整 Attention，复杂度从 O(L²) 降为 O(Lk)
- 可扩展 RL 框架——扩大 post-training 阶段计算量，通过测试时计算扩展（Speciale 变体）使推理能力跳跃
- Agentic 任务合成流水线——自动生成大规模工具使用训练数据，显著提升 Agent 场景泛化

**关键图/公式**：DSA 稀疏化流程：Lightning Indexer → Top-k token 选取 → 局部完整 Attention；Dense 预热 1000 步后切换 Sparse 训练。O(L²) → O(Lk)，k << L，长序列推理速度提升显著，性能几乎无退化。

**实验设置**：
- 规模/数据：在 DeepSeek-V3.1-Terminus 基础上持续训练，加入稀疏注意力和 Agentic 数据
- 对比：GPT-5-High、Claude-4.5-Sonnet，多个数学/代码/Agent 基准

**最强证据**：AIME 2025 = 99.2%（V3.2-Speciale，第一，超 GPT-5 的 93.1%）；Codeforces 3000（超 GPT-5 的 2537）；TerminalBench 2.0 = 77.2%（第一）；2025 年 IMO 和 IOI 金牌表现。

**最弱证据**：HLE（Humanity's Last Exam）42.8%，远低于 GPT-5 的 54.2%，说明 V3.2 在广博世界知识上有明显短板；SWE-Verified 80.3% 低于 Claude-4.5-Sonnet 的 85.4%，实际工程代码任务仍逊色；DSA 的 Top-k 参数选择对不同任务的影响未充分分析。

**可复用点**：DSA 的"持续训练添加稀疏注意力"策略——从已有 Dense 模型出发，先做 Dense 预热再切换 Sparse 训练，可以低成本升级已有模型的长上下文效率，无需从头训练。

**和哪些论文相关**：
- [[Wiki/论文笔记/09_长上下文记忆/82_DeepSeek-V4百万Token高效长上下文]] — V4 是 V3.2 架构的进一步升级，引入 CSA/HCA 替代 DSA
- [[Wiki/论文笔记/02_前沿模型报告/89_GLM-5从氛围编码到Agent工程]] — GLM-5 同样采用 DSA，是该技术在另一家公司的验证
- [[Wiki/概念/01_架构技术/稀疏注意力与长上下文效率]] — DSA 是稀疏注意力方向的最新工程实现
- [[Wiki/概念/03_推理与评测/测试时计算扩展]] — Speciale 变体通过测试时算力扩展实现推理能力跳跃
- [[Wiki/概念/02_训练方法/GRPO优化算法]] — 可扩展 RL 框架与 GRPO 类优化器的结合

**我能拿它做什么**：
- 数学竞赛/AIME 类任务首选 V3.2-Speciale（99.2%，开源最强）
- 代码竞赛（Codeforces 3000）和终端 Agent 任务（TerminalBench 77.2%）使用 V3.2
- 将 DSA 的"持续训练添加稀疏注意力"方法用于改造已有 Dense 长上下文模型

**3天后要回忆的问题**：
1. DSA（DeepSeek Sparse Attention）的核心实现步骤是什么？
2. V3.2 的 AIME 2025 得分是多少？与 GPT-5 的差距？
3. V3.2-Speciale 和 V3.2-Thinking 的区别是什么？
4. DSA 从 Dense 预热切换到 Sparse 训练的原因是什么？
5. V3.2 在哪些任务上依然弱于 GPT-5 和 Claude？

## 原子概念索引
- [[Wiki/概念/01_架构技术/稀疏注意力与长上下文效率]] — DSA 是该方向的代表性工程实现
- [[Wiki/概念/03_推理与评测/测试时计算扩展]] — V3.2-Speciale 通过测试时计算扩展实现竞赛级推理能力
