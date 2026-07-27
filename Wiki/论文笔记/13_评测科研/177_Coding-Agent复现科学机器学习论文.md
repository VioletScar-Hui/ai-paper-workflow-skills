---
tags: [论文笔记, 评测基准, Coding-Agent, 论文复现, 科学机器学习, harness工程, 验证机制]
paper_id: "177"
filename: "177 - Coding-agents can replicate scientific machine learning papers.pdf"
authors: "Atharva Hans（Purdue University机械工程学院；现就职于 Eli Lilly and Company）, Ilias Bilionis（Purdue University机械工程学院）"
year: 2026
笔记层级: 骨干
成熟度: 🧪
复核日期: 2026-07-15
---

# Coding-agents can replicate scientific machine learning papers（编程Agent能否复现科学机器学习论文）

📄 **原文 PDF**：[[RAW/177 - Coding-agents can replicate scientific machine learning papers.pdf]]

## PM 速判（30秒）
> 一句话：给 coding agent 配一套"目标—证据包—验证门"的持久化工作区（而非只给一句"去复现这篇论文"的指令），就能让它在禁用作者代码、仅凭论文材料的条件下，对 4 篇经典科学机器学习论文完成 12 次独立复现、158 个自设目标全部留痕通过校验——但这套"100% 完成率"衡量的是"证据链条是否留痕完整"，不是"科学结论是否被独立核验为真"。作者自己在讨论部分也承认了这一点，PM 评估任何"AI 自动复现论文"类能力时必须先分清这两件事。

## 双层费曼 🗣️
> **给CEO的一句话**：以前让AI"去复现一篇论文的结果"，AI经常自己说"搞定了"就收工，但没人知道它是真做出来的、还是蒙的、还是抄了论文里现成的图；这篇论文给AI配了一套"留痕记账本"——要复现的每一个结论都必须留下完整证据（跑过什么命令、代码依据论文哪句话、结果和论文差多少）才能打勾，AI自己嘴上说完成不算数，账本状态说了算。
>
> **给工程师的一段话**：Paper-replication 把"复现一篇论文"拆解为若干 target，每个 target 要求候选结果 ŷj=Fj(Dj;θj,ωj) 配齐证据包 Ej=(ŷj,Rj,Pj,Cj,Gj)——运行记录 Rj、把生成物链接到实现文件/配置/种子/论文原文段落的溯源记录 Pj、按 claim 类型（数值/分布/结构/视觉）定制比较规则的对比结果 Cj、以及报告收录记录 Gj；五者同时存在于工作区文件里、且外部校验脚本（哈希查重、溯源一致性检查、完成门 Eq.3）全部通过，target 才能被记为 MATCHED，完成态是工作区文件状态的布尔与运算结果，不是对话里的 agent 自陈。

## 问题域定位 🎯
- **之前卡在哪**：论文复现类研究此前分两条路——一条是"论文转代码"（PaperCoder、ResearchCodeBench、ResearchCodeAgent），只管把论文变成能跑的代码仓库，不检验代码是否真的复现了 claim；另一条是"复现基准"，但要么直接提供作者代码/数据包降低难度（CORE-Bench），要么靠人工事后评分（PaperBench 用 author-informed rubric）。两条路都没解决"agent 在长任务中途会忘记还剩哪些 claim 没做、会把自己的进度描述当证据、会把论文自带插图误当'生成结果'"这类 prompt-only 失败模式（论文引用 Huang et al., 2024 关于 LLM 无法在没有外部反馈时可靠自我纠错的研究，以及 Manheim and Garrabrant, 2018 的 Goodhart's Law 变体分类）。
- **回应什么根本约束**：只给论文材料（LaTeX 源、图表、附录、数据引用），不给作者代码、种子、采样器状态、预处理约定——这比"重跑一个已发布的代码包"难得多，因为缺失的实现细节必须由 agent 自己假设并验证。论文把这类缺失细节明确定义为"待验证的假设"而非"确定值"，这是它与"给代码复现"类基准的根本区别。
- **开启/关闭了哪条技术路线**：开启了"harness engineering"（论文引用 Lopopolo, 2026 提出的术语）在科研复现场景的具体落地——用持久化工作区文件 + 外部校验脚本取代纯 prompt 指令，把"完成"变成脚本可机械判定的工作区状态。但它没有解决"谁来给验收标准兜底"的问题：当论文没写清楚容差时，是执行复现的 agent 自己定规则、自己按规则判自己（§2.3 原话："When the paper does not state such a criterion, the agent records the convention it uses and the reason for using it"）——本质上是把"自由发挥"的环节从"生成结果"挪到了"制定验收标准"，这一点在下面"证据审计"部分会重点展开。

## 核心机制

Paper-replication 的工作流对应原文 Figure 1，按"agent 动作 → 工作区记录 → 外部校验"三层执行，五个阶段串联：

```
论文材料 P：LaTeX 源 + 图表/表格 + 附录 + 数据引用（策略：不允许使用作者代码）
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│ 阶段① 检查论文材料                                             │
│  记录 → source_inventory.json + 论文资产哈希 + 编译渲染页哈希    │
│  校验 → paper-asset checks：生成结果不得与论文资产/渲染页哈希撞车 │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│ 阶段② 记录目标清单 T = {t1, …, tJ}（每个 target = 一条待复现claim）│
│  记录 → reproduction_matrix（每个target的状态）+ task_ledger    │
│  校验 → active-target check：ledger 与 matrix 的"当前活动target"│
│         必须唯一且一致，禁止同时挂多个或零个活动target            │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│ 阶段③ 记录方法重建（把论文方程/算法复述为实现计划）               │
│  记录 → specification files（方程复述、假设、未决细节、验证方式） │
│  校验 → specification check：字段齐全才允许进入实验阶段           │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│ 阶段④ 运行计算实验，产出候选结果 ŷj = Fj(Dj; θj, ωj)             │
│  记录 → run_record（命令/成败/耗时/产物哈希）+ provenance_record │
│         （链接实现文件、配置、种子、论文段落依据）                │
│  校验 → provenance + comparison checks：按target类型强制使用     │
│         数值/分布/结构/视觉对应的比较规则；拒绝"只有可视化证据"    │
│         证明数值 claim；拒绝没有方法溯源的运行结果                │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│ 阶段⑤ 比较并撰写复现报告                                        │
│  记录 → report coverage + replication_report → report/main.pdf │
│  校验 → report-coverage check + 完成门 V_complete（Eq.3）        │
└────────────────────────────────────────────────────────────┘

target 的证据包：Ej = ( ŷj , Rj , Pj , Cj , Gj )
                     候选结果  运行记录 溯源记录 对比结果 报告收录

完成门：V_complete = V_spec ∧ V_progress ∧ V_report
                   ∧ (对全部 j∈{1..J}：sj = MATCHED)
                   ∧ (当前无活动 target) ∧ (report/main.pdf 已生成)

关键设计：完成是"工作区文件状态"的布尔与运算结果，不是agent对话里的
最后一句话——论文原话"the agent's final message is not enough evidence"。
```

## 设计决策解剖 ⚖️

| 决策点 | 他们的选择 | 被放弃的方案 | 为什么 | 这个选择何时失效 |
|---|---|---|---|---|
| 完成判据 | 要求 target 的证据包 Ej=(ŷj,Rj,Pj,Cj,Gj) 全部在工作区文件里成立、且外部校验通过才算 MATCHED（§2.1-2.3） | 相信 agent 自己在对话里宣称"复现完成" | 长任务中 agent 会提前报告完成、把自己的进度描述当证据（§1 引用 Huang et al., 2024 关于 LLM 无法可靠自我纠错的研究）；论文明确写"the agent's final message is not enough evidence" | 外部校验脚本本身检查的是"证据字段是否存在、哈希是否一致、结构是否完整"这类**程序性**条件，不检查"科学判断是否正确"；对结构/分布型target（PIFT全部+SINDy部分），"每个target每次run只有一条记录的比较结果，没有重复独立评判"（§4原话），意味着一次自信但错误的定性判断（如"后验恰好两个峰"）不会被当前校验体系拦下 |
| 数据/代码可用性策略 | 明确禁止使用作者代码，只给LaTeX源、图表、附录、数据引用，逼agent从论文文本重建方法（§2） | 像CORE-Bench那样提供代码/数据包降低难度 | 更贴近"读者手头只有论文本身"的真实场景，且能测试agent能否真正从文字/公式重建方法而非直接跑作者脚本 | 该策略只能防止"复制论文仓库里的资产"（用哈希查重实现），防不住agent凭**预训练记忆**写出的第三方复现代码——PINN两篇论文（Raissi et al., 2017a/b）和SINDy（Brunton et al., 2016）都是2016-2017年的高被引经典论文，网上早已有大量教程/GitHub复现，GPT-5.4 Extra High的训练语料大概率覆盖过这些内容，而"作者代码不可用"这条策略对此完全无效 |
| 数值保真度的复核方式 | 在workspace自设的acceptance rule之外，另外用论文自身准确度量表中**固定不变**的阈值做一次独立"paper-anchored"标量分析（§2.5、§3.2、Table 1） | 只报告workspace的MATCHED/未MATCHED状态作为唯一保真度指标 | 因为acceptance rule在论文未写清容差时由agent自己记录换算规则，容易变成自己给自己定及格线；固定阈值能防住这一点 | 该分析只覆盖13个"标量锚点"（4个PINN-I + 8个PINN-II + 1个SINDy），PIFT完全没有标量锚点、必须靠"一次性、无独立复核"的分布/结构判断（§3.2, §4）；且阈值本身是作者事后选定的"分析惯例"——如PINN-II的10%阈值，论文自己承认"paper reports clean-data coefficient errors below five percent...we use ten percent as an analysis convention"（Table 1脚注），并非论文原文的硬性claim |
| target 的粒度划分方式 | 让agent自由决定如何把一篇论文拆成多少个target，不预设固定rubric（对照PaperBench的author-informed rubric），并把"拆分粒度是否稳定"本身作为研究对象（§2.5, §3.1） | 由人类预先给每篇论文定义一份固定的target清单/checklist | 更符合"仅凭论文材料"的开放式设定，也让论文能够研究"agent如何理解论文的claim边界"这一变量本身 | PIFT三次run的target数是8、8、25，分解比达3.1倍（Fig 3）——一次run把复合图的每个子图拆成独立target，直接让"158个target全部MATCHED"这个总量失去跨run可比性；这说明"target数"和"完成率"在没有人工对齐（论文后续靠"按claim内容分组对齐"的补救措施）之前，都不是稳定可比的审计单位 |

## 成本与量级 💰
- **评测规模**：12 次独立复现 = 4 篇论文（PIFT、PINN-I、PINN-II、SINDy）× 每篇 3 次独立 run；共记录 158 个 target，全部 MATCHED，全部有报告收录，12/12 工作区通过完成门（Table 2、§3）。
- **执行agent**：Codex + GPT-5.4，推理设置为 Extra High（§3 开篇）。
- **本地算力**：MacBook Pro，M4 Max 芯片，128GB 内存（§3）。
- **集群算力**：通过 cluster-execution skill 接入 Purdue Gautschi 集群——CPU 节点 192 核 / 384GB 内存；GPU 节点 8×H100、112 CPU 核、每节点约 1TB 内存（§3）。实测中 PIFT 和 SINDy 全程本地跑完，部分 PINN-I / PINN-II 的 target 提交了集群作业（§3）。
- **续跑机制**：每次 run 配一条固定的、含 10 条 follow-up prompt 的队列，用于在 agent 中途停止时让它重新打开同一工作区、跑状态检查、继续未完成的 target（§2.5）。
- **耗时**：观测到的总体范围 1.2–13.0 小时；四篇论文的后验中位数（95% CI）：PIFT 2.2h [1.1, 4.4]、PINN-I 5.0h [2.5, 9.9]、PINN-II 6.9h [3.0, 13.4]、SINDy 1.9h [1.0, 4.3]（Table 2、§3.3）。两类PINN论文比非PINN论文耗时更长的后验概率在四组两两比较中为 0.947–0.972（§3.3）。
- **纠错成本**：全语料共 25 次"被取代的已跟踪执行"（tracked executions later superseded），PINN-I 11 次、PINN-II 10 次、PIFT 3 次、SINDy 1 次；同一篇论文内部也有巨大波动——PINN-I 的 run1 记录 71 次执行、其中 11 次被取代，而 run2/run3 各自只有 12 次执行且 0 次被取代，相差近 6 倍（§3.3）。
- **统计分析算力**：论文的贝叶斯层级模型全部用 NumPyro + NUTS 采样器拟合，4 条链，每链保留 2000 个后验抽样（§2.5 末）。
- **未披露项（必须明确写出，不能编造）**：论文全文**未披露** token 消耗量、Codex/API 调用的美元成本、GPU-小时汇总数字，也未披露每次 run 的工具调用次数/推理token数这类可比较的agent行为规模指标——只报告了wall-clock耗时和"tracked executions"计数，没有资源成本换算。
- **最小可行配置**：论文未给出显式的"最省资源复现1篇论文"配置声明，但从 Table 2 实测看，SINDy 和 PIFT 在无需集群、仅本地 MacBook Pro 算力下即可在约 2 小时中位数耗时内跑完，是四篇论文里成本最低的配置档位。
- 论文随附代码与数据公开在 GitHub（https://github.com/PredictiveScienceLab/paper-replication-paper），包含 skill 本身、初始/续跑 prompt、全部 12 个agent生成的工作区快照及分析脚本——这是一个值得肯定的透明度实践，理论上使下文"证据审计"里的多数疑点可被第三方独立复核（本笔记未实际下载验证该仓库内容）。

## 证据审计 🔬

- **实验设计公平吗？基线选取有什么猫腻？**
  最大的公平性缺口是论文自己承认的：**没有做"不用 Paper-replication skill、只用裸 prompt"的对照组**。原文讨论部分明确写道："It does not include an ablation without the skill, so it characterizes what Paper-replication produces rather than estimating the workflow's effect on an unstructured prompt."（§4）。这意味着 158/158 的完成率里，有多少归功于工作区+验证门设计、有多少只是因为 GPT-5.4 Extra High + Codex + 最长13小时 + 10次续跑机会本身就足够强，论文完全无法区分。
  第二个值得关注的点：四篇待复现论文里，PIFT（Alberts and Bilionis, 2023）的第二作者正是本文的第二作者 Ilias Bilionis——即评测语料的 25% 由论文作者之一本人撰写，且 PIFT 恰好是四篇论文中 target 数量波动最大的一篇（8/8/25，分解比 3.1×，见 Table 2、Fig 3）。论文没有讨论这一层关联可能带来的选题/评判便利，虽不构成明显的实验作假，但审稿人理应追问。
  第三，四篇选用论文（PINN 两篇 2017 年 arXiv 稿、SINDy 2016 年 PNAS 论文、PIFT 2023 年 JCP 论文）全部是各自子领域的高被引经典工作，网络上早已存在大量第三方复现代码、教程和博客讲解。论文的防污染手段仅限于"用哈希比对生成结果与论文自带资产（图表、渲染页、源文件）是否撞车"（§2.2, §2.3），并明确承认"Hash checks do not rule out every transformed copy or adversarial reuse of paper-provided material by the agent. In these runs, we did not observe such reuse."（§2.2）——这是作者自陈的判断，不是独立检测方法能覆盖的结论，无法排除agent凭预训练记忆写出的、形式上"独立"实则源自记忆的实现。

- **最强证据（PDF 中的真实数字 + 成立条件）**：
  39 个"标量锚点-run"观测中有 37 个（约94.9%）落在作者预先从论文自身准确度量表中固定下来的独立阈值内——这个阈值不依赖 agent 在具体工作区里自设的 acceptance rule（§3.2, Table 1）。成立条件：仅覆盖 13 个标量锚点（PINN-I 4 个解误差 + PINN-II 8 个系数误差 + SINDy 1 个 Lorenz 系数误差），PIFT 完全没有标量锚点、不在此列。
  SINDy 在每一次 run、每一个系统上都精确恢复了稀疏支持（exact sparse support in every run for every system），轨迹图保留了吸引子、分岔、慢流形几何结构（§3.2）——这是四篇论文中结构类证据最稳定的一项。

- **最可疑的数字**：
  1. "158/158 target 全部 MATCHED、12/12 工作区完成"这一headline数字本身——因为 acceptance rule 在论文未写清容差时由 agent 自行记录并自行套用（§2.3），相当于执行复现的一方部分参与了给自己判卷标准的制定；完成率不是不可能为0（agent 可以把 target 标记为unmatched并写明理由），但用它作为"AI能自主复现科学论文"的单一headline证据，掩盖了标准制定环节本身也在agent手里这一事实。
  2. PINN-II 的判断类型一致率仅 5/11 = 0.46（§3.4, Fig 6）——同一篇论文的3次独立run，对近一半的对齐claim连"这算数值target还是结构target"这个最基本的分类都无法达成一致，说明"MATCHED"背后所依据的评判尺子在同一篇论文的不同run之间并不稳定。
  3. PINN-II 里"clean Burgers λ2 系数误差"三次run分别是 7.3%、0.14%、0.014%（§3.2）——最松和最紧相差约500倍，但因为10%阈值足够宽松，三次全部落在阈值内、全部记为达标。10%这个阈值本身是作者事后选定的"analysis convention"（Table 1脚注明确承认），这么宽的容差是否足以掩盖巨大的run间不确定性，是一个值得怀疑的设计选择。

- **审稿人会要求补充**：
  1. 一组"无 skill、纯 prompt"的对照实验，用完全相同的 agent/模型/预算复现同样四篇论文，才能把 workflow 的因果贡献从"模型本身很强+允许长时间反复迭代"中剥离出来——论文自己承认没做（§4）。
  2. 换一批更冷门、最好是模型训练数据截止之后才发表的科学机器学习论文重跑一遍，以检验158/158这类高完成率有多少归因于对经典论文（PINN、SINDy）的预训练记忆污染，而非真正从论文文本独立重建方法。
  3. 对结构/分布型 target（PIFT全部、SINDy部分）补充独立复核机制——论文自己承认"each distributional or structural target has one recorded comparison per run, so repeated independent judging of the same evidence remains future work"（§4），目前"后验恰好两个模态""扩散系数被清晰识别、反应系数未被识别"这类定性判断完全由同一个agent一次性给出并写进报告，没有第二agent或人类复核环节。

- **最小复现实验：代理指标 + 预算**：
  选 2-3 篇冷门或明显发表于模型训练窗口之后的科学机器学习论文（例如 2026 年上半年才挂上 arXiv、尚未被广泛讨论的工作），用同一套 Codex GPT-5.4 Extra High + Paper-replication skill 各跑 3 次独立复现，对比"完成率"和"落入 paper-anchored 固定阈值的标量锚点比例"是否显著低于本文报告的 158/158 与 37/39 的水平。预算：3 篇论文 × 3 次 run × 论文自身观测到的中位耗时约 5 小时/run ≈ 45 agent-小时，参照 PIFT/SINDy 在纯本地 MacBook Pro 算力下即可完成的先例，无需 H100 集群即可执行，是检验"预训练污染 vs 真实复现能力"这一疑点的最低成本实验设计。

## 可复用点（PM决策）

**何时采用**：当产品需求是"给定一篇方法边界清晰、已正式发表的科学计算/科学机器学习论文，需要一个可审计、留痕、能在中途中断后继续跑的复现工作流"时，Paper-replication 的"目标—证据包—验证门"三层设计值得直接借鉴到内部"AI自动复现/自动验证"类工具的 harness 里——尤其是"同一时刻只允许一个活动target + task ledger强制显式记录未完成工作"和"生成产物与论文原始资产强制分离存放、用哈希查重"这两个具体机制，可以移植到任何"防止AI把抄来的东西当自己做的成果"的场景。

**何时规避/谨慎**：不要把这类系统的"100%完成率"直接当作"AI已经能自主做科研复现"的营销素材使用——所有验收标准在论文语焉不详时都由执行复现的同一个 agent 自己制定并自己套用（§2.3），论文明确没做无技能对照组，也没有换一批非高被引/低训练数据污染风险的论文来验证泛化性。如果产品目标是评估"AI能否复现一篇它此前完全没见过的新论文"，这篇论文提供的证据强度还不够，不能直接作为采购依据。

**供应商拷问清单**：
1. "你们的'AI复现论文/AI验证论文'系统里，当论文本身没写清楚容差或评判标准时，这个标准是谁定的——是执行复现的同一个agent自己定，还是有独立于它的裁判环节来定？"
2. "你们敢不敢现场挑几篇最近几个月刚发表、还没被大量博客/GitHub讨论过的论文让我看一次完整复现？而不是只用PINN、SINDy这类几乎人人都写过教程的经典论文来演示成功率。"
3. "你们的'复现成功/MATCHED'指标，是只看'流程性证据齐全'（跑过、有留痕文件、哈希不撞车），还是真的有独立的人或独立的模型核对过科学结论本身是对的？"

## 关联网络 🕸️

**相关论文**：
- [[Wiki/论文笔记/13_评测科研/35_AutoLab超长视野自动研究工程任务评测]] — 印证：AutoLab 发现"成功取决于持续迭代次数，而非初始尝试质量"，Paper-replication 里"固定10条follow-up prompt排队、允许agent中途停下后被重新唤醒继续"的机制，正是这一发现在论文复现场景下的具体工程实现；PINN-II 中位耗时6.9小时、部分target要靠反复纠错（10次被取代的执行）才能达标，也印证了"一次过"式的单轮prompt在这类任务上并不可靠。
- [[Wiki/论文笔记/13_评测科研/129_NanoGPT-Bench自主研究Agent评测]] — 能力边界互补对照：NanoGPT-Bench 测的是"开放式发现"——agent要自己找到比人类现有记录更好的新方法，512 H100 GPU小时预算下最强 agent（Autoresearch）也只恢复了人类5个月研究进展的9.3%；Paper-replication 测的是"给定已知答案去复现"，158/158全部MATCHED。二者合起来划出当前AI科研Agent的能力边界：复现已发表的既定路径接近100%完成率，独立发现未知的新路径不到10%——这是两个性质完全不同的任务，不能互相佐证"AI科研能力已经很强"。
- [[Wiki/论文笔记/03_Agent系统设计/43_AutoScientists自组织Agent团队科学发现]] — 路线对比：AutoScientists 走的是多Agent自组织分工做"从0到1"科学发现的路线；Paper-replication 走的是单Agent+持久化工作区做"从已发表结果反推复现"的路线，二者是"AI科研Agent"问题域下完全不同的两个子任务（发现 vs 复现），不应被同一句"AI能做科研了"的宣传语言混为一谈。
- [[Wiki/论文笔记/13_评测科研/10_NatureBench科研Agent评测]] — 轻度参照：据本库 AI科研Agent 概念笔记汇总，NatureBench 测自然科学实验设计能力，最佳AI约42%正确率，对比人类专家约91%；这类"设计新实验"的任务更接近"发现"端，与Paper-replication"复现给定实验"的任务性质不同，可作为同一能力谱系上的另一参照点。
- [[Wiki/论文笔记/13_评测科研/161_自动化科研论文审稿-Google论文助手]] — 轻度参照：该论文关注AI在"审稿"环节的角色定位（四级角色分类法：作者工具/评审工具/支持性评审/全自动评审），与本文关注的"复现"环节同属AI介入科研论文生命周期的不同阶段，二者可共同构成"AI×论文全生命周期"地图上的两个点（发表前把关 vs 发表后复现验证）。
- [[Wiki/论文笔记/06_训练对齐RL/160_验证边界-Coding-Agent奖励设计无银弹]] — **冲突/张力**：160号论文的核心论断是"可扩展性、忠实性、抗攻击鲁棒性三者不可兼得，验证器必须独立于被验证的策略模型并与之共同进化，否则一旦策略能力追上验证器就会被hack"（该论文实测：未加独立行为监控时，SWE任务的hacked-resolved率高达28.57%）。Paper-replication 的验证层虽然引入了工作区文件和外部校验脚本，但校验的主要是"证据字段是否存在、哈希是否一致、流程是否完整"这类程序性条件，而语义层面"acceptance rule该定多严"这件事在论文未写清容差时仍由**同一个**执行复现的agent自行记录并自行套用（§2.3）——这正是160号论文警示的"验证器与生成者同源"风险的一个具体案例：Paper-replication 大幅收紧了"AI自称完成"的下限，但没有从根本上解决"谁来独立于AI给AI的产出打分"这个更深的问题。

**相关概念**：
- [[Wiki/概念/04_Agent框架/AI科研Agent]] — 上位概念，提供 Level 1–5 能力层级框架。该概念笔记原文明确警示："如果任务是'把已经发表过的论文方法在新数据集上跑一遍、看能不能复现'，这属于 Level 1/2 范畴，当前系统能可靠完成，不应该被算作'AI 已经能做原创科研发现'的证据——很多宣传里混淆了'复现已知方法'和'提出新发现'这两件事。" Paper-replication 158/158 的完成率精确落在这条边界划定的 Level 1/2（验证/补全）范畴内，读它的结论时必须绷住这根弦，不能因为完成率高就误判为AI已具备Level 4/5（猜想/发现）能力。

## 动手练习 💻

练习目标：用一个可运行的最小示例，复现 Paper-replication 最核心也最值得警惕的机制——"工作区自设验收规则判定的 MATCHED"和"论文口径固定阈值的独立复核"是两把不同的尺子，一个 target 可能通过前者却过不了后者（对应论文 §3.2 中 2/39 个标量锚点的真实情况）。

```python
"""
最小复现实验：模拟 Paper-replication 的"目标-证据-验证"评分流水线（对应论文 §2.1-2.3, §3.2）。
核心目的：直观展示论文最关键的审计发现——
一个 target 在"工作区自设验收规则"下被判为 MATCHED，
未必能通过"论文口径固定阈值"的独立复核。
不依赖真实 LLM，用随机噪声模拟"agent复现结果"与"论文原始数值"之间的差异。
"""
import random

random.seed(6)

# 1. 模拟四个待复现目标（对应论文的 target），每个目标含：
#    paper_value    —— 论文报告的真值 y*
#    kind           —— 目标类型：numeric（数值型）/ structural（结构型）
#    workspace_tol  —— agent 在工作区里自己记录的验收容差（对应论文 §2.3："当论文未写清
#                       容差时，agent 记录自己使用的换算惯例"，这里故意设得比 paper_tol 宽松）
#    paper_tol      —— 论文口径下固定不变的阈值（对应 §3.2 的 paper-anchored threshold，
#                       独立于 workspace_tol，只在事后复核时使用）
#    has_provenance —— 该输出是否链接到"方法重建+可执行run"（对应 Pj 溯源记录；
#                       False 模拟"复制了论文自带资产"或"未记录来源"，必须一票否决）
targets = [
    {"id": "PINN-I solution error", "kind": "numeric",
     "paper_value": 0.008, "workspace_tol": 0.03, "paper_tol": 1e-2, "has_provenance": True},
    {"id": "PINN-II coeff error", "kind": "numeric",
     "paper_value": 0.03, "workspace_tol": 0.10, "paper_tol": 0.10, "has_provenance": True},
    {"id": "SINDy sparse support", "kind": "structural",
     "paper_value": {"x", "y", "xy"}, "workspace_tol": None, "paper_tol": None, "has_provenance": True},
    {"id": "未溯源的复制图", "kind": "numeric",
     "paper_value": 0.005, "workspace_tol": 0.03, "paper_tol": 1e-2, "has_provenance": False},
]


def agent_reproduce(target):
    """模拟 agent 的候选结果 ŷ = F(D; θ, ω)：在真值基础上加随机扰动，
    代表随机种子/超参/实现细节缺失导致的 run-to-run 波动（对应论文 §3.2 观察到的现象，
    如 clean Burgers λ2 系数误差三次 run 分别是 7.3%、0.14%、0.014%）。"""
    if target["kind"] == "numeric":
        noise_scale = random.choice([0.5, 2.0, 4.0])  # 模拟论文里观察到的"数量级波动"
        return target["paper_value"] * random.uniform(1 / noise_scale, noise_scale)
    else:
        # 结构型目标：多数情况下精确复现稀疏支持，小概率漏掉一个非零项
        support = set(target["paper_value"])
        if random.random() < 0.15:
            support.discard(random.choice(list(support)))
        return support


def workspace_accept(target, candidate):
    """工作区自设验收规则（对应论文 Cj + acceptance rule，§2.3）：
    数值目标用 agent 自己记录的容差 workspace_tol 判定，可能比论文口径更宽松；
    结构目标用集合相等判定。"""
    if not target["has_provenance"]:
        return False, "无溯源记录，一票否决（对应论文的 provenance 检查）"
    if target["kind"] == "numeric":
        ok = abs(candidate - target["paper_value"]) <= target["workspace_tol"]
        return ok, f"|差|={abs(candidate - target['paper_value']):.4f} vs 自设容差={target['workspace_tol']}"
    else:
        ok = candidate == target["paper_value"]
        return ok, f"候选支持={candidate} vs 论文支持={target['paper_value']}"


def paper_anchored_audit(target, candidate):
    """独立于工作区判定的"论文锚定"复核（对应论文 §3.2 的第二层分析）：
    只对数值型目标生效，使用固定不变的论文口径阈值 paper_tol，不采信 workspace_tol。"""
    if target["kind"] != "numeric":
        return None  # 结构/分布型目标不纳入标量复核，论文本身也是这样处理的
    discrepancy = abs(candidate - target["paper_value"])
    return discrepancy <= target["paper_tol"]


# 2. 跑一遍流水线，统计"工作区完成率"与"独立锚定复核通过率"之间的差异
matched_count, audit_pass_count, audit_checked = 0, 0, 0
print(f"{'目标':<24}{'workspace判定':<10}{'原因':<42}{'锚定复核':<8}")
for t in targets:
    y_hat = agent_reproduce(t)
    matched, reason = workspace_accept(t, y_hat)
    audit = paper_anchored_audit(t, y_hat) if matched else None
    matched_count += int(matched)
    if audit is not None:
        audit_checked += 1
        audit_pass_count += int(audit)
    print(f"{t['id']:<24}{str(matched):<10}{reason:<42}{str(audit):<8}")

# 3. 完成门（对应论文 Eq.3）：仅当全部目标都被记为 MATCHED，才算论文级完成
completion_gate = matched_count == len(targets)
print(f"\n完成门通过（全部MATCHED）：{completion_gate}（{matched_count}/{len(targets)}）")
print(f"其中数值目标通过独立锚定复核：{audit_pass_count}/{audit_checked}")
print("结论：'workspace_tol判定通过' 不等于 '通过论文口径的paper_tol复核'——")
print("workspace判定和论文锚定复核是两把不同的尺子，")
print("论文报告的158/158 MATCHED完成率，衡量的是前者，不能不加说明地等同于后者。")
```

实测输出（seed=6）：`PINN-I solution error` 一项 workspace 判定为 `True`（候选值落在 agent 自设的 0.03 容差内），但 `paper_anchored_audit` 判定为 `False`（同一候选值超出论文口径的 0.01 固定阈值）——这就是最小规模上复刻出的论文 §3.2 真实情况：Schrödinger run3、Navier–Stokes λ2 run1 两个标量锚点在各自工作区里都记为 MATCHED，却双双落在 paper-anchored 阈值之外。`未溯源的复制图`按预期被 provenance 检查一票否决，`SINDy sparse support`本次因随机丢失一个非零项而未通过结构比对，完成门整体判定为 `False`（2/4），直观说明"target数全部MATCHED"和"经得起独立复核"是两回事。

## 自测三层 🎓

- **L1 复述（考记忆）**：Paper-replication 要求每个 target 的证据包 Ej 包含哪五个组成部分？完成门 V_complete（Eq.3）的六个必要条件分别是什么？
- **L2 解释（"为什么不用别的方案"型）**：论文为什么不直接把"agent 在对话里说'我完成了'"当作完成判据，而要单独搭一套工作区文件 + 外部校验脚本？结合论文引用的"LLM 无法在没有外部反馈时可靠自我纠错"（Huang et al., 2024）和 Goodhart's Law 变体分类（Manheim and Garrabrant, 2018），说明这个设计选择具体想防住的失败模式是什么，以及为什么"验证检查本身仍由同一个agent触发和部分定义"没有彻底解决这个问题。
- **L3 应用（迁移到具体产品场景）**：如果你所在公司要做一个"用AI自动核验竞品白皮书里的benchmark数字是否可信"的内部工具，请借鉴 Paper-replication 的 target-evidence-validation 三层设计，给出至少3个"必须留痕的证据字段"和1条"完成门"判定规则，并说明如何借鉴 Ej 中 Pj（溯源）和 Cj（比较）这两项，防止"AI直接采信白皮书里的图表当作自己核验过的证据"这一失败模式。

📅 知识时间锚：论文首次公开于 arXiv:2607.02134（2026-07-02 提交，v2 于 2026-07-10 更新）；四篇待复现论文分别发表于 2016 年（SINDy，PNAS）、2017 年（PINN-I/II，arXiv）、2023 年（PIFT，Journal of Computational Physics）· 本笔记复核于 2026-07-15。
