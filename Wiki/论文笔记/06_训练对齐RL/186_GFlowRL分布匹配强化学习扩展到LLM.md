---
tags: [论文笔记, 强化学习, GFlowNet, 分布匹配, 训练对齐, 后训练, MoE]
paper_id: "186"
filename: "186 - GFlowRL - Scaling Distribution-Matching RL to Large Language Models.pdf"
authors: "Xiaodong Liu, Michael Xu, Jack W. Stokes, Paul Smolensky, Doug Burger, Jianfeng Gao（Microsoft Research）"
year: 2026
笔记层级: 骨干
成熟度: 🧪
掌握度: 未拷问
复核日期: 2026-07-21
---

# 186 GFlowRL：把分布匹配强化学习扩展到大语言模型

📄 **原文 PDF**：[[RAW/186 - GFlowRL - Scaling Distribution-Matching RL to Large Language Models.pdf]]

> arXiv:2607.13394v1 (cs.CL)，2026-07-15。Microsoft Research。代码将开源于 github.com/microsoft/gflowrl

## PM 速判（30秒）

> 主流 RL 后训练（PPO/GRPO）是"奖励最大化"，会把概率质量堆到单一最优解上、坍缩解的多样性；GFlowNet 系（按奖励比例采样、保留所有高奖励路径）本可解决，但此前的 FlowRL 在大模型上训不稳。本文一句话诊断：**罪魁是那个"可学习配分函数 Z"**——它随机初始化、要在几百步内从零学会一个复杂量，而策略只是微调预训练模型，两者学习视野严重不匹配，Z 沦为噪声源，梯度范数比 GRPO 大 14 个数量级。GFlowRL 的解法极简：**用当前 rollout 组的批内蒙特卡洛均值直接估 Z，删掉整个辅助网络**，配两个稳定器。结果 14B 拿到 Codeforces 2048 Elo（距 o3-mini 仅 25 分），红队攻击和 235B MoE 上 FlowRL 全发散而它稳。PM 价值：这是"删掉一个组件反而更强"的教科书案例，也是"要多样性又要稳定"这对矛盾的当前最优解。

## 双层费曼 🗣️

> **给 CEO 的一句话**：训练推理模型有个老毛病——追求答对率会让模型只会一种解法，遇到变体就崩。有个更好的训练范式能让模型保留多种解法，但一直训不稳、烧钱还失败；我们找到了病根是一个多余的辅助零件，把它删掉换成一个几乎零成本的估算，模型立刻又稳又强，在编程竞赛上逼近 OpenAI 的顶尖闭源模型，还第一次在超大稀疏模型上训成功。

> **给工程师的一段话**：目标分布是 p(y|x) ∝ exp(β·r(x,y))，即按奖励的 softmax 采样而非 argmax。FlowRL/FOR 用 GFlowNet 的 trajectory balance（TB）目标逼近它，TB 里有个配分函数 Z(x)=Σexp(β·r)，FlowRL 把它参数化成"prompt 末隐状态上接 3 层 MLP"。本文的根因分析（learning-horizon mismatch）：策略从预训练 checkpoint 出发、几百步梯度更新就够；Z 是全新随机初始化、要在同样短窗口内从零学会一个 prompt-conditional 的复杂量——训练大部分时间里 log Z 就是 prompt 的随机函数。两个实证铁证：(1) 把 Z 换成 N(0.5,1) 高斯噪声，性能不降反升（36.19% vs 35.61%），说明 Z 没编码有用信息；(2) FlowRL 梯度范数均值 3.2×10¹⁴、421 步里 55 步爆炸超 10⁶，而 GRPO/GFlowRL 均值 0.24/0.07。解法：利用 TB 最优点性质——最优时对每条轨迹 log Z(x) 都等于 r(x,y)+log π_ref−log π_θ，于是直接用当前 rollout 组 G 个样本的批内均值当 Z 的估计（Eq.4），作为 stop-gradient 基线插进策略梯度。两个稳定器：① importance-sampling 修正 rollout 策略 π_old 与当前 π_θ 的漂移；② 非对称 flow-gap clipping（正向界 > 负向界，给正确但早期欠采样的解更大上推空间）。关键红利：Zt(x) 的尺度自动与 r/β、log π 同量级，梯度爆炸消失；且零额外参数、零单独学习率、零跨模块同步——基础设施与 GRPO 几乎一致。

## 问题域定位 🎯

- **根本约束**：RL 后训练是当代推理模型（o1/o3、DeepSeek-R1、Gemini）的决定性阶段。但奖励最大化目标（PPO/GRPO/OMD）结构性地坍缩多样性——CoT 推理需要多条有效路径来泛化和鲁棒，熵奖励/自适应裁剪等只是事后正则，没动核心目标。
- **之前的方案卡在哪里**：GFlowNet 的分布匹配范式对症，但 FlowRL 把可学习 Z 和大策略联合训练，规模一大就脆——梯度尖峰频发、辅助模块加同步开销、小 Z 网络追不上大预训练策略。MoE 路由的非确定性 + 隐式 off-policy 失配把问题放大到发散，但作者强调 MoE 只是"最难的压力测试"，不是病根。
- **它开启的路线**：证明"分布匹配 RL 能扩展到大模型"的可行性（首个在 dense + MoE 双架构都稳的 GFlowNet 系 RL，最大到 235B）；更重要的元教训——"扩展 GFlowNet RL 靠的不是加辅助机器，而是识别原目标里哪些部件在这个 regime 下是多余的"。
- **它关闭的问题**：可学习配分函数在 LLM 后训练里"是否必要"——答案是否定的，且有害。

## 核心机制（ASCII 图）

```text
奖励最大化 vs 分布匹配（问题动机）

  奖励最大化(PPO/GRPO):  概率质量 ──堆到──→ 单一最高奖励模式  → 模式坍缩,多样性1.2
        p(y|x)              ▂▂▂▂█▂▂▂▂
  分布匹配(GFlowNet):    按奖励比例覆盖所有高奖励模式        → 多样性3.9
        p(y|x)∝exp(βr)      ▂▂█▂▂█▂█▂

──────────────────────────────────────────────────────
TB 目标里的配分函数 Z，两条路线：

  FlowRL(旧):  Z(x) = 3层MLP(prompt末隐状态)    ← 随机初始化,从零学
       │
       │  learning-horizon mismatch:
       │    策略 π: 预训练十亿参数, 只需几百步微调  ┐
       │    Z:      随机初始化, 同样几百步从零学复杂量 ┘ 严重不对称
       │                    ↓
       │    log Z ≈ prompt的随机噪声 → 梯度被方差主导
       │    梯度范数均值 3.2e14, 421步里55步爆炸>1e6  ✗发散
       ↓
  GFlowRL(新): 删掉Z网络, 用批内蒙特卡洛估计
       │
       │  利用TB最优点性质: 最优时每条轨迹的隐式target相同
       │  Zt(x) = (1/G)·Σ[ r(x,yi) + log π_ref(yi) − log π_old(yi) ]
       │           ↑ 复用GRPO本就要采的G个rollout, 零额外成本
       │           ↑ 作为 stop-gradient 基线插入(只centering,不带梯度)
       │
       │  + 稳定器1: importance sampling 修正 π_old/π_θ 漂移
       │  + 稳定器2: 非对称 flow-gap clip (−low, +high), low<high
       │             给正确但欠采样的解更大上推空间
       ↓
    梯度范数均值 0.07(与GRPO 0.24同量级), 0次爆炸  ✓稳定
    基础设施≈GRPO: 零额外参数/学习率/跨模块同步
```

**固定点保证**（Prop B.1）：clip 未触发时残差恰为 TB 残差，任何零损失自洽固定点满足 π_θ(y|x) ∝ π_ref(y|x)·exp(β·r(x,y))——保留了 TB 的目标分布语义。clip 只在离群 rollout 上生效，最优点邻域内失活，不扰动稳态分布（Remark B.3）。

**关键实验结果（原文 Table 1-6，节选）**：

| 场景 | 指标 | GRPO | FlowRL | GFlowRL | 备注 |
|------|------|------|--------|---------|------|
| 7B 数学 | Avg@16 | 32.48 | 35.63 | **40.92** | GFlowRL 超 GRPO +8.44、超 FlowRL +5.29 |
| Z 消融 | Avg@16 | — | 35.61（学习Z） | 36.19（随机噪声Z） | 噪声替换不降反升 → Z 无用 |
| 梯度范数 | 均值 / 爆炸步数 | 0.24 / 0 | 3.2×10¹⁴ / 55 | 0.07 / 0 | FlowRL 421 步里 55 步爆炸>10⁶ |
| 14B 代码 | Codeforces Elo | — | 1904 | **2048** | 超 DeepCoder-14B +112、o1 +157，距 o3-mini 25 |
| 红队 | AdvBench/HarmBench ASR@1 | — | 发散 | **82.5% / 79.5%** | FlowRL 不收敛，GFlowRL 超 SEMA +2.4/+4.5 |
| MoE 30B-A3B | Codeforces Elo | — | 发散 | **1999** | 仅 3B 激活参数，超 o1 +108 |
| MoE 235B-A22B | 数学 Avg | 82.40 | 发散 | **83.35** | 仅训 30 步 vs GRPO 100 步 |
| 多样性 | o4-mini 评分(1-5) | 1.21 | 2.64 | **3.93** | GFlowRL 是 GRPO 的 3.2 倍 |
| clip 消融 | Avg@16 | — | — | 40.92→37.02（去clip） | 去 clip 掉 3.9 分、梯度均值涨 6.3 倍 |

## 设计决策解剖 ⚖️

| 决策点 | 他们的选择 | 被放弃的方案 | 为什么 | 这个选择何时失效 |
|--------|----------|------------|--------|---------------|
| 配分函数 Z | 批内 MC 均值估计（复用 rollout 组），stop-gradient 基线 | FlowRL 的 3 层 MLP / FOR 的全局共享标量 / Zhang 2023 的 log-partition 方差损失 | 消除 learning-horizon mismatch，尺度自动匹配 r/β，零额外参数/同步 | rollout 组 G 太小则 MC 估计方差大；奖励尺度剧烈非平稳时批内均值滞后；单 prompt 只采 1 条时估计退化 |
| 离群控制 | 非对称 flow-gap clipping（+high > −low） | 对称裁剪 / 不裁剪 | 正确解早期欠采样，需更激进上推；去 clip 掉 3.9 分、梯度均值涨 6.3 倍 | 奖励本身有偏时非对称界会放大偏置；界值需按任务调（数学适用，别的域未验证） |
| off-policy 修正 | importance sampling 权重 w=min(π_θ/π_old, 1+ε) | 不修正 | rollout 与 trainer 存在漂移（MoE 下尤甚） | 极端漂移时 IS 权重方差爆炸；上界截断引入偏置 |
| 长度归一化 | 按响应长度归一 log 概率 | 不归一 | 数千 token 的推理 rollout 下 log 概率随长度膨胀会主导损失 | 响应长度分布不均匀时引入近似误差（作者承认，Remark B.4） |
| 温度 β | 默认 β=8（倒 U，β∈[1,10] 内波动<0.5 分） | 单一固定或不调 | 控制目标分布锐度，实测低敏感 | β 太大逼近 argmax 退回模式坍缩，太小过度均匀失去奖励信号 |

## 成本与量级 💰

- **训练成本 vs 基线**：与 GRPO 几乎同构——删掉了辅助 Z 网络及其优化器状态、单独学习率、跨模块同步，所以比 FlowRL **更省**（少一个网络）而非更贵。这是罕见的"更强且更省"。
- **惊人的样本效率证据**：235B MoE 因 GPU 约束只训了 **30 步**（GRPO 训 100 步），仍以约 3 倍更少步数超过 GRPO +0.95 分，且停止时尚未饱和。
- **工程复杂度**：低。基于 verl/deepcoder/FlowRL/SEMA 现有框架，用默认配置；对已有 GRPO 管线是"改造"而非"重写"，估计数人周接入。
- **我的产品最小可行配置**：若已有 GRPO 后训练管线，加 GFlowRL 只需替换优势/基线计算模块 + 加 flow-gap clip；先在 7B 数学基准上验证多样性得分是否从 ~1.2 提到 ~3.9 再上规模。

## 证据审计 🔬

- **实验设计公平吗**：基线较全（PPO/GRPO/GRPO+/REINFORCE++/FlowRL），用各自默认配置、同框架，数学/代码/红队三域覆盖。但"距 o3-mini 25 Elo"是拿开源 14B 蒸馏模型比闭源系统，Codeforces Elo 的评估协议（Appendix 11）未在正文交代，跨系统 Elo 可比性存疑。
- **最强证据**：随机噪声替换 Z 实验（36.19% vs 35.61%，甚至略升）+ 梯度范数对照表（FlowRL 均值 3.2×10¹⁴ / 55 次爆炸 vs GFlowRL 0.07 / 0 次）。这是"Z 既无用又有害"的双向铁证，逻辑闭环、可复现（附了 FlowRL 的 wandb 原始日志链接）。
- **最可疑的数字**：多样性得分 3.93 vs FlowRL 2.64 vs GRPO 1.21。它是 **GPT-o4-mini 当裁判**在 1–5 尺度上打的（5 次均值，方差 ~0.53）——用一个 LLM 主观打分来量化"多样性"这个本就模糊的概念，且裁判可能偏好"看起来不同"而非"真正不同"的解；2.7 分差声称是 5 倍裁判方差，但裁判系统偏差（非方差）无法被重复实验消除。
- **另一个软肋**：MoE 的 GRPO 基线在 Table 5 里只给了单个 Avg 数字（75.78/82.40），没有分基准明细，而 GFlowRL 全明细——对照不对等，读者无法核验 GRPO 是否在某些基准上其实更强。
- **如果我是审稿人，会要求补充**：(1) 235B 训到饱和（非 30 步截断）的完整曲线，否则"超 GRPO"可能是步数未对齐的假象；(2) 用非 LLM 的客观多样性指标（如解法聚类数、pass@k 曲线）复核多样性 claim；(3) Codeforces Elo 的完整评估协议与 o3-mini 的可比性论证；(4) rollout 组大小 G 对 MC 估计方差的敏感性分析。
- **最小复现实验**：在 Qwen2.5-7B 上用开源代码跑 100 步数学基准，只对比 GRPO vs GFlowRL 的 Avg@16 和梯度范数曲线，验证"稳定性恢复"这一核心 claim；预算数百 GPU 小时。

## 可复用点（PM 决策）

- **何时采用**：① 需要**解的多样性**的场景（数学有多解、代码有多实现、红队要多样攻击、创意生成）——分布匹配天生优于奖励最大化；② 已在跑 GRPO 但遇到多样性坍缩或想上 MoE；③ 红队/安全对抗训练这类**奖励噪声大而稀疏**的场景——正是 FlowRL 发散而 GFlowRL 独活之处。
- **何时规避**：① 任务只有唯一正确解、不需要多样性——GRPO 更简单够用；② 奖励信号本身有系统偏置——非对称 clip 会放大它；③ rollout 组太小（G 小）——批内 MC 估计不可靠。
- **供应商拷问清单**：
  1. 你们的 RL 后训练是奖励最大化还是分布匹配？多样性坍缩测过吗（pass@k 而非 pass@1）？
  2. 若用 GFlowNet 系，配分函数是学习型还是批内估计？梯度稳定性有对照数据吗？
  3. 多样性提升是用 LLM 裁判打分还是客观指标（解法聚类/pass@k）证明的？
  4. 在 MoE 上训过吗？FlowRL 类方法在你们规模上收敛吗？
  5. 相比 GRPO 基线，额外的训练开销和基础设施改动有多大？

## 关联网络 🕸️

- [[Wiki/概念/02_训练方法/GRPO优化算法]] → **直接基座**：GFlowRL 的基础设施与 GRPO 几乎一致，批内 MC 估计"结构上类比 GRPO 的组均值基线，只是作用在 TB 残差而非优势上"。理解 GRPO 是读懂本文的前提。
- [[Wiki/概念/02_训练方法/PPO近端策略优化]] → **对照**：PPO/GRPO 都是奖励最大化范式，本文正是要超越它们的模式坍缩；非对称 flow-gap clip 借鉴了 PPO 的信赖域裁剪精神。
- [[Wiki/论文笔记/05_推理思维链/84_DeepSeek-R1强化学习激励推理能力]] → **印证/对照**：R1 用 GRPO 证明 RL 可激励推理，是奖励最大化路线的旗舰；本文用其蒸馏模型（DeepSeek-R1-Distill-Qwen-7B/14B）当代码实验骨干，并主张分布匹配在多样性上系统性胜出——两者是"同一目标、对立范式"。
- [[Wiki/论文笔记/15_安全隐私/185_信息访问对LLM监控检测破坏能力的影响]] → **交叉**：185 是防御侧（检测破坏），本文红队实验是攻击侧（训练更强的多轮攻击者，AdvBench 82.5%）——攻防两篇对读，可见 RL 训练同时在武装矛与盾。
- **冲突**：与 FlowRL（Zhu et al. 2026）——本文直接论证 FlowRL 的核心组件（可学习 Z）无用且有害，并在多个 setting 复现其发散。这是同一 GFlowNet 系内部的"后浪拍前浪"，也提醒：一个刚发表的方法的核心设计可能几个月内就被证伪。
- **印证**：与 Xu et al. 2025（奖励最大化坍缩多样性）、Zhang et al. 2023（log-partition 方差损失绕开学 Z）一脉相承；GFlowRL 是后者思想在 LLM on-policy rollout 场景的落地。

## 动手练习 💻（约 35 分钟）

练习目标：在一个玩具多模态目标上亲手对比"奖励最大化"与"分布匹配"，看到前者模式坍缩、后者覆盖多峰，并理解"批内均值估 Z"为什么等价于给 TB 残差做基线校准。

```python
import numpy as np

rng = np.random.default_rng(0)

# ── 玩具世界: 3 个离散"解法"a/b/c, 奖励不同 ──
# 真实奖励: a=1.0(最优), b=0.9(次优,与a几乎并列), c=0.2(差)
rewards = np.array([1.0, 0.9, 0.2])
beta = 8.0                                            # 逆温度, 论文默认值

# ── 目标分布(分布匹配想达到的): p ∝ exp(beta*r) ──
target = np.exp(beta * rewards)
target /= target.sum()                               # 归一化
print("分布匹配目标 p(a,b,c) =", np.round(target, 3))  # a,b 都该有可观概率(保留多解)

# ── 方案1: 奖励最大化(GRPO式) —— 用 softmax 策略, 梯度上推期望奖励 ──
logits_rm = np.zeros(3)                               # 策略参数(3个解的logit)
lr = 0.5
for step in range(300):
    p = np.exp(logits_rm); p /= p.sum()               # 当前策略分布
    # 策略梯度: 提升高于平均奖励的解 (GRPO组均值基线)
    baseline = (p * rewards).sum()                    # 组内期望奖励当基线
    grad = p * (rewards - baseline)                   # ∇ E[r] 的估计
    logits_rm += lr * grad
p_rm = np.exp(logits_rm); p_rm /= p_rm.sum()
print("奖励最大化收敛到   =", np.round(p_rm, 3), "← 质量堆到单一最优 a, 坍缩")

# ── 方案2: 分布匹配(GFlowRL式) —— 用批内均值估 log Z, 逼近 TB 固定点 ──
# TB 残差: delta_i = logZ_hat + log pi(y_i) - beta*r_i, 目标是让所有 delta_i 相等(=0)
logits_dm = np.zeros(3)
ref = np.log(np.ones(3) / 3)                          # 参考策略取均匀(log pi_ref)
for step in range(300):
    logp = logits_dm - np.logsumexp(logits_dm)        # 当前策略 log pi_theta
    # 关键: 批内蒙特卡洛估 logZ = mean_i[ beta*r_i + log pi_ref_i - log pi_theta_i ]
    #      (对应论文 Eq.4, 复用"整组样本"求均值当 stop-gradient 基线)
    logZ_hat = np.mean(beta * rewards + ref - logp)
    # 每个解的 TB 残差(希望它=0): 用它当梯度信号把策略拉向目标固定点
    delta = logZ_hat + logp - ref - beta * rewards    # = log pi_theta - (log pi_ref + beta r - logZ)
    logits_dm -= lr * delta                           # 下降残差
p_dm = np.exp(logits_dm); p_dm /= p_dm.sum()
print("分布匹配收敛到     =", np.round(p_dm, 3), "← 逼近目标 p, a/b 双峰都保留")

# ── 结论对照 ──
print("\n目标      :", np.round(target, 3))
print("奖励最大化:", np.round(p_rm, 3), "(多样性差: 几乎全押 a)")
print("分布匹配  :", np.round(p_dm, 3), "(多样性好: 贴合目标)")
# 教训: 同样的奖励, 奖励最大化坍缩到单峰, 分布匹配用一个"批内均值Z"就复原了多峰目标 —
# 而这个Z不需要额外学一个网络, 这正是 GFlowRL 删掉 FlowRL 那个 MLP 的核心洞见
```

预期收获：亲眼看到奖励最大化把并列的 a(1.0)/b(0.9) 坍缩成只押 a，而分布匹配用一行"批内均值估 Z"就恢复了双峰目标——直观理解为什么本文能"删掉一个网络还更强"。

## 自测三层 🎓
> 判分点随笔记入库；自测先作答再展开。L1 不过不进 L2/L3——答错即记卡点。

> [!question]- L1 复述：说出奖励最大化与分布匹配的目标分布差异；learning-horizon mismatch 指哪两个组件的什么不匹配；两个实证铁证的数字（噪声替换 36.19 vs 35.61；梯度范数 3.2e14 vs 0.07）；两个稳定器是什么。
> **参考答案**：奖励最大化（PPO/GRPO）把概率质量堆到单一最高奖励模式，结构性坍缩多样性；分布匹配的目标是 p(y|x) ∝ exp(β·r(x,y))——按奖励比例覆盖所有高奖励模式而非 argmax。learning-horizon mismatch：策略 π 从预训练 checkpoint 出发、几百步微调就够；FlowRL 的配分函数 Z（prompt 末隐状态接 3 层 MLP）随机初始化、要在同样短的窗口内从零学会一个 prompt-conditional 复杂量——学习视野严重不对称，训练大部分时间里 log Z 只是 prompt 的随机函数。两个铁证：①把 Z 换成 N(0.5,1) 随机噪声，性能不降反升（36.19% vs 35.61%）——Z 没编码有用信息；②FlowRL 梯度范数均值 3.2×10¹⁴、421 步里 55 步爆炸超 10⁶，GFlowRL 均值 0.07、零爆炸。两个稳定器：importance sampling 修正 rollout 策略 π_old 与当前 π_θ 的漂移；非对称 flow-gap clipping（正向界>负向界）。
> **判分点**：必须答出 分布匹配 p∝exp(βr) vs 奖励最大化堆单峰、mismatch=预训练策略几百步 vs 随机初始化 Z 同窗口从零学、两个铁证（噪声替换不降反升/梯度范数差 14 个数量级）、两个稳定器（IS 修正+非对称 clip）（缺一个即卡点）

> [!question]- L2 解释：为什么用批内 rollout 组的均值就能替代一个学习型配分网络？（TB 最优点性质：每条轨迹隐式 target 相同，均值是其无偏估计基线）为什么 flow-gap clip 要非对称？为什么 Z 的问题在 MoE 上被放大到发散而不只是变慢？
> **参考答案**：TB 目标的最优点性质：最优时对每条轨迹都有 log Z(x) = r(x,y)+log π_ref−log π_θ，即每条轨迹的隐式 target 相同——所以用当前 rollout 组 G 个样本对该量取批内蒙特卡洛均值就是 Z 的估计，作为 stop-gradient 基线插入策略梯度即可；它复用 GRPO 本就要采的 G 个 rollout（零额外参数/学习率/同步），且尺度自动与 r/β、log π 同量级，梯度爆炸随之消失。clip 非对称：正确但早期欠采样的解需要更激进的上推空间，故正向界大于负向界；消融证明去 clip 掉 3.9 分、梯度均值涨 6.3 倍。MoE 放大到发散：MoE 路由的非确定性 + rollout 与 trainer 的隐式 off-policy 失配把 Z 的噪声进一步放大——但作者强调 MoE 只是最难的压力测试，病根仍是可学习 Z 本身（dense 模型上 FlowRL 同样梯度爆炸）。
> **判分点**：必须答出 TB 最优点每条轨迹隐式 target 相同→批内均值即 MC 估计（stop-gradient、零额外参数）、非对称=给欠采样的正确解更大上推空间、MoE=路由非确定性+off-policy 失配放大噪声但病根是 Z（缺一个即卡点）

> [!question]- L3 应用：你要训一个红队攻击 Agent，要求它能生成**多样**的越狱策略而非反复用同一招，且奖励信号稀疏噪声大。基于本文，你选 GRPO 还是 GFlowRL？如何用客观指标而非 LLM 裁判验证多样性？
> **参考答案**：选 GFlowRL，两条理由：①多样性是硬需求——GRPO 的奖励最大化会坍缩到单一攻击模式（多样性评分 1.21 vs GFlowRL 3.93），红队反复用同一招毫无价值；②奖励稀疏噪声大正是本文红队实验的场景——Table 4 直接证据：AdvBench/HarmBench 上 FlowRL 发散不收敛，GFlowRL 稳定收敛且 ASR@1 达 82.5%/79.5%（超 SEMA +2.4/+4.5）。客观多样性验证（不依赖 LLM 裁判的主观分）：①ASR@k / pass@k 曲线——k 增大时增益越陡说明攻击分布越多样，坍缩模型的 k 增益曲线平坦；②攻击策略聚类——对生成的攻击做嵌入聚类，报告有效簇数与簇分布熵（这正是本笔记证据审计里要求补充的客观指标，因为 o4-mini 裁判打分有系统偏差且无法用重复实验消除）。
> **判分点**：必须答出 选 GFlowRL 的双理由（GRPO 坍缩单一攻击模式 1.21 vs 3.93 / 噪声奖励下 FlowRL 发散而 GFlowRL 独活的 Table 4 证据）、至少两个客观指标（pass@k 或 ASR@k 曲线斜率 / 攻击聚类数与熵）（缺一个即卡点）

📅 **知识时间锚**：2026-07-15 (arXiv v1)。基线模型为 Qwen2.5-7B/32B、DeepSeek-R1-Distill-Qwen-7B/14B、Qwen3-30B-A3B/235B-A22B 世代；对标 o1(1891)/o3-mini(2073) Codeforces Elo。核心洞见"短视野后训练下随机初始化的辅助网络是负资产"预计比具体 Elo 数字更耐久；FlowRL 作为被反驳对象发表于 2026 年初，本文距其仅数月——RL 后训练算法迭代以月计，2026 年底应复核是否已有新方法同时改进 Z 估计与多样性度量。