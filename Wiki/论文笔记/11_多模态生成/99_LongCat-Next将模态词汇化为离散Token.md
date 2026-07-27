---
tags: [论文笔记, LongCat-Next, 统一多模态, 离散Token, DiNA, dNaViT, Meituan]
笔记层级: 参考
paper_id: "99"
filename: "99 - LongCat-Next - Lexicalizing Modalities as Discrete Tokens.pdf"
authors: "Meituan LongCat Team"
year: 2025
成熟度: 🧪
---

# LongCat-Next：将模态词汇化为离散 Token

📄 **原文 PDF**：[[RAW/99 - LongCat-Next - Lexicalizing Modalities as Discrete Tokens.pdf]]

## PM 速判（30秒）

> **文本/视觉/音频统一为离散 Token，用单个自回归模型"看图、画图、说话"。** 美团 LongCat 团队提出 DiNA（Discrete Native Autoregressive）框架：将所有模态表示在共享离散空间中，核心是 dNaViT（Discrete Native Any-resolution Visual Transformer）——支持任意分辨率的层次化离散视觉 Token。在理解和生成任务上都达到强竞争力，同时解决了离散 Token 在理解任务上长期存在的性能天花板。

| 年份 | 成熟度 | PM 相关性 |
|------|--------|---------|
| 2025 | 🧪 | **中**（统一多模态大模型架构趋势，理解下一代 AI 产品基础）|

## 核心机制

```
核心问题：
  现有多模态系统是"以语言为中心"的 → 视觉/音频是附件
  导致：碎片化架构 + 次优的跨模态融合

DiNA（Discrete Native Autoregressive）框架：
  ① 所有模态 → 共享离散空间（统一词汇表）
  ② 单一 AR 建模目标（Next-Token Prediction）
  ③ 最少模态特定设计

核心创新——dNaViT：
  任意分辨率 Tokenization/De-tokenization
  层次化离散视觉 Token（Hierarchical）
  解决方案：Semantic-and-Aligned Encoder (SAE)
  突破：解决离散视觉 Token 在理解任务上的性能天花板
  
音频 Tokenizer：
  类似的离散化方案用于音频

语言模型主干 + 多模态组件：
  End-to-End Multimodal Embedding
  Multimodal Head（输出多模态）
  Internal Linguistic Guidance（内部语言引导机制）

训练基础：
  理解-生成冲突的统一解决方案
  "Seeing（理解）、Painting（生成）、Talking（语音）"单框架
```

## 关键优势

| 能力 | 描述 |
|------|------|
| 理解 | 超越离散 Token 的传统性能上限 |
| 生成 | 竞争力强的视觉生成能力 |
| 音频 | 统一语音处理 |
| 架构 | 极简：单 AR 模型，最少模态特定组件 |

## PM 结论

- 📅 2025 年，"统一多模态离散 AR 模型"是学界争相探索的范式（GPT-4o 内部也类似思路）
- **关键在于 dNaViT**：任意分辨率离散化是难点，之前的离散视觉 Token 在理解上性能差
- 对 PM：单模型处理看/画/说 → 大幅降低产品基础设施复杂度（不用维护 3 个独立模型）
- 与 LongCat-AudioDiT / Video-Avatar 的关系：同属 LongCat 系列，代表美团在多模态生成领域的系统布局

## 论文精读卡片

**一句话**：DiNA 框架将文本/视觉/音频统一为共享离散 Token 空间，核心创新 dNaViT 支持任意分辨率的层次化视觉离散化，突破了离散视觉 Token 在理解任务上的历史性能天花板，实现单模型"看图+画图+说话"。

**问题**：多模态系统长期是"以语言为中心"的碎片化架构——视觉/音频作为附件通过连续嵌入接入 LLM；这导致架构复杂、跨模态融合次优，且连续嵌入难以支持真正统一的生成建模。而离散 Token 统一方案的历史瓶颈是视觉理解性能差。

**核心方法**：
- DiNA（Discrete Native Autoregressive）框架：所有模态→共享离散空间，单一 Next-Token Prediction 目标，最少模态特定组件
- dNaViT（Discrete Native Any-resolution Visual Transformer）：支持任意分辨率的层次化离散视觉 Token，Semantic-and-Aligned Encoder (SAE) 解决理解性能瓶颈
- Internal Linguistic Guidance：内部语言引导机制，统一协调多模态生成与理解

**关键图/公式**：核心架构极简化——语言模型主干 + End-to-End 多模态嵌入 + 多模态输出 Head；与连续嵌入方案的对比：离散化统一词汇表使 AR 建模目标真正统一，而不是混合目标。

**实验设置**：
- 规模/数据：具体参数量未在笔记中明确；理解和生成双重评估；对比离散 Token 历史性能上限
- 对比：离散 Token 基线（传统离散视觉方案）；连续嵌入多模态模型；分别评估理解和生成能力

**最强证据**：突破离散 Token 在视觉理解任务上的"传统性能上限"——这是该方向长期以来的根本性障碍，dNaViT 的解决标志着离散统一方案的可行性证明。

**最弱证据**：笔记中缺少具体的数值结果（如 VQA、图像生成 FID 等）；"强竞争力"是定性描述；与 GPT-4o 等连续嵌入系统的精确比较未提供；任意分辨率支持的计算成本未量化。

**可复用点**：Semantic-and-Aligned Encoder (SAE) 的设计理念——通过语义对齐解决离散化导致的信息损失，可作为任何需要离散化连续模态的系统的参考架构。

**和哪些论文相关**：
- [[Wiki/概念/01_架构技术/原生多模态嵌入]] — DiNA 是原生多模态建模的离散 Token 实现路径
- [[Wiki/论文笔记/11_多模态生成/98_LongCat-AudioDiT波形潜在空间的高保真扩散TTS]] — 同属 LongCat 系列，音频处理方案可能与 DiNA 的音频 Tokenizer 相关
- [[Wiki/论文笔记/11_多模态生成/97_LongCat-Video-Avatar-1.5音频驱动视频生成]] — LongCat 系列的视频生成能力基础

**我能拿它做什么**：
- 在多模态系统架构选型时，评估离散 Token 统一方案 vs 连续嵌入混合方案的权衡
- 用 dNaViT 的任意分辨率离散化思路解决视觉输入分辨率归一化问题
- 作为"单模型多能力"产品化方案的技术依据（减少多模型维护成本）

**3天后要回忆的问题**：
1. DiNA 和"以语言为中心"的传统多模态架构的根本区别是什么？
2. dNaViT 如何突破离散视觉 Token 在理解任务上的性能天花板？SAE 的作用是什么？
3. 为什么"共享离散空间"使 AR 建模目标更统一？连续嵌入方案的建模目标有什么问题？
4. 离散 Token 方案在视觉理解和视觉生成上各有什么历史挑战？LongCat-Next 如何分别解决？
5. Internal Linguistic Guidance 的作用机制是什么？

## 原子概念索引
- [[Wiki/概念/01_架构技术/原生多模态嵌入]] — DiNA 框架是离散 Token 统一多模态建模的核心实现
