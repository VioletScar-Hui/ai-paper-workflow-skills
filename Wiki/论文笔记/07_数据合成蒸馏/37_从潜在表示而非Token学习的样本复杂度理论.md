---
tags: [论文笔记, 样本复杂度, 自监督学习, 潜在空间, 理论ML, EPFL]
笔记层级: 标准
paper_id: "37"
filename: "37 - Learn from your own latents and not from tokens - A sample-complexity theory.pdf"
authors: "Daniel J. Korchinski, Alessandro Favero, Matthieu Wyart (EPFL / Cambridge / Johns Hopkins)"
year: 2026
成熟度: 🔬
---

# 从潜在表示而非 Token 学习：样本复杂度理论

📄 **原文 PDF**：[[RAW/37 - Learn from your own latents and not from tokens - A sample-complexity theory.pdf]]

## PM 速判（30秒）

> **深度 ML 理论论文，PM 直接影响极小。** 理论证明：data2vec/JEPA 类"预测自身潜在表示"的自监督方法（Learn-from-Latents SSL）只需 O(vm³) 样本，而 Token 级预测（Masked LM / Diffusion）需要 O(vm^L)（随层深指数增长）——这从信息论角度解释了为什么潜在预测 SSL 比 Token 级 SSL 更数据高效。

| 项目 | 评估 |
|------|------|
| **理论框架** | 随机层级模型 (RHM/PCFG) + 样本复杂度分析 |
| **核心结论** | 潜在预测 SSL: O(vm³)（与深度 L 无关）vs Token 级: O(vm^L)（指数增长）|
| **算法验证** | ILC 算法 + SLC 神经网络 + data2vec 实证，均符合 vm³ scaling |
| **成熟度** | 🔬 基础理论，EPFL / Cambridge |

## 核心结论（带时间锚）

1. **为什么 JEPA/data2vec 比 Masked LM 更数据高效有了理论解释** 📅 2026  
   潜在预测使每一层的学习问题统计强度相同；而 Token 级预测越深越难，指数级需要更多数据。

2. **data2vec 的首个样本复杂度证明**（理论里程碑）📅 2026

## 权威学习资源

- 📄 论文：Korchinski, Favero, Wyart（EPFL / Cambridge / JHU），2026

## 论文精读卡片

**一句话**：理论证明 data2vec/JEPA 类"预测自身潜在表示"的自监督方法样本复杂度为 O(vm³)，而 Token 级预测（Masked LM/Diffusion）随网络深度 L 指数增长为 O(vm^L)，首次为 JEPA 的数据高效性提供严格信息论依据。

**问题**：data2vec 和 JEPA 在实践中表现出比 Masked Language Model（BERT 类）更好的样本效率，但这种优势一直缺乏理论解释。为什么预测潜在表示比预测原始 token 更数据高效？这个问题关系到选择自监督训练范式的理论基础。

**核心方法**：
- 建立随机层级模型（Random Hierarchical Model，RHM）作为理论分析框架，模拟层级结构数据（语言/图像）的生成过程
- 在 RHM/PCFG 框架下推导两类 SSL 方法的样本复杂度上界：Token 级预测 O(vm^L) 和潜在预测 O(vm³)
- 设计 ILC 算法和 SLC 神经网络验证理论上界，并在 data2vec 实证中验证 vm³ scaling

**关键图/公式**：核心不等式：N_token ≥ O(vm^L) 而 N_latent ≥ O(vm³)；其中 v = 词汇量，m = 分支因子，L = 层级深度；当 L > 3 时 Token 级方法指数级需要更多数据，而潜在方法始终保持 O(m³) 样本复杂度（与 L 无关）；核心洞察：潜在预测使每层学习的统计强度相同，消除深度带来的指数放大。

**实验设置**：
- 规模/数据：理论分析 + 合成 RHM/PCFG 数据集上的算法验证 + data2vec 实证（自然语言基准）
- 对比：Token 级 SSL（MLM/Masked Autoencoder）vs 潜在预测 SSL（ILC/data2vec/JEPA 类），样本复杂度实测与理论上界对比

**最强证据**：ILC 算法在 RHM 上的实证样本复杂度符合 O(vm³)（而非 O(vm^L)），且 data2vec 的实证 scaling 曲线与 vm³ 理论预测一致；这是 data2vec 的首个理论样本复杂度证明。

**最弱证据**：RHM 是理想化模型，与真实语言/图像数据的差距未被充分讨论；理论在 PCFG 等层级结构数据上成立，但 Transformer 的实际学习机制是否匹配 RHM 假设未被验证；分析仅考虑泛化样本复杂度，未讨论计算复杂度。

**可复用点**：为选择自监督预训练范式提供理论依据——数据稀缺场景优先选 JEPA/data2vec 类潜在预测方法；O(vm³) vs O(vm^L) 的对比可用于向团队解释为什么潜在预测 SSL 更数据高效；vm³ scaling 可用于估计不同数据规模下潜在预测模型的预期性能。

**和哪些论文相关**：
- [[Wiki/概念/03_推理与评测/思维链与推理模型]] — JEPA 类潜在空间预测与 CoT 中间步骤表示有概念联系：两者都在内部表示层面而非 token 层面进行计算
- [[Wiki/概念/01_架构技术/LoRA与参数高效微调]] — 样本复杂度理论对参数高效微调的数据需求估计有理论借鉴意义

**我能拿它做什么**：
- 在数据稀缺的领域（医疗/法律等专有领域）选择预训练范式时，优先考虑 JEPA/data2vec 类潜在预测方法而非 MLM
- 用 O(vm³) 样本复杂度估计为数据采集量决策提供理论依据（"需要多少数据才能训好"）
- 向关注预训练效率的机器学习工程师分享这个理论，帮助他们在数据预算有限时做出更好的架构选择

**3天后要回忆的问题**：
1. Token 级 SSL 和潜在预测 SSL 的样本复杂度分别是什么？关键参数是什么？
2. 为什么潜在预测的样本复杂度与深度 L 无关？
3. 论文使用了什么理论框架来推导这些结论（RHM 是什么）？
4. data2vec 的实证结果如何支持理论预测？
5. 这个理论的主要假设局限是什么？在什么情况下可能不成立？

## 原子概念索引

- [[Wiki/概念/02_训练方法/RLHF与RLAIF]] — 样本复杂度理论对自监督预训练数据效率的分析，与强化学习微调的数据需求形成互补视角
