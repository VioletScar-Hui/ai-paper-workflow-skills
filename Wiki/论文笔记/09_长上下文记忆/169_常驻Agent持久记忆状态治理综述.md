---
tags: [论文笔记, Agent记忆, 持久状态, 治理, 综述, 长上下文记忆, LLM-Agent]
paper_id: "169"
filename: "169 - Always-On Agents - A Survey of Persistent Memory, State, and Governance in LLM Agents.pdf"
authors: "Tianyu Ding, Aditya Nannapaneni, Bingfan Liu, Ling Zhang"
year: 2026
笔记层级: 骨干
成熟度: 🧪
复核日期: 2026-07-15
---

# 常驻 Agent：LLM Agent 中持久记忆、状态与治理综述（Always-On Agents: A Survey of Persistent Memory, State, and Governance in LLM Agents）

📄 **原文 PDF**：[[RAW/169 - Always-On Agents - A Survey of Persistent Memory, State, and Governance in LLM Agents.pdf]]

## PM 速判（30秒）
> 一句话：这篇 107 页正文、编码 435 篇文献的综述提出——"常驻 Agent"的正确分析单元不是"记忆"而是"持久状态"（涵盖任务台账、权限凭证、承诺、溯源审计、共享状态、触发条件），并系统性证明整个领域在"积累和检索状态"上高度发达，但在"治理状态"（撤销、传播删除、回滚）上几乎空白——435 篇文献里只有 27 篇涉及任何回滚机制，authority（权限授权）是六个状态维度中覆盖最稀缺的（72篇）。PM 必须关注：任何"我们的 Agent 有记忆"的供应商话术，都要追问"删除请求真的传播到所有派生副本了吗？权限撤销后旧记忆还能授权动作吗？出错了能回滚吗？"——论文用小规模 pilot 证明，主流做法目前绝大多数答不上来这三个问题。

## 双层费曼 🗣️
> **给CEO的一句话**：一个"记性好"的 AI 助手不等于一个"靠谱"的 AI 助手——它记得你上个月说过的偏好很棒，但如果它不知道你上周已经改主意了、不能在你要求"忘掉这件事"时真正删干净、也没法在记错了之后撤销已经造成的后果，这种"记忆"就是隐患而不是资产，而今天几乎所有的记忆系统都停留在"记得住"这一步，没做到"管得住"。
>
> **给工程师的一段话**：论文把"始终在线（always-on）"的 Agent 重新定义为"持久状态系统"而非"记忆增强系统"，用六个诊断维度刻画任意一条状态记录——authority（谁授权它影响动作）、scope（用于哪个用户/任务/时间窗）、mutability（能否被修订/废止）、provenance（来源与变换链条）、recoverability（能否回滚）、actionability（是证据/偏好/策略还是可执行承诺）——并把状态的生命周期拆成十个阶段（observe→write→validate→organize→retrieve→act 前向弧，update→forget→audit→rollback 回程弧），归纳出五条不变量（authority monotonicity、scope non-expansion、deletion propagation、provenance preservation、rollback traceability）。基于 435 篇编码文献，作者发现前向弧（尤其 retrieve 269篇、write 200篇）远比回程弧发达（rollback 仅27篇、authority 仅72篇覆盖），并据此提出 Always-On Evaluation Protocol（AOEP）——一套用事件流+快照schema、确定性不变量校验（不用LLM-judge）打分"义务是否履行"与"是否零泄漏"的评测协议，小规模pilot显示主流记忆方案（朴素追加、全上下文、向量RAG、Mem0类抽取式记忆）在治理类义务上普遍只拿到15项里的3~7项，且抽取式记忆反而比原始文本存储更差，因为抽取过程把结构化的权限/删除信息压扁成了纯文本。

## 问题域定位 🎯
- **回应什么根本约束**：LLM Agent 领域过去默认把"记忆"当作分析单元——写、组织、检索、遗忘几个操作，配合情景/语义/程序记忆几种分类。论文指出这个单元太窄：一个被撤销的 API 令牌、一笔未完成的购买、一条只传播了一半的删除请求，都不自然地算作"记忆"，但都决定着 Agent 未来行为是否安全。论文援引认知科学传统（Soar、ACT-R 等认知架构）指出，这些经典架构本就把记忆当作"受治理的系统组件"而非被动存储库，且靠架构本身（闭合的符号系统）强制类型边界；但 LLM Agent 的记忆类型只是文本条目上的一个标签，没有运行时强制的类型隔离机制——"程序技能可以悄悄覆盖语义事实、过期权限可以授权工具调用、被删除的记录可以在派生层里存活"，这个隔离必须靠治理重新建立，而不是像经典架构那样天然免费获得。
- **之前卡在哪**：既有的记忆综述把记忆当作"能不能被正确召回"的问题来研究，评测集中在长期对话问答（LoCoMo 类）、检索精度、遗忘曲线；这条路线在"记得对不对"上已经相当成熟，但对"这条记忆有没有权限影响当前动作""它是否还有效""能不能追溯删除"这些问题几乎没有覆盖——论文用编码统计证明：语料库里检索（retrieve, 269篇）和写入（write, 200篇）占绝对主导，而撤销、追责、回滚这类"回程弧"操作系统性稀缺（forget 66篇、audit 88篇、rollback 仅27篇）。
- **开启/关闭了什么技术路线**：论文没有提出一个"终极记忆架构"，而是把"记忆系统研究"重新定义为"持久状态治理研究"，并给出可执行的评测契约（AOEP）作为下一步的度量基础设施——这为"治理层应该是记忆系统的一等公民而非事后补丁"提供了系统性论证框架，同时警示了一条容易被忽视的岔路：单纯堆砌更大上下文窗口、更强检索并不能填补治理缺口（论文用多组读者规模对比证明，从 3B 到 8B 甚至换成更强模型都不改变治理义务的通过率，因为缺失的是"结构字段"而非"读取能力"）。

## 核心机制

持久状态记录的数据模型（论文式 s = ⟨v,a,c,m,p,r,k,τ⟩）与十阶段生命周期（复现自论文 Fig.5 + 正文数字）：

```
持久状态记录：s = ⟨ v, a, c, m, p, r, k, τ ⟩
                值  授权 范围 可变性 溯源 可恢复性 可执行性 时间戳
（多数记忆系统只显式存储 v，其余七个字段构成"治理信封"，
  丢弃信封＝丢弃了未来做治理判断所需的一切依据）

┌─────────────────── 前向弧：积累与使用（语料库中高度发达）───────────────────┐
│ observe → write → validate → organize → retrieve → act                    │
│  观察      写入     校验(缺失)   组织/压缩    检索(最发达)  执行            │
│  68篇     200篇     87篇       128篇        269篇        141篇             │
└──────────────────────────────────────────────────────────────────────────┘
                                    │ 结果已知，进入回程弧
┌─────────────────── 回程弧：治理与恢复（语料库中系统性稀薄）───────────────────┐
│ act → update → forget → audit → rollback                                  │
│        更新      遗忘     审计     回滚(最稀缺)                            │
│        127篇     66篇     88篇     仅27篇，且无一篇报告"恢复成功率/成本"      │
└──────────────────────────────────────────────────────────────────────────┘

五条不变量（贯穿生命周期，每种失败模式=违反其一）：
  authority monotonicity  —— 授权只能收窄，不能悄悄扩大（validate/act 处校验）
  scope non-expansion     —— 范围只能收窄或保持，不能悄悄扩大（organize/retrieve 处校验）
  deletion propagation    —— 删除必须级联到全部派生副本（forget 处执行，audit 处核验）
  provenance preservation —— 压缩/整合不能丢弃溯源链（organize/update 处保留）
  rollback traceability   —— 每个动作都要留一个指回授权记录的handle（act 处埋点，rollback 处消费）

六个状态轴中最稀缺 vs 最常见（435篇语料库统计）：
  authority（授权） 72篇 ◄── 最稀缺，几乎没人管"谁授权这条记录影响动作"
  recoverability     112篇
  provenance         153篇
  mutability         160篇 ◄── 最常见（遗忘/更新研究最多，但多为"浅层衰减"）
```

## 设计决策解剖 ⚖️

| 决策点 | 他们的选择 | 被放弃的方案 | 为什么 | 这个选择何时失效 |
|---|---|---|---|---|
| 分析单元 | 把"持久状态"（含权限、凭证、任务台账、溯源审计、共享状态、触发条件、外部承诺）定义为分析单元，比"记忆"更宽 | 沿用既有"记忆形式分类学"（情景/语义/程序/偏好记忆）作为综述组织框架 | 撤销的 API 令牌、未完成的转账、只传播一半的删除请求都不自然算"记忆"，但都决定 Agent 未来行为是否安全；窄单元会把治理相关的记录系统性排除在分析范围外 | 部署场景本身极其受限（单用户、单会话、无持久权限、无外部副作用）时，宽泛的持久状态框架是过度设计——不需要为一个"问一句答一句"的无状态问答机器人套用六轴治理模型 |
| 语料库构建策略 | 用15轮迭代检索，后期大部分轮次刻意瞄准治理相关关键词（rollback、authority、deletion propagation、durable execution、数据库/分布式系统/HCI文献），即"过度采样"治理方向 | 仅用记忆类关键词做有机检索（"agent memory"、"long-term memory" 等） | 若只用有机检索，治理覆盖率低可以被质疑为"你们没找对地方"；论文用"专门加码搜索治理仍然只把治理占比从约1/6提到1/3、且后期轮次收益递减"来反驳这个质疑，把证据做实 | 若读者把 N=435 当作详尽普查（exhaustive census）而非"有范围的地图（scoped map）"，会误用论文数字做全领域流行率估计——论文自己在方法论附录明确警示这一点 |
| AOEP 的打分方式 | 用确定性不变量校验（validator 自己重算每个布尔值，从不信任被测系统的自我汇报），刻意不用 LLM-as-judge | 用 LLM-as-judge 对轨迹做整体打分（呼应 agent-as-judge 一类研究路线） | LLM-judge 已知存在位置偏差、冗长偏差、自我抬升偏差，而治理类检查（"被tombstone的值是否仍可检索到"）本质是可判定谓词，没必要冒judge偏差的风险；论文明确把"可判定谓词"和"需要主观判断的整体轨迹质量"分开处理 | 遇到无法规约为确定性谓词的治理性质时（如"这次主动打扰用户的时机是否恰当"这类 trigger 校准问题），确定性校验无能为力，论文承认这类"何时该主动出手"的问题目前仍缺乏治理化的评测手段 |
| AOEP 打分表拆分 | 把"义务通过率"（obligation pass，系统必须主动满足的正向义务）和"负向不变量通过率"（negative-invariant pass，零泄漏检查）分开报告，不合成一个总分 | 用单一综合分数（如"平均通过率"）衡量系统的治理水平 | 一个什么都不存的系统（no-memory floor）在负向不变量上天然满分（它没存过东西所以不可能泄漏），若混合计分，"失忆"系统会显得很体面；拆开后能清楚看到 no-memory floor 是"0/15 义务 + 41/41 负向"——即靠彻底不作为换来的假安全 | 当下游只看"一个总分"做产品对比（报告风险）时，拆分设计的价值会被压缩——如果读者只截取其中一个数字引用，仍可能重现"合成分数"的误导效果 |

## 成本与量级 💰
- 语料库构建成本：论文未披露具体人力/工时，描述为"记录在案的检索协议"，15轮迭代检索 + 双人盲评236篇样本做信度检验，属于系统综述量级的人力投入（人月级，论文未给精确数字）。
- AOEP pilot 的算力：全部本地计算、贪婪解码，读者模型为 Qwen2.5-7B（另做了 Qwen2.5-3B / Llama-3.1-8B / Mistral-7B-Instruct 的读者稳健性对比）+ all-MiniLM 向量嵌入 + 本地 mem0ai 包配置，未使用云端大模型或 GPU 集群，量级是"单机可复现的小规模验证"，不是工业级评测。
- Pilot 规模：9种"故障模式"（3个手写worked episode + 6个新增模式）+ 3个模拟真实基准风格的长episode（tau-bench式退款、TheAgentCompany式排期、AppWorld真实Venmo/Spotify/文件系统轨迹）+ 5个随机种子的采样解码稳健性检验。论文自己反复强调这是"pilot 而非 benchmark"，样本量小，不能当作产品级排行榜。
- 我的产品要用的最小可行配置：若要在自己的 Agent 系统上验证"记忆是否被治理"，论文提供的最小复现路径是——定义 AOEP 风格的事件 schema（idempotency key、causal links、permission epoch、trust tier、retention policy、11种操作类型），构造几个含"重启-冲突-删除请求"的合成episode，跑通"义务通过率 vs 负向不变量通过率"两条线，不需要大模型或大预算，一个工程师一周内可以做出最小验证版本。

## 证据审计 🔬
- **实验设计公平吗？基线选取有什么猫腻**：AOEP pilot 里"朴素追加 / 全上下文 / 向量RAG"三种系统在9个故障模式上打出完全相同的 7/15 义务分——论文没有回避这个"太整齐"的结果，反而把它当作核心证据：三者检索策略不同但分数相同，说明缺失的不是检索能力而是治理字段（permission epoch、deletion ledger、conflict record）根本不存在于这些文本存储里；论文进一步用读者规模扫描（3B到8B、三个模型家族）和5个随机种子的采样解码复现了这个平台效应，方法论上是扎实的收敛性检验。但要注意：全部pilot只用了9个故障模式+3个真实episode，属于"小样本方向性证据"，论文自己称之为"representational, not benchmark-grade"。
- **最强证据（PDF中的真实数字+成立条件）**：435篇编码语料库中，检索（retrieve）出现在269篇、写入（write）出现在200篇，而回滚（rollback）仅出现在27篇、authority轴仅覆盖72篇——这是一个数量级差距（约10倍），论文正确地指出：即使读者不信任精确的"27"这个数字（信度检验显示双人盲评对 rollback 标签仅在19篇中的11篇达成一致），这个方向性结论（回程弧远比前向弧稀薄）也很难通过任何合理的重新分类来推翻。成立条件：该结论仅在论文的检索框架（arXiv cs.AI/CL/LG/CR + Semantic Scholar + OpenReview + ACL Anthology，2023-2026窗口）内成立，是"有范围的地图"而非全领域普查。
- **最可疑的数字**：27/435这个"回滚机制"计数本身——论文附录A.6主动承认这个数字合并了三种机制上完全不同的操作（内部状态回滚、工作流回滚、外部副作用回滚），对最容易的子类（内部状态回滚）是宽松估计，对最难、最有实际意义的子类（外部副作用回滚，比如撤销一笔已经发生的转账）几乎肯定是高估——即真正能做到"撤销已发生的不可逆外部动作"的系统数量，可能远少于27。另外双人盲评在 rollback 标签上仅11/19一致，说明这个数字本身的置信区间相当宽。
- **审稿人会要求补充**：
  1. 论文自己点名的"envelope ablation"实验——给抽取式记忆（如Mem0-style）加上最小治理信封（permission epoch + deletion ledger + trust tier）后重跑9个故障模式，看差距能不能被补上——论文明确说"我们没有做这个实验"，这是最直接会被追问的缺口，因为它决定了"现有系统只是没装治理字段"还是"抽取式记忆架构本身有结构性损失"这两种完全不同的结论。
  2. 回滚的恢复时间/恢复点目标（RTO/RPO）与恢复成本——论文承认435篇里没有一篇报告"修复成功率"或"修复代价"，AOEP目前也只把rollback当"通过/不通过"的二元义务打分，没有量化代价。
  3. 语料库信度检验只做了236篇的双人盲评抽样（超过一半但非全量），且state axes的Cohen's κ=0.44（按常规区间只能算"中等偏弱"一致性）——大概率会被要求全量双编码或更高信度的复核。
- **最小复现实验（代理指标+预算）**：仿照论文AOEP pilot的最小版本——手写5~8个"故障模式"事件流（含一次重启后的重复写入、一次冲突性更新、一次删除请求、一次权限撤销后的滞后动作），分别喂给（a）一个只做向量检索的简单RAG系统和（b）一个手写规则的"治理版reducer"（作为oracle上界），对比两者在"删除后是否真的检索不到""撤销权限后是否阻止了动作"这两条负向/正向检查上的通过率差异。预算：不需要GPU，用任意本地7B模型跑几十次推理即可在半天内得到方向性信号，足以验证"文本存储缺治理字段"这一核心论断的最小可行版本。

## 可复用点（PM决策）
- **何时采用**：任何要采购或自建"带记忆的Agent"产品，且这个记忆会跨会话持久化、会影响后续动作（尤其是有外部副作用，如下单、发邮件、改权限）的场景，都应该用本文的六轴框架（authority/scope/mutability/provenance/recoverability/actionability）去逐项拷问供应商，而不是只看"记忆召回准不准"这一个维度。
- **何时规避/谨慎**：不要被"我们用了向量数据库/知识图谱/Mem0类记忆系统"这类描述误导为"记忆已经被治理好了"——论文的pilot证明，抽取式记忆（Mem0-style）在治理义务上反而可能比最朴素的"全部塞进上下文"更差（4/15甚至3/15 vs 7/15），因为抽取过程把结构化的权限/来源信息压扁没了。只看长期问答准确率（LoCoMo类基准）的测评完全测不出这类风险。
- **供应商拷问清单**：
  1. "用户要求删除一条记忆后，这条删除请求会传播到摘要、向量嵌入、缓存的 prompt、已经派生出的其他记忆条目吗？你们怎么验证删除是'真删除'而不是'从主存储里删了一行，其他地方还在'？"
  2. "一条记忆被写入时的授权（比如某个用户当时的同意/权限）后来被撤销了，你们的系统会不会还在用这条旧记忆去授权新的动作？有没有'权限纪元（permission epoch）'这类机制在执行动作时重新校验授权是否仍然有效？"
  3. "如果发现某条记忆是错的或被污染的（比如被间接提示注入写入的），你们能不能定位到'这条记忆导致了哪些下游决策'并把它们撤销？这个回滚过程本身会不会因为重放请求而重复触发一次不可逆的外部动作（比如重复扣款）？"

## 关联网络 🕸️
- 相关论文：[[Wiki/论文笔记/09_长上下文记忆/124_MeMo记忆即模型]] — 124号论文探讨"记忆即模型"的参数化记忆路线，与本文第5.1节"parametric state and test-time learning"讨论的张力高度呼应：本文明确指出参数化记忆"efficient to read and hard to govern at write time"——没有原生的 tombstone（删除标记），无法把某次行为归因到具体的某次权重更新并撤销它；这为124号论文的路线提供了一个治理视角的补充警示：把记忆压进权重虽然高效，但天然放弃了 provenance preservation 和 rollback traceability。
- 相关论文：[[Wiki/论文笔记/09_长上下文记忆/163_AutoMem记忆作为认知技能自动学习]] — 163号论文把记忆构建本身当作可学习的认知技能来自动优化，属于本文taxonomy中"reflection and experience learners"（反思与经验学习者）与"skill and workflow induction"（技能/工作流归纳）两个机制家族的交叉；本文对这类机制的治理批评（"validation"环节缺失——一次失败的轨迹可能被写成一条看似可信实则未经验证的持久规则）值得在复核163号笔记时对照检查：163号的自动学习循环是否包含独立于"自我判断"的校验门（validation gate）。
- 相关论文：[[Wiki/论文笔记/09_长上下文记忆/02_Agent记忆系统综述]] — 库内已有的另一篇记忆综述（本笔记撰写时未逐篇核对其具体内容），本文明确把"agent-memory surveys"列为最接近的既有综述类别之一，并声称自己的增量贡献是"把记忆当作持久状态记录的一个操作层面的把手"——即从"记忆存了什么、怎么检索"转向"这条记录凭什么权限影响动作、能否撤销"，建议复核时对照本文第1.2节的"related surveys"表格核实02号笔记的具体覆盖范围。
- 相关概念：[[Wiki/概念/05_记忆与检索/记忆即模型]] — 与上述124号论文的关联同理，本文的"parametric substrate"讨论（第5.1节）是这个概念的治理视角补充。
- 相关概念：[[Wiki/概念/05_记忆与检索/Agent记忆系统四模块]] — 本文的"state substrates"六分类（parametric/context window/vector store/knowledge graph/multimodal/agentic-OS）比"四模块"框架更精细地区分了"记忆存放在哪里"和"记忆以什么形式存在"两个维度，可作为该概念页的扩展视角：本文额外强调不同substrate对六个治理轴的原生支持程度差异极大（向量存储和参数化记忆都不原生支持authority和recoverability，只有agentic-OS/运行时substrate和双时态知识图谱能原生支持全部六轴）。
- **冲突/印证**：本文与"更大上下文窗口/更强检索能解决记忆问题"这一行业默认假设存在直接张力——本文第12.2节专门论证"为什么更大上下文和更好检索不够"：长上下文研究反复证明模型会"lost in the middle"（中间信息利用不足），检索质量的失败模式（噪声鲁棒性、拒绝无关段落）也早有文献刻画，但即便检索做到完美召回，也只回答了"我记得什么"，回答不了"这条记录现在是否还有权限影响动作""它该不该被视为已过时""用户能不能撤销它"——这些是授权、可变性、可恢复性层面的决策，不是检索半径或窗口大小能解决的问题；本文用AOEP pilot的读者规模扫描（3B→8B/更强模型不改善治理义务通过率）提供了直接的实证支持。这构成对"Scale is all you need"式记忆工程思路的有力反证。

## 动手练习 💻
练习目标：用一个可运行的最小示例复现论文核心论断——"文本记忆存储无论用什么检索策略，只要不显式携带治理字段（授权纪元、删除账本），就无法正确处理'删除请求'和'权限撤销后的滞后动作'这两类治理义务"，对比一个"朴素文本存储"和一个"带治理信封的存储"在同一批事件流上的表现。

```python
"""
最小复现实验：模拟论文 AOEP pilot 的核心发现——
朴素文本记忆存储 vs 带治理信封（governance envelope）的存储，
在"删除传播"和"权限撤销后动作校验"两类义务上的通过率差异。
不依赖真实 LLM，用规则模拟检索/写入逻辑，便于本地直接运行。
"""
from dataclasses import dataclass, field

# 1. 定义两种存储：朴素文本存储 只存 value，不存治理信封
@dataclass
class NaiveRecord:
    key: str
    value: str
    deleted: bool = False  # 朴素存储对"删除"的唯一表示：打个标记，但summary/cache不会同步

@dataclass
class GovernedRecord:
    """带治理信封的记录：对应论文 s = <v,a,c,m,p,r,k,τ> 中的关键字段"""
    key: str
    value: str
    authority_epoch: int      # a: 写入时的权限纪元
    tombstoned: bool = False  # r: 是否已被删除（正式墓碑标记）
    derived_keys: list = field(default_factory=list)  # 记录所有派生副本（摘要/缓存等）


class NaiveStore:
    """朴素文本存储：模拟 naive append / 全上下文 / 向量RAG 的共同弱点"""
    def __init__(self):
        self.records: dict[str, NaiveRecord] = {}
        self.summary_cache: dict[str, str] = {}  # 派生摘要，删除时"忘了"同步清理

    def write(self, key, value):
        self.records[key] = NaiveRecord(key, value)
        # 朴素存储会顺手生成一份摘要缓存，但删除逻辑不知道要清理它
        self.summary_cache[key] = f"摘要: {value[:10]}..."

    def delete(self, key):
        # 只标记主记录，派生的 summary_cache 不会被清理 —— deletion propagation 失败的核心
        if key in self.records:
            self.records[key].deleted = True

    def retrieve(self, key):
        rec = self.records.get(key)
        if rec and not rec.deleted:
            return rec.value
        # 关键failure：即使主记录标记删除，摘要缓存仍可被检索到
        return self.summary_cache.get(key)

    def act(self, key, current_epoch):
        # 朴素存储没有 authority_epoch 概念，只要能检索到就直接授权动作
        val = self.retrieve(key)
        return val is not None


class GovernedStore:
    """带治理信封的存储：模拟论文 governed reducer（oracle 上界）"""
    def __init__(self):
        self.records: dict[str, GovernedRecord] = {}

    def write(self, key, value, authority_epoch):
        self.records[key] = GovernedRecord(key, value, authority_epoch)

    def delete(self, key):
        rec = self.records.get(key)
        if rec:
            rec.tombstoned = True
            # 级联清理：把所有派生副本也标记删除（deletion propagation）
            for dk in rec.derived_keys:
                if dk in self.records:
                    self.records[dk].tombstoned = True

    def retrieve(self, key):
        rec = self.records.get(key)
        if rec and not rec.tombstoned:
            return rec.value
        return None  # 墓碑记录 = 真正不可检索，无论是主记录还是派生副本

    def act(self, key, current_epoch):
        rec = self.records.get(key)
        if rec is None or rec.tombstoned:
            return False
        # authority monotonicity 校验：写入时的授权纪元必须仍然有效
        if rec.authority_epoch < current_epoch:
            return False  # 权限已过期（被撤销），拒绝执行动作
        return True


def run_scenario():
    print("=== 场景：写入偏好 -> 权限撤销 -> 尝试用旧偏好授权动作 -> 删除请求 ===\n")

    naive = NaiveStore()
    governed = GovernedStore()

    naive.write("pref_fast_checkout", "允许免确认快速结账")
    governed.write("pref_fast_checkout", "允许免确认快速结账", authority_epoch=1)

    # 权限纪元推进（用户撤销了原授权，比如修改了账户安全设置）
    current_epoch = 2

    print(f"{'检查项':<28}{'朴素存储':<12}{'治理信封存储':<12}")
    naive_ok = naive.act("pref_fast_checkout", current_epoch)
    governed_ok = governed.act("pref_fast_checkout", current_epoch)
    print(f"{'权限撤销后是否仍授权动作':<26}{str(naive_ok):<14}{str(governed_ok):<12}")
    print("  -> 朴素存储没有 authority_epoch 概念，只要记录还在就无脑授权，是典型的",
          "authority monotonicity 违反")

    naive.delete("pref_fast_checkout")
    governed.delete("pref_fast_checkout")
    naive_leak = naive.retrieve("pref_fast_checkout") is not None
    governed_leak = governed.retrieve("pref_fast_checkout") is not None
    print(f"{'删除后主记录仍可检索':<27}{str(naive_leak):<14}{str(governed_leak):<12}")

    # 派生摘要缓存的删除传播检查（论文的 deletion propagation 不变量）
    naive_summary_leak = naive.summary_cache.get("pref_fast_checkout") is not None
    print(f"{'删除后派生摘要缓存仍存在':<26}{str(naive_summary_leak):<14}",
          "N/A（治理存储在delete()中已级联清理derived_keys）")

    print("\n结论：朴素存储在两条核心治理义务上都失败（权限校验缺失 + 删除不传播），")
    print("这正是论文AOEP pilot中naive append/全上下文/向量RAG三者义务分打平在7/15、")
    print("而governed reducer打满15/15的最小化复现。")


if __name__ == "__main__":
    run_scenario()
```

预期输出会显示：朴素存储在"权限撤销后是否仍授权动作"一栏打印 True（错误地继续授权），在"删除后派生摘要缓存仍存在"一栏也是 True（删除没有真正传播）；治理信封存储在两项上都正确返回 False。这直观复现了论文 pilot 里"检索策略不同、义务分数却打平在低位"的现象背后的根因——不是检索能力问题，是压根没有可供校验的治理字段。

## 自测三层 🎓
- **L1 复述（考记忆）**：论文提出的六个状态诊断轴分别是什么？哪一个在435篇编码语料库中覆盖率最低？十阶段生命周期中，"前向弧"和"回程弧"分别包含哪些阶段？
- **L2 解释（"为什么不用别的方案"型）**：论文的 AOEP 评测协议特意把"义务通过率（obligation pass）"和"负向不变量通过率（negative-invariant pass）"分开报告，而不是合成一个总分。如果你是论文作者，为什么不能只用一个总分？这个设计对你日常评估"Agent是否安全可靠"的打分体系有什么启发（提示：想想"什么都不做的系统"在单一总分下会显得多"安全"）？
- **L3 应用（迁移到具体产品场景）**：假设你所在公司要为企业客服 Agent 设计一个跨会话记忆系统，客服 Agent 需要记住客户的历史工单、当前的退款权限、以及客户要求"删除我的所有历史记录"的请求。请结合本文的六轴框架和五条不变量，给出至少三条你会要求工程团队在系统设计阶段就落地的具体治理检查点（例如：权限纪元校验、删除级联传播的验证方式等），并说明如果只做"记忆召回准确率"测试为什么测不出这些风险。

📅 知识时间锚：论文发表于 2026-06-29（arXiv:2606.30306v1），本笔记复核于 2026-07-15。
