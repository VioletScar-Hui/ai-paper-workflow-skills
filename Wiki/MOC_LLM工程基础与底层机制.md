---
tags: [MOC, 工程基础, 底层原理, 导航]
created: 2026-07-09
updated: 2026-07-09
---

# 🔧 MOC：LLM 工程基础与底层机制

> **从"这个技术叫什么"到"它的公式/代码到底怎么写"。** 本 MOC 汇总一次性消化自 [Alisa's Book of LLMs](https://alisawuffles.notion.site/alisa-s-book-of-llms)（NLP 研究者 Alisa Liu 的个人技术笔记）的全部内容——新建了 `00_LLM工程基础` 簇，并给 9 个既有页面（RoPE、Flash Attention、投机解码、KV 缓存、自注意力机制、SSM、RLHF、GRPO、DPO）追加了更严谨的机制与公式补充。这批内容偏数学/工程推导，是给"想把 LLM 面试聊到实现细节层面"的 AI PM 准备的底层弹药库，不是产品视角的论文笔记，请配合 [[Wiki/概念导航索引]] 使用。

---

## 一、神经网络与数学基础（新簇：00_LLM工程基础）

- [[Wiki/概念/00_LLM工程基础/多层感知机与线性层]] — 从单神经元到批量矩阵乘法的形状推导
- [[Wiki/概念/00_LLM工程基础/激活函数选择]] — sigmoid/tanh/ReLU/Swish/SwiGLU 谱系
- [[Wiki/概念/00_LLM工程基础/反向传播与自动微分]] — 链式法则、计算图、softmax+CE 梯度的干净结果
- [[Wiki/概念/00_LLM工程基础/Adam优化器与学习率调度]] — Adam/AdamW/梯度裁剪
- [[Wiki/概念/00_LLM工程基础/信息论基础(交叉熵与KL散度)]] — CE = KL + 熵
- [[Wiki/概念/00_LLM工程基础/数值稳定性技巧(LogSumExp与在线Softmax)]] — FlashAttention 数学基础的来源
- [[Wiki/概念/00_LLM工程基础/评测统计检验方法]] — 判断"模型A比模型B好"是否显著的方法论
- [[Wiki/概念/00_LLM工程基础/采样的可微化(Gumbel-Softmax与直通估计)]] — 让离散采样可以传梯度

## 二、Transformer 实现细节（01_架构技术，新建）

- [[Wiki/概念/01_架构技术/Transformer完整前向传播与实现细节]] — 一层 Transformer 从 embedding 到 logits 的完整链路
- [[Wiki/概念/01_架构技术/Transformer显存与算力核算]] — 参数量/FLOPs/训练与推理显存怎么估
- [[Wiki/概念/01_架构技术/RMSNorm与LayerNorm]]
- [[Wiki/概念/01_架构技术/SwiGLU门控前馈网络]]
- [[Wiki/概念/01_架构技术/RNN与LSTM]] — 含"RNN 表达能力边界"（正则语言/DFA 编码）

## 三、既有页面的机制补强（本次「扩展」，原有内容未改动，只追加了"补充"小节）

| 页面 | 本次追加了什么 |
|------|----------------|
| [[Wiki/概念/01_架构技术/Flash Attention]] | 4GB 显存问题的精确计算、SRAM/HBM 读写瓶颈 |
| [[Wiki/概念/01_架构技术/RoPE旋转位置编码]] | 相对位置不变性的完整证明、复数重述、工程实现 |
| [[Wiki/概念/01_架构技术/投机解码]] | 接受概率=目标分布的完整证明 |
| [[Wiki/概念/05_记忆与检索/上下文压缩与KV缓存]] | KV cache 公式、MQA/GQA/MLA、跨层与局部窗口 |
| [[Wiki/概念/01_架构技术/自注意力机制]] | causal mask、QK-Norm、GQA reshape、滑窗与稀疏注意力 |
| [[Wiki/概念/01_架构技术/SSM状态空间模型]] | SSM/Transformer/RNN 三方复杂度对比表 |
| [[Wiki/概念/02_训练方法/RLHF与RLAIF]] | KL 惩罚公式、Bradley-Terry 奖励模型训练推导 |
| [[Wiki/概念/02_训练方法/GRPO优化算法]] | 组内相对优势公式、Dr.GRPO 长度/难度偏差修正 |
| [[Wiki/概念/02_训练方法/DPO直接偏好优化]] | RLHF目标→闭式解→DPO loss 的完整推导链条 |

## 四、后训练算法基础 + 断链修复（02_训练方法）

- [[Wiki/概念/02_训练方法/PPO近端策略优化]] — policy gradient/baseline/重要性采样/PPO裁剪，是 RLHF/GRPO/DPO 共同的前置知识（修复了 [[Wiki/概念/02_训练方法/预训练范式]] 里原有的断链）
- [[Wiki/概念/02_训练方法/Scaling Law]] — μP + LR/Loss 与算力的幂律拟合方法论（修复断链，与「预训练范式」的 Chinchilla 实践结论互补）

## 五、并行与推理服务（00_LLM工程基础 + 03_推理与评测）

- [[Wiki/概念/00_LLM工程基础/混合精度训练]] — 训练时精度策略（区别于推理量化）
- [[Wiki/概念/00_LLM工程基础/分布式训练通信原语与数据并行]] — collective ops + ZeRO 1/2/3
- [[Wiki/概念/00_LLM工程基础/流水线并行与张量并行]]
- [[Wiki/概念/03_推理与评测/推理批处理与采样策略]] — continuous batching + temperature/top-k/top-p

## 六、多模态（06_多模态生成，断链修复）

- [[Wiki/概念/06_多模态生成/多模态对齐与CLIP]] — ViT/LLaVA/LLaVA-NeXT/Qwen2-VL 2D-RoPE/CLIP（修复断链）

---

## 场景化阅读路径

**路径 A · 面试被追问"你真的懂 LLM 训练机制吗"（4 站，约 40 分钟）**
1. [[Wiki/概念/01_架构技术/Transformer完整前向传播与实现细节]] — 先搞懂一层 Transformer 到底在算什么
2. [[Wiki/概念/02_训练方法/PPO近端策略优化]] → [[Wiki/概念/02_训练方法/RLHF与RLAIF]] → [[Wiki/概念/02_训练方法/GRPO优化算法]] — 后训练算法的演进逻辑
3. [[Wiki/概念/01_架构技术/Transformer显存与算力核算]] — 能不能当场估算"7B 模型部署要多少显存"
4. [[Wiki/概念/00_LLM工程基础/混合精度训练]] — 训练成本怎么压下来

**路径 B · 判断一个推理服务/API 报价是否合理（3 站）**
1. [[Wiki/概念/03_推理与评测/推理批处理与采样策略]]
2. [[Wiki/概念/05_记忆与检索/上下文压缩与KV缓存]]（含新补充的 MQA/GQA/MLA）
3. [[Wiki/概念/01_架构技术/Transformer显存与算力核算]]

**路径 C · 从零搞懂 RLHF/DPO/GRPO 三者的关系（4 站）**
1. [[Wiki/概念/00_LLM工程基础/反向传播与自动微分]] — 先有梯度/链式法则的底子
2. [[Wiki/概念/02_训练方法/PPO近端策略优化]] — policy gradient 是三者的共同祖先
3. [[Wiki/概念/02_训练方法/RLHF与RLAIF]] — PPO 加奖励模型
4. [[Wiki/概念/02_训练方法/GRPO优化算法]] 与 [[Wiki/概念/02_训练方法/DPO直接偏好优化]] — 两条不同的简化路径（去 Critic vs 去奖励模型）

---

📅 **最后更新**：2026-07-09
> 全部内容源自单一笔记 [Alisa's Book of LLMs](https://alisawuffles.notion.site/alisa-s-book-of-llms)，本地存档见 `RAW/alisa_book_of_llms.md`。这份笔记数学密度高、覆盖面广但部分小节（如 GPU 硬件、学习率 warmup）本身内容单薄，已在对应页面中如实标注，不是消化时的遗漏。
