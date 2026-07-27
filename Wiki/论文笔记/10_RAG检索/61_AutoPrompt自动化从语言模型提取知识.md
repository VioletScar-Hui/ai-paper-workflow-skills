---
tags: [论文笔记, AutoPrompt, 提示优化, 梯度搜索, 基础论文, Berkeley]
笔记层级: 标准
paper_id: "61"
filename: "61 - AutoPrompt - Eliciting Knowledge from Language Models with Automatically Generated Prompts.pdf"
authors: "Taylor Shin et al. (UC Berkeley)"
year: 2020
成熟度: ✅
---

# AutoPrompt：用自动生成的提示从语言模型提取知识

📄 **原文 PDF**：[[RAW/61 - AutoPrompt - Eliciting Knowledge from Language Models with Automatically Generated Prompts.pdf]]

## PM 速判（10秒）

> **经典论文，背景知识。** 提出通过梯度引导的离散 prompt 搜索（gradient-guided token search），自动发现能最大化模型知识提取的触发词序列——是后续 Prompt Optimization（如 DSPy、APE 等）的先驱。

| 年份 | 成熟度 | PM 相关性 |
|------|--------|---------|
| 2020 | ✅ | 背景知识 |

## 论文精读卡片

**一句话**：AutoPrompt 通过梯度引导的离散 token 搜索（Universal Adversarial Triggers 改编），在情感分析上达到 91% 准确率、在关系抽取上超越人工 prompt 10%，无需手工设计提示词。

**问题**：手工设计的 prompt 依赖专家经验且效果不稳定——能否用自动化方法找到触发语言模型最优输出的离散词序列，并系统化地从预训练模型中提取事实知识？

**核心方法**：
- 梯度引导 Token 搜索（Gradient-Guided Token Search）：对每个 trigger token 位置，计算对目标标签的梯度，找到使损失最小的候选词（Hot-Flip 算法思想），迭代替换直至收敛
- AUTOPROMPT 框架：固定模板 `[输入] [trigger_1] ... [trigger_k] [MASK]`，其中触发词由搜索算法生成，[MASK] 位置的预测词即为模型答案
- 知识探测（Knowledge Probing）：在 LAMA 基准（维基百科三元组填空）上测试 LM 存储了哪些事实知识

**关键图/公式**：触发词更新规则——对位置 i 的 token，找到使 `∇_{e_i} L` 点积最大（方向最接近梯度）的词表候选词：`argmax_{t∈V} e_t · ∇_{e_i} L`——离散搜索而非连续梯度下降，避免输入空间不可微问题。

**实验设置**：
- 规模/数据：RoBERTa-Large（355M）、BERT-Large（340M）；LAMA（46K 知识探测 cloze 题）；SST-2 情感分析；RE 关系抽取
- 对比：人工设计 prompt、LPAQA 自动模板

**最强证据**：SST-2 情感分类 91%（人工 prompt 88%）；LAMA 关系探测超越人工 prompt 的 P@1 约 10 个百分点；触发词通常是人类难以理解的词序列（如"zoning tympanic Wren"），揭示 LM 知识编码方式与人类直觉不符。

**最弱证据**：离散搜索产生的 trigger 通常语义上无意义（adversarial 特性），不能作为真实用户 prompt；搜索过程需要大量前向/梯度计算，扩展到大模型时代计算成本极高；对白盒模型（需要梯度）才可用。

**可复用点**：梯度引导 token 搜索的思想——在红队测试（red-teaming）和对抗提示（adversarial prompt）领域仍然使用；它证明语言模型内部有大量隐性知识可被显式提取，是 prompt 可以触发 LM 能力的理论支撑。

**和哪些论文相关**：
- [[Wiki/论文笔记/01_LLM基础架构/60_GPT-3语言模型是小样本学习者]] — GPT-3 证明 ICL 有效，AutoPrompt 提供了自动化发现最优 prompt 的方法
- [[Wiki/论文笔记/01_LLM基础架构/55_GPT-2语言模型是无监督多任务学习者]] — AutoPrompt 证明 BERT 类模型中存储了大量 GPT-2 时代即开始关注的世界知识
- [[Wiki/概念/03_推理与评测/思维链与推理模型]] — AutoPrompt 是自动化 prompt 优化的早期工作，与 chain-of-thought 的手工 prompt 形成对比

**我能拿它做什么**：
- 在评估"prompt 工程 vs 自动化优化"时，引用 AutoPrompt 作为自动化优化路线的早期证据
- 理解 LLM 安全中"越狱 prompt"（jailbreak）的技术来源——梯度引导的对抗 trigger 和越狱有相同的技术根基
- 在讨论 DSPy、APE 等现代 prompt 优化框架时，能清晰说明它们与 AutoPrompt 的继承与改进关系

**3天后要回忆的问题**：
1. AutoPrompt 如何在离散词表上做梯度搜索？为什么不能直接梯度下降？
2. 为什么 AutoPrompt 找到的触发词对人类来说没有语义？这说明了什么？
3. AutoPrompt 和 GPT-3 的 few-shot ICL 有什么本质区别？
4. AutoPrompt 的方法只能用于白盒模型——这个限制如何在后续工作中被克服？
5. 今天的 DSPy 和 AutoPrompt 相比，最大的改进是什么？

## 原子概念索引

- [[Wiki/概念/03_推理与评测/思维链与推理模型]] — AutoPrompt 代表自动化离散 prompt 优化路线，是后续 prompt 工程方法论的早期参照
