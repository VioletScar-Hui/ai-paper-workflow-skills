---
tags: [概念, 架构技术, Transformer, 前向传播]
成熟度: ✅
---
# Transformer 完整前向传播与实现细节

## 一句话定义
> 一个现代 Transformer 语言模型的单层前向传播，是"RMSNorm → QKV 投影（含 GQA）→ RoPE → 注意力计算 → 输出投影 → 残差 → RMSNorm → SwiGLU FFN → 残差"这一固定链路的循环堆叠，最终经过一次全局归一化和 unembedding 得到词表 logits。

## 寓言学习

### 寓言正文

老师傅守着抛光镀层车间的第三道工位。每一枚毛坯戒指从这里过一遍，动作永远是固定的三步：先用卡尺量一下这枚戒指此刻的实际厚度，记下这个基准数；按这批订单的镀层配方，往上叠一层新的镀层；镀完之后，把新叠上去的这层厚度加回到刚才记的基准数上，作为这枚戒指离开这道工位时的最终厚度——绝不会用镀完之后的新厚度去覆盖或者抹掉之前的基准数，新老两笔厚度是加在一起的。戒指顺着流水线依次走过八道结构完全相同的工位，每道工位都重复这三步，只是配方换了几次，最后送到定型车间统一抛光出片。

有一次赶工，第五道工位的老师傅图省事，觉得这批订单和上一批差不多，就没重新量厚度，直接凭手感估了个数就往上叠镀层。表面上戒指照样顺利地从这道工位送到了下一道，一路走完了剩下三道工位，谁也没拦下它。等到最后统一抛光定型时，质检员发现整批戒指的镀层比订单要求厚了一圈，查了半天才揪出问题——就出在第五道工位那次"图省事"漏掉的测量。因为厚度是一道道往上叠加的，那道工位记错的基准数，被后面几道工位原样继承下去，越往后偏差被放大得越厉害。

### 概念解析

**概念名称**：Transformer 完整前向传播（单层子层的固定处理链路与残差堆叠机制），属于神经网络架构 / 深度学习工程实现问题域。

**学习卡片**
- 核心张力：希望每一层都能对输入表示做实质性加工（注意力、FFN 各自学到的变换）vs 又必须保证多层堆叠之后，最初的输入信息不会在层层加工中被冲刷丢失。
- 关键不变量：(1) 每个子层在做完自己的变换后，必须把结果以"相加"的方式并入原始输入（残差连接 X ← X + F(X)），而不是直接替换掉原始输入；(2) 每一层的处理链路结构完全相同（归一化→变换→残差），只是权重不同，靠这种结构一致性才能稳定堆叠 L 层；(3) 某一步的计算依赖前一步的正确输出，接口（张量形状）必须严格匹配。
- 触发条件：需要用统一、可重复堆叠的处理单元，把原始输入逐层加工为带有上下文信息的最终表示。
- 常见误解/失败模式：把"多层堆叠"想象成每层都在互相独立地重新计算，忽视了残差连接让信息以"累积叠加"的方式向后传递——某一层如果算错了（比如漏掉一次归一化、用错了基准），这个错误会被当作"基底"原样带入后续所有层，而不会在下一层被自动纠正。
- 非适用边界：如果一个系统本来就是单次、无需分层加工的变换（不需要堆叠多个结构相同的处理单元），残差连接和逐层堆叠这套机制没有意义。

**一句话核心定义**：Transformer 的前向传播是把"归一化→变换（注意力或 FFN）→残差相加"这一固定结构的处理单元重复堆叠 L 层，每层都在前一层输出的基础上做增量式加工，而不是推倒重算。

**它试图解决的问题**：如何让一个很深（很多层）的网络，既能让每一层都学到有意义的新变换，又不会因为堆叠太深导致最初的输入信息被层层加工冲刷丢失、梯度难以往回传递。

**故事元素映射**：
- 卡尺量基准厚度 = 每层子层开头的归一化（RMSNorm），先建立一个稳定的基准状态。
- 按配方叠一层新镀层 = 该子层的实质变换（注意力计算或 FFN 计算）。
- 把新叠的厚度加回基准数、而不是覆盖 = 残差连接 X ← X + F(X)，变换结果是叠加而非替换。
- 八道结构相同的工位 = L 层结构相同、参数不同的 Transformer 层堆叠。
- 第五道工位漏测厚度、偏差被后续工位继承放大 = 某一层若跳过应有步骤或算错，错误会作为"基底"被后续层的残差连接原样带下去，不会被自动修正。

**比喻边界**：故事把"厚度"简化成一个标量，现实中每层传递的是一个高维张量（形状 B×S×D），残差相加是逐元素的向量加法；而且真实的 Transformer 层内部还包含 QKV 投影、RoPE 旋转、因果掩码、softmax 等多个子步骤，并不是故事里"一步到位"的镀层，这些细节故事没有展开。

**非适用例**：如果某个模型结构在每一层之间完全重新初始化、不使用残差连接（比如早期没有残差结构的深层网络），层与层之间就不是"叠加"关系，而更接近"完全重做"——那种结构不适合用这个"基准+叠加"的故事去理解，实际上也正因为缺少残差连接，训练很深的网络会遇到梯度消失等问题。

**现实例子**：在标准 Transformer 实现代码里，每个子层的最后一步都会写成 `return x + output`（先算出这一层的变换结果 output，再和输入 x 相加），而不是 `return output`；如果哪个实现漏写了这个 `x +`，模型在数值上依然能跑，但训练效果会明显变差——这正对应故事里"漏测厚度却照样送到下一道工位"的失败模式。

### 检验问题

1. 理解检验：故事里第五道工位的错误，为什么没有在第六、第七道工位被自动发现或纠正，反而被一路带到了最后？这和 Transformer 里残差连接的哪个特性对应？
2. 迁移检验：如果有人说"某个 Transformer 层的输出算错了，但反正后面还有很多层，模型整体应该没事"，这个说法哪里有问题？

## 核心机制

### 符号表

| 符号 | 含义 |
|------|------|
| B | batch 中的序列条数 |
| L | 层数 |
| T | 序列长度（本次要生成的 token 数） |
| S | 序列长度（已提供的上下文长度） |
| V | 词表大小 |
| D | 隐藏维度 |
| H | 每个注意力头的维度 |
| F | FFN 中间维度（一般 $F = 8D/3$） |
| N | Query 头的数量，满足 $N \times H = D$ |
| K | Key/Value 头的数量；标准多头注意力中 $K=N$，GQA（分组查询注意力）中 $K < N$ |
| G | GQA 的分组大小，$G = N / K$（每 G 个 Query 头共享一组 K/V 头） |

### Token Embedding

用 embedding 矩阵 $\mathbf{W}_e \in \mathbb{R}^{V \times D}$ 把输入 token 序列查表得到初始隐藏状态：

$$
\mathbf{X}^{(0)} = \mathbf{W}_e[\text{tokens}] \in \mathbb{R}^{B \times S \times D}
$$

之后进入 $\ell \in [0, \ldots, L-1]$ 的逐层循环。

### 每一层的注意力子层

**第 1 步 RMSNorm**（详见 [[Wiki/概念/01_架构技术/RMSNorm与LayerNorm]]）：

$$
\bar{\mathbf{X}}^{(\ell)} = \frac{\mathbf{X}^{(\ell)}}{\text{RMS}(\mathbf{X}^{(\ell)}) + \epsilon} \odot \gamma_{\text{attn}}^{(\ell)}
$$

**第 2 步 QKV 投影**：用 $W_Q \in \mathbb{R}^{D \times D}$、$W_K, W_V \in \mathbb{R}^{D \times KH}$（GQA 场景下 K/V 的输出维度是 $K \times H$，比 Q 的 $D=N \times H$ 小）：

$$
\mathbf{Q} = \bar{\mathbf{X}} \mathbf{W}_Q \in \mathbb{R}^{B \times T \times D}, \quad
\mathbf{K} = \bar{\mathbf{X}} \mathbf{W}_K \in \mathbb{R}^{B \times S \times KH}, \quad
\mathbf{V} = \bar{\mathbf{X}} \mathbf{W}_V \in \mathbb{R}^{B \times S \times KH}
$$

**第 3 步【可选】QK-norm**：对 Q、K 向量再做一次 RMSNorm，控制点积输入的幅度，进一步稳定训练。

**第 4 步 reshape 出 head 维度**：把 D 拆成 $N \times H$（对 K/V 是拆成 $K \times H$），再把序列长度维和 head 维互换位置（为了让后续矩阵乘法在 head 维上"批量并行"）：

$$
\mathbf{Q} \in \mathbb{R}^{B \times T \times D} \rightarrow \mathbb{R}^{B \times N \times T \times H}
$$
$$
\mathbf{K}, \mathbf{V} \in \mathbb{R}^{B \times S \times (KH)} \rightarrow \mathbb{R}^{B \times K \times S \times H}
$$

**第 5 步 GQA 场景下 expand K/V**：把 K 个 K/V 头复制/广播成 N 个，让每 G 个 Query 头共享同一组 K/V：

$$
\mathbf{K}, \mathbf{V} \in \mathbb{R}^{B \times K \times S \times H} \rightarrow \mathbb{R}^{B \times N \times S \times H}
$$

GQA 的意义在于：Q 头数量不变（保留表达能力），但 K/V 头数量减少，从而显著降低 KV cache 的显存占用（详见 [[Wiki/概念/01_架构技术/Transformer显存与算力核算]]）。

**第 6 步 施加 RoPE 到 Q、K 上**：在位置 $m$ 上，把 head 内每一对维度 $(2i, 2i+1)$ 按角度 $m\theta_i$（$\theta_i = \Theta^{-2i/H}$）旋转：

$$
\mathbf{R}_m^{(i)} = \begin{bmatrix} \cos(m\theta_i) & -\sin(m\theta_i) \\ \sin(m\theta_i) & \cos(m\theta_i) \end{bmatrix}, \quad
\mathbf{q}_m \leftarrow \mathbf{R}_m \mathbf{q}_m, \quad \mathbf{k}_m \leftarrow \mathbf{R}_m \mathbf{k}_m
$$

RoPE 只旋转 Q 和 K，不作用于 V。完整机制见 [[Wiki/概念/01_架构技术/RoPE旋转位置编码]]。

**第 7 步 计算注意力分数**：

$$
\mathbf{A} = \frac{\mathbf{Q}\mathbf{K}^\top}{\sqrt{H}} \in \mathbb{R}^{B \times N \times T \times S}
$$

**为什么要除以 $\sqrt{H}$**：如果不除，点积的数值量级会随 head 维度 H 增大而增大（$\sqrt{H}$ 是点积方差的自然尺度）。过大的输入送进 softmax 会让输出分布变得过于"尖锐"（peakier），梯度对这种饱和区域的更新会变得迟钝、难以调整——除以 $\sqrt{H}$ 是把点积拉回一个数值稳定的范围。

**第 8 步 施加因果掩码（causal mask）**：

$$
\mathbf{A}_{ij} \leftarrow \begin{cases} \mathbf{A}_{ij} & j \le i \\ -\infty & j > i \end{cases}
$$

即每个位置只能看到自己和之前的位置，禁止看到未来的 token。

**第 9 步 softmax**：

$$
\mathbf{A} = \text{softmax}(\mathbf{A}) = \frac{\exp(\mathbf{A}_{ij})}{\sum_{k=1}^{S}\exp(\mathbf{A}_{ik})}
$$

**第 10 步 用注意力权重对 V 加权求和**：

$$
\mathbf{O} = \mathbf{A}\mathbf{V} \in \mathbb{R}^{B \times N \times T \times H}
$$

**第 11 步 reshape 回原始维度**：$\mathbf{O} \in \mathbb{R}^{B \times N \times T \times H} \rightarrow \mathbb{R}^{B \times T \times D}$

**第 12 步 输出投影**（把不同 head 的信息混合到一起）：

$$
\mathbf{O}_{\text{proj}} = \mathbf{O}\mathbf{W}_O, \quad W_O \in \mathbb{R}^{D \times D}
$$

**第 13 步 残差连接**：

$$
\mathbf{X}^{(\ell)} \leftarrow \mathbf{X}^{(\ell)} + \mathbf{O}_{\text{proj}}
$$

### 每一层的 FFN 子层

FFN 子层同样先做一次 RMSNorm（用独立的 $\gamma_{\text{ffn}}$），再经过 gate/up 投影、SwiGLU 激活、down 投影，最后做残差连接：

$$
\mathbf{X}^{(\ell+1)} = \mathbf{X}^{(\ell)} + \mathbf{F}
$$

完整公式和 gate/up 各自的角色讲解见 [[Wiki/概念/01_架构技术/SwiGLU门控前馈网络]]，此处不重复。

### 最终归一化与 unembedding

L 层循环结束后，做最后一次 RMSNorm（用 $\gamma_{\text{final}}$）：

$$
\mathbf{X}_{\text{final}} = \frac{\mathbf{X}^{(L)}}{\text{RMS}(\mathbf{X}^{(L)}) + \epsilon} \odot \gamma_{\text{final}}
$$

再用 unembedding 矩阵 $\mathbf{W}_u \in \mathbb{R}^{D \times V}$ 投影到词表维度，得到 logits：

$$
\mathbf{Z} = \mathbf{X}_{\text{final}} \mathbf{W}_u \in \mathbb{R}^{B \times T \times V}
$$

## 实现层面的关键细节

### masked_fill 的用法

因果掩码在代码里通常这样实现：

```python
scores.masked_fill(~mask, -torch.inf)
```

约定 `mask` 中 `True` 表示"这个位置可以被看到"。`tensor.masked_fill(mask, value)` 会把 `mask` 为 `True` 的位置填充为 `value`——注意这里传入的是 `~mask`（取反），也就是把"不可见"的位置填成 $-\infty$，这样过 softmax 之后这些位置的权重会变成 0。

### RoPE 的实现：预计算 cos/sin + 偶奇维度交错旋转

**预计算**（初始化时算好，避免每次前向重复计算）：

```python
positions = torch.arange(max_seq_len, device=device)  # shape (max_seq_len,)
thetas = self.theta ** (-torch.arange(0, d_k, 2, device=device) / d_k)  # shape (d_k // 2,)
angles = positions.unsqueeze(-1) * thetas.unsqueeze(0)  # 每个 (位置, 频率索引) 对应的角度
```

**实际旋转不是真的构造 2x2 矩阵做 matmul**，而是直接用点积形式展开计算，效率更高：

```python
# 把最后一维 H 拆成 (H/2, 2)，取出偶数索引和奇数索引
x_pairs = x.reshape(*x.shape[:-1], -1, 2)
x_even = x_pairs[..., 0]
x_odd = x_pairs[..., 1]

# 按旋转矩阵展开后的公式直接计算
x_out_even = x_even * cos - x_odd * sin
x_out_odd = x_even * sin + x_odd * cos

# 交错拼回原始形状：先在新维度堆叠，再展平
result = torch.stack([x_out_even, x_out_odd], dim=-1).flatten(start_dim=-2)
```

这里的核心是：把 H 维向量看成 H/2 对，每一对按预先算好的 $\cos(m\theta_i)$、$\sin(m\theta_i)$ 做"旋转公式"的直接代数展开（而不是构造旋转矩阵做矩阵乘法），计算上更高效，也是绝大多数开源实现（如 LLaMA 系列）采用的写法。

### 标准 attention 前向代码（含 reshape 全过程）

```python
batch_size, seq_len, _ = x.shape
x_norm = self.norm(x)
qkv = self.qkv_proj(x_norm)  # (batch, seq_len, 3 * d_model)
qkv = qkv.reshape(batch, seq_len, 3, self.num_heads, self.head_dim)
q, k, v = qkv.unbind(dim=2)  # 拆出 q/k/v，各为 (batch, seq_len, num_heads, head_dim)
q = q.transpose(1, 2)  # (batch, num_heads, seq_len, head_dim)
k = k.transpose(1, 2)
v = v.transpose(1, 2)
causal_mask = torch.tril(torch.ones(seq_len, seq_len)).bool()
output = scaled_dot_product_attention(q, k, v, causal_mask)  # (batch, num_heads, seq_len, head_dim)
output = output.transpose(1, 2)  # 换回 (batch, seq_len, num_heads, head_dim)
output = output.reshape(batch, seq_len, d_model)  # 合并 head 维
output = self.out_proj(output)  # 输出投影
return x + output  # 残差连接
```

关键的 reshape/transpose 操作：
- `.reshape()` 把 `d_model`（D）展开成 `num_heads x head_dim`（N × H）
- `qkv.unbind()` 按拆分维度切出 Q/K/V（具体实现依框架而异，也可以用三个独立的 Linear 层分别投影）
- `.transpose()` 交换 `num_heads` 和 `seq_len` 两个维度，使得注意力计算能在 head 维上批量并行
- 计算完 attention 输出后，需要再 `.transpose()` + `.reshape()` 换回原始的 `(batch, seq_len, d_model)` 形状

### scaled_dot_product_attention 的标准实现

```python
def scaled_dot_product_attention(q, k, v, mask):
    """
    q, k: (batch_size, ..., seq_len, d_k)
    v:    (batch_size, ..., seq_len, d_v)
    返回:  o (batch_size, ..., seq_len, d_v)
    """
    d_k = q.shape[-1]
    scores = (q @ k.transpose(-2, -1)) / math.sqrt(d_k)
    scores = scores.masked_fill(~mask, -torch.inf)
    return softmax(scores, dim=-1) @ v
```

这段代码把上面公式化描述的"QK^T / √H → 掩码 → softmax → 加权求和 V"四步浓缩成四行，是理解 Flash Attention 等优化手段（见 [[Wiki/概念/01_架构技术/Flash Attention]]）的基准对照实现——Flash Attention 要优化的正是这里 `scores` 这个 $(seq\_len \times seq\_len)$ 矩阵的显存读写开销。

## 为什么 AI PM 要懂这个

- 能完整复述一层 Transformer 从输入到输出的数据流，是审"模型结构改动"类技术方案（比如"要不要加 QK-norm"、"GQA 分组大小怎么定"）的基本功，不至于只能停留在"attention 是核心机制"这种表层理解。
- 理解 GQA 的 K/V 头数量为什么可以和 Query 头数量不同，能看懂各家开源模型 config 里 `num_key_value_heads` 这个参数的实际含义和它对显存/推理速度的影响。
- 理解为什么要除以 $\sqrt{H}$、为什么要施加因果掩码，是回答"这个模型的注意力实现是不是有 bug"这类工程排查问题时的基础判断力。
- 知道 RoPE 在实现上是"偶奇维度交错的代数展开"而非真的做矩阵乘法，有助于理解为什么某些推理加速库（vLLM、FlashAttention 等）能把 RoPE 融合进 kernel 里做得很快。

## 相关概念
- [[Wiki/概念/01_架构技术/RMSNorm与LayerNorm]] — 注意力子层和 FFN 子层前各有一次独立的 RMSNorm
- [[Wiki/概念/01_架构技术/SwiGLU门控前馈网络]] — 每层 FFN 子层的具体实现
- [[Wiki/概念/01_架构技术/RoPE旋转位置编码]] — 施加在 Q/K 上的位置编码机制
- [[Wiki/概念/01_架构技术/自注意力机制]] — 注意力分数计算与加权求和的基础原理
- [[Wiki/概念/01_架构技术/Flash Attention]] — 优化本文标准实现中 attention 矩阵的显存读写
- [[Wiki/概念/01_架构技术/Transformer显存与算力核算]] — 本文前向传播对应的参数量/显存/FLOPs 核算

> 来源：Alisa's Book of LLMs（https://alisawuffles.notion.site/alisa-s-book-of-llms），本地存档见 `RAW/alisa_book_of_llms.md`
