---
tags: [论文笔记, DeepSeek-V4, MoE, 长上下文, 百万token, 架构创新, CSA, HCA, DeepSeek]
paper_id: "82"
笔记层级: 骨干
复核日期: 2026-07-04
---

# DeepSeek-V4：面向百万 Token 的高效长上下文智能

📄 **原文 PDF**：[[RAW/82 - DeepSeek-V4 - Towards Highly Efficient Million-Token Context Intelligence.pdf]]

## PM 速判
> 首个高效原生支持 100 万 Token 上下文的开源顶级模型。CSA+HCA 混合注意力将 1M 场景 KV Cache 压缩至 V3.2 的 10%（Flash 仅 7%），推理 FLOPs 降至 27%（Flash 仅 10%）。Codeforces 3206 首次超越所有闭源模型，LiveCodeBench 93.5% 登顶。**百万 Token 推理从"实验室可行"变成"生产可部署"。**

## 双层费曼
> **给CEO**：以前让 AI 读完一本《三体》三部曲（约 90 万字），显存会爆。DeepSeek-V4 通过"选择性记忆"机制，处理百万字长文时显存只需以前的十分之一，成本断崖式下降。这意味着合同审查、医疗记录分析、全代码库重构这类需要"通读全文"的任务，现在可以在一张 GPU 上跑了。
>
> **给工程师**：V4 在 Transformer 层间交替插入 CSA（Compressed Sparse Attention，将长序列压缩为固定大小的稀疏索引块）和 HCA（Heavily Compressed Attention，进一步激进压缩为极少量全局 token），配合 Manifold-Constrained Hyper-Connections（mHC）替换残差连接改善梯度流，以及 Muon 优化器替代 AdamW。后训练用 On-Policy Distillation 合并 10+ 领域专家模型（各自由 SFT+GRPO 独立训练），避免多任务梯度冲突。FP4 量化感知训练用于路由专家参数。

## 问题域定位
- **根本约束**：标准 Attention 的 KV Cache 随序列长度 O(n) 增长，1M token 时即使 GQA + KV8 量化，单个 H100 80GB 也无法容纳
- **之前卡点**：长上下文方案要么牺牲精度（滑动窗口丢失远期信息），要么计算爆炸（Full Attention O(n^2)），要么工程复杂（RingAttention 需多机通信）
- **V4 开启的路线**：混合注意力（CSA+HCA 交替）证明可以用"压缩摘要 + 局部精确"的组合将长上下文成本降至可单机部署
- **V4 关闭的路线**：纯 Full Attention 长上下文的可行性——在 1M 规模下，即使 H200 141GB 也无法经济部署

## 核心机制

```
DeepSeek-V4 架构（相比 V3.2 的四大升级）：

┌─────────────────────────────────────────────────────────┐
│                    V4 Transformer Layer                  │
│  ┌─────────────────┐    ┌─────────────────┐             │
│  │  CSA Layer      │    │  HCA Layer      │  ← 交替排列  │
│  │  (奇数层)        │    │  (偶数层)        │             │
│  │  稀疏索引压缩    │    │  激进全局压缩    │             │
│  │  KV Cache: ~10%  │    │  KV Cache: ~1%   │             │
│  └─────────────────┘    └─────────────────┘             │
│           ↓                      ↓                       │
│  ┌──────────────────────────────────────────┐           │
│  │  mHC (Manifold-Constrained Hyper-Conn)   │           │
│  │  替代传统 skip connection，约束残差流形   │           │
│  └──────────────────────────────────────────┘           │
│           ↓                                              │
│  ┌──────────────────────────────────────────┐           │
│  │  MoE FFN (路由专家 + 共享专家)            │           │
│  │  V4-Pro: 1.6T/49B | V4-Flash: 284B/13B  │           │
│  │  路由专家使用 FP4 量化感知训练             │           │
│  └──────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────┘

后训练范式：
  Specialist Training → On-Policy Distillation → Unified Model
  ① 按领域独立训练专家：数学/代码/Agent/指令跟随/写作/...
  ② 各专家用 SFT + GRPO 分别优化（避免多任务梯度冲突）
  ③ On-Policy 蒸馏：从专家采样 → 学生模型在线学习 → 合并为单一模型

Muon 优化器：
  通过 Newton-Schulz 迭代近似矩阵平方根进行梯度正交化
  替代 AdamW → 收敛更快，训练更稳定
```

## 设计决策解剖

| 决策点 | 选择 | 被放弃方案 | 为什么 | 何时失效 |
|--------|------|-----------|--------|---------|
| 注意力压缩策略 | CSA+HCA 混合（交替排列） | 纯 CSA / 纯 HCA / 纯滑动窗口 | CSA 保留局部精确性（最近 token），HCA 提供全局摘要；单独一种都会丢失另一维度的信息 | 当任务需要每个 token 间精确交互（如 DNA 碱基配对）时，压缩的全局摘要可能丢失关键依赖 |
| 残差连接 | mHC（流形约束超连接） | Pre-Norm / Post-Norm / DeepNorm | mHC 约束残差流在低维流形上演化，防止深层网络的梯度爆炸/消失——V4 比 V3 更深更宽，传统方案不够稳定 | 浅层网络（<20层）时 mHC 的约束可能过度正则化，反而限制表达能力 |
| 优化器 | Muon（Newton-Schulz 正交化） | AdamW / Lion / Sophia | Muon 对 MoE 架构的路由不稳定性更鲁棒，收敛曲线更平滑；但需要额外计算矩阵平方根 | 小规模模型（<1B）时 Muon 的计算开销可能超过收益，AdamW 足够 |
| 后训练合并 | On-Policy Distillation | 混合 RL / 多任务联合训练 / 模型合并（Model Soup） | 多任务联合训练存在梯度冲突（数学优化方向 vs 写作优化方向可能正交）；On-Policy 蒸馏让学生模型在线从专家采样，分布偏移更小 | 领域差异过小时（如仅数学 vs 物理），独立专家训练 + 蒸馏的额外成本不值得 |
| 量化方案 | FP4 QAT（仅路由专家 + CSA Indexer） | FP8 / INT4 PTQ / 全模型 FP4 | 路由专家参数占比最大（1.6T 中的 ~1.5T），FP4 可节省 1/3 推理成本；训练时用 QAT 而非 PTQ 避免精度崩塌 | 当前硬件（H100/H200）无原生 FP4 支持，收益短期无法兑现；需等待 Blackwell 或 Ascend 后续代 |

## 成本与量级

| 指标 | V4-Pro | V4-Flash | V3.2（参考） |
|------|--------|----------|-------------|
| 总参数 / 激活参数 | 1.6T / 49B | 284B / 13B | 671B / 37B |
| 预训练数据 | 33T tokens | 32T tokens | 14.8T tokens |
| 1M Token 推理 FLOPs（相对 V3.2） | **27%** | **10%** | 100%（基准） |
| 1M Token KV Cache（相对 V3.2） | **10%** | **7%** | 100%（基准） |
| KV Cache vs GQA-bf16 基线 | ~2% | ~1.4% | — |

- **单机可行性**：V4-Flash 在 1M token 时 KV Cache 约为 V3.2 的 7%，意味着 8xH100 80GB 即可服务百万上下文（V3.2 需要 ~80 张）
- **FP4 潜在收益**：路由专家 FP4 → 推理显存再降约 1/3，但需等硬件支持

## 证据审计

### 实验公平性
- 对比模型均使用各自最强推理配置（GPT-5.4-xHigh, Opus-4.6-Max, Gemini-3.1-Pro-High），公平
- 但 V4-Pro-Max 使用多数投票（majority voting），闭源模型是否同等配置未披露 → 可能高估 V4 相对优势

### 最强证据（数字 + 条件）
- **LiveCodeBench 93.5%**（V4-Pro-Max，第一）——代码生成综合基准，覆盖面广，差距显著
- **Codeforces 3206**（第一，超 GPT-5.4 的 3168）——竞争编程，含时间压力，难以作弊
- **Apex Shortlist 90.2%**（第一，超 Gemini 的 89.1%）——企业级代码补全
- 条件：上述均为 V4-Pro-Max（含多数投票）；单次推理分数会低约 3-5pp

### 最可疑数字及原因
- **SimpleQA-Verified 57.9% vs Gemini 75.6%**：差距 17.7pp，世界知识类任务暴露 V4 的预训练语料覆盖不如 Gemini——这个差距在面向 C 端用户时可能是致命短板
- **复杂指令跟随 46.9% vs Opus 53.1%**：V4 在需要精确遵循多约束指令时仍弱于 Claude，说明 On-Policy Distillation 在"对齐细粒度人类偏好"上不如 RLHF 精细
- **FP4 收益未实测**：论文说 FP4 QAT 已训练，但无实际硬件推理基准——可能训练时 FP4 的模拟与真实硬件的数值行为有偏差

### 审稿补充
- 未看到长上下文困惑度（PPL）随位置的变化曲线——压缩注意力是否在序列中段（远离两端）存在"信息丢失谷"？
- 1M token 的"大海捞针"（Needle-in-a-Haystack）结果应补充，尤其是针在不同位置的检索准确率

### 最小复现实验
如果只能做一个实验验证核心声明：在 128K/256K/512K/1M 四个上下文长度下，对比 V4-Flash 与 V3.2 的 KV Cache 大小（bytes）和单 token 推理延迟（ms），绘制双 Y 轴图。预期 V4-Flash 延迟随长度增长斜率远低于 V3.2。

## 可复用点

### 何时采用
- 需要处理 >128K token 的长文档，且预算不支持多机部署
- 开源模型选型，需要原生 1M 上下文（非 RoPE 外推拼凑）
- 代码/数学推理场景，且对世界知识要求不高

### 何时规避
- 世界知识密集型问答（选 Gemini）/ 复杂多轮指令跟随（选 Claude）
- 硬件为 H100 且需要 FP4 即时收益（硬件的 FP4 需 Blackwell+ 代）
- 上下文 <32K 的短任务（CSA/HCA 的压缩开销可能得不偿失）

### 供应商拷问清单
1. "你们的 1M 上下文是原生训练的还是 RoPE 外推的？"→ 原生 > 外推
2. "KV Cache 在 1M token 时实际占用多少 GB？"→ 要求实测数字，不只是论文比例
3. "长上下文中段的检索准确率是多少？"→ 警惕两端的"边界效应"
4. "On-Policy Distillation 合并了多少个专家？各专家的 RL 奖励函数差异多大？"

## 关联网络

- **前置**：[[Wiki/论文笔记/02_前沿模型报告/83_DeepSeek-V3.2开放大模型新前沿]] — V3.2 是 V4 的直接前身，V4 在 V3 架构上叠加压缩注意力
- **并行**：[[Wiki/论文笔记/02_前沿模型报告/86_DeepSeek-Coder-V2突破代码智能闭源壁垒]] — 代码能力演进路线的里程碑
- **技术依赖**：[[Wiki/概念/01_架构技术/MoE混合专家架构]] — V4 的 MoE 底座；[[Wiki/概念/01_架构技术/稀疏注意力与长上下文效率]] — CSA/HCA 是稀疏注意力方向的前沿落地
- **冲突/印证**：Gemini 3.1-Pro 在 SimpleQA 上领先 17.7pp，印证了"Dense 模型 + 海量知识预训练"在世界知识任务上仍优于 MoE + 压缩注意力——压缩注意力可能在压缩 KV 的同时也压缩了细粒度知识检索精度
- **工具链**：vLLM 已原生支持 V4 系列（2026-04-24），包括 hybrid KV cache、kernel fusion、disaggregated serving

## 动手练习

```python
"""
练习：估算 1M Token 在不同注意力方案下的显存与时延
目标：理解为什么 CSA+HCA 是百万 Token 可部署的关键
"""
import math

# ===== 模型配置（以 V4-Flash 为参考） =====
SEQ_LEN = 1_000_000          # 1M tokens
NUM_LAYERS = 48              # 总层数
NUM_KV_HEADS = 8             # GQA KV 头数
HEAD_DIM = 128               # 每头维度
BYTES_PER_ELEM = 1           # KV8 量化（1 byte/elem）

# ===== 三种注意力方案的 KV Cache 计算 =====
def kv_cache_full_attention(seq_len):
    """标准 Full Attention：每层存储全部 token 的 K+V"""
    per_layer = 2 * NUM_KV_HEADS * HEAD_DIM * seq_len * BYTES_PER_ELEM
    return per_layer * NUM_LAYERS  # bytes

def kv_cache_sliding_window(seq_len, window=8192):
    """滑动窗口 Attention：只存最近 window 个 token"""
    per_layer = 2 * NUM_KV_HEADS * HEAD_DIM * window * BYTES_PER_ELEM
    return per_layer * NUM_LAYERS

def kv_cache_csa_hca(seq_len, csa_ratio=0.10, hca_ratio=0.01):
    """
    CSA+HCA 混合（V4 方案）：奇数层 CSA(10%)，偶数层 HCA(1%)
    平均每层约为 5.5% 的 Full Attention KV Cache
    """
    half = NUM_LAYERS // 2
    per_csa_layer = 2 * NUM_KV_HEADS * HEAD_DIM * int(seq_len * csa_ratio) * BYTES_PER_ELEM
    per_hca_layer = 2 * NUM_KV_HEADS * HEAD_DIM * int(seq_len * hca_ratio) * BYTES_PER_ELEM
    return half * per_csa_layer + half * per_hca_layer

# ===== 时延估算（简化模型） =====
def estimate_latency(kv_cache_bytes, mem_bw_gb=2000):
    """
    假设 memory-bound：每 token 生成需从 HBM 读取全部 KV Cache
    mem_bw_gb: H100 HBM 带宽 ~2TB/s
    返回每 token 时延（ms）
    """
    return (kv_cache_bytes / (mem_bw_gb * 1e9)) * 1000

# ===== 计算并输出 =====
full_kv = kv_cache_full_attention(SEQ_LEN)
window_kv = kv_cache_sliding_window(SEQ_LEN, 8192)
csa_hca_kv = kv_cache_csa_hca(SEQ_LEN)

print(f"===== 1M Token KV Cache 显存估算 =====\n")
for name, kv_bytes in [("Full Attention", full_kv), ("Sliding Window(8K)", window_kv), ("CSA+HCA(V4-Flash)", csa_hca_kv)]:
    print(f"{name:.<25s} {kv_bytes/1e9:>8.3f} GB  | 延迟: {estimate_latency(kv_bytes):>6.2f} ms/token")

print(f"\nCSA+HCA 是 Full Attention 的 {csa_hca_kv/full_kv*100:.1f}%")
print(f"CSA+HCA 是 Sliding Window 的 {csa_hca_kv/window_kv*100:.1f}%")
# 结论：Full Attention 在 1M token 需要 ~98GB KV Cache（超 H100 80GB）
# CSA+HCA 降至 ~5.4GB，Sliding Window 仅 0.8GB 但丢失远期信息
# 实际 V4-Flash 是 7%（因还包含其他结构），但数量级一致
```

## 自测三层

**L1 记忆**：V4-Pro/V4-Flash 的参数量各是多少？CSA 和 HCA 的全称是什么？1M token 场景下 V4-Pro 的 FLOPs 和 KV Cache 分别是 V3.2 的多少？

**L2 理解**：为什么 CSA+HCA 要交替排列而不是全部使用 CSA 或全部使用 HCA？On-Policy Distillation 相比多任务联合训练的优势是什么？mHC 解决的是什么梯度问题？

**L3 应用**：如果你要为一家法律科技公司选型长文档处理模型，候选是 V4-Flash 和 GPT-5.4-mini（128K 原生上下文 + RoPE 外推到 256K），你会问供应商哪些关键问题？哪些场景下你会选 V4-Flash，哪些场景下选 GPT-5.4-mini？为什么 SimpleQA 差距 17.7pp 在法律场景可能不重要而在客服场景致命？

📅 知识时间锚：2025 年 DeepSeek 发布 V4 系列，开源模型首次在 Codeforces（3206）等代码竞赛上超越所有闭源模型。vLLM 于 2026-04-24 宣布原生支持 V4 系列。

## 原子概念索引
- [[Wiki/概念/01_架构技术/稀疏注意力与长上下文效率]] — CSA/HCA 混合注意力是该方向的前沿实现
- [[Wiki/概念/01_架构技术/MoE混合专家架构]] — V4 延续 DeepSeek 的 MoE 架构路线
- [[Wiki/概念/01_架构技术/量化技术]] — FP4 量化感知训练是该方向在训练阶段的应用
