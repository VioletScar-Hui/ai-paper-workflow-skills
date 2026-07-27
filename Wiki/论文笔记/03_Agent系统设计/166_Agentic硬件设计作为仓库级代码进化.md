---
tags: [论文笔记, Agent系统设计, 硬件设计, RTL, EDA, 自进化Agent, Git原生Agent, NVIDIA]
paper_id: "166"
filename: "166 - Agentic Hardware Design as Repository-Level Code Evolution.pdf"
authors: "Cunxi Yu, Chenhui Deng, Nathaniel Pinckney, Brucek Khailany (NVIDIA Research)"
year: 2026
笔记层级: 骨干
成熟度: 🧪
复核日期: 2026-07-08
---

# HORIZON：把硬件设计当作仓库级代码进化来做的自进化 Agent 框架

📄 **原文 PDF**：[[RAW/166 - Agentic Hardware Design as Repository-Level Code Evolution.pdf]]

## PM 速判（30秒）
> 一句话：NVIDIA 提出 HORIZON——把一段人写的 Markdown"任务说明书"编译成一个自带评估器和验收门槛的 git 仓库，再让 Agent 在这个仓库里反复改代码、跑仿真、提交或打回，直到通过；在 ChipBench、RTLLM-2.0、Verilog-Eval 和 CVDP 九大类共 12 个基准套件上，全部**跑到 100% 通过率、全程无人工介入**。PM 必须关注：这是"代码即 Harness"思路第一次被系统性地搬到芯片设计（RTL）这种强时序/强正确性约束的专业领域，证明只要有"可执行的对错标准"，Agent 自进化范式就能跨出软件工程，向任何有"编译-仿真-验收"闭环的工程领域复制——但论文自己也承认，这离"解决芯片设计"还差得远，尤其是 reward hacking 和长反馈延迟两个问题目前无解。

## 双层费曼 🗣️
> **给 CEO 的一句话**：以前让 AI 写芯片电路代码，就是让它"蒙一次交卷"，对不对全靠运气；HORIZON 给 AI 配了一个"带考官的靶场"——AI 写完代码自己拿去仿真验证，考官说不通过就退回重写，AI 会一直改到通过为止，全程不用人盯着，结果是本该很难的题目也能被"磨"到 100% 通过，代价是有些难题要改上百次、烧几千万 token。
>
> **给工程师的一段话**：HORIZON 把一份结构化 Markdown harness 经 bootstrap agent 编译成 project pack `p=(π_agent, E_p, A_p, Γ_p, Ω_p)`（策略提示词、可执行评估器、验收谓词、git/运行时策略、领域知识），随后 agent 在一个隔离 git worktree 里跑"编辑→跑评估器→打分→按验收谓词提交或打回"的 hands-free 循环，git commit/diff/notes/log 本身就是免费的状态管理和可回放 trace buffer；作者借用半马尔可夫决策过程的记号（state=仓库快照、option=一次内部编辑-调用-审阅的变长片段、reward 向量含 Δpass/Δcoverage/-tokens）纯粹是为了给记录的对象命名，并不声称过程是马尔可夫的，也没有做任何 RL 训练——agent backbone（GPT-5.3）整个campaign 期间固定不变。

## 问题域定位 🎯
- **回应什么根本约束**：RTL（寄存器传输级硬件描述代码）生成不同于普通代码补全——正确性依赖周期级时序行为、复位约定、ready-valid 握手协议、位宽、边界情况，这些在自然语言里往往描述不全；"语法正确的 Verilog"离"能用的电路"还差一大截，必须靠编译-仿真-波形回溯的执行反馈才能收敛。
- **之前的方案卡在哪**：(1) RTL 专用模型路线（VeriGen、RTLCoder、ChipNeMo、OriGen、CraftRTL、ScaleRTL）只在提升生成器本身的首次准确率，不定义 agent 该如何针对可执行反馈迭代；(2) 迭代式/agentic RTL 路线（AutoChip、RTLFixer、VerilogCoder、MAGE、ACE-RTL）都是围绕单个模块的"生成-修复"流水线，没有把整个基准套件当作一个可版本管理的仓库来推进到完成；(3) 仓库级代码自进化此前只用于 EDA **软件系统**本身——AlphaEvolve 优化算法内核、SATLUTION 进化整个 SAT 求解器仓库、ABCEvo 重写百万行的 ABC 逻辑综合系统，进化的都是"工程师在跑的程序"，从未进化过"工程师设计的硬件产物"本身。
- **开启/关闭了哪条技术路线**：开启了"用 git 仓库同时承载 RTL 生成、补全、修改、验证生成、调试"多种任务的统一协议，理论上让任何有"编译/仿真/工具"可执行判据的芯片设计子问题（架构探索、验证规划、物理设计交互、EDA 软件、方法学）都能套用同一框架（但论文本身只实证了 RTL 这一支）；同时也把"reward hacking / 过拟合评估器"这个此前在 SWE-bench 系软件基准里已经被讨论过的问题，正式摆上了硬件 agent 基准的桌面，倒逼未来 RTL benchmark 需要像 SWE-bench 一样做"诊断反馈 vs 隐藏评分"分离。
- **基准本身的背景**：论文选用的 CVDP（Comprehensive Verilog Design Problems）是相对新的基准，包含 783 个人工撰写的问题，覆盖 13 个任务类别，其代码生成侧包括 RTL 代码补全、自然语言规格转 RTL、代码修改、模块复用、lint/质量改进、测试激励生成、检查器生成、断言生成、调试；经质量过滤后分为 617 个非 agentic 问题与 166 个 agentic 问题（后者打包成供 Docker 化 agent 检查文件/调用工具的迷你仓库）。CVDP 论文自身指出，现有单轮模型在验证类任务（测试激励、检查器、断言生成、bug 修复）上表现尤其差，这正是 HORIZON 选它作为"检验仓库级反馈能否超越首次尝试准确率"的理由。

## 核心机制

```
                         HORIZON 整体流程（对应论文 Figure 1）
┌───────────────────────────────────────────────────────────────────────┐
│ 人类输入：结构化 Markdown Harness                                       │
│   目标/objective ｜ 领域知识方向 ｜ 评估器规格 ｜ 验收谓词                │
└───────────────────────────┬───────────────────────────────────────────┘
                             │ Bootstrap Agent：p = G_φ(m)
                             ▼
┌───────────────────────────────────────────────────────────────────────┐
│ project_pack 控制面（编译一次，campaign 期间固定不变）                    │
│   π_agent 策略/工具契约 ｜ E_p 可执行评估器(编译/仿真/覆盖率/断言检查)     │
│   A_p 验收谓词 ｜ Γ_p 版本控制&产物策略 ｜ Ω_p 领域技能与仓库说明          │
└───────────────────────────┬───────────────────────────────────────────┘
                             ▼
        ┌───────────────────────────────────────────────────┐
        │           HORIZON Agent Loop（hands-free）           │
        │  ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌───────┐ │
        │  │任务/行动计划│→│文件编辑&  │→│评估&打分 │→│正确性 │ │
        │  │           │ │工具调用   │ │(E_p)    │ │门+复核 │ │
        │  └─────────┘  └──────────┘  └─────────┘  └───┬───┘ │
        │        ▲                                     │     │
        │        │      A_p(y_t)=1 → Commit(打勾)        │     │
        │        │      A_p(y_t)=0 → RejectLog(打回)      │     │
        │        └─────────────────────────────────────┘     │
        └───────────────────────────┬───────────────────────┘
                                     ▼
┌───────────────────────────────────────────────────────────────────────┐
│ Git 仓库：隔离 worktree W，状态迁移 S_w,t → S_w,t+1                       │
│   diff 暴露改动 ｜ commit 定义验收检查点 ｜ log 还原轨迹 ｜ notes 挂评估证据│
│   depth-k trace buffer：版本历史 + 状态增量 + 评估结果 + 运行时摘要        │
└───────────────────────────────────────────────────────────────────────┘
```

关键点：git 不是附带的记录工具，而是"隔离执行环境 + trace 基底"本身——git diff --cached 检查暂存改动、每次接受的尝试变成一次 commit（message/notes 携带评估器判决和 reward）、git log 还原完整版本历史、独立审阅步骤在提交前 diff 候选版本。**Agent 全程不修改 evaluator 或 acceptance predicate**——project_pack 在 bootstrap 阶段编译完成后即固定，agent 只能在给定的"考纲"下反复答题，不能改考纲。

**形式化记号（仅用于记录检索，不是行为假设）**：论文借用半马尔可夫决策过程的词汇给可回放对象命名，并明确声明 agent 本身是自由形式、依赖历史的 LLM 策略，不假设其行为是马尔可夫的。在外层检查点 t：
- 状态 `s_t = (tree(w_t), p, z_t, ℓ≤t, μ_t)`——当前 worktree 的 git tree、project pack、campaign 状态、累计日志/评估产物、策略可条件化的声明式记忆；论文强调这只是"记账用的检查点"，不是 agent 推理的充分统计量。
- 一次 option（一次接受版本之间的变长片段）：`a_t = (Δt, u_t,1:Kt, ρt)`——提出的补丁/生成的产物集合、内部 Kt 次工具调用与观测、最终审阅/提交决定。
- 评估器产出证据：`y_t = E_p(w_t ⊕ Δt)`；验收规则为分段函数——`A_p(y_t)=1` 时 `Commit(w_t ⊕ Δt, y_t, Γp)`，`A_p(y_t)=0` 时 `RejectLog(s_t, Δt, y_t)`。
- 奖励可为标量或向量：`r_t = R_p(y_t) = (Δpass, Δcoverage, ΔQoR, -tokens, -time)`，某个分量只有在评估器真正给出对应信号时才会被填充；本文只报告 Δpass、Δcoverage、-tokens 三项，ΔQoR 留作未来工作。
- 一条深度为 D 的执行轨迹：`τ_0:D = {(s_t, a_t, r_t, s_t+1, y_t)}`，D 不由 benchmark 固定，而由 campaign 预算/收敛/停止规则决定——这使得该 trace 天然适合用于策略分析、reward 建模、课程构造或离线 agent-RL 训练，但论文本身**没有做**这最后一步，只是把接口设计出来。

## 设计决策解剖 ⚖️

| 决策点 | 他们的选择 | 被放弃的方案 | 为什么 | 这个选择何时失效 |
|---|---|---|---|---|
| 状态与轨迹管理 | 直接用原生 git（diff/commit/log/notes）做 trace buffer | 单独搭建外部数据库/日志系统记录每步状态 | git 本身免费提供版本化、可回放、可 diff 的记录，无需额外基础设施；"评论"用 git notes 挂评估证据 | 当任务状态不是文件系统可表示的对象（如物理版图连续优化的大规模二进制布局），或内部工具调用粒度远细于 commit 边界、commit 层面已丢失关键中间信息时 |
| 验收门槛 | 直接复用 benchmark 自带的 pass/fail harness 作 acceptance predicate，而非另设 coverage/formal 目标 | 把 RTL 问题重构成一个软件工程代理任务（如转成单元测试） | 保留基准"原样"以便任何 backbone 都能在同一协议下被评测收敛速度，而非只评测首次尝试准确率 | 当 benchmark 自身 harness 有规格缺陷时（论文里 ChipBench 恰有 1 个任务因 spec-harness 不匹配无法通过），"通过=正确"这一等价关系直接失效 |
| 反馈可见性 | agent 全程可见 simulator 消息、评估日志、失败 trace（诊断反馈与最终评分同一套 harness） | 像 SWE-bench 一样把 fail-to-pass/pass-to-pass 隐藏测试与调试期反馈分离 | 更贴近真实工程师调试工作流，让长程修复可行；论文明确承认这是留给未来的开放问题 | 当 agent 针对"可见的" evaluator 特异性做过拟合修复（论文称为 over-solving），"通过可见 harness" 不再等价于"满足任意合理测试下的规格"，尤其在检查器/断言生成这类高风险验证任务上 |
| Backbone 与多智能体 | 单一固定 backbone（GPT-5.3）+ 单 agent 模式贯穿全部实验 | 像 MAGE 那样多 agent 分工协作(检索/修复/校验分角色) | 把"仓库级反馈机制能否驱动收敛"作为唯一自变量剥离出来验证，避免多 agent 设计混淆结论 | 当单任务需要极大并行修复量时（如 CID 002 需要 82 次迭代、5600 万 token），单 agent 长 session 可能在 token/上下文成本上失控，多 agent 分工可能更省 |
| Reward 的取用范围 | 只报告 Δpass、Δcoverage、-tokens 三个分量，ΔQoR（面积/功耗/时序）明确留作未来工作 | 直接引入综合 PPA（性能-功耗-面积）reward 驱动收敛 | 本文聚焦"能否收敛到正确"这一更基础问题，PPA 评估的 turnaround 远慢于 pass/fail，混进来会拖垮当前实验节奏 | 真实芯片项目里 PPA 权衡才是核心价值点，若长期不引入 QoR reward，agent 可能收敛到"能跑但性能很差"的 RTL |

## 成本与量级 💰
- **训练成本**：无。论文明确声明"我们在这项工作中不训练或更新 RL 策略，agent backbone 在整个 campaign 期间固定"——HORIZON 是纯推理时 agentic 循环，没有任何模型训练/微调步骤。
- **推理成本（token）**：全部 12 个套件跑到"earliest-best iteration"共消耗 **209.9M tokens**；其中三个传统套件（ChipBench+RTLLM-2.0+Verilog-Eval）合计仅 6.0M（2.9%），九个 CVDP 类别合计 203.9M（97.1%）。单类别最贵的是 CID 002（RTL 代码补全）56.0M token（26.7%）、CID 003（自然语言到 RTL）38.0M（18.1%）、CID 012（激励生成）32.2M（15.3%）。关键结构性数字：**约 91% 的 token 是缓存命中的输入 token**（靠持久化 model session 复用 harness/project pack/历史调试上下文），这是控制边际迭代成本的核心手段。论文未披露 $ 计价的 API 费用，也未披露具体缓存定价假设。
- **硬件/算力**：所有 campaign 跑在单台 **AMD EPYC 9334 32 核处理器 + 512GB RAM** 的主机上，无需 GPU——因为算力开销主要在开源/商业 EDA 工具的编译与仿真，而非模型本身的推理（推理走 API）。CVDP CID 012–014（激励/检查器/断言生成）需要商业仿真器，其余套件用开源 EDA 工具链即可。
- **最小可行配置**：论文未给出显式的"最小配置"表述；可推断的下限是——一个开源 RTL 仿真/编译工具链 + 一个支持长上下文/prompt cache 的模型 API + 单台多核 CPU 主机（无需 GPU），即可复现开源子集（ChipBench/RTLLM/Verilog-Eval 及 CVDP 非商业类别）。
- **Table 2 token 消耗明细**（截至各套件的 earliest-best iteration，单位百万 token）：

| 套件/类别 | 收敛迭代 | Token(M) | 占比 |
|---|---|---|---|
| ChipBench | 5 | 2.8 | 1.3% |
| RTLLM-2.0 | 2 | 1.3 | 0.6% |
| Verilog-Eval v2 | 2 | 2.0 | 1.0% |
| CVDP CID 002 | 82 | 56.0 | 26.7% |
| CVDP CID 003 | 24 | 38.0 | 18.1% |
| CVDP CID 004 | 36 | 23.7 | 11.3% |
| CVDP CID 005 | 14 | 9.1 | 4.4% |
| CVDP CID 007 | 24 | 21.6 | 10.3% |
| CVDP CID 012 | 32 | 32.2 | 15.3% |
| CVDP CID 013 | 19 | 14.2 | 6.7% |
| CVDP CID 014 | 1 | 0.3 | 0.1% |
| CVDP CID 016 | 13 | 8.8 | 4.2% |
| **合计** | — | **209.9** | **100.0%** |

可以看到成本高度集中在少数困难的 CVDP 类别（三个传统套件合计仅 2.9%），这也是论文得出"token 效率而非最终通过率才是更值得改进的指标"这一结论的直接依据。

## 证据审计 🔬
- **实验设计公平吗？基线选取有什么猫腻？** 论文里唯一的"基线"是"iteration 0"（agentic 循环跑完第一步之后的仓库状态），并且作者反复强调这**不是**标准的模型 Pass@1/零样本单轮生成结果，而是同一套 prompting 策略跑了一步之后的产物——这意味着表里没有任何一个真正意义上的"裸模型单轮基线"数字，也没有和 MAGE、ACE-RTL、VerilogCoder 等已发表的 agentic RTL 系统在同一 benchmark 上做直接数值对比（论文只在 Related Work 里文字提及这些系统，从未放进 Table 1 做横向比较）。这是评审时最容易被抓的公平性缺口。
- **最强证据**：单一 hands-free agent 循环在 ChipBench、RTLLM-2.0、Verilog-Eval 和全部九个 CVDP 类别上都收敛到 **100% 通过率**（Table 1），且长尾类别（CID 002 需 82 次迭代、CID 013 从 3.8% 首轮通过率出发但呈近线性稳定爬升到 100%）显示的是真实的渐进式修复过程，而非侥幸的一次性通过。成立条件：这一结果建立在"agent 全程可见 evaluator 详细失败信息"这一反馈接口之上，尚未在隐藏测试/扰动测试下验证鲁棒性。论文 Table 1 完整数据摘录如下（Iter.0 为 agentic 循环跑完第一步之后的仓库状态，并非裸模型 Pass@1）：

| 套件/类别 | 评估重点 | EDA 后端 | Iter.0 通过率 | 收敛迭代 | HORIZON 最终 |
|---|---|---|---|---|---|
| ChipBench | 混合 RTL 生成任务 | 开源 | 20.0% | 5 | 100%（1 题因 spec-harness 缺陷，计入后视为已解决） |
| RTLLM-2.0 | 自然语言规格转 RTL | 开源 | 78.0% | 2 | 100% |
| Verilog-Eval v2 | HDLBits 风格 Verilog 生成 | 开源 | 86.2% | 2 | 100% |
| CVDP CID 002 | RTL 代码补全 | 开源 | 3.2% | 82 | 100% |
| CVDP CID 003 | 自然语言规格转 RTL | 开源 | 19.2% | 24 | 100% |
| CVDP CID 004 | RTL 代码修改 | 开源 | 10.9% | 36 | 100% |
| CVDP CID 005 | 规格转 RTL 的模块复用 | 开源 | 9.1% | 14 | 100% |
| CVDP CID 007 | Lint/QoR 改进 | 开源 | 0.0% | 24 | 100% |
| CVDP CID 012 | 测试计划转激励生成 | 商业 | 47.8% | 32 | 100% |
| CVDP CID 013 | 测试计划转检查器生成 | 商业 | 3.8% | 19 | 100% |
| CVDP CID 014 | 测试计划转断言生成 | 商业 | 79.1% | 1 | 100% |
| CVDP CID 016 | 调试与 bug 修复 | 开源 | 25.7% | 13 | 100% |
| **整体** | 全部评测 RTL 基准 | — | **47.8%** | — | **100%** |
- **最可疑的数字**：CID 007（linting/QoR improvement）**iteration 0 通过率为 0.0%**——整个类别在第一步agentic迭代后没有一个任务通过。这既可能反映该任务类别确实极难（QoR/lint 改进本身缺乏明确的"正确性"信号，更依赖代码风格/质量判断），也可能是初始 project pack 编译或首轮策略设定上的系统性问题；论文没有对这个"整类别 0% "的现象做专门归因分析，只在 Table 1 里一带而过。
- **审稿人会要求补充**：①在隐藏/随机扰动测试下的鲁棒性数据（论文自己在 Section 5 里承认这是缺失的）；②与 MAGE/ACE-RTL/VerilogCoder 等既有 agentic RTL 系统在相同 benchmark 上的直接数值对比表；③以 $ 计价而非仅 token 计价的真实成本；④更换 agent backbone（非 GPT-5.3）后收敛率/成本的敏感性分析；⑤对"通过的设计"做独立形式化等价检查，而不仅是通过 benchmark 自带 harness。
- **一个容易被忽视但很诚实的补充证据**：Table 3/Figure 4 显示 HORIZON 的验收门槛是 pass/fail，而不是覆盖率目标——CID 012（测试激励生成）在 iteration 0 时平均覆盖率 86.5%、通过率 47.8%，到 iteration 32 收敛时通过率到 100%，但平均覆盖率只爬升到 97.9%，并未到 100%；这是"只要通过 CVDP 自带的 harness 就停止"这一验收规则的直接后果，论文自己坦诚说明"覆盖率是观察性的次要信号，不是被优化的目标"。CID 014（断言生成）则相反，起点覆盖率就有 98.1%，1 次迭代即饱和到 100%，且这个 100% 只是在 67 个设计里有解析出覆盖率日志的 60 个设计上计算的，另外 7 个设计没有可解析的覆盖率报告——这个"7 个设计缺失日志"的细节容易被读者忽略，但恰恰说明"100% 通过率"和"100% 验证充分性"是两回事。
- **最小复现实验**：取 CVDP CID 002（RTL 代码补全，论文里 iteration 0 通过率仅 3.2%、最终需 82 次迭代收敛）的一个 10-15 题小子集,换用一个更便宜的 backbone（如 GPT-5-mini 级别模型),跑 20 次迭代,记录 best-so-far pass rate 曲线和累计 token 消耗,与官方论文的 3.2%→100%/82 次迭代/56.0M token 做形状对比（收敛速度是否类似、是否出现同样的长尾）。预算：以 CPU 仿真为主，几十次 API 调用，预计几十万到百万级 token,成本远低于全量复现。

## 可复用点（PM决策）
- **何时采用**：团队已经有一套"可执行的对错标准"——编译器、仿真器、lint 工具、正式的 testbench/assertion 检查——且当前瓶颈是"人工反复跑仿真-改代码-再跑"这种体力密集的迭代循环,而不是模型本身生成能力不足时,HORIZON 这类"harness→git worktree→hands-free 循环"的模式值得直接借用,尤其适合 RTL 补全/修改/测试激励生成这类有backlog 积压的工作。
- **何时规避**：①正确性判据本身评估周期是天/周级(如物理设计的时序收敛、功耗签核)——论文自己承认这类长延迟反馈会让"朴素的编辑-评估-修复循环"变得太慢,需要完全不同的agent设计；②任务的"通过"标准存在明显的可被钻空子的空间(检查器/断言生成、安全关键硬件)而又没有隐藏测试/独立参考模型做兜底验证时,100% 通过率可能只是"满足了可见 harness"而非真正满足规格。
- **供应商拷问清单**：
  1. 你们宣称的"通过率"是在 agent 全程可见 evaluator 日志/失败信息的条件下达到的,还是在隐藏的/随机扰动过的测试集上验证过？有没有做过"过拟合评估器"的专项检测？
  2. 收敛所需的平均迭代次数和 token/时间成本分布是什么样的？是否存在类似论文里 CID 002 那样 82 次迭代、5600 万 token 的长尾类别,你们打算怎么设定超时或预算上限来兜底？
  3. 如果换成我们自己的 EDA 工具链、我们自己的 agent backbone(而非你们测试用的那一个固定模型),收敛率和成本会有多大波动？有没有做过 backbone 敏感性测试或跨工具链的可移植性验证？

## 关联网络 🕸️
- **相关论文**：[[Wiki/论文笔记/03_Agent系统设计/43_AutoScientists自组织Agent团队科学发现]]——两者都是"长视野、自主迭代、靠反馈信号驱动收敛"的自进化 agent 系统,但形态几乎相反：AutoScientists 是去中心化多 agent 团队,围绕生物医学假设自发组队、靠共享的自然语言状态(Champion/论坛/死路记录)做模糊的假设评审,没有 git 原生的状态基底,验收标准是排名百分位这种连续、软性的信号；HORIZON 是单 agent、git 原生、面向 RTL 这种有严格 pass/fail 二元验收门槛的工程领域。两者共同印证"自主迭代+可信反馈信号"是当前 agent 长程任务的核心配方,但"反馈信号是连续排名还是二元通过"决定了系统该往"多agent 群体探索"还是"单agent 仓库级收敛"设计。
- [[Wiki/论文笔记/03_Agent系统设计/122_代码作为Agent框架调查]]——这篇综述提出"代码的可执行/可检查/有状态三属性使其成为最好的 agent harness"这一论点,HORIZON 正是该论点在硬件设计领域的一次具体延伸：把 RTL 源码、testbench、验证产物当作"代码"本身来仓库化演化,git 仓库承担的正是 122 号笔记里说的"Harness 接口+机制"角色。差异在于 122 关注软件工程通用框架,HORIZON 把同一套逻辑第一次系统地搬到了强时序约束的硬件描述语言领域。
- [[Wiki/论文笔记/03_Agent系统设计/32_Harness更新vs获益-自进化Agent能力解耦]] 与 [[Wiki/论文笔记/03_Agent系统设计/22_Self-Harness-Agent自我改进运行框架]]——**冲突/印证**：这两篇论文都主张"让 agent 在运行中自我修改/更新 harness(提示词、工具、评估策略)本身"能带来性能提升,32 号甚至发现小模型就能胜任"产生 harness 更新"这一角色。但 HORIZON 的设计恰恰相反——project_pack(评估器 E_p、验收谓词 A_p)在 bootstrap 阶段编译一次后,整个 campaign 期间**固定不变**,agent 只能在给定的"考纲"下产生代码/修复尝试,不能修改判分标准本身。这是两种设计哲学的直接张力：动态自改进 harness 追求更强适应性,而 HORIZON 选择"不让 agent 碰评分标准",这恰恰回应了论文 Section 5 里对 reward hacking 的担忧——如果连 acceptance predicate 都能被 agent 改动,作弊空间会比"仅仅利用可见反馈过拟合"大得多。这提示一个可迁移的设计原则：**自我改进的对象(生成的产物)和裁判的对象(评估标准)必须结构性分离**,否则自进化系统很容易滑向"既是运动员又是裁判"的风险。
- [[Wiki/论文笔记/03_Agent系统设计/139_AEVO元编辑Agent进化框架]]——AEVO 更进一步,让"元 agent"直接编辑"进化机制本身"(搜索/上下文管理策略),与 HORIZON"固定 project_pack、只演化仓库内容"形成鲜明对照,可作为"该不该让 agent 碰自己的进化规则"这一设计谱系上的另一端参照。
- **相关概念**：[[Wiki/概念/04_Agent框架/Harness设计模式]]、[[Wiki/概念/02_训练方法/递归自我改进]]——HORIZON 的 project_pack 编译流程和固定-evolving 分离,是"Harness设计模式"在硬件设计场景下的具体实现；而其"迭代收敛但不做模型层面自我改进"的做法,恰好是"递归自我改进"这一概念谱系里偏保守、偏工程可控的一端。
- **背景关联（库外文献，仅作上下文，不建 wiki-link）**：论文的直接技术谱系是 AlphaEvolve（LLM+自动评估器优化算法内核）→ SATLUTION（把同一思路扩展到整个 SAT 求解器仓库,连进化规则本身也自我进化,最终超过人类设计的 SAT 竞赛冠军方案）→ ABCEvo（多 agent 协同重写百万行 ABC 逻辑综合系统）——这三者进化的都是"工程师在跑的 EDA/科学软件"，HORIZON 是这条线第一次把进化对象换成"工程师设计的硬件产物本身"。另一条相关谱系是"把 git 本身当 agent 脚手架"：EvoGit 用纯 Git 谱系图协调去中心化 agent 群体（无共享内存/消息传递），Git Context Controller 把 agent 上下文从瞬时 token 流升级为带 COMMIT/BRANCH/MERGE 的持久化版本记忆——两者都面向软件工程，HORIZON 是把同一套 git 语义用在"承载单个硬件设计问题"这一新目的上。这些论文目前均未收录入本知识库，如后续消化可补充 wiki-link。

## 动手练习 💻
练习目标：用一个高度简化的"仓库级代码进化"模拟,复现 HORIZON 核心循环的骨架——候选修改(patch)→可执行评估器打分→验收谓词决定 commit 或 reject→trace buffer 记录版本历史,直观体会"为什么修复轨迹会有长尾"。

```python
import random

# 用一个简化的"设计规格"代替 RTL:目标是让一个整数列表变成严格递增序列
# 这里的"仓库状态"就是当前候选列表,"通过"条件是严格递增且长度不变
TARGET_LEN = 8

def make_initial_state():
    # 初始"设计"故意乱序,模拟第一版 RTL 里常见的时序/逻辑错误
    return [random.randint(0, 20) for _ in range(TARGET_LEN)]

def evaluator(state):
    # E_p: 可执行评估器,返回 (是否通过, 未通过的"逆序对"数量)当作评估证据 y_t
    inversions = sum(
        1 for i in range(len(state) - 1) if state[i] >= state[i + 1]
    )
    passed = inversions == 0
    return passed, inversions

def acceptance_predicate(y_t):
    # A_p: 验收谓词,只有 evaluator 判定"通过"才允许 commit
    passed, _ = y_t
    return passed

def propose_patch(state):
    # 生成候选修改 delta_t:随机交换一对相邻元素,模拟 agent 的一次局部修复尝试
    new_state = state[:]
    i = random.randint(0, len(new_state) - 2)
    new_state[i], new_state[i + 1] = new_state[i + 1], new_state[i]
    return new_state

def horizon_loop(max_iters=500):
    state = make_initial_state()
    trace_buffer = []  # 对应论文里的 depth-k trace buffer,记录每次迭代的 (状态, 评估结果, 是否commit)
    best_inversions = evaluator(state)[1]

    for t in range(max_iters):
        candidate = propose_patch(state)          # a_t 的核心部分:候选 patch
        y_t = evaluator(candidate)                  # y_t = E_p(w_t ⊕ Δt)
        accepted = acceptance_predicate(y_t)

        if accepted:
            # A_p(y_t) = 1 -> Commit:接受新版本,推进仓库状态
            state = candidate
            trace_buffer.append((t, "commit", y_t))
            print(f"[iter {t}] COMMIT  逆序对={y_t[1]}  最终态={state}")
            break
        else:
            passed, inversions = y_t
            if inversions < best_inversions:
                # 即使没有完全通过,只要比当前最优更接近目标,也接受这次局部修复
                # (模拟论文里 best-so-far pass rate 的爬升过程,而非严格的一次性通过)
                state = candidate
                best_inversions = inversions
                trace_buffer.append((t, "improve", y_t))
            else:
                # A_p(y_t) = 0 且没有改进 -> RejectLog:打回,仅记录负样本
                trace_buffer.append((t, "reject", y_t))

    return state, trace_buffer

if __name__ == "__main__":
    final_state, trace = horizon_loop()
    commits = sum(1 for _, kind, _ in trace if kind == "commit")
    improves = sum(1 for _, kind, _ in trace if kind == "improve")
    rejects = sum(1 for _, kind, _ in trace if kind == "reject")
    print(f"总迭代记录数={len(trace)}  改进步数={improves}  打回步数={rejects}  最终是否收敛={commits > 0}")
```

代码要点说明：`propose_patch` 对应论文里 agent 每个 option 产生的候选补丁 Δt；`evaluator` 对应可执行评估器 E_p,返回的"逆序对数量"就是最粗糙版本的评估证据 y_t；`acceptance_predicate` 对应验收谓词 A_p，只有严格递增才算真正"通过"；trace_buffer 里区分 commit / improve / reject 三种记录方式,对应论文强调的"成功 commit 是正样本、被打回的尝试是负样本"这一表述——多跑几次会发现,越接近目标状态,继续找到"净改进"的随机 patch 越难,这正是论文里 CID 002 需要 82 次迭代才收敛的"长尾"现象的一个玩具级复现。

## 自测三层 🎓
- **L1 复述**：HORIZON 的核心流程是什么？project_pack 里的五个组成部分(π_agent、E_p、A_p、Γ_p、Ω_p)分别对应什么职责？
- **L2 解释**：为什么 HORIZON 选择让 acceptance predicate 在整个 campaign 期间保持固定,而不像 Self-Harness/AEVO 那样允许 agent 自我修改评估标准或进化机制？这个选择在 reward hacking 风险上带来了什么权衡？
- **L3 应用**：如果你是一家做 AI 编程助手的产品经理,想把"仓库级代码进化"这套思路搬到"自动修复公司内部微服务的 CI 失败用例"场景,你会怎么设计这个场景里的 Markdown harness、evaluator 和 acceptance predicate？哪些地方需要引入类似论文 Section 5 提到的"隐藏测试/诊断反馈分离"机制来防止 agent 过拟合 CI 日志？

📅 知识时间锚：论文发表于 2026-06(arXiv:2606.28279,cs.AR)；本笔记复核 2026-07-08
