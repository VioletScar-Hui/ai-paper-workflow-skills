---
tags: [论文笔记, 检索增强生成, Agent架构, 长记忆, 工具调用]
paper_id: "133"
filename: "133 - Is Grep All You Need - How Agent Harnesses Reshape Agentic Search.pdf"
authors: "Sahil Sen, Akhil Kasturi, Elias Lumer, Anmol Gulati, Vamse Kumar Subbiah (PwC US)"
year: 2026
笔记层级: 骨干
复核日期: 2026-07-04
---

# Grep vs 向量检索：Agent 搜索的框架依赖性

📄 **原文 PDF**：[[RAW/133 - Is Grep All You Need - How Agent Harnesses Reshape Agentic Search.pdf]]

## PM 速判 > 一句话

> 在 Agent 长记忆场景（LongMemEval-S 116题），grep 准确率系统性高于向量检索（Claude Opus 4.6 + Chronos + grep inline = 93.1% vs vector inline = 83.6%），但更重要的发现是 Agent 框架（Harness）的影响超过检索方法本身——同一模型 GPT-5.4 在同一检索方法下，换框架准确率从 89.7% 跌到 55.2%（差 34.5pp）。这意味着"检索效果"报告如果不写 Harness 和工具交付路径，就等于没报告。

## 双层费曼

> **给 CEO**：给 Agent 系统加"记忆搜索"功能时，不要默认去买向量数据库。这项研究发现：在精确事实查找（日期、名字、数字）场景中，传统的 grep（关键词搜索）比向量搜索更准；而且你选什么 Agent 框架（Chronos/Claude Code/Codex CLI）比选什么搜索技术影响更大——换个框架准确率能差 34.5%。

> **给工程师**：在 116 题 LongMemEval-S 长对话记忆基准上，实验 1（全因子设计：4 Harness × 5 模型 × 2 检索方法 × 2 交付方式）发现 (a) inline grep 在所有 Harness-模型对上均优于 inline vector；(b) 同一模型（Claude Opus 4.6）从 Chronos 换到 Claude Code CLI，grep inline 从 93.1% 跌到 76.7%（差 16.4pp）；(c) file-based 交付方式对弱模型是灾难（Claude Haiku 4.5 + Claude Code + grep file-based = 37.1%）。实验 2（渐进增加干扰会话 s5→s30→full）发现 grep 和 vector 的优势顺序随 Harness 和噪声量交叉——不存在"一个永远更好"的结论。核心洞察：Agent 场景中检索 ≠ 独立管道评测，而是"检索+编排"联合体。

## 问题域定位

**根本约束**：传统 RAG 评测假设"固定查询→检索→拼接→LLM回答"的管道。但在 Agent 场景中，模型自主决定搜什么、发多少查询、判断结果是否足够、是否需要重搜——检索策略和 Agent 能力不可分离。而此前文献缺乏对检索策略 × Agent 架构 × 工具交付方式的系统交互分析。

**之前卡点**：
- IR 社区大量评测 BM25 vs Dense Retrieval，但在隔离管道中做的
- Agent 评测（如 SWE-bench）报告整体分数，不区分检索贡献和推理贡献
- "用向量数据库做 Agent 记忆"成为默认选择，但缺乏与 grep 在相同 Agent 循环中的 head-to-head 对比

**路线开启**：证明了 (a) Harness 架构是检索效果的主导变量，(b) grep 在精确事实检索的 Agent 场景中是合法首选，(c) Agent 检索评测必须报告检索方法+Harness+工具交付路径三个维度。

**路线关闭**：结论限定在长对话记忆 QA（LongMemEval 的分布：知识更新/多会话/用户偏好/时序推理）。在语义模糊查询、多语言、代码语义搜索、科学文献合成等场景中，grep 与向量的相对优势可能翻转。论文明确声明"否定了 grep 在一般意义上打败向量"这一结论。

## 核心机制

```
┌──────────────────────────────────────────────────────────────────┐
│              传统评测 vs Agent 场景：检索角色的本质差异              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  传统 RAG 评测管道 (已过时):                                       │
│    固定查询 ──→ 检索器 ──→ top-k 拼入提示 ──→ LLM 回答            │
│    检索被视为独立组件，排行榜比较"哪个检索器召回率更高"               │
│                                                                  │
│  Agent 场景 (实际运行):                                            │
│    LLM 决定搜什么 ──→ 工具调用 ──→ 得到结果 ──→ 判断是否足够        │
│         ↑                                              │         │
│         └──────── 不够，构建新查询，再搜 ─────────────────┘         │
│                                                                  │
│  ┌────────────────────── 三个变量交互 ──────────────────────────┐ │
│  │                                                              │ │
│  │  检索方法:  grep (regex精确) vs vector (embedding语义)         │ │
│  │                                                              │ │
│  │  Agent Harness:                                              │ │
│  │    Chronos (定制, 类别条件提示, 动态策略)                       │ │
│  │    Claude Code CLI (provider原生, bash工具, sandbox)          │ │
│  │    Codex CLI (OpenAI原生)                                     │ │
│  │    Gemini CLI (Google原生)                                    │ │
│  │                                                              │ │
│  │  工具交付方式:                                                 │ │
│  │    Inline: 结果直接注入上下文窗口                                │ │
│  │    File-based: 结果写文件, Agent自行打开读取                     │ │
│  │                                                              │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  grep 在 Agent 场景中的结构性优势:                                   │
│    LLM 动态构建 grep 命令（关键词/flags/目标文件）                    │
│    → "检索策略" 成为 "Agent 能力" 的延伸，而非固定 API               │
│    → 无需嵌入模型、无需向量索引基础设施、无索引延迟                     │
│    → 在精确匹配（日期、专有名词、数字）上天然胜出                       │
│                                                                  │
│  file-based 的隐藏成本:                                            │
│    省了上下文窗口，但要求 Agent 可靠完成 "定位→打开→整合→重试" 循环    │
│    → 这本质上是工具使用能力的压力测试                                 │
│    → 弱模型在某个步骤失败 → 检索质量再好也没用                         │
└──────────────────────────────────────────────────────────────────┘
```

**实验设计（全因子）**：
- 实验 1: 4 Harness × 5 模型 × (grep/vector) × (inline/file-based) = 80 条件
- 实验 2: 对每个 Harness-模型对，在 5 种噪声级别（s5/s10/s20/s30/full）下分别测 grep-only 和 vector-only
- 评测: GPT-4o 作为 grader，按类别条件评分（允许 off-by-one 时序容差等）

## 设计决策解剖

| 决策点 | 选择 | 被放弃方案 | 为什么 | 何时失效 |
|--------|------|-----------|--------|---------|
| 评测基准 | LongMemEval-S（116题长对话记忆） | SWE-bench（代码检索）/ 通用 QA 基准 | LongMemEval 天然具备 oracle session + distractor session 结构，适合测"从噪声中检索事实"；问题覆盖 6 个类别（知识更新/多会话/偏好/事实/时序推理） | 当任务不依赖精确事实提取（如观点总结、语义聚类），LongMemEval 的分布会系统性低估向量检索的价值 |
| grep 实现 | 结构化事件文件上的 bash grep + Chronos 的类别条件提示 | BM25/ElasticSearch（也是词法但更复杂） | grep 是 Agent 可直接调用的最简词法工具——LLM 天然会写 grep 命令；BM25 需要额外索引 | 当语料大到无法全部扫描时（如 10GB+），grep 的线性扫描成本不可接受 |
| 工具交付对比 | inline（stdout/工具消息）vs file-based（写文件+Agent读取） | 只测 inline（大多数论文的做法） | file-based 是生产系统中常见的"省上下文窗口"策略，但其间接性引入的工具使用压力此前未被系统研究 | 当 Agent 上下文窗口大到足以容纳所有检索结果时，file-based 的间接性代价无收益 |
| 干扰会话生成 | 从 LongMemEval 池中随机采样无关会话 | 合成固定噪声模板 | 随机采样保持生态效度——真实长记忆中干扰的分布不是均匀的 | 随机采样的 variance 使得 s5-s30-full 的"单调性"不可预期（论文承认中段峰值可能来自有利的采样） |

## 成本与量级

| 维度 | 数据 |
|------|------|
| 数据集 | LongMemEval-S: 116 题，每题包含 1+ oracle session + 可变 distractors；6 个问题类别 |
| 实验 1 条件数 | 4 Harness × 5 模型 × 2 检索方法 × 2 交付方式 = 80 条件（部分模型在部分 Harness 上不可用，实际略少） |
| 模型 | Claude Opus 4.6, Claude Haiku 4.5, GPT-5.4, Gemini 3.1 Pro, Gemini 3.1 Flash-Lite |
| 实验 2 噪声级别 | 5 级（s5/s10/s20/s30/full），每个条件 116 题 |
| API 成本 | 未报告；每个条件 116 次 Agent 循环，每次循环可能多次工具调用 |
| 关键数字源 | Table 1（全因子准确率）, Tables 2-3（噪声实验）, Table 4（per-category accuracy） |

## 证据审计

**实验公平性**：公平性较强。所有条件使用相同的 116 题、相同的 grader（GPT-4o）、相同的 grader prompt 模板和解码设置——确保跨条件差异来自 Agent 管线而非评测噪声。但 Chronos 使用了类别条件提示（给不同问题类别不同的系统指令），而 CLI harness 使用通用提示——这引入了"定制提示 vs 通用提示"的混淆变量。

**最强证据**：
- **inline grep > inline vector 对于所有 Harness-模型对都成立**（Table 1）：不是某些条件偶尔赢，而是全覆盖。最高值 Claude Opus 4.6 + Chronos + grep inline = 93.1%。
- **GPT-5.4 极端案例**：Chronos + grep inline = 89.7%，Codex CLI + grep file-based = 55.2%，差 34.5pp——同一模型、同一检索方法，仅 Harness+交付方式不同。
- **Claude Haiku 4.5 + Claude Code + grep file-based = 37.1%**（vs Chronos + grep inline = 83.6%）：弱模型在 file-based 模式下基本失效。
- **实验 2 的跨点验证**：噪声增加时 grep 和 vector 的优势顺序会交叉（Chronos Opus: vector 在 s5-s20 领先，grep 在 s30 反转；Gemini CLI Pro: vector 在所有噪声级别领先）——排除了"某个方法在所有条件下都更好"的简化结论。

**最可疑数字及原因**：
- **93.1% 是单一条件的最优值**，不是平均：Chronos 是作者自研框架，其类别条件提示和动态策略可能对该基准过拟合（作者也承认了这一点）。
- **实验 2 中 Chronos Opus grep 在 s20=90.5%, s30=85.3%, full=89.7% 的非单调模式**：论文解释为干扰会话的随机重采样导致的"有利/不利干扰集"，但这种非单调性削弱了"grep 随噪声增加而稳定下降"的叙事。
- **Codex vector 在实验 2 中数据不完整**：论文明确标注 "Codex vector row is complete only at full; intermediate configurations are pending"，使跨 CLI 的噪声扩展对比不完整。

**审稿补充**：论文在 Limitations 中明确声明 "We do not claim that grep 'beats' vector in general, only that it can win end-to-end under the task distribution and corpora we study"——这是一个高质量的学术诚实。

**最小复现**：单个模型 × 单个 Harness × 116 题 × 两个检索方法 = 约 232 次 Agent 循环。如果使用 Chronos 框架（开源? 未知）+ Claude API，成本约 $50-200（取决于模型和迭代轮数）。

## 可复用点 + 供应商拷问清单

**可复用点**：
1. Agent 搜索评测必须同时报告 Harness 架构 + 工具交付路径（不能只报告"用 BM25"——这和只报告"用 PyTorch"不报告模型架构一样不完整）
2. 强模型可用 file-based（省上下文），弱模型必须用 inline（稳定性优先）的工具交付选型原则
3. "先测 grep 方案"作为 Agent 长记忆设计的起点原则——在精确事实场景中很可能已经足够好
4. 工具调用交付方式不是实现细节——它是上下文工程的核心决策

**供应商拷问清单**（评估 Agent 长记忆方案时）：
- [ ] 你们的检索评测是在什么 Harness 下做的？是独立管道还是 Agent-in-the-loop？
- [ ] 如果我切换 Harness（从自研框架切到 Claude Code CLI），检索效果会下降多少？
- [ ] 你们的工具结果是 inline 注入还是 file-based？对弱模型有单独适配吗？
- [ ] 你们的 "grep" 方案在多少噪声级别下测试过？随噪声增加的效果退化曲线是什么形状？
- [ ] 你们的评测是否报告了系统提示（system prompt）？不同提示对检索策略的引导效果差异有多大？

## 关联网络

- [[Wiki/论文笔记/03_Agent系统设计/122_代码作为Agent框架调查]] -- Harness 框架对检索结果的影响印证了本文"Harness 影响 > 检索方法"的核心发现
- [[Wiki/论文笔记/09_长上下文记忆/124_MeMo记忆即模型]] -- 互补的记忆方案：MeMo 用 Memory 模型内化知识，本文用 grep/向量外部检索，两者可组合
- [[Wiki/概念/05_记忆与检索/RAG检索增强生成]] -- 本文是 RAG 在 Agent 场景中重新评估的直接实证研究
- [[Wiki/概念/05_记忆与检索/Agent长期记忆]] -- grep vs 向量检索是 Agent 长期记忆实现的两种核心路径对比
- [[Wiki/概念/04_Agent框架/Harness设计模式]] -- 本文证明 Harness 选择的影响超过检索算法本身

**冲突/印证**：
- **印证**：[[122_代码作为Agent框架调查]] 的核心论点"框架设计影响 Agent 表现"→ 本文用精确数字（34.5pp 差距）为其提供了检索维度的定量证据。
- **印证**：传统的 BEIR 基准中 BM25 在零样本设置下常优于早期稠密检索模型 → 本文在 Agent 场景中复现了类似的"简单词法基线不弱"现象，但多了 Harness 交互这一层。
- **潜在冲突**：本文发现 grep 在精确事实任务中系统占优，但现代 RAG 系统的趋势是"混合检索（lexical+dense）+ 重排序"——文中未测这个 hybrid 方案（实验设计限定为 grep-only 和 vector-only 的对比），混合方案的表现是未知的。

## 动手练习

```python
"""
同一代码库分别用关键词检索和向量检索回答相同问题，对比效果
模拟实验 1 的核心对比：grep (关键词) vs vector (语义) 在同一 Harness 下
"""
import json, re, math
from collections import Counter

# ===== 1. 构造模拟代码库 =====
# 相当于 LongMemEval 的对话历史被替换为了代码文件
CODEBASE = {
    "auth.py": """
class UserAuth:
    def login(self, username: str, password: str) -> str:
        # 使用 bcrypt 验证密码哈希
        token = jwt.encode({"user": username}, SECRET_KEY, algorithm="HS256")
        return token
    def logout(self, token: str) -> None:
        cache.delete(f"session:{token}")
""",
    "payment.py": """
class PaymentGateway:
    def charge(self, amount_cents: int, card_token: str) -> dict:
        # 调用 Stripe API 扣款，超时设为30秒
        response = stripe.Charge.create(amount=amount_cents, currency="usd", source=card_token)
        return {"id": response.id, "status": "succeeded"}
    def refund(self, charge_id: str, amount_cents: int = None) -> dict:
        return stripe.Refund.create(charge=charge_id, amount=amount_cents)
""",
    "cache.py": """
class RedisCache:
    def __init__(self, host: str = "localhost", port: int = 6379):
        self.client = redis.Redis(host=host, port=port, decode_responses=True)
    def get(self, key: str): return self.client.get(key)
    def set(self, key: str, value: str, ttl: int = 3600): self.client.setex(key, ttl, value)
    def delete(self, key: str): self.client.delete(key)
""",
    "config.py": """
SECRET_KEY = "<SECRET_KEY>"
STRIPE_API_KEY = "<STRIPE_API_KEY>"
DATABASE_URL = "postgresql://<user>:<password>@db.internal:5432/main"
REDIS_HOST = "redis.internal"
CACHE_TTL = 3600
MAX_RETRIES = 3
""",
}

# ===== 2. 检索实现 =====
def grep_search(query: str, corpus: dict) -> list[tuple[str, float]]:
    """关键词检索：模拟 grep —— 精确匹配 + 模糊子串匹配"""
    results = []
    query_lower = query.lower()
    keywords = [kw.strip() for kw in query_lower.replace("?", "").split()]

    for filename, content in corpus.items():
        content_lower = content.lower()
        score = 0.0

        # 精确子串匹配（高权重）
        if query_lower in content_lower:
            score += 3.0

        # 关键词匹配
        for kw in keywords:
            count = content_lower.count(kw)
            if count > 0:
                score += count * 0.5
            # 额外奖励：关键词出现在函数名/类名/变量名附近
            def_lines = [l for l in content.split("\n") if kw in l.lower()]
            score += len(def_lines) * 0.3

        if score > 0:
            results.append((filename, round(score, 2)))

    results.sort(key=lambda x: x[1], reverse=True)
    return results


def vector_search(query: str, corpus: dict) -> list[tuple[str, float]]:
    """向量检索：用简化的 TF-IDF 余弦相似度模拟语义检索"""
    # 构建全局词汇表
    all_words = set()
    for content in corpus.values():
        all_words.update(re.findall(r"\w+", content.lower()))
    vocab = {w: i for i, w in enumerate(all_words)}

    def tfidf_vector(text: str) -> list[float]:
        words = re.findall(r"\w+", text.lower())
        word_counts = Counter(words)
        total = max(len(words), 1)
        vec = [0.0] * len(vocab)
        for word, count in word_counts.items():
            if word in vocab:
                tf = count / total
                # IDF: 包含该词的文件数越少，权重越高
                doc_count = sum(1 for c in corpus.values() if word in c.lower())
                idf = math.log((len(corpus) + 1) / (doc_count + 1)) + 1
                vec[vocab[word]] = tf * idf
        # L2 归一化
        norm = math.sqrt(sum(v * v for v in vec)) + 1e-10
        return [v / norm for v in vec]

    query_vec = tfidf_vector(query)
    results = []
    for filename, content in corpus.items():
        doc_vec = tfidf_vector(content)
        # 余弦相似度
        dot = sum(a * b for a, b in zip(query_vec, doc_vec))
        if dot > 0.01:
            results.append((filename, round(dot, 4)))

    results.sort(key=lambda x: x[1], reverse=True)
    return results

# ===== 3. 评测 =====
QUESTIONS = [
    ("登录时密码用什么算法验证？", "auth.py"),          # 精确事实
    ("退款接口支持部分退款吗？", "payment.py"),          # 精确事实
    ("Redis 缓存的默认 TTL 是多少秒？", "cache.py"),     # 精确数字
    ("生产环境的密钥叫什么名字？", "config.py"),          # 专有名词
    ("数据库连接的用户名是什么？", "config.py"),          # 精确事实
    ("如何安全地管理 session？", "auth.py"),            # 概念问题
]

print("=" * 70)
print("  同一代码库: grep (关键词) vs vector (语义) 检索对比")
print("=" * 70)

grep_correct = 0
vec_correct = 0

for query, expected_file in QUESTIONS:
    print(f"\n查询: {query}")
    print(f"  期望文件: {expected_file}")

    # grep 检索
    grep_results = grep_search(query, CODEBASE)
    grep_top = grep_results[0][0] if grep_results else "NOT FOUND"
    grep_hit = grep_top == expected_file
    print(f"  [grep]    Top-1: {grep_top:15s} | {'命中' if grep_hit else '未命中'}")
    if grep_hit:
        grep_correct += 1

    # 向量检索
    vec_results = vector_search(query, CODEBASE)
    vec_top = vec_results[0][0] if vec_results else "NOT FOUND"
    vec_hit = vec_top == expected_file
    print(f"  [vector]  Top-1: {vec_top:15s} | {'命中' if vec_hit else '未命中'}")
    if vec_hit:
        vec_correct += 1

    # 对比分析
    if grep_hit and not vec_hit:
        print(f"  → grep 胜: 精确关键词匹配优势")
    elif vec_hit and not grep_hit:
        print(f"  → vector 胜: 语义相似度优势（grep关键词未命中）")
    elif grep_hit and vec_hit:
        print(f"  → 平局: 两者都命中")
    else:
        print(f"  → 两者都未命中: 可能需要混合检索或更好的查询构建")

print(f"\n{'='*70}")
print(f"  总计: grep {grep_correct}/{len(QUESTIONS)} | vector {vec_correct}/{len(QUESTIONS)}")
print(f"  观察: 精确事实查询倾向 grep；语义概念查询倾向 vector")

# ===== 4. 核心启示 =====
print(f"\n[启示] 如论文所述，结果取决于:")
print(f"  1. 查询类型分布（精确事实 vs 语义概念）")
print(f"  2. Agent 构建查询的能力（这个模拟中我们用了固定查询）")
print(f"  3. 在真实 Agent 场景中，模型可以迭代 refine 查询 ——")
print(f"     grep 的动态命令构建优势在这里的简化版中未被测量")
```

## 自测三层

**L1 记忆**：实验 1 中 inline grep 在所有 Harness-模型对上均优于 inline vector，这种全覆盖结论意味着什么？GPT-5.4 在两个 Harness 下的 grep inline 准确率分别是什么？

**L2 理解**：为什么 file-based 交付方式对弱模型是灾难但对强模型影响较小？这与 SDB（随机-确定边界，Paper 128）中的哪个部件有关联？

**L3 延伸**：如果你的业务场景是"搜索内部 Wiki 和设计文档回答技术问题"（既需要精确匹配 API 名称，也需要语义理解架构意图），仅靠 grep 或仅靠向量都不够。设计一个混合策略：什么时候用 grep、什么时候用向量、什么时候需要 rerank？如何在 Agent 的tool-calling 循环中自动路由？用本文的 Harness 影响洞察来评估你的设计是否会因框架选择而失效。

---

知识时间锚 2026（PwC US）。截至 2026 年，这是唯一的对 Agent 场景中检索策略 × Harness 架构 × 工具交付方式进行全因子系统对比的实证研究。核心教训：Agent 评测必须报告 Harness 和工具交付路径——"我们用了向量搜索"和没说一样不完整。

[[Wiki/论文笔记/03_Agent系统设计/122_代码作为Agent框架调查]] [[Wiki/概念/05_记忆与检索/Agent长期记忆]] [[Wiki/概念/05_记忆与检索/RAG检索增强生成]] [[Wiki/概念/04_Agent框架/Harness设计模式]]
