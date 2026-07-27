---
tags: [论文笔记, 长上下文, 检索评测, in-context retrieval, 注意力稀释, 稀疏注意力, NIAH, Benchmark设计]
paper_id: "175"
filename: "175 - Can Language Models Actually Retrieve In-Context - Drowning in Documents at Million Token Scale.pdf"
authors: "Siddharth Gollapudi, Prasann Singhal, Sewon Min (UC Berkeley); Nilesh Gupta (UT Austin)"
year: 2026
笔记层级: 骨干
成熟度: 🧪
复核日期: 2026-07-15
---

# 语言模型能否真正实现上下文内检索？——百万Token规模下的"文档淹没"实测

📄 **原文 PDF**：[[RAW/175 - Can Language Models Actually Retrieve In-Context - Drowning in Documents at Million Token Scale.pdf]]

## PM 速判（30秒）
> 首个系统评测"语言模型直接在上下文里检索文档"（in-context retrieval, ICR）在**万级文档、百万token规模**下真实能力的论文。核心发现：模型内部的注意力信号几乎从不丢失（N=10,000篇文档时，仍有至少一个注意力头把正确文档排第一，命中率100%），但最终**生成**正确文档编号的准确率却从N=500时的95.8%坍缩到N=10,000时的0.2%——"心里想对，嘴上说错"。供应商宣称的"百万token上下文/NIAH高分"不能作为真实检索能力的证据，二者可能完全脱钩。

## 双层费曼 🗣️
> **给CEO的一句话**：把十万份合同扔进一个"支持百万字上下文"的大模型问"哪份提到了XX条款"，模型内部其实经常已经"看到"了正确答案，但语料越大，它把这个正确答案说出来的成功率就越低——这和考生划对了答案、涂卡时却涂错选项是同一类失误，语料规模越大，涂错概率越高。
>
> **给工程师的一段话**：BLOCKSEARCH基于Qwen3-0.6B，用随机4位doc code+block-sparse attention+on-policy辅助损失训练。核心发现：N=10k时L19层attention ranking（R19_any）仍100%命中gold，但解码Recall@1从95.8%坍缩到0.2%——GoldShare从0.91降到0.01，而层输出幅度仅降36%。SSMax+路由缓解后以1/7参数量追平MSA-4B。

## 问题域定位 🎯
- **根本约束**：检索长期由向量检索（dense retrieval）主导；ICR提出更激进的替代方案——把检索变成条件生成，模型直接从上下文语料里解码出相关文档的标识符，理论上能把检索和生成合并为一个模型，替代两阶段RAG pipeline，并支持超越内积相似度的复杂检索行为（§1）。
- **之前卡在哪**：此前工作要么依赖专有系统且无受控评测（[2]主要评测Gemini），要么只做小候选池上的**重排序**（BlockRank/ICR2，候选数远小于真实语料规模），没人在**语料级规模**（万级文档、百万token）上系统测过ICR是否真的可用。论文明确定义了两个此前被忽视的必要条件：C1（能否扩展到百万token语料）与C2（能否泛化到远超训练规模的语料）——此前没人同时验证过两者。
- **开启的路线**：证明了0.6B级小模型经过针对性训练/架构改动（随机doc code、on-policy辅助损失、block-sparse attention），可在语料规模上10×长度外推，配合两种缓解手段后，在百万token规模上追平dense retrieval，在dense retrieval天生弱的任务（LIMIT，词法相似度）上大幅超越——为"语料级ICR"建立了第一个可信、可控的验证基准。值得一提，本文作者Nilesh Gupta同时是前作BlockRank（reranking规模的ICR，[5]）的第一作者，BLOCKSEARCH是把该团队自己的reranking方案直接推向corpus-scale retrieval的自然延伸。
- **关闭的路线**：证明了"naive ICR"（前人方案：顺序整数ID+无辅助损失，即BLOCKSEARCH-position）在N=5,000时就会在每个数据集上坍缩到接近零，此路径基本被判定不可行；更重要的是，论文用并发模型MSA-4B的具体反例说明，**用NIAH/RULER这类合成大海捞针基准去推断"长上下文检索能力"这条评测路线本身是不可靠的**——靠NIAH分数推断真实检索能力的做法，被本文实证关闭。

## 核心机制

```
【评测任务：N篇文档 + 1个query，模型需吐出4位数字code定位相关文档】

  文档块(block-sparse,块内因果,互不可见)                      Query块(可看全部N个文档块)
┌────────────────────┐  ┌────────────────────┐        ┌─────────────────────────┐
│<bos>Doc 7421: ...   │  │<bos>Doc 0394: ...   │  ...   │Instruct: 给定query,     │
│(Doc 7421)<eos>      │  │(Doc 0394)<eos>      │  N篇    │检索最相关文档            │
│RoPE位置 0→300(重置)  │  │RoPE位置 0→300(重置)  │        │Answer: [beam搜索4位code] │
└────────────────────┘  └────────────────────┘        └─────────────────────────┘
   Tdoc≤300 token/篇，随机采样4位code(打破位置/语义关联)        ↑ RoPE位置300,查询全部N篇

【核心发现：L19层"注意力排序正确"与"最终生成答对"发生解离(§4，MS MARCO)】

 层号 0 ──────11~20──────── L19 ────── 25 ─ 27(输出)
       │      ←检索带,RL_sum │   ←解码带,commit到首位数字│
 N=500 : R19_any=1.00  GoldShare=0.91  最终Recall@1=95.8%  ← 三者一致
 N=10k : R19_any=1.00  GoldShare=0.01  最终Recall@1= 0.2%  ← 排序与生成解离!
          ↑ 每次query至少1个头仍将gold排第1   ↑ softmax分母被N-1篇干扰文档撑大,
            (attention ranking信号从未真正丢失)   gold的post-softmax权重被稀释淹没
            (Table 2 R19_any行/§4.2)             (Table 1: a19仅降36%,GoldShare降130×)

【两类缓解手段，作用在管道的不同环节(§5)】

 干扰文档数(N-1)↑ → softmax分母↑ → gold归一化权重↓（且gold自身分数也在缓慢侵蚀,Table 9）
        │                                         │
        ├─ 长度感知softmax(改分母的转换方式)        ├─ 文档级路由(改参与softmax的候选集)
        │   加性sink: 学阈值,漏概率质量进"沉没槽"    │   L16对全部N篇粗打分,只留Top-B=256篇
        │   (效果弱,MS MARCO N=10k仅0.2%→2.5%)      │   进入L17+精细attention
        │   SSMax: 预softmax分数×s·logN            │   (相当于模型内部插入一次"检索",
        │   (效果强,同条件→16.5%)                   │    但违背ICR"去掉外部检索"的初衷)
        └────────────────┬──────────────────────────┘
                          ↓
   MS MARCO N=10k Recall@1: 无干预0.2% → sink 2.5% → SSMax 16.5% → routing 18.8%
                            → SSMax+routing 20.5%（追平dense baseline 20.2%，Table 2）
```

## 设计决策解剖 ⚖️

| 决策点 | 他们的选择 | 被放弃的方案 | 为什么 | 这个选择何时失效 |
|---|---|---|---|---|
| 文档标识符 | 每步训练随机采样4位数字code（打破code-语义-位置关联） | 前人方案：按文档顺序位置分配1~N的连续整数ID | 训练时ID集合是推理时ID集合的严格子集，若用位置ID会诱导模型过拟合绝对位置（§3.2）；随机code消除了这条捷径 | 若产品需要长期稳定引用同一批文档（如跨会话都要认得"文档#4821"），随机code每次训练/推理都会变，无法当持久标识符使用，需额外一层稳定ID↔训练态随机code的映射 |
| Query前缀 | 去掉query prefix（前人方案会在语料前也拼一份query） | 保留query prefix（[5,2]证明能提升性能） | 保留prefix会阻止跨query复用同一份语料的KV cache prefill，在万级文档规模下"cost-prohibitive"（§3.2） | 当corpus很小或QPS很低、prefill本身不是瓶颈时，加回query prefix换取的性能提升可能比省下的prefill成本更划算 |
| 两种缓解机制并用 | SSMax（乘性重缩放）+ document-level routing 组合使用 | 只用其中一种 | 加性sink几乎无效；SSMax单独在MS MARCO N=10k只到16.5%仍落后dense的20.2%；routing单独到18.8%接近但仍差1.4点；只有叠加才在N=10k以20.5%微弱反超dense（Table 2） | 若产品架构要求"绝不能有独立检索/路由步骤"（这正是ICR的初衷），routing方案不可用，只能依赖SSMax单独工作，此时HotpotQA N=10k会留下明显缺口（SSMax单独56.8% vs dense 79.5%，Table 2） |
| 训练规模 vs 评测规模 | 训练时batch内语料仅256篇文档（b=16×16docs），却要求泛化到10,000篇 | 直接在万级文档规模上训练 | block-sparse attention只降低了推理阶段的注意力开销，训练阶段in-batch负例的prefill规模仍受限于显存/工程成本，只能靠"长度泛化能力"而非"直接大规模训练"覆盖目标场景（§3.2 Training regime） | 当目标语料规模超过训练规模约10×（约合50万token门槛，论文实测的"cliff"）时，无论叠加哪种缓解手段，性能都会退化到远低于dense baseline的水平（MS MARCO N=10k即便SSMax+routing也只是勉强追平dense，而非可靠超越，Table 2） |
| 评测语料的干扰文档来源 | 用Qwen3-Embedding-8B检索出的24个hard negatives构造评测语料 | 随机负例（如前人工作[2]） | "makes the input corpus far more challenging"，比随机负例更贴近真实检索的难度（§3.3脚注2） | 若实际部署语料中的干扰文档并非"语义上很像但错"的高难度负例，而是大量完全无关的噪声文档（更接近真实邮件/网盘归档的分布），当前构造方式可能高估了真实检索难度，也可能因训练本身就针对hard negative优化，遇到简单负例时的鲁棒性未知 |

## 成本与量级 💰

| 维度 | 数值 | 来源 |
|---|---|---|
| 骨干模型规模 | BLOCKSEARCH/Qwen3-dense均基于Qwen3-0.6B（约6亿参数）；对照组MSA-4B约42亿参数（7×大） | §3.2, §3.3, Table 7 |
| 训练硬件与轮次 | 8×NVIDIA A100（DDP），BLOCKSEARCH与Qwen3-dense均各训练1 epoch | Appendix D Table 7；Appendix E Table 8 |
| 训练数据规模 | RLHN过滤后522,487条(query,16文档)样本，训练语料共约1亿token，来自MS MARCO/HotpotQA/FEVER/NQ/SCIDOCS-RR/FiQA/ArguAna共7个子集 | Appendix C, Table 5 |
| 训练时单batch语料规模 | 每次训练仅256篇文档共享同一prefill（b=16 query×16 docs） | §3.2 Training regime |
| 评测语料规模（MS MARCO/HotpotQA） | 最高N=10,000篇，约95万~116万token | Table 2, Table 4 |
| 评测语料规模（NQ） | 因单文档更长（均139.6 token/篇），截断到N=8,607篇，约120万token（§3.3正文表述为"约1.2M token"，Table 2标题表述为"1M-token prefill budget"，两处措辞不完全一致） | §3.3, Table 2标题, Table 4 |
| LIMIT语料规模 | N=46（仅gold）到N=5,000，对应8K~85万token的length-generalization sweep | Table 3, §6 |
| OBLIQ语料规模 | Math N=3,507；Twitter N=10,000（72K全量子采样）；Writing N=4,000（10,388全量子采样） | Table 14, Appendix L |
| 解码开销 | beam width=25，每步剪枝到top-5，返回top-5候选 | Appendix J, Algorithm 2 |
| 训练总时长/GPU小时/美元成本 | 论文未披露 | — |
| 端到端推理延迟（单query） | 论文未披露 | — |
| 最小可行配置（论文自身呈现的下限） | 0.6B骨干 + 8×A100 + 约1亿token训练语料，即可在百万token语料上追平dense retrieval | 综合Table 2, Appendix D |

## 证据审计 🔬

**实验设计公平吗？** Qwen3-dense基线与BLOCKSEARCH共享同一backbone（Qwen3-0.6B）、同一份RLHN训练数据、同一批作者训练（Appendix E），是全文最受控、最公平的对照。相比之下，MSA-4B并非作者训练，是"as reported in the original paper"的并发工作，论文自己也称其为"oracle reference"——这层对比更多是"我们用1/7参数量做到什么程度"的语境说明，而非严格意义的消融对照。HotpotQA的Recall@1采用"多个gold中命中任意一个即算对"的宽松口径，论文明确承认这是"primarily to ensure compatibility with MSA, which does not support more than recall@1"（§3.4）——为了让并发基线可比，评测口径被适当放宽，这一点论文有透明披露。

**测试集是否可能被污染/检索任务设计是否代表真实场景？** 评测语料的24个hard negative由Qwen3-Embedding-8B检索得到，而这同一个embedding模型也被用来给RLHN训练数据打分、筛除假负例（§3.2 Training data），并为on-policy辅助损失提供teacher相关性分数。也就是说，"训练难度"和"评测难度"由同一个judge模型定义——这不构成数据泄漏，但存在评测构造上的循环：BLOCKSEARCH和Qwen3-dense本质上都在被训练/评测于"Qwen3-Embedding-8B认为难"的分布上，这可能部分解释了二者在MS MARCO/NQ/HotpotQA中大N之前为何贴得如此接近。

**最强证据**：Table 1和Table 9（MS MARCO，400 queries）对L19层的信号/噪声分解是全文最扎实的机制性证据——不是简单的"准确率随N下降"相关性描述，而是给出了可证伪的定量预测："纯O(N)分母稀释"应导致约20倍的gold权重坍缩，而实测的gold_post19坍缩幅度约150倍（0.0320→0.0002，N=500→10k），倒逼作者承认这是gold自身分数侵蚀（约3个logit单位）与噪声log-sum-exp增长（约3.5个单位）的复合效应（Appendix G）。这种"用理论下界排除简单解释、倒逼复合机制"的论证方式，比大多数评测论文"我们测了、分数变差了"的叙述扎实得多。

**最可疑的数字**：摘要声称BLOCKSEARCH在LIMIT上"achieving a 3× higher score"，但Table 3实际比值在不同N上并不稳定——N=46时0.439 vs 0.176（约2.5倍），N=5,000时0.149 vs 0.035（约4.3倍）——摘要给出单一"3×"却未注明是哪个N、哪个指标（R@1/R@2/R@5）下的比值，是对复杂结果的一种简化包装。更值得警惕的是Appendix L的OBLIQ结果（Table 15）：在Math和Writing两个"oblique"（抽象关联）检索任务上，BLOCKSEARCH-SSMax-routing的Recall@1（Math 0.008、Writing 0.001）反而**低于**Qwen3-dense（Math 0.014、Writing 0.009），Recall@5同样是dense占优（Math 0.057 vs 0.032，Writing 0.025 vs 0.006）——摘要"3×超越dense retrieval"的叙事只在LIMIT成立，换到OBLIQ的抽象推理式检索任务上dense retrieval反而更好，这个反例被放进了附录而非主文。

**审稿人会要求补充**：(1) 训练/推理的wall-clock时间、GPU小时或美元成本，全篇未披露；(2) MSA-4B的NIAH成绩是"as reported in the original paper"，175号作者并未独立复现验证，用它论证"NIAH虚高"时这一层间接性值得注明；(3) 加性sink作为正式提出的"两种缓解手段"之一，效果在所有数据集大N处都接近无效（MS MARCO N=10k仅0.2%→2.5%），是否值得作为独立方法呈现、而非仅在消融表里出现，值得追问；(4) n=400/1,000的query抽样没有报告跨seed的方差或置信区间。

**最小复现实验**：不需要8×A100。取一个开源小模型（如Qwen2.5-0.5B），零样本或轻量微调，构造N∈{50,500,2000,8000}的合成key-value检索任务，测两个代理指标——(a) 对query最后一个token做logit lens/attention权重探测，统计"至少一个头把gold文档排第一"的比例（对应R19_any）；(b) decode最终答案的Recall@1。预算：单卡GPU、几十分钟即可看到"排序比例几乎不降但生成准确率随N坍缩"这一定性现象是否复现；本笔记"动手练习"提供了不依赖真实模型权重的进一步简化版本。

## 可复用点（PM决策）

**何时采用ICR/生成式检索**：查询分布里存在大量"dense retrieval结构性做不好"的任务（词法/条件相似度而非语义相似度，参考LIMIT的设计动机——嵌入检索的理论局限[4]），且语料规模能控制在骨干模型训练规模的10倍以内。

**何时规避**：(1) 语料规模会超出训练规模10×这条线（论文实测的坍缩临界点），且没有预算做SSMax+routing这类针对性缓解；(2) 查询需要真正抽象/间接的关联推理（OBLIQ式"这段代码和这道数学题用了同一种论证技巧"），目前ICR和dense retrieval在这类任务上都接近失效，不存在能显著解决它的现成方案；(3) 产品架构上不能接受"routing"重新引入的检索-路由步骤——这在scale up时几乎是必需品，等于部分放弃了ICR"去掉外部检索"的初衷。

**供应商拷问清单**：
1. "你们百万token检索能力的证据是NIAH/RULER这类合成基准，还是带真实困难负例的语料？能否提供Recall@1在N=500/5,000/10,000等不同语料规模下的完整曲线，而不是单点通过率？"（直接对应本文MSA-4B案例：RULER NIAH近乎满分，但MS MARCO N=10,000实测Recall@1仅16.0%，低于dense baseline的20.2%）
2. "检索失败时，是模型内部真的'没看到'，还是'看到了但没能正确生成出来'？能否提供层间/头间的注意力排序诊断？"（对应本文R19_any与最终Recall@1的解离：前者N=10k时仍100%，后者坍缩到0.2%）
3. "你们的'百万token'是训练时就见过规模内的插值，还是外推到远超训练规模的语料？外推倍数是多少？"（对应本文C1 scale/C2 length-generalization的区分，以及10×训练规模处的实测坍缩临界点）

## 关联网络 🕸️

**相关论文**：
- [[Wiki/论文笔记/09_长上下文记忆/82_DeepSeek-V4百万Token高效长上下文]] — 该笔记"审稿补充"栏目自己提出"1M token的NIAH结果应补充"的疑问，175号给出了直接呼应（见下文冲突/印证）。
- [[Wiki/论文笔记/09_长上下文记忆/132_Lighthouse-Attention长上下文预训练]] — 该笔记建议PM"用NIAH作为长上下文检索能力的快速代理指标"，与175号构成本笔记库内最明确的一组数字级冲突（见下文冲突/印证）。
- [[Wiki/论文笔记/09_长上下文记忆/124_MeMo记忆即模型]] — 与175号的ICR构成"参数化记忆 vs 上下文化检索"的方法论对照：MeMo将语料SFT进专门的记忆模型参数（更新需重训，但避免检索噪声），175号的ICR将语料显式放进上下文（无需重训即可更新，但要正面对抗attention dilution）。175号Related Work（§2）明确点出了这组对照："generative retrieval... stores corpus information in model parameters, requiring retraining when the corpus changes. In contrast, ICR encodes the corpus explicitly in context"。
- [[Wiki/论文笔记/10_RAG检索/133_Grep-vs-向量检索Agent搜索对比]] — 两篇论文在不同抽象层级发现同一母题："找得到"与"能正确交付出来"是两件事。133号发现Agent框架（Harness）对检索准确率的影响可超过检索方法本身（同一模型换框架相差34.5pp）；175号发现模型内部attention ranking几乎从不丢失（R19_any常年100%），但把结果正确"交付"为生成token的环节才是真正瓶颈。

**相关概念**：
- [[Wiki/概念/01_架构技术/稀疏注意力与长上下文效率]] — 该概念笔记提到的"迷失在中间"问题（"即使context放进去了，模型也不一定注意到中间的信息"），被175号的attention dilution机制分析（Appendix G的信号/噪声复合分解）进一步精细化、量化：不是位置效应，而是随文档数N增长的softmax分母稀释+gold自身分数侵蚀的复合效应。
- [[Wiki/概念/05_记忆与检索/RAG检索增强生成]] — 该概念笔记的寓言强调"检索这一步一旦失败，后面答案组织得再好也无法弥补"，讨论的是**检索完全没找到**（商品名对不上、压根没查到条目）；175号揭示了一种更隐蔽的失败模式——**检索内部其实找到了，但没能正确交付成最终输出**（attention ranking命中100%，生成Recall@1仍坍缩到0.2%），是对"检索失败"这一概念的一次细分。同时175号明确将ICR定位为RAG两阶段pipeline的替代方案，但其document-level routing缓解手段又在模型内部重新引入了"先检索再精读"的结构，作者自己承认这"reintroduces a retrieve-then-read (e.g. RAG) decomposition inside the model"——是对"ICR能否真正取代RAG"的一次自我证伪式讨论。
- [[Wiki/概念/03_推理与评测/Benchmark设计与污染]] — 175号的核心质疑（NIAH/RULER"potentially overstating real-world retrieval capability"）严格来说是**构造效度**问题（评测任务是否真的代表目标能力），与该笔记聚焦的**数据污染/标签泄漏**问题不完全是同一件事，但同属"benchmark是否测到了它宣称测的东西"这一方法论问题域，可作为该概念笔记未来补充"构造效度"维度的素材。

**冲突/印证**：
- **冲突**：132号笔记建议"用NIAH作为长上下文检索能力的快速代理指标"（PM决策参考），与175号的MSA-4B实证反例直接冲突——MSA-4B在RULER NIAH上"near-perfect"（据其原论文），但在175号独立构造的MS MARCO N=10,000语料上Recall@1仅16.0%，低于同规模dense baseline的20.2%（Table 2）。NIAH高分不能保证真实语料检索准确率。
- **印证/深化**：82号笔记"审稿补充"栏目自己提出"1M token的NIAH结果应补充"的疑问，被175号进一步深化——即使补充NIAH数据也不足以证明真实检索能力，因为175号的实验证明二者可以完全解离：L19层attention排序命中率（R19_any）在N=10,000时仍保持100%，但同一模型最终生成的Recall@1却坍缩到0.2%（Table 1、Table 2）。

## 动手练习 💻

```python
"""
练习目标：复现175号论文的核心机制——注意力稀释（Attention Dilution）。
论文发现（Table 1/Table 2/§4.2）：语料规模N增大时，
  (1) gold文档的attention排序命中率（R19_any，argmax是否选中gold）几乎不降；
  (2) 但post-softmax归一化后gold分到的权重（GoldShare）随N增大坍缩。
本练习用一个极简单头打分模型复现这个"排序正确但权重坍缩"的解离，
并对比SSMax缓解手段（预softmax分数×s·logN）能在多大程度上延缓坍缩。
"""
import numpy as np

rng = np.random.default_rng(42)

def simulate_corpus_scores(N, gold_score=8.0, distractor_mean=2.0, distractor_std=1.5):
    """模拟一次prefill中N篇文档在某层某头的MaxSim分数（对应论文式3），最后一个是gold"""
    distractor_scores = rng.normal(distractor_mean, distractor_std, size=N - 1)
    scores = np.append(distractor_scores, gold_score)
    return scores, N - 1  # 返回打分数组和gold的下标

def attn_rank_hit(scores, gold_idx):
    """AttnRank：argmax是否命中gold，对应论文的R19_any"""
    return int(np.argmax(scores) == gold_idx)

def gold_share(scores, gold_idx):
    """GoldShare：softmax归一化后gold分到的权重，对应论文式1、Table 1的a_G/a"""
    exp_scores = np.exp(scores - scores.max())  # 数值稳定softmax
    weights = exp_scores / exp_scores.sum()
    return weights[gold_idx]

def ssmax_gold_share(scores, gold_idx, N, s=0.43):
    """SSMax缓解：预softmax分数先乘s*log(N)再softmax，对应论文§5.1"""
    scaled = scores * (s * np.log(max(N, np.e)))
    exp_scores = np.exp(scaled - scaled.max())
    weights = exp_scores / exp_scores.sum()
    return weights[gold_idx]

# 扫描不同"文档密度"（语料规模N），观察三个指标随N的变化
N_sweep = [50, 200, 1000, 5000, 10000, 50000]
n_trials = 300  # 重复多次取平均，模拟论文n=400 queries的做法

print(f"{'N':>8} {'AttnRank命中率':>14} {'GoldShare均值':>14} {'SSMax后GoldShare':>16}")
for N in N_sweep:
    hits, shares, ssmax_shares = [], [], []
    for _ in range(n_trials):
        scores, gold_idx = simulate_corpus_scores(N)
        hits.append(attn_rank_hit(scores, gold_idx))
        shares.append(gold_share(scores, gold_idx))
        ssmax_shares.append(ssmax_gold_share(scores, gold_idx, N))
    print(f"{N:>8} {np.mean(hits)*100:>13.1f}% {np.mean(shares)*100:>13.4f}% {np.mean(ssmax_shares)*100:>15.4f}%")

# 预期观察（对应论文定性结论，非精确复现Table 1的具体数值）：
# - AttnRank命中率应始终接近100%——gold_score显著高于distractor分布均值，
#   这正是论文里R19_any长期维持~100%（即使N=10,000）现象的简化版本。
# - 未处理的GoldShare应随N增大单调下降：干扰文档越多，softmax分母被稀释越严重。
# - SSMax后的GoldShare下降速度应明显慢于未处理版本，因为log(N)缩放部分抵消了分母膨胀。
# 本模型只隔离了"分母随N增长"这一个变量，未还原论文Table 9发现的
# "gold自身分数也在侵蚀"的复合效应，因此这里的坍缩速度会比论文实测的~150×更温和——
# 这恰好呼应了附录G的论点：真实坍缩比单纯分母稀释预测的更严重，是复合机制。
```

## 自测三层 🎓

**L1 复述**：在MS MARCO N=10,000时，L19层的注意力排序命中率（R19_any）和BLOCKSEARCH最终生成的Recall@1分别是多少？这组数字说明检索失败的根源在"排序"还是"读出"？

**L2 解释**：论文为什么要同时使用SSMax和document-level routing两种缓解手段，而不是只选效果更强的一个？各自的短板具体是什么（结合Table 2在MS MARCO/HotpotQA上的数字回答）？为什么不直接把训练语料规模也扩大到万级文档，一次性解决长度泛化问题？

**L3 应用**：你所在公司要为一个十万级文档的知识库选型"新一代大模型原生检索"方案，供应商展示了"200万token上下文+NIAH准确率99%"的成绩单。结合本文的发现，列出你会追问的至少三个具体问题，并说明如果供应商只能补充一项数据，你会优先要哪一项、为什么。

📅 知识时间锚：论文提交于2026年7月1日（arXiv 2607.01538），本笔记复核于 2026-07-15。
