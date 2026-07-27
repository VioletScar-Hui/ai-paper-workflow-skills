---
tags: [论文笔记, Agent, 代码Agent, 多智能体, 系统设计, 综述]
paper_id: "122"
filename: "122 - Code as Agent Harness.pdf"
authors: "Xuying Ning, Katherine Tieu et al. (UIUC + Meta + Stanford)"
year: 2026
成熟度: 🧪
笔记层级: 骨干
复核日期: 2026-07-04
---

# 代码作为 Agent 框架（Code as Agent Harness）：综述

📄 **原文 PDF**：[[RAW/122 - Code as Agent Harness.pdf]]

## PM 速判（30秒）
> 一句话：这篇 102 页综述（arXiv 2605.18747）把"代码"从 AI 的**输出产物**重新定义为 Agent 的**运行基础设施**（Harness）——可执行、可检查、有状态三属性解释了为什么编程 Agent（Claude Code/Codex）是当前最强的 Agent 类型。PM 必须懂这个框架，因为它直接回答"下一个 Agent 产品的护城河在模型还是在 Harness"。

## 双层费曼 🗣️
> **给 CEO 的一句话**：AI 助手真正可靠的秘密不是"更聪明的大脑"，而是给它配一个"工作台"——能动手试、能看到错在哪、能记住做到哪一步；代码恰好是目前最好的工作台，所以写代码的 AI 最先成熟，其他行业谁先造出等价的工作台，谁先拿到可靠的 AI Agent。
>
> **给工程师的一段话**：论文区分三个耦合要素：模型内在能力、系统预置的 harness 基础设施（工具/沙箱/权限/遥测）、以及**Agent 自建代码工件**（临时脚本、回归测试、PLAN 文件、技能库）——第三类是综述的核心焦点。框架分三层：①Harness 接口（代码承载推理外化、行动接口、环境建模）；②Harness 机制（规划=合约形成、记忆与上下文工程、工具使用、Plan-Execute-Verify 控制环）；③Harness 扩展（多 Agent 在共享代码库上按 Manager/Planner/Coder/Reviewer/Tester 分工协作）。PEV 环中 harness 扮演"控制论调节器"：用 linter/测试/CI 等**确定性传感器**观测状态，决定继续/修订/终止。

## 问题域定位 🎯
- **根本约束**：LLM 是无状态的、输出不可验证的采样器；长程自主任务需要状态持久化和可证伪的进度信号，纯上下文窗口 + 自然语言推理提供不了。
- **之前的方案卡在哪**：ReAct 式循环把错误信息直接回喂模型，缺少确定性验证步骤；已有综述只把代码当"生成目标"研究（代码生成质量），没人系统研究"代码作为 Agent 的运行介质"。
- **开启/关闭的路线**：开启"Harness 工程"作为独立学科（与 Anthropic/OpenAI 2025-26 工程博客呼应）；对非代码领域（法律/医疗/科研）提出明确门槛——先找到等价的"可执行规范语言"，否则达不到编程 Agent 的成熟度。

## 核心机制

```
                 代码三属性 → 为什么代码是最好的 Harness
   可执行 Executable ── 意图跑一下就能验证（可证伪）
   可检查 Inspectable ─ 执行轨迹/报错日志可被 Agent 自己诊断
   有状态 Stateful ──── 程序/文件状态跨步骤存活（不怕上下文重置）

三层框架与 PEV 控制环：
┌──────────────────────────────────────────────────────┐
│ Layer 3  Scaling: 多Agent 共享代码库/测试/轨迹          │
│   角色: Manager/Planner/Coder/Reviewer/Tester         │
│   拓扑: 集中式 / 分布式 / 流水线(Streamlined)           │
├──────────────────────────────────────────────────────┤
│ Layer 2  Mechanisms: PEV 环（harness = 控制论调节器）    │
│                                                       │
│   Plan(合约形成) ──→ Execute(沙箱+权限分层) ──→ Verify   │
│      ↑  计划=文件/涉及范围/验收命令/回滚点      │        │
│      └────── 验证失败 → 更新计划 ←─────────────┘        │
│   权限三层: 只读 → 沙箱编辑 → 完全访问(必须人工门)        │
│   确定性传感器: linter/类型检查/单测/CI/fuzzer           │
├──────────────────────────────────────────────────────┤
│ Layer 1  Interface: 代码承载 推理外化 / 行动 / 环境建模   │
│   PAL, Chain of Code / Code-as-Policies / SWE-bench   │
└──────────────────────────────────────────────────────┘
```

## 设计决策解剖 ⚖️
| 决策点 | 他们的选择 | 被放弃的方案 | 为什么 | 这个选择何时失效 |
|--------|----------|------------|--------|---------------|
| 综述的研究对象 | Agent 自建代码工件（临时工具/测试/计划文件） | 只研究系统预置工具（MCP/API 目录） | 预置工具已被大量研究；Agent 运行中自己造的工件才是长任务适应性的来源，且严重欠研究 | 若未来模型上下文足够长且推理足够稳，"自建工件"可能退化为纯提示工程 |
| 控制环 | Plan-Execute-Verify（确定性传感器验证） | ReAct（观察直接回喂）、LLM-as-judge 为主的验证 | 自然语言批评不可复现；linter/测试/CI 是确定性的，可作为控制信号；LLM 批评只应"解读传感器输出"而非替代 | 无法写出确定性检查的领域（创意写作、开放问答）——传感器不存在时 PEV 退化为 ReAct |
| 执行权限 | 三层分级：只读 / 沙箱编辑 / 完全访问+强制 HITL | 单一沙箱（全隔离）或全信任 | 低风险观察和高风险副作用的成本不对称；完全访问层（网络/凭证/部署/Git 历史）后果溢出沙箱，必须人工门 | 高频自动化场景（每天数千次部署）人工门成为吞吐瓶颈，需要策略引擎替代人 |
| 多Agent协调基底 | 形式化共享基底（持久文件存储/仓库+测试） | 复杂自适应拓扑（动态 DAG、工作流变异） | 综述 4.4 节发现：**拓扑复杂度与共享状态形式化程度负相关**——L2MAC 有清晰文件基底就只需简单顺序链；EvoMAC/SEW 缺基底才需要复杂拓扑补偿 | 任务本身需要对抗性/辩论式交互（红队、多方案竞争）时，简单拓扑不够 |
| 计划的形态 | 计划=harness 工件（文件里的合约：涉及文件/不变量/验收命令/回滚点） | 计划=不可观察的推理痕迹（CoT 里想一想） | 文件化计划约束后续行动空间、跨上下文重置存活、可被人审核 | 秒级短任务：写计划文件的开销超过任务本身 |

## 成本与量级 💰
- 训练成本：零（综述不训练模型；harness 是推理时工程）。
- 推理成本 vs 基线：PEV 环意味着每个任务 3-10 倍以上的模型调用（计划+执行+验证+修订）；沙箱执行/测试的算力另计但通常远低于 LLM 调用成本。
- 我的产品要用：最小可行配置 = Docker 沙箱 + pytest（或任意确定性检查命令）+ 一个 PLAN.md 约定 + 权限白名单。一台开发机即可，无 GPU 需求。

## 证据审计 🔬
- 实验设计公平吗：这是综述，**没有受控实验**——三层分类法和 4.4 节的模式归纳是作者对文献的解读，存在选择性引用风险；40+ 位作者（含 Meta 团队）意味着覆盖广但立场可能偏向工业实践叙事。
- 最强证据：4.4 节的横向归纳"拓扑复杂度与共享状态形式化程度负相关"（L2MAC vs EvoMAC/SEW 的对比）——这是综述少有的、可被后续实验证伪的具体命题。
- 最可疑的数字：本文几乎不给数字，最可疑的是**无数字的核心断言**"编程 Agent 最成熟因为代码是最好的 harness"——它与"代码领域恰好有最多基准和训练数据"混淆，因果方向未被排除。
- 如果我是审稿人，会要求补充：同一任务分布上 PEV vs ReAct 的受控对比；"确定性传感器覆盖率"与任务成功率的定量关系；至少一个非代码领域移植三属性的案例研究。
- 最小复现实验：取 20 个小任务，同一模型分别跑"无验证单轮生成"vs"带 pytest 验证的 PEV 循环"，比较成功率。预算：一个周末 + 约 $5-20 API 费。

## 可复用点（PM 决策）
- **何时采用**：长任务（>30 分钟）、有天然验证信号（测试/编译/schema）的领域 → 立即套 PEV + 计划文件 + 权限分层。
- **何时规避**：秒级对话任务（PEV 开销不划算）；无确定性验证信号的创意领域（先解决"传感器"问题再谈 harness）。
- 供应商拷问清单：
  1. "你们 Agent 的 Verify 步骤用什么确定性传感器？如果只有 LLM 自评，凭什么说可靠？"
  2. "上下文重置后 Agent 怎么恢复任务？计划和状态存在哪个文件/存储里？"
  3. "哪些操作在完全访问层？人工审批门在哪一步、能否被 Agent 绕过？"

## 关联网络 🕸️
- [[Wiki/论文笔记/03_Agent系统设计/128_生产LLM-Agent运行架构模式方法论]] → 印证：SDB 四部件（提案/验证/提交/拒绝）正是本文 PEV 环在生产架构层的形式化；两文互为理论-实践对照。
- [[Wiki/论文笔记/03_Agent系统设计/34_Agent-Harness有效反馈计算扩展律]] → **约束/张力**：本文鼓励更丰富的 harness 机制与更多执行循环，但 Paper 34 证明原始工具调用次数与成功率 R²≈0.14——堆机制不等于堆有效反馈，PEV 的价值只在于它提高了反馈的有效性（V 因子），不在于多跑几圈。
- [[Wiki/论文笔记/03_Agent系统设计/126_弱推理模型的Agentic增强]] → 印证：执行/测试信号增强弱模型 = Verify 步骤的实证支持。
- [[Wiki/论文笔记/04_Agent技能工具/12_SKILLWEAVER组合技能路由]] → 关系：技能库是本文"长期记忆=代码工件"的具体实现。
- [[Wiki/论文笔记/14_AI经济/109_Agentic-AI工作委托转型]] → 关系：工作委托的可行边界由 harness 成熟度决定。
- 相关概念：[[Wiki/概念/04_Agent框架/Harness设计模式]]、[[Wiki/概念/04_Agent框架/动态智能体脚手架]]、[[Wiki/概念/04_Agent框架/ReAct与Agent推理行动框架]]

## 动手练习 💻（30-45分钟）
练习目标：亲手验证本文核心论点——同一任务，让 LLM 通过"固定 JSON 工具调用"完成 vs 让它"直接写 Python 代码执行"完成，对比成功率，体会"代码=更强的行动接口"。

```python
# 依赖: pip install openai
# 运行前设置环境变量 OPENAI_API_KEY（也可改 base_url 用兼容服务如 DeepSeek）
import json                          # 解析模型返回的 JSON 工具调用
import re                            # 用正则从回复中提取代码块
from openai import OpenAI            # OpenAI 官方客户端（兼容多数国产 API）

client = OpenAI()                    # 创建客户端，自动读取环境变量里的 API key
MODEL = "gpt-4o-mini"                # 用便宜的小模型，差距才明显

# ---- 5 个测试任务：每个任务给定输入数据和标准答案 ----
TASKS = [
    ("求列表中所有质数的和", [12, 7, 9, 23, 15, 11, 4], 41),
    ("求列表的中位数", [3, 1, 4, 1, 5, 9, 2, 6], 3.5),
    ("统计列表中出现次数最多的数字出现了几次", [1, 2, 2, 3, 3, 3, 4], 3),
    ("求列表中相邻两数之差的最大绝对值", [10, 3, 8, 20, 1], 19),
    ("求列表中能被3整除的数的乘积", [2, 3, 6, 7, 9], 162),
]

# ---- 方式A：JSON 工具调用——模型只能用两个固定的简单工具 ----
TOOLS_DOC = """你只能通过 JSON 调用以下工具，每次回复一个 JSON:
{"tool": "filter", "keep_if": "<对单个元素x的Python布尔表达式>"}  # 过滤列表
{"tool": "reduce", "op": "sum|max|min|count"}                    # 聚合当前列表
{"tool": "answer", "value": <数字>}                              # 提交最终答案
注意: 工具无法排序、无法两两比较相邻元素、无法求乘积。"""

def run_json_agent(task, data, answer, max_steps=6):
    """方式A: 模型只能发 JSON 工具调用，工具能力故意有限（模拟贫瘠接口）"""
    cur = list(data)                                  # 当前工作列表
    msgs = [{"role": "user", "content":
             f"{TOOLS_DOC}\n任务: {task}\n初始列表: {data}"}]
    for _ in range(max_steps):                        # 最多允许 6 步
        r = client.chat.completions.create(model=MODEL, messages=msgs)
        text = r.choices[0].message.content           # 取出模型回复文本
        try:
            call = json.loads(re.search(r"\{.*\}", text, re.S).group())
        except Exception:
            return False                              # JSON 格式错 → 直接失败
        if call.get("tool") == "answer":              # 模型提交答案
            return abs(float(call["value"]) - answer) < 1e-6
        if call.get("tool") == "filter":              # 执行过滤工具
            try: cur = [x for x in cur if eval(call["keep_if"])]
            except Exception: pass
        elif call.get("tool") == "reduce":            # 执行聚合工具
            ops = {"sum": sum, "max": max, "min": min, "count": len}
            cur = [ops.get(call.get("op"), sum)(cur)] if cur else [0]
        msgs.append({"role": "assistant", "content": text})   # 记录历史
        msgs.append({"role": "user", "content": f"当前列表: {cur}"})
    return False                                      # 步数耗尽 → 失败

def run_code_agent(task, data, answer):
    """方式B: 让模型直接写 Python 代码，我们执行它（代码=行动接口）"""
    r = client.chat.completions.create(model=MODEL, messages=[
        {"role": "user", "content":
         f"写一段Python代码完成任务并把结果存入变量 result。\n"
         f"任务: {task}\n数据: data = {data}\n只回复代码块。"}])
    m = re.search(r"```(?:python)?\n(.*?)```", r.choices[0].message.content, re.S)
    if not m: return False                            # 没找到代码块 → 失败
    env = {"data": list(data)}                        # 给代码一个独立命名空间
    try:
        exec(m.group(1), env)                         # 执行模型写的代码
        return abs(float(env.get("result", 1e18)) - answer) < 1e-6
    except Exception:
        return False                                  # 代码报错 → 失败

# ---- 主实验：每个任务两种方式各跑一次，统计成功率 ----
score_a = score_b = 0
for task, data, ans in TASKS:
    a = run_json_agent(task, data, ans)               # 方式A: JSON 工具
    b = run_code_agent(task, data, ans)               # 方式B: 直接写代码
    score_a += a; score_b += b
    print(f"{task[:14]:<16} JSON工具:{'成功' if a else '失败'}  写代码:{'成功' if b else '失败'}")
print(f"\nJSON工具调用成功率: {score_a}/5   直接写代码成功率: {score_b}/5")
# 预期: 写代码明显更高——因为代码接口的表达力覆盖了任务需求，
# 而贫瘠的固定工具集让模型"有智力没有手"。这就是"代码作为行动接口"的含义。
```

## 自测三层 🎓
- **L1 复述**：代码的哪三个属性使其成为最优 Agent Harness？PEV 环比 ReAct 多了哪个步骤、用什么实现？
- **L2 解释**：为什么综述主张验证要用"确定性传感器"（linter/测试/CI）而不是 LLM-as-judge？在什么领域这个主张会失效？
- **L3 应用**：你要给律所做一个合同审查 Agent。套用本文框架回答：这个领域的"可执行规范"候选是什么？Verify 步骤能用什么确定性传感器？如果找不到，你会把产品边界收缩到哪里？

📅 知识时间锚：2026-05（arXiv 2605.18747v1，文献覆盖至 2026 年上半年；Claude Code/Codex/OpenHands 为当时代表系统）
