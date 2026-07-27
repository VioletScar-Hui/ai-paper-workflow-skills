---
tags: [论文笔记, 多智能体, 综述, 失败归因, 自进化, 协作]
paper_id: "136"
filename: "136 - Beyond Individual Intelligence - Surveying Collaboration, Failure Attribution, and Self-Evolution in LLM-based Multi-Agent Systems.pdf"
authors: "Shihao Qi, Jie Ma, Rui Xing et al. (Xi'an Jiaotong University + Lenovo AI + Sydney)"
year: 2026
笔记层级: 骨干
复核日期: 2026-07-04
---

# 多 Agent 系统综述：协作、失败归因与自进化

📄 **原文 PDF**：[[RAW/136 - Beyond Individual Intelligence - Surveying Collaboration, Failure Attribution, and Self-Evolution in LLM-based Multi-Agent Systems.pdf]]

## PM 速判 > 一句话

> 89 页综述，用 **LIFE 框架**（能力奠基 L → 交互协作 I → 失败归因 F → 自进化 E）统一多 Agent 系统全生命周期——最大贡献是将"失败归因"识别为从协作到自进化的关键桥梁，并明确指出这是当前研究最稀少但在生产中消耗最多工程时间的环节。

## 双层费曼

> **给 CEO**：部署多 Agent 协作系统（多个 AI 互相配合完成任务）时，最消耗工程时间的不是让 Agent 变聪明，而是——当一个任务的最终结果出错了，你要花几个小时去追查"是哪个 Agent 的哪一步出了错"。本文指出这是整个行业研究最薄弱的环节，并给出了从构建到进化的完整框架。结论：先把错误追踪做好，再考虑让系统自动进化，否则进化方向是随机的。

> **给工程师**：LIFE 框架覆盖 420+ 篇文献，将多 Agent 系统分解为 L（单 Agent 推理/记忆/规划/工具使用）→ I（组织拓扑：中心化 Orchestrator-Worker vs 去中心化 P2P；通信协议；任务分解）→ F（失败归因：数据驱动/约束引导/因果推断三类方法，定位错误在哪个 Agent 的哪个步骤传播）→ E（自进化：Agent级/System级/Meta级三层，核心公式 S(k+1)=Γ(S(k), H(k))，其中 H(k) 必须包含执行轨迹+错误归因+同伴批评）。关键洞察：没有精确的 F，自进化的 Γ 映射只能靠随机搜索（低效）。"

## 问题域定位

**根本约束**：多 Agent 系统中，单个 Agent 的错误可以通过消息和交互轮次传播到其他 Agent——经过多轮后，根因和最终表现可能相差很远。这是单 Agent 系统没有的挑战（单 Agent 失败可以就地定位）。

**之前卡点**：
- L（能力）、I（协作）、E（自进化）三个方向均有大量独立综述，但 F（失败归因）没有任何系统性综述
- 实际生产中，"Agent 行为异常"的调试工时是最隐性但最高的维护成本，但学术研究严重滞后于工程需求
- 自进化研究通常假设"成功/失败"信号可以直接指导改进，但实际上不知道"哪里失败"就改不了"什么"

**路线开启**：LIFE 框架首次覆盖完整生命周期；F 阶段的三类归因方法（数据驱动/约束引导/因果推断）为工程实践提供了技术选型地图；自进化公式明确了历史上下文 H(k) 必须包含的三类信息。

**路线关闭**：作为综述，缺少自己的实验验证；LIFE 的四阶段划分有一定主观性——特别是 F（归因）和 E（进化）在实践中边界模糊（好的归因本身就是一种进化信号）；420+ 文献的覆盖深度因主题而异（L 和 I 阶段远厚于 F 阶段）。

## 核心机制

```
┌──────────────────────────────────────────────────────────────────┐
│                   LIFE 框架：多 Agent 系统全生命周期                │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  L — Lay the Capability (能力奠基)                       │    │
│  │  单 Agent 核心能力: 推理 · 记忆 · 规划 · 工具使用          │    │
│  │  关键洞察: 各模块优化相互独立，但部署中模块间耦合是瓶颈       │    │
│  └───────────────────────┬─────────────────────────────────┘    │
│                          ↓                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  I — Interaction & Collaboration (交互协作)               │    │
│  │                                                          │    │
│  │  组织维度:                                                │    │
│  │    角色 (Role) ── 通信 (Communication)                    │    │
│  │    编排拓扑 (Topology) ── 任务分解 (Task Decomposition)    │    │
│  │                                                          │    │
│  │  两类主要模式:                                            │    │
│  │    中心化 Orc
│  │      Orchestrator         去中心化 P2P                    │    │
│  │      ┌────┐                ┌────┐   ┌────┐              │    │
│  │      │Orc │◄──►Worker      │ A1 │◄─►│ A2 │              │    │
│  │      └────┘  / \           └────┘   └────┘              │    │
│  │             W1 W2 W3        ↕   ↕    ↕    ↕              │    │
│  │      确定性强              灵活但一致性难保证             │    │
│  │      瓶颈在中心节点                                        │    │
│  └───────────────────────┬─────────────────────────────────┘    │
│                          ↓                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  F — Failure Attribution (失败归因) ★ 现有研究空白        │    │
│  │                                                          │    │
│  │  核心问题: 错误可以跨 Agent 和交互轮次传播 → 根因难定位    │    │
│  │                                                          │    │
│  │  三类归因方法:                                            │    │
│  │    ① 数据驱动: 从执行轨迹学习误差传播模式                  │    │
│  │    ② 约束引导: 业务规则/状态机谓词定位违反点               │    │
│  │    ③ 因果推断: 构建反事实对比，定位根因                    │    │
│  │                                                          │    │
│  │  类比: 蚂蚁群体 → 信息素消退 = 失败归因（隐性）            │    │
│  │                   寻找新路径 = 基于归因的自进化             │    │
│  └───────────────────────┬─────────────────────────────────┘    │
│                          ↓                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  E — Evolution & Self-Improvement (自进化)               │    │
│  │                                                          │    │
│  │  核心公式: S(k+1) = Γ(S(k), H(k))                        │    │
│  │    S(k): 第 k 轮的系统状态                                │    │
│  │    Γ: 自进化映射（修改提示/角色/拓扑/通信协议）              │    │
│  │    H(k): 历史上下文 = 执行轨迹 + 错误归因 + 同伴批评       │    │
│  │                                                          │    │
│  │  三个层级:                                                │    │
│  │    Agent级: 自优化提示/记忆/工具                          │    │
│  │    System级: 重组拓扑、角色、通信协议                       │    │
│  │    Meta级: 进化机制本身也会演化                            │    │
│  │                                                          │    │
│  │  核心洞察: 没有精确的 F(归因) → Γ 只能靠随机搜索 → 低效   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  为什么 F 是缺失的关键环节？                                       │
│                                                                  │
│    协作 → 错误传播:                                               │
│      紧耦合协作放大风险——A1的错误通过消息→A2→A3→最终输出          │
│      错误来源与最终表现可能相差 N 步（调试困难）                    │
│                                                                  │
│    失败归因 → 自进化:                                             │
│      精确归因"哪个 Agent 的哪一步" → 自进化可针对性修改            │
│      无精确归因 → 自进化在随机搜索空间中浪费迭代                    │
└──────────────────────────────────────────────────────────────────┘
```

## 设计决策解剖

| 决策点 | 选择 | 被放弃方案 | 为什么 | 何时失效 |
|--------|------|-----------|--------|---------|
| 中心化 vs 去中心化拓扑 | 取决于任务特征：确定性需求高选中心化，灵活性需求高选去中心化 | 单一推荐 | I 阶段的拓扑选择直接决定 F 阶段的可归因性——中心化系统中错误沿 Worker→Orch→下游传播，归因相对清晰；去中心化系统中消息图复杂，归因难度剧增 | 当任务同时需要强确定性和高灵活性时（如金融交易+创意生成混合流），单一拓扑不够 |
| 三类归因方法的适用边界 | 数据驱动用于高频重复任务（有大量轨迹数据）；约束引导用于业务规则明确的任务；因果推断用于关键决策 | 单一归因方法 | 不同任务的错误模式不同——高频任务可以learn pattern，关键决策需要反事实推理，违反业务规则则用约束直接定位 | 当三类方法给出不同归因时（如数据驱动指向 A1 但因果推断指向 A2），需要元归因机制 |
| 自进化三层级的独立性 | 三层可独立运作，但 System 级进化（改拓扑）可能使 Agent 级进化（改提示）失效 | 只做 Agent 级进化 | 修改拓扑相当于改变了 Agent 的交互环境，之前优化的提示可能不再有效——需要对"下层进化后上层是否需重新校准"进行显式追踪 | 当三层同时进化时，状态空间爆炸且评估链路变长 |
| H(k) 历史上下文构成 | 轨迹+归因+批评 | 仅轨迹 / 仅最终结果 | 最终成功/失败信号太弱，无法指导Γ做定向修改；批评提供多角度反馈但可能引入新的噪声 | 批评的质量本身取决于批评 Agent 的能力——如果批评 Agent 能力不足，批评信息会误导进化方向 |

## 成本与量级

| 维度 | 数据 |
|------|------|
| 综述规模 | 89 页，420+ 篇文献 |
| 覆盖阶段 | L(能力) / I(协作) / F(归因) / E(进化)，每个阶段下细分多个维度 |
| 基准地图 | AgentBench, GAIA, TheAgentCompany, SWE-bench, Terminal-Bench, WebArena, BrowseComp, LongMemEval, LoCoMo, PaperBench, MLE-Bench 等 |
| 研究密度对比 | L/I/E 三方向均有大量独立综述；F（失败归因）在文章中篇幅最短、引用最少——直接反映了领域空白 |
| 作者机构 | 西安交通大学 + Lenovo AI + University of Sydney（跨国多机构合作） |

## 证据审计

**实验公平性**：作为综述论文，其"证据"来自文献综述而非实验。各阶段的文献覆盖密度不均衡——L（能力）和 I（协作）的引用密度远超 F（归因）。这是该领域的客观现状而非方法偏差，但读者应注意 F 阶段的结论更多是"提出问题"而非"有答案"。

**最强证据**：
- **文献密度对比是最有力的证据**：作者通过系统检索发现 L/I/E 三方向均已有独立综述，而 F 没有任何系统性综述——这个空白本身是多 Agent 系统中被忽视的"房间里的大象"。
- **自进化公式 S(k+1)=Γ(S(k), H(k)) 的形式化**：将直觉（"需要知道哪里错了才能改进"）转化为可操作的框架（H(k) 必须包含三类信息），为后续研究提供了可量化的研究方向（H 的信息完整性与 Γ 的进化效率之间的定量关系尚未被研究）。

**最弱证据**：
- **没有实验验证**：LIFE 框架作为组织框架的价值有待实验验证——是否能通过遵循 LIFE 构建系统来实际减少调试时间或提高进化效率？
- **四阶段边界的模糊性**：F 与 E 之间在实践中难以清晰分离——一次成功的归因本身可能就是一次进化（Agent 知道了自己的错误模式）。框架将它们分为两个独立阶段在理论上清晰，但在工程中可能过于简化。
- **F 阶段的具体技术方案稀疏**：三类归因方法（数据驱动/约束引导/因果推断）的描述停留在高层次，缺少可操作的实现指南和对比评测。

**审稿补充**：I 阶段的拓扑讨论缺少对"动态拓扑"（Agent 在运行中自主切换中心化/去中心化）的覆盖——这在 MetaCogAgent（Paper 127）等元认知框架中已有初步实践。

**最小复现**：不需要计算资源。对一个现有多 Agent 系统：列出每个 Agent 的输入/输出/工具调用 → 画出消息传播 DAG → 选一个端到端失败案例 → 标注错误在哪个 Agent 的哪个步骤被引入 → 在哪个下游步骤被放大 → 评估当前系统是否有能力自动完成这个标注过程（即 F 阶段能力的自评）。

## 可复用点 + 供应商拷问清单

**可复用点**：
1. LIFE 框架可直接用于评估任何多 Agent 产品的完备性——快速定位 L/I/F/E 哪个阶段投入不足
2. 错误传播链的可视化（Agent 消息 DAG）是 F 阶段的第一步工程实践——不需要复杂方法就能开始
3. 自进化公式 S(k+1)=Γ(S(k), H(k)) 可作为自进化 Agent 系统的设计规范——先定义 H(k) 包含什么信息，再设计 Γ 的修改操作
4. 中心化 vs 去中心化的拓扑分类帮助早期架构决策——选错拓扑会直接增加 F 阶段的归因难度

**供应商拷问清单**（评估多 Agent 平台/框架时）：
- [ ] 你们的系统在 Agent 间错误传播时，能否追溯到"哪个 Agent 的哪个工具调用"？追溯粒度是 Agent 级还是 step 级？
- [ ] 你们有内置的失败归因机制吗？是数据驱动（需要多少轨迹？），还是约束引导（需要我定义什么规则？），还是因果推断（需要多少次反事实实验？）
- [ ] 如果我想做自进化（让系统自己优化），你们需要我提供什么格式的反馈信号？仅成功/失败？还是需要结构化归因？
- [ ] 你们的中心化编排器中，如果 Orchestrator 自己犯了错，有二级监督者吗？还是单点故障？
- [ ] 当系统从 2 Agent 扩展到 10 Agent 时，错误归因的难度如何变化？你们的可观测性面板能跟上吗？

## 关联网络

- [[Wiki/论文笔记/03_Agent系统设计/128_生产LLM-Agent运行架构模式方法论]] -- SDB 和 6 种运行模式对应 F 阶段的工程实践基础；SDB 处理单 Agent 边界安全，LIFE-F 处理多 Agent 错误传播链
- [[Wiki/论文笔记/03_Agent系统设计/127_MetaCogAgent元认知多智能体框架]] -- 元认知自评估对应 I 阶段协作模式的具体实现，动态拓扑的早期实践
- [[Wiki/论文笔记/03_Agent系统设计/139_AEVO元编辑Agent进化框架]] -- AEVO 是 LIFE 框架 E 阶段的具体实现路径
- [[Wiki/概念/04_Agent框架/LLM集体智能]] -- 多 Agent 协作的理论基础，对应 I 阶段
- [[Wiki/概念/04_Agent框架/动态智能体脚手架]] -- LIFE 框架 I 阶段的编排机制
- [[Wiki/概念/05_记忆与检索/Agent长期记忆]] -- L 阶段记忆模块的底层概念

**冲突/印证**：
- **印证**：[[128_生产Agent架构模式]] 的核心论点"先建 Dashboard 再建 Agent"→ LIFE 框架的 F 阶段直接将可观测性作为归因的前提条件——两个框架在"先有可见性才能改进"上完全一致。
- **印证**：[[125_AIRA]] 中 Agent 在开放任务（LRA）的 NS < 0.26，远低于人类 SOTA → LIFE 框架的解读：AIRA-Design 在 I 阶段（协作？不，单 Agent）和 F 阶段（归因 OOM/NaN 失败）都弱，导致 E 阶段无法有效定向改进。
- **潜在冲突**：LIFE 框架明确将"失败归因"作为 E（自进化）的前提——但 [[139_AEVO]] 的元编辑 Agent 可能通过"编辑→评估→选择"的进化循环绕过显式的失败归因（通过选择压力隐式归因）。这两种范式（显式归因 vs 隐式选择）的优劣尚无对比研究。

## 动手练习

```python
"""
双 Agent 辩论脚本：人为注入错误，观察错误如何在 Agent 间传播
模拟 LIFE 框架 I→F 阶段的错误传播链
"""
import random
from dataclasses import dataclass, field

@dataclass
class DebateStep:
    """辩论中的一步，包含谁说了什么、是否有错误"""
    agent_name: str
    message: str
    has_error: bool = False     # 这一步是否被注入了错误
    error_type: str = ""        # 错误类型
    error_description: str = ""

@dataclass
class DebateTrace:
    """完整的辩论追踪——模拟 LIFE 框架中"执行轨迹""""
    steps: list[DebateStep] = field(default_factory=list)
    final_verdict: str = ""

# ===== 1. 定义两个 Agent =====
class DebateAgent:
    """模拟一个参与辩论的 Agent（简化版，不用真实 LLM）"""

    def __init__(self, name: str, knowledge: dict, error_rate: float = 0.0):
        self.name = name
        self.knowledge = knowledge
        self.error_rate = error_rate  # 人为注入错误的概率
        self.memory: list[DebateStep] = []  # Agent 的局部记忆

    def respond(self, topic: str, opponent_last: DebateStep | None) -> DebateStep:
        """生成辩论回应——可能包含人为注入的错误"""
        self.memory.append(opponent_last) if opponent_last else None

        # 正常知识检索
        truth = self.knowledge.get(topic, "未知")

        # 人为注入错误（模拟 I 阶段的失败）
        has_error = random.random() < self.error_rate
        error_type = ""
        error_desc = ""

        if has_error:
            error_choices = [
                ("factual_shift", "把事实从正确值逐步偏移"),
                ("hallucination", "编造了一个听起来合理但不存在的数字"),
                ("omission", "遗漏了关键约束条件"),
                ("misinterpretation", "误解了对方的上一条论点的含义"),
            ]
            error_type, error_desc = random.choice(error_choices)

            if error_type == "factual_shift":
                # 把真实值上下浮动 20-50%
                if truth.replace(".", "").replace("-", "").isdigit():
                    truth = str(float(truth) * random.uniform(0.5, 0.8))[:6]
            elif error_type == "hallucination":
                truth = str(random.randint(1, 100))
            elif error_type == "omission":
                truth = truth.split("，")[0]  # 丢掉后半段约束
            elif error_type == "misinterpretation":
                if opponent_last:
                    truth = f'对方说 "{opponent_last.message[:20]}..." ——但这是错误的，实际上 {truth}'

        message = f"[{self.name}] 关于 {topic}: {truth}"
        return DebateStep(
            agent_name=self.name,
            message=message,
            has_error=has_error,
            error_type=error_type,
            error_description=error_desc,
        )

# ===== 2. 辩论编排器 =====
def run_debate(
    topic: str,
    agent_a: DebateAgent,
    agent_b: DebateAgent,
    rounds: int = 4,
) -> DebateTrace:
    """编排一场双 Agent 辩论，记录完整追踪"""
    trace = DebateTrace()

    # Agent A 开场（I 阶段：中心化编排，A 先发言）
    step = agent_a.respond(topic, None)
    trace.steps.append(step)
    print(f"Round 1: {step.message}")
    if step.has_error:
        print(f"  ⚠ 错误注入 [{step.error_type}]: {step.error_description}")

    # 交替辩论
    for r in range(1, rounds):
        # Agent B 回应（可能受 A 的错误影响）
        step_b = agent_b.respond(topic, trace.steps[-1])
        trace.steps.append(step_b)
        print(f"Round {r+1}a: {step_b.message}")
        if step_b.has_error:
            print(f"  ⚠ 错误注入 [{step_b.error_type}]: {step_b.error_description}")
        elif trace.steps[-2].has_error:
            # B 没有新错误，但检查 A 的错误是否传播到了 B
            a_error = trace.steps[-2]
            if a_error.error_type == "factual_shift":
                print(f"  ↳ 错误传播: B 使用了 A 的错误数值 (来自 {a_error.agent_name})")

        # Agent A 再回应
        step_a = agent_a.respond(topic, trace.steps[-1])
        trace.steps.append(step_a)
        print(f"Round {r+1}b: {step_a.message}")
        if step_a.has_error:
            print(f"  ⚠ 错误注入 [{step_a.error_type}]: {step_a.error_description}")

    # F 阶段：失败归因分析
    print(f"\n{'='*60}")
    print(f"  F 阶段：失败归因分析")
    print(f"{'='*60}")

    error_steps = [s for s in trace.steps if s.has_error]
    print(f"  总步数: {len(trace.steps)} | 含错误步数: {len(error_steps)}")

    if error_steps:
        first_error = error_steps[0]
        print(f"  首次错误: Round {trace.steps.index(first_error)+1}")
        print(f"    来源: {first_error.agent_name}")
        print(f"    类型: {first_error.error_type}")
        print(f"    描述: {first_error.error_description}")

        # 追踪错误传播链
        print(f"\n  错误传播链:")
        first_idx = trace.steps.index(first_error)
        for i in range(first_idx, len(trace.steps)):
            s = trace.steps[i]
            if s.has_error:
                print(f"    步骤{i+1} [{s.agent_name}]: 新错误 - {s.error_type}")
            elif i > first_idx and trace.steps[i-1].has_error:
                # 虽然没有新错误，但上一步有错误 → 可能被无声传播
                prev_err = trace.steps[i-1]
                print(f"    步骤{i+1} [{s.agent_name}]: 可能受步骤{i} [{prev_err.agent_name}] 影响 (无声传播)")

        # 尝试归因根因
        print(f"\n  ★ 归因结论: 根因在 Agent '{first_error.agent_name}' 的步骤{first_idx+1}")
        print(f"    如果这是生产系统，修复应集中在 {first_error.agent_name} 的 {first_error.error_type} 防护上")
    else:
        print(f"  无错误: 辩论正常完成")

    return trace

# ===== 3. 运行三组对比实验 =====
TOPIC = "量子计算的商业化时间线"
KNOWLEDGE_A = {TOPIC: "预计2030-2035年实现1000+逻辑量子比特的纠错量子计算机，主要瓶颈在错误率从10^-3降到10^-6"}
KNOWLEDGE_B = {TOPIC: "目前NISQ时代(50-100物理量子比特)的量子优势已在特定问题上展示，但通用容错量子计算仍需10-15年"}

print("=" * 70)
print("  多 Agent 辩论：错误传播实验 (LIFE 框架 I→F 阶段)")
print("=" * 70)

# 实验 1: 无错误注入（对照组）
print("\n--- 实验 1: 对照组 (无错误注入) ---")
agent_a = DebateAgent("Agent-Alpha", KNOWLEDGE_A, error_rate=0.0)
agent_b = DebateAgent("Agent-Beta", KNOWLEDGE_B, error_rate=0.0)
trace1 = run_debate(TOPIC, agent_a, agent_b, rounds=3)

# 实验 2: 只有 Agent A 有错误注入
print("\n\n--- 实验 2: Agent-Alpha 有 50% 错误率 ---")
agent_a2 = DebateAgent("Agent-Alpha", KNOWLEDGE_A, error_rate=0.5)
agent_b2 = DebateAgent("Agent-Beta", KNOWLEDGE_B, error_rate=0.0)
trace2 = run_debate(TOPIC, agent_a2, agent_b2, rounds=3)

# 实验 3: 双方都有错误注入
print("\n\n--- 实验 3: 双方各有 30% 错误率 (最差情况) ---")
agent_a3 = DebateAgent("Agent-Alpha", KNOWLEDGE_A, error_rate=0.3)
agent_b3 = DebateAgent("Agent-Beta", KNOWLEDGE_B, error_rate=0.3)
trace3 = run_debate(TOPIC, agent_a3, agent_b3, rounds=3)

print(f"\n{'='*70}")
print(f"  关键洞察:")
print(f"  1. 单点错误如果未被纠正，会在后续轮次中无声传播")
print(f"  2. 多 Agent 场景下，最终输出错误≠第一个出错的 Agent")
print(f"  3. 没有 F 阶段(归因)，E 阶段(进化)只能猜测改谁——大概率改错目标")
```

## 自测三层

**L1 记忆**：LIFE 四个字母分别代表什么？I 阶段的两类主要拓扑分别是什么？自进化公式中 H(k) 包含哪三类信息？

**L2 理解**：为什么"失败归因"是从协作到自进化的关键桥梁？如果跳过 F 直接做 E，系统的行为会是什么样？用具体的多 Agent 场景（如代码审查：Agent A 生成代码 → Agent B Review → Agent C 测试）说明错误传播链。

**L3 延伸**：设计一个"归因质量评分"的指标来评估多 Agent 系统的 F 阶段成熟度。考虑归因粒度（Agent 级 vs step 级 vs token 级）、归因准确率（与人工标注对比）、归因延迟（从失败发生到归因结果可用的时间）。然后评估你负责的多 Agent 产品在这个评分下能打几分——最需要提升的是哪个维度？

---

知识时间锚 2026。LIFE 框架是截至 2026 年最全面的多 Agent 系统综述。核心贡献不是覆盖量（420+ 文献），而是识别出 F（失败归因）这个"最稀少却最关键"的研究空白——为 2026-2027 年的多 Agent 研究提供了明确的方向指引。与 [[128]] 的 SDB 框架互补：SDB 处理单 Agent 边界安全，LIFE-F 处理多 Agent 错误传播链——两者结合构成 Agent 系统可靠性的完整图景。

[[Wiki/论文笔记/03_Agent系统设计/128_生产LLM-Agent运行架构模式方法论]] [[Wiki/论文笔记/03_Agent系统设计/127_MetaCogAgent元认知多智能体框架]] [[Wiki/概念/04_Agent框架/LLM集体智能]] [[Wiki/概念/04_Agent框架/动态智能体脚手架]]
