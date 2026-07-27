---
tags: [论文笔记, Agent架构, 生产系统, 分布式系统, 架构模式]
paper_id: "128"
filename: "128 - A Methodology for Selecting and Composing Runtime Architecture Patterns for Production LLM Agents.pdf"
authors: "Vasundra Srinivasan (Stanford School of Engineering / O'Reilly)"
year: 2026
笔记层级: 骨干
复核日期: 2026-07-04
---

# 生产 LLM Agent 运行架构模式方法论

📄 **原文 PDF**：[[RAW/128 - A Methodology for Selecting and Composing Runtime Architecture Patterns for Production LLM Agents.pdf]]

## PM 速判 > 一句话

> 生产 Agent 出事的 71% 是"架构问题不是模型问题"——本文给出 SDB（随机-确定边界）四部件合约 + 3 类运行时 × 6 种模式 + 5 步选型方法论 + 六行 ADR 模板，让 PM/工程师可以从任意 Agent 失败后报告里定位"哪个边界没守住"，并按方法论系统选择正确模式。

## 双层费曼

> **给 CEO**：生产环境部署的 AI Agent 经常出问题，调查发现 71% 的失败不是因为 AI 模型本身不够聪明，而是因为系统架构在"AI 输出"和"确定性执行"之间的边界没守住。本文提供了一套命名体系和选型流程，让团队能在设计阶段就选对架构，而不是出事后救火。

> **给工程师**：SDB（Stochastic-Deterministic Boundary）将 Agent 的本质结构抽象为随机核心（LLM）与确定性系统（调度/验证/存储/控制）之间的接缝，由四部件合约定义：Proposer（LLM 输出）→ Verifier（确定性检查：Schema/策略/状态机谓词/分类器）→ Commit（通过后写入/副作用）→ Reject Signal（失败时的类型化反馈）。分析了 5 个开源框架的 21 个 LLM→Action 调用点，发现从 openai/swarm 单行 JSON 解析到 MetaGPT 多阶段 pydantic+LLM-as-judge，实现强度差异巨大。6 种模式按 Coordination/State/Control 三关注点分类。5 步选型中 Step 2（选脊梁/状态模式）是迁移成本最高的决定。

## 问题域定位

**根本约束**：LLM 的输出本质是概率分布采样（非确定），但生产系统需要在正确的时间执行正确的副作用（确定）。当 LLM 直接驱动数据库写入、API 调用、消息发送时，概率性错误会放大为生产事故。而工程团队缺乏系统性语言来讨论这个问题——大家说"Agent 幻觉了"，但实际上可能是 SDB 的 Verifier 没拦住。

**之前卡点**：
- 5 个主流框架（OpenAI Swarm, AutoGPT, LangChain, CrewAI, AutoGen）都有 Agent 循环，但没有人系统比较它们的 SDB 实现强度差异
- 失败后报告散落在 GitHub Issues/Discord，没有统一的概念框架来编码和分析
- "选架构模式"靠直觉，没有形式化决策流程——特别是状态模式（事件溯源 P3 vs 状态机 P5）的选择往往在后期才发现选错了

**路线开启**：SDB 命名原语提供了跨框架的故障诊断语言；6 种模式 + 5 步选型让"选架构"从直觉变为可审计的工程决策；六行 ADR 模板使决策可追溯、可复盘。

**路线关闭**：SDB 本身不解决"Verifier 怎么写"的问题——它只告诉你需要 Verifier，但 Schema 设计、策略引擎、分类器训练仍需工程投入。21 个失败报告样本量小，不构成统计意义上可靠的故障分布。

## 核心机制

```
┌──────────────────────────────────────────────────────────────────┐
│              生产 Agent 的本质结构与 SDB 四部件                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────┐                    ┌─────────────────────┐   │
│   │  随机核心     │                    │   确定性系统          │   │
│   │  (LLM)       │  ═══ SDB ═══════   │   调度/验证/存储/控制  │   │
│   │  概率分布采样  │     接缝             │   确定性状态转换      │   │
│   └──────┬───────┘                    └──────────┬──────────┘   │
│          │                                       │              │
│   Proposer (提案)                               │              │
│   对分布采样 → 输出 action/json/text             │              │
│          │                                       │              │
│          ▼                                       │              │
│   ┌─────────────────┐                           │              │
│   │  Verifier (验证)  │◄── 确定性检查 ─────────────┘              │
│   │  Schema验证      │                                          │
│   │  策略规则         │   失败 ──→  Reject Signal (拒绝信号)       │
│   │  状态机谓词       │             类型化响应返回给 Proposer       │
│   │  快速分类器       │   通过 ──→  Commit (提交步)               │
│   └─────────────────┘             持久写入 / 外部副作用           │
│                                                                  │
│   实现强度光谱:                                                    │
│   弱 ◄──── openai/swarm          MetaGPT ────► 强               │
│        core.py 单行JSON解析       ActionNode 多阶段               │
│        无 schema 检查             pydantic + LLM-as-judge        │
│                                  自动审查循环                     │
└──────────────────────────────────────────────────────────────────┘

三类运行时 × 六种模式：

  ┌──────────────────────────────────────────────────────┐
  │  Coordination (协调)     State (状态)    Control (控制) │
  │  工作如何拆分与合并？     如何跨暂停记忆？  谁决定运行/何时停？│
  ├──────────────────────────────────────────────────────┤
  │  P1 分层委托              P3 事件溯源      P4 监督者+门 │
  │  (Hierarchical            (Event-Driven    (Supervisor │
  │   Delegation)              Sequencing)      + Gate)    │
  │                           ┌─回放漂移风险─┐             │
  │  P2 扇出+Saga             P5 共享状态机    P6 人在回路  │
  │  (Scatter-Gather          (Shared State    (Human in   │
  │   + 补偿日志)               Machine+CAS)    the Loop)  │
  └──────────────────────────────────────────────────────┘

五步选型方法论：

  Step 1: 分类运行时 (Conversational/Autonomous/Long-Horizon)
  Step 2: 选"脊梁" (P5 if >1h暂停+状态不可重建+世界会变; P3 否则)
          ★ 迁移成本最高 → 高级工程师书面确认
  Step 3: 包裹协调 (P1单所有者+P2有副作用 → 可同时用)
  Step 4: 绑定控制 (杀开关/升级/审批/限流 → 四个控制面不可省略)
  Step 5: 序列化构建 (面板→Gate→编排器→子Agent→控制面)
```

**六行 ADR 模板**（方法论的输出）：

| 步骤 | 模式 | 触发谓词 | 选错后的失败签名 |
|------|------|---------|--------------|
| 运行时 | Long-Horizon | 工作单元 >1h，世界会变 | 延迟预算违反 |
| 脊梁 | P5 | 三条件全满足 | 跨模型版本回放漂移 |
| 协调 | P1+P2 | 单一所有者+外部副作用 | 部分写入不一致 |
| 控制 | P4+P6 | 副作用+法律后果 | 幻觉折扣实际发货 |
| 序列 | 面板先行 | 可观测性先于 Agent | 盲目运维 |
| 日期/模型 | 2026 Q2 / Claude Sonnet 4.6 | — | — |

## 设计决策解剖

| 决策点 | 选择 | 被放弃方案 | 为什么 | 何时失效 |
|--------|------|-----------|--------|---------|
| 状态模式脊梁 | P5 持久版本化状态机+CAS 作为 Long-Horizon 默认 | P3 事件溯源（更简单，append-only） | Long-Horizon 场景中世界在运行中变化，P3 从事件日志重建状态会因为 LLM 消费者的不确定性产生"回放漂移"——传统事件溯源没有这个问题 | 如果工作单元 <1h 且世界不变（如短会话 Agent），P3 的简单性优势超过 P5 的安全保障——此时选 P5 是 over-engineering |
| SDB 命名原语 | 四部件 Proposer/Verifier/Commit/Reject Signal | 直接说"Agent 安全/Agent 护栏" | 四部件明确区分了"谁提出"和"谁验证"，使故障可定位到具体部件；"安全"太笼统无法指导修复 | 当 Verifier 本身也是 LLM-based（如 LLM-as-judge）时，Verifier 的不确定性模糊了 SDB 本身——需要二阶 SDB |
| 五步选型顺序 | Step 5 指定"面板先行，Agent 后行" | 先建 Agent 原型再加可观测性 | 没有面板就无法验证 SDB 四部件是否在运行中被正确遵守，也无法做 ADR 中的任何审计——盲目运维的代价远超延迟 Agent 开发的成本 | 如果项目处于纯探索/原型阶段（0→1），"先建面板"可能过度投资——但本文主张即使原型也至少建轻量 logging |
| 实证基础范围 | 5 个开源框架，21 个调用点，21 个失败后报告 | 企业闭源系统 + 更大样本 | 开源框架代码可直接审计（实现强度对比透明），失败报告可从公开 Issues 和 Discord 提取 | 企业生产环境（多租户、合规约束、大规模并行）的失败模式可能与开源框架有系统性差异 |

## 成本与量级

| 维度 | 数据 |
|------|------|
| 实证基础 | 5 个开源框架（OpenAI Swarm, AutoGPT, LangChain, CrewAI, AutoGen）；21 个 LLM→Action 调用点的代码级分析；21 个失败后报告 |
| SDB 实现强度跨度 | openai/swarm core.py 单行 JSON 解析（零 schema 检查） → MetaGPT ActionNode 多阶段 pydantic+LLM-as-judge（数十行验证代码） |
| 关键统计 | 71% (15/21) 失败定位于 SDB 弱点；81% (17/21) 的文档化修复加强了 SDB 四部件之一 |
| 方法论适用范围 | Conversational（秒级，上下文窗口）、Autonomous（分钟级，队列/消息）、Long-Horizon（小时/天级，持久存储） |
| 迁移成本 | P3→P5 迁移是最高成本决策（需要重写状态层），作者要求高级工程师书面确认后再继续 |

## 证据审计

**实验公平性**：作为方法论论文而非实验论文，其"证据"来自对开源代码的审计和对失败报告的编码分析，而非受控实验。代码审计透明可复现（开源代码可被独立检查），失败报告编码存在一定主观性。

**最强证据**：5 个框架的 21 个调用点中，19 个有明确的 SDB 验证+提交逻辑——说明这个结构是自然涌现的，不是作者强加的概念框架。同时，实现强度从"无 schema 检查"到"多阶段审查循环"的光谱存在，直接证明了概念框架的区分力。

**最弱证据**：
- 21 个失败报告样本量小且来源偏向开源社区（GitHub Issues/Discord）——可能与大型企业生产环境中的失败分布不同
- "回放漂移"是新命名的失败模式（将传统事件溯源的确定性假设破缺归因于 LLM 消费者），但防御方案（锁定模型版本、测试回放一致性）的实施成本和有效性未量化
- 71%/81% 的统计基于对失败报告的人工编码——存在编码者偏见和分类边界模糊（如"模型错误"和"SDB 的 Proposer 失败"有时难以区分）

**审稿补充**：六种模式的命名参考了分布式系统经典模式（Actor Model, Sagas, The Log），但在 Agent 系统中的适配程度不同——P2（Saga+补偿日志）在 LLM 场景中的补偿逻辑远比微服务场景复杂（LLM 生成的文本如何精确补偿？）

**最小复现**：不需要计算资源。对一个现有 Agent 项目：列出所有 LLM→Action 调用点 → 对每个调用点标注 SDB 四部件的实现方式 → 对比 openai/swarm 和 MetaGPT 的实现强度 → 选一个调用点升级 SDB 实现（如从无 schema 检查升级到 pydantic 验证）。

## 可复用点 + 供应商拷问清单

**可复用点**：
1. SDB 四部件作为 Agent 故障诊断的首检框架：出事时先问"Proposer 输出了什么 → Verifier 拦住没有 → Commit 执行前 check 了什么 → Reject Signal 被正确处理了吗"
2. 六行 ADR 模板可直接嵌入项目文档，替代随意的架构选型
3. "先建 Dashboard 再建 Agent"是可引用的生产级原则，有方法论支撑
4. 三类运行时分类用于估算工程投入——Long-Horizon 的工程投入远高于 Conversational

**供应商拷问清单**（评估 Agent 框架/平台时）：
- [ ] 你们的 SDB 实现水平在哪个位置？接近 openai/swarm（单行解析）还是 MetaGPT（多阶段审查）？
- [ ] 如果我的 Agent 需要做外部副作用（退款/发邮件），你们提供什么级别的 Verifier？是内置的还是需要我自己写？
- [ ] 你们的框架支持 P5（持久版本化状态机+CAS）吗？如果不支持，用什么替代方案处理 >1h 的暂停+世界变化？
- [ ] 框架自带的可观测性面板能回答"Agent 在第几步的 SDB 哪个部件失败了"吗？
- [ ] 跨模型版本升级时，状态恢复的兼容性如何处理？有没有"回放漂移"的检测机制？

## 关联网络

- [[Wiki/论文笔记/03_Agent系统设计/122_代码作为Agent框架调查]] -- 代码即 Agent 脚手架是 SDB 实现的具体基础设施；Plan-Execute-Verify 对应 SDB 循环
- [[Wiki/论文笔记/03_Agent系统设计/126_弱推理模型的Agentic增强]] -- 验证信号（测试/执行）在 SDB 中是 Verifier 部件的核心实现
- [[Wiki/论文笔记/03_Agent系统设计/136_多Agent系统协作失败归因自进化综述]] -- LIFE 框架的 F（失败归因）阶段与 SDB 四部件的诊断逻辑互补
- [[Wiki/概念/04_Agent框架/Harness设计模式]] -- SDB 是 Harness 设计模式的架构层抽象
- [[Wiki/概念/04_Agent框架/分层Agent协议栈]] -- 六种模式可映射到分层协议栈的不同层级
- [[Wiki/概念/04_Agent框架/随机确定边界]] -- SDB 四部件合约的独立概念条目

**冲突/印证**：
- **印证**：[[136_多Agent系统协作失败归因]] 中"错误传播而非错误发生是多Agent系统的独特挑战"→ SDB 在单 Agent 的 Verifier 若不过关，多 Agent 场景下错误会指数级放大——两个框架的诊断逻辑一致。
- **印证**：[[122_代码作为Agent框架调查]] 中分析的 Agent 框架在 SDB 实现上存在相同的光谱差异（弱端→强端）→ SDB 是跨框架通用的分析工具。
- **潜在冲突**：本文主张 P5（状态机）是 Long-Horizon 默认脊梁，但 [[136]] 的 LIFE 框架中自进化公式 S(k+1)=Γ(S(k),H(k)) 暗示"事件历史 H(k) 的完整性"在自进化场景中比状态一致性更重要——P3 的完整事件日志可能更适合需要分析"为什么 Agent 做了某决定"的场景。

## 动手练习

```python
"""
客服 Agent 架构模式选型决策树
基于 128 论文的五步选型方法论，实现一个可交互的决策流
"""
from dataclasses import dataclass
from enum import Enum

class RuntimeType(Enum):
    CONVERSATIONAL = "对话型 (秒级)"
    AUTONOMOUS = "自主型 (分钟级)"
    LONG_HORIZON = "长视野型 (小时/天级)"

@dataclass
class ScenarioInput:
    """场景输入：模拟客服 Agent 的需求分析"""
    # 业务特征
    max_task_duration_hours: float     # 最长任务耗时
    world_changes_during_task: bool    # 任务期间外部世界是否变化
    has_external_side_effects: bool    # 是否有外部副作用（退款/发货）
    has_legal_consequences: bool       # 是否有法律后果
    single_owner_per_task: bool        # 每个任务是否有单一明确的结果负责人
    sub_tasks_independent: bool        # 子任务是否基本独立
    state_reconstructable: bool        # 暂停期间状态能否从原始输入重建
    pause_exceeds_1hour: bool          # 是否有超过1小时的暂停或外部等待

# ===== 决策引擎 =====
def step1_classify_runtime(scenario: ScenarioInput) -> RuntimeType:
    """Step 1: 分类运行时类型"""
    if scenario.max_task_duration_hours > 1:
        return RuntimeType.LONG_HORIZON
    elif scenario.max_task_duration_hours > 0.05:  # >3分钟
        return RuntimeType.AUTONOMOUS
    else:
        return RuntimeType.CONVERSATIONAL

def step2_select_backbone(scenario: ScenarioInput) -> str:
    """Step 2: 选脊梁（状态模式）——迁移成本最高的决定"""
    cond1 = scenario.pause_exceeds_1hour
    cond2 = not scenario.state_reconstructable
    cond3 = scenario.world_changes_during_task

    if cond1 and cond2 and cond3:
        return (
            "P5 持久版本化状态机 + CAS",
            "三个条件全满足: (1)>1h暂停 (2)状态不可重建 (3)世界会变 → "
            "这是迁移成本最高的决定，建议高级工程师书面确认后再继续。"
        )
    elif not cond1:
        return (
            "P3 事件溯源日志（append-only）",
            "仅条件(1)不满足: 无>1h的暂停/等待 → 不需要持久状态机。"
            "但注意: 如果使用了 LLM 消费者，需锁定模型版本防回放漂移。"
        )
    else:
        return (
            "P3 + P5 混合",
            "条件部分满足: 在暂停点使用 P5 的持久状态快照，非暂停段用 P3 日志。"
        )

def step3_choose_coordination(scenario: ScenarioInput) -> str:
    """Step 3: 包裹协调模式"""
    if scenario.single_owner_per_task and scenario.sub_tasks_independent:
        base = "P1 分层委托（Hierarchical Delegation）"
    else:
        base = "P2 扇出+Saga（Scatter-Gather + 补偿日志）"

    if scenario.has_external_side_effects:
        return base + " + P2 在有副作用的子任务层（可同时用: P1 外层, P2 内层）"
    return base

def step4_bind_control(scenario: ScenarioInput) -> list[str]:
    """Step 4: 绑定控制面——四个控制面不可省略"""
    controls = ["杀开关 (Kill Switch)"]
    if scenario.has_external_side_effects or scenario.has_legal_consequences:
        controls.append("升级 (Escalation)")
        controls.append("审批 (Approval)")
        controls.append("限流 (Throttling)")
    else:
        controls.append("限流 (Throttling)")
    return controls

# ===== 客服场景 =====
def evaluate_customer_service_agent():
    """为一个典型的客服 Agent 场景做架构模式选型"""

    # 场景定义: 电商客服 Agent，处理退款/换货请求
    scenario = ScenarioInput(
        max_task_duration_hours=0.02,      # 客服对话约1-2分钟
        world_changes_during_task=False,   # 对话期间库存/价格一般不变
        has_external_side_effects=True,    # 退款=真实金钱流动！
        has_legal_consequences=True,       # 消费纠纷可能有法律后果
        single_owner_per_task=True,        # 一个客服 Agent 负责一个会话
        sub_tasks_independent=True,        # 查订单、查库存、发起退款相对独立
        state_reconstructable=True,        # 会话历史可重建
        pause_exceeds_1hour=False,         # 客服不暂停
    )

    print("=" * 60)
    print("  客服 Agent 架构模式选型决策树")
    print("=" * 60)

    # Step 1
    rt = step1_classify_runtime(scenario)
    print(f"\n[Step 1] 运行时类型: {rt.value}")

    # Step 2
    backbone, reason = step2_select_backbone(scenario)
    print(f"\n[Step 2] 脊梁（状态模式）: {backbone}")
    print(f"         理由: {reason}")

    # Step 3
    coord = step3_choose_coordination(scenario)
    print(f"\n[Step 3] 协调模式: {coord}")

    # Step 4
    controls = step4_bind_control(scenario)
    print(f"\n[Step 4] 控制面: {', '.join(controls)}")

    # Step 5: 序列化构建
    print(f"\n[Step 5] 构建序列:")
    print(f"  1. 状态 schema + 可观测性面板 (不是 Agent！)")
    print(f"  2. Gate (P4) + 审计日志 (退款操作必须完整记录)")
    print(f"  3. 编排器 + 第一个子 Agent (查订单)")
    print(f"  4. 剩余子 Agent (发退款、发通知)")
    print(f"  5. P6 控制面: 杀开关→升级(大额)→审批(>500元)→限流")

    # ADR 摘要
    print(f"\n{'='*60}")
    print(f"  六行 ADR 摘要")
    print(f"{'='*60}")
    print(f"  运行时: {rt.value}")
    print(f"  脊梁:   {backbone} ← 注意: 虽非Long-Horizon但退款有副作用")
    print(f"  协调:   {coord}")
    print(f"  控制:   {', '.join(controls)}")
    print(f"  序列:   面板先行")
    print(f"  关键风险: 退款Verifier必须有金额上限+人工审批 (P6)")

evaluate_customer_service_agent()
```

## 自测三层

**L1 记忆**：SDB 四个部件分别是什么？六种模式的编号和名称？五步选型中哪一步是迁移成本最高的？

**L2 理解**：为什么"回放漂移"是 LLM Agent 特有的架构失败模式（传统事件溯源没有）？SDB 最强的实现（MetaGPT）和最弱的实现（openai/swarm）的差异在哪个部件上体现最明显？

**L3 延伸**：如果你的业务需要从 Conversational Agent 升级到 Long-Horizon Agent（如从"客服对话"升级到"自动处理一个持续3天的退货流程"），SDB 的哪些部件会被打破？哪些模式选择需要推翻重来？为什么作者要求脊梁选择需要"高级工程师书面确认"？

---

知识时间锚 2026-05（Stanford / O'Reilly）。SDB 命名原语 + 6 种模式 + 5 步选型方法论构成了截至 2026 年最系统的 Agent 架构模式选型框架。与 [[136]] LIFE 框架的 F 阶段互补——SDB 处理单 Agent 的边界安全，LIFE-F 处理多 Agent 的错误传播链。

[[Wiki/论文笔记/03_Agent系统设计/122_代码作为Agent框架调查]] [[Wiki/论文笔记/03_Agent系统设计/136_多Agent系统协作失败归因自进化综述]] [[Wiki/概念/04_Agent框架/随机确定边界]] [[Wiki/概念/04_Agent框架/Harness设计模式]]
