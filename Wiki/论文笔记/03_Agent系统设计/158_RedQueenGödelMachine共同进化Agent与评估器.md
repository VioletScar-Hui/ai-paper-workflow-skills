---
tags: [论文笔记, 自进化, Agent进化, 共同进化, LLM-as-a-Judge, Gödel机器, 评估器设计]
paper_id: "158"
filename: "158 - The Red Queen Gödel Machine - Co-Evolving Agents and Their Evaluators.pdf"
authors: "Alex Iacob, Andrej Jovanović, William F. Shen (等同贡献), Daniel Burkhardt, Meghdad Kurmanji, Nurbek Tastan, Lorenzo Sani, Niccolò Alberto Elia Venanzi, Ambroise Odonnat, Zeyu Cao, Bill Marino, Xinchi Qiu, Nicholas D. Lane（University of Cambridge / NVIDIA / Flower Labs / MBZUAI / Inria）"
year: 2026
笔记层级: 骨干
成熟度: 🧪
复核日期: 2026-07-08
---

# 红皇后哥德尔机：共同进化的 Agent 与其评估器（The Red Queen Gödel Machine: Co-Evolving Agents and Their Evaluators）

📄 **原文 PDF**：[[RAW/158 - The Red Queen Gödel Machine - Co-Evolving Agents and Their Evaluators.pdf]]

## PM 速判（30秒）
> 一句话：现有自进化 Agent（DGM、HGM、HyperAgents）都假设"评分标准是固定不变的外部裁判"，这篇论文让**评分标准本身也参与进化**——评估器和被评估的 Agent 一起变强，同时用"按 epoch 冻结评分标准 + 只在 epoch 边界替换评估器"的机制保住原有的自改进理论保证。PM 必须关注：这是"评测体系本身会过时/被刷分"这个痛点的正面解法，直接决定你的 Agent 评估基建能不能撑过多轮迭代而不失真。

## 双层费曼 🗣️
> **给CEO的一句话**：以前让 AI 自己越改越强，靠的是一把固定不变的"尺子"打分；但尺子用久了会被摸透（reward hacking），有些任务（比如论文写作）本来就没有客观尺子。这篇论文让"尺子"也跟着被评分的 AI 一起变准、变严格，像物种和天敌互相适应一样，从而在没有基准或基准会过时的场景里也能持续变强。
>
> **给工程师的一段话**：RQGM 把搜索划分为若干 evolutionary epoch，每个 epoch 内评估器（learned evaluator）冻结、提供平稳的效用信号，保证 HGM 的单 epoch 自改进收敛保证成立；在 epoch 边界，挑战者评估器只有在评测无关的 ground-truth anchor 上的 𝜖-best-belief 分数超过在任评估器时才会被提升为新的冻结评估器，随后执行 selective erasure（只擦除依赖被替换评估器的效用记录，而非整个档案），从而让效用函数可以跨 epoch 演化而不破坏单 epoch 的搜索有效性证明。

## 问题域定位 🎯
- **回应什么根本约束**：DGM（Darwin Gödel Machine）、HGM（Huxley Gödel Machine）、HyperAgents 等自改进搜索方法都要求一个"在搜索循环之外保持不变"的评价标准（固定的 verifier / benchmark / 打标数据集）。这在三类场景里卡壳：(1) 目标任务本身没有直接基准（论文写作、证明写作没有基准，但论文评审、证明打分有）；(2) 评估很慢或信息量弱；(3) 静态基准饱和或在 Agent 变强后被 reward hacking。
- **之前卡在哪**：静态评价标准假设"环境不随物种进化而变化"，这违背了生物演化的核心特征——物种要跟着共同进化的天敌调整适应度（Van Valen 的红皇后假说，1973）。此前哪怕是 Multi-Agent Evolve、Group-Evolving Agents 这类"多智能体互相博弈"的方法，评价标准本身仍然是固定的。
- **开启的技术路线**：把"评估标准"也纳入搜索空间——用可学习的评估器（agent-as-a-judge）代替静态基准，评估器和任务 Agent 共享同一个可编辑工作区，二者的代码可以互相复用、共同进化。这打开了"benchmark 饱和/无基准领域/评审偏见校正"三类此前自改进方法无法触及的场景。
- **关闭的路线**：论文明确放弃了"评估标准完全静止直到人工重建基准"这条路径——作者指出基准重建成本高且任何变更都会强迫搜索重启，RQGM 用可控效用演化（controlled utility evolution）取代了这条路径。

## 核心机制

RQGM 建立在 HGM 的 clade metaproductivity（CMP）树搜索之上，做了四处改动：(i) 每个档案节点从单一 Agent 变成多智能体共享工作区；(ii) 评估器本身也是可学习的 Agentic 流程；(iii) 驱动搜索的效用函数可以在指定的"演化 epoch"边界改变；(iv) 评估器可以在这些边界被替换。

```
                          ┌───────────────── 档案树 T ──────────────────┐
                          │   节点 a = 共享多智能体工作区（K个角色）      │
                          │   角色 = {任务Agent(写手/程序员/证明者), ... │
                          │           评估器(评审/打分/代码审查)}        │
                          └───────────────────────────────────────────┘
                                        │  Thompson采样 over CMP
                                        ▼
        ┌───────────── Epoch j （评估标准冻结区间）─────────────┐
        │  评估器E被冻结 → 对该epoch内所有task agent提供平稳效用信号 │
        │  三层采样：node(CMP) → role(最少评测优先) → task(最少评测优先) │
        │  UCB-Air 扩张门：N_t^α ≥ |T_t| 时扩张节点，否则评测节点     │
        └───────────────────────────────────────────────────────┘
                                        │ 到达 checkpoint（指数间隔: τ,2τ,4τ,8τ...）
                                        ▼
        ┌────────────── 评估器替换检查点 ──────────────┐
        │ 挑战者评估器们 vs 在任评估器  →  都在"评测无关"的     │
        │ ground-truth anchor（如APReS人工决策/IMO人工评分）上   │
        │ 计算 𝜖-best-belief 分数 BB_ε = I_ε^{-1}(1+S_gt,1+F_gt) │
        │ 取分数最高者晋升，平局偏向在任评估器（避免无谓擦除）    │
        └───────────────────────────────────────────────┘
                                        │ 若发生替换 (e* ≠ 在任评估器)
                                        ▼
        ┌────────────── 选择性擦除 (Selective Erasure) ──────────────┐
        │ 只删除"依赖被替换评估器"的效用记录（dep(z)包含该slot的记录）│
        │ 保留：所有评测无关证据 + 谱系记录 + 其余角色的效用记录        │
        │ 重新计算：节点级/clade级成功失败计数、Thompson统计量         │
        └─────────────────────────────────────────────────────────┘
                                        │
                                        ▼
                    进入 Epoch j+1（新评估器冻结，继续搜索）
                     → 效用曲线在替换点下跌（旧记录被擦），
                       随后在更严格的新标准下重新爬升（Fig.1/5/6）
```

关键点：**epoch 内是静态搜索问题**（Prop. 3，Epoch-Local Validity），因此 HGM 原有的单 epoch 自改进收敛保证可以直接套用；**epoch 之间**通过"只用评测无关 anchor 衡量挑战者"来保证评估器是单调朝更优方向替换的（Anchor Lower Bound，替换后的评估器在 anchor 上的 𝜖-best-belief 分数不劣于被替换者）；**替换的记账成本**通过指数间隔 checkpoint 从 O(B²)（每次评测后都允许换评估器）降到 O(B)（Prop. 6, Bounded Recovery）。

## 设计决策解剖 ⚖️

| 决策点 | 他们的选择 | 被放弃的方案 | 为什么 | 这个选择何时失效 |
|---|---|---|---|---|
| 效用信号的时变粒度 | 按 epoch 冻结评估器，只在checkpoint边界允许整体替换 | 每次评测后都允许评估器变化（连续在线更新） | 连续更新会让效用信号非平稳，破坏 HGM 赖以收敛的"固定评价标准"假设；论文证明连续替换会把记账成本从 O(B) 拉高到 O(B²)（Prop.6/Rem.7），且会让 Thompson 采样的成功/失败计数无法收敛到正确目标（Prop.1） | 当单个 epoch 内的搜索预算过小（远低于文中 12,288 次评测/run 的量级）时，评估器可能永远等不到晋升机会，epoch机制退化成"从未真正更新过的固定评估器" |
| 评估器替换的判据 | 只依据评测无关的 ground-truth anchor（APReS 人工决策 / IMO-GradingBench 人工评分 / CRAVE 代码评审标签）上的 𝜖-best-belief 分数，与被评估的task agent完全解耦 | 用task agent自身的搜索效用（比如任务通过率）来判断评估器好坏 | 若评估器晋升标准和任务效用绑定，写手和评审可能"合谋"互相打高分（reward hacking），退化为自我偏好偏差（self-preference bias）；用独立anchor能保证替换后的评估器"确实更准"这件事有统计学保证（Prop.4/Rem.3） | 当 anchor 数据集本身就有偏差或噪声时（论文在Limitations里承认APReS决策和IMO-GradingBench评分本身不完美），"anchor验证"这个安全阀本身失效，评估器可能被带偏而无法察觉 |
| 效用擦除粒度 | 选择性擦除：只删除依赖被替换评估器的记录，保留其余角色/谱系的历史证据 | 全量擦除重排（每次替换后整个档案重新评测）或完全不擦除（保留旧评估器打的分） | 全量重排会浪费预算在搜索可能永不再访问的节点上；完全不擦除会让"新旧评价标准"的证据混在一起，违反epoch内平稳性要求。实验(Fig.3)显示no-erasure控制组的Spearman ρ始终≥0.90，证明不擦除时新评估器根本无法重新排序档案 | 当角色之间高度耦合、几乎所有效用记录都同时依赖多个评估器slot时，"选择性"擦除退化为等价于全量擦除，起不到省预算的作用 |
| 主实验模型选择 | 全程使用 GPT-5.5 (low)（"desirable cost-intelligence trade-off"），仅在消融实验里换用 Nemotron 3 Super/Ultra | 用更强模型（如GPT-5.5高算力档位）跑主实验以追求更高绝对分数 | 论文的目标是验证机制本身是否有效，而非刷绝对SOTA分数；用统一的中等成本模型能让HGM-H基线和RQGM在同一起跑线上比较 | 当模型能力不足以稳定生成/判别某个领域的产出时（论文自己提到：如果task agent基础模型能力不够，比如构造新颖数学证明，换用更便宜模型反而会增加搜索成本，因为不同meta-agent扩张之间难以区分质量优劣） |
| 论文评审的目标函数 | 主动放弃"评审的原始 APReS 准确率"这个直接优化目标，转而在evaluator replacement之后注入对抗性目标（拒绝显示偏袒AI生成文本的评审者） | 直接把"评审对ground-truth的准确率"作为唯一晋升标准 | APReS 数据集的接受/拒绝比例天然偏向宽松评审，导致评审对AI生成论文的偏袒（self-preference bias）和对ground truth的准确率"往同一个方向走"；HGM-H基线评审对AI论文的接受率是人类论文的1.42×-1.91×。RQGM用epoch边界注入对抗样本池纠正这一点，代价是牺牲了部分原始准确率 | 当"公平对待AI与人类作品"不是产品目标、只追求评审与人工标注的最大一致率时，这个校正反而会拉低评审的表面准确率（论文承认最终评审的原始准确率低于宽松的HGM-H评审） |

## 成本与量级 💰
- **搜索预算**：每次 run 固定 12,288 次评测（8,192+4,096，作者称是相对 Wang et al. [HGM] 单任务 8,000 次评测的"经济折中"）。
- **Coding 域（Polyglot）**：RQGM 相对 HGM-H 在达到相同/更高 held-out 通过率时节省 1.35×-1.72× 的 blended tokens（输出token按输入的5倍成本折算，含生成+评估调用）。RQGM held-out 通过率 71.7% vs HGM-H 的 69.9%。
- **Paper 域**：co-evolved writer 在匹配算力点上已经比 HGM-H 高 1.78×（generalist）；全程搜索结束后最高达 1.86×（specialist），对应接受率从 HGM-H 的 21.8% 提升到 38.8%/40.5%（四评审面板均值）。
- **Grader（证明评分）**：co-evolved grader 在 specialist 选取点上比 HGM-H grader 省 3× 搜索成本；即使到更贵的 generalist 选取点（多花1.35×），仍优于 HGM-H 在同等预算内能找到的最好 grader。
- **成本构成（Fig.9）**：三个主实验run里，validation评测占 blended tokens 的 65%-69%，扩张（expansion）占 18%-23%，训练时评测占 12%-14%——**评估本身才是主要成本驱动，而非节点扩张**。
- **混合模型降本实验**：meta-agent 仍用 GPT-5.5(low)，但把搜索期间 task agent 的调用换成更便宜的 Nemotron 3 Ultra（DeepInfra定价：GPT-5.5 $5/M输入 $30/M输出；Nemotron 3 Ultra $0.5/M输入 $2.2/M输出），在 paper 域上以约 13.0× 更低的（GPT-5.5价格等效）搜索成本逼近纯 GPT-5.5 方案的最终准确率。
- **最小可行配置**：论文未披露"最小可行"配置的具体阈值（比如低于多少次评测epoch机制会失效），只披露了实际使用的固定配置（𝜖=0.05，五次结果的anchor最小样本量，指数checkpoint比率 ρ=2）。

## 证据审计 🔬
- **实验设计是否公平**：搜索预算在 RQGM 和 HGM-H 之间是匹配的（"Search budgets are matched across runs, so cost differences reflect search dynamics rather than unequal allocation"），这点比较扎实。但主实验只用了一个模型（GPT-5.5 low）跑全部三个领域各一次，没有跨种子、跨模型重复实验，样本量偏小（例如 Polyglot 只有 166 个held-out任务，reviewer/grader 面板测试集各100条）。
- **基线选取的猫腻**：HGM-H 基线是作者自己"用HGM算法替换HyperAgents的DGM算法"重新实现的，而非HyperAgents原始论文的公开结果（因为Zhang et al.未公布最好的Polyglot agent）；这意味着coding域没有真正意义上的第三方SOTA基线可比，HGM-H某种程度上是"作者精心构造的对照组"而非独立可复现的既有系统。
- **最强证据**：Fig.3 的 no-erasure 消融——不做选择性擦除时，Spearman ρ 始终 ≥0.90（几乎不重排），而做擦除的三个领域 ρ 都掉到远低于1且再未恢复，这是"擦除机制确实是重排序生效的必要条件"的干净对照实验，成立条件是三个领域（代码评审/论文评审/证明评分）共同验证，说服力较强。
- **最可疑的数字**：论文写作域"co-evolved writer 接受率提升 1.78×-1.86×"，因为这个指标是用 RQGM 自己训练出来的评审器（含对抗性校正过的reviewer）作为四评审面板之一去打分，面板本身的构成（Sakana / DGM-H / HGM-H / RQGM adversarial）里只有一个是完全独立于本文方法的（Sakana），其余都与自改进搜索族有亲缘关系，可能低估了"外部真实人类评审"会给出的接受率差异。此外，"9%" ground-truth准确率提升这句摘要表述在正文Table中并未直接以"9%"形式出现，需要与IMO-GradingBench的具体准确率对照才能复现（论文摘要提到"co-evolved graders reach 9% higher ground-truth accuracy"，正文Fig.5显示RQGM grader准确率约76%附近，vs HGM-H约68%附近，量级吻合但摘要数字未在正文表格中逐字复现，值得警惕摘要和图表口径是否完全一致）。
- **审稿人会要求补充**：(1) 跨模型（不止GPT-5.5 low）的复现实验，验证机制收益是否是模型特定的；(2) 更长搜索时域下 RQGM 是否会持续优于 HGM-H 而非仅在两三次epoch内领先（论文自己承认"短搜索时域"是预实验性质）；(3) 人工评审对生成论文/证明的直接打分，用来验证agent-as-a-judge面板本身是否可信（论文承认"没有对生成的论文或证明做人工评分"）。
- **最小复现实验（代理指标+预算）**：在一个轻量代码评审任务（例如30-50个单元测试题）上，用两个廉价LLM角色（生成器+评审器）跑一个只含2个epoch、checkpoint间隔为16/32次评测的迷你RQGM，代理指标为"held-out通过率"和"replacement后Spearman rank相关性"，预算控制在几万tokens、几十次LLM调用即可验证"选择性擦除是否比不擦除排序变化更大"这一核心机制假设。

## 可复用点（PM决策）
- **何时采用**：当你的AI产品的评估体系存在"目标任务没有客观基准"（如内容创作类、策略建议类agent）或"现有评审/裁判Agent已经出现明显偏见/被刷分迹象"（比如LLM-judge偏向自己生成的内容、验收标准被针对性优化）时，值得考虑"评估器也纳入迭代"的架构，而不是无限期地依赖一份手写的评分 rubric。
- **何时规避**：如果你的任务有稳定、权威、几乎不会过时的客观基准（比如单元测试通过率这种硬指标），引入评估器共同进化的额外复杂度（多套epoch管理、选择性擦除记账、anchor数据维护）收益有限——论文自己在coding域的提升也主要来自"额外的代码评审信号"这种锦上添花，而非替代掉测试执行这个硬指标。
- **供应商拷问清单**：
  1. 你们的"评估器/裁判Agent"多久更新一次？更新时如何证明新裁判确实比旧裁判更准，而不是简单地"看起来更严格"？（对应RQGM的anchor best-belief替换判据）
  2. 裁判更新后，之前用旧裁判打分的历史数据是全部作废重新跑，还是有增量、可审计的处理方式？处理成本是线性还是二次方增长？（对应selective erasure + 指数checkpoint）
  3. 你们如何验证裁判Agent没有系统性偏袒AI生成内容（self-preference bias）？有没有单独的对抗性校正机制，还是只用原始准确率作为唯一指标？

## 关联网络 🕸️
- [[Wiki/概念/02_训练方法/递归自我改进]] —— RQGM 是"递归自我改进"这一原子概念在"评估标准也非平稳"场景下的扩展实例：传统RSI假设评价标准固定，RQGM主张评价标准也要参与自我改进循环，是对该概念边界条件的直接挑战。
- [[Wiki/论文笔记/03_Agent系统设计/139_AEVO元编辑Agent进化框架]] —— 二者都属于"进化机制自我修改"主题，但进化的对象不同：AEVO进化的是"搜索机制/元Agent的编辑策略"本身，RQGM进化的是"评价标准/评估器"本身；两者可以看作自改进搜索在不同维度上的正交扩展（一个动"怎么搜"，一个动"怎么评分"），值得对比二者是否可以叠加使用。
- [[Wiki/论文笔记/03_Agent系统设计/32_Harness更新vs获益-自进化Agent能力解耦]] —— 32号笔记讨论的是"harness更新"与"能力获益"是否解耦的问题；RQGM的核心张力恰恰相反方向：它主张评估harness（评分标准）**必须**与被评估Agent的能力共同演化才能避免评价失真，二者可以作为"harness是否该跟着Agent一起变"这一争议的正反案例对读。
- [[Wiki/概念/04_Agent框架/学习型编排器]] —— RQGM 中的 meta-agent 相当于一个可编辑元角色/task agent配置的编排器，且editable surface（工具调用次数、超时、meta-agent自身指令）本身也在搜索中被修改，可与"学习型编排器"概念中"编排逻辑本身可学习"的论述互相印证。
- [[Wiki/概念/04_Agent框架/Agent寿命工程]] —— RQGM 用"epoch"划分Agent/评估器的生命周期阶段（冻结期→checkpoint→替换→重新冻结），是"Agent寿命工程"里"阶段性冻结与迭代"这一设计模式在多角色协同场景下的具体化。
- **冲突/印证**：与 [[Wiki/论文笔记/03_Agent系统设计/32_Harness更新vs获益-自进化Agent能力解耦]] 构成一定的立场冲突——32号笔记的核心关切之一是"harness变了，能力提升到底是Agent变强还是harness在帮忙",而RQGM的做法是主动承认并拥抱"评分标准也在变",用ground-truth anchor + 𝜖-best-belief统计检验来控制"评分标准变化"本身不失控。这提示PM在评估"某自进化系统变强了多少"时，必须先确认其评价标准在测量期间是否发生了变化，RQGM提供了一种可审计该变化的具体方法论（anchor lower bound），可以作为审视其他自进化系统汇报数字时的检验框架。

## 动手练习 💻
练习目标：用一个30-80行的玩具Python脚本，模拟RQGM最核心的机制——"候选生成器 + 评估器 共同进化，评估器按epoch冻结、在checkpoint处用ground-truth anchor决定是否替换、替换后做选择性擦除"，观察"不擦除"和"擦除"两种策略下候选质量收敛曲线的差异（对应论文Fig.3的核心发现）。

```python
import random

random.seed(0)

# ---- 玩具世界设定 ----
# 候选者(candidate)的"真实质量"用0~1之间的数表示（人类永远看不到，只有ground truth anchor能采样到近似值）
# 评估器(evaluator)是一个带偏置(bias)和噪声(noise)的打分函数，偏置越接近0越准

def true_quality(candidate):
    return candidate["skill"]  # 候选者的真实技能水平

def make_evaluator(bias, noise):
    # 评估器给候选者打二元分（pass/fail），受bias和noise影响
    def score(candidate):
        p_pass = min(max(candidate["skill"] + bias + random.gauss(0, noise), 0), 1)
        return 1 if random.random() < p_pass else 0
    return score

def anchor_eval(evaluator, n=30):
    # ground-truth anchor：用真实质量已知的固定题目集测评估器本身准不准
    # anchor候选者的真实技能是均匀分布的已知值，用评估器打分后和真实标签比对
    correct = 0
    for _ in range(n):
        true_skill = random.random()
        fake_candidate = {"skill": true_skill}
        pred = evaluator(fake_candidate)
        true_label = 1 if true_skill > 0.5 else 0
        correct += (pred == true_label)
    return correct / n  # 越高说明评估器和ground truth越一致

# ---- 候选生成：模拟meta-agent每步小幅编辑候选者 ----
def mutate(candidate):
    return {"skill": min(max(candidate["skill"] + random.gauss(0, 0.05), 0), 1)}

def run_search(use_erasure, epochs=4, steps_per_epoch=25):
    archive = [{"skill": 0.3}]  # 初始候选（技能较弱）
    records = []  # 每条记录: (candidate_idx, score, evaluator_id)
    evaluator_id = 0
    # 初始评估器：偏置+0.25（偏宽松，容易高估候选质量），噪声0.1
    evaluators = {0: make_evaluator(bias=0.25, noise=0.1)}
    history = []  # 记录每步"当前存档最佳候选"的真实质量，用于画收敛曲线

    for epoch in range(epochs):
        current_eval = evaluators[evaluator_id]
        for _ in range(steps_per_epoch):
            parent = max(archive, key=lambda c: true_quality(c))  # 简化版CMP：直接选真实质量最高者做父节点
            child = mutate(parent)
            archive.append(child)
            s = current_eval(child)
            records.append({"cand": child, "score": s, "eval_id": evaluator_id})
            # 存档最佳候选：按"该记录评估器打分为1的候选中,真实质量最高者"选出
            passed = [r["cand"] for r in records if r["score"] == 1]
            best = max(passed, key=true_quality) if passed else archive[0]
            history.append(true_quality(best))

        # ---- checkpoint: 生成挑战者评估器，偏置更严格（更接近0） ----
        challenger_bias = max(0.25 - 0.08 * (epoch + 1), 0.0)
        challenger = make_evaluator(bias=challenger_bias, noise=0.1)
        incumbent_bb = anchor_eval(current_eval)
        challenger_bb = anchor_eval(challenger)

        if challenger_bb > incumbent_bb:  # 挑战者在anchor上更准 -> 晋升
            evaluator_id += 1
            evaluators[evaluator_id] = challenger
            if use_erasure:
                # 选择性擦除：只删除依赖"被替换评估器"的记录
                records = [r for r in records if r["eval_id"] != evaluator_id - 1]
            # 不擦除策略：什么都不删，旧评估器打的分继续算数

    return history

# ---- 对比实验 ----
hist_with_erasure = run_search(use_erasure=True)
hist_without_erasure = run_search(use_erasure=False)

print("末尾候选真实质量(有选择性擦除):", round(hist_with_erasure[-1], 3))
print("末尾候选真实质量(无擦除):      ", round(hist_without_erasure[-1], 3))
print("有擦除时质量提升轨迹样例(每10步):", [round(x, 2) for x in hist_with_erasure[::10]])
print("无擦除时质量提升轨迹样例(每10步):", [round(x, 2) for x in hist_without_erasure[::10]])
```

预期观察：`use_erasure=True` 时，每次评估器变严格（bias降低）之后，"存档最佳候选"会短暂被淘汰重新筛选（因为旧的宽松评估器打的高分记录被清掉），随后在更严格标准下重新收敛到更高真实质量；而 `use_erasure=False` 时，旧评估器打的宽松分数一直占着"最佳候选"的位置，新评估器的严格标准很难把它排挤下去——这正是论文 Fig.3 中"no-erasure 控制组 Spearman ρ 始终≥0.90（几乎不重排）"这一现象的最小化复现。

## 自测三层 🎓
- **L1 复述**：RQGM 用什么两个机制保证"评估器可以随时间变化"却依然能沿用 HGM 原有的自改进收敛保证？（答案要点：①按epoch冻结评估器，使每个epoch内部是固定评价标准的搜索问题；②评估器替换只依据评测无关的ground-truth anchor上的𝜖-best-belief分数，并配合选择性擦除清理失效记录。）
- **L2 解释**：为什么 RQGM 不能像论文写作那样，直接让论文写手和评审在同一个反馈循环里"自由对练"（类似AlphaGo自我博弈），而必须引入一个独立于两者之外的、评测无关的 ground-truth anchor（APReS人工决策）？如果去掉这个anchor，只让评审和写手互相打分会出现什么问题？（提示：联系self-preference bias、HGM-H评审对AI论文1.42×-1.91×过度接受的实测数据。）
- **L3 应用**：假设你在负责一个"AI面试模拟陪练"产品，里面既有"候选人回答生成Agent"，又有"面试官评分Agent"。产品上线半年后，你发现评分Agent似乎在系统性地给AI生成的示范回答打高分、给真实用户的口语化回答打低分。请用RQGM的思路，设计一个"评分Agent迭代校正方案"，说明：你的ground-truth anchor该是什么数据、多久做一次评估器替换检查、替换后哪些历史打分记录需要作废重算、哪些可以保留。

📅 知识时间锚：论文发表于 2026年6月（arXiv:2606.26294v2，2026-06-29），本笔记复核于 2026-07-08。
