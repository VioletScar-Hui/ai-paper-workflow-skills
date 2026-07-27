---
tags: [论文笔记, RoPE, 旋转位置编码, 位置嵌入, 基础论文, 追一科技]
paper_id: "62"
filename: "62 - RoFormer - Enhanced Transformer with Rotary Position Embedding.pdf"
authors: "Jianlin Su, Yu Lu, Shengfeng Pan, Ahmed Murtadha, Bo Wen, Yunfeng Liu (Zhuiyi Technology)"
year: 2021
成熟度: ✅
笔记层级: 骨干
复核日期: 2026-07-04
---

# RoFormer：带旋转位置嵌入的增强 Transformer

📄 **原文 PDF**：[[RAW/62 - RoFormer - Enhanced Transformer with Rotary Position Embedding.pdf]]

## PM 速判
> 一句话：RoPE（旋转位置编码）通过将位置信息编码为向量在复数平面上的旋转，用绝对位置编码实现了相对位置的效果——点积结果天然只依赖位置差。已成为 LLaMA/Qwen/ChatGLM/Mistral 等所有主流开源 LLM 的事实标准位置编码方案。

## 双层费曼
> **给 CEO 的一句话**：Transformer 模型天生"看不见"句子中词的顺序——"猫吃鱼"和"鱼吃猫"对它是一样的。以前的方法要么给每个位置贴个编号（像给队列里的人发号码牌），要么复杂地计算词与词的"相对距离"。RoPE 的做法特别优雅：把词向量在数学空间中"旋转"，转的角度由词的位置决定——位置越远的词，对信息传递的"衰减"就越大。这个设计让今天所有主流大模型都能处理几万字的超长文本。

> **给工程师的一段话**：RoPE 对 attention 的 Q 和 K 向量在每对维度 (2i, 2i+1) 上施加 2D 旋转，旋转角为 m * theta_i，其中 m 是 token 的绝对位置，theta_i = 10000^{-2i/d} 是第 i 对维度的旋转频率（高频维度快速旋转，低频维度缓慢旋转）。数学上：f_{q,k}(x_m, m) = (W_{q,k} x_m) e^{im*theta}（复数表示），点积 q_m^T k_n 可化简为 g(x_m, x_n, m-n)——仅依赖相对位置差。与之前的相对位置编码（Transformer-XL、T5 bias）相比：(1) 不需要修改 attention 计算图，(2) 直接与线性 attention 兼容，(3) 天然具有长程衰减特性。LLaMA 系列直接采用 theta_i = 10000^{-2i/d} 的原始配置，可通过调整 base（NTK-aware scaling）或缩放频率（YaRN）实现上下文窗口外推。

## 问题域定位
- **回应什么根本约束？** Transformer 的置换等变性——self-attention 本身对 token 位置不敏感，需要额外的位置编码来注入顺序信息。更具体地说：如何设计一种位置编码，使得 (1) 显式编码位置信息，(2) 注意力分数天然依赖相对位置（m-n），(3) 实现简单、计算开销小，(4) 可以外推到训练时未见过的更长序列。
- **之前卡在哪？** 绝对位置编码（BERT/GPT 的 learned/sinusoidal position embedding）：只能表示训练时见过的位置，无法外推到更长的序列。相对位置编码（Transformer-XL、Shaw et al.）：需要修改 attention 计算——在 QK^T 之前加上 position-to-position 和 position-to-content 的 bias 项，实现复杂且与高效的 attention kernel（如 flash attention）不兼容。T5 的 relative bias：放弃了显式的位置-内容交互，仅用可学习的相对距离 bias，表达能力有限。核心瓶颈：如何在"编码绝对位置"和"表达相对关系"之间找到统一的数学形式。
- **开启/关闭了哪条技术路线？** 开启了"用旋转操作统一绝对和相对位置编码"的路线——RoPE 证明了旋转是连接绝对位置和相对位置的数学桥梁。后续的线性 attention 变体（如 CosFormer、Linear RoPE）、长度外推方法（NTK-aware scaling、YaRN、ReRoPE）都建立在这个框架上。关闭了"绝对和相对位置编码必须二选一"的二元思维。

## 核心机制

```
RoPE 旋转位置编码全景图

【核心思想】在 Q/K 向量的每对维度上施加 2D 旋转，用绝对位置做旋转角度，
           但点积结果只依赖相对位置差


【第一步：将 d 维向量视为 d/2 个 2D 子空间的拼接】

  d 维 Q/K 向量:
  [x0, x1, x2, x3, x4, x5, ..., x_{d-2}, x_{d-1}]
   |    |    |    |    |    |          |        |
   v    v    v    v    v    v          v        v
  (0,1)    (2,3)    (4,5)   ...   (d-2, d-1)    ← d/2 对


【第二步：对每对维度施加旋转，角度由位置 m 和频率 theta_i 决定】

  第 i 对维度 (x_{2i}, x_{2i+1}) 在位置 m 处:

    [ cos(m*theta_i)   -sin(m*theta_i) ] [ x_{2i}   ]
    [ sin(m*theta_i)    cos(m*theta_i) ] [ x_{2i+1} ]

    其中 theta_i = 10000^{-2i/d}
    i=0       → theta_0 = 1.0        (最低频, 旋转最慢)
    i=d/2-1   → theta_max = 1/10000  (最高频, 旋转最快)

  旋转后: x' = 原来的向量 在 2D 平面上逆时针旋转 m*theta_i 弧度


【第三步：复数视角（更优雅的等价表述）】

  将每对 (x_{2i}, x_{2i+1}) 视为复数 z_i = x_{2i} + i*x_{2i+1}
  旋转操作等价于: z'_i = z_i * e^{i*m*theta_i}
  即: 复数乘法 = 旋转 + 缩放（旋转矩阵的行列式为1，纯旋转无缩放）


【第四步：Q 和 K 分别旋转，点积产生相对位置】

  Q: q_m = R_m * (W_q * x_m)    旋转 m*theta 弧度
  K: k_n = R_n * (W_k * x_n)    旋转 n*theta 弧度

  attention score = q_m^T * k_n
                  = (R_m * q)^T * (R_n * k)
                  = q^T * R_m^T * R_n * k
                  = q^T * R_{n-m} * k          ← 关键! 旋转的减法性质
                  = q^T * R_{Δ} * k            ← Δ = n-m, 相对位置!

  因为 R_m^T * R_n = R_{n-m}（旋转矩阵的可加性）
  所以点积结果只依赖相对位置 Δ，不依赖绝对位置 m 和 n


【第五步：长程衰减的天然性质】

  对不同频率的维度对:
    theta_i 大 (高频): 旋转快 → 较小 Δ 就产生较大相位差 → 短期依赖
    theta_i 小 (低频): 旋转慢 → 需要大 Δ 才能有显著相位差 → 长期依赖

  多频率叠加后: attention score 随 |Δ| 增大而自然衰减
  —— 不需要手动设置衰减函数
```

**关键公式（复数形式）**：
```
q = f_q(x_m, m) = (W_q x_m) * e^{i*m*theta}    [逐元素复数乘法]
k = f_k(x_n, n) = (W_k x_n) * e^{i*n*theta}

q^T k = Re[ q^* * k ]
      = Re[ sum_j (W_q x_m)_j^* * (W_k x_n)_j * e^{i*(n-m)*theta_j} ]
      = g(x_m, x_n, n-m)                         [只依赖相对位置!]
```

## 设计决策解剖

| 决策点 | 他们的选择 | 被放弃的方案 | 为什么 | 这个选择何时失效 |
|--------|----------|------------|--------|---------------|
| 位置编码的作用对象 | 只对 Q 和 K 施加旋转，V 不旋转 | 对 Q/K/V 都旋转，或只对 K 旋转 | attention score 只由 Q 和 K 决定（score = QK^T），V 的信息聚合不涉及 token 间的交互。对 V 旋转会增加计算但不会影响 attention pattern。只旋转 Q/K 是最小必要干预 | 如果扩展到 cross-attention（如 encoder-decoder 模型），可能需要为 encoder 和 decoder 选择不同的位置编码策略——但只操作 Q/K 的原则仍然适用 |
| 旋转频率的设计 | theta_i = 10000^{-2i/d}，完全继承自原始 sinusoidal position encoding | 可学习的频率参数，或均匀频率分布 | 10000 的几何级数分布在实践中效果最好，且有天然的"多尺度"解释：高频捕捉局部依赖，低频捕捉全局依赖。可学习频率可能在小数据上 overfit，在长序列外推时泛化差 | 当序列长度远超训练长度（如从 2K 外推到 32K+）时，低频维度的旋转角度在训练范围内变化太小（几乎不转），模型没有学到此频率下的位置区分度。NTK-aware scaling 通过增大 base（如 10000→500000）来解决，本质上改变了频率分布 |
| 实现方式 | 在 attention 计算前，对 Q 和 K 的隐层做逐元素的旋转矩阵乘法 | 修改 attention 计算（在 QK^T 上直接加相对位置 bias，如 T5/ALiBi） | 预计算旋转矩阵然后逐元素乘加是标准线性代数操作，可以直接被 GPU 的 GEMM kernel 高效执行。而且不修改 attention 的计算图意味着可以无缝使用 FlashAttention 等优化的 attention kernel | 当 d_model 非常小（<64）时，每对维度独立旋转的粒度太粗，位置信息的表示能力受限。但对于现代 LLM（d_model >= 4096），这个不是问题 |
| 相对位置的表达方式 | 隐式：通过旋转矩阵的减法性质 R_{n-m} = R_m^T R_n 在点积空间中自然表达 | 显式：直接在输入中添加可学习的相对位置 embedding（如 Shaw et al., Transformer-XL） | 隐式表达的优雅之处：(1) 不需要为每个相对距离存储单独的 embedding 向量，(2) 相对距离的表示是通过函数（三角函数）而非查表得到的，因此天然连续可导、支持任意距离值，(3) 不需要修改 attention 计算，与任何 attention kernel 兼容 | 如果任务需要学习"非常特定的"位置模式（如某些相对距离有特殊语义），可学习的相对位置 bias 可能比 RoPE 的固定函数形式更灵活。但实践中 RoPE 的多频率设计已经提供了足够的表达能力 |
| 与线性 attention 的兼容性 | RoPE 天然兼容——因为旋转是在 Q/K 向量上做线性变换（旋转是线性操作） | 需要在 attention 矩阵上加 bias 的方案（T5 bias, ALiBi）——这些方案与线性 attention 的核分解（phi(Q)phi(K)^T）不兼容 | 线性 attention 把 exp(QK^T) 近似为 phi(Q)phi(K)^T 以降低复杂度。RoPE 直接在 Q/K 上操作，所以 phi(Q_rotated)phi(K_rotated)^T 仍然可以分解——而 bias 方案需要在分解后的矩阵上加 bias，没有数学上合理的分解方式 | 当使用不需要线性复杂度的标准 softmax attention 时，这个优势不显现。但对于长序列场景（linear attention, Mamba 类状态空间模型），RoPE 的线性兼容性是关键优势 |

## 成本与量级
- **训练成本**：RoPE 本身不增加训练成本——旋转矩阵可以在前向传播中实时计算（只是 sin/cos + 逐元素乘法），计算量 O(d*L) 远小于 attention 的 O(L^2*d)。预计算所有位置的 cos/sin 表只需 O(d*L_max) 的显存，极低。
- **推理成本**：同样可以忽略不计。旋转操作已经高度优化：在现代 GPU 上，sin/cos 查表 + 逐元素乘法在 attention 的 overhead 中占比 < 1%。
- **工程复杂度**：实现只需 ~20 行 PyTorch 代码。HuggingFace 的 LLaMA 实现中 RoPE 约 30 行。改 base 参数（从 10000 到 500000）来扩展上下文窗口只需改一个数字——这是 RoPE 相比其他方案的一大工程优势。

## 证据审计
- **实验设计公平吗？** 实验设计合理但规模有限。RoFormer 在 RoBERTa-Base 规模（125M 参数，12 层）的中文语料上训练，在 CLUE 中文 NLU 基准上评估。基线包括绝对位置编码（BERT 式的 learned PE 和 sinusoidal PE）和相对位置编码变体。公平方面：(1) 同数据、同训练步数、同模型结构的对比，(2) 覆盖了分类、匹配、序列标注等多种任务类型。不公平：(1) 中文基准仅 6 个任务，不如英文 GLUE 全面，(2) 未与当时最强的相对位置方案 DeBERTa 对比，(3) 实验规模太小——无法证明 RoPE 在大规模模型（1B+）上的优势。
- **最强证据**：长序列外推实验——在训练序列长度为 512 的模型上，测试序列长度从 512 扩展到 4096 时的困惑度（PPL）退化。RoPE 的 PPL 退化速度明显慢于绝对位置编码——在 4 倍训练长度时，RoPE PPL 约上升 2-3 点而绝对编码上升 10+ 点。更关键的是，这个实验设计直接测试了论文的核心 claim（相对位置带来外推能力），而非偶然达到的某个 SOTA 数字。
- **最可疑的数字**：CLUE 基准上的绝对提升——RoPE vs sinusoidal PE 的差距在某些任务上只有 0.2-0.5 个百分点。这些小差距在 125M 参数规模下可能来自随机性（如不同初始化种子）。论文没有报告多次运行的标准差。对于 0.2 个点的提升声称"超越"，审稿人会质疑统计显著性。
- **审稿人会要求补充**：(1) 多次随机种子运行的标准差和统计检验，(2) 在英文 GLUE 基准上的复现——中文单语种的结论可能无法推广，(3) 不同频率 base 参数（100, 1000, 10000, 100000）的消融实验——证明 10000 确实是最优的，(4) 在大规模（1B+）模型上的验证——证明 RoPE 能 scaling。
- **最小复现实验**：在 huggingface 上用 GPT-2 Small（117M）+ 替换位置编码（learned PE vs sinusoidal PE vs RoPE）在 WikiText-2 上做语言模型训练和长度外推测试。预算：单 A100，数小时。代理指标：不同测试长度下的 PPL。核心验证：RoPE 是否确实在训练长度之外 PPL 退化更慢。注：RoPE 后来在 LLaMA 等千亿参数模型上的大规模验证（虽然不是论文作者做的）是对 RoPE 有效性的最强背书。

## 可复用点
- **何时采用**：今天如果你从零训练 LLM，RoPE 是默认位置编码选择：(1) 自回归语言模型（GPT/LLaMA 风格），(2) 需要支持长度外推（从训练长度扩展到推理长度），(3) 需要使用 FlashAttention 等高效 kernel——RoPE 的简洁性使其与几乎所有 attention 优化兼容。如果你用现有预训练模型，不用担心——LLaMA/Qwen/ChatGLM/Mistral 已经内置 RoPE。
- **何时规避**：(1) 已经在用别人的预训练模型且不想改位置编码，(2) 只需要 encoder-only 固定长度任务（如 BERT 做分类）——绝对位置编码也够用且更简单，(3) 对"超超长"序列（>100K tokens）——RoPE 即使调 base 后也有外推极限，可能需要考虑 ALiBi（天然无长度限制）或 NoPE（完全不要位置编码，发现大模型可以隐式学习位置）。
- **供应商拷问清单**："你们模型的位置编码方案是什么？如果是 RoPE，base 参数是多少？""你们的上下文窗口扩展是用什么方法？NTK-aware scaling？YaRN？还是直接调 base？""RoPE 的外推极限是多少？4x 训练长度后 PPL 退化多少？""支持 LongRoPE 或 Self-Extend 吗？"

## 关联网络
- [[Wiki/论文笔记/01_LLM基础架构/51_AttentionIsAllYouNeed-Transformer原始论文]] — RoPE 替代了原始 Transformer 的 sinusoidal position encoding，后者也是用三角函数但加在输入 embedding 上（而非对 Q/K 做旋转），两者有共同的数学渊源（傅里叶分析）但效果差异巨大
- [[Wiki/论文笔记/01_LLM基础架构/54_Transformer-XL超越固定长度上下文的语言模型]] — Transformer-XL 首次提出相对位置编码（在 attention score 中加可学习的 relative bias），RoPE 可视为其"在向量空间中实现相对位置"的优雅替代方案
- [[Wiki/概念/01_架构技术/稀疏注意力与长上下文效率]] — RoPE 的外推能力是长上下文效能的基石——如果没有好的位置编码，稀疏注意力节省的计算量会被 PPL 退化吃掉
- **冲突/印证**：与 ALiBi（Attention with Linear Biases, Press et al. 2021）构成直接设计冲突——两者都追求"相对位置 + 外推能力"，但采取完全不同的路径。ALiBi 的做法：在 attention score 上加一个固定的、不可学习的线性衰减 bias（bias = -|m-n| * slope），0 参数，0 计算开销，天然无限长度外推。RoPE 的做法：用旋转编码相对位置，有少量计算开销，但保留了"内容-位置交互"（即 Q 的旋转角度会影响 attention，而 ALiBi 的 bias 完全不受内容影响）。实验结论：ALiBi 在极长序列（>10K）的外推上优于原始 RoPE，但 RoPE 在训练长度内的性能更好。工业界最终选择 RoPE（因为通过 NTK/YaRN 可以弥补外推差距，且"内容相关的位置编码"在大模型上展现出更好的表示能力）。**印证**：NoPE（No Positional Encoding, Kazemnejad et al. 2023）的研究反直觉地证明了大模型在没有任何显式位置编码的情况下也能隐式学到位置信息——这与 RoPE 的"显式编码"不矛盾，而是说明了位置信息有多条路径可以被学习（显式 + 隐式），而 RoPE 选择了最直接、最可控的那条。

## 动手练习（50-70行，可运行Python代码）
练习目标：用 NumPy 手写旋转位置编码（RoPE），可视化不同位置和不同频率维度的旋转角度，直观理解 RoPE 的"绝对位置旋转 + 相对位置点积"原理。

```python
"""
RoPE 动手练习：NumPy 手写旋转位置编码 + 可视化
练习目标：(1) 理解旋转矩阵如何在每对维度上编码位置信息
         (2) 可视化不同位置和频率的旋转角度
         (3) 验证点积结果只依赖相对位置差
环境：pip install numpy matplotlib
"""

import numpy as np
import matplotlib.pyplot as plt

# ============================================================
# 第 1 步：实现 RoPE 核心函数
# ============================================================
def compute_rope_frequencies(d, base=10000.0):
    """
    计算 RoPE 的旋转频率 theta_i = base^{-2i/d}
    返回 d/2 个频率值，每个对应一对维度
    """
    # 对每对维度 (2i, 2i+1) 计算 theta_i
    # i 从 0 到 d/2-1
    i = np.arange(d // 2)                    # [0, 1, 2, ..., d/2-1]
    theta = base ** (-2 * i / d)             # theta_i = 10000^{-2i/d}
    return theta


def apply_rope(x, positions, theta):
    """
    对输入向量 x 施加 RoPE 旋转
    x: [seq_len, d] 或 [1, d] 的 Q 或 K 向量
    positions: [seq_len] token 的绝对位置
    theta: [d/2] 每对维度的旋转频率
    返回: [seq_len, d] 旋转后的向量
    """
    seq_len, d = x.shape
    assert d == 2 * len(theta), f"d={d} 必须是 2 * len(theta)={2*len(theta)}"

    # 计算每个位置 * 每个频率的旋转角度: [seq_len, d/2]
    angles = np.outer(positions, theta)      # angles[m, i] = m * theta_i

    cos_vals = np.cos(angles)                # [seq_len, d/2]
    sin_vals = np.sin(angles)                # [seq_len, d/2]

    # 对每对维度执行 2D 旋转:
    # x'_{2i}   = x_{2i} * cos(theta_i) - x_{2i+1} * sin(theta_i)
    # x'_{2i+1} = x_{2i} * sin(theta_i) + x_{2i+1} * cos(theta_i)
    x_rotated = np.zeros_like(x)
    # 偶数索引维度 (0, 2, 4, ...) → 每对中的第一个分量
    x_rotated[:, 0::2] = x[:, 0::2] * cos_vals - x[:, 1::2] * sin_vals
    # 奇数索引维度 (1, 3, 5, ...) → 每对中的第二个分量
    x_rotated[:, 1::2] = x[:, 0::2] * sin_vals + x[:, 1::2] * cos_vals

    return x_rotated, angles

# ============================================================
# 第 2 步：验证相对位置性质 —— q_m · k_n 只依赖 (m-n)
# ============================================================
print("=" * 60)
print("验证 RoPE 的核心性质: q_m^T k_n 只依赖相对位置 Δ = m-n")
print("=" * 60)

d = 64                 # 向量维度（d 必须是偶数）
base = 10000.0
theta = compute_rope_frequencies(d, base)

# 生成随机的 Q 和 K 原始向量（模拟经过 W_q / W_k 线性变换后的向量）
np.random.seed(42)
q_raw = np.random.randn(1, d)   # [1, d] — 同一个内容向量
k_raw = np.random.randn(1, d)   # [1, d] — 同一个内容向量

# 在不同位置旋转它们
m_positions = [0, 1, 2, 3, 10, 100]   # Q 的绝对位置
n_positions = [0, 1, 2, 5, 12, 105]   # K 的绝对位置

print("\n测试句子: q_m^T k_n 在不同 (m, n) 下的值")
print("-" * 55)
print(f"{'m':>3s} {'n':>3s} {'Δ=m-n':>6s} {'dot product':>15s}  {'验证'}")
print("-" * 55)

for m in m_positions:
    for n in n_positions:
        delta = m - n
        # 对 Q 在位置 m 旋转
        q_rotated, _ = apply_rope(q_raw, np.array([m]), theta)
        # 对 K 在位置 n 旋转
        k_rotated, _ = apply_rope(k_raw, np.array([n]), theta)
        # 计算点积
        dot_val = np.dot(q_rotated[0], k_rotated[0])

        # 验证: q_m^T k_n 应该等于 q_0^T k_{n-m}
        # 即: 只依赖相对位置 delta
        q_0, _ = apply_rope(q_raw, np.array([0]), theta)
        k_delta, _ = apply_rope(k_raw, np.array([abs(delta)]), theta)
        dot_delta_only = np.dot(q_0[0], k_delta[0])

        # 两个值应该相等（在浮点精度内）
        match = "OK" if abs(dot_val - dot_delta_only) < 1e-5 else "MISMATCH"
        print(f"{m:3d} {n:3d} {delta:+6d} {dot_val:15.6f}  {match}")

print("\n结论: 点积确实只依赖 Δ = m-n，不依赖绝对位置 m, n 本身")
print("这意味着 attention 天然具有平移不变性——")
print("两个词的相对位置相同，attention score 就相同")

# ============================================================
# 第 3 步：可视化 — 不同位置的旋转角度
# ============================================================
print("\n" + "=" * 60)
print("生成可视化...")

fig, axes = plt.subplots(2, 2, figsize=(14, 10))

# (a) 不同频率 theta_i 随维度索引 i 的变化
d_viz = 128
theta_viz = compute_rope_frequencies(d_viz, base=10000.0)
i_range = np.arange(len(theta_viz))

axes[0, 0].plot(i_range, theta_viz, color='#2C3E50', linewidth=1.5)
axes[0, 0].set_xlabel('Dimension pair index i', fontsize=11)
axes[0, 0].set_ylabel('theta_i = 10000^{-2i/d}', fontsize=11)
axes[0, 0].set_title('RoPE Frequency Distribution (d=128)', fontsize=12, fontweight='bold')
axes[0, 0].set_yscale('log')
axes[0, 0].grid(True, alpha=0.3)
axes[0, 0].text(0.95, 0.95, f'Range: [{theta_viz[-1]:.6f}, {theta_viz[0]:.1f}]',
                transform=axes[0, 0].transAxes, ha='right', va='top',
                fontsize=9, bbox=dict(boxstyle='round', facecolor='wheat', alpha=0.5))

# (b) 不同位置 m 在最低频维度 (i=0) 上的旋转角度
positions = np.arange(0, 128)
# 选择 3 个频率（低、中、高）
freq_indices = [0, d_viz//4, d_viz//2 - 1]  # theta ≈ 1.0, ~0.1, ~0.0001
freq_labels = [f'i=0 (theta≈{theta_viz[0]:.1f})',
               f'i={d_viz//4} (theta≈{theta_viz[d_viz//4]:.4f})',
               f'i={d_viz//2-1} (theta≈{theta_viz[d_viz//2-1]:.6f})']

colors_rot = ['#E74C3C', '#2ECC71', '#3498DB']
for idx, freq_idx in enumerate(freq_indices):
    angles = positions * theta_viz[freq_idx] * 180 / np.pi  # 转为度数
    axes[0, 1].plot(positions[:64], angles[:64],
                    color=colors_rot[idx], linewidth=2,
                    label=freq_labels[idx])

axes[0, 1].set_xlabel('Absolute position m', fontsize=11)
axes[0, 1].set_ylabel('Rotation angle (degrees)', fontsize=11)
axes[0, 1].set_title('RoPE: Rotation Angle vs Position (3 Frequencies)', fontsize=12, fontweight='bold')
axes[0, 1].legend(fontsize=9)
axes[0, 1].grid(True, alpha=0.3)

# (c) 2D 平面上的旋转轨迹 — 展示旋转的几何含义
# 选一个 2D 向量在位置 0..15 上的旋转轨迹
vec_2d = np.array([1.0, 0.0])  # 初始向量 (1, 0)
theta_demo = 0.3                # 固定一个频率做演示
positions_demo = np.arange(0, 16)

trajectory_x = vec_2d[0] * np.cos(positions_demo * theta_demo) - vec_2d[1] * np.sin(positions_demo * theta_demo)
trajectory_y = vec_2d[0] * np.sin(positions_demo * theta_demo) + vec_2d[1] * np.cos(positions_demo * theta_demo)

axes[1, 0].quiver(np.zeros_like(positions_demo), np.zeros_like(positions_demo),
                  trajectory_x, trajectory_y,
                  angles=np.linspace(0, 360, len(positions_demo)),
                  scale=1, scale_units='xy',
                  cmap='viridis', alpha=0.8)
# 画单位圆
circle = plt.Circle((0, 0), 1, fill=False, color='gray', linestyle='--', alpha=0.5)
axes[1, 0].add_patch(circle)
axes[1, 0].set_xlim(-1.3, 1.3)
axes[1, 0].set_ylim(-1.3, 1.3)
axes[1, 0].set_aspect('equal')
axes[1, 0].set_xlabel('Dimension 2i', fontsize=11)
axes[1, 0].set_ylabel('Dimension 2i+1', fontsize=11)
axes[1, 0].set_title(f'2D Rotation of Vector (1,0) at theta={theta_demo}', fontsize=12, fontweight='bold')
axes[1, 0].grid(True, alpha=0.3)

# (d) 相对位置 Δ 对点积的影响 — 长程衰减可视化
max_delta = 100
deltas = np.arange(max_delta + 1)
q_fixed = np.random.randn(1, d_viz)
k_fixed = np.random.randn(1, d_viz)

dot_values = []
for delta in deltas:
    q_0, _ = apply_rope(q_fixed, np.array([0]), theta_viz)
    k_delta, _ = apply_rope(k_fixed, np.array([delta]), theta_viz)
    dot = np.dot(q_0[0], k_delta[0])
    dot_values.append(dot)

axes[1, 1].plot(deltas, dot_values, color='#2C3E50', linewidth=1.5)
# 添加平滑趋势线
from scipy.ndimage import uniform_filter1d  # 如果 scipy 不可用，跳过
try:
    smoothed = uniform_filter1d(dot_values, size=5)
    axes[1, 1].plot(deltas, smoothed, color='#E74C3C', linewidth=2.5, label='Smoothed trend')
except ImportError:
    pass
axes[1, 1].set_xlabel('Relative distance Delta = |m-n|', fontsize=11)
axes[1, 1].set_ylabel('Dot product q_0^T k_Delta', fontsize=11)
axes[1, 1].set_title('RoPE Long-Range Decay: Attention Score vs Distance', fontsize=12, fontweight='bold')
axes[1, 1].axhline(y=0, color='gray', linestyle='--', alpha=0.5)
axes[1, 1].legend(fontsize=9)
axes[1, 1].grid(True, alpha=0.3)

plt.tight_layout()
plt.savefig('rope_visualization.png', dpi=150, bbox_inches='tight')
print("图表已保存为 rope_visualization.png")

# ============================================================
# 第 4 步：总结核心直觉
# ============================================================
print(f"\n{'='*60}")
print("RoPE 核心直觉总结:")
print(f"{'='*60}")
print("1. 高频维度 (i 大, theta 小):")
print("   旋转快 → 微小位置变化就有大角度差 → 捕捉局部依赖")
print("   体现在图(b)中蓝色曲线的陡峭上升")
print("2. 低频维度 (i 小, theta 大):")
print("   旋转慢 → 需要大距离才有显著角度差 → 捕捉全局依赖")
print("   体现在图(b)中红色曲线的平缓变化")
print("3. 多频率叠加:")
print("   所有 d/2 个频率的 attention score 叠加后 → 自然的长程衰减")
print("   体现在图(d)中点积值随距离增大而振荡减小")
print("4. 关键数学性质:")
print("   R_m^T * R_n = R_{n-m}  (旋转矩阵的可加性)")
print("   因为 cos(a)cos(b)+sin(a)sin(b)=cos(a-b)")
print("   所以 绝对位置编码 → 相对位置效果")
print(f"\n这就是为什么 LLaMA/Qwen/Mistral/ChatGLM 全部使用 RoPE——")
print(f"简洁的数学、优雅的性质、极低的开销、天然的外推能力")
```

## 自测三层
- **L1 复述**：RoPE 的旋转矩阵作用在 attention 的哪些向量上（Q/K/V 中的哪几个）？theta_i = 10000^{-2i/d} 中 i 的范围是什么？高频维度和低频维度分别对应什么 i 值？
- **L2 解释**：RoPE 声称"使用绝对位置编码实现了相对位置的效果"。请用 R_m^T R_n = R_{n-m}（旋转矩阵的可加性）解释这个 claim 的数学原理。为什么这个性质对于长度外推至关重要？
- **L3 应用**：你正在训练一个 LLaMA-7B 风格的模型，上下文窗口是 2048 tokens。产品需要支持 16384 tokens 的上下文（8 倍扩展）。你有两个选择：(A) 重新训练，把上下文窗口直接设为 16384，(B) 保持 2048 训练但用 NTK-aware scaling 把 RoPE 的 base 从 10000 调到 500000。你选哪个？请从：(1) 训练成本，(2) 长序列 perplexity，(3) 短序列性能损失，三个维度分析你的决策。

知识时间锚：2021-04（ArXiv 初版），2023 年起被 LLaMA/Qwen/ChatGLM 等主流开源模型全面采用。RoPE 是当代 LLM 基础设施中最重要的位置编码方案，没有之一。
