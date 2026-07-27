---
tags: [论文笔记, DeepSeek-R1, 强化学习, 推理, CoT, GRPO, DeepSeek]
paper_id: "84"
笔记层级: 骨干
复核日期: 2026-07-04
---

# DeepSeek-R1：通过强化学习激励 LLM 的推理能力

📄 **原文 PDF**：[[RAW/84 - DeepSeek-R1 - Incentivizing Reasoning Capability in LLMs via Reinforcement Learning.pdf]]

## PM 速判
> **证明了推理能力可以从纯强化学习中涌现，无需人工标注推理轨迹。** DeepSeek 2025 年 1 月发布 R1，通过 GRPO（Group Relative Policy Optimization）训练 DeepSeek-V3 基础模型，使其自发产生自我反思、验证、动态策略调整等推理行为。R1 在 AIME 2024 上达到 79.8%（vs o1-mini 63.6%），代码竞赛、数学等任务全面对标 OpenAI o1。同步开源了 R1-Distill 系列蒸馏模型（1.5B-70B）。

| 年份 | 成熟度 | PM 相关性 |
|------|--------|---------|
| 2025-01 | ✅ | **最高**（推理模型时代的奠基论文，o1 的开源对标）|

## 双层费曼
> **给CEO**：以前要让 AI 学会"一步步推理"，需要雇几百人手工写几万道题的详细解题步骤。DeepSeek-R1 证明，只需要告诉 AI 最终答案对不对（像批改选择题一样简单），AI 就能自己琢磨出推理过程——甚至会自己发现"等等，让我重新想想"这种反思能力。这意味着任何有标准答案的领域（数学、代码、合同审查），都可以低成本训练出推理专家。
>
> **给工程师**：核心算法 GRPO 替代了传统 RLHF 中的 PPO。PPO 需要同时维护 Actor、Critic、Reference、Reward 四个模型，GRPO 对同一问题采样 G 个答案，用组内均值/标准差归一化奖励作为优势估计，省掉 Critic 网络（节省约 50% 训练显存）。奖励设计只用结果正确性 + 格式合规两项，没有过程奖励模型（PRM）。训练分四阶段：Cold Start SFT（少量高质量 CoT 防早期崩溃）→ 纯 RL（GRPO）→ 拒绝采样 SFT → 再次 RL（加入帮助性/无害性）。

## 问题域定位
- **根本约束**：推理能力（特别是多步推导、自我验证、回溯纠错）此前被认为是"标注出来的"——需要人工编写详细推理链，成本高、天花板低
- **之前卡点**：o1 证明了推理模型可行，但技术细节（如何训练、需要什么奖励信号、是否需要 PRM）完全不公开；学术界只能猜测
- **R1 开启的路线**：纯结果奖励 + GRPO → 推理能力自发涌现。证明"有验证器的任务就能训练推理"，极大降低推理模型门槛。开源蒸馏模型（1.5B-70B）让推理能力可高效迁移至小模型
- **R1 关闭的路线**：PRM（过程奖励模型）是训练推理模型的必需品——R1 证明结果奖励足够；"推理必须从人类标注的推理链中学习"——R1-Zero 证明纯 RL 也能涌现

## 核心机制

```
DeepSeek-R1 四阶段训练流程：

阶段 0: Cold Start SFT
  ┌──────────────────────────────────────────┐
  │ 数千条高质量 CoT 样本（人工标注）         │
  │ 目的：防止 RL 早期探索崩溃               │
  │ 输出格式统一、语言一致                    │
  └──────────────────────────────────────────┘
                    ↓
阶段 1: 纯 RL 训练（GRPO）
  ┌──────────────────────────────────────────┐
  │ 算法: GRPO（Group Relative Policy Opt）   │
  │ 奖励: 答案正确性 + 格式合规               │
  │ 无 PRM（过程奖励模型）                    │
  │ 每组采样 G=64 个回答                      │
  │ 优势: Â_i = (R_i - mean(R_group)) / std  │
  │ 省去 Critic 网络 (~50% 训练显存)          │
  │                                           │
  │ 涌现行为: 自我反思、验证、回溯、          │
  │          "aha moment"（顿悟式自我纠正）    │
  └──────────────────────────────────────────┘
                    ↓
阶段 2: 拒绝采样 SFT
  ┌──────────────────────────────────────────┐
  │ 从阶段1模型采样大量推理链                 │
  │ 过滤: 答案正确 + 推理过程可读 + 语言一致  │
  │ 加入非推理数据（写作/问答/翻译等）        │
  │ 用筛选后的高质量数据做 SFT                │
  └──────────────────────────────────────────┘
                    ↓
阶段 3: 全场景 RL
  ┌──────────────────────────────────────────┐
  │ 加入帮助性奖励 + 无害性奖励               │
  │ 对齐人类偏好（不只是推理正确）            │
  │ 输出: DeepSeek-R1                        │
  └──────────────────────────────────────────┘

R1-Zero（纯 RL 无 Cold Start）对比：
  优势: 自发出现 "aha moment"（纯 RL 涌现自我反思）
  缺陷: 输出格式混乱、中英混杂、可读性差
  → Cold Start 解决了可读性问题

蒸馏系列：
  R1 推理链 → SFT Qwen/Llama 小模型
  R1-Distill-Qwen-7B > 同规模直接 RL 训练
  原因: 蒸馏学到了 671B 模型的推理模式，而非从头探索
```

## 设计决策解剖

| 决策点 | 选择 | 被放弃方案 | 为什么 | 何时失效 |
|--------|------|-----------|--------|---------|
| RL 算法 | GRPO（组内相对优势） | PPO / DPO / REINFORCE | PPO 需同时维护 4 个模型（Actor/Critic/Ref/Reward），显存开销大且 Critic 在推理任务中价值函数估计不稳定。GRPO 用组内统计替代 Critic，适合"同一问题可采样多个答案"的场景 | 任务无法对同一输入采样多个独立输出时（如连续对话），组内归一化失去意义，需退回到 PPO |
| 奖励设计 | 纯结果奖励（正确性+格式） | PRM（过程奖励）+ 结果奖励 | PRM 需要人工标注中间步骤的正确性，成本极高且可能引入标注者偏见。R1 证明在数学/代码这类有明确对错的领域，结果信号足够驱动推理涌现 | 开放域任务（创意写作、策略建议）没有客观"对错"时，结果奖励不可用，需回归偏好标注或 LLM-as-Judge |
| Cold Start | 数千条人工 CoT 热身 | 纯 RL 从零开始（R1-Zero）/大规模 SFT 预训练 | 纯 RL 早期输出不可读（中英混杂/格式混乱），大规模 SFT 成本高；数千条刚好解决可读性问题，成本可控 | 基础模型本身已有强推理能力时（如从 R2 升级），Cold Start 可能冗余 |
| 蒸馏 vs 直接 RL | 用 R1 推理链蒸馏小模型 | 对小模型直接做 GRPO | 小模型参数量不足以独立探索复杂推理空间；蒸馏让 7B 模型"站在巨人肩膀上"，直接学到了 671B 已发现的推理模式 | 当推理任务与 R1 的训练分布差异极大时，蒸馏的迁移效果可能不如领域内直接 RL |
| 阶段 2 数据混合 | 推理数据 + 非推理数据混合 SFT | 纯推理数据 | 纯推理数据会让模型在非推理任务（对话/写作）上退化；混合数据保持通用能力 | 如果是专用推理模型（如仅用于数学竞赛），纯推理数据效率更高 |

## 成本与量级

| 指标 | DeepSeek-R1 | R1-Distill-Qwen-7B | R1-Distill-Qwen-32B |
|------|------------|-------------------|---------------------|
| 基础模型参数量 | 671B (V3) | 7B | 32B |
| 训练显存（GRPO） | ~2x V3 推理（约 16×H100） | — | — |
| 推理成本（相对 o1） | 开源/自部署 | 极低（单 GPU） | 低（单 GPU） |
| GRPO 采样数 G | 64 | — | — |
| 省去 Critic 节省的显存 | ~50% vs PPO | N/A（蒸馏） | N/A（蒸馏） |

- **训练优势**：GRPO 比 PPO 节省约一半训练显存（无 Critic + 无 Reference 持续前向），同一硬件下可训练更大 batch
- **推理成本**：开源 + 可自部署，相比 o1 API 按 token 计费（含隐藏推理 token），总成本可降 90%+
- **蒸馏成本**：从 R1 采样推理链 + SFT 小模型，7B 模型单卡几小时即可完成

## 证据审计

### 实验公平性
- 与 o1/o1-mini 对比时，R1 使用多数投票（majority voting @64），o1 的采样配置未完全公开——可能高估 R1 优势约 3-5pp
- AIME 2024 是数学竞赛题，benchmark contamination 风险存在（V3 预训练数据可能含类似题目）

### 最强证据（数字 + 条件）
- **AIME 2024: R1 79.8% vs o1-mini 63.6%**（+16.2pp）——AIME 是美国数学邀请赛最难级别，每题需多步推导，领先幅度显著
- **MATH-500: 97.3% vs o1-mini 90.0%**（+7.3pp）——接近饱和，天花板效应
- **蒸馏效率**: R1-Distill-Qwen-7B 超越同规模直接 RL 训练——证明推理能力可通过蒸馏高效迁移，不是必须在大模型上做 RL
- 条件：上述均使用 majority voting，单次推理分数会低

### 最可疑数字及原因
- **"aha moment" 真的是涌现吗？**论文展示了一个定性案例（模型输出中突然出现"wait, let me reconsider"），但缺乏统计分析（出现频率、触发条件、与奖励的因果关系）。可能是模型在预训练数据中已见过类似模式，RL 只是激发而非创造
- **AIME 2024 benchmark contamination**：V3 的 14.8T tokens 预训练语料中是否包含 AIME 题目？论文未做充分的 decontamination 分析。如果存在泄露，79.8% 可能包含记忆成分而非纯推理
- **SWE-Verified 49.2%** 的具体 scaffold 配置未完整披露：用了什么代码执行环境？给了多少次尝试机会？这些工程细节对分数影响很大

### 审稿补充
- 应补充 GRPO 采样数 G 的消融实验（G=4/16/64/256），展示优势估计方差 vs 计算成本的 tradeoff
- 应补充推理链长度与正确率的关系图——长推理链是否真的更高正确率（验证 CoT 有效性）
- 跨领域迁移实验：数学训练的 GRPO 模型在代码推理上是否有正向迁移？

### 最小复现实验
在 GSM8K（小学数学）上用 7B 模型复现 GRPO 核心效果：一半用 Cold Start SFT → GRPO，一半只用 SFT。对比两组的最终准确率和推理链长度。预期 GRPO 组推理链更长（自我验证步骤多），准确率更高。

## 可复用点

### 何时采用
- 任务有客观可验证的输出（数学答案、代码测试用例、格式检查）
- 需要推理能力但不想标注大量推理链
- 基础模型已有一定能力，RL 来"激发"而非从零训练

### 何时规避
- 任务无客观对错（创意写作、情感支持）
- 基础模型过弱（<3B），探索空间太大导致 RL 无法收敛
- 推理链必须是特定格式（法律引用格式等）——GRPO 不约束推理风格

### 供应商拷问清单
1. "你们的推理模型是用 GRPO/PPO 训练的，还是用 o1 蒸馏的？"→ GRPO 自主训练 > 蒸馏
2. "奖励函数包括哪些维度？有没有过程奖励？"→ 了解其推理质量保证机制
3. "推理 token 数平均是多少？和输出 token 的比例？"→ 推理效率评估
4. "做了哪些 benchmark decontamination？"→ 关键信任问题

## 关联网络

- **前置**：[[Wiki/论文笔记/05_推理思维链/68_ChainOfThought思维链提示激发大模型推理]] — CoT 的 prompting 前身，R1 将 CoT 从"提示技巧"升级为"训练目标"
- **前置**：[[Wiki/论文笔记/06_训练对齐RL/70_InstructGPT训练语言模型遵循人类指令]] — RLHF 的 PPO 基础，GRPO 是对 PPO 的简化改进
- **并行**：[[Wiki/论文笔记/05_推理思维链/71_LetsVerifyStepByStep过程奖励模型]] — PRM 路线，R1 证明结果奖励可以替代 PRM
- **依赖**：[[Wiki/论文笔记/02_前沿模型报告/85_DeepSeek-V3技术报告]] — R1 的基础模型
- **冲突/印证**：OpenAI o1 的技术报告至今未公开训练细节，但 R1 的 GRPO + 结果奖励方案在 AIME 上超越 o1-mini，**印证**了"结果奖励足够驱动推理涌现"的假设。另一方面，o1 在 GPQA Diamond 上 77.3% vs R1 71.5%，**暗示**在需要领域知识的推理任务上，o1 的（未公开）训练方案可能有额外优势
- **概念**：[[Wiki/概念/02_训练方法/GRPO优化算法]]、[[Wiki/概念/03_推理与评测/思维链与推理模型]]、[[Wiki/概念/02_训练方法/DPO直接偏好优化]]、[[Wiki/概念/03_推理与评测/蒙特卡洛树搜索MCTS]]

## 动手练习

```python
"""
练习：numpy 实现 GRPO 组内优势估计
目标：理解 GRPO 如何用组内统计替代 Critic 网络估计优势函数
"""
import numpy as np

def grpo_advantage(rewards, group_size=4, epsilon=1e-8):
    """
    GRPO 组内相对优势估计
    参数:
        rewards: shape (N,) 所有样本的奖励值
        group_size: 每组采样数 G（同一问题的不同回答）
        epsilon: 防止除零
    返回:
        advantages: shape (N,) 每个样本的优势估计
    """
    n = len(rewards)
    assert n % group_size == 0, f"样本数 {n} 必须整除 group_size {group_size}"
    n_groups = n // group_size

    # 重塑为 (n_groups, group_size)
    r_groups = rewards.reshape(n_groups, group_size)

    # 组内均值 & 标准差
    group_mean = r_groups.mean(axis=1, keepdims=True)  # (n_groups, 1)
    group_std  = r_groups.std(axis=1, keepdims=True)   # (n_groups, 1)

    # GRPO 优势: (R_i - mean(R_group)) / std(R_group)
    advantages = (r_groups - group_mean) / (group_std + epsilon)

    return advantages.flatten()

# ===== 对比 PPO 方案（简化示意：用全局 baseline） =====
def ppo_style_advantage(rewards, baseline=None):
    """PPO 需要 Critic 网络估计 value，这里简化为全局均值 baseline"""
    if baseline is None:
        baseline = rewards.mean()
    return rewards - baseline

# ===== 模拟实验 =====
np.random.seed(42)

# 模拟 4 个问题，每个问题采样 4 个回答
# 问题难度不同 → 奖励分布不同
rewards_hard_q   = np.array([0.1, 0.3, 0.2, 0.15])   # 难题：整体低
rewards_medium_q = np.array([0.5, 0.7, 0.4, 0.6])    # 中题：中等
rewards_easy_q   = np.array([0.9, 0.85, 0.95, 0.8])  # 易题：整体高
rewards_mixed_q  = np.array([0.3, 0.9, 0.4, 0.2])    # 混合：有最好也有最差

all_rewards = np.concatenate([rewards_hard_q, rewards_medium_q,
                               rewards_easy_q, rewards_mixed_q])

# GRPO 优势
grpo_adv = grpo_advantage(all_rewards, group_size=4)
# PPO 风格优势（全局 baseline）
ppo_adv = ppo_style_advantage(all_rewards)

print("=" * 65)
print("样本 | 奖励  | GRPO优势 | PPO优势 | 解释")
print("-" * 65)
for i in range(len(all_rewards)):
    grpo_sign = "✓(好)" if grpo_adv[i] > 0 else "✗(差)"
    ppo_sign  = "✓(好)" if ppo_adv[i] > 0 else "✗(差)"
    remark = ""
    if grpo_adv[i] > 0 and ppo_adv[i] < 0:
        remark = "← GRPO 发现组内优秀，PPO 因全局均值而惩罚"
    elif grpo_adv[i] < 0 and ppo_adv[i] > 0:
        remark = "← PPO 因低全局均值而奖励，但组内相对较差"
    print(f"  {i:2d}  | {all_rewards[i]:.3f} | {grpo_adv[i]:+7.3f} | {ppo_adv[i]:+7.3f} | {remark}")

# 关键洞察：难题组(样本0-3)在全局 PPO 下因低于全局均值被惩罚，
# 但 GRPO 只和同题比较——难题组内 0.3 就是"好答案"！
print("\n关键洞察：GRPO 的"组内相对"避免了对困难问题的系统性惩罚。")
print("难题(0-3)的 GRPO 优势 > PPO 优势 → 鼓励探索困难问题。")
```

## 自测三层

**L1 记忆**：R1 训练的四阶段各是什么？GRPO 的全称是什么，它省掉了 PPO 的哪个组件？R1 在 AIME 2024 上的得分是多少（vs o1-mini）？

**L2 理解**：为什么 GRPO 用组内标准差归一化而不是全局归一化？Cold Start 解决了 R1-Zero 的什么问题？蒸馏方案为什么 7B 蒸馏 > 7B 直接 RL？

**L3 应用**：你要为公司的代码审查系统训练推理能力。你有 10 万道编程题的测试用例（可自动验证代码正确性），但没有任何人工推理链。设计一个训练方案：你会直接用 GRPO 还是先做 Cold Start？你需要多少条 Cold Start 数据？奖励函数怎么设计？如果发现模型推理链中英混杂，你的干预手段有哪些？

📅 知识时间锚：2025 年 1 月 DeepSeek 发布 R1，开源推理模型时代正式开启。此前推理能力被 o1 垄断且技术细节完全不公开。GRPO 成为后续几乎所有开源推理模型（Qwen3、LLaMA-4 等）的标准 RL 算法。

## 原子概念索引
- [[Wiki/概念/02_训练方法/GRPO优化算法]] — R1 的核心训练算法
- [[Wiki/概念/03_推理与评测/思维链与推理模型]] — CoT/推理模型演进位置
- [[Wiki/概念/01_架构技术/知识蒸馏]] — R1-Distill 系列通过推理轨迹蒸馏将 671B 模型能力迁移至 7B/70B 小模型
- [[Wiki/概念/02_训练方法/DPO直接偏好优化]] — R1 使用 GRPO 而非 DPO，展示了 DPO 离线优化在推理场景的局限性
- [[Wiki/概念/03_推理与评测/后训练推理数据]] — R1 的冷启动数据+拒绝采样+SFT再RL流程是后训练推理数据工程的前沿案例
- [[Wiki/概念/02_训练方法/过程奖励模型PRM]] — GRPO 的组内相对奖励与 PRM 的过程监督互补
- [[Wiki/概念/03_推理与评测/蒙特卡洛树搜索MCTS]] — GRPO 的组内相对奖励估计与 MCTS 的统计回溯共享多路径探索思想
