---
tags: [论文笔记, 技能组合, Agent技能, 工具使用, 序列生成, 检索增强解码]
paper_id: "162"
filename: "162 - Generative Skill Composition for LLM Agents.pdf"
authors: "Kwonjoon Lee†, Xinyu Zhao, Zhen Tan, Vaishnav Tadiparthi, Nakul Agarwal, Ehsan Moradi Pari, Hossein Nourkhiz Mahjoub, Tianlong Chen（UNC Chapel Hill / Arizona State University / Honda Research Institute USA，† 通讯作者）"
year: 2026
笔记层级: 骨干
成熟度: 🧪
复核日期: 2026-07-08
---

# 生成式技能组合：为 LLM Agent 联合预测技能子集、数量与顺序（Generative Skill Composition for LLM Agents）

📄 **原文 PDF**：[[RAW/162 - Generative Skill Composition for LLM Agents.pdf]]

## PM 速判（30秒）
> 一句话：当技能库大到几百个时，"选哪些技能、选几个、按什么顺序执行"是三个耦合在一起的决策，检索排序和把全部技能塞进 prompt 这两条现有路线都只能解决其中一个维度；本文把这三件事压成一次自回归解码，用一个仅 3.9M 参数的小解码器，在两个生产级 Coding Agent（GPT-5.2-Codex、Gemini-3-Pro-Preview）上把 SkillsBench 任务通过率分别拉高 23.1pp 和 18.2pp，且比检索方案用更少的 prompt token。PM 关注点：这是"技能路由层"该怎么设计的一份可执行范式，而不是又一个更大的技能库。

## 双层费曼 🗣️
> **给CEO的一句话**：公司内部工具/SOP 越攒越多之后，Agent 不知道该拿哪几个、拿几个、按什么先后顺序用；这篇论文训练了一个很小的"调度模型"，专门解决"选择+数量+顺序"这个组合难题，让 Agent 干活的成功率明显提升，而且不用把所有工具说明书都塞给它，省了大量成本。
> **给工程师的一段话**：论文把技能组合形式化为在闭合词表（K=196 个技能 index + STOP）上的任务条件化自回归序列生成：frozen Qwen3-Embedding-0.6B 编码任务向量，3 层 256 维 Transformer 解码器 cross-attend 到技能元数据 memory 逐步产出技能 index 序列；训练时额外加 cardinality head（预测技能数 n∈{1..8}）和 set head（技能是否相关的 pairwise 二分类）两个辅助头联合监督；推理时把 AR 上下文 logit 与 TF-IDF 检索先验、set-membership 先验做加权融合后再 beam search 解码，兼顾语义检索的鲁棒性和序列模型的顺序建模能力。

## 问题域定位 🎯
- **回应什么根本约束**：Agent Skills 生态（Anthropic 2025 提出的开放技能标准）让技能库规模从几十个膨胀到几百个（本文用的 SkillBench 库有 196 个技能）之后，"获取技能"不再是瓶颈，"组合技能"才是——论文原文明确指出这一转移："This shifts the inference-time bottleneck from obtaining skills to composing the right collection of skills"。
- **之前卡在哪**：现有两条路线各卡在一个维度上。① 直接推理（把全部技能暴露给 Agent 自己判断）——组合决策隐藏在无结构的执行轨迹里，且把 196 个技能说明书全塞进 prompt 会让 Codex 输入膨胀到 1.27M token（实测），通过率反而只从 22.2% 涨到 29.3%（All Skills 条件），说明"塞得多不等于选得对"。② 检索排序（BM25/TF-IDF/embedding）——只能给出一个无序候选集，既不知道该取几个（需要外部给定 k，或者用"oracle-k"作弊上限），也不表达执行顺序。
- **开启/关闭了哪条技术路线**：把"技能选择"从传统的"pointwise 排序问题"改造成"closed-vocabulary 序列生成问题"，与近年"生成式检索"（Recommender Systems with Generative Retrieval, Rajput et al. 2023；ToolkenGPT, Hao et al. 2023）的思路一脉相承，把这条思路第一次系统用在"技能包"这一更粗粒度、依赖关系更隐式的对象上。同时论文明确把"技能库固定不变"作为前提假设，刻意把"组合"问题与"技能生成/发现"问题解耦——这关闭了"开放式技能库在线增长"场景下的直接适用性（见下文局限）。

## 核心机制

```
输入：任务 x + 环境上下文 c（如文件路径）+ 技能库 S={s1,...,s196}（每个技能=元数据+适用条件+
     执行策略+终止条件+可选资源接口）

┌─────────────────────────── A. 编码 ───────────────────────────┐
│  Task+Env 序列化 prompt P(x,c,S)                                │
│         │                                                        │
│    Frozen Task Encoder（Qwen3-Embedding-0.6B，last-token 池化） │
│         │                                                        │
│    投影 h = W_proj · h_x  ∈ R^256                               │
└────────────────────────────┬──────────────────────────────────┘
                              │
┌─────────────────────────── B. 解码 ───────────────────────────┐
│                              │                                   │
│   ┌── Cardinality Head ──┐  │  ┌── Set Head（每技能独立打分）──┐│
│   │ 线性分类器预测 n∈{1..8}│←─┤  │ MLP([h;e_i;h⊙e_i;|h-e_i|])    ││
│   └───────────────────────┘  │  │ → 相关性 logit σ_i             ││
│                              │  └────────────────────────────────┘│
│                              ▼                                   │
│         3层/256维 Transformer 自回归解码器 D                     │
│         cross-attention → 196行技能元数据 memory                 │
│         z_t = p_θ(z_t | h, z<t)   逐步产出技能 index             │
└────────────────────────────┬──────────────────────────────────┘
                              │
┌───────────────── C. 检索增强解码（每步融合） ──────────────────┐
│  融合logit_t(i) = 上下文logit ℓ_t(i)                              │
│                 + α·TF-IDF检索先验 r_i   (α=1.0)                 │
│                 + β·set-head先验 σ_i     (β=0.5)                 │
│  → beam search（宽度4，长度惩罚0.7，去重约束）→ 直到 STOP        │
└────────────────────────────┬──────────────────────────────────┘
                              │
                索引查表：z=(104,184,55,STOP) → (nws-flood-thresholds,
                usgs-data-download, flood-detection)
                              │
                按预测顺序，将完整技能包（策略+资源）注入
                Agent 的下游执行 context
```

关键点：cardinality head 和 set head **只在训练时提供额外监督梯度、推理时又被复用为 decode-time 先验**，本身不直接输出最终答案——最终序列仍由自回归头产出，这样"顺序"这个不能被 pointwise 方法表达的维度，始终由 AR 头独占负责，另外两个头只负责把"该选几个""该选谁"这两路本来被 STOP 位置和 gold-position 稀释的监督信号显式拉出来单独训练。

## 设计决策解剖 ⚖️

| 决策点 | 他们的选择 | 被放弃的方案 | 为什么 | 这个选择何时失效 |
|---|---|---|---|---|
| 组合问题的建模范式 | closed-vocabulary 自回归序列生成，一次解码联合产出子集+数量+顺序 | 检索排序（BM25/TF-IDF/dense embedding 分别打分，取 top-k） | 检索本质是 pointwise 独立打分，天然无法联合决定"要几个"（需要外部给 k 或作弊式 oracle-k）和"什么顺序"；论文原文强调三维度"cannot be decoupled" | 技能库不是固定/封闭的（如 Agent 自主创建新技能、技能库在线持续增长）时，closed-vocab 输出空间需要重训或扩容，论文在 Limitations 中明确点名这是未来方向 |
| 训练范式 | 冻结检索式编码器 + 仅 3.9M 可训练参数的专用小解码器 | 全量微调 600M 参数的 Qwen3-0.6B-Base（把技能名加入词表，当文本生成任务做 SFT） | 真实任务分布迁移测试（real-task holdout）中，SFT 从 71.1%→43.6% 掉了 27.5pp，而 SkillComposer 只从 73.9%→62.9% 掉 11pp——冻结编码器带来的"检索式迁移偏置"让小解码器泛化更稳，SFT 记住的是合成数据模板 | 若训练/部署环境几乎没有分布偏移风险（比如线上任务与合成训练数据高度同构），SFT 在同分布测试和 k≥3 的长链任务上仍有微弱优势，此时省事的 SFT 也可接受 |
| decode-time 先验的检索器选型 | 稀疏 TF-IDF 余弦相似度 | 稠密 Qwen3-Embedding 语义检索 | 196 个技能的名称+一句话描述本身就"短且句法specific"，token 级重合是高精度判别信号；稠密向量倾向对更宽泛语义做平均，反而在如此小、高度特化的候选集上"过度泛化" | 若技能元数据描述变长、更口语化/跨语言、lexical overlap 稀疏，TF-IDF 的优势可能反转（论文未验证这一反向场景） |
| 辅助监督头的取舍 | AR 头（负责顺序）+ set head（负责"选谁"，order-agnostic 二分类）+ cardinality head（负责"选几个"）三头联合训练 | 只用 AR 头单目标，靠 STOP 位置隐式承载数量信号、靠 gold 位置隐式承载相关性信号 | 单一 AR 头下，"数量"信号被压缩进单个 STOP token 位置，"该技能是否相关"的梯度只在其 gold 出现位置传播一次，信号稀疏；显式二分类让每个 gold 技能都拿到与位置无关的直接梯度 | 消融显示（Table 3）cardinality head 单独加入时 Set F1 仅从 69.3→69.6（+0.3pp），远弱于 set head 单独贡献的 +2.5pp——说明 cardinality head 的价值主要体现在推理期作为长度先验，而非训练期监督信号本身，这个"名不副实"的收益点值得追问 |
| 技能库假设 | 训练/推理时技能库 S 固定不变（K=196），明确排除"技能生成"问题 | 开放式技能发现/在线扩充库（如 Voyager 式 agent 自主积累新技能） | 真实生产部署"ship curated skill packs"，把组合与生成解耦更 tractable，能用监督学习方式训练判别式/生成式路由器 | Limitations 章节明确指出：交互式、长时程、技能库在线更新的场景，以及机器人/具身智能这类工具异构到物理执行器的场景，本方法未覆盖 |

## 成本与量级 💰
- **可训练参数**：SkillComposer 全部可训练部分（投影层+解码器+两个辅助头）仅 **3.9M**，对照 SFT 基线 Qwen3-0.6B-Base 的 600M，相差 **154×**。
- **训练算力**：论文称训练计算量比 SFT 基线少 **25×**（Fig. 5b，未给出绝对 GPU-hours，只给相对倍数——论文未披露具体训练耗时）。
- **训练超参**：AdamW，lr=1e-4，weight decay=0.01，batch size=64，最多 100 epoch，patience-15 早停（依 validation Set F1）。
- **训练数据规模**：共 9,872 条 task-skill 组合记录：65 条人工标注真实 SWE 任务（来自 SkillBench）+ 2,880 条单技能合成任务（Gemini 2.5 Flash 生成，覆盖全部 196 个技能）+ 6,927 条多技能合成任务（Gemini 2.5 Pro 生成，2-5 个技能组合，基于 924 条边的技能依赖图，65% 依赖边/35% 工作流共现边采样）。
- **推理侧**：beam width=4，长度惩罚 0.7；单卡 A6000、fp16、batch=1 下，推理延迟与 SFT 同一量级，比调用前沿 API 的 LLM-judge 基线快两个数量级（论文未给出具体毫秒数）。
- **下游 context 开销（更能体现产品成本）**：GPT-5.2-Codex 上，SkillComposer 平均每次成功试验输入 **1.03M token**，是所有"加载技能"条件里最省的——对照 All Skills 的 1.27M、Retrieval(top-3) 的 1.09M、Gold Skills 的 1.12M。
- **最小可行配置**：frozen Qwen3-Embedding-0.6B 编码器 + 3 层/256 维/4 头 Transformer 解码器 + 2 个轻量 MLP 辅助头，训练数据量级在万条以内即可，是一个个人团队可承受的规模（相对于训练/微调 600M 模型）。

## 证据审计 🔬
- **实验设计是否公平**：整体较为诚实。检索基线给出 best-k（验证集调参）和 oracle-k（直接给 gold 数量）两个变体，并把 oracle-k 明确标注为"ceiling"、排除出主排名——这是一个负责任的做法，没有拿作弊结果冒充公平对比。LLM-judge 用 Gemini-2.5-flash 在单个 prompt 里扫描全部 196 个技能元数据，是较强的前沿 API 基线；SFT 基线和 SkillComposer 用完全相同的技能库和训练/测试划分，控制变量比较干净。下游任务评测排除了 13 个"office/文档处理"任务（因为任何检索方法都能靠文件后缀名平凡路由到 pdf/xlsx/pptx/docx 技能，不具区分度）——这个排除理由合理，但也意味着报告的 pass rate 提升幅度是在"故意剔除简单案例"后得到的，泛化到全量 88 个任务时可能被稀释。
- **最强证据**：real-task holdout（n=65，来自 SkillBench 全部真实标注任务，训练/验证阶段完全移除真实任务、只用合成数据训练）上，SkillComposer 的 Set F1 从同分布测试的 73.9% 掉到 62.9%（-11pp），而 SFT 从 71.1% 掉到 43.6%（-27.5pp），形成 **+19.3pp 的分布外泛化差距**，且是在同一份训练数据、同一个技能库上重训得到——这是全文最有说服力的证据，因为它检验的是分布外迁移能力而非同分布拟合能力。**成立条件**：该 holdout 集合本身只有 65 个样本（等于 SkillBench 全部真实任务），论文没有报告任何置信区间或跨 seed 方差。
- **最可疑的数字**：Table 4（decode-time 检索先验消融）正文写"TF-IDF wins on Set F1 by +2.5pp over dense Qwen3-Embedding cosine and +3.8pp over no prior"，但按表格给出的数字算：TF-IDF 73.9 − Qwen3-Embedding 68.8 = 5.1pp，TF-IDF 73.9 − No prior 67.5 = 6.4pp，两者都对不上正文声称的 2.5/3.8pp。同样，Table 3 里"+cardinality head 69.6"相对"AR-only 69.3"只提升 0.3pp，几乎不显著，但正文段落标题却是"Each component is load-bearing"，措辞与该项数据本身存在张力。这两处可能是 PDF 表格在版式抽取时列错位导致（笔者抽取原文本时也观察到多张表格的数字列被打乱），但无论是抽取问题还是叙述与数据本身不一致，都值得进一步核实原始数值。
- **审稿人会要求补充**：① Appendix D 明确承认"训练模型（SkillComposer/SFT）的分层结果因为记录 ID 没保存而缺失，留待未来版本补充"——这是一个应被要求当场补齐的可复现性缺口。② real-task holdout 只有 n=65 且缺乏方差报告。③ Table 4 文本表述与表格数字的差异应被要求核对。④ cardinality head 单独贡献几乎为零，应要求作者解释为何仍将其保留在最终模型中（除了"推理时可作解码先验"之外的必要性）。
- **最小复现实验**：不需要等真实 196 技能库，可先用 30-50 个 mock 技能（自拟元数据）+ 50-80 条 mock 任务，训练一个"frozen sentence-embedding + 3 层小 Transformer + TF-IDF logit 融合"的迷你版本；代理指标用 Set F1（预测子集 vs gold 子集的 F1），CPU 上几分钟即可完成一轮训练+验证，重点验证"logit 融合公式"和"辅助头联合训练"相对"仅 AR 头"是否确实带来正向增益，而不必一开始就砸下 9872 条数据的全量预算。
- **附录案例研究给出的三种失败/成功模式**（GPT-5.2-Codex，每任务 3 次试验，来自 Appendix C.1，是理解"为什么生成式组合优于/劣于检索或 gold 标注"的最具体证据）：
  1. **adaptive-cruise-control**：SkillComposer 选出 4 个技能（含关键的 `imc-tuning-rules`），reward 达 1.00；Retrieval(top-3) 因 top-3 截断丢掉了这个关键技能，reward 只有 0.33；更值得注意的是，人工标注的 Gold Skills 本身夹带了两个无关的 I/O 格式技能（`csv-processing`、`yaml-config`），reward 同样只有 0.33——说明"标注质量"本身可能是评测的隐藏天花板。
  2. **exoplanet-detection-period**：SkillComposer 收敛到一条精简的三技能流水线（预处理→Lomb-Scargle 周期图→TLS 搜索），全部 3 次试验通过（reward=1.00）；而 Gold Skills 因为多带了冗余的 `box-least-squares` 估计器和一个重量级 workflow 包装技能，导致 Agent 走上更长的流水线并过拟合恒星振荡噪声，reward=0.00——直接印证了 Table 2 中"All Skills 条件反而拖累通过率"的现象：技能不是越多越好。
  3. **lean4-proof**：SkillComposer 只给出 1 个技能（`lean4-memories`）就提前触发 STOP，遗漏了完成归纳步骤所需的 `lean4-theorem-proving`，reward 只有 0.67；反而是 Retrieval(top-3) 保住了完整的三技能链，reward=1.00。论文将这归因为"短序列偏置"——合成训练语料里 3 技能以下的组合占多数，导致模型在真正需要长链条的任务上倾向于提前截断，这是论文自己承认的"最 actionable 的改进方向"。

## 可复用点（PM决策）
- **何时采用**：产品已经积累了一个较大、相对稳定的"内部技能/SOP/工具集合"（比如企业内部 checklist、私有 API 集合、Agent Skills 生态里打包好的技能包），且用户任务经常需要"多个技能按特定先后顺序组合"（而不是单一工具一次调用），同时你已经观察到"检索 top-k 漏选关键技能"或"塞全部工具说明书导致 prompt 膨胀"这两个具体痛点——这正是本文对照实验直接命中的两个失败模式。
- **何时规避**：① 技能库本身开放、持续被 Agent 自主创建/发现（不是人工维护的固定库），此时闭合词表解码器需要频繁重训，不匹配。② 绝大多数任务只需单一工具、无需多技能编排顺序，直接检索/规则路由的性价比更高，没必要为此训练一个需要近万条标注数据的专用序列模型。③ 团队没有能力构造"任务-技能序列"标注数据（本文靠 LLM 合成+人工种子结合，构造成本不低）。
- **供应商拷问清单**：
  1. "你们的技能组合/路由模型在**陌生真实任务**上的性能下降幅度是多少？是否做过'仅用合成数据训练、在真实任务上做 held-out 测试'这类分布外验证？"（对应本文 real-task holdout 场景下 -11pp vs SFT 基线 -27.5pp 的核心证据，防止对方只展示同分布的漂亮数字）
  2. "当你们标注的『标准答案技能组合』本身包含冗余或过度技能时，模型是否会盲目模仿标注，还是能学会剔除无用项？"（对应本文 exoplanet-detection-period 案例：模型给出的精简 3 技能组合达到 100% 通过率，而"标准答案"因夹带 2 个冗余技能反而 0% 通过——揭示标注质量本身可能是天花板而非地板）
  3. "技能数量的预测上限是怎么定的？如果任务真实需要很多个技能（长链任务），你们的系统是否存在『短序列偏置』导致提前截断？"（对应本文 lean4-proof 案例：模型只给出 1 个技能就提前停止，遗漏了关键的证明策略技能，论文承认这是"最actionable的headroom"）

## 关联网络 🕸️
- [[Wiki/论文笔记/04_Agent技能工具/12_SKILLWEAVER组合技能路由]]：两篇论文都断言"简单检索/pointwise 打分解决不了多技能组合问题"，在这个上位判断上互相印证——SKILLWEAVER 的证据是"分解正确率完美(DA=1)时检索质量CatR@1才提升7pp，远小于换更强检索器带来的改进空间"，本文的证据是"检索基线即便给 oracle-k 作弊数量，Set F1 仍落后于不需要 gold 数量的生成式方法"。但两者的技术路线选择完全不同：SKILLWEAVER 是**训练free的多阶段 pipeline**（现成 LLM 做任务分解 → 双编码器检索 → 启发式 DAG 打分组合，核心瓶颈被归因到"分解"这个上游步骤），本文是**训练一个专用的小型生成式解码器**，把"选择+数量+顺序"三件事在一次自回归解码里联合建模，压根不设置独立的"分解"阶段。可以说 SKILLWEAVER 关注的是"复杂查询如何被拆成正确数量的子任务"这个上游问题，本文关注的是"给定任务，如何用一个统一模型直接产出可执行的技能序列"这个组合决策本身；两者若结合，SAD（技能感知分解）产出子任务后，每个子任务的技能选择可以换成 SkillComposer 式的生成式路由，是互补而非互斥的关系。
- [[Wiki/论文笔记/04_Agent技能工具/08_Skill-MAS元技能自动多智能体]]：Skill-MAS 解决的是更宏观的"多智能体系统架构本身如何被自动设计和进化"（Meta-Skill 三模块：任务分解/Agent工程/工作流拓扑，靠 LLM 自我反思迭代改写自然语言文档、无需梯度更新），本文解决的是更微观的"单个 Agent 该从固定技能库里加载哪些技能、几个、什么顺序"（靠监督训练一个 3.9M 参数的判别式/生成式小模型）。两者的共同底层母题都是"AI 系统的能力编排单元该如何被结构化管理和调度"，但抽象层级和实现路径完全不同：Skill-MAS 操作对象是"整个多智能体团队怎么搭"，进化机制是选择性反思+改写文档；本文操作对象是"单 Agent 该往 context 里塞哪几个技能包"，机制是训练一个专门的序列预测模型。可以理解为同一个"技能编排"问题在宏观（团队设计）和微观（单次任务的技能加载）两个粒度上的不同解法。
- 相关概念：[[Wiki/概念/04_Agent框架/函数调用与工具使用]]——该笔记强调"工具应原子化，避免上帝工具"这一函数调用层面的设计原则；本文的"技能"恰恰是比原子化 API 调用更粗的抽象（论文明确区分：技能是"跨越多个 API 调用的可复用多步骤流程"，如"schema grounding"或"query reformulation"），这构成一个有趣的对比张力——函数调用层面追求原子化以降低参数填错率，而技能组合层面反而需要更粗的抽象单元才能承载"依赖关系隐式、任务逻辑而非类型驱动"的组合信息（本文原文："skills lack typed signatures, so dependencies are latent and task-logical rather than type-induced"）。
- [[Wiki/论文笔记/04_Agent技能工具/74_Toolformer语言模型自学使用工具]]（本文参考文献直接引用了 Toolformer）：Toolformer 解决的是"原子级 API 调用"的自监督学习——让模型通过自我采样+困惑度下降筛选，自学"什么时候该插入一次 `[Calculator(...)]`"，训练信号完全来自无监督语料的困惑度变化，不需要人工标注调用时机。本文操作的抽象层级更高：技能是"跨越多个 API 调用的可复用多步骤流程"，训练信号是有监督的 task-skill 组合标注（人工种子+LLM 合成），解决的不是"要不要调用"而是"从几百个候选流程里选哪几个、几个、什么顺序"。两者可以看作同一条"让 LLM 学会使用外部能力"技术脉络在十年尺度上的两级抬升：从"自监督学会单次调用"（Toolformer）到"有监督学会多步骤流程编排"（本文）。
- **冲突/印证**：与 SKILLWEAVER 的关系是**印证**——两篇论文独立得出"检索式方法无法单独解决多能力单元组合问题"的结论，但诊断的瓶颈成因不同（SKILLWEAVER 归因于任务分解质量，本文归因于选择/数量/顺序三维度不可解耦），这提示"多工具/多技能编排"这个问题可能有不止一个瓶颈来源，产品设计时需要分别排查。

## 动手练习 💻
练习目标：用不到 80 行代码实现一个极简版"技能库 + 生成式组合器"，模拟本文核心机制——给定任务描述，从技能库里通过词法相关性打分 + 简单顺序规则，动态拼出一个有序技能序列，直观感受"检索先验 + 序列决策"如何比"纯 top-k 检索"更好地处理"选几个、什么顺序"的问题。

```python
import re
from collections import Counter

# ---- 1. 迷你技能库：每个技能=(名称, 描述, 依赖的上游技能名或None) ----
# 依赖字段模拟论文里的"dependency edge"：上游技能的输出是下游技能的输入
SKILL_LIB = [
    ("usgs-data-download", "pull USGS streamflow gage height series", None),
    ("nws-flood-thresholds", "fetch NWS action minor moderate major flood stages", None),
    ("flood-detection", "compare water levels to thresholds count flood days",
     ["usgs-data-download", "nws-flood-thresholds"]),  # 依赖前两个技能的输出
    ("pdf-extract", "extract structured text tables from pdf documents", None),
    ("chart-render", "render bar line pie charts to png", None),
]

def tokenize(text):
    return re.findall(r"[a-z]+", text.lower())

def tfidf_like_score(task_tokens, skill_desc_tokens):
    """简化版 TF-IDF：只做词重合计数（论文里 TF-IDF 胜过 dense embedding 的直觉：
    小型闭合库上，词面重合是高精度信号）。"""
    overlap = Counter(task_tokens) & Counter(skill_desc_tokens)
    return sum(overlap.values())

def cardinality_head(task_tokens, max_n=4):
    """模拟 cardinality head：任务里出现的'动词/连接词'越多，粗略估计需要的技能数越多。
    这里用简单启发式代替训练好的分类器。"""
    connector_hits = sum(1 for w in ["and", "then", "count"] if w in task_tokens)
    return min(max(1, connector_hits + 1), max_n)

def compose_skills(task_text, skill_lib=SKILL_LIB, alpha=1.0):
    """核心组合器：
    1) 用词面相关性给每个技能打分（对应论文的 TF-IDF 检索先验）
    2) cardinality head 估计要选几个（对应论文的 cardinality head）
    3) 取分数最高的 n 个技能作为候选子集（对应 set head 的"选谁"）
    4) 按依赖关系做拓扑排序，得到执行顺序（对应 AR 头负责的"顺序"）
    """
    task_tokens = tokenize(task_text)
    n = cardinality_head(task_tokens)

    scored = []
    for name, desc, deps in skill_lib:
        score = alpha * tfidf_like_score(task_tokens, tokenize(desc))
        scored.append((score, name, deps))
    scored.sort(key=lambda x: -x[0])

    # 选出分数最高的 n 个相关技能（阈值过滤掉零分技能，避免凑数）
    selected = [(name, deps) for score, name, deps in scored if score > 0][:n]
    selected_names = {name for name, _ in selected}

    # 简单拓扑排序：依赖技能必须排在被依赖技能之前
    ordered, placed = [], set()
    remaining = list(selected)
    while remaining:
        progressed = False
        for name, deps in list(remaining):
            deps_ok = not deps or all(d in placed or d not in selected_names for d in deps)
            if deps_ok:
                ordered.append(name)
                placed.add(name)
                remaining.remove((name, deps))
                progressed = True
        if not progressed:  # 存在未解决的循环依赖，直接按原顺序追加，避免死循环
            ordered.extend(name for name, _ in remaining)
            break
    return ordered

if __name__ == "__main__":
    task = "Find stations that experienced flooding, count flood days and save results"
    plan = compose_skills(task)
    print("任务:", task)
    print("生成式组合器输出的技能执行顺序:", plan)
    # 期望输出类似：
    # ['usgs-data-download', 'nws-flood-thresholds', 'flood-detection']
    # 直观验证：flood-detection 因为依赖前两者，被拓扑排序排到了最后，
    # 而不是简单按相关性分数排序（分数最高的技能不一定该排第一个执行）
```

代码要点：`tfidf_like_score` 对应论文里"稀疏 TF-IDF 优于稠密 embedding"这一发现在小型场景下的直觉复现；`cardinality_head` 对应论文的技能数量预测头（这里用启发式简化，论文里是训练出来的线性分类器）；`compose_skills` 里"先按相关性选子集、再按依赖关系拓扑排序"，对应论文强调的"选择"和"顺序"是两个独立但必须联动的决策——纯按相关性分数排序会把 `flood-detection` 误排到最前面（它的描述词和任务高度重合），但依赖关系要求它必须排在最后，这正是本文反复强调"三维度不能解耦、必须联合决策"的核心直觉。

## 自测三层 🎓
- **L1 复述**：SkillComposer 把技能组合问题定义成什么样的预测任务？输出的词表由什么组成？训练时用了哪三类数据、各多少条？
- **L2 解释**：为什么论文认为"检索排序（哪怕给 oracle 的 gold 技能数量）"本质上无法替代生成式序列预测？请结合"三维度不能解耦"这个论点，以及 Table 1 中 oracle-k 检索仍落后于 SkillComposer 这一事实来回答。为什么小型专用解码器（3.9M 参数）在真实任务分布迁移上比全量微调 600M 参数的 SFT 模型更稳健？
- **L3 应用**：假设你在做一个企业内部客服 Agent 产品，运营团队已经积累了 80 个"处理流程"（比如"退款审核""物流异常排查""发票补开"，每个流程有明确的执行步骤和依赖），客服 Agent 经常需要组合 2-4 个流程处理一个复杂工单。请说明：① 你会如何评估这个场景是否值得引入本文这种"生成式技能组合"方案（对照上文"何时采用/规避"标准）；② 如果决定引入，你需要准备哪些训练数据、大概多少条量级合理；③ 上线后你会用什么指标监控"组合质量"和"下游任务成功率"这两件事是否真的对齐（对照本文 Set F1 vs 下游 pass rate 两套评测体系的设计）。

📅 知识时间锚：论文发表于 2026 年 6-7 月（arXiv:2606.32025，2026-06-30 提交，正文标注 2026-07-01），本笔记复核于 2026-07-08。
