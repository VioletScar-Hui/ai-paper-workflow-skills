---
tags: [论文笔记, FLAN, 指令微调, Instruction Tuning, 零样本, Google, LaMDA]
paper_id: "65"
笔记层级: 骨干
复核日期: 2026-07-04
---

# FLAN：微调语言模型是零样本学习者

📄 **原文 PDF**：[[RAW/65 - Finetuned Language Models Are Zero-Shot Learners.pdf]]

## PM 速判
> 指令微调（Instruction Tuning）的奠基论文。在 62 个 NLP 任务上用指令格式微调 137B LaMDA，零样本超越 GPT-3 175B few-shot 在 25 个任务中的 20 个——证明"教会模型理解指令"比"给模型看示例"更能泛化到未见任务。直接启发了 ChatGPT 的 SFT 阶段。

## 双层费曼 🗣️
> **给CEO**：大模型像是一个读过很多书但听不懂人话的天才。FLAN 的方法很简单——用 62 种不同类型的任务，每个任务都用"请做X"这种指令格式去教它，教完之后它遇到从没见过的任务也能听懂指令了。结果：一个 137B 参数的模型，不需要任何示例，表现就超过了 175B 参数的 GPT-3（后者需要看几个示例才会做）。这证明了"教模型理解指令"这一步骤的巨大价值。
> **给工程师**：将 62 个 NLP 数据集（覆盖分类、翻译、摘要、问答、NLI 等任务族）统一转化为 instruction 格式——每个任务配 10 条手工编写的自然语言指令模板，输入保持不变。在 137B LaMDA-PT 上做标准 LM 微调，loss 仍然是 next-token prediction。测试时在训练期间未见过的任务上做零样本推理。核心发现：(1) 任务类型多样性比单任务数据量更关键；(2) 规模效应：8B 模型指令微调几乎无收益，137B 收益显著——指令微调的泛化能力需要模型本身足够大才能"解锁"。

## 问题域定位 🎯
- **回应什么根本约束？** 大模型 few-shot 能力强但 zero-shot 弱——这是一个"用户体验"约束。Few-shot 需要用户精心挑选和编写示例，在真实产品中这是巨大的认知负担。如果能用训练（而非推理时的 prompt engineering）来缩小 zero-shot 和 few-shot 的差距，产品体验会质变。
- **之前卡在哪？** GPT-3 展示了强大的 few-shot 能力（通过上下文示例激活），但 zero-shot 显著落后。大家默认"不提供示例，模型就不知道该怎么做"——FLAN 在质疑这个默认假设：是否可以训练模型"理解任务指令"，而不是每次都靠示例来"回忆"该怎么做？
- **开启/关闭了哪条路线？** 开启：指令微调训练路线（FLAN → T0 → InstructGPT → ChatGPT），"多任务指令化训练作为通用接口"的范式。关闭：纯依赖 prompt engineering（精心设计的 few-shot 示例）作为提升模型表现的唯一手段。**未关闭但被深化**：FLAN 只用 62 个任务，后续发现任务数到 1.8K（FLAN-T5）和更多（FLAN-PaLM）时效果持续提升——"62 是下限而非上限"。

## 核心机制
```
                    62 个 NLP 数据集
                    (分类 / 翻译 / 摘要 / 问答 / NLI ...)
                            │
                            │ 每个任务手写 ~10 条指令模板
                            ▼
              ┌─────────────────────────────┐
              │  Instruction 格式训练数据     │
              │                              │
              │  "Translate to French:       │
              │   The cat is on the mat.     │
              │   → Le chat est sur le tapis."│
              │                              │
              │  "Is this review positive    │
              │   or negative?               │
              │   'This movie was terrible.' │
              │   → negative"                │
              │                              │
              │  "Answer the question:       │
              │   What is the capital of FR? │
              │   → Paris"                   │
              └─────────────────────────────┘
                            │
                            │ 标准 LM fine-tuning (next-token prediction)
                            ▼
                    FLAN (137B LaMDA-PT)
                            │
                            │  测试: 在训练时从未见过的任务上 zero-shot
                            ▼
              ┌─────────────────────────────┐
              │  未见过任务举例:              │
              │  "Determine if the following │
              │   sentence entails ..."      │
              │  → FLAN 直接作答，无需示例    │
              └─────────────────────────────┘

        任务族聚类划分 train/test (按任务类型而非随机划数据点)
        确保测试任务来自训练中从未见过的任务族
```

关键设计点：(1) 指令模板多样性——同一任务用 10 种不同的自然语言表述（如 "Translate X" / "Say X in French" / "French version of X"），防止模型死记硬背特定措辞；(2) 按任务族划分 train/test——如果训练集出现过"自然语言推理"类型的任务，测试时不应再测 NLI 类型的新任务，否则测的是同类型泛化而非跨类型泛化。

## 设计决策解剖 ⚖️
| 决策点 | 选择 | 被放弃方案 | 为什么 | 何时失效 |
|--------|------|----------|--------|---------|
| 训练目标 | 标准 next-token LM loss（与预训练相同） | 分类头/seq2seq 专用 loss | 保持与预训练目标一致，模型不需要学习新的输出格式——所有输出都是自然语言 token 流，统一扩散到任意任务 | 当任务需要结构化输出（如 JSON）时，LM loss 不保证格式正确性；需要 constrained decoding 补充 |
| 任务数量 | 62 个任务，按族聚类 | 更多任务（如后来 1.8K）或更少任务（如仅 10 个） | 62 是在当时标注预算内的折中；消融显示 8→62 收益递增但未饱和，暗示越多越好 | 当任务覆盖面足够广后（~1K+），边际收益递减；FLAN 论文本身未找到上限，说明 62 远非最优 |
| 指令来源 | 人工手写 10 条模板/任务 | 自动生成模板或用 few-shot prompt 代替 | 人工写保证自然性和覆盖度，自动生成当时容易产生不自然的表达 | 任务数扩展到千级时手工写不可持续，需改用 LLM 自动生成指令模板（如 Self-Instruct 路线） |
| 评测方式 | 零样本（zero-shot），完全不提供示例 | Few-shot 评测 FLAN | 论文的目标就是证明指令微调能缩小 zero-shot gap，用 few-shot 测就偏离了核心论证。与 GPT-3 few-shot 对比时，FLAN 用 zero-shot，突出"训练让 zero-shot 超过 few-shot" | 当 zero-shot 已经足够好时（如 FLAN-T5 已远超前代），few-shot 边际增益变小，评测重点应转向更难的任务和数据效率 |
| 模型规模 | 137B（单一大模型） | 多个小模型 ensemble 或 8B 版本 | 消融显示 8B 指令微调几乎无收益——指令跟随能力是涌现能力，需要百 B 级别才能显现 | 后续 FLAN-T5 在 11B 上效果也很好——说明尺度阈值不是绝对的，随数据质量和任务数量提升，小模型也能受益 |

## 成本与量级 💰
- **训练**：137B LaMDA-PT 微调，62 个任务共约数百万条训练样本。论文未报告确切 GPU-hours，但 137B 的全量微调在当时（2021）成本不低——保守估计 64×TPUv4 数天。
- **推理**：137B 参数推理需大量显存（~274GB FP16），无法单卡部署。论文的学术定位明确，不关心推理成本。
- **数据成本**：每个任务手写 10 条模板——62×10=620 条指令模板，约 1-2 人天。这是整个流程中最便宜但最关键的步骤。后续 Self-Instruct 等方法将其自动化。
- **最小可行配置**：用 T5-base（220M）+ 5 个任务族即可验证"指令微调提升零样本泛化"的定性趋势，估计 1×A100 数小时内完成。

## 证据审计 🔬
- **实验公平性**：对比基线包括未微调的 LaMDA-PT zero-shot、GPT-3 zero-shot、GPT-3 few-shot。但 FLAN 用了 62 个任务微调，GPT-3 用的是 prompt 工程——能力来源不对等（训练 vs 推理时技巧）。更公平的对比应该包括"GPT-3 也做类似的多任务微调"（但当时 GPT-3 权重未公开，不可行）。
- **最强证据**：FLAN zero-shot 在 ANLI（对抗性 NLI，最能测泛化的 NLP 任务之一）上以 43.6 超过 GPT-3 few-shot 的 40.1——在"最难泛化"的任务上取胜，论证最强。
- **最可疑数字及原因**：FLAN 在部分任务上 zero-shot 超过 GPT-3 few-shot，但 GPT-3 的 few-shot 质量高度依赖 prompt 模板质量——论文中 GPT-3 few-shot 的 prompt 是"标准做法"，但未做 prompt 优化，可能低估了 GPT-3 few-shot 的潜力。另外，62 个任务的训练数据可能无意中包含了与测试任务语义重叠的数据（尽管按任务族划分，但语义重叠难以完全避免）。
- **审稿补充要求**：缺少小模型（<8B）上的指令微调分析；缺少指令微调对有害/偏见输出的影响分析；缺少指令模板数量的消融（每个任务配 1/5/10/50 条的影响）。
- **最小复现设计**：用开源 T0 模型（基于 T5）在 10 个任务族上做消融，验证"任务族数量增长→零样本泛化提升"的曲线。T0 论文本身已是 FLAN 的最强复现和推广。

## 可复用点
- **何时采用**：(1) 构建垂直领域 LLM 助手时，优先收集多类型的指令-回答对（而非单一任务的大量数据），参照 FLAN 的"任务族多样性"哲学；(2) 需要零样本能力但模型在特定类别任务上表现差时，加入该任务族的少量指令微调数据；(3) 作为 SFT 阶段的数据格式参考（instruction-input-output 三元组）。
- **何时规避**：(1) 目标只有一个或少数几个任务时——直接在目标任务上微调比"多任务指令微调"更高效，FLAN 的优势体现在目标外的泛化；(2) 模型很小（<7B）时——指令微调的泛化收益不明显，不如做专门的任务微调；(3) 有充分的 few-shot 数据可用时——few-shot 在某些场景的准确度仍然高于 zero-shot。
- **供应商拷问清单**：
  1. "你们的指令微调数据覆盖了多少个任务族？是按什么标准划分任务族的？"
  2. "你们的指令模板是人工写的还是 LLM 生成的？多样性怎么保证？"
  3. "评测时有没有按照任务族划分 train/test？还是随机划分（会高估泛化能力）？"
  4. "在哪些未见过的任务族上你们的模型 zero-shot 显著退步？（负迁移检测）"
  5. "你们的指令微调数据有没有可能和测试任务语义重叠？怎么排查的？"

## 关联网络 🕸️
- **相关论文**：
  - [[Wiki/论文笔记/06_训练对齐RL/70_InstructGPT训练语言模型遵循人类指令]] — InstructGPT 在 FLAN 指令微调基础上加入 RLHF，形成 SFT→RM→PPO 三步流程。FLAN 证明了"指令微调能泛化"，InstructGPT 证明了"再加 RLHF 能进一步对齐人类偏好"
  - [[Wiki/论文笔记/05_推理思维链/68_ChainOfThought思维链提示激发大模型推理]] — 同第一作者 Jason Wei。CoT 本质是在 prompt 中加了推理指令；FLAN-CoT 后来将 CoT 融入指令微调框架，形成推理能力的训练路线
  - [[Wiki/论文笔记/01_LLM基础架构/69_PaLM用Pathways扩展语言建模]] — Google 后续 FLAN-PaLM 将指令微调扩展到 540B，验证了规模继续扩展后的指令微调收益
  - [[Wiki/论文笔记/02_前沿模型报告/64_Codex评估在代码上训练的大型语言模型]] — 同期（2021）的"专项数据微调"双生子。Codex 走代码专项路线，FLAN 走多任务泛化路线，互相印证"微调解锁特定能力"
- **相关概念**：
  - [[Wiki/概念/02_训练方法/RLHF与RLAIF]] — 指令微调 (SFT) 是 RLHF 三步的第一步
  - [[Wiki/概念/03_推理与评测/提示工程与指令跟随]] — 指令微调训练模型遵循指令，是提示工程的"训练侧"补充
- **冲突/印证**：[[Wiki/论文笔记/02_前沿模型报告/64_Codex评估在代码上训练的大型语言模型]] 与 FLAN 存在有趣的路线张力——Codex 暗示"在单一领域深入（159GB Python）可解锁强能力"，FLAN 暗示"跨任务广度比单领域深度更重要"。这看似矛盾实则互补：FLAN 追求的是通用 zero-shot 泛化（范围广），Codex 追求的是单一领域的功能正确性（深度深）。后续 FLAN-Collection 用 1.8K 任务证明——同时追求广度和深度才是终极路线，但计算成本也远超两者。

## 动手练习 💻
**练习目标**：构造 FLAN 风格的 instruction 格式数据集，理解 SFT 数据的三元组结构。
```python
"""
FLAN 风格 Instruction 数据集构造器

FLAN 的核心贡献之一是将异构 NLP 任务统一为 instruction-input-output
三元组，使模型能通过同一个 LM loss 学习所有任务。本练习演示：
1. 不同任务类型如何映射到统一的 instruction 格式
2. 指令模板的多样性如何影响数据
3. 简单的 data loader 实现
"""

import json
import random
from dataclasses import dataclass, field
from typing import List, Dict, Optional

# ============ 数据结构 ============

@dataclass
class InstructionExample:
    """FLAN 风格的单条训练样本。"""
    instruction: str   # 任务描述 / 指令
    input_text: str    # 任务的输入 (可为空字符串)
    output_text: str   # 期望的输出 / 答案
    task_family: str   # 属于哪个任务族 (用于分析多样性)

    def to_prompt_completion(self) -> tuple:
        """转成标准的 (prompt, completion) 对，用于 SFT。"""
        # FLAN 原始格式: "instruction\n\ninput\n\n→ output"
        # 这里用更通用的模板
        if self.input_text:
            prompt = f"{self.instruction}\n\n{self.input_text}\n\nAnswer:"
        else:
            prompt = f"{self.instruction}\n\nAnswer:"
        completion = f" {self.output_text}"
        return prompt, completion

    def to_messages_format(self) -> dict:
        """转成 OpenAI Chat 兼容的 messages 格式 (后续模型的通用做法)。"""
        user_content = self.instruction
        if self.input_text:
            user_content += f"\n\n{self.input_text}"
        return {
            "messages": [
                {"role": "user", "content": user_content},
                {"role": "assistant", "content": self.output_text}
            ]
        }


# ============ 任务 → Instruction 转换 ============

class TaskToInstruction:
    """
    演示如何将不同 NLP 任务统一映射到 instruction 格式。
    每种任务类型有多个指令模板 — 每次随机选一个以增加多样性。
    """

    # 各任务类型的指令模板池
    TEMPLATES = {
        "sentiment": [
            "Classify the sentiment of the following review as positive or negative.",
            "Is the following review positive or negative?",
            "Determine the sentiment: positive or negative?",
            "What is the sentiment expressed in this review? Choose positive or negative.",
            "Read this review and decide if the author's attitude is positive or negative.",
        ],
        "translation_en_fr": [
            "Translate the following English sentence to French.",
            "Convert this English text to French.",
            "Provide the French translation of this English sentence.",
            "How would you say this in French?",
            "Translate to French:",
        ],
        "summarization": [
            "Summarize the following article in one sentence.",
            "Provide a one-sentence summary of this text.",
            "What is the main point of this passage? Summarize it briefly.",
            "In one sentence, what is this about?",
            "Give me a concise summary:",
        ],
        "question_answering": [
            "Answer the following question based on the provided context.",
            "Read the context and answer the question.",
            "Given this information, answer the question.",
            "Context: ... Question: ... What is the answer?",
            "Use the provided text to find the answer to this question.",
        ],
    }

    @classmethod
    def convert_sentiment(cls, review: str, label: str) -> InstructionExample:
        instruction = random.choice(cls.TEMPLATES["sentiment"])
        return InstructionExample(
            instruction=instruction,
            input_text=review,
            output_text=label,  # "positive" or "negative"
            task_family="sentiment",
        )

    @classmethod
    def convert_translation(cls, english: str, french: str) -> InstructionExample:
        instruction = random.choice(cls.TEMPLATES["translation_en_fr"])
        return InstructionExample(
            instruction=instruction,
            input_text=english,
            output_text=french,
            task_family="translation_en_fr",
        )

    @classmethod
    def convert_summarization(cls, article: str, summary: str) -> InstructionExample:
        instruction = random.choice(cls.TEMPLATES["summarization"])
        return InstructionExample(
            instruction=instruction,
            input_text=article,
            output_text=summary,
            task_family="summarization",
        )

    @classmethod
    def convert_qa(cls, context: str, question: str, answer: str) -> InstructionExample:
        instruction = random.choice(cls.TEMPLATES["question_answering"])
        return InstructionExample(
            instruction=instruction,
            input_text=f"Context: {context}\n\nQuestion: {question}",
            output_text=answer,
            task_family="question_answering",
        )


# ============ 演示 ============

if __name__ == "__main__":
    print("=" * 60)
    print("FLAN 风格 Instruction 数据集构造演示")
    print("=" * 60)

    # 构造一个小的混合数据集
    dataset: List[InstructionExample] = []

    # 情感分析样本 (2条)
    dataset.append(TaskToInstruction.convert_sentiment(
        review="This movie was absolutely fantastic! I loved every minute.",
        label="positive"
    ))
    dataset.append(TaskToInstruction.convert_sentiment(
        review="Waste of time. Terrible acting and boring plot.",
        label="negative"
    ))

    # 翻译样本 (2条)
    dataset.append(TaskToInstruction.convert_translation(
        english="The cat is on the mat.",
        french="Le chat est sur le tapis."
    ))
    dataset.append(TaskToInstruction.convert_translation(
        english="Hello, how are you?",
        french="Bonjour, comment allez-vous?"
    ))

    # 摘要样本 (1条)
    dataset.append(TaskToInstruction.convert_summarization(
        article="Scientists discovered a new exoplanet orbiting Proxima Centauri. "
                "The planet is within the habitable zone and may contain liquid water.",
        summary="A potentially habitable exoplanet was discovered orbiting Proxima Centauri."
    ))

    # QA 样本 (1条)
    dataset.append(TaskToInstruction.convert_qa(
        context="Paris is the capital of France. It has a population of 2.1 million.",
        question="What is the capital of France?",
        answer="Paris"
    ))

    # 打印数据集概览
    task_family_counts = {}
    for ex in dataset:
        task_family_counts[ex.task_family] = task_family_counts.get(ex.task_family, 0) + 1

    print(f"\n数据集概览: 总计 {len(dataset)} 条样本")
    for family, count in task_family_counts.items():
        print(f"  {family}: {count} 条")

    print(f"\n{'='*60}")
    print("样例展示 (prompt → completion 格式)")
    print(f"{'='*60}")
    for i, ex in enumerate(dataset):
        prompt, completion = ex.to_prompt_completion()
        print(f"\n--- 样本 {i+1} [{ex.task_family}] ---")
        print(f"PROMPT:\n{prompt}")
        print(f"COMPLETION:\n{completion}")

    print(f"\n{'='*60}")
    print("关键设计原则总结")
    print(f"{'='*60}")
    print("1. 所有任务统一为 instruction + input → output 三元组")
    print("2. 同任务类型使用多个指令模板 → 防止模型记忆特定措辞")
    print("3. 数据集按任务族划分: 测试任务族不出现在训练中")
    print("4. LM loss (next-token) 直接学习 instruction→answer 的映射")
    print("5. 任务类型多样性 >> 单任务数据量 (FLAN 核心发现)")
```

## 自测三层 🎓
**L1 复述**：(1) FLAN 在多少个任务上做指令微调？基础模型是什么？(2) FLAN zero-shot 和 GPT-3 few-shot 的对比结果——在多少个任务中 FLAN 胜出？

**L2 解释**：(1) 为什么 FLAN 选择在 137B 模型上微调，而不是 8B？如果 8B 指令微调几乎无收益，这对小模型的微调策略有什么启示？(2) 按任务族划分 train/test 和随机划分有什么本质区别？如果只做随机划分会有什么误导性结论？

**L3 应用**：(1) 你要为一个法律 AI 产品构造指令微调数据。目标场景包括：合同审查、法规问答、案例摘要。请参照 FLAN 方法论，设计你的数据收集策略（任务族分类、指令模板设计、train/test 划分方案）。(2) 你的客户是一个电商平台，想让 LLM 理解 20 种不同商品品类的问题（比如"这件衣服缩水吗？""这个手机的电池能用多久？"）。你是应该为每个品类收集等量数据，还是优先覆盖尽可能多的品类但每个品类数据量少？请用 FLAN 的消融实验结论支撑你的决策。

📅 知识时间锚：2021-09（FLAN 发布），与 Codex 同期，两篇论文共同开启了"专项微调"时代。ChatGPT 的 SFT 阶段直接继承了 FLAN 的 instruction 格式范式。
