---
tags: [论文笔记, MoA, FFN, 激活函数, 表达力, 架构创新, ByteDance]
笔记层级: 标准
paper_id: "95"
filename: "95 - More Expressive Feedforward Layers - Part I. Token-Adaptive Mixing of Activations.pdf"
authors: "Mingze Wang, Jinbo Wang, Yikuan Xia, Kai Shen, Shu Zhong (ByteDance Seed & Peking Univ)"
year: 2025
成熟度: 🧪
---

# 更具表达力的 FFN 层：Token 自适应激活混合（MoA）

📄 **原文 PDF**：[[RAW/95 - More Expressive Feedforward Layers - Part I. Token-Adaptive Mixing of Activations.pdf]]

## PM 速判（30秒）

> **不增加参数、不路由到不同专家，但让每个 token 获得不同的激活函数。** 现有 LLM 的 FFN 层对所有 token 使用相同的激活函数（SwiGLU 等）。ByteDance Seed 提出 MoA（Mixture of Activations）：共享线性投影，但用轻量级输入依赖门控为每个 token 选择最优的激活函数混合比例。理论证明 MoA 严格比固定激活更具表达力。实验：在 0.12B-2B 规模跨架构（dense/MoE）、多 token 预算下，MoA 持续实现更低终端损失。

| 年份 | 成熟度 | PM 相关性 |
|------|--------|---------|
| 2025 | 🧪 | **低**（LLM 架构研究，主要对模型训练团队有参考价值）|

## 核心机制

```
关键对比：
  MoE（专家混合）：路由 token 到不同参数专家（参数量增加）
  MoA（激活混合）：共享相同线性投影，但每个 token 选择不同激活函数混合

三层递进设计：
  1. 固定激活 FFN（传统 SwiGLU/ReLU）
     所有 token 用同一激活函数
     
  2. LA（Learnable Activations，输入无关）
     对激活函数字典做线性组合（参数共享，跨 token 一致）
     Type-I: f(x) = W₂(Σ αₖ σₖ(W₁x))
     Type-II SwiGLU 版：One-sided / Bi-sided / Quadratic 变体
     
  3. MoA（Mixture of Activations，输入依赖）
     每个 token 获得独特的激活混合系数
     轻量级门控网络（相比 MoE 参数开销极低）

理论结果：
  Fixed FFN ⊂ LA ⊂ MoA（严格包含关系）
  即：MoA 严格比 LA 严格比固定激活更具表达力
  
激活字典（Type-I/Type-II 各有不同）：
  {ReLU, GELU, SiLU, LeakyReLU, ReLU², tanh, ...}
```

## 关键数据 📅 2025

| 实验设置 | MoA 效果 |
|---------|---------|
| Dense 模型 0.12B-2B | **持续更低终端损失** |
| MoE 模型 | **持续更低终端损失** |
| 多 token 预算 | **稳定提升** |
| 多优化器/学习率调度 | **稳定提升** |

## PM 结论

- 📅 2025 年，这是 FFN 设计的精细化研究，属于"以更低代价提升表达力"的架构优化方向
- 与 MoE 不同：**MoA 不增加参数，只改激活选择方式**——在成本固定时能提升性能
- 实用价值：对于需要最大化固定参数规模内模型能力的场景（如边缘部署），MoA 是可选的架构优化
- 系列论文（Part I），后续可能有 Part II

## 论文精读卡片

**一句话**：MoA（Mixture of Activations）通过轻量级输入依赖门控让每个 token 获得不同激活函数混合，在 0.12B-2B 规模、dense 和 MoE 架构上持续实现更低终端损失，且零额外参数成本。

**问题**：现有 LLM 的 FFN 层对所有 token 使用相同激活函数（SwiGLU 等），这在表达力上是次优的——不同 token 的语义需求差异巨大，用统一激活函数是一刀切的设计；但 MoE 路由方案会大幅增加参数量。

**核心方法**：
- LA（Learnable Activations）：先实现输入无关的激活函数线性组合，建立对固定激活的严格超集
- MoA（Mixture of Activations）：引入轻量级门控网络，为每个 token 生成独特的激活混合系数（输入依赖）
- 理论证明：Fixed FFN ⊂ LA ⊂ MoA 严格包含关系，MoA 不增加线性层参数，只增加门控网络参数

**关键图/公式**：Type-I MoA: f(x) = W₂(Σ αₖ(x) σₖ(W₁x))，其中 αₖ(x) 是 token 依赖的门控系数；关键是 αₖ(x) 对 x 的依赖性——这使得 MoA 严格比 LA 更具表达力。

**实验设置**：
- 规模/数据：0.12B 到 2B 参数；dense 和 MoE 两类架构；多种 token 预算（不同训练步数）
- 对比：固定激活（SwiGLU/ReLU）、LA（可学习固定激活）、MoA；多种优化器和学习率调度

**最强证据**：跨架构（dense/MoE）、跨规模（0.12B-2B）、跨 token 预算的持续更低终端损失，说明 MoA 的优势来自系统性原因而非过拟合某一设置。

**最弱证据**：缺乏在实际下游任务（如 MMLU、HumanEval）上的绝对提升数字；只有损失曲线；规模停在 2B，未验证 7B+ 规模的稳定性；系列论文 Part I，最终能否应用于生产未知。

**可复用点**：门控网络实现输入依赖激活混合的思想——在固定参数预算内提升表达力，可作为 FFN 层的插件式替换，对边缘部署场景（参数固定时优化能力上限）有直接价值。

**和哪些论文相关**：
- [[Wiki/概念/01_架构技术/MoE混合专家架构]] — MoA 与 MoE 的对比：MoE 路由不同参数，MoA 路由不同激活函数
- [[Wiki/概念/01_架构技术/LoRA与参数高效微调]] — 类似"轻量级附加组件"的设计哲学

**我能拿它做什么**：
- 在固定参数预算的模型训练中，将 SwiGLU 替换为 MoA 作为低成本性能改进尝试
- 关注 Part II 系列论文，了解 MoA 在更大规模上的验证结果
- 理解"表达力"的严格数学定义用于评估其他架构改进方案

**3天后要回忆的问题**：
1. MoA 与 MoE 的根本区别是什么？各自的参数开销如何？
2. Fixed FFN ⊂ LA ⊂ MoA 的严格包含关系如何证明？
3. MoA 的门控网络参数量占总参数的比例大约是多少？
4. 为什么在 0.12B 到 2B 跨规模都验证比只在一个规模验证更有说服力？
5. 如果你要在生产模型中引入 MoA，需要做哪些工程评估？

## 原子概念索引
- [[Wiki/概念/01_架构技术/MoE混合专家架构]] — MoA 与 MoE 的关键对比：激活混合 vs 参数路由
