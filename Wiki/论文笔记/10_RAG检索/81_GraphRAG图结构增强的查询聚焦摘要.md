---
tags: [论文笔记, GraphRAG, 知识图谱, RAG, 摘要, 基础论文, Microsoft]
paper_id: "81"
笔记层级: 骨干
复核日期: 2026-07-04
---

# GraphRAG：图结构增强的查询聚焦摘要

📄 **原文 PDF**：[[RAW/81 - From Local to Global - A Graph RAG Approach to Query-Focused Summarization.pdf]]

## PM 速判
> **一句话**：GraphRAG用LLM从文档集构建知识图谱+Leiden社区检测+社区摘要，在"整个语料库描述了什么"类全局问题上完整性和多样性显著超越向量RAG——代价是索引成本高出数量级，适合离线大规模文档集的全局推理。

## 双层费曼 🗣️
> **给CEO**：传统RAG搜索像"在书里翻到相关的那页"，对"找某某人的电话号码"这种精确问题很有效。但如果你问"这100篇研究报告的共识结论是什么"，传统RAG就会遗漏信息——因为它只看与问题相似的那几段。GraphRAG的做法是先通读所有文档，提取出"谁做了什么、和谁有关系"，画成关系网（知识图谱），然后把网络上紧密相连的信息点聚合成"社区"，给每个社区写摘要。查询时先定位相关社区，再从社区摘要中综合答案。这就像从"在100本书里翻页"升级到"先读完所有书画出思维导图再看问题"——回答全局性问题时完整性提升显著，代价是预处理成本高（需要LLM读全文+建图+写摘要）。
> **给工程师**：GraphRAG = 离线知识图谱索引 + 在线Map-Reduce全局查询。离线阶段：(1) 文档分块后每块调用LLM提取实体和关系三元组（subject, predicate, object），输出结构化JSON；(2) 合并共指实体，构建异构图（节点=实体，边=关系+声明）；(3) 用Leiden算法做分层社区检测（多层粒度以适应不同范围的问题）；(4) 每层每个社区调用LLM生成摘要（社区内实体+关系的自然语言总结）。在线阶段：局部查询=向量检索→相关社区→LLM生成；全局查询=对所有社区并行生成局部答案→Map→按重要性排序→Reduce聚合为综合答案。关键设计：社区检测是多层级的（Leiden的resolution参数控制粒度），查询时按问题范围选择合适层级——这解决了"回答粒度"的挑战。

## 问题域定位 🎯
- **解决什么根本约束**：标准向量RAG的根本局限——检索的本质是"相似度匹配"，只能找到与查询语义相近的局部chunks，无法聚合跨文档的全局结构和模式。当用户问"这个语料库的核心叙事是什么？""各方的立场分歧在哪？""趋势随时间如何变化？"时，这些问题不是任何单个chunk能回答的。
- **之前卡在哪**：已有的全局理解方案：全文摘要（summarization）对所有文档生成一个摘要——信息压缩率过高，细节丢失严重；多跳QA（multi-hop QA）只能处理预定义的关系模式，不支持开放域全局推理；分层摘要（hierarchical summarization）的聚合粒度和方式缺乏结构化指引。没有一个方案能同时覆盖"结构化实体关系表达"和"分层语义聚合"这两个需求。
- **开启/关闭了哪条路线**：开启了"图+RAG混合架构"路线——GraphRAG证明知识图谱作为RAG的索引结构在全局查询上有向量RAG无法替代的优势。开启了LLM驱动的非结构化→结构化转换作为索引构建步骤。明确了GraphRAG不替代向量RAG而是互补——局部精确检索用向量、全局理解用图谱。

## 核心机制

```
┌──────────────────────────────────────────────────────────────────┐
│                    GraphRAG 完整管线                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ 离线索引阶段 (Source Documents → Graph Index)            │    │
│  │                                                         │    │
│  │  Step 1: 文档分块 (Text Chunking)                       │    │
│  │    Input: 100篇新闻文章                                  │    │
│  │    → 每篇分块至 ~300 tokens/chunk                       │    │
│  │                                                         │    │
│  │  Step 2: LLM提取实体+关系 (Entity & Relation Extraction) │    │
│  │    For each chunk:                                      │    │
│  │      Prompt: "Extract entities and relationships..."    │    │
│  │      Output: {                                           │    │
│  │        entities: [{name, type, description}],            │    │
│  │        relationships: [{source, target, description}]    │    │
│  │      }                                                   │    │
│  │    ↳ 100块 × ~10实体/块 = ~1000个实体                     │    │
│  │                                                         │    │
│  │  Step 3: 构建知识图谱 (Graph Construction)               │    │
│  │    ┌─────────────────────────────────────┐             │    │
│  │    │  节点: 实体 (去重+共指消解后)        │             │    │
│  │    │  边: 关系 (权重=共现次数)            │             │    │
│  │    │                                     │             │    │
│  │    │  (Elon Musk)──[founded]──(SpaceX)   │             │    │
│  │    │       │                      │       │             │    │
│  │    │  [owns]                 [launched]   │             │    │
│  │    │       │                      │       │             │    │
│  │    │   (Twitter)           (Starship)     │             │    │
│  │    └─────────────────────────────────────┘             │    │
│  │                                                         │    │
│  │  Step 4: Leiden社区检测 (Community Detection)            │    │
│  │    ┌─────────────────────────────────────┐             │    │
│  │    │  Leiden算法 分层聚类:                 │             │    │
│  │    │  Level 0 (fine):  社区大小 ~5-10实体 │             │    │
│  │    │  Level 1 (medium): 社区大小 ~20-50   │             │    │
│  │    │  Level 2 (coarse): 社区大小 ~100+    │             │    │
│  │    │                                     │             │    │
│  │    │  [SpaceX/Starship] (Community A)     │             │    │
│  │    │  [Twitter/Musk]    (Community B)     │             │    │
│  │    │  → 高层: [Musk商业帝国] (Community AB)│            │    │
│  │    └─────────────────────────────────────┘             │    │
│  │                                                         │    │
│  │  Step 5: 社区摘要生成 (Community Summarization)          │    │
│  │    For each community at each level:                    │    │
│  │      Prompt: "Summarize the entities and                │    │
│  │               relationships in this community"          │    │
│  │      Output: 自然语言摘要 (嵌入→向量索引)                │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                     │
│                            ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ 在线查询阶段 (User Query → Final Answer)                  │    │
│  │                                                         │    │
│  │  用户问: "What are the main topics in this corpus?"     │    │
│  │         │                                                │    │
│  │    ┌────┴────┐                                          │    │
│  │    │  局部查询 │  (Local Search)                         │    │
│  │    │  向量检索 → 相关社区摘要                             │    │
│  │    │  → 适用: 精确事实性问题                              │    │
│  │    └─────────┘                                          │    │
│  │                                                         │    │
│  │    ┌──────────┐                                         │    │
│  │    │  全局查询  │  (Global Search) ← GraphRAG核心创新     │    │
│  │    │          │                                         │    │
│  │    │  MAP 阶段: 选定层级的所有社区并行                    │    │
│  │    │    每个社区 → LLM生成"该社区对问题的局部答案"        │    │
│  │    │           → 附带重要性评分 (社区大小/密度)            │    │
│  │    │                                                    │    │
│  │    │  REDUCE 阶段:                                       │    │
│  │    │    按重要性排序 → 选Top-N个局部答案                   │    │
│  │    │    → LLM聚合为综合答案                               │    │
│  │    │    → 附引用源 (社区→实体→原文chunk)                  │    │
│  │    └──────────┘                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  关键代价:                                                       │
│  离线索引: ~N_chunks × LLM_call_per_chunk + N_entities² 图处理   │
│  在线查询(全局): ~N_communities × LLM_call (Map) + 1 (Reduce)   │
└──────────────────────────────────────────────────────────────────┘
```

## 设计决策解剖 ⚖️
| 决策点 | 选择 | 被放弃方案 | 为什么 | 何时失效 |
|--------|------|----------|--------|---------|
| 图构建方式 | LLM逐chunk提取实体+关系 | NER + rule-based关系抽取 | Rule-based受限于预定义实体类型和关系schema，对新领域/新实体泛化差。LLM可以在zero-shot下提取任意类型实体和关系（如"政治联盟""商业竞争"），覆盖范围远大于预定义schema | 当文档量极大（百万级）且LLM调用成本不可接受时——NER+pipeline更经济。或者当领域需要极高精确率且关系类型固定（如法律文书的关键条款引用），LLM的过度灵活性反而引入噪音 |
| 社区检测算法 | Leiden（模块度最大化，分层聚类） | Louvain (无分层) 或 Spectral Clustering | Leiden比Louvain更快且保证社区连通性；分层设计允许用户在不同粒度查询（fine-grained查具体主题、coarse查全局趋势）。Louvain可能产生不连通社区 | 当图规模极大（百万节点）时——Leiden的单层检测仍然O(N log N)，但多层检测的叠加代价显著。或当知识图谱是星型结构/无天然社区时（如围绕单一核心实体的实体关系网），社区检测退化为无意义分割 |
| 全局查询的聚合策略 | Map-Reduce（并行局部答案→聚合） | 将所有社区摘要拼接后直接生成 | 当社区数量>100时，拼接所有摘要会超过LLM的context窗口限制。Map-Reduce天然可扩展到任意数量的社区（Map并行，Reduce可控）。Reduce阶段按重要性排序截断，避免噪声社区污染最终答案 | 当不同社区的答案之间存在复杂依赖关系时（如"A社区的观点被B社区反驳"）——Map-Reduce的独立性假设引入信息损失，需要多轮迭代或全量聚合 |
| 实体/声明的共指消解 | LLM驱动的批量去重 | 基于embedding相似度的聚类 | 实体名称多种多样："Elon Musk""Musk""Elon""@elonmusk"指向同一实体。LLM可以在上下文中理解这些变体并去重，embedding相似度对短名称不可靠 | 当实体名过于相似且表示不同实体时（如不同人的同名同姓）——需要额外的消歧信息（如时间戳、组织机构上下文），纯LLM无法解决 |

## 成本与量级 💰
- **索引成本（离线）**：对1000篇中等长度文章（约500K tokens），需要~3000次LLM调用（分块+提取+摘要），按GPT-4o价格（$2.5/1M input, $10/1M output tokens），总cost约$50-150（取决于prompt长度和输出token数）
- **索引成本估算公式**：Cost ≈ N_chunks × (C_extract + C_summarize_per_community)，其中C_extract随chunk大小增长（输出包含所有实体关系），C_summarize随社区内实体数增长
- **查询成本（在线）**：局部查询≈1次LLM调用（向量搜+生成），全局查询≈N_communities×1（Map）+1（Reduce）。Leiden的fine级社区可能有50-200个，Map阶段cost约$0.2-1.0
- **索引时间**：对1000篇文章，离线索引总时间约10-30分钟（取决于LLM API延迟和并行度）
- **最小可行配置**：对<100篇文章的小规模语料库，用GPT-4o-mini做提取和摘要即可，总索引成本可控制在$5以内
- **与向量RAG的成本对比**：向量RAG索引成本≈零（仅chunk+embed，无LLM调用）；GraphRAG索引成本高出10-50倍。但全局查询质量提升在特定场景下值得这个代价

## 证据审计 🔬
- **实验公平性**：基本公平，但有局限。对比方为向量RAG（NaiveRAG），评测方式为人类盲测（对完整性和多样性的主观评分）。两个系统都基于相同的底层LLM，排除了LLM本身能力差异的干扰。但人类评估者只有2-3人，且评估标准（"完整性"和"多样性"）的主观定义未经过严格的inter-rater reliability验证。
- **最强证据**：人类评估中GraphRAG在全局问题的"完整性"和"多样性"两个维度上均一致胜出向量RAG——且优势不仅在均值，几乎在所有测试问题上都胜出（效应一致性强）。更有说服力的是：向量RAG在全局问题上甚至不如"全文摘要"基线——这说明向量RAG的局部chunk检索在全局问题上是有害的（碎片化信息误导了LLM），而非简单的"检索量不够"。
- **最可疑数字及原因**：(1) 论文未报告索引成本的具体数字——这是GraphRAG最大的实际障碍，但被"在未来工作"一笔带过。任何企业决策者需要知道：为10万篇文档建GraphRAG索引需要花多少钱？(2) 全局查询的Map-Reduce聚合中，Map阶段生成的局部答案质量参差不齐——论文未分析"坏社区答案"对最终聚合的污染程度。如果Reduce阶段LLM的甄别能力有限，可能存在GIGO（Garbage In Garbage Out）风险。(3) 评估语料规模太小（百到千篇量级）——无法证明方案在万级/十万级文档规模下的可扩展性。Leiden算法在大图上的行为可能显著不同。
- **审稿补充要求**：需补充索引成本与语料库规模的量化关系表；需补充多人类评估者的inter-rater reliability分数（如Krippendorff's alpha）；需补充GraphRAG在精确事实性问题上的表现（验证"局部查询"模式的有效性）；需补充对Leiden resolution参数的敏感性分析；需补充实体提取质量的错误分析（LLM漏提取/误提取的比例和类型）。
- **最小复现设计**：取50篇维基百科文章（同一主题，如"人工智能"）→ 用GPT-4o-mini逐段提取实体+关系 → 用networkx构建知识图谱 → 实现Leiden算法（用igraph库）→ 生成社区摘要 → 设计3个全局问题（如"What are the major subfields and their relationships?"）→ 对比向量RAG和GraphRAG的回答质量。核心代码约250行。

## 可复用点
- **何时采用**：(1) 需要回答跨文档全局问题的企业知识库（如年度报告分析、科研文献综述、产品文档全览）——GraphRAG是当前最成熟的方案；(2) 文档集相对稳定（更新频率低），可以承受离线索引成本——新闻日更不适合，季度报告适合；(3) 用户群体需要探索型/分析型查询而非精确检索型查询。
- **何时规避**：(1) 纯精确检索场景（"产品X的规格是多少"）——向量RAG在精确性和延迟上都更优；(2) 实时/近实时文档入库——GraphRAG的离线索引流程不能几秒内完成；(3) LLM调用预算极其有限——对于大规模语料（百万级文档），索引成本可能达到数千美元；(4) 图结构稀疏（如纯产品手册、API文档——实体关系很少）——图索引的额外收益不足以覆盖成本。
- **供应商拷问清单**：(1) 你们的RAG系统是否使用图谱索引？图谱是如何构建和更新的？(2) 实体提取的LLM调用成本是多少？对100K文档的索引总成本是多少？(3) 全局查询的Map-Reduce过程中，社区数量如何控制？如何保证Reduce不丢失关键信息？(4) 当源文档更新时，图谱索引是增量更新还是全量重建？更新延迟多久？(5) 在你们的评测中，全局问题的答案完整性和精确事实问题的准确性分别是多少？各自与纯向量RAG的差距？

## 关联网络 🕸️
- **相关论文**：
  - [[Wiki/论文笔记/02_前沿模型报告/76_GPT-4技术报告]] — GraphRAG在实体提取和社区摘要环节严重依赖GPT-4级LLM的能力——提取质量和模型能力强相关
  - [[Wiki/论文笔记/10_RAG检索/81_GraphRAG图结构增强的查询聚焦摘要]] — 本论文自身
- **相关概念**：
  - [[Wiki/概念/05_记忆与检索/RAG检索增强生成]] — GraphRAG是RAG架构的图结构扩展版本
  - [[Wiki/概念/05_记忆与检索/GraphRAG图结构检索]] — 本论文定义了该概念的核心范式：LLM抽取实体关系+Leiden社区检测+社区摘要
  - [[Wiki/概念/05_记忆与检索/嵌入模型]] — GraphRAG的局部查询模式依赖嵌入模型对社区摘要进行向量检索
- **冲突/印证**：
  - **印证**：HippoRAG(2024, Ohio State)独立验证了"图谱增强RAG在全局任务上优于纯向量RAG"的结论——HippoRAG用不同的图构建方式（个性化PageRank而非Leiden社区检测）也达到了类似的全局理解提升。两者共同印证了图索引对RAG的增益不是特定GraphRAG实现的产物，而是结构化的实体关系索引固有的优势。
  - **补充/挑战**：LightRAG(2024, 港大)提出了<10%成本的简化图RAG方案——不对实体做社区检测，直接在图上游走检索。LightRAG在部分任务上达到GraphRAG的85-90%效果、成本降低90%。这挑战了GraphRAG"社区检测+分层摘要"的必要性——是否简化的图遍历索引足以捕捉全局结构？

## 动手练习 💻
**练习目标**：用LLM从短文中抽取实体-关系三元组，用networkx构建简易知识图谱并做社区检测。

```python
"""
简易GraphRAG核心管线：文本→实体关系三元组→知识图谱→社区检测
实际运行需要填入LLM API key，此处展示完整代码框架
"""
import json
import networkx as nx
from collections import defaultdict

# ====== 1. 模拟文本数据 ======
documents = [
    "Elon Musk founded SpaceX in 2002. SpaceX developed the Falcon 9 rocket and Starship. "
    "SpaceX is headquartered in Hawthorne, California.",

    "Elon Musk acquired Twitter in 2022 for $44 billion. He renamed it to X. "
    "Twitter's headquarters is in San Francisco. Musk also owns Tesla.",

    "Tesla was founded by Martin Eberhard and Marc Tarpenning in 2003. "
    "Elon Musk joined Tesla as chairman in 2004 and later became CEO. "
    "Tesla produces electric vehicles including Model 3 and Model Y.",

    "SpaceX launches Starlink satellites for global internet coverage. "
    "Starlink is a constellation of satellites in low Earth orbit. "
    "The Falcon 9 rocket is used to launch Starlink satellites.",

    "Tesla acquired SolarCity in 2016. SolarCity was founded by Lyndon Rive and Peter Rive. "
    "Elon Musk was the chairman of SolarCity. The acquisition was controversial."
]

# ====== 2. LLM提取实体关系三元组 ======
def extract_triples(text: str, api_key: str = None) -> list:
    """用LLM从文本中提取(entity, relation, entity)三元组"""
    prompt = f"""Extract all entity-relationship triples from the text.
Output as JSON list of [subject, predicate, object].
Example: [["Elon Musk", "founded", "SpaceX"]]

Text: {text}"""

    # 实际调用:
    # response = openai.ChatCompletion.create(
    #     model="gpt-4o-mini",
    #     messages=[{"role": "user", "content": prompt}],
    #     response_format={"type": "json_object"}
    # )
    # return json.loads(response.choices[0].message.content)

    # 模拟返回（实际使用时替换为LLM输出）
    # 这里手动标注以展示完整管线
    return []

# 手动标注三元组（模拟LLM提取结果）
all_triples = [
    # Document 1
    ["Elon Musk", "founded", "SpaceX"],
    ["SpaceX", "developed", "Falcon 9"],
    ["SpaceX", "developed", "Starship"],
    ["SpaceX", "headquartered in", "Hawthorne"],
    # Document 2
    ["Elon Musk", "acquired", "Twitter"],
    ["Elon Musk", "renamed", "Twitter→X"],
    ["Twitter", "headquartered in", "San Francisco"],
    ["Elon Musk", "owns", "Tesla"],
    # Document 3
    ["Martin Eberhard", "founded", "Tesla"],
    ["Marc Tarpenning", "founded", "Tesla"],
    ["Elon Musk", "joined", "Tesla"],
    ["Elon Musk", "became CEO of", "Tesla"],
    ["Tesla", "produces", "Model 3"],
    ["Tesla", "produces", "Model Y"],
    # Document 4
    ["SpaceX", "launches", "Starlink"],
    ["Starlink", "is a", "satellite constellation"],
    ["Falcon 9", "launches", "Starlink satellites"],
    # Document 5
    ["Tesla", "acquired", "SolarCity"],
    ["Lyndon Rive", "founded", "SolarCity"],
    ["Peter Rive", "founded", "SolarCity"],
    ["Elon Musk", "was chairman of", "SolarCity"],
]

# ====== 3. 构建知识图谱 ======
G = nx.Graph()  # 无向图（为简化社区检测）

# 添加边，权重=关系出现次数
edge_weights = defaultdict(int)
for subj, pred, obj in all_triples:
    # 标准化实体名
    subj = subj.strip().lower()
    obj = obj.strip().lower()
    if subj != obj:  # 避免自环
        edge_weights[(subj, obj)] += 1  # 无向图用排序元组
    # 实际GraphRAG中保留有向图和关系类型

# 建图
for (e1, e2), weight in edge_weights.items():
    G.add_edge(e1, e2, weight=weight)

print(f"=== 知识图谱统计 ===")
print(f"节点 (实体) 数: {G.number_of_nodes()}")
print(f"边 (关系) 数: {G.number_of_edges()}")
print(f"平均度数: {sum(dict(G.degree()).values()) / G.number_of_nodes():.2f}")
print(f"实体列表: {list(G.nodes())}")

# ====== 4. 社区检测 (使用Louvain近似Leiden, networkx原生支持) ======
from networkx.algorithms.community import louvain_communities

communities = louvain_communities(G, weight='weight', seed=42)
communities = [list(c) for c in communities]  # 转为列表

print(f"\n=== 社区检测结果 ({len(communities)} 个社区) ===")
for i, comm in enumerate(communities):
    print(f"  社区 {i+1}: {comm}")

# ====== 5. 生成社区摘要 (LLM调用) ======
def generate_community_summary(community_entities: list) -> str:
    """对给定社区的实体列表生成摘要"""
    # 实际实现: 收集社区内所有三元组，组装prompt，调用LLM
    prompt = f"Summarize these related entities: {community_entities}"
    # response = openai.ChatCompletion.create(
    #     model="gpt-4o-mini",
    #     messages=[{"role": "user", "content": prompt}]
    # )
    # return response.choices[0].message.content
    return f"[摘要] 这些实体围绕 {community_entities[0] if community_entities else 'N/A'} 形成关联群组"

# 为每个社区生成摘要
print(f"\n=== 社区摘要 ===")
for i, comm in enumerate(communities):
    # 找出社区内最重要的实体（度中心性）
    subgraph = G.subgraph(comm)
    if len(subgraph) > 0:
        centrality = nx.degree_centrality(subgraph)
        top_entity = max(centrality, key=centrality.get)
    else:
        top_entity = comm[0] if comm else "N/A"

    summary = f"社区核心: {top_entity}, 涉及 {len(comm)} 个相关实体"
    print(f"  社区 {i+1}: {summary}")

# ====== 6. 模拟全局查询 (Map-Reduce) ======
def global_query(question: str, communities: list) -> str:
    """模拟GraphRAG的Map-Reduce全局查询"""
    # MAP阶段: 每个社区生成局部答案
    local_answers = []
    for i, comm in enumerate(communities):
        local_answer = f"[社区{i+1}] 关于'{question}'的相关信息：{comm}"
        local_answers.append(local_answer)

    # REDUCE阶段: 聚合所有局部答案
    # 实际实现中按重要性排序、截断、调用LLM聚合
    aggregated = f"综合 {len(local_answers)} 个社区的信息：\n" + \
                  "\n".join(local_answers[:3])  # 简化：取前3个
    return aggregated

question = "What companies and projects is Elon Musk involved with?"
answer = global_query(question, communities)
print(f"\n=== 全局查询示例 ===")
print(f"问题: {question}")
print(f"答案 (Map-Reduce聚合):\n{answer}")

# ====== 7. 图可视化 (简单文本形式) ======
print(f"\n=== 知识图谱邻接关系 ===")
for node in sorted(G.nodes()):
    neighbors = list(G.neighbors(node))
    if neighbors:
        print(f"  {node} -- {neighbors}")
```

## 自测三层 🎓
**L1 复述**：GraphRAG的四步离线索引流程是什么？局部查询和全局查询的区别是什么？Leiden算法的作用是什么？
**L2 解释**：为什么向量RAG在全局问题上甚至比"全文摘要"表现更差？从信息论角度看，碎片化检索在回答全局问题时为什么会产生误导？社区检测为什么需要分层（多层粒度）？
**L3 应用**：你的团队要用GraphRAG为10000篇内部技术文档构建"研发资产图谱"（哪些团队做哪些项目、用了什么技术栈、产出什么组件）。设计实施计划，包括：(1) 实体类型定义（项目/团队/技术/组件/API）；(2) 分块策略（技术文档的结构特点决定chunk方式）；(3) 成本估算（10000篇×~500 words/篇，LLM调用数）；(4) 更新策略（新项目/新组件每月增加）——增量更新还是全量重建？(5) 评估方案：设计5个测试问题（2个精确+3个全局），对比纯向量RAG和GraphRAG的表现。

📅 知识时间锚：2024-04（GraphRAG论文发布），GraphRAG的图+RAG混合架构开启了企业知识库从"搜索"到"理解"的范式升级。同期工作HippoRAG和LightRAG从不同路径（个性化PageRank/轻量图遍历）验证了图索引对RAG的增益。