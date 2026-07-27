---
tags: [论文笔记, Agent失败模式, 综述, 评测方法论, 测量效度, 牛津大学]
paper_id: "174"
filename: "174 - Beyond the Leaderboard - A Synthesis of Tool-Use, Planning, and Reasoning Failures in Large Language Model Agents.pdf"
authors: "Wael Albayaydh, Rui Zhao, Ivan Flechais (University of Oxford)"
year: 2026
笔记层级: 骨干
成熟度: 🧪
复核日期: 2026-07-15
---

# 超越排行榜：LLM Agent 工具使用、规划与推理失败模式综述（Beyond the Leaderboard）

📄 **原文 PDF**：[[RAW/174 - Beyond the Leaderboard - A Synthesis of Tool-Use, Planning, and Reasoning Failures in Large Language Model Agents.pdf]]

## PM 速判（30秒）

> 一句话：牛津大学团队综合 27 篇论文（覆盖 19 个 benchmark），首次把工具使用、规划、长程推理、多 Agent 协作、安全、测量效度六个此前互不对话的失败研究方向整合成一套跨领域分类法——核心结论是"排行榜分数持续上涨"和"六类结构性失败仍然存在"这两件事同时成立。PM 做 Agent 产品决策时必须按失败簇分别诊断，不能被单一头条准确率数字误导，也不能因为看到大量失败案例就否定真实存在的进步（单轮工具选择、短程网页导航、窄范围编码任务是货真价实地在变强）。

## 双层费曼 🗣️

> **给CEO的一句话**：AI Agent 的排行榜分数一直在涨，但这篇综述扒开 27 篇论文发现，真正让 Agent 在生产环境翻车的六类问题——记错工具用法、任务一长就崩溃、多个 AI 互相配合反而添乱、遇到模糊指令就乱猜、被恶意内容诱导做坏事、评测分数本身可能是刷出来的——一个都没被真正解决；但作者也很公道地指出，短流程、边界清晰的任务是真的在稳步变强，进步不是幻觉，只是被排行榜的整体涨幅掩盖了"哪里真的好、哪里还是老样子"的细节，PM 需要按任务类型分别判断能不能上线。
>
> **给工程师的一段话**：本文对 27 篇 benchmark/taxonomy/audit 论文（2023-2026，覆盖 19 个 benchmark）做自下而上迭代分类，归纳出六个失败簇：工具调用参数级错误、多约束规划失败、长程上下文退化、多 Agent 协调失败、安全与安全性失败、测量效度问题。三条跨簇结构性发现：失败随任务长度非线性复合（"no-recovery bottleneck"）、子技能能力不能可靠组合成端到端成功（compositionality gap）、额外 scaffolding（更多 Agent/工具/推理量/上下文）收益不均衡甚至为负。同时明确区分真实能力进步与测量误差修正（如 SWE-bench 去污染后 12.5%→4%）。

## 问题域定位 🎯

- **回应什么根本约束？** 自 Toolformer/ReAct/Reflexion（均 2023）确立"工具调用+推理行动循环+自我反思"的现代 Agent 范式以来，评测领域形成了两条互不对话的叙事：排行榜叙事（WebArena/SWE-bench/METR 等分数持续上涨，暗示核心工程问题接近解决）与审计叙事（各 benchmark 自己的错误分析论文持续记录幻觉工具名、无法联合满足约束、协作开销抵消收益等具体失败）。本文回应的约束是：这两条叙事此前从未被放进同一个框架对照阅读，读者只看其中一条就会得出片面结论。
- **之前卡在哪？** 工具使用（ToolScan 等）、规划（TravelPlanner 等）、长程（METR 等）、多 Agent（MAST 等）、安全（ToolEmu/InjecAgent）、测量效度（SWE-bench 污染研究等）六个方向的失败研究彼此独立发展，各自定义各自的错误分类法（27 篇论文里散落着 60 多个具名错误类别），没有人系统比较过这些分类法之间的关系。经典多 Agent 系统理论（Wooldridge, 2009）早就指出协调开销和通信协议设计是老问题，但 LLM Agent 文献很少与这条理论脉络对话——本文在 Section 2 明确指出，LLM 多 Agent 协作失败不是全新问题，新的只是自然语言角色设定、隐式而非形式化验证的通信协议这些具体机制。
- **开启/关闭了哪条技术路线？** 开启：跨簇失败诊断框架（六簇覆盖 Agent 推理-行动管线的不同阶段，且以独立标注体系的交叉收敛作为效度证据）+ 显式区分"真实能力进步"与"测量误差修正"的方法论（Section 5 vs 4.6）+ 面向研究者/实践者/政策制定者的三层建议（Section 8）。关闭：关闭了"只看单一 benchmark 头条准确率数字判断 Agent 是否成熟"的简化评估习惯，也关闭了"看到大量失败案例就认为进步是虚假的"这种一刀切悲观论——本文明确反对这两种极端简化。

## 核心机制

```
Agent 推理-行动管线 × 六大失败簇（论文自身的呈现顺序，Section 4.1→4.6，Figure 1 文字重构）
═══════════════════════════════════════════════════════════════════════

【最细粒度】单次工具调用的机制层面
┌───────────────────────────┐
│ 4.1 工具调用与参数级失败       │  ToolScan 7类错误(幻觉函数名/参数值/
│ "at the most granular level" │  类型/格式…)；结构化function-calling
└─────────────┬─────────────┘  模式反而更易出错(77.5 vs 21次错误
              │                 调用/组，对比prompting式调用)
              ▼ 从单次调用升级到多步计划
┌───────────────────────────┐
│ 4.2 规划与约束满足失败         │  TravelPlanner: GPT-4联合约束成功率
│ "moving from single tool     │  仅0.6%(o1-preview~10%；验证器增强
│  calls to multi-step plans"  │  ~65%，仍有相当比例无法收敛)
└─────────────┬─────────────┘
              ▼ 从多步计划延伸到长轨迹
┌───────────────────────────┐
│ 4.3 长程退化与上下文管理失败    │  METR: <4分钟人类任务~100%成功，
│ "emerges specifically as     │  >4小时任务<10%成功，时长每7个月
│  tasks lengthen"            │  翻倍；"no-recovery bottleneck"：
└─────────────┬─────────────┘  早期错误不可回滚
              ▼ 从单Agent长轨迹扩展到多Agent系统
┌───────────────────────────┐
│ 4.4 多Agent协调失败           │  MAST: 14种失败模式/7框架/200+任务
│ "distribute a task across    │  (specification~42%+misalignment
│  cooperating LLM agents"     │  ~37%≈79%，verification仅~21%)
└─────────────┬─────────────┘
              ▼ 与能力失败正交的独立维度
┌───────────────────────────┐
│ 4.5 安全与安全性失败           │  ToolEmu: 最安全Agent仍23.9%严重
│ "distinct from pure          │  后果；InjecAgent: 24%→47%(强化
│  capability failures"        │  攻击prompt后)，开源无安全微调>80%
└─────────────┬─────────────┘
              ▼ 覆盖以上全部阶段的元层问题
┌───────────────────────────┐
│ 4.6 测量效度问题               │  分数本身是否可信，而非能力是否存在
│ "whether benchmark scores    │  SWE-bench去污染:~12.5%→~4%(一项
│  reliably measure..."        │  配置)；HAL: 21,000+ rollouts/9模型
└───────────────────────────┘  ×9benchmark/~$4万；更高推理努力多数
                                run里降低了准确率

分类法推导过程（Section 3.3，自下而上迭代分组，非预设框架）：
  27篇论文原始错误类别(初始清单>60个具名类别，如ToolScan 7类/MAST 14类)
        │ 迭代合并同质类别(如ToolScan"幻觉函数名"+BFCL"错误调用计数"→同一主题)
        ▼
  6个概念不重叠、分别对应管线不同阶段的簇
        │ 交叉验证：MAST三大类独立收敛到4.4/4.6边界；
        │ ToolScan/REST-API的4/7类taxonomy独立收敛到4.1边界；
        │ 长程退化文献独立收敛出"上下文积累"是有别于规划(4.2)和
        │ 工具使用(4.1)的独立瓶颈
        ▼
  独立标注体系收敛到相近簇边界 → 作为分类法效度证据(而非单一团队自证)
```

## 设计决策解剖 ⚖️

| 决策点 | 他们的选择 | 被放弃的方案 | 为什么 | 这个选择何时失效 |
|--------|-----------|-------------|--------|-----------------|
| 综述方法论 | 显式声明为"叙事综合"（narrative synthesis），非预注册协议的系统综述 | 正式 PRISMA 系统综述流程 | 原文 Section 3 开篇明确写"This is a narrative synthesis, not a formal systematic review...we make this explicit rather than overstating its rigor"，用显式 scope+准入准出标准+显式分类法推导过程部分补偿严谨性缺口 | 当读者/审稿人需要审计"初筛检索到多少篇候选、复筛掉了多少篇"时——本文只给出最终入选的 27 篇，未报告检索漏斗，无法计算真正的选择性偏差（见证据审计） |
| 分类法生成方式 | 自下而上：从 27 篇论文原始错误类别（>60 个具名类别）迭代合并成 6 个簇 | 套用某个已有理论框架（如经典 MAS 理论或某单篇论文的 taxonomy）直接作为顶层结构 | 作者强调这样能获得"独立标注体系收敛"的交叉验证证据——MAST 的 3 大类独立对应到本文 4.4/4.6，ToolScan 和 REST-API taxonomy 的 4/7 类独立对应到 4.1，这种收敛被认为"是比任何单一团队内部一致性更好的效度证据"（Section 3.3 原话） | 当某个错误案例本身横跨多簇边界时——作者自己承认"a different research team...might draw the six cluster boundaries somewhat differently"（Section 9），比如"参数类型错误"到底算 4.1 工具调用失败还是 4.2 规划失败导致的连带传参错误，迭代分组法给不出唯一答案 |
| 是否辟出"真进步"章节 | 显式设 Section 5，专门论证哪些能力"确凿地"变强了（单轮工具选择、短程网页导航、窄范围编码任务、安全微调） | 只做失败分类学综述（大多数同类论文的默认做法，包括本库 136 号 LIFE 综述基本聚焦失败侧） | 作者担心"综述若只组织失败模式，会给人一种能力停滞或虚假的错误印象"（Section 5 开篇），因此主动用 WebArena 14%→61.7%、WebVoyager 从"well under 50%"到约 85-90% 等数字去平衡叙事 | 当"进步"证据本身依赖本文自己质疑的同一批排行榜数字时——例如 WebArena 61.7% 来自 Marreed et al. 2025 一篇非同行评审的企业 pre-print，本文虽标注了其身份，仍把它当作"genuine architectural convergence"的核心支撑，这与本文在 4.6 节批评"审慎对待行业数字"的立场存在自我豁免张力 |
| 引入作者自己的旁观者隐私研究 | 在 4.5 节末尾专门引用作者此前关于约旦智能家居旁观者隐私的三篇论文（CHI'22/USENIX Security 23/24），并明确声明"这些研究没有评测 LLM agent，我们不把它当作直接证据" | 完全不提这条不直接相关的人本研究线，只讨论 ToolEmu/InjecAgent 本身的量化结果 | 指出 agent 安全评测的一个具体空白：ToolEmu 的高风险工具箱包含智能家居/IoT，而现有 agent 安全评测（含 ToolEmu 本身）都是围绕"对指令下达者的伤害"设计的，没有覆盖非自愿的旁观者伤害 | 当读者把这段误读为"ToolEmu 已经实证了智能家居 Agent 会伤害旁观者"时——原文明确排除了这种推论，这段纯粹是概念性 gap-spotting，没有任何 agent-bystander 量化数据支撑，脱离限定条件引用就会造成过度外推 |

## 成本与量级 💰

- **覆盖论文数**：27 篇 benchmark/taxonomy/audit 论文，时间跨度 2023-2026（来源：Abstract、Table 1）
- **覆盖 benchmark/框架数**：19 个具名 benchmark 或评测框架（来源：Abstract、Section 3.1）
- **Table 1 细分**（来源：Table 1；各类别计数总和 >27，因为论文可能跨类别）：提出/验证失败分类法 6 篇；聚焦规划/约束满足 5 篇；聚焦工具使用/函数调用错误 6 篇；聚焦长程/上下文退化 6 篇；聚焦多 Agent 协调 2 篇；聚焦安全/安全性 2 篇；聚焦测量效度/污染 4 篇；非同行评审的 industry 或二手来源 2 篇
- **子研究规模**（均为本文转引，来源标注到具体章节/原始论文）：
  - HAL（测量效度簇实证基础）：21,000+ agent rollouts，9 模型 × 9 benchmark，约 4 万美元评测成本（Section 4.6/6，Kapoor et al. 2025）
  - MAST（多 Agent 协调簇实证基础）：7 个框架、200+ 任务、Cohen's Kappa 0.88 人工标注、14 种失败模式（Section 4.4，Cemri et al. 2025）
  - ToolEmu（安全簇实证基础之一）：36 个高风险工具箱、144 个测试用例、68.8% 失败人工确认率（Section 4.5，Ruan et al. 2023）
  - InjecAgent（安全簇实证基础之二）：1,054 个测试用例（Section 4.5，Zhan et al. 2024）
  - API-Bank → ToolBench/StableToolBench：从 73 个工具扩展到"数千个真实世界 API"（Section 4.1，原文未给精确数字，仅表述为"thousands of real-world APIs"）
- **论文未披露的部分**（明确标注，不编造）：初始检索到的候选论文总数与筛选漏斗（多少篇被排除、排除理由分布）——本文只给出最终入选的 27 篇，没有 PRISMA 式流程图；27 篇中同行评审与预印本的精确篇数比例——正文仅笼统提及"the majority of arXiv-only citations...are pre-print"，未逐篇列出评审状态清单；六个失败簇之间的案例重叠率——论文承认簇边界主观（Section 3.3/9），但未量化重叠程度
- **最小可行配置**：本文是纯文献综合，不涉及模型训练/推理配置。若把"用本文方法论对一个新领域做同类综合"视为最小可行配置，参考 Section 3.2 的准入门槛——一篇论文只要满足"benchmark+量化错误分析""人工标注失败分类法+标注一致性统计""专门审计某 benchmark 测量效度"三条准入标准中任一条即可纳入，理论上一人一周内可完成 10-15 篇规模的同类综合初稿

## 证据审计 🔬

- **实验设计公平吗？基线选取有什么猫腻？** 本文不做自己的实验，是转引 27 篇论文的数字，Section 6 明确声明"Figures are drawn from the specific studies cited and are not independently re-verified by the present authors"——诚实的免责声明，但也意味着如果被转引论文本身有问题（如某数字后来被撤回/修正），本文无法察觉。文献检索没有报告初筛/复筛漏斗，只说"27 papers met our inclusion criteria"，未说明初始检索到多少篇、经过几轮筛选，这是叙事综述相对系统综述的常见弱点。
- **最强证据**（带成立条件）：
  1. 独立标注体系的收敛：MAST（Cohen's Kappa 0.88，3 大类）、ToolScan（7 类）、REST-API taxonomy（4 类）分别独立开发，但边界与本文六簇高度重合，这是比单一团队自洽性更强的效度证据（Section 3.3），成立条件是这些原始论文各自的标注方法本身可信
  2. TravelPlanner 的三点证据链：0.6%（GPT-4）→约 10%（o1-preview）→约 65%（验证器增强，arXiv:2404.11891），三个数字来自不同的独立后续论文、同一 benchmark，构成清晰的技术演进链，但即便最好配置仍有相当比例任务失败
  3. METR 时间跨度分析：任务时长每约 7 个月翻倍（自 2019 年起），是全文唯一有严格"计时人类专家"基线对照的量化指标（Kwa et al. 2025），比模型对模型的自比更有说服力
- **最可疑的数字**：
  - WebVoyager 过滤后成功率：Table 2 给出"57-90%"（Skyvern filtered benchmark, 2025），但 Section 5.2 正文叙述"approximately 85-90%"——同一 benchmark，本文自己的表格和自己的叙事文本对不上；正文只挑了区间高端来论证"genuine improvement"，57% 这个低端从未在进步叙事里被提及，因为它会削弱"进步确凿"的论证
  - GAIA/HAL "64.8-93%"：接近 30 个百分点的区间被压进 Table 2 一个单元格，没有注明哪端对应哪种 scaffold/成本梯度，容易被下游引用者截断成"93%"单独使用
  - "higher reasoning effort reduced accuracy in the majority of evaluated runs"：这句话在 Section 4.2 和 4.6/6 被引用两次，但"majority of runs"没有给出具体 run 数或百分比，也未说明是否控制了任务难度分布，是全文论证力度最强却量化最弱的一句话
- **审稿人会要求补充**：初始检索的 PRISMA 式流程图（检索了多少篇→排除多少篇→为什么排除）；六个簇之间错误案例的重叠计数（比如"长程上下文丢失导致工具参数传错"该算 4.1 还是 4.3，原始论文可能各自归类）；27 篇论文逐一的同行评审状态清单，而非"the majority"这种模糊表述
- **最小复现实验**：代理指标——不需要重新跑模型，找 3-5 个团队内部真实 Agent 失败 case，人工按本文六簇打标签，统计能覆盖的比例（若频繁出现"无法归入任何一簇"或"同时符合两簇"，说明分类法边界在自己的场景里不够贴合）；预算：1 名工程师 2-3 小时人工标注，零计算成本。更完整版本：复现 TravelPlanner 的 0.6% 数字（该 benchmark 公开可跑），用同代模型验证是否仍在同一量级；预算：数十次 API 调用，约 10-30 美元

## 可复用点（PM决策）

**何时采用**：作为 Agent 产品上线前的"失败模式体检清单"——按六簇逐一自查：工具 schema 是否清晰、错误信息是否具体（4.1）；多约束任务是否有 verifier 或 critic 环节，而不是裸模型硬扛（4.2）；长会话是否有上下文折叠/摘要机制，能否在早期错误后回滚（4.3）；多 Agent 架构是否做过"vs 单 Agent 基线"的受控对比（4.4，可直接调用 [[Wiki/论文笔记/03_Agent系统设计/40_多Agent是否有助于性能提升受控评测]] 的 BenchAgent 方法论）；面向外部不可信内容的场景是否做过安全微调而非只靠 prompt 约束（4.5）；宣传的 benchmark 分数是否披露了污染/测试集校正情况（4.6）。

**何时规避**：不要把本文的六簇当作严格互斥的工程分类标准写进系统需求文档——作者自己承认簇边界主观，实际失败案例常跨簇（如"轨迹里的重复调用"，究竟算 4.1 工具误用还是 4.3 长程退化，取决于具体诱因，本文自己也没有给出可操作的判别规则）。

**供应商拷问清单**：
1. "你们 Agent 在多轮/多约束场景下的成功率，和营销材料里给的单轮/单约束成功率，是同一个数字吗？"（直击 4.1/4.2 "单轮强、多轮弱"的落差）
2. "你们的 benchmark 分数是否经过污染/测试集泄漏审计？能否提供类似 SWE-bench Verified 那样'去污染后从约 12.5% 掉到约 4%'的修正记录？"（直击 4.6）
3. "如果我的场景需要处理超过人类 4 小时完成时间的任务，你们有 METR 式的人类基线对照数据，还是只有模型自己跟自己比的成功率？"（直击 4.3 + METR 方法论）

## 关联网络 🕸️

**相关论文**：
- [[Wiki/论文笔记/03_Agent系统设计/136_多Agent系统协作失败归因自进化综述]] — 136 号 LIFE 框架专注"多 Agent 系统"单一维度，把失败归因（F 阶段）拆成数据驱动/约束引导/因果推断三类方法；174 号覆盖范围更广（六簇涵盖工具使用/规划/长程/多 Agent/安全/测量），但对"多 Agent 协调失败"这一簇的处理明显单薄于 136——174 在 4.4 节只引用了 MAST 一篇论文，136 是一篇 420+ 文献的专述。两者互补：174 提供跨领域全景地图，136 提供多 Agent 这一格子里的深度钻取
- [[Wiki/论文笔记/03_Agent系统设计/45_LIFE-HARNESS适配接口而非模型]] — 45 号核心发现"确定性环境中 90% 的失败是接口问题、只有 9.9% 是真推理失败"，与 174 号 4.1/7.3 节"工具调用错误集中在参数规范和函数命名，而非高层任务理解"的论点高度印证，但证据来源完全独立（174 引用 ToolScan/BFCL/REST-API taxonomy 等外部 benchmark，45 是独立单篇实验室研究：18 模型 × 7 环境），两条独立证据链收敛到同一结论
- [[Wiki/论文笔记/03_Agent系统设计/34_Agent-Harness有效反馈计算扩展律]] — 174 号 8.1 节建议研究者"报告 accuracy-cost frontier 而非只报告准确率提升"，与 34 号 EFC（有效反馈计算）框架"不能只看原始计算量，要看反馈质量"是同一方法论诉求在不同层面的呼应——34 号是 Agent 训练侧的效率指标，174 号是 Agent 评测侧的效率呼吁
- [[Wiki/论文笔记/03_Agent系统设计/40_多Agent是否有助于性能提升受控评测]] — 40 号受控实验（5/6 个 MAS 落后单 Agent 基线 2.56-11.29 点）为 174 号 4.4 节"多 Agent 协作收益经常 minimal"提供独立的、更极端的量化印证——174 引用的 MAST 统计的是失败模式分布，40 号直接给出"协作 vs 不协作"的性能差值，两篇论文分别回答"为什么失败"和"失败到什么程度"

**相关概念**：
- [[Wiki/概念/04_Agent框架/Harness设计模式]] — 该概念笔记已有"Agent 失败归因"小节（来自 45 号论文，确定性环境四层归因：动作实现 33.6%/轨迹退化 33.3%/合约误解 23.2%/推理失败 9.9%）。与 174 号是不同颗粒度、不同视角的互补关系：45 号/Harness 笔记的归因体系针对单一 Agent 在确定性规则环境（ALFWorld/OSWorld/BirdSQL 等）中的执行层失败，四类互斥且加总到 100%；174 号的六簇分类法针对跨基准、跨论文的失败研究本身，六簇不互斥、不加总到 100%，且额外覆盖了 45 号完全没涉及的三个维度——多 Agent 协作（4.4）、安全性（4.5）、测量效度本身是否可信（4.6）。可以把 45 号的四类归因理解为 174 号 4.1+4.3 在单篇论文尺度上的精细化重现，而 4.2/4.4/4.5/4.6 是 Harness 笔记目前完全未覆盖的失败面向
- [[Wiki/概念/03_推理与评测/Benchmark设计与污染]] — 174 号 4.6 簇的 SWE-bench 污染案例（约 12.5%→约 4%）、UTBoost 测试集补强改变 25-41% 排名的发现，是该概念笔记"数据污染三种形式"（完全重叠/风格模板污染/标签泄露）在 Agent 评测场景的具体实例，UTBoost 可以补充为该笔记"动态 Benchmark 解决方案"之外的第四种应对思路：不换题目，而是补强测试用例本身的严格性

（另有同批次论文《The Harness Effect: How Orchestration Design Sets the Token Economics of Enterprise Agentic AI》，RAW 目录内编号 172，主题涉及编排设计对企业级 Agent token 经济学的影响，与本文 4.1/4.4/7.4 节"scaffolding 收益不均衡"高度相关，但尚未被写入 Wiki，暂不加链接，仅作提示。）

**冲突/印证**：
- **印证**：174 号 4.4 节引用 MAST（Cemri et al. 2025）"specification issues + inter-agent misalignment 合计约占失败的 79%（42%+37%），verification/termination 仅占约 21%"，与 [[Wiki/论文笔记/03_Agent系统设计/136_多Agent系统协作失败归因自进化综述]] "F（失败归因）是当前研究最薄弱环节"的判断相互印证但视角不同——136 号说的是"学术界研究 F 阶段的论文数量少"（文献计量层面的空白），174 号引用的 MAST 则是"F 阶段内部，specification/misalignment 比 verification 更常见"（失败原因内部占比层面的实证）。
- **潜在冲突（分类维度差异）**：136 号把"失败归因"本身作为 LIFE 四阶段之一（L/I/F/E），归因是独立于"自进化"（E）的诊断环节；174 号则把 MAST 的归因方法论直接归入 4.4（多 Agent 协调失败簇的证据来源之一），而不是像 136 号那样把归因方法论单独抽出来作为跨领域的方法论层。同一份 MAST 数据，136 号读出"归因方法论"，174 号读出"失败簇里的一个数据点"——这提示两篇论文对"失败"的分类维度并不完全一致：136 号按"生命周期阶段"分类，174 号按"失败发生的系统层次"分类，两套坐标系正交而非重合。

## 动手练习 💻

**练习目标**：按 174 号论文的六大失败簇（4.1-4.6），对一条模拟 Agent 轨迹自动打标签，并复现论文 7.1 节"失败随任务长度非线性复合"的核心论点。

```python
"""
按 174 号论文的六大失败簇（Section 4.1-4.6），对一条模拟 Agent 轨迹自动打标签，
并复现论文 7.1 节"失败随任务长度非线性复合"的核心论点（no-recovery bottleneck）。
4.6(测量效度)不是单步性质的失败，是"整条轨迹的评分方法是否可信"的元问题，
不适合按步打标，因此本练习只对 4.1-4.5 五个"发生在轨迹内部"的簇建模。
"""
import random
from dataclasses import dataclass

CLUSTERS = ["4.1_工具调用参数级", "4.2_规划约束满足", "4.3_长程退化",
            "4.4_多Agent协作", "4.5_安全安全性"]

@dataclass
class Step:
    idx: int
    tool_call: bool = False
    tool_args_valid: bool = True
    revisits_prior_step: bool = False   # 重复历史动作 = 轨迹退化信号(4.3)
    violates_constraint: bool = False   # 违反任务约束 = 规划失败信号(4.2)
    involves_other_agent: bool = False  # 涉及跨Agent消息 = 协作失败信号(4.4)
    unsafe_without_confirmation: bool = False  # 未澄清歧义就执行敏感操作(4.5)

def classify_step(step: Step, traj_len: int):
    """打标规则直接对应论文对每个簇的原始定义，而非泛化的成功/失败二分类。"""
    if step.tool_call and not step.tool_args_valid:
        return "4.1_工具调用参数级"          # ToolScan: 参数值/名称/类型错误
    if step.violates_constraint:
        return "4.2_规划约束满足"            # TravelPlanner: 单约束满足≠联合约束满足
    if step.revisits_prior_step and traj_len >= 15:
        # 论文4.3强调长程退化是"任务变长后才出现"的失败；
        # 同样的重复动作发生在短轨迹里更可能只是普通retry，不计入此簇
        return "4.3_长程退化"
    if step.involves_other_agent and step.revisits_prior_step:
        return "4.4_多Agent协作"             # MAST: 信息流断裂导致重复沟通
    if step.unsafe_without_confirmation:
        return "4.5_安全安全性"              # ToolEmu: 未澄清歧义即行动
    return None

def simulate(task_length: int, seed: int):
    """轨迹越长，重复历史动作的概率越呈超线性上升——模拟论文的"no-recovery bottleneck"。"""
    random.seed(seed)
    steps = []
    for i in range(task_length):
        revisit_p = 0.02 * (i / 10) ** 1.5 if task_length >= 15 else 0.02
        steps.append(Step(
            idx=i,
            tool_call=random.random() < 0.6,
            tool_args_valid=random.random() > 0.08,      # ~8%参数错误率，量级对应BFCL多轮掉分
            revisits_prior_step=random.random() < min(revisit_p, 0.6),
            violates_constraint=random.random() < 0.05,
            involves_other_agent=random.random() < 0.15,
            unsafe_without_confirmation=random.random() < 0.03,
        ))
    return steps

def audit(steps, task_length: int) -> dict:
    """跑一遍分类，返回簇分布计数——对应论文 Table 2 风格的失败簇汇总。"""
    counts = {k: 0 for k in CLUSTERS}
    for step in steps:
        label = classify_step(step, task_length)
        if label:
            counts[label] += 1
    success_rate = 1 - sum(counts.values()) / len(steps)
    return {"success_rate": success_rate, **counts}

# ===== 复现论文 7.1："失败随任务长度非线性复合" =====
print(f"{'长度':>6}{'步级成功率':>10}  " + "  ".join(CLUSTERS))
for length in [5, 10, 20, 40, 80]:
    report = audit(simulate(length, seed=length), length)
    row = "  ".join(f"{report[k]:>6}" for k in CLUSTERS)
    print(f"{length:>6}{report['success_rate']*100:>9.1f}%  {row}")

print("\n预期现象：'4.3_长程退化'列应随长度增长呈超线性上升，")
print("其余各列大致与长度线性增长——这是论文 7.1 节'非线性复合'论点的最小复现。")
print("(4.6测量效度不出现在此表中——它衡量的是这张表本身是否可信，而非表里的某一格)")
```

## 自测三层 🎓

**L1 复述**：
1. 本文六个失败簇分别是什么？各自最主要依赖哪一篇/哪几篇原始 benchmark 论文作为证据？
2. 本文综合了多少篇论文、覆盖多少个 benchmark？六簇分类法是自上而下预设的还是自下而上迭代推导出来的？

**L2 解释**（为什么不用别的方案）：
1. 为什么作者选择"叙事综合"而不是正式 PRISMA 系统综述？这个方法论选择在 Limitations 里被如何自我批评？
2. 为什么本文要专门辟出 Section 5 论证"真进步"，而不是像大多数失败模式论文一样只列失败？如果没有这一节，本文的说服力会受到什么影响？
3. 为什么"测量效度"（4.6）要被单独列为第六个簇，而不是并入其他五个失败簇中的某一个？

**L3 应用**（具体产品场景迁移）：
1. 你是一个企业客服 Agent 产品的 PM，上线后发现 Agent 处理"退款+换货+积分补偿"三个条件同时满足的复杂工单时经常出错，但单独处理任何一个条件都没问题。用本文的分类框架判断这属于哪个失败簇，并参考 4.2 节 TravelPlanner 的应对路径（更强推理模型 + 外部形式化验证器/critic）设计一个具体的技术改进方案。
2. 你的团队想把现有单 Agent 客服系统改造成"接待 Agent + 专家 Agent + 审核 Agent"三 Agent 协作架构，立项评审时，你会用本文哪一节的哪些具体数字去说服团队先做一次受控 A/B 对比（可结合 [[Wiki/论文笔记/03_Agent系统设计/40_多Agent是否有助于性能提升受控评测]] 的 BenchAgent 方法论）？

📅 知识时间锚：论文综合 2023-2026 文献（含 2026 年 1-3 月发布的多篇 arXiv 预印本引用）· 本笔记复核 2026-07-15
