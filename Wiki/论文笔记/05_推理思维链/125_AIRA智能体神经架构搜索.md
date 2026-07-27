---
tags: [论文笔记, 神经架构搜索, 递归自我改进, AI科研, Meta-FAIR]
paper_id: "125"
filename: "125 - Agentic Discovery of Neural Architectures - AIRA-Compose and AIRA-Design.pdf"
authors: "Alberto Pepe, Chien-Yu Lin et al. (FAIR at Meta)"
year: 2026
笔记层级: 骨干
复核日期: 2026-07-04
---

# AIRA：智能体自主发现神经架构

📄 **原文 PDF**：[[RAW/125 - Agentic Discovery of Neural Architectures - AIRA-Compose and AIRA-Design.pdf]]

## PM 速判 > 一句话

> Meta FAIR 用 11 个并行 LLM 智能体（24h H200 预算，Claude Opus 4.6/Gemini 3 Pro 作推理引擎），在预定义计算原语（Attention/MLP/Mamba2）的 16 位置排列空间中自主搜索出 14 个新架构，AIRAhybrid-D 准确率超 Llama 3.2 +3.8%，AIRAformer-C 扩展效率比 Llama 3.2 快 54%——"AI 设计 AI"（递归自我改进 RSI）完成早期可行性验证。

## 双层费曼

> **给 CEO**：Meta 让 AI 自己设计更好的 AI 模型。AI 在 24 小时内找出的架构，比人类工程师精心设计的 Llama 3.2 更准更快——未来模型升级的节奏会从"人类思考→实验"变成"AI 自主搜索→人类验证"，这是 AI 自我改进的第一步。

> **给工程师**：AIRA-Compose 将 NAS 问题转化为 AIRS-Bench 标准任务，11 个 Agent 通过 AIRA-dojo 框架的 One-Shot/Greedy 脚手架，在 3 计算原语（多头注意力 M、MLP M、Mamba2 Mb）的 16 层排列空间中迭代设计候选架构。小规模（百万参数）快速筛选后，top 候选通过 Composer 的聚合+外推（stretching/stacking）扩到 350M/1B/3B。AIRA-Design 则让最多 20 个 Agent 通过 Draft→Debug→Improve→Analyze 四操作符直接编写 model.py/train.py，在 LRA 和 Autoresearch 基准上优化。关键发现：AI 搜索在硬件延迟-验证损失的 Pareto 前沿上推进了人类此前建立的最优边界。

## 问题域定位

**根本约束**：神经网络架构设计长期依赖人工直觉——Transformer 的 1:1 Attention/MLP 交替排列是人设计的，替代方案探索受限于人类搜索的有限时间和认知偏差。

**之前卡点**：传统 NAS（贝叶斯优化、进化算法）在组合爆炸的搜索空间中效率低，且无法利用 LLM 积累的领域知识（论文、代码模式）来指导搜索方向。

**路线开启**：AIRA 证明了 LLM Agent 可以利用其预训练中的架构设计知识来引导搜索——不是随机突变，而是"有教育的假设生成"。这开启了 RSI（递归自我改进）的技术路径：用 AI 智能体设计更好的 AI 模型，下一代模型反过来成为更强的智能体引擎。

**路线关闭**：当前搜索空间仍限定在预定义计算原语的排列组合（非开放式架构创新）；AIRA-Design 的开放代码编写任务中 Agent 仍远低于人类 SOTA（NS < 0.26），说明 Agent 在真正的科学创新层面仍有巨大差距。

## 核心机制

```
┌─────────────────────────────────────────────────────────────┐
│                    AIRA 双框架架构                           │
├─────────────────────────────────────────────────────────────┤
│  AIRA-Compose（高层组合搜索）          AIRA-Design（低层机制实现）│
│  ┌─────────────────────────┐        ┌─────────────────────┐ │
│  │ 搜索空间：3个计算原语       │        │ 搜索空间：自由编写代码    │ │
│  │ M=mh-attention           │        │ model.py + train.py  │ │
│  │ mA=MLP                   │        │ 目标：LRA长序列推理    │ │
│  │ Mb=Mamba2 SSM            │        │       Autoresearch优化│ │
│  │ 16位置排列（3^16组合）     │        └─────────────────────┘ │
│  └─────────────────────────┘                                 │
│         ↓                                      ↓             │
│  AIRS-Bench 任务格式 ←→ AIRA-dojo 执行环境                    │
│  project_description.md / evaluate.py / metadata.yaml         │
│         ↓                                      ↓             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  脚手架（Scaffold）: One-Shot / Greedy (MCTS-like)    │   │
│  │  四操作符: Draft → Debug → Improve → Analyze         │   │
│  │  推理引擎: Claude Opus 4.6, Gemini 3 Pro, GPT-5 等   │   │
│  │  预算: 24h/run, 最多500 steps, 1×H200 GPU            │   │
│  └──────────────────────────────────────────────────────┘   │
│         ↓                                      ↓             │
│  小规模筛选(百万参数)                    直接评估              │
│  → Composer 聚合+外推                    LRA: accuracy       │
│  → 350M/1B/3B 全量预训练                 Autoresearch: BPB   │
│    isoToken (37.5B) + isoFLOP (2e19~4e20)                    │
└─────────────────────────────────────────────────────────────┘
```

**搜索流程（以 AIRA-Compose Greedy 为例）**：
1. Agent 读取 project_description.md 理解任务
2. Draft 生成初始 5 个候选架构（各为16个原语排列字符串）
3. evaluate.py 独立训练并返回 validation fitness
4. Improve 以最高分节点为父节点，基于推理生成新候选
5. Analyze 检测 OOM/语法错误/原语数量错误
6. Debug 修复问题后重新提交
7. 迭代数百个节点，覆盖所有 Agent 的所有 seed
8. 汇总所有提交 → 聚合（layer-wise clustering）→ 外推（stretch/stack）

## 设计决策解剖

| 决策点 | 选择 | 被放弃方案 | 为什么 | 何时失效 |
|--------|------|-----------|--------|---------|
| 搜索空间粒度 | 16层预定义原语排列（3^16 ≈ 43M） | 完全开放的代码编写（如AIRA-Design） | Composer 已验证16层小模型与大规模性能相关；结构化输出使VSR接近100% | 当需要发现全新的计算原语（不同于 Attention/MLP/Mamba2）时，此空间无法表达 |
| 脚手架策略 | Greedy tree search（5个初始候选+迭代improve） | 纯随机搜索 / MCTS | Greedy 平衡了探索广度与计算成本；MCTS 在无明确rollout环境的代码生成中实现复杂 | 当任务需要极端探索多样性时（如开放式科学发现），Greedy 的 exploitation bias 过强 |
| 推理模型选择 | 非SOTA模型用于Compose（GPT-4o/CWM/o3-mini）；SOTA模型用于Design | 全部用最强模型 | Compose 只需格式化字符串映射到预定义块；Design需要写完整代码——模型能力要求不同 | 如果预定义原语本身需要模型理解其数学性质才能有效排列（如Mamba的序列长度特性），弱模型可能错过非直觉组合 |
| 聚合外推方式 | Layer-wise clustering → stretch/stack | 直接选最高分单架构外推 | 单seed得分有噪声（proxy training的方差），聚合平滑了过拟合 | 当小规模和大规模的相关性被聚合破坏时（某些精细结构被cluster投票抹去） |
| 计算预算分配 | 24h/run × 单H200（Compose）；5min wall-clock（Autoresearch） | 更长训练预算 | 在固定总预算下，多跑不同seed比单seed长时间训练获得更多多样性 | 当某些架构需要更长训练才能展现优势时（如需要warmup的复杂attention），短预算会系统性低估它们 |
| 任务标准化 | AIRS-Bench 格式（project_description + evaluate.py + metadata.yaml） | 自建任务框架 | 可复用、可对比、可与AIRS-Bench其他科研任务统一排名 | 当任务需求超出AIRS-Bench的{problem, dataset, metric}三元组抽象时 |

## 成本与量级

| 维度 | 数据 |
|------|------|
| AIRA-Compose 搜索 | 2-prim: 150 runs (50 greedy + 100 one-shot) × 3 datasets = 450 runs；3-prim: 170 runs；共搜索 2,307+2,248=4,555 个独特架构；覆盖搜索空间 3.17%（2-prim）和 0.0052%（3-prim） |
| 单 run 资源 | 24h × 1 H200 GPU（BabiStories/DCLM 60h）；每 run 100-200 步（Compose），数百步（Design） |
| 大规模验证 | 350M: 8×H200；1B/3B: 16×H200；isoToken (37.5B tokens ≈ 38 TPP)；isoFLOP 120+165=285 次全量实验 |
| AIRA-Design 规模 | 20 agents × 12 models；greedy: 720 runs + one-shot: 960 runs = 1,680 runs 总计 |
| 推理模型 API 成本 | 未公开，但 greedy 模式每 run 数十到数百次 LLM 调用（每步一次 Draft/Improve/Debug/Analyze） |
| 总估算 | 数百 H200-GPU-天（搜索+验证）；实际在 Meta FAIR 集群上运行 |

## 证据审计

**实验公平性**：强。所有架构使用相同的 Composer 训练管线（优化器、超参数、token budget）；evaluate.py 是独立评分脚本，Agent 在搜索阶段无法访问测试集；isoFLOP 使用 5 个 FLOPs 预算 × 3 个参数规模统一对比。

**最强证据**：
- AIRAformer-D Stretched val loss 2.734 vs Llama 3.2 2.815（isoToken 1B scale，3 seed 平均，低方差）：+2.4% 平均零样本准确率。这并非 cherry-pick——6 个 AIRAformer 全部超过 Llama 3.2。
- AIRAhybrid-D Stretched val loss 2.719 vs Nemotron-2 Approx 2.732：在 attention-MLP-Mamba 三原语空间超越人工设计的 Nemotron-2 近似版。同时 AIRAhybrid-D (St.) 在延迟-损失 Pareto 前沿上推进了此前 Composer 建立的最优边界（Figure 7）。
- AIRAformer-C Stacked 的 isoFLOP 斜率 Δq = -0.34 vs Llama 3.2 Δq = -0.63（注意：q 越小=斜率越陡=扩展效率越高）：AI 找到的架构不仅在某一点更好，而且在计算增加时改进更快——这是工业部署最关心的性质。

**最可疑数字及原因**：
- "扩展效率快 54%/71%" 是基于 isoFLOP 最优前沿斜率的推导，而非直接测量"用 54% 更少的钱达到相同性能"。实际部署中的 batch size、序列长度、硬件利用率差异可能缩小或放大这个数字。
- Nemotron-H/Nemotron-2 是 "approximated" 版本（MoE 替换为 MLP），与原始 Nemotron 的性能关系不完全等价——AI 超越的是"近似版"而非真实 Nemotron。
- 2-prim 和 3-prim 空间使用了不同的数据集（MAD/BabiStories/DCLM），两个空间的结果不能直接对比。

**审稿补充**：LRA 任务中 Agent 设计的最优架构准确率在人类 SOTA 的 2.3pp（Retrieval 82%）和 2.6pp（Text 91%）以内——但这只是 "best test accuracy during exploration"，有效提交率（VSR）在较弱模型上低于 10%。One-shot Agent 产生 0 个有效提交（单轮代码生成对机械设计完全不够）。

**最小复现**：AIRA-Compose 需 Composer 代码库 + H200 GPU 集群；最小可行复现为 2-prim 空间 + 单 Agent + One-Shot scaffold + MAD 数据集，约 24 GPU-小时。AIRA-Design 最小复现约为单 LRA 任务（Text） + Greedy scaffold，约 24 GPU-小时。

## 可复用点 + 供应商拷问清单

**可复用点**：
1. 两阶段搜索策略（小规模快速筛选 + 大规模验证）可迁移到任何需要昂贵评估的 Agent 优化任务
2. AIRA-dojo 四操作符（Draft/Debug/Improve/Analyze）是代码生成 Agent 的标准循环模式
3. AIRS-Bench 任务格式（project_description.md + evaluate.py + metadata.yaml）作为 Agent 科研任务的标准化模板
4. "选弱模型做结构简单任务，选强模型做代码生成任务"的模型分配策略

**供应商拷问清单**（评估 AI 科研 Agent 平台时）：
- [ ] 搜索空间是预定义原语的排列，还是开放式代码生成？前者 VSR 高但创新上限低
- [ ] 小规模 proxy 和大规模最终性能的 rank correlation 是多少？没有这个就无法信赖快速筛选
- [ ] 如何处理 OOM 和数值不稳定？这是 AIRA-Design 中最常见的失败模式
- [ ] 是否支持多模型并行搜索（ensemble of agents）？单模型的搜索多样性有限
- [ ] 有效提交率（VSR）是多少？100% 可能意味着任务太简单，<20% 意味着 Agent 在做无效探索

## 关联网络

- [[Wiki/论文笔记/13_评测科研/129_NanoGPT-Bench自主研究Agent评测]] -- 互补：AIRA 在有约束空间超越人类，NanoGPT-Bench 在开放任务中 Agent 仍大幅落后
- [[Wiki/论文笔记/02_前沿模型报告/101_LongCat-Flash-Thinking技术报告]] -- Mamba-Transformer 混合架构在工业落地的案例，AIRA 的搜索方向与此一致
- [[Wiki/概念/02_训练方法/递归自我改进]] -- AIRA 是当前最接近可运行 RSI 的工程实现
- [[Wiki/概念/04_Agent框架/动态智能体脚手架]] -- AIRA-dojo 是 Agent 脚手架的具体实例

**冲突/印证**：
- **印证**：AIRAHybrid 系列系统性优于 AIRAformer 系列 → 印证了 [[101_LongCat]] 等工业趋势（混合架构 > 纯 Transformer）。AI 自主发现了人类近年的研究方向。
- **印证**：[[129_NanoGPT-Bench]] 中 Agent 在开放任务大幅落后人类 → AIRA-Design 中 NS < 0.26，印证了"当前 Agent 擅长有约束优化，不擅长开放式科学创新"。
- **潜在冲突**：AIRA-Design 中 Greedy Gemini 3 Pro 在 LRA 任务上平均最优准确率最高，但 Autoresearch 中 Greedy Opus 4.5/4.6 的 BPB 最低——不同基准的"最强推理引擎"结论不一致，不能简单说"某个模型最好"。

## 动手练习

```python
"""
进化循环：LLM 提出函数变体并评估
模拟 AIRA-Compose 的"提出候选 → 评估 → 选择最优 → 改进"循环
"""
import random, math, itertools
from typing import Callable

# ===== 1. 定义搜索空间：数学函数 =====
# 相当于 AIRA 的计算原语池
PRIMITIVES = {
    "x":       lambda x: x,
    "x2":      lambda x: x * x,
    "sin":     lambda x: math.sin(x),
    "cos":     lambda x: math.cos(x),
    "sqrt":    lambda x: math.sqrt(abs(x) + 1e-8),
    "log1p":   lambda x: math.log1p(abs(x)),
    "neg":     lambda x: -x,
    "inv":     lambda x: 1.0 / (abs(x) + 1e-8),
    "identity": lambda x: x,
    "exp":     lambda x: math.exp(-abs(x)),  # 有界变体
}

# ===== 2. 组合函数：两个函数的复合或加权和 =====
# 相当于 AIRA 的 16 层排列
def compose(f: Callable, g: Callable) -> Callable:
    """f ∘ g"""
    return lambda x: f(g(x))

def weighted_sum(f: Callable, g: Callable, w: float = 0.5) -> Callable:
    """w*f + (1-w)*g"""
    return lambda x: w * f(x) + (1 - w) * g(x)

# ===== 3. 评估函数 =====
# 相当于 evaluate.py 的独立评分
def evaluate(func: Callable, test_points: list[float]) -> float:
    """目标：逼近 y = sin(x) + 0.2*x^2 的形状（在[-3,3]上）"""
    target = lambda x: math.sin(x) + 0.2 * x * x
    errors = [abs(func(x) - target(x)) for x in test_points]
    return -sum(errors) / len(errors)  # 负MSE，越大越好

# ===== 4. 进化循环（模拟 AIRA-Compose Greedy） =====
# 这相当于一个简化版 AI Agent：不需要真实 LLM，
# 而是用启发式规则来"提出"和"改进"候选方案

def run_evolution(
    primitives: dict,
    test_points: list[float],
    generations: int = 10,
    population_size: int = 5,
) -> list[tuple[str, float]]:
    """进化主循环：提出 → 评估 → 保留最优 → 变异改进"""
    names = list(primitives.keys())

    # 第1代：随机生成候选（相当于 Draft 5个初始解）
    population = []
    for _ in range(population_size):
        f_name = random.choice(names)
        g_name = random.choice(names)
        # 随机选择组合方式
        if random.random() < 0.5:
            func = compose(primitives[f_name], primitives[g_name])
            desc = f"{f_name}({g_name}(x))"
        else:
            w = round(random.uniform(0.2, 0.8), 2)
            func = weighted_sum(primitives[f_name], primitives[g_name], w)
            desc = f"{w}*{f_name}(x)+(1-{w})*{g_name}(x)"
        score = evaluate(func, test_points)
        population.append((desc, score))

    history = []

    for gen in range(generations):
        # 评估所有候选
        population.sort(key=lambda x: x[1], reverse=True)
        best = population[0]
        history.append(best)
        print(f"Gen {gen+1}: best = {best[0]} | score = {best[1]:.4f}")

        # Improve：从最优个体变异产生新一代
        # 相当于 AIRA 的 Improve 操作符
        new_pop = [best]  # elitism：保留最优
        parent_name = best[0]
        for _ in range(population_size - 1):
            # 变异策略：换一个原语 or 调权重
            mutate_type = random.choice(["swap_primitive", "adjust_weight", "add_layer"])
            if mutate_type == "swap_primitive":
                old_p = random.choice(names)
                new_p = random.choice(names)
                mutated = parent_name.replace(f"{old_p}(", f"{new_p}(")
                mutated = mutated.replace(f"{old_p})", f"{new_p})")
                # 简化：直接生成新组合
                f_name, g_name = random.sample(names, 2)
                func = compose(primitives[f_name], primitives[g_name])
                desc = f"{f_name}({g_name}(x))"
            elif mutate_type == "adjust_weight":
                w = round(random.uniform(0.1, 0.9), 2)
                f_name, g_name = random.sample(names, 2)
                func = weighted_sum(primitives[f_name], primitives[g_name], w)
                desc = f"{w}*{f_name}(x)+{round(1-w,2)}*{g_name}(x)"
            else:  # add_layer：三层复合
                f, g, h = random.sample(names, 3)
                func = compose(primitives[f], compose(primitives[g], primitives[h]))
                desc = f"{f}({g}({h}(x)))"

            # Debug（如果计算结果全是 NaN/Inf 则丢弃重试）
            try:
                score = evaluate(func, test_points)
                if math.isnan(score) or math.isinf(score):
                    continue
            except Exception:
                continue
            new_pop.append((desc, score))

        population = new_pop

    return history


# ===== 5. 运行 =====
test_x = [x / 10 for x in range(-30, 31)]  # [-3, -2.9, ..., 3]
print("=== AIRA 风格进化搜索：逼近 sin(x) + 0.2x^2 ===\n")
results = run_evolution(PRIMITIVES, test_x, generations=8, population_size=6)

print(f"\n最终最优: {results[-1][0]} | MSE = {-results[-1][1]:.6f}")
print(f"初始最优: {results[0][0]}  | MSE = {-results[0][1]:.6f}")
print(f"改进倍数: {-results[0][1] / -results[-1][1]:.1f}x")
# 观察：进化循环是否有系统性改进？是否在某个局部最优卡住？
# 这对应 AIRA 中 Greedy search 的 exploitation-exploration 权衡
```

## 自测三层

**L1 记忆**：AIRA 两个框架分别解决什么问题？Compose 用几个 Agent/什么计算预算/找到几个新架构？

**L2 理解**：为什么 2-prim 空间和 3-prim 空间不能用同一个数据集？AIRAformer-C 扩展效率比 Llama 3.2 快 54% 是怎么测出来的？为什么不能用"省了 54% 钱"来直接理解？

**L3 延伸**：如果让你设计 AIRA 的下一步实验，你会选择 (a) 增加原语类型（如 RetNet/RWKV），(b) 让 Agent 同时搜索原语排列和训练超参数，(c) 把搜索空间从 16 层扩展到 48 层？各有什么风险和预期收益？结合 LRA 任务中 Agent 距离人类 SOTA 仍有巨大差距这一事实。

---

知识时间锚 2026-05（arXiv:2605.15871v1）。AIRA 代表"AI 科研 Agent"从概念验证到产生超越人工设计结果的转折点——但仍限定在有约束搜索空间内。配套基础设施 AIRS-Bench + AIRA-dojo 为后续 RSI 研究提供了标准化任务和执行环境。

[[Wiki/论文笔记/13_评测科研/129_NanoGPT-Bench自主研究Agent评测]] [[Wiki/概念/02_训练方法/递归自我改进]] [[Wiki/概念/04_Agent框架/动态智能体脚手架]]
