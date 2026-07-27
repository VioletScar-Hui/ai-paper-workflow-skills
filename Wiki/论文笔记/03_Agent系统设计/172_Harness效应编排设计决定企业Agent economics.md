---
tags: [论文笔记, Agent框架, Harness, Token经济学, 编排设计, 成本优化, 生产系统, Writer]
paper_id: "172"
filename: "172 - The Harness Effect - How Orchestration Design Sets the Token Economics of Enterprise Agentic AI.pdf"
authors: "Muayad Sayed Ali, Aliaksandra Novik, 等共33位作者（Writer, Inc.）"
year: 2026
笔记层级: 骨干
成熟度: 🧪
复核日期: 2026-07-16
---

# The Harness Effect：编排设计如何决定企业级 Agentic AI 的 Token 经济学

📄 **原文 PDF**：[[RAW/172 - The Harness Effect - How Orchestration Design Sets the Token Economics of Enterprise Agentic AI.pdf]]

## PM 速判（30秒）
> Writer公司做了一次"只换编排层、不换模型"的对照实验：同一套22个任务、同六个模型（Claude Sonnet 4.6/Gemini 3.1/Gemini Flash 3.5/Qwen 3.6/GLM 5.1/Palmyra X6），只把外面那套"调度系统"（harness）从自家旧生产循环换成新Harness，成本降41%（$0.21→$0.12/任务）、延迟降44%、token降38%，质量基本持平（0.78→0.81）——编排层对成本的影响比换模型本身还大（换最贵到最便宜的模型只省36%，换Harness省33%-61%）。但收益不均等：效率增益对所有模型无条件生效，质量增益却只有强模型能吃到（"Harness杠杆"，r=0.99），弱模型在子Agent委派、复杂工具调用这类高阶编排特性上反而会被拖累到质量倒退。PM必须关注，因为这把"该不该自建编排层、该拷问供应商什么问题"变成了可以用美元和token量化的采购决策，而不再只是"选哪个模型更强"的单维选型。

## 双层费曼 🗣️
> **给CEO的一句话**：AI Agent 干一件事不是问一句答一句，而是像员工在电脑前反复翻资料、调工具、试错，每一步都在花钱（花的是token，可以理解为AI公司按处理的文字量计费）。多数团队想省钱只会想到换便宜模型，但这篇论文发现：真正决定这笔钱花多花少的，其实是模型外面那套调度系统——怎么组织每一步该看什么、记什么、扔什么。换掉这套调度系统、模型完全不变，同样的活儿能省四成钱、快一倍，效果不打折——前提是你用的模型本身要够强，才能把这套更精细的调度系统用好；能力不够的模型反而会被复杂调度"压垮"，在最考验协作的任务上表现更差。
>
> **给工程师的一段话**：核心是把单任务token账单拆解为 T_in=S(系统提示)+H(历史)+G(工具schema)+R(检索)+U(用户输入)，朴素实现中H因全量重放呈O(k²)增长。Harness用"两区prompt"（字节稳定前缀+易变尾部，最多4个provider缓存断点，实测99.9%命中即≈0.1×原价）+ 80%预算触发的增量式结构化压缩（保留4-12条verbatim尾部）+ 子Agent上下文防火墙（8KB摘要上限）把增长压到O(k)，并用类型化失败分类+熔断替代无脑重试限制乘数项。六模型×22任务配对置换实验中，效率增益（成本/token -38%~-41%）在所有模型上一致出现，但质量增益（headline+0.03持平）与模型基线能力强相关（r=0.99, n=6），且子Agent委派这一净新增能力只在两个最强模型上越过0.85+的可用可靠性门槛。

## 问题域定位 🎯
- **回应什么根本约束**：行业默认应对"Agent能力需求上升"的方式是砸更多token——更长推理链、更多轮次、更宽工具payload、更大的重放上下文。per-token价格持续下降掩盖了这个趋势，但总支出仍在上升，论文称之为"token maxing"，是典型的Jevons悖论（效率提升反而刺激总消费增长，引用Jevons 1865《煤炭问题》）。已发表的效率工作要么攻击单次模型调用内部（prompt压缩、预算约束推理、投机解码），要么在模型之间做路由/级联——两类工作都把编排层当作既定前提，从未把它当作可优化对象。
- **之前卡在哪**：编排层（harness）实际控制了token账单里除模型自身表达冗余度之外的几乎每一项——系统提示是否被重放或缓存、历史是否被重放或压缩、工具schema是否被广播或按需暴露、检索payload多大、失败后是否重试——但此前没人把这层单独拎出来做控制变量实验。这需要一个足够成熟、能在多模型间零配置切换（模型名只是配置值）的编排层作为实验对象；此前业界主流编排层大多与单一模型/供应商绑定（如Claude Code只服务Anthropic模型），无法拿来做跨模型受控对照。
- **开启/关闭了哪条路线**：开启了"harness作为一等经济学对象"的路线——用形式化token账单分解（Eq.1-4）+ 六个机制家族 + CPM（每百万token任务完成数）/η$（每美元质量）这类效率KPI，把编排设计从工程细节提升为可在采购/发布决策中量化比较的经济变量，并主张CPM该和quality一起进发布门槛（对标芯片设计里performance-per-watt必须和performance一起报）。削弱了"能力提升必然靠更多token购买"的单一叙事：论文的核心论点是效率提升与质量提升不是同一次trade——效率收益无条件（换谁都省），质量收益有条件（够强的模型才吃得到），这也隐含削弱了"只看模型benchmark分数做采购决策"的传统做法。

## 核心机制

```
【一、单任务的token账单从哪来（Eq.2-3）】

第 i 轮输入token拆解：
  T_in_i = S_i(系统提示) + H_i(历史) + G_i(工具schema) + R_i(检索) + U_i(用户输入)

朴素实现：H_i = Σ_{j<i}(用户输入_j + 模型输出_j + 工具输出_j) ← 每轮把此前全部重放一遍
        ⇒ 累计输入token ≈ O(k²)  二次增长（k=agent轮次）

Harness管理：压缩历史 + 缓存不变前缀 + 卸载工具输出到文件/摘要
        ⇒ 累计输入token ≈ O(k)   线性增长
        （论文图1为示意图，非实测；作者原话："Schematic; measured
          aggregates appear in Section 6"——真实测量数字见下方"成本与量级"）

【二、价格本身不是一个数：有效输入价格公式（Eq.4）】

  p_eff_in = p_in × (1 − h×(1−κ))，  κ≈0.1（缓存读取≈原价的0.1倍）
  h = 命中缓存的输入token占比

  h 不是模型属性、也不是供应商恩惠，而是"prompt字节是否逐轮稳定"的函数——
  这完全由编排层决定，所以harness同时控制了"账单有多少项"和"每项计多少钱"。

【三、Harness的物理实现：两区Prompt（图2，唯一有实测数据支撑的机制）】

┌────────────────────────────────────┐
│ 工具Schema全目录 (~35个工具)     [C1]│ ← 缓存断点1
├────────────────────────────────────┤   byte-stable
│ 稳定系统Prompt (身份/契约)       [C2]│ ← 缓存断点2   前缀
├────────────────────────────────────┤   0.1×价格读取
│ 检查点摘要 (若发生过压缩)             │
├────────────────────────────────────┤
│ 持久对话记录 (仅追加)         [C3][C4]│ ← 沿最新两轮滑动的断点
└────────────────────────────────────┘
▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
▓ 易变尾部：时钟/文件列表/计划状态/一次性提醒 ▓ ← 每轮重建；缓存断点逻辑
▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓  在此处及之后被强制拒绝设置

实测：identical-prefix调用下 7,876/7,886 token（99.9%）served as cache reads。
"字节稳定"是纠错规则，不是可调优化项——任何逐轮变化的内容被结构性禁止进入前缀。
```

另外五个机制家族（均攻击 Eq.2 的具体项，见下方设计决策解剖展开其中三个）：结构化增量压缩（80%预算触发，折叠为含"决策/约束/被拒方案"的类型化checkpoint，助手模型跑摘要不占付费主循环）；上下文卸载（子Agent 8KB摘要上限、skill渐进式披露、超20K字符的工具输出溢出到文件、图片≤4张/2MB）；零token等待（等待是"挂起并按事件恢复"而非轮询循环，WAL+代序防护保证崩溃后不重买token）；失败支出治理（类型化失败分类、中途失败作废且无副作用、3次同参数失败触发熔断、循环上限50轮、工具并行度上限4）；模型无关地板（路由表作为数据、只走原生工具调用、$refs内联等schema卫生处理）。

## 设计决策解剖 ⚖️
| 决策点 | 他们的选择 | 被放弃的方案 | 为什么 | 这个选择何时失效 |
|--------|----------|------------|--------|---------------|
| 缓存断点放置规则 | 物理强制分离byte-stable前缀（工具schema+系统prompt+仅追加transcript）与volatile尾部；断点逻辑"拒绝"设在首条易变消息处或之后 | baseline的做法：49KB系统prompt与易变内容混排、整体重放，不做前缀/尾部区分 | 前缀哪怕多一个字节变化，累积缓存就整体失效；这是正确性规则而非优化选项——把易变内容排除在外是保住缓存命中率的唯一方式 | 当任务本质上要求几乎每轮都改变"系统级"上下文时（强约束逐轮变化，volatile tail装不下），byte-stable前缀被迫拆得很小，缓存收益趋近于零，维护两区结构的开销可能得不偿失 |
| 历史压缩方式 | 80%预算阈值触发，折叠为含durable memory/8节执行摘要/verbatim用户需求/skill引用的类型化checkpoint；保留最近4-12条消息（≤30%预算）原文不动；摘要在更便宜的helper model上跑 | baseline的"destructive middle-truncation when context overflows"——上下文溢出时直接从中段砍掉一段 | 中段截断不可逆且无结构，可能砍掉关键约束；checkpoint式压缩保留结构化字段，且重新生成的prompt本身成为新的可缓存前缀，与缓存机制协同设计 | 摘要degraded或为空时机制设计为直接放弃本次压缩而非硬压；若兜底的"缩尾部→中段截断"梯形仍不够用，最终退化为baseline式截断；且压缩质量依赖helper model表现，论文未披露helper model的具体成本/延迟数据 |
| 子Agent委派架构 | 子Agent在独立上下文里检索/阅读，只把≤8KB摘要+独立metadata sidecar（父模型不读）返回父循环；委派深度加帽且在重试下幂等 | CrewAI/AutoGen式共享transcript多Agent对话——各参与Agent共享并重读同一份不断增长的对话历史 | 共享transcript架构下每个Agent都要重读增长中的完整对话且各带一份角色前言，结构上必然是token乘数（论文引用Anthropic实测：多Agent系统消耗约15倍于chat基线token） | 这一净新增能力本身有能力地板：仅两个最强模型（Sonnet 4.6、Palmyra X6，0.85-0.86）越过可用可靠性门槛，Gemini 3.1(0.70)/GLM 5.1(0.58)明显退化，快速档模型(0.42-0.45)尚不可靠——把这个特性开放给弱模型档位目前不成立 |
| 实验基线选取 | baseline = Writer自己"以前的生产系统"，run once并冻结在2026-06-07 | 未选择LangGraph/CrewAI/Claude Code等外部框架做受控对照；Table 1对这些框架的比较明确"assessed from public documentation... not head-to-head measurement" | 要"只换编排层"做纯净的控制变量实验，两个编排层必须跑在完全同一套task/model/judge脚手架上；接入外部框架需重新适配调用协议，会引入额外混杂变量 | -41%等定量结论严格意义上只是"Writer新Harness vs Writer旧loop"的差值，不能证明Writer Harness优于市面其他成熟框架；论文Section 8自己承认"One pair...the magnitudes are specific to this pair" |

## 成本与量级 💰
- **训练成本**：不适用——本文是编排层软件工程+受控评测，不训练模型；论文未披露Harness软件本身的开发人月投入（明确的未披露项）。
- **推理成本 vs 基线（Table 3，blended across 22任务×6模型）**：成本/任务 $0.21→$0.12（-41%）；wall-clock中位数 48s→27s（-44%，1.8倍）；tokens/任务 14.2k→8.8k（-38%）；quality per dollar (η$) 3.71→6.75（+82%）；CPM（每百万token任务完成数）54.9→92.0（+68%）。
- **per-model范围（Table 4）**：成本降幅最小Gemini 3.1 -33%，最大Flash 3.5 -61%；延迟降幅33%(Qwen)-55%(Flash 3.5)；**每个模型无一例外都变便宜**，这是论文认定"层级效应"而非"模型特定行为"的关键证据。
- **缓存价格杠杆**：命中缓存的input token按≈0.1×原价计费（κ≈0.1），实测两区prompt在identical-prefix调用下达到99.9%命中率；论文引用外部数据称生产agent的输入输出token比接近100:1（input-dominated），因此缓存命中率是"单个杠杆最大"的成本变量。
- **舰队规模经济（图7）**：100万次agent任务/月，baseline月支出$210k，harness月支出$120k，**年省$1.08M**；论文强调这一节省"model-portable"（换模型/供应商仍乘数生效，因为实现在API之上）+"volume-linear"（随任务量线性放大）+ 可与per-token降价/路由/prompt压缩等其他优化叠加而非替代。
- **工程复杂度**：论文未给出团队人月投入的具体数字；但从"trace shim + WAL持久化 + 类型化失败分类 + 4个provider缓存断点latching"等描述看，这是生产级中间件量级的工程投入，而非prompt模板优化量级。
- **最小可行配置**：若只做一件事，论文自身的机制排序隐含建议——优先做"两区prompt+缓存断点"（机制1），因为这是"single highest-leverage cost variable"，理论上单这一项就能把dominant input term的价格打到约0.1×list。

## 证据审计 🔬
- **实验设计公平吗？基线选取有什么猫腻？** 最大的猫腻是利益结构：全体作者受雇于Writer Inc.，该公司同时开发被评测的Harness和六个模型中的Palmyra X6，末位作者是联合创始人兼CTO（论文在Disclosure中主动披露，透明度值得肯定，但评测方、Harness开发方、候选模型开发方三者合一的结构性利益冲突依然存在）。其次，baseline是Writer自己的旧生产系统，run once并冻结，没有测量baseline自身的run-to-run方差——如果那一次冻结跑恰好是运气差的一次，差值会被高估，论文未说明。baseline的描述（49KB系统prompt每轮重放、正则解析XML工具调用、破坏性中段截断、轮询等待）读起来颇有"稻草人"色彩，是否能代表2026年"业界一般水准"存疑；Table 1对LangGraph/CrewAI/Claude Code的比较论文自己承认"not head-to-head measurement"，真正跑分对照的只有Writer vs Writer自己。
- **最强证据（数字+成立条件）**：①效率类delta（成本/token/延迟）在22任务×6模型全部方向一致下降、无一反例，这种跨越132个任务实例无反例的一致性是比n=22本身更有说服力的证据形式；成立条件是这一"decisive"判断（Section 5.5）是作者自己的定性描述，并非经过正式统计检验（无置信区间、无p值）。②缓存命中率99.9%（7876/7886 token）是一个可独立复核、不涉及LLM-judge主观打分的硬指标，直接验证了Eq.4理论价格公式在真实前缀上生效，是全文最"干净"的单点证据。
- **最可疑的数字**：headline quality "0.78→0.81，+0.03"被反复强调为parity/wash，但这个blended平均数掩盖了方向相反的真实变化——48个能力×模型格子里30升11平7降，且"multi-step research synthesis"（成本最贵的任务）质量从0.80暴跌到0.60（同时成本降46%）。论文6.6/7.5节其实主动坦白了这一点（"the one place the aggregate parity conceals a real trade"），但只读摘要容易被"+0.03持平"带偏，忽略这个真实倒退。次可疑：harness leverage的r=0.99相关系数只基于n=6个模型点，六点拟合出0.99的相关系数统计上极脆弱，作者自己承认"suggestive rather than conclusive"，但摘要仍用"almost perfectly"这样的强语言，容易被下游引用者截断成孤立的"r=0.99"反复传播。
- **审稿人会要求补充**：①baseline的多次独立运行（至少3-5次）估计方差；②针对六个机制家族的消融实验——目前是"harness整体 vs baseline整体"的bundle对比，不知道38%的token节省里两区缓存、结构化压缩、子Agent卸载各贡献多少；③与至少一个非Writer开发、真正独立部署的第三方harness做同任务集的head-to-head测量；④更大的模型面板以让harness leverage的r=0.99有更稳的统计基础。
- **最小复现实验**：只复现机制①缓存塑形本身（影响力最大、最容易独立验证）。选1个支持prompt caching的模型（如通过OpenRouter），手写最小"两区prompt"封装：工具schema+系统prompt+仅追加历史放稳定前缀，时间戳/临时状态放尾部；在8-12轮的10-20个多轮任务上分别跑"朴素全量重放"与"两区结构"，读取API返回的cache_read token数计算命中率h，代入Eq.4验证有效输入价格是否降到约0.1×list。预算：数十次多轮调用，约$10-30美元，1-2天可完成。

## 可复用点（PM决策）
- **何时采用**：组织已在多模型/多供应商间切换评估，且agent任务量级接近"月度百万级"（论文的$1.08M/年案例是在此量级下算出，量级小很多倍时ROI需重新核算）；agent工作流本质是多轮、input-dominated（检索/工具结果反复重放），对缓存塑形和历史压缩最敏感；团队愿意先建per-task可观测性（token/成本/延迟trace）再谈优化——论文反复强调"不可测量就不可管理"，这本身是低成本高杠杆的起手式。
- **何时规避**：任务集本身低轮次/单次调用为主，Harness多数机制（压缩、防火墙委派、失败治理）用不上，直接prompt层优化收益更高；计划把子Agent委派/复杂多步Playbook这类高阶特性无差别开放给较弱/低成本模型档位——论文数据明确显示这类特性有能力地板，弱模型用了可能不仅不省心还倒退质量；只有一次性/短期项目、没有月度task volume积累，编排层投入的固定工程成本未必能被摊薄。
- **供应商拷问清单**：①"能否提供按任务拆分的token/成本/延迟trace，而非只有账期汇总账单？"——直击Table 1里五个对比系统都没有的"per-task accounting"能力，没有它token maxing就不可见即不可管理。②"你们的prompt缓存命中率(h)大概多少？系统prompt和工具schema是否放在字节稳定前缀里？"——对应Eq.4，答不出这个数字大概率意味着没做两区prompt式塑形。③"子Agent/委派功能在哪些模型档位上有实测可靠性数据？是否所有档位都开放同一套高阶编排功能？"——直击能力地板发现，警惕对弱模型档位一视同仁开放高阶功能导致质量倒退。

## 关联网络 🕸️
- **相关论文**
  - 本文强关联 [[Wiki/概念/04_Agent框架/Harness设计模式]]——此前该概念页聚焦"接口/状态管理如何提升成功率"（Harness-1状态外化、LIFE-HARNESS四层生命周期、EFC扩展律），本文补上了"这套接口设计如何决定token账单"的经济学维度，可作为该概念页后续新增"Token经济学"子板块的候选来源。
  - [[Wiki/论文笔记/03_Agent系统设计/34_Agent-Harness有效反馈计算扩展律]]——方法论呼应：34号EFC扩展律主张"harness的原始计算量(token/调用次数/时长)几乎不能预测任务成功率(R²≈0~0.27)，只有反馈质量(EFC)才能"；172号的CPM/η$框架是反过来在成本侧问同一类问题——不能只看质量或token这两个独立数字，要看比值（效率）才是真正该优化的KPI。两篇论文分别在"反馈质量"和"经济产出"维度上呼吁用复合指标替代原始单一指标，逻辑同构，测量对象不同。
  - [[Wiki/论文笔记/03_Agent系统设计/128_生产LLM-Agent运行架构模式方法论]]——同属生产级Agent运行时架构但视角互补：128号SDB框架关注正确性边界（Proposer/Verifier/Commit/Reject Signal，防止错误副作用提交到真实世界）；172号关注同一运行时的经济性边界（同样做对的事，账单能不能更小）。172号的"failure-spend governance"（类型化失败分类+熔断+丢弃部分执行结果）实质是SDB四部件在token经济学视角下的重新表述——Reject Signal对应typed failure class，Commit对应"no side effects from a discarded attempt"。
  - [[Wiki/论文笔记/03_Agent系统设计/45_LIFE-HARNESS适配接口而非模型]]——**张力关系**：45号核心论点是接口修复的收益跨模型可迁移——在弱模型(Qwen3-4B)上演化出的Harness直接迁移到17个更强模型上仍普遍改善(116/126格子提升)；172号的"harness leverage"发现则是收益强烈依赖模型基线能力(r=0.99)，弱模型(Qwen 3.6均值-0.031)在orchestration-heavy能力(MCP工具调用、Playbooks)上反而退步。这不是直接矛盾：45号测的是确定性规则环境下相对基础的格式/合约修复，172号测的是企业级检索/工具调用/委派这类更复杂的编排结构——可推测基础接口修复对所有模型普惠，但更高阶编排结构存在模型强度门槛，值得后续追踪。
  - [[Wiki/论文笔记/03_Agent系统设计/40_多Agent是否有助于性能提升受控评测]]——**印证关系**：40号受控实验发现固定角色/共享transcript的传统多Agent系统5/6落后单Agent基线2.56-11.29点，唯一表现好的例外是"runtime-generated workflow"（Claude Code风格动态spawn子Agent，且token消耗更低）；172号把子Agent委派设计为"上下文防火墙"（8KB摘要上限，父循环不重读子Agent探索过程），并直接批评CrewAI/AutoGen式共享transcript架构是token乘数。两篇论文独立收敛到同一判断——动态、上下文隔离的委派模式优于固定角色共享对话模式；172号补充的新信息是即使更优的委派模式仍只在两个最强模型上越过可用门槛。
  - [[Wiki/论文笔记/13_评测科研/174_超越排行榜Agent失败模式综述]]（该笔记末尾此前已预标注"与172号相关但尚未入库、暂不加链接"，现正式建立此链接）——**印证关系**：174号Section 7核心论点之一是"额外scaffolding（更多Agent/工具/推理量/上下文）收益不均衡甚至为负"；172号6.3节的实测数据正是这一论点的具体量化实例——48个能力×模型格子中7个退步，全部落在三个较弱模型上，且集中在orchestration-heavy能力（MCP工具调用、Playbooks、Presentations）。172号用真实美元和quality分数把174号偏定性的论点坐实为一组可引用的具体数字。
- **相关概念**
  - [[Wiki/概念/05_记忆与检索/上下文压缩与KV缓存]]——不同层级的同源资源：该概念页讨论模型/推理服务层的KV缓存压缩（稀疏注意力、量化、MLA，解决"显存装不下"）；172号讨论应用/编排层的prompt缓存塑形（解决"同样前缀能否按折扣价重复计费"）。两者层级不同但依赖同一底层机制——provider的prefix caching本质是对服务端KV缓存的复用，172号"字节稳定前缀"设计正是从应用层主动配合、最大化这种复用率。该概念页中AdaCoM的实测结果（动态上下文管理相比固定窗口截断，准确率+12%、成本-40%）与172号"结构化压缩优于破坏性中段截断"的机制结论相互印证。
  - [[Wiki/概念/04_Agent框架/MCP]]——172号"模型无关地板"机制直接涉及MCP工具schema的暴露与调用中介（$refs内联、双重编码JSON参数恢复、schema卫生处理），是MCP概念页"工具集成"抽象功能层在token经济学视角下的具体工程实现。
- **冲突/印证**：核心张力已在45号条目展开（接口修复的模型无关普惠性 vs 高阶编排特性的能力地板）；核心印证已在40号和174号条目展开（动态委派优于共享transcript的架构判断；scaffolding收益不均衡的定量实例）。

## 动手练习 💻
**练习目标**：复现论文核心机制——朴素全量重放 vs Harness式两区管理在k轮Agent循环下的token增长差异（Eq.2-3，对应图1的"示意"逻辑），代入有效输入价格公式（Eq.4）估算真实成本节省量级，并用论文Table 3的真实数字验证CPM/η$两个效率KPI的计算方式（Eq.5）。

```python
"""
练习：Harness Effect 论文核心机制的最小复现
第1-2部分是"量级直觉"模拟（参数为示意值，非论文精确复现——
论文图1本身也明确标注为 illustrative/schematic，未披露真实m_bar/S/G数值）；
第3-4部分直接代入论文Table 3的真实测量数字，可精确复现论文报告的KPI结果。
"""
import numpy as np

# ===== 场景参数（示意值，用于建立O(k^2) vs O(k)的量级直觉） =====
k = 12          # agent 轮次，对应论文图1横轴上限
S = 400         # 系统提示token数（示意值，非论文49KB baseline的精确换算）
m_bar = 500     # 朴素实现下每轮平均新增payload（用户输入+模型输出+工具输出）
G = 300         # 工具schema token数（示意~35个工具的目录）
R_i = 150       # 每轮检索payload（简化为常数）

def naive_cumulative_tokens(k, S, m_bar, G, R_i):
    """朴素harness每轮把此前全部历史重新提交一次：
    第i轮历史项 H_i = 前(i-1)轮payload之和，H_i本身随i线性增长，
    对i累加后总输入token呈O(k^2)——对应论文Eq.3的二次项 k(k-1)/2 * m_bar。"""
    cumulative, total = [], 0
    for i in range(1, k + 1):
        H_i = (i - 1) * m_bar          # 全量重放前i-1轮
        total += S + H_i + G + R_i
        cumulative.append(total)
    return cumulative

def harness_cumulative_tokens(k, S, m_bar, G, R_i, tail_window=6):
    """Harness用checkpoint压缩历史：只保留最近tail_window轮verbatim，
    更早历史折叠为近似固定大小的摘要（此处简化为0.3*m_bar的常数开销），
    H_i不再随i无限增长，累计token总量趋于线性于k。"""
    cumulative, total = [], 0
    for i in range(1, k + 1):
        recent = min(i - 1, tail_window)
        checkpoint_cost = 0.3 * m_bar if (i - 1) > tail_window else 0
        H_i = recent * m_bar + checkpoint_cost
        total += S + H_i + G + R_i
        cumulative.append(total)
    return cumulative

naive = naive_cumulative_tokens(k, S, m_bar, G, R_i)
harness = harness_cumulative_tokens(k, S, m_bar, G, R_i)

print("轮次   朴素重放累计token   Harness管理累计token   节省比例")
for i in range(k):
    saved_pct = (1 - harness[i] / naive[i]) * 100
    print(f"{i+1:>4}   {naive[i]:>12,}       {harness[i]:>12,.0f}       {saved_pct:>5.1f}%")
print("（真实agent轮次/payload通常远大于此处示意值，O(k^2)与O(k)的差距会随k")
print(" 增大而急剧拉开——可自行调大k或m_bar观察发散速度）\n")

# ===== 2. 有效输入价格公式（Eq.4）：缓存命中率h如何压低每token实付价格 =====
def effective_input_price(p_in, h, kappa=0.1):
    """h=命中缓存的输入token占比，kappa=缓存读取相对原价的折扣倍率(≈0.1)。
    论文实测两区prompt在identical-prefix调用下 h=99.9%（7876/7886 token，图2）。"""
    return p_in * (1 - h * (1 - kappa))

p_in_list = 3.0 / 1_000_000     # 示例基准单价 $3/百万token（非论文具体数字）
p_eff_baseline = effective_input_price(p_in_list, h=0.0)     # baseline几乎不命中缓存
p_eff_harness = effective_input_price(p_in_list, h=0.999)    # harness实测命中率
print(f"Eq.4 有效输入价格：baseline≈${p_eff_baseline*1e6:.3f}/M tokens，"
      f"harness≈${p_eff_harness*1e6:.3f}/M tokens "
      f"（降至约{p_eff_harness/p_eff_baseline*100:.1f}%）\n")

# ===== 3-4. 效率KPI：CPM与η$（Eq.5）——直接代入论文Table 3真实数字验证 =====
def efficiency_kpis(Q, C, tau):
    """Q=质量分数[0,1]，C=单任务成本(美元)，tau=单任务总token数。"""
    return Q / C, Q * 1_000_000 / tau      # 返回 (η$, CPM)

Q_base, C_base, tau_base = 0.78, 0.21, 14200          # 论文Table 3 baseline实测
Q_harn, C_harn, tau_harn = 0.81, 0.12, 8800            # 论文Table 3 harness实测

eta_base, cpm_base = efficiency_kpis(Q_base, C_base, tau_base)
eta_harn, cpm_harn = efficiency_kpis(Q_harn, C_harn, tau_harn)

print("论文Table 3 headline数字复现：")
print(f"  baseline: η$={eta_base:.2f}  CPM={cpm_base:.1f}")
print(f"  harness : η$={eta_harn:.2f}  CPM={cpm_harn:.1f}")
print(f"  η$提升 {(eta_harn/eta_base-1)*100:.0f}%（论文原文：+82%）")
print(f"  CPM提升 {(cpm_harn/cpm_base-1)*100:.0f}%（论文原文：+68%）")
# 两行输出应精确匹配论文摘要的 +82% 和 +68%——因为KPI公式(Eq.5)是纯代数运算，
# 不依赖任何模拟假设，可用来验证"读懂了公式"而不只是"记住了结论"。
```

## 自测三层 🎓
**L1 复述**
- Q：论文定义的"token maxing"（Definition 1）具体指什么，两个判据是什么？
- A：一个开发轨迹{(Q_t,τ_t)}呈现token maxing，当且仅当：①token强度持续增长（τ_{t+1}>τ_t）；且②边际质量/token在下降（(Q_{t+1}-Q_t)/(τ_{t+1}-τ_t) < Q_t/τ_t，即每次迭代买到的质量兑换率比系统历史平均更差）。
- Q：受控置换实验测出的blended headline三个效率数字和质量数字分别是多少？
- A：成本/任务 -41%（$0.21→$0.12）；wall-clock中位数 -44%（48s→27s）；tokens/任务 -38%（14.2k→8.8k）；质量headline持平（0.78→0.81，n=22下判定为directional非显著）。

**L2 解释**
- Q：子Agent委派被设计成"上下文防火墙"（8KB摘要上限+独立sidecar），为什么不直接让子Agent和父Agent共享同一个transcript（像CrewAI/AutoGen那样做多Agent对话）？
- A：共享transcript架构下每个参与Agent都要重读不断增长的完整对话历史，且各自携带一份角色前言，结构上必然是token乘数（论文引用Anthropic实测：多Agent系统消耗约15倍于chat基线token）。防火墙式设计把子Agent的探索过程隔离在自己的上下文里，只把浓缩摘要和引用sidecar带回父循环，父循环从不为子Agent"怎么找到这个答案"的过程付费——委派成本变成有界旁路支出，而非随委派深度持续复合增长的主循环负担。
- Q：论文为什么坚持用CPM和η$这两个复合指标，而不是像大多数benchmark一样只报告quality/task-completion？
- A：因为只报告质量会让token maxing对团队而言"理性"——若KPI只看质量分数，用更多token换一点质量提升在考核上划算，但token是"别人的账目"（组织付费）。CPM和η$把"花多少资源换这个质量"直接编码进KPI本身，报告CPM/η$的团队没法再靠堆token刷分——这是Section 7.3"change the KPI"的核心论点，类比芯片设计里performance-per-watt必须和performance一起报的逻辑。

**L3 应用**
- Q：你是一家医疗AI问诊助手的PM，正评估是否把现有单体prompt式Agent（每轮把完整问诊记录和知识库检索结果重新塞进prompt）改造成本文的两区prompt+结构化压缩架构。结合"能力地板"和"harness leverage"发现，你会如何设计上线策略，需要追问哪些问题？
- A（思路参考，非标准答案）：①按模型层分阶段灰度——效率增益在所有模型上无例外成立，可先在全部模型档位上换编排层拿效率收益；但"多步问诊工作流""委派"这类高阶编排特性先只对接最强1-2个模型档位，避免对成本更低档位开放后出现Qwen 3.6/GLM 5.1式的质量倒退，医疗场景对此类倒退的容忍度更低。②要求提供按能力维度拆分的before/after对比（类似论文48格矩阵），尤其单独审查长程/多步问诊这类orchestration-heavy场景是否退步。③追问缓存命中率h的实测值——问诊场景高频复用同一套知识库检索结果和安全护栏prompt，若未做成字节稳定前缀，等于放弃了Eq.4里最大的单一折扣杠杆。④即便blended headline质量显示parity，也要求单独核查最复杂那几类任务是否被平均数掩盖了真实倒退（对应论文"multi-step research synthesis"0.80→0.60的教训）——这类任务往往正是医疗场景风险最高、最不能接受质量倒退的部分。

📅 知识时间锚：论文提交 arXiv 2026-07-08（baseline冻结于2026-06-07）· 本笔记复核 2026-07-16
