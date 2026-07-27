---
tags: [论文笔记, PaLM, 扩展律, Pathways, 基础论文, Google]
笔记层级: 标准
paper_id: "69"
filename: "69 - PaLM - Scaling Language Modeling with Pathways.pdf"
authors: "Aakanksha Chowdhery et al. (Google)"
year: 2022
成熟度: ✅
---

# PaLM：用 Pathways 扩展语言建模

📄 **原文 PDF**：[[RAW/69 - PaLM - Scaling Language Modeling with Pathways.pdf]]

## PM 速判（10秒）

> **经典大模型论文，背景知识。** Google 2022 年的 5400 亿参数语言模型，展示了 Pathways 分布式训练的高效性，以及在多步推理、代码生成等任务上的新兴能力（Emergent Abilities）。PaLM 2 是 Gemini 的前身。

| 年份 | 成熟度 | PM 相关性 |
|------|--------|---------|
| 2022 | ✅ | 背景知识 |

## 论文精读卡片

**一句话**：PaLM（5400 亿参数）用 Pathways 跨 6144 个 TPU 芯片训练，在 Big-Bench 150+ 任务中的 29% 上首次超越人类平均水平，同时首次记录了在代码、数学、多语言理解上的"涌现能力"（Emergent Abilities）。

**问题**：此前最大的 LLM（GPT-3 175B、Megatron 530B）在复杂推理和多步任务上仍有明显能力瓶颈；需要验证继续扩展规模能否突破这些瓶颈，以及分布式训练基础设施能否支撑 500B+ 量级。

**核心方法**：
- Pathways 分布式系统：跨数据中心的 6144 个 TPU v4 芯片协同训练，通过 DCN（数据中心网络）连接，实现接近线性的扩展效率
- 训练数据：780B token 高质量混合数据（网页、书籍、GitHub、维基百科、对话、多语言），比 GPT-3 数据量大得多
- 架构改进：SwiGLU 激活函数、RoPE 位置编码、Multi-Query Attention——均为后来 LLaMA 等采用

**关键图/公式**：Pathways 训练效率：PaLM 540B 在 6144 个 TPU 上实现约 46.2% 的硬件 FLOP 利用率（MFU），远高于当时同规模模型的约 30%，证明基础设施突破。

**实验设置**：
- 规模/数据：8B/62B/540B 三个规模；780B token；BIG-Bench（150+ 任务）、GSM8K、MATH、HumanEval 等
- 对比：GPT-3（175B）、Chinchilla（70B）、Codex、GLaM；zero-shot/few-shot 对比

**最强证据**：PaLM 540B + CoT 在 GSM8K 达 58.1%（当时 SOTA）；BIG-Bench 上 29% 的任务首次超越人类均值；多语言翻译（WMT 等）zero-shot 超越专项微调模型。

**最弱证据**：PaLM 未开源，无法独立复现；涌现能力（Emergent Abilities）的观察依赖于特定评测集选择，后续研究（Schaeffer 2023 等）质疑其是否是测量方法的幻觉；540B 参数的训练计算量远超 Chinchilla 最优比例（Compute-optimal scaling）。

**可复用点**：PaLM 的架构选择（SwiGLU、RoPE、MQA）被 LLaMA 系列直接继承，是今天开源 LLM 标准架构的来源；BIG-Bench 可作为全面能力评测的参考基准；"涌现能力"框架是解释大模型能力跃迁的标准叙事。

**和哪些论文相关**：
- [[Wiki/论文笔记/05_推理思维链/68_ChainOfThought思维链提示激发大模型推理]] — PaLM + CoT 是 GSM8K SOTA 的组合，两文共同奠定推理能力研究基础
- [[Wiki/论文笔记/01_LLM基础架构/75_LLaMA高效开放的基础语言模型]] — LLaMA 直接继承 PaLM 的架构选择（SwiGLU、RoPE）并开源
- [[Wiki/论文笔记/01_LLM基础架构/66_GLaM专家混合高效扩展语言模型]] — GLaM（MoE）vs PaLM（密集）：同期 Google 的两条大规模扩展路线
- [[Wiki/概念/03_推理与评测/思维链与推理模型]] — PaLM 是 CoT 推理能力最强展示平台，两者共同定义了"推理能力涌现"

**我能拿它做什么**：
- 引用 PaLM 论文向团队解释"涌现能力"的概念：为什么 100B 以上的模型质变而非量变
- 在评估 LLM 架构讨论时，用 PaLM → LLaMA 的传承说明 SwiGLU/RoPE/MQA 组合的经过验证的价值
- 使用 BIG-Bench 子集作为内部能力评测的设计参考

**3天后要回忆的问题**：
1. PaLM 有多少参数？用了多少个 TPU 芯片训练？
2. PaLM 在 BIG-Bench 上的关键结论是什么（具体数字）？
3. PaLM 引入了哪三个架构改进？它们后来被哪个模型继承？
4. "涌现能力"（Emergent Abilities）的概念是什么——为什么有研究者质疑它？
5. PaLM 与 Chinchilla Optimal 扩展定律的关系是什么——PaLM 540B 是否"浪费"了计算？

## 原子概念索引
- [[Wiki/概念/03_推理与评测/思维链与推理模型]] — PaLM + CoT 是涌现推理能力的核心实验平台
- [[Wiki/概念/01_架构技术/MoE混合专家架构]] — PaLM（密集）vs GLaM（MoE）代表同期 Google 两条扩展路线，理解 MoE 价值需要与 PaLM 对比
