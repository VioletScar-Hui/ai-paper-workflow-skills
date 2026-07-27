---
tags: [论文笔记, GLaM, MoE, 混合专家, 高效扩展, 基础论文, Google]
paper_id: "66"
filename: "66 - GLaM - Efficient Scaling of Language Models with Mixture-of-Experts.pdf"
authors: "Nan Du et al. (Google)"
year: 2021
成熟度: ✅
笔记层级: 骨干
复核日期: 2026-07-04
---

# GLaM：通过混合专家高效扩展语言模型

📄 **原文 PDF**：[[RAW/66 - GLaM - Efficient Scaling of Language Models with Mixture-of-Experts.pdf]]

## PM 速判（30秒）
> 一句话：Google 用 1.2T 总参数、每次只激活 8%（96.6B）的稀疏 MoE 模型，在 29 个 NLP 任务上平均打败 GPT-3，训练能耗只有 GPT-3 的 1/3——这是"参数可以白嫖、计算才要花钱"路线的工业级首证。今天 Mixtral、DeepSeek-V3、GPT-4（传闻）全走这条路，PM 必须理解"总参数 ≠ 推理成本"。

## 双层费曼 🗣️
> **给 CEO 的一句话**：GLaM 像一家有 64 个专科门诊的医院——病人（每个词）只挂两个最相关的科室，所以医院规模可以做到 7 倍大，但每个病人的看病成本反而更低，效果还更好。
>
> **给工程师的一段话**：GLaM 是 decoder-only Transformer，每隔一层把 FFN 换成 64 个专家的 MoE 层。一个可学习的门控网络对每个 token 算 softmax 得分，只把 token 发给得分 top-2 的专家，输出按门控权重加权求和。为防止所有 token 挤向少数专家，加辅助负载均衡损失（专家使用频率 f_i × 路由概率 P_i 的和），并用容量因子限制单专家 token 上限。结果：1.2T 总参数，每 token 只算 96.6B 参数的 FLOPs（180 GFLOPs/token，GPT-3 是 350）。

## 问题域定位 🎯
- **根本约束**：密集模型的能力随参数增长，但训练/推理 FLOPs 也线性随参数增长——GPT-3 之后再往上堆，电费和芯片都不可持续（训练 GPT-3 耗 1287 MWh）。
- **之前的方案卡在哪里**：Shazeer 2017 的 MoE 和 GShard/Switch Transformer 证明了稀疏化可行，但要么是机器翻译/encoder-decoder 场景，要么只在微调基准上验证，没人证明**稀疏 decoder-only 模型在 few-shot 上下文学习上能打赢密集模型**。
- **开启/关闭的路线**：开启了"稀疏 LLM 做通用 few-shot"路线（→ Mixtral、DeepSeek-MoE、Grok）；同时宣判了"纯靠密集堆参数"路线在成本上的天花板——后来 LLaMA 用另一条路（小密集模型多喂数据）回应同一个约束。

## 核心机制
```
输入 token x ──► [Attention 层 (密集)] ──► [MoE 层 (稀疏)] ──► ... 交替堆叠
                                              │
              ┌───────────────────────────────┘
              ▼
        门控网络 g = softmax(x·W_gate)         64 个专家得分
              │
              ▼
        取 top-2：如 专家#7 (0.6)、专家#23 (0.3)
              │
       ┌──────┴──────┐
       ▼             ▼
   [专家#7 FFN]  [专家#23 FFN]     其余 62 个专家完全不计算 ←稀疏激活
       │             │
       └──权重归一化加权求和──► 输出 y = 0.67·E7(x) + 0.33·E23(x)

  训练时附加：L_aux = α · N · Σᵢ (fᵢ · Pᵢ)   ← 逼门控把 token 摊匀，防专家崩塌
```

## 设计决策解剖 ⚖️
| 决策点 | 他们的选择 | 被放弃的方案 | 为什么 | 这个选择何时失效 |
|--------|----------|------------|--------|---------------|
| 每 token 激活几个专家 | top-2 | top-1（Switch Transformer）；top-4+ | top-1 质量掉、梯度信号弱；top-4 FLOPs 涨向密集。top-2 是质量/效率折中，且第二名专家提供路由梯度 | 推理是 memory-bound 而非 compute-bound 时（如单卡小 batch 部署）：权重全要放显存，FLOPs 省了但显存/带宽没省 |
| 专家数量 | 每层 64 个 | GShard 的 2048 个超多专家 | 专家太多每个专家分到的数据太少、路由退化；64 个在质量和 all-to-all 通信成本间平衡 | 集群规模变化时：专家数受设备拓扑约束，跨机 all-to-all 通信在低带宽互联上会吃掉全部 FLOPs 收益 |
| MoE 层放置 | 每隔一层替换 FFN（交替密集/稀疏） | 每层都是 MoE | 全 MoE 训练不稳定、通信翻倍；交替结构保留稳定的密集骨架 | 追求极限稀疏率时（DeepSeek-V3 用细粒度专家+共享专家，证明了更激进的稀疏化设计可行） |
| 负载均衡 | 辅助损失 + 容量因子 | 硬性均匀分配；无约束自由路由 | 无约束会专家崩塌（少数专家吃掉所有 token）；硬分配破坏语义路由 | 辅助损失与语言建模目标冲突，分布外数据上路由质量下降；后来 DeepSeek-V3 用无辅助损失的 bias 调节替代 |
| 数据质量 | 自训练质量分类器过滤网页，1.6T token | 直接用原始 CommonCrawl | 论文实验证明小模型+高质量数据 > 大模型+脏数据 | 数据总量成为瓶颈时——过滤太狠会撞"数据墙" |

## 成本与量级 💰
- 训练能耗 456 MWh（GPT-3 为 1287 MWh，省 64.6%）；用 1024 块 TPU-v4 训练，GPU 时未按美元披露，量级估算在数百万美元级
- 推理：180 GFLOPs/token vs GPT-3 350 GFLOPs/token（约一半）；但**显存需求按 1.2T 总参数算**，部署需要大规模多机切分
- 我的产品要用：不要自己训 MoE。最小可行 = 直接用开源 MoE（如 Mixtral-8x7B 级别：约 13B 激活参数的推理成本、接近 70B 密集的质量），且必须确认推理服务商按激活参数还是总参数计费/占卡

## 证据审计 🔬
- **实验设计公平吗**：与 GPT-3 的对比不是严格控制变量——GLaM 数据（1.6T token 自过滤语料）比 GPT-3 数据更新更干净，"MoE 架构收益"和"数据质量收益"纠缠在一起。论文自己的消融也承认数据质量影响巨大
- **最强证据**：同一数据、同激活 FLOPs 下 MoE 变体全面优于密集变体（论文图 3 系列消融），29 任务 0/1/few-shot 平均分 + 训练能耗 456 vs 1287 MWh——这个内部对照比跟 GPT-3 的跨机构对比可信得多
- **最可疑的数字**："能耗仅 1/3"，因为 GPT-3 能耗是按 V100 估算的、GLaM 用 TPU-v4 实测——硬件代际差贡献了相当一部分，不全是架构功劳
- **如果我是审稿人，会要求补充**：同硬件、同数据下密集 175B 的完整复训对照；MoE 在下游微调（而非 few-shot）上的表现；路由决策的可解释性分析
- **最小复现实验**：用 numpy/PyTorch 在 WikiText 上训 4 专家 top-2 小 MoE vs 同激活参数密集模型，比较相同 FLOPs 下的 perplexity；单卡可跑，预算 < 10 美元云 GPU

## 可复用点（PM 决策）
- **何时采用**：吞吐量优先、批量大的服务端场景（总显存充足时，MoE 每 token 成本显著低）；需要"一个模型覆盖多领域"时（专家自然分化）
- **何时规避**：边缘/端侧部署（显存装不下总参数）；小 batch 低延迟场景（稀疏收益被通信和访存开销吃掉）；团队没有多机推理工程能力时
- **供应商拷问清单**：
  1. "你们报的参数量是总参数还是激活参数？推理按哪个计费？"
  2. "专家负载均衡在我们的垂直领域数据上验证过吗——路由熵是多少？"
  3. "MoE 模型量化后专家路由精度是否退化？有实测数据吗？"

## 关联网络 🕸️
- [[Wiki/论文笔记/01_LLM基础架构/69_PaLM用Pathways扩展语言建模]] → 同为 Google 基础设施级论文，PaLM 走密集路线，两者是内部路线赛马
- [[Wiki/论文笔记/02_前沿模型报告/76_GPT-4技术报告]] → GPT-4 被广泛传闻采用 MoE 架构，GLaM 是这条路线的公开先驱
- [[Wiki/概念/01_架构技术/MoE混合专家架构]] → GLaM 是工业级 LLM MoE 的奠基实现
- [[Wiki/概念/01_架构技术/稀疏注意力与长上下文效率]] → 同属"计算稀疏化"大方向
- **冲突/印证**：与 [[Wiki/论文笔记/01_LLM基础架构/75_LLaMA高效开放的基础语言模型]] **路线冲突**——面对同一个"扩展太贵"约束，GLaM 说"参数稀疏化"（大而稀疏），LLaMA 说"小模型多喂数据"（小而密集）。2024 年后 DeepSeek-V3 证明两者可以合流：MoE + 超量 token

## 动手练习 💻（30分钟）
练习目标：用 numpy 手写一个玩具版 top-2 门控路由，亲手看到"64 个专家只算 2 个"的稀疏激活，并算出负载均衡损失。

```python
import numpy as np                          # 导入 numpy，唯一依赖

np.random.seed(42)                          # 固定随机种子，保证每次运行结果一致

# ===== 1. 配置一个迷你 MoE 层 =====
num_tokens = 8                              # 一个 batch 里有 8 个 token
d_model = 16                                # 每个 token 用 16 维向量表示（GLaM 是 8192 维）
num_experts = 4                             # 4 个专家（GLaM 每层 64 个）
top_k = 2                                   # 每个 token 只激活 2 个专家 ← GLaM 核心设计

# ===== 2. 初始化输入和权重 =====
tokens = np.random.randn(num_tokens, d_model)            # 模拟 8 个 token 的隐层向量
W_gate = np.random.randn(d_model, num_experts) * 0.1     # 门控网络：一个 16x4 的矩阵
W_experts = np.random.randn(num_experts, d_model, d_model) * 0.1  # 每个专家是一个 16x16 的 FFN（简化版）

def softmax(x):
    """把任意分数变成和为 1 的概率分布"""
    e = np.exp(x - x.max(axis=-1, keepdims=True))        # 减最大值防止指数溢出
    return e / e.sum(axis=-1, keepdims=True)             # 归一化

# ===== 3. 门控打分：每个 token 给每个专家打分 =====
logits = tokens @ W_gate                    # 矩阵乘法 → 形状 (8, 4)，8 个 token 各有 4 个专家分
probs = softmax(logits)                     # 转成概率，每行加起来 = 1

# ===== 4. top-2 路由：每个 token 挑分数最高的 2 个专家 =====
top2_idx = np.argsort(probs, axis=1)[:, ::-1][:, :top_k] # 按分数降序取前 2 个专家的编号
top2_p = np.take_along_axis(probs, top2_idx, axis=1)     # 取出这 2 个专家对应的概率
top2_w = top2_p / top2_p.sum(axis=1, keepdims=True)      # 把 2 个概率重新归一化成组合权重

# ===== 5. 稀疏前向：只计算被选中的专家 =====
output = np.zeros_like(tokens)              # 准备输出容器
flops_used = 0                              # 记录实际用了多少次专家计算
for t in range(num_tokens):                 # 逐个 token 处理（真实实现是批量算，这里为了看清逻辑）
    for r in range(top_k):                  # 只遍历它的 top-2 专家
        e = top2_idx[t, r]                  # 第 r 名专家的编号
        w = top2_w[t, r]                    # 该专家的组合权重
        output[t] += w * (tokens[t] @ W_experts[e])  # 专家计算结果按权重累加
        flops_used += 1                     # 用掉一次专家前向

# ===== 6. 负载均衡损失：检查 token 是否被摊匀 =====
f = np.zeros(num_experts)                   # f_i：路由到专家 i 的 token 比例
for e in range(num_experts):
    f[e] = np.mean(np.any(top2_idx == e, axis=1))        # 有多大比例的 token 选中了专家 e
P = probs.mean(axis=0)                      # P_i：门控给专家 i 的平均概率
aux_loss = num_experts * np.sum(f * P)      # GLaM 的辅助损失：越不均匀这个值越大（均匀时 ≈ top_k）

# ===== 7. 打印结果，观察稀疏性 =====
print("每个 token 选中的专家编号：\n", top2_idx)
print("各专家被使用的比例 f：", np.round(f, 2))            # 如果某专家接近 1.0 → 专家崩塌！
print("负载均衡损失：", round(aux_loss, 3))
dense_flops = num_tokens * num_experts      # 密集模型：每个 token 要算全部 4 个"专家"
print(f"专家前向次数：MoE={flops_used} vs 密集={dense_flops} → 省了 {1-flops_used/dense_flops:.0%} 计算")
```

跑完后自己动手改两处：(1) 把 `top_k` 改成 1 和 4，看计算节省比例怎么变；(2) 把 `W_gate` 乘 10（门控分数变极端），观察 f 是否向少数专家集中——这就是"专家崩塌"的雏形。

## 自测三层 🎓
- **L1 复述**：GLaM 总参数、激活参数各是多少？每层几个专家、激活几个？训练能耗相对 GPT-3 是多少？
- **L2 解释**：为什么选 top-2 而不是 top-1（Switch 的做法）？为什么不能让门控自由路由，非要加辅助负载均衡损失？
- **L3 应用**：你的团队要为一个日均千万请求的客服产品选模型，候选是 Mixtral-8x7B（47B 总参/13B 激活）和 Qwen-14B 密集模型，两者质量评测接近。从显存占用、吞吐量、单 token 成本三个维度，你会怎么分析选型？什么情况下密集模型反而赢？

📅 知识时间锚：2021-12 发布。此时 Switch Transformer 已出（2021-01），Chinchilla（2022-03）、Mixtral（2023-12）、DeepSeek-V3（2024-12）均未出现——GLaM 是"MoE 能做通用 LLM"的第一个规模化证据。
