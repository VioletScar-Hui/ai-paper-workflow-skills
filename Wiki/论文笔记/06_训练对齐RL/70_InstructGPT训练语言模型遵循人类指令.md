---
tags: [论文笔记, InstructGPT, RLHF, 人类对齐, PPO, ChatGPT, Bradley-Terry, OpenAI]
paper_id: "70"
笔记层级: 骨干
复核日期: 2026-07-04
---

# InstructGPT：训练语言模型遵循人类指令

📄 **原文 PDF**：[[RAW/70 - Training language models to follow instructions with human feedback.pdf]]

## PM 速判
> ChatGPT 的直接前身，RLHF 的奠基工程论文。提出 SFT→RM→PPO 三步流程把 GPT-3 变成会遵循指令的助手：1.3B InstructGPT 以 85% 频率被偏好于 175B GPT-3，代价是公开 NLP 基准轻微退步（"对齐税"）。这套方法直接催生了 ChatGPT。

## 双层费曼 🗣️
> **给CEO**：GPT-3 像个博学但不可控的天才——什么都知道，但经常胡说八道、输出有害内容、或者答非所问。InstructGPT 用三步调教法解决这个问题：第一步，雇人写标准答案教模型什么叫"好回答"；第二步，让模型生成多个回答，雇人排序好坏，训练一个"品味判官"；第三步，用判官打分来继续优化模型。结果：一个只有 GPT-3 百分之一大小的模型，85% 的情况下人类更愿意用它的回答。这就是 ChatGPT "听话"的秘密。
> **给工程师**：三步 RLHF 流程——(1) SFT：40 名标注员为 13K prompt 写示范回答，对 GPT-3 做监督微调；(2) RM：标注员对同一 prompt 的 4-9 个模型输出排序，用 Bradley-Terry 偏好模型训练 6B 奖励模型，loss = −log(σ(r_θ(x, y_w) − r_θ(x, y_l)))，即最大化"好回答得分 − 差回答得分"的对数几率；(3) PPO：以奖励模型信号通过 PPO 优化 SFT 模型，目标函数 = E[r_θ(x,y)] − β·KL[π_φ(y|x) || π_SFT(y|x)]，KL 惩罚阻止策略偏离 SFT 太远导致 reward hacking。

## 问题域定位 🎯
- **回应什么根本约束？** "有能力"和"有帮助"之间有鸿沟。LLM 的预训练目标是 next-token prediction（最大化文本似然），但用户期望的是"安全、诚实、有用、遵循意图"。这两者非但不等价，甚至冲突——预训练学到的很多模式恰好是用户不想要的（如编造事实、跟随有害引导、产出冗长无意义的文本）。
- **之前卡在哪？** GPT-3 展示了大模型的强大 in-context learning 能力，但其输出质量高度依赖 prompt 工程，且本质上不可控——用户无法告诉模型"什么才是好回答"。大家知道 RL 能优化任意 reward 函数，但不知道如何为"语言质量"定义一个可计算的 reward。关键卡点是：reward 信号从哪里来？如何在不引入 reward hacking 的前提下优化？
- **开启/关闭了哪条路线？** 开启：RLHF 作为标准的 LLM 对齐范式（被 Anthropic、Meta、Google 等所有主流玩家采用），"用人类偏好数据训练奖励模型 → 用 RL 优化策略"的两阶段路线，偏好排序标注的工程实践（排序比打分一致性更高）。关闭：纯 prompt engineering 作为让模型"变好"的主要手段——RLHF 证明训练侧的干预远远超越推理侧的 prompt 技巧。**未关闭但持续争议**：RLHF 是否过度限制模型创造力？对齐税能否在更大模型上消失？

## 核心机制
```
    ┌──────────────────────────────────────────────────────┐
    │                  RLHF 三步流程                        │
    └──────────────────────────────────────────────────────┘

    GPT-3 (预训练基座, 无法遵循指令)
    │
    │  Step 1: SFT (Supervised Fine-Tuning)
    │  标注员为 13K prompt 写高质量示范回答
    │  用 next-token prediction 微调 → 初步对齐
    ▼
    π_SFT (SFT 模型, 能理解指令但质量不够高)
    │
    │  Step 2: RM (Reward Model) 训练
    │  对同一 prompt 生成 4-9 个回答，标注员排序 (配对偏好)
    │  Bradley-Terry: P(y_w ≻ y_l | x) = σ(r(x,y_w) − r(x,y_l))
    │  loss = -E[log σ(r(x,y_w) - r(x,y_l))]     ← 成对排序损失
    ▼
    r_θ(x, y)  (6B 奖励模型, 输入 prompt+回答 → 标量分数)
    │
    │  Step 3: PPO 强化学习
    │  π_φ 初始化为 π_SFT
    │  maximize E[r_θ(x,y)] − β·KL[π_φ || π_SFT]
    │  └────奖励────┘   └───KL 惩罚, 防止偏离太远──┘
    │  PPO clipped objective + 预训练梯度混合 (PPO-ptx)
    ▼
    π_PPO (InstructGPT, 对齐后策略)

    关键约束:
    — KL 惩罚 β 控制: 太弱 → reward hacking (刷分而非提升质量)
                      太强 → 没学到新东西, ≈ SFT 模型
    — RM 是 6B (比 175B 策略小很多): 防止 RM 本身的 reward hacking
    — PPO-ptx: 混合预训练数据 loss → 缓解"对齐税"
```

## 设计决策解剖 ⚖️
| 决策点 | 选择 | 被放弃方案 | 为什么 | 何时失效 |
|--------|------|----------|--------|---------|
| 偏好信号形式 | 成对排序（A 比 B 好） | 绝对打分（1-7 李克特量表） | 排序: 标注者间一致性高（Kappa ~0.7），认知负担低（比"打几分"更自然）。绝对分: 不同人的分数不可比，需要大量校准 | 当 K 个回答的质量高度相近时排序变成"随机猜"，信号噪声比下降。另外当回答很多（>10）时排序成本 O(K²)，不如绝对打分 O(K) |
| RM 架构 | 6B 独立模型，从 SFT 初始化后在 [CLS] 头加线性层 | (1) 微调更大的 175B 模型，(2) 用 SFT 模型本身作为 RM | 6B 足够捕获人类偏好的主要变化，且推理快；175B RM 训练慢且容易成为策略的 "reward oracle"（策略找到 RM 盲区钻空子）。策略和 RM 独立可避免 reward circularity | 当偏好判断需要更强的语义理解时（如复杂代码审查），6B RM 的容量可能不够 |
| KL 惩罚 | 加在 PPO 目标里（−β·KL），β 动态调整 | 不加 KL（纯 RL 优化 reward）或 固定 β | 无 KL → reward hacking 必然发生（模型发现生成乱码/极度谄媚的文本能骗过高分的 RM）。固定 β → 不同 prompt 对"偏离 SFT"的容忍度不同 | 动态 β 调整不够快时——如果 RM 突然喜欢某种新回答风格，KL 惩罚可能在策略学到它之前就被压回去 |
| PPO-ptx 混合 | 将预训练梯度混入 PPO 更新（系数 γ） | 只用 PPO，不加预训练 loss | 纯 RL 优化让模型在公开 NLP 基准（SQuAD 等）上退步——模型"忘记"预训练学到的泛化能力。混入预训练数据像"锚"一样保持知识 | γ 太大则 RL 信号淹没，太小则对齐税不缓解。需要针对每个任务调优，不是 one-size-fits-all |
| 数据收集条件 | API 用户的真实 prompt（去重后 31K）+ 标注员自写 prompt | 只用标注员写的 prompt | 真实 prompt 分布比标注员的想象更接近"用户实际要什么"——评估的是分布内泛化而非实验室设定 | 真实 prompt 中可能包含敏感内容、低质量输入，标注员不愿意标注——需要额外的过滤和 safety review |

## 成本与量级 💰
- **数据成本**（核心成本，远超计算）：
  - SFT: 40 名标注员 × 13K prompt × 每人手写示范回答 → 估计数百人天，按美国标注市场价 ~$50/hr 算，约 $100K-200K
  - RM: 33K prompt × 每个 prompt 4-9 回答排序 → 排序比 SFT 快（不需要写，只需要读+比较），估计 ~200K 次比较，约 $50K-100K
  - PPO: 31K 真实 API prompt + 标注员自写 prompt → 不需要额外标注（RL 自动采样），成本主要在前两步
- **训练计算**：三步都在 GPT-3 175B 级别做微调/RL。按 2022 年的云计算成本，估计 $50K-200K 总计。
- **推理**：训练后的 InstructGPT 仍是 175B（PPO 版本）或更小（1.3B/6B），与标准 GPT-3 推理成本一致。
- **最小可行配置**：用 GPT-2 级别（~1.5B）做 RLHF 在 IMDB 情感控制或 TL;DR 摘要任务上验证三步流程可行，估计 1 人周标注 + 1×A100 几天训练。

## 证据审计 🔬
- **实验公平性**：对比基线合理——GPT-3 zero-shot/few-shot、FLAN、T0、SFT-only（无 RLHF）。但标注员本身就是"评判者"，而 SFT 数据也来自标注员——存在标注员偏好闭环风险：标注员定义"好"标准，同时也在评估"好"标准，模型本质是在迎合这一小群人的品味。
- **最强证据**：1.3B InstructGPT vs 175B GPT-3 在 85% 对比中被偏好——即使参数量小 100 倍，对齐后的模型也更受人类青睐。这是"对齐 > 规模"的最强量化证据。TruthfulQA 真实性提升约 2 倍（从 ~20% → ~40%），毒性产出显著降低。
- **最可疑数字及原因**：(1) "对齐税"的量化不完整——论文报告了一些 NLP 基准退步，但没有系统性的"能力退化图谱"。哪些能力退了、退了多少、能否恢复，文档不充分。(2) 标注员群体（40 人，主要为英语母语者）的偏好可能不代表目标用户——例如中国用户对"礼貌"的期望不同，可能不喜欢过于正式的回应风格。(3) RM 的准确率仅在标注员级别——RM 在预测人类排序时准确率约 70%，这意味着 30% 的 RL 奖励信号是噪声，PPO 在优化部分"错误"的信号。
- **审稿补充要求**：缺少对"标注员群体偏好偏差"的消融——如果换一批标注员，RM 和最终策略会有多大变化？缺少对"reward hacking 的检测和量化"——哪些回答实为 reward hacking 产物？缺少对齐税的恢复方案——PPO-ptx 之外的替代策略？
- **最小复现设计**：在 TL;DR 摘要任务上复现 RLHF——用 T5-base + 简单 RM（语言模型 + MLP head）+ 30 条人工排序数据 + PPO。OpenAI 后公开的 Triton RLHF 教程已将此流程标准化。

## 可复用点
- **何时采用**：(1) 构建对话型 LLM 产品时，RLHF 是当前最成熟的对齐方案（如果预算充足）；(2) 用户反馈"模型有时好有时差"但无法量化为规则时，偏好排序数据是收集隐式质量标准的有效手段；(3) SFT 模型已经不错但总觉得"差一点"——RLHF 的边际收益在 SFT 质量基础上最明显。
- **何时规避**：(1) 预算有限且标注力量不足——RLHF 的三步都需要高质量人工标注，低成本替代方案如 DPO 可用偏好数据直接优化策略，省掉 RM+PPO；(2) 输出空间的正确性有明确可计算的验证机制（如代码执行、数学验证）——可验证 reward 比学习 reward 更可靠；(3) 领域极其专业（如医学诊断）——标注员不具备评判能力，直接 RLHF 可能引入噪声甚至危险信号。
- **供应商拷问清单**：
  1. "你们的 RLHF 标注员是什么背景？有多少人？他们能代表目标用户群体吗？"
  2. "奖励模型的偏好预测准确率是多少？在哪些类型的输入上最不准？"
  3. "RLHF 前后，你们在公开 NLP 基准上的'对齐税'是多少？哪些能力退步了？"
  4. "PPO 阶段 KL 惩罚系数 β 怎么调的？有没有出现过 reward hacking 的案例？"
  5. "你们的标注数据是否会周期性更新（用户偏好漂移）？如果不更新，模型过多久会'落伍'？"

## 关联网络 🕸️
- **相关论文**：
  - [[Wiki/论文笔记/01_LLM基础架构/65_FLAN微调语言模型是零样本学习者]] — FLAN 的指令微调是 InstructGPT 的 Step 1（SFT）的理论基础和方法来源。InstructGPT 在 FLAN 之上加了 RM+PPO，但 FLAN 的多任务指令化思想贯穿整个流程
  - [[Wiki/论文笔记/06_训练对齐RL/67_WebGPT浏览器辅助问答与人类反馈]] — WebGPT 在 InstructGPT 之前先驱了成对偏好 RM 训练（用于网页搜索问答），InstructGPT 将其推广到通用助手场景
  - [[Wiki/论文笔记/05_推理思维链/71_LetsVerifyStepByStep过程奖励模型]] — PRM 是将 InstructGPT 的"结果级 RM"细化为"步骤级 RM"的推理特化版。PRM 的 loss 同样基于 Bradley-Terry 偏好模型
  - [[Wiki/论文笔记/05_推理思维链/73_ReAct推理与行动的协同]] — ReAct 需要在语言模型基础上做 agent 决策，RLHF 确保 agent 的行为符合人类意图而非偏离
- **相关概念**：
  - [[Wiki/概念/02_训练方法/RLHF与RLAIF]] — InstructGPT 是 RLHF 的标志性实现；RLAIF 后来用 AI 反馈替代人工标注降低成本
  - [[Wiki/概念/02_训练方法/GRPO优化算法]] — GRPO 是 DeepSeek 对 InstructGPT PPO 步骤的改进——去掉独立 critic 模型，用组内比较替代 KL 约束
  - [[Wiki/概念/02_训练方法/DPO直接偏好优化]] — DPO 绕过 InstructGPT 的 RM+PPO 阶段，直接从偏好数据优化策略
  - [[Wiki/概念/02_训练方法/过程奖励模型PRM]] — PRM 是 InstructGPT RM 在推理步骤上的细粒度推广
  - [[Wiki/概念/03_推理与评测/Constitutional AI]] — CAI 继承 RLHF 三步框架，用宪法原则替代人工标注，解决标注扩展性问题
- **冲突/印证**：[[Wiki/论文笔记/01_LLM基础架构/65_FLAN微调语言模型是零样本学习者]] 印证了"SFT 是 LLM 对齐的关键第一步"——FLAN 证明纯指令微调就能解锁零样本泛化，InstructGPT 在此基础上证明 RLHF 能进一步将"能用"提升到"好用"。但两者在"最优数据量 vs 数据多样性"上存在张力——FLAN 强调覆盖 62 个任务族的多样性，InstructGPT 的 SFT 数据仅 13K 条但来自真实用户 prompt 分布，暗示"分布匹配可能比绝对多样性更重要"。后续 LIMA 论文（"少样本对齐"）从另一角度印证了后一个猜想——高质量但少量的 SFT 数据就够。

## 动手练习 💻
**练习目标**：实现 Bradley-Terry 偏好模型的损失函数，理解 RLHF 中奖励模型的核心训练逻辑。
```python
"""
Bradley-Terry 偏好模型损失函数实现

InstructGPT 的 RM (奖励模型) 使用 Bradley-Terry 模型对成对比较
进行建模:
  P(y_w ≻ y_l | x) = σ(r(x, y_w) − r(x, y_l))
  
其中:
  — y_w: 被人类标注为"更好"的回答 (winner)
  — y_l: 被人类标注为"更差"的回答 (loser)
  — σ: sigmoid 函数
  — r(x, y): 奖励模型给 (prompt x, 回答 y) 的打分

损失函数 (负对数似然):
  loss = −log σ(r(x, y_w) − r(x, y_l))

这个公式的直觉:
  — 如果 r(w) >> r(l)，σ → 1，loss → 0 (很好)
  — 如果 r(w) << r(l)，σ → 0，loss → +∞ (很差，模型学反了)
  — 如果 r(w) ≈ r(l)，σ → 0.5，loss ≈ 0.693 (区分度不够)
"""

import torch
import torch.nn as nn
import torch.nn.functional as F
import numpy as np


# ============ Bradley-Terry 损失实现 ============

def bradley_terry_loss(
    rewards_w: torch.Tensor,  # shape: (batch_size,)
    rewards_l: torch.Tensor,  # shape: (batch_size,)
) -> torch.Tensor:
    """
    Bradley-Terry 偏好模型的标准损失函数。
    
    计算过程:
    1. diff = reward_w − reward_l        # 分差
    2. prob = σ(diff)                     # winner 被偏好的概率
    3. loss = −log(prob)                  # 负对数似然
    
    参数:
        rewards_w: 每个样本中被偏好回答的奖励分数
        rewards_l: 每个样本中不被偏好回答的奖励分数
    
    返回:
        标量 loss (batch 内平均)
    """
    diff = rewards_w - rewards_l
    # 稳定性技巧: 用 logsigmoid 而不是 log(sigmoid(diff))
    # logsigmoid 在数值上更稳定 (避免 exp → inf)
    loss_per_sample = -F.logsigmoid(diff)
    return loss_per_sample.mean()


class BradleyTerryRM(nn.Module):
    """
    基于 Bradley-Terry 模型的简化奖励模型 (只训练线性层)。
    
    完整 RLHF 中 RM 通常是 transformer backbone + linear head。
    这里为了教学目的，简化为: 对 (prompt, response) 的特征向量
    做线性打分。
    """

    def __init__(self, feature_dim: int = 768):
        super().__init__()
        # 奖励模型: 线性层从特征向量映射到标量分数
        self.head = nn.Linear(feature_dim, 1)  # 输出一个标量

    def forward(self, features: torch.Tensor) -> torch.Tensor:
        """
        参数:
            features: (batch_size, feature_dim) 回答的特征表示
        返回:
            rewards: (batch_size,) 每个回答的标量分数
        """
        return self.head(features).squeeze(-1)  # (batch_size,)

    def compute_loss(
        self,
        features_w: torch.Tensor,  # winner 的特征
        features_l: torch.Tensor,  # loser 的特征
    ) -> torch.Tensor:
        """
        计算 Bradley-Terry 损失。
        """
        rewards_w = self.forward(features_w)
        rewards_l = self.forward(features_l)
        return bradley_terry_loss(rewards_w, rewards_l)


# ============ 仿真与验证 ============

def compute_accuracy(
    rewards_w: torch.Tensor, rewards_l: torch.Tensor
) -> float:
    """
    计算奖励模型的准确率: 它是否正确地给 winner 打了更高分?
    """
    correct = (rewards_w > rewards_l).float().mean().item()
    return correct


if __name__ == "__main__":
    print("=" * 60)
    print("Bradley-Terry 偏好模型: 损失函数演示")
    print("=" * 60)

    # 设置
    torch.manual_seed(42)
    batch_size = 8
    feature_dim = 768
    num_epochs = 200

    # 模拟训练数据: 每对 (winner特征, loser特征) 是一组偏好比较
    # 真实奖励信号: 我们构建的 winner 和 loser 是有区分度的
    # 让 winner 的第 0-99 维特征稍大 → 学习这些维度判别 better/worse
    features_w = torch.randn(batch_size, feature_dim)
    features_w[:, :100] += 0.3  # winner 在前 100 维略高
    features_l = torch.randn(batch_size, feature_dim)

    # 训练
    model = BradleyTerryRM(feature_dim=feature_dim)
    optimizer = torch.optim.Adam(model.parameters(), lr=0.01)

    print("\n训练过程:")
    print(f"  {'Epoch':>5} | {'Loss':>10} | {'Acc':>8}")
    print(f"  {'-'*5}-+-{'-'*10}-+-{'-'*8}")

    for epoch in range(num_epochs):
        optimizer.zero_grad()
        loss = model.compute_loss(features_w, features_l)
        loss.backward()
        optimizer.step()

        if epoch % 40 == 0 or epoch == num_epochs - 1:
            with torch.no_grad():
                rw = model.forward(features_w)
                rl = model.forward(features_l)
                acc = compute_accuracy(rw, rl)
            print(f"  {epoch:>5} | {loss.item():>10.4f} | {acc:>7.2%}")

    # 验证
    print(f"\n{'='*60}")
    print("验证与分析")
    print(f"{'='*60}")
    with torch.no_grad():
        rw = model.forward(features_w)
        rl = model.forward(features_l)
        diffs = rw - rl
        print(f"\n  Winner 平均分数: {rw.mean().item():.4f}")
        print(f"  Loser  平均分数: {rl.mean().item():.4f}")
        print(f"  平均分差:        {diffs.mean().item():.4f}")
        print(f"  准确率 (rw > rl): {compute_accuracy(rw, rl):.2%}")
        print(f"  最后 loss:        {loss.item():.4f}")

    print(f"\n关键解读:")
    print(f"  1. 分差 > 0 (越大越好): 模型学到了区分 winner/loser")
    print(f"  2. loss 持续下降: 梯度的方向是"拉大分差"")
    print(f"  3. 准确率 → 100%: 损失函数驱动 rw >> rl")
    print(f"  4. BT loss = -logsigmoid(rw-rl):")
    print(f"     当 rw-rl = -5 时, sigmoid≈0.007, loss≈5.0  (惩罚很重)")
    print(f"     当 rw-rl = +5 时, sigmoid≈0.993, loss≈0.007 (奖励很小)")
    print(f"     当 rw-rl =  0 时, sigmoid=0.5,  loss≈0.693 (区分不出来)")

    print(f"\n{'='*60}")
    print("Bradley-Terry vs 绝对打分 (MSE)")
    print(f"{'='*60}")
    print("如果不用 BT 而用 MSE (reward_w → 1, reward_l → 0):")
    print("  优点: 看似更直观")
    print("  缺点: 不同标注者对'绝对好坏'的标准不一致")
    print("        (A 觉得 7 分算好, B 觉得 5 分就算好)")
    print("  BT 通过 pairwise 比较避开了这个校准问题:")
    print("  只需判断 A > B, 不需要知道 A 和 B 各自该打多少分")
    print("  这就是 InstructGPT 选 BT 而非绝对打分的核心原因。")
```

## 自测三层 🎓
**L1 复述**：(1) 写出 Bradley-Terry 损失函数的公式，解释每个符号的含义。(2) RLHF 三步流程是什么？每一步用了什么数据、产出什么？

**L2 解释**：(1) 为什么 PPO 目标函数里要加 KL 惩罚 −β·KL[π_φ || π_SFT]？如果去掉会发生什么？(2) 1.3B InstructGPT 就能在 85% 对比中胜过 175B GPT-3——这个结果背后的深层含义是什么？是"对齐比规模重要"还是"RLHF 弥补规模差距"？

**L3 应用**：(1) 你要为一个医疗咨询 AI 设计 RLHF 管线。标注员需要具备什么资质？Bradley-Terry 排序在医学场景下有什么潜在问题？你会如何修改数据收集流程？(2) 你在生产中观察到"对齐税"——RLHF 后的模型在 SQL 生成能力上退步了 15%。有哪些工程手段可以缓解？你能从 InstructGPT 论文中找到什么已有的缓解方案？

📅 知识时间锚：2022-03（InstructGPT 发布），同年 11 月 ChatGPT 发布——将 RLHF 对齐的大模型首次以消费产品形式呈现给大众。
