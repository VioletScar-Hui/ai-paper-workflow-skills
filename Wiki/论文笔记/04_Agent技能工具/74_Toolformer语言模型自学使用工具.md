---
tags: [论文笔记, Toolformer, 工具使用, 自学习, 基础论文, Meta]
paper_id: "74"
笔记层级: 骨干
复核日期: 2026-07-04
---

# Toolformer：语言模型可以自学使用工具

📄 **原文 PDF**：[[RAW/74 - Toolformer - Language Models Can Teach Themselves to Use Tools.pdf]]

## PM 速判
> LLM 自主工具使用的奠基论文。Toolformer 让 6.7B 的 GPT-J 通过自监督学习学会了"什么时候该调用计算器、搜索引擎、翻译 API"——无需人工标注调用时机，且数学准确率从 29.4% 跳到 44.0%，超过 GPT-3 175B 三倍。这是 OpenAI Function Calling 和 Claude Tool Use 的学术前身。

## 双层费曼 🗣️
> **给CEO**：之前的 LLM 像一个只看过书但从未用过手机的人——它不会查资料、不会用计算器。要让模型学会"用工具"，传统做法是雇人标注几万条"在这个位置该调计算器"的示例，成本极其高昂。Toolformer 的方法是：让模型自己在海量文本中尝试插入工具调用，然后只保留那些"真的帮助预测下文"的调用——像一个自学的学生，试了 100 种用法，只留下真正有用那 20 种。结果：6.7B 小模型 + 工具能打败 175B 大模型不用工具。
> **给工程师**：Toolformer 的核心创新是自监督数据标注管线：(1) 在语料库文本中，用 few-shot 提示让模型生成潜在的 API 调用候选（如 `[Calculator(123+456) → 579]`）；(2) 实际执行 API 获取返回值；(3) 计算插入 API 调用前后该位置后续 token 的困惑度变化 ΔL = L(无调用) - L(有调用)，只保留 ΔL > τ（τ 为阈值）的调用——即"工具调用使模型更容易预测下文"。筛选后的文本直接用于 GPT-J 6.7B 标准 LM 微调。训练后模型在推理时自主决定何时生成 `[Tool(args)]` token，解码器遇到 `→` 时暂停、执行 API、将结果插回继续生成。

## 问题域定位 🎯
- **回应什么根本约束？** LLM 的预训练将世界知识压缩进参数，但：(1) 参数知识有截止日期，无法获取实时信息；(2) 精确计算是参数模型的系统性弱点（token 形式的加法与数值计算本质不同）；(3) 人工标注"工具调用时机"成本极高且不可扩展。核心约束是：如何让模型自己学会"何时求助外部工具"而不依赖人类标注。
- **之前卡在哪？** 业界方案要么靠 few-shot prompt 引导（如 ReAct，依赖提示工程且不稳定），要么靠人工标注微调（成本不可扩展）。自监督工具学习这条线是空白——没人系统性地设计过"让模型在无监督语料中自学工具调用时机"的算法。
- **开启/关闭了哪条路线？** **开启了**：自监督工具学习路线（后续的 Gorilla、Berkeley Function Calling Leaderboard 均受益于这一范式）。**部分关闭了**：纯人工标注工具调用时机作为唯一方案的路线——Toolformer 证明自监督的质量在数学任务上已超过人工标注等价物。

## 核心机制

```
┌──────────────────────────────────────────────────────────────────┐
│              Toolformer 自监督工具学习管线                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Phase 1: 候选生成（Sampling）                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ 输入文本: "The population of Paris in 2021 was "            │  │
│  │          ↓ few-shot prompt 引导                            │  │
│  │ 候选: "[Wikipedia(Paris population 2021)]"  ← 模型生成      │  │
│  │       "[QA(Paris population)]"                               │  │
│  │       "[Calculator(2,161,000 + 0)]"   ← 可能生成多个候选     │  │
│  └────────────────────────────────────────────────────────────┘  │
│                          ↓                                       │
│  Phase 2: API 执行 + 结果插入                                     │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ 执行 Wikipedia API → 返回: "2,161,000 (2021 est.)"          │  │
│  │ 插入后文本:                                                  │  │
│  │   "The population of Paris in 2021 was "                    │  │
│  │   "[Wikipedia(Paris) → 2,161,000] 2,161,000."              │  │
│  └────────────────────────────────────────────────────────────┘  │
│                          ↓                                       │
│  Phase 3: 困惑度筛选（Filtering）                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ 计算: L_i = -log P(next_tokens | prefix_without_API)       │  │
│  │       L'_i = -log P(next_tokens | prefix_with_API_result)   │  │
│  │ 保留条件: L_i - L'_i > τ    (τ ≈ 1.0)                       │  │
│  │           ↓                                                 │  │
│  │  含义: API 调用使后续 token 预测损失降低 > τ，才保留         │  │
│  │          否则丢弃（工具调用无帮助 = 噪音）                    │  │
│  └────────────────────────────────────────────────────────────┘  │
│                          ↓                                       │
│  Phase 4: 微调（Fine-tuning）                                     │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ 筛选后文本 → 标准 LM loss 微调 GPT-J 6.7B                    │  │
│  │推理时: 模型自主生成 [Tool(args)] → 解析 → 执行 API → 插回   │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  工具集: Calculator, Wikipedia Search, MT (翻译),                │
│          Calendar (日历), QA (问答系统)                           │
└──────────────────────────────────────────────────────────────────┘
```

## 设计决策解剖 ⚖️

| 决策点 | 选择 | 被放弃方案 | 为什么 | 何时失效 |
|--------|------|-----------|--------|---------|
| 标注方法 | 自监督：模型自己生成候选 + 困惑度筛选 | 人工标注调用时机（传统方案） | 人工标注不可扩展且质量波动大；自监督让模型用自己的"需要"决定调用 | 当模型能力极弱（<1B）时生成候选质量差，自监督信号不足以筛选有效调用 |
| 筛选标准 | 困惑度降低 ΔL > τ（τ≈1.0） | 固定概率采样（保留所有候选/top-k） | 仅困惑度降低是客观的"有用性"信号，避免引入人工 bias | 当 API 返回值总是长篇文本时（如搜索返回 10 篇文档），困惑度变化可能仅反映长度而非有用性 |
| 工具调用格式 | 文本内嵌 `[Tool(args) → result]` | 独立 tool-use API（如 function calling schema） | 2023 年无标准 API，文本内嵌使模型在 LM 目标下自然学习调用格式 | 当工具参数复杂（嵌套 JSON、多字段）时，纯文本格式易解析错误，结构化 API 更强 |
| 工具集规模 | 5 类预定义工具，固定集合 | 动态可扩展工具（如基于 API 描述自动发现） | 固定集确保训练稳定性，避免模型学习过于稀疏的调用模式 | 当需要快速集成新工具（如新增 Slack API）时，需重新训练而非即插即用 |
| 链式调用 | 不支持——单次 API 调用的独立决策 | 多步链式调用（ReAct 式循环） | 简化训练目标：每次微调样本只有 0 或 1 个工具调用，loss 无歧义 | 对于需要多步搜索（如"查 A→用结果查 B"）的任务，单次调用不够 |

## 成本与量级 💰
- **训练成本**：候选生成阶段需对整个语料库（CCNet，筛选后约 15 亿 token）采样执行 API——每个候选位置都需实际调用外部 API，这是主要计算开销（非 GPU 训练而是 API 调用成本）。微调阶段：GPT-J 6.7B 在筛选后数据上做标准 LM 微调，约需 8×A100 GPU 几天。
- **推理成本**：推理时模型自主生成工具调用 token，遇到 `→` 时暂停生成、执行 API、插回结果继续——每步 API 调用的延迟取决于外部工具（搜索 ~200ms、计算 ~1ms），text generation 部分与普通 LLM 推理相同。
- **最小可行配置**：GPT-J 6.7B（或 LLaMA 7B）微调 + 1 个工具（Calculator 即可演示核心效应）+ CCNet 1M 文档子集用于候选生成，约 4×A100 可在一周内完成全流程复现。
- **API 调用成本**：候选生成阶段对语料库中每个潜在位置执行 API 调用——若采样 1M 个位置，每个调用约 $0.001（搜索 API），总 API 成本约 $1,000，与人工标注（$0.1-1/样本 × 1M = $100K-1M）相比低 2-3 个数量级。

## 证据审计 🔬
- **实验公平性**：对比基线的选择具有说服力——GPT-J 6.7B（同架构无工具）、OPT 66B（10x 参数量无工具）、GPT-3 175B（26x 参数量无工具）——证明 Toolformer 用 6.7B+工具超过了裸奔的大得多的模型，不是简单的"大模型更好"。但不同模型架构（GPT-J vs OPT vs GPT-3）引入了混淆变量。
- **最强证据**：ASDiv 数学准确率：Toolformer 6.7B 44.0% vs GPT-3 175B 14.0%（3.1x 提升，且小 26 倍的模型）；LAMA 知识问答：含 Wikipedia 工具的 Toolformer 在闭卷不可回答的事实问题上大幅超越所有无工具基线；TempLAMA（时间敏感问答）：模型自主决定在涉及 2020+ 事件时调用搜索，而不在历史事实上调用——证明了工具调用的"时机选择性"而非"盲目调用"。
- **最可疑数字及原因**：工具调用频率较低——在某些需要工具的任务中约 30-40% 的案例模型选择不调用工具。可能原因：(1) τ 阈值设置偏保守，过滤掉了"略有帮助但不显著"的调用；(2) GPT-J 6.7B 的基础能力本身不足以识别所有需要工具的场景。降低 τ 或改用更大基座模型可能改善，但原文未做此消融实验。
- **审稿补充实验**：建议补充消融：(1) 困惑度筛选 vs 随机保留 50% 候选的对比（量化筛选步骤的贡献）；(2) 不同 τ 值（0.5, 1.0, 1.5, 2.0）对工具调用频率和准确率的 trade-off 曲线；(3) 在更大基座模型（如 LLaMA 13B/33B）上的结果，验证自监督工具学习的 scaling 特性。
- **最小复现设计**：GPT-J 6.7B + Calculator 单工具 + ASDiv 数学数据集 + 困惑度筛选管线（τ=1.0）——预计 3 天内可在 4×A100 上复现核心结果。关键挑战在候选生成阶段的 API 执行效率（需批量调用）。

## 可复用点
- **何时采用**：当你需要让 LLM 学会调用 API/工具但缺乏人工标注数据时，Toolformer 的自监督管线是最直接的方案。特别适合工具种类较少（2-5 种）且每种工具的调用格式固定、返回值简洁的场景（计算器、翻译、搜索等）。如果你的任务是训练而非 prompt-based 的工具使用，这是首选范式。
- **何时规避**：当工具调用需要复杂的链式推理（多步工具协同）时——Toolformer 不支持 multi-step tool use。当需要快速集成新工具时——训练管线需要重新采样、筛选、微调，周期较长。当你的模型已经足够强（GPT-4 级别）且仅需 few-shot prompt 即可稳定调用工具时，训练方案的成本收益比不划算。
- **供应商拷问清单**：
  1. "你们的模型工具使用能力是通过训练（如 Toolformer 式自监督）还是 prompt（如 ReAct 式 few-shot）实现的？如果工具列表新增一个 API，需要重新训练吗？"
  2. "你们的工具调用筛选管线用什么指标判断'工具调用是否有用'？是否会像 Toolformer 一样面临调用频率过低的问题？"
  3. "当工具返回错误或无关结果时，模型是否会被训练去忽略错误返回值（而非盲从工具输出）？你们的训练数据中是否包含了工具失败案例？"

## 关联网络 🕸️
- **相关论文**：
  - [[Wiki/论文笔记/05_推理思维链/73_ReAct推理与行动的协同]] — ReAct 用 prompt 引导工具使用（推理时范式），Toolformer 用训练植入工具使用（训练时范式），两者分别解决了"如何让模型用工具"的 prompt 侧和 training 侧问题。ReAct 优势在灵活性和动态推理，Toolformer 优势在小模型上的稳定性。
  - [[Wiki/论文笔记/06_训练对齐RL/67_WebGPT浏览器辅助问答与人类反馈]] — WebGPT 用人工标注 + RL 训练搜索行为，Toolformer 用自监督训练多种工具使用——前者需要昂贵的人工过程监督，后者仅需无标签语料。
- **相关概念**：
  - [[Wiki/概念/04_Agent框架/ReAct与Agent推理行动框架]] — Toolformer 的工具调用能力是 ReAct 框架中 Action 步骤的训练侧支撑
  - [[Wiki/概念/02_训练方法/合成数据智能体]] — Toolformer 的自监督标注管线是合成工具调用训练数据的早期经典范式
  - [[Wiki/概念/04_Agent框架/函数调用与工具使用]] — 本论文是该概念的核心奠基工作
- **冲突/印证**：
  - **印证**：[[Wiki/论文笔记/05_推理思维链/73_ReAct推理与行动的协同]] — 工具使用 + 推理的协同价值在两篇论文中交叉验证：ReAct 从 prompt 侧证明 Thought 能引导工具的合理使用，Toolformer 从训练侧证明模型可以学会"何时该调工具"。两篇独立工作得出相容结论，增强了"工具使用是 LLM 必要能力"论点的可信度。
  - **待冲突验证**：当模型规模进一步扩大（如 70B+），Toolformer 的自监督训练是否仍有必要？GPT-4 级别的模型通过 few-shot prompt 就能稳定调用工具——规模是否"买断"了工具学习的需求？这与 Toolformer 的核心论点（小模型需要训练才能用好工具）是正交而非矛盾，但需额外的 scaling 实验区分。

## 动手练习 💻
**练习目标**：实现一个简易的 function calling 系统——解析 LLM 输出的工具调用指令，执行对应 Python 函数，将结果插回上下文。这是理解 Toolformer 推理阶段和现代 Function Calling 原理的最小实现。

```python
"""
简易 Function Calling 实现
解析 LLM 输出的工具调用 → 执行 → 返回结果
运行: python tool_calling_mini.py
"""

import re
import json
import math
from typing import Any, Callable

# ====== 工具注册表 ======
def calculator(expression: str) -> str:
    """安全的数学表达式求值（仅支持基础运算）"""
    # 白名单过滤：只允许数字、运算符、括号、空格
    allowed = set("0123456789+-*/().%^ ")
    if not all(c in allowed for c in expression):
        return f"Error: unsafe expression '{expression}'"
    try:
        # 替换 ^ 为 **（Python 幂运算）
        safe_expr = expression.replace("^", "**")
        result = eval(safe_expr, {"__builtins__": {}}, {"sqrt": math.sqrt})
        return str(result)
    except Exception as e:
        return f"Error: {e}"

def get_weather(city: str) -> str:
    """模拟天气查询（生产环境替换为真实 API）"""
    weather_db = {
        "beijing": "Beijing: Sunny, 28°C, humidity 45%",
        "shanghai": "Shanghai: Cloudy, 25°C, humidity 70%",
        "tokyo": "Tokyo: Rainy, 22°C, humidity 85%",
    }
    return weather_db.get(city.lower(),
                          f"No weather data for '{city}'")

# 工具注册表：名称 → (函数, 参数说明)
TOOLS: dict[str, tuple[Callable, str]] = {
    "calculator": (calculator, "expression: str"),
    "get_weather": (get_weather, "city: str"),
}

# ====== 工具调用解析器 ======
def parse_tool_call(llm_output: str) -> list[dict]:
    """
    从 LLM 输出中解析工具调用。
    支持两种格式：
      格式A: <tool_call>{"name": "xxx", "args": {...}}</tool_call>
      格式B: [TOOL: calculator("2+3")]
    返回: [{"name": "calculator", "args": {"expression": "2+3"}}, ...]
    """
    calls = []
    # 格式A: JSON 风格（类似 OpenAI function calling）
    pattern_a = r'<tool_call>(.*?)</tool_call>'
    for match in re.finditer(pattern_a, llm_output, re.DOTALL):
        try:
            call_data = json.loads(match.group(1))
            calls.append(call_data)
        except json.JSONDecodeError:
            continue

    # 格式B: 简化风格 [TOOL: name(args)]
    pattern_b = r'\[TOOL:\s*(\w+)\((.*?)\)\]'
    for match in re.finditer(pattern_b, llm_output):
        name, args_str = match.group(1), match.group(2)
        # 简单参数解析：按逗号分割关键字参数
        args = {}
        for part in args_str.split(","):
            part = part.strip()
            if "=" in part:
                k, v = part.split("=", 1)
                args[k.strip()] = v.strip().strip('"\'')
            else:
                # 位置参数 → 用注册表中的参数名映射
                param_names = TOOLS.get(name, (None, ""))[1].split(", ")
                idx = len(args)
                if idx < len(param_names):
                    args[param_names[idx].split(":")[0]] = part.strip('"\'')
        calls.append({"name": name, "args": args})

    return calls

# ====== 工具执行引擎 ======
def execute_tools(calls: list[dict]) -> dict[str, str]:
    """执行解析出的工具调用，返回 {call_id: result}"""
    results = {}
    for i, call in enumerate(calls):
        name = call["name"]
        args = call["args"]
        if name not in TOOLS:
            results[f"call_{i}"] = f"Error: Unknown tool '{name}'"
            continue
        func, _ = TOOLS[name]
        try:
            result = func(**args)
            results[f"call_{i}"] = str(result)
        except Exception as e:
            results[f"call_{i}"] = f"Error executing {name}: {e}"
    return results

# ====== 完整 function calling 流程 ======
def process_with_tools(user_query: str, llm_response: str) -> str:
    """
    处理一次 LLM 响应的完整 tool calling 流程：
    1. 解析工具调用
    2. 执行工具
    3. 将结果格式化插回上下文
    """
    calls = parse_tool_call(llm_response)
    if not calls:
        return llm_response  # 无工具调用，直接返回 LLM 输出

    results = execute_tools(calls)

    # 格式化工具结果为 LLM 可读格式
    result_text = "\n[Tool Results]\n"
    for i, call in enumerate(calls):
        key = f"call_{i}"
        result_text += f"  {call['name']}({call['args']}) → {results[key]}\n"

    return result_text

# ====== 演示 ======
if __name__ == "__main__":
    # 模拟 LLM 输出（包含工具调用）
    llm_outputs = [
        '我需要先计算这个表达式。\n<tool_call>{"name": "calculator", "args": {"expression": "123 * 456 + 789"}}</tool_call>',
        '让我查一下北京和东京的天气。\n[TOOL: get_weather(city="beijing")]\n[TOOL: get_weather(city="tokyo")]',
    ]

    for i, output in enumerate(llm_outputs, 1):
        print(f"{'='*60}")
        print(f"Example {i}:")
        print(f"LLM Output: {output[:80]}...")
        result = process_with_tools("demo query", output)
        print(f"Results:\n{result}")

    # 边界情况测试：不支持的工具
    print(f"{'='*60}")
    print("Edge Case: Unknown tool")
    unknown = '<tool_call>{"name": "send_email", "args": {"to": "a@b.com"}}</tool_call>'
    print(process_with_tools("test", unknown))
```

## 自测三层 🎓
- **L1 复述**：Toolformer 的自监督标注管线包含哪四个阶段？每个阶段的输入和输出分别是什么？
- **L2 解释**：困惑度筛选步骤中的公式 ΔL = L(无调用) - L(有调用) 为什么能衡量"工具调用是否有用"？如果 API 返回了错误结果，这个筛选会如何表现？τ 阈值的选择在什么 trade-off？
- **L3 应用**：假设你要用 Toolformer 的管线训练一个 LLM 使用"SQL 数据库查询"工具。你需要修改哪些步骤？SQL 查询和 Calculator 在困惑度筛选阶段有什么本质不同？如何设计筛选标准来避免模型过度依赖数据库（即所有问题都查库而非用参数知识）？

📅 知识时间锚：2023-02（Toolformer 发布）→ 2023-06（OpenAI Function Calling beta）→ 2023-11（Claude Tool Use）→ 2024（Gorilla/BFCL 标准化评测）
