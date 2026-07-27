---
tags: [论文笔记, ReAct, 推理与行动, Agent, 工具使用, 基础论文, Princeton]
paper_id: "73"
笔记层级: 骨干
复核日期: 2026-07-04
---

# ReAct：语言模型中推理与行动的协同

📄 **原文 PDF**：[[RAW/73 - ReAct - Synergizing Reasoning and Acting in Language Models.pdf]]

## PM 速判
> 所有现代 LLM Agent 框架的直接鼻祖——"想（Thought）→做（Action）→观察（Observation）"循环就是这篇论文定义的。没有 ReAct，就没有 LangChain Agent、AutoGPT、OpenAI Function Calling 的 prompt 范式。

## 双层费曼 🗣️
> **给CEO**：之前 LLM 要么只会"想"（写推理过程但可能胡说），要么只会"做"（调用工具但缺乏规划）。ReAct 让模型像人类一样交替思考和行动——先想清楚下一步做什么，执行后观察结果，再根据结果调整计划。这套"想→做→看→想"循环直接成为 ChatGPT 插件、AI Agent 产品的标准架构。
> **给工程师**：ReAct 将 LLM 的每一步输出定义为三元组序列 (Thought_t, Action_t, Observation_t)。Thought 是自然语言推理，Action 是对外部环境的结构化操作（如 Search[query]），Observation 是环境返回值。与 Act-only（无推理）相比，HotpotQA F1 提升 17.8%（27.4→45.2）；与 CoT-only（无行动）相比，幻觉率减半。关键是 Thought 作为"内部世界模型"桥接了推理和行动——推理引导行动选择，行动结果接地推理，形成闭环纠错。

## 问题域定位 🎯
- **回应什么根本约束？** LLM 被训练为纯文本生成器，缺乏与外部世界交互的原语。CoT 展示了推理能力，但推理在封闭语境中进行，无法获取新信息或验证假设。Act-only（直接调用工具）缺乏多步规划，容易在中间步骤迷失方向。
- **之前卡在哪？** 推理（内部）和行动（外部）被视为两个独立范式研究。CoT 派认为"推理就够了"，工具调用派认为"行动就够了"。没人把它们交错在一起，导致 Agent 要么是"盲目的行动者"要么是"空谈的推理者"。
- **开启/关闭了哪条路线？** **开启了**：推理+行动交织的 Agent 框架路线（LangChain、AutoGPT、OpenAI Assistants 均沿用）。**关闭了**：纯推理（CoT-only）或纯行动（Act-only）作为 Agent 独立方案的可能性——论文证明两者都不如交织。

## 核心机制

```
┌─────────────────────────────────────────────────────────┐
│                    ReAct 循环架构                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────┐    ┌──────────┐    ┌──────────────┐      │
│  │ Thought₁ │───▶│ Action₁  │───▶│ Observation₁ │      │
│  │ "需要先查 │    │ Search   │    │ "搜索返回:    │      │
│  │ 询X的出生 │    │ [X birth]│    │  X生于1980年"│      │
│  │  年份"    │    │          │    │              │      │
│  └──────────┘    └──────────┘    └──────┬───────┘      │
│                                         │               │
│  ┌──────────┐    ┌──────────┐    ┌──────▼───────┐      │
│  │ Thought₂ │◀───│ Action₂  │◀───│ Observation₂│      │
│  │ "现在知道 │    │ Search   │    │ "搜索返回:    │      │
│  │ 年龄=Z，  │    │ [Y death]│    │  Y卒于2020年"│      │
│  │  计算差值" │    │          │    │              │      │
│  └──────────┘    └──────────┘    └──────────────┘      │
│                                                         │
│  关键不变量:                                            │
│  • Thought 是内部状态更新 + 下一步计划                   │
│  • Action 是外部环境交互（搜索、计算、API）              │
│  • Observation 接地推理，消除幻觉                        │
│  • 循环终止条件: Thought 输出最终答案（无下一步Action）  │
└─────────────────────────────────────────────────────────┘
```

**Few-shot Prompt 构造**：将 6-11 个人工编写的 (Thought, Action, Observation) 序列放在 context 中，模型通过 in-context learning 学会格式。每个示例展示一个完整的推理-行动链，从问题到最终答案。格式关键：每个 Thought 以 "Thought:" 开头、Action 以 "Action:" 开头且后跟结构化参数、Observation 以 "Observation:" 开头——这种格式约定直接成为后来所有 Agent prompt 的 proto-standard。

## 设计决策解剖 ⚖️

| 决策点 | 选择 | 被放弃方案 | 为什么 | 何时失效 |
|--------|------|-----------|--------|---------|
| 推理与行动的粒度 | 每步一次 Thought→Action→Observation，细粒度交替 | 先推理完整链再批量执行（Plan-then-Execute） | 细粒度允许动态调整；行动结果反馈到下一步推理，形成纠错闭环 | 当行动成本极高（如单次 API 调用费时数分钟）时，批量推理更高效 |
| 推理格式 | 自由形式自然语言 Thought | 结构化推理（如 JSON schema 限制推理步骤） | 自然语言保持 LLM 表达灵活性，few-shot 示例足够约束格式 | 当需要程序化解析推理步骤时（如自动验证推理正确性），结构化格式更强 |
| 工具调用格式 | Prompt 内嵌结构化 Action（如 `Search[entity]`） | 独立 tool-use API（如后来的 function calling） | 2022 年尚无标准 tool-use API，prompt 内嵌是唯一可行方案 | 当模型调用复杂工具（多参数、嵌套结构）时，prompt 文本易出错，API 更可靠 |
| 示例构造策略 | 人工编写 6-11 个高质量 few-shot 示例 | 自动从语料库检索相关示例（RAG 式 few-shot） | 人工示例质量可控，确保 Thought 推理逻辑正确且可泛化 | 当任务种类极多时，人工编写覆盖不足；需自动检索或微调替代 |
| 与 CoT-SC 的集成 | ReAct + CoT-SC：将 ReAct 轨迹的多个采样合并，取多数答案 | 只用单次 ReAct 轨迹 | 自一致性提升事实准确性，抵消单次采样的随机错误 | 当推理链差异极大（轨迹间无法对齐比较）时多数投票无意义 |

## 成本与量级 💰
- **训练成本**：无需训练——纯 few-shot prompting（PaLM 540B），推理时每次生成消耗约 2K-4K tokens（含 prompt 示例和交互轨迹）
- **推理成本**：每条 ReAct 轨迹平均 4-7 步交互（取决于任务复杂度），每步约 200-500 tokens。对比 CoT-only（单次生成，无交互成本），ReAct 多 2-4x 的推理开销来自多轮生成
- **最小可行配置**：任意可访问的 LLM API（GPT-3.5+ 或开源 7B+）+ 一个搜索工具（Wikipedia API 即可）——核心是 prompt 格式而非模型规模或工具复杂度
- **标注成本**：论文中 6-11 个 few-shot 示例为人工编写，每个示例约需 5-15 分钟的专家时间（共约 1-2 人时）

## 证据审计 🔬
- **实验公平性**：对比组选择合理（Act-only、CoT-only、CoT-SC、ReAct、ReAct+CoT-SC），所有方法使用相同的 PaLM 540B 基座，消除模型差异。但 few-shot 示例数量不同（CoT 4-6 个 vs ReAct 6-11 个），对 ReAct 可能存在轻微提示优势。
- **最强证据**：HotpotQA F1，ReAct 45.2 vs Act-only 27.4（+65% 相对提升）；ALFWorld 成功率，ReAct 71% vs Act-only 45%（+26 个百分点）；幻觉分析：ReAct 轨迹中事实错误率约为 CoT 的一半（人工标注 50 条轨迹）。三项跨两个任务类型的结果一致，证据链强。
- **最可疑数字及原因**：FEVER 事实核查任务上 ReAct 仅比 Act-only 高 2.5%（60.9 vs 58.4）——提升微小。可能因为 FEVER 的"验证一句话真假"任务不需要多步信息检索，单次搜索基本足够，ReAct 的多步推理优势在此场景下不明显。报告时单独强调 HotpotQA 的 17.8% 提升而淡化 FEVER 的 2.5%，存在选择性报告风险。
- **审稿补充实验**：建议在代码生成（如 HumanEval）和数学推理（如 GSM8K）上补充 ReAct+代码解释器能力，这两类任务是现代 Agent 的主要使用场景但原文未覆盖。
- **最小复现设计**：5 个 Wikipedia 搜索示例作为 few-shot prompt + GPT-3.5-turbo + HotpotQA dev set 50 题——预计 1 小时内可复现核心效应；核心只需实现 Search API 和 ReAct 格式的 prompt 模板。

## 可复用点
- **何时采用**：任何需要 LLM 与外部工具交互的场景——搜索引擎、代码执行、数据库查询、API 调用。当你给 LLM 配备工具时，prompt 中必须包含 Thought 步骤，这是 ReAct 的成本几乎为零的提升（仅多几百 tokens 让模型"想清楚再行动"）。
- **何时规避**：单步即可完成的任务（如翻译、简单问答）不需要 ReAct 循环；延迟敏感的应用（如实时对话）中，多轮 Thought→Action 可能增加不可接受的响应时间。
- **供应商拷问清单**：
  1. "你们的 Agent 框架的 prompt 格式是否包含显式的推理步骤（类似 Thought）？如果有，它与 Action 是如何交错的？"
  2. "你们的 Agent 在行动失败或返回意外结果时，能否根据 Observation 自动调整下一步计划？还是固定执行预设步骤？"
  3. "你们的 Agent 框架支持哪些类型的 Action（搜索、计算、代码执行、API 调用）？新增一个 Action 类型需要改动多少代码？"

## 关联网络 🕸️
- **相关论文**：
  - [[Wiki/论文笔记/05_推理思维链/68_ChainOfThought思维链提示激发大模型推理]] — ReAct 的 Thought 部分是 CoT 在 Agent 环境中的自然延伸，CoT 提供推理基础，ReAct 加上行动闭环
  - [[Wiki/论文笔记/04_Agent技能工具/74_Toolformer语言模型自学使用工具]] — Toolformer 同期从训练侧解决工具使用（自监督微调），ReAct 从推理侧解决（prompt 引导），两篇互补构成 Agent 工具使用的完整图景
  - [[Wiki/论文笔记/06_训练对齐RL/67_WebGPT浏览器辅助问答与人类反馈]] — WebGPT 是 ReAct 在搜索场景的早期原型，用 RL 训练而非 prompt 引导
- **相关概念**：
  - [[Wiki/概念/04_Agent框架/ReAct与Agent推理行动框架]] — 本论文是该概念的直接源头
  - [[Wiki/概念/03_推理与评测/思维链与推理模型]] — ReAct 的 Thought 是 CoT 的特化形式
  - [[Wiki/概念/07_安全对齐/幻觉与接地]] — ReAct 通过 Observation 接地推理，论文证明幻觉率减半
- **冲突/印证**：
  - **印证**：[[Wiki/论文笔记/05_推理思维链/68_ChainOfThought思维链提示激发大模型推理]] — CoT 的"逐步推理"假设与 ReAct 的 Thought 机制一致，CoT 是被动推理，ReAct 是主动推理+行动，两篇共同支撑"推理步骤拆分提升 LLM 表现"的核心结论
  - **待冲突验证**：ReAct 的 few-shot 范式 vs Toolformer 的训练范式。在模型规模缩小时（如 1B-3B），few-shot ReAct 可能失效而 Toolformer 的微调方法更稳定——原文未在小型模型上验证此点

## 动手练习 💻
**练习目标**：手写一个最小 ReAct 循环（Thought → Action → Observation），带一个模拟搜索工具，理解 Agent 推理-行动交替的核心机制。

```python
"""
最小ReAct Agent：Thought → Action → Observation 循环
带一个模拟 Wikipedia 搜索工具
运行: python react_mini.py
"""

import re
from typing import Callable

# ====== 工具定义 ======
def search_wikipedia(query: str) -> str:
    """模拟 Wikipedia 搜索工具（生产环境替换为真实 API）"""
    # 模拟知识库：现实中这里调用 Wikipedia API 或向量数据库
    knowledge = {
        "albert einstein": "Albert Einstein (1879-1955), German physicist, "
                           "developed theory of relativity, Nobel Prize 1921.",
        "relativity": "Theory of relativity: special relativity (1905) and "
                      "general relativity (1915) by Einstein.",
        "nobel prize": "Nobel Prize established 1895 by Alfred Nobel, "
                       "awarded in Physics, Chemistry, Medicine, Literature, Peace.",
    }
    query_lower = query.lower().strip()
    for key, value in knowledge.items():
        if query_lower in key or key in query_lower:
            return value
    return f"No results found for '{query}'."

# ====== ReAct Agent 核心循环 ======
def react_agent(question: str, tools: dict[str, Callable],
                llm_think: Callable) -> str:
    """
    ReAct 循环：Thought → Action → Observation → Thought → ...
    直到 Thought 输出最终答案（不再需要 Action）
    """
    context = f"Question: {question}\n"  # 累积上下文（Thought+Action+Observation历史）
    max_steps = 5  # 安全阀：最多 5 步防止无限循环

    for step in range(max_steps):
        # ---- Step 1: Thought（推理下一步做什么）----
        thought = llm_think(context)
        print(f"[Thought {step+1}] {thought}")

        # ---- Step 2: 判断 Thought 中是否包含最终答案 ----
        # 如果 Thought 以 "Answer:" 开头，表示推理完成
        if thought.strip().startswith("Answer:"):
            return thought.strip().replace("Answer:", "").strip()

        # ---- Step 3: 解析 Action（从 Thought 中提取工具调用）----
        # 预期格式: Action: ToolName[argument]
        action_match = re.search(r'Action:\s*(\w+)\[([^\]]+)\]', thought)
        if not action_match:
            print(f"  [ERROR] 无法解析 Action，请检查 Thought 格式")
            continue
        tool_name, tool_arg = action_match.group(1), action_match.group(2)

        # ---- Step 4: Observation（执行工具并获取结果）----
        if tool_name not in tools:
            observation = f"Error: Unknown tool '{tool_name}'"
        else:
            observation = tools[tool_name](tool_arg)
        print(f"  → Action: {tool_name}[{tool_arg}]")
        print(f"  ← Observation: {observation}")

        # ---- Step 5: 将 Observation 追加到上下文，供下一轮 Thought 参考 ----
        context += (f"\nAction: {tool_name}[{tool_arg}]\n"
                    f"Observation: {observation}\n")

    return "Failed to answer within step limit."

# ====== 模拟 LLM 的 Thought 生成（简化版规则引擎） ======
def simple_llm_think(context: str) -> str:
    """
    生产环境中这里替换为真实的 LLM API 调用。
    本简化版用规则匹配演示 ReAct 循环的完整流程。
    """
    step_count = context.count("Observation:")

    if step_count == 0:
        # 第一步：还没有任何信息，需要搜索
        return ("Thought: I need to search for key information in the question. "
                "Action: Search[Albert Einstein]")
    elif step_count == 1:
        # 第二步：有了基础信息，检查是否需要更多搜索
        if "nobel" in context.lower() or "prize" in context.lower():
            return ("Thought: I have Einstein's basic info and Nobel Prize context. "
                    "I can now answer. "
                    "Answer: Albert Einstein won the Nobel Prize in Physics in 1921.")
        return ("Thought: I need more information about related topics. "
                "Action: Search[Nobel Prize]")
    else:
        # 通用兜底：尝试综合已有信息给出答案
        return ("Thought: I have sufficient information to answer. "
                "Answer: Based on available information, Albert Einstein "
                "won the Nobel Prize in Physics in 1921 for his contributions "
                "to theoretical physics.")

# ====== 运行演示 ======
if __name__ == "__main__":
    tools = {"Search": search_wikipedia}
    question = "What Nobel Prize did Albert Einstein win?"
    print(f"{'='*60}")
    print(f"Question: {question}")
    print(f"{'='*60}")
    answer = react_agent(question, tools, simple_llm_think)
    print(f"{'='*60}")
    print(f"Final Answer: {answer}")
```

## 自测三层 🎓
- **L1 复述**：ReAct 循环的三个核心组成部分是什么？它们之间如何传递信息？
- **L2 解释**：为什么 ReAct 的幻觉率约为纯 CoT 的一半？Observation 步骤在机制层面如何"接地"推理？用一个具体例子说明。
- **L3 应用**：如果你要给一个 ReAct Agent 加上"代码执行"工具（如 Python 解释器），Action 和 Observation 的格式该如何设计？需要额外处理哪些安全风险？如何在 Thought 步骤中引导模型判断何时该写代码而非搜索？

📅 知识时间锚：2022（ReAct 提出）→ 2023（LangChain 产品化 Agent）→ 2024（OpenAI Function Calling 标准化）
