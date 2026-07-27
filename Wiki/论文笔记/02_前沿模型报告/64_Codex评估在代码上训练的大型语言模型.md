---
tags: [论文笔记, Codex, 代码生成, HumanEval, pass@k, GitHub Copilot, OpenAI]
paper_id: "64"
笔记层级: 骨干
复核日期: 2026-07-04
---

# Codex：评估在代码上训练的大型语言模型

📄 **原文 PDF**：[[RAW/64 - Evaluating Large Language Models Trained on Code.pdf]]

## PM 速判
> GitHub Copilot 的底层模型，AI 编程辅助的奠基论文。GPT-3 在 159GB GitHub Python 代码上微调得到 Codex，提出 HumanEval 基准和 pass@k 无偏估计，12B 版本 pass@1 = 28.8%，证明了"代码数据专项微调"的巨大价值。

## 双层费曼 🗣️
> **给CEO**：GPT-3 是个知识渊博但不会写代码的通才。给它喂159GB GitHub 代码"补课"后，它变成了能写 Python 函数的专业程序员。我们还得设计一套考试题（HumanEval，164道题）来客观衡量它的编程水平——不是看它"答得像不像"，而是看它"写的代码能不能跑通"。
> **给工程师**：基于 GPT-3 decoder-only Transformer，在 54M 公开 GitHub 仓库的 159GB Python 上做因果语言模型微调。核心创新在评测：用函数签名+文档字符串作为 prompt，通过单元测试验证功能正确性；pass@k 指标用 n 次采样中 c 次通过来无偏估计"从 k 个样本中至少有一个正确"的概率，公式 pass@k = 1 − C(n−c, k)/C(n, k)。采样策略用 temperature=0.8 + top_p=0.95 核采样，避免贪心解码的单次波动。

## 问题域定位 🎯
- **回应什么根本约束？** 自然语言文本的 BLEU/ROUGE 等表面匹配指标无法衡量代码质量——两段语义完全不同的代码可能实现同一功能，而语义相同的代码可能表面完全不同。代码评测需要"功能正确性"而非"文本相似度"。
- **之前卡在哪？** GPT-3 零样本代码生成近乎 0%（HumanEval pass@1 = 0%），但没人确切知道是"能力不足"还是"没学过写代码"。此前代码生成任务（如 CodeSearchNet）停留在检索和补全片段层面，缺乏"给定意图，写完整函数"的系统性评测。
- **开启/关闭了哪条路线？** 开启：代码专项数据微调路线（Codex → CodeLLaMA → StarCoder → DeepSeek-Coder），功能正确性评测路线（HumanEval → MBPP → DS-1000 → SWE-bench），采样多样性 + pass@k 评测范式。关闭：仅用 BLEU 评测代码生成质量的做法。**未关闭但受质疑**：仅关注单文件函数级任务是否足以衡量真实编程能力。

## 核心机制
```
                    GPT-3 (175B, 预训练)
                            │
                            │ 在 159GB GitHub Python 上因果LM微调
                            ▼
                      Codex (12B max)
                            │
                            │ 输入: 函数签名 + docstring
                            │ 采样: T=0.8, top_p=0.95, n=200
                            ▼
                    ┌─── n 个候选代码 ───┐
                    │  candidates[0]     │──单元测试──► pass/fail
                    │  candidates[1]     │──单元测试──► pass/fail
                    │  ...               │    ...
                    │  candidates[n-1]   │──单元测试──► pass/fail
                    └───────────────────┘
                            │
                            │ c 个通过 / n 个总计
                            ▼
                   pass@k = 1 − C(n−c, k) / C(n, k)
                       无偏估计: k次采样至少一次通过
```

关键创新：(1) 评测用功能正确性而非文本匹配；(2) pass@k 的数值稳定估计器——对每个题 n≈200 次采样即可稳健估计 k≤100 的 pass@k，避免对每个题真的做 k 次采样再重复多次带来的巨额计算量。公式本质：1 减去"从 n 个样本中取 k 个且全都不通过"的概率。

## 设计决策解剖 ⚖️
| 决策点 | 选择 | 被放弃方案 | 为什么 | 何时失效 |
|--------|------|----------|--------|---------|
| 评测指标 | pass@k（功能正确性） | BLEU/ROUGE（文本匹配） | 代码的正确性 != 表面文本相似度；两段功能相同的代码可能迥异。单元测试是客观、可自动化的 gold standard | 当测试覆盖率不足时（如仅有 happy-path 测试）pass@k 同样虚高；需要边界条件测试配合 |
| pass@k 估计器 | 单次 n 采样 + 无偏估计公式 | 重复多次 k 采样取平均 | 直接 k 采样方差大（k 小时尤其），n 采样后再用组合数学估计可大幅降低方差，且一次采样即可计算所有 k 的 pass@k | 当 n 远小于 k 时（如 n=10, k=100）估计不可靠，需要 n >> k。论文中 n=200, k≤100 满足 |
| 采样策略 | T=0.8 + top_p=0.95 | 贪心解码（T=0） | 贪心解码每次选最高概率 token，多样性极低，导致"要么全对要么全错"的二元分布，无法有效利用 pass@k 的多次采样优势 | 如果需要确定性输出（如生产环境 API），高温采样引入的不确定性不可接受；需额外 reranking 策略 |
| 训练数据范围 | 仅 Python（159GB） | 多语言混合 | Python 生态最成熟、测试基础设施最好（pytest/unittest）、HumanEval 构建成本最低；先证明概念再扩展 | 推广到多语言时，需要为每种语言重新构建等价于 HumanEval 的评测集（如 MultiPL-E 做了但覆盖不全） |
| 模型规模 | 12B 参数（最大） | 175B（GPT-3 原规模） | 代码数据有限（159GB vs GPT-3 的 TB 级文本），大模型在有限代码数据上会过拟合；12B 是在 HumanEval 上性价比最优的点 | 当代码数据量远超 159GB 时（如 The Stack 数据集数 TB），更大模型可能更有优势 |

## 成本与量级 💰
- **训练**：基于 GPT-3 已有的 175B checkpoint，代码微调阶段的计算约 GPT-3 预训练的 1/100（粗略估计，论文未给精确 GPU-hours）。训练数据 159GB Python，从 54M 仓库爬取过滤。
- **推理**：12B 参数，FP16 约 24GB 显存，单次推理延迟 < 1s（取决于生成长度）。HumanEval 评测每道题 200 次采样，164 道题合计约 32,800 次推理。
- **最小可行配置**：HumanEval 评测无需训练 Codex——用任何代码模型 + 164 道题的公开 HumanEval 仓库即可复现评测管线。训练从零复现最小配置约需 100GB Python 代码 + 8×A100 (80G)。
- **HumanEval 成本**：164 道题由 OpenAI 内部人工编写，包括函数签名、文档字符串（英文）和单元测试，估计 2-4 人周。

## 证据审计 🔬
- **实验公平性**：对比基线合理——GPT-3 零样本、GPT-J (6B)、TabNine。但 Codex 本身是基于 GPT-3 微调的，与 GPT-3 零样本比有"本钱不同"的优势。GPT-J 仅 6B 且训练数据不同，不完全可比。
- **最强证据**：pass@100 从 GPT-3 零样本的 0% 提升到 Codex-12B 的 72.3%——增幅绝对值极高，置信度高。Codex-S（额外 SFT 微调 12B）pass@1 = 37.7%，证明监督微调叠加代码预训练有进一步增益。
- **最可疑数字及原因**：pass@100 = 77.5%（Codex-S）——100 次采样中有一次通过就算通过，这意味着模型尝试 100 次能写出正确答案，但用户在 Copilot 中通常只看 1-3 个建议。这个数字容易误导人以为"模型很可靠"，实际用户感知的 pass@1 仅 37.7%。此外，HumanEval 题目偏向"算法题"风格（如求两数和、字符串处理），与现实工程中的 API 调用、框架使用、多文件组织差距较大。
- **审稿补充要求**：缺少不同编程语言的对比（仅 Python）；未分析 pass@k 随 k 增长的收益递减拐点；缺少"代码安全性/漏洞"维度的评测。
- **最小复现设计**：用 GPT-2 级别模型（~100M 参数）+ 10GB Python 代码微调 + HumanEval 子集（50 题）即可验证"代码微调提升功能正确性"的核心假设，估计 1×A100 数小时。

## 可复用点
- **何时采用**：(1) 需要评测代码生成模型时，优先用 HumanEval/MBPP 的 pass@k 框架而非 BLEU；(2) 设计产品化的代码补全系统时，参考 docstring → function 的 prompt 格式；(3) 需要向非技术方解释"AI 编程能力"时，pass@1 = 28.8% 是 2021 年的锚点数字。
- **何时规避**：(1) 评测对象是多文件项目、API 调用密集型代码时，HumanEval 代表性不足；(2) 需要确定性输出（如 CI/CD 中的自动修复）时，pass@k 的高温采样策略不适用；(3) 评测中文代码生成时，HumanEval 英文 prompt 不能直接用，需翻译/重写（已有中文 HumanEval 变体）。
- **供应商拷问清单**：
  1. "你们的 pass@1 是多少？用什么评测集测的？是 HumanEval 原版还是自建？"
  2. "如果你们的模型在 HumanEval 上 pass@1=80%，那在 SWE-bench（真实 GitHub Issue 修复）上是多少？"
  3. "你们训练用了多少代码数据？来自哪些源？经过什么过滤？"
  4. "评测有没有排除训练数据泄露？你怎么知道模型不是背答案？"
  5. "你们代码模型的延迟和吞吐是多少？200 次采样的成本在实际产品中可接受吗？"

## 关联网络 🕸️
- **相关论文**：
  - [[Wiki/论文笔记/01_LLM基础架构/65_FLAN微调语言模型是零样本学习者]] — FLAN 同期（2021）探索指令微调提升零样本泛化，Codex 探索代码微调提升代码能力，两者都是"专项数据微调解锁特定能力"的早期证据
  - [[Wiki/论文笔记/01_LLM基础架构/75_LLaMA高效开放的基础语言模型]] — LLaMA 开源后派生 CodeLLaMA，将 Codex 的代码微调思想应用到开源模型，是该路线的规模化延续
  - [[Wiki/论文笔记/05_推理思维链/73_ReAct推理与行动的协同]] — ReAct 框架中的"代码行动空间"依赖 Codex 开启的代码生成能力
  - [[Wiki/论文笔记/05_推理思维链/68_ChainOfThought思维链提示激发大模型推理]] — CoT 与代码生成结合产生 Program-of-Thought（PoT），将自然语言推理转换为代码执行，是两线交叉的产物
- **相关概念**：
  - [[Wiki/概念/03_推理与评测/思维链与推理模型]] — PoT 方向
  - [[Wiki/概念/01_架构技术/LoRA与参数高效微调]] — 后来的 LoRA 等 PEFT 方法降低了代码微调的成本
- **冲突/印证**：[[Wiki/论文笔记/01_LLM基础架构/65_FLAN微调语言模型是零样本学习者]] 印证了"任务类型多样性比数据量更重要"的规律——FLAN 在 62 种任务上微调获得泛化能力，Codex 在单一语言（但 54M 仓库→多样代码模式）上微调获得函数级编程能力，两者都显示专项数据微调的巨大收益。但 Codex 暗示"数据量更重要"（159GB 单一 Python），FLAN 暗示"多样性更重要"（62 任务但每个数据不多），这对矛盾在后续 StarCoder（多语言+巨量数据）中得到部分调和——两者都重要，取决于目标任务的分布宽度。

## 动手练习 💻
**练习目标**：实现 pass@k 的无偏估计函数，理解为什么用组合数学估计比直接采样估计更稳定。
```python
import math
import numpy as np
from typing import List

# ============ pass@k 无偏估计 ============

def pass_at_k(n: int, c: int, k: int) -> float:
    """
    从 n 个样本中 c 个通过，无偏估计 pass@k。
    
    公式: pass@k = 1 - C(n-c, k) / C(n, k)
    含义: 1 减去 "从 n 个样本中取 k 个且全部不通过" 的概率
    
    参数:
        n: 总采样次数 (对每道题生成候选代码的数量)
        c: 通过测试的次数 (c ≤ n)
        k: 考察的采样次数 (k ≤ n)
    
    返回: 估计的 pass@k 概率，范围 [0, 1]
    """
    # 边界情况: 如果通过的样本不够填满 k 个位置
    # 则从总样本中取 k 个必然包含至少一个不通过的——即不可能"恰好取到 k 个全通过"
    if n - c < k:
        # C(n-c, k) 当 n-c < k 时为 0 ——无法从 n-c 个不通过的里面选出 k 个
        # 所以 pass@k = 1 - 0 = 1.0，即必然至少有一个通过
        return 1.0
    # 通用情况: 计算 C(n-c, k) / C(n, k)，即 k 个全不通过的概率
    # math.comb(m, r) 如果 r > m 返回 0，但我们已在上面显式处理
    return 1.0 - math.comb(n - c, k) / math.comb(n, k)


def estimate_pass_at_k_from_samples(
    per_problem_results: List[List[bool]], k_values: List[int]
) -> dict:
    """
    从多道题的采样结果中计算整体的 pass@k。
    
    参数:
        per_problem_results: 列表，每个元素是一道题的 n 次采样结果 (bool列表)
                              True 表示该次采样生成的代码通过了单元测试
        k_values: 需要计算哪些 k 值，如 [1, 10, 100]
    
    返回: 字典 {k: 整体pass@k}
    """
    num_problems = len(per_problem_results)
    result = {}
    for k in k_values:
        pass_at_k_per_problem = []
        for results in per_problem_results:
            n = len(results)       # 该题的总采样次数
            c = sum(results)       # 该题的通过次数
            if n == 0:
                continue
            # 单道题的 pass@k 估计
            p = pass_at_k(n, c, k)
            pass_at_k_per_problem.append(p)
        # 整体 pass@k = 各题 pass@k 的算术平均 (论文中的标准做法)
        result[k] = float(np.mean(pass_at_k_per_problem))
    return result


# ============ 仿真验证: 对比直接采样 vs 无偏估计 ============

def simulate_direct_pass_at_k(true_pass_rate: float, k: int, n_trials: int = 1000):
    """
    用直接重复采样的方式估计 pass@k (用于对比验证)。
    
    这个方法对每个 trial: 独立采样 k 次，检查是否至少有一次成功。
    是 pass@k 的"定义式"估计，但需要大量 trials 才能稳定。
    
    参数:
        true_pass_rate: 模型真正的单次通过概率 (在仿真中我们设定它)
        k: 考察的采样次数
        n_trials: 重复试验次数(越大越准但越慢)
    """
    successes = 0
    for _ in range(n_trials):
        # 独立采样 k 次
        samples = np.random.random(k) < true_pass_rate
        if samples.any():  # 至少有一个通过
            successes += 1
    return successes / n_trials


# ============ 演示 ============

if __name__ == "__main__":
    print("=" * 60)
    print("pass@k 无偏估计 vs 直接采样对比")
    print("=" * 60)

    # 仿真参数: 模型真实单次通过率 30%, Codex-12B 级别的合理设定
    true_rate = 0.30
    n = 200        # 每道题采样 200 次 (论文标准)
    k = 10         # 考察 pass@10

    # 生成仿真的 200 次采样结果 (每道题)
    np.random.seed(42)
    sample_results = np.random.random(n) < true_rate
    c = int(sample_results.sum())

    # 无偏估计
    estimated = pass_at_k(n, c, k)

    # 直接采样估计 (作为 ground truth 参考)
    direct = simulate_direct_pass_at_k(true_rate, k, n_trials=10000)

    print(f"\n设定: 真实单次通过率 = {true_rate:.0%}, n={n}次采样, c={c}次通过")
    print(f"考察: pass@{k}")
    print(f"  无偏估计 (组合数学): {estimated:.4f}")
    print(f"  直接采样 (10000 trials): {direct:.4f}")
    print(f"  理论值: 1-(1-{true_rate})^{k} = {1 - (1-true_rate)**k:.4f}")
    print(f"\n关键洞察: 无偏估计仅需 1 次 n=200 采样即可得到稳定估计;")
    print(f"直接采样需要 10000 个独立的 k=10 批次才能达到同等精度。")
    print(f"这就是 pass@k 估计器\"无偏+低方差\"的价值所在。")

    # ======== 多道题的完整评测场景 ========
    print("\n" + "=" * 60)
    print("多道题 HumanEval 风格完整评测")
    print("=" * 60)

    # 模拟 5 道题，每道题 n=200 次采样
    per_problem = []
    problem_rates = [0.5, 0.3, 0.1, 0.7, 0.05]  # 5道题各有不同难度
    for rate in problem_rates:
        results = np.random.random(200) < rate
        per_problem.append(results.tolist())

    # 计算 pass@1, pass@10, pass@100
    eval_result = estimate_pass_at_k_from_samples(per_problem, [1, 10, 100])
    for k, val in eval_result.items():
        print(f"  整体 pass@{k:>3}: {val:.4f}")

    print(f"\n解读: pass@100 >> pass@1 说明对于难题(k=1总选错),")
    print(f"多次采样后选最优能极大弥补单次能力的不足。")
    print(f"这是 test-time compute 思想的早期体现。")
```

## 自测三层 🎓
**L1 复述**：(1) 写出 pass@k 的无偏估计公式，并解释 C(n−c, k) / C(n, k) 这个分数代表什么？(2) HumanEval 有多少道题？评测的是什么语言？Codex-12B 的 pass@1 和 pass@100 分别是多少？

**L2 解释**：(1) 为什么用文档字符串 + 函数签名作为 prompt，而不是用自然语言描述问题？这种设计对后续 Copilot 产品有什么影响？(2) 如果 n=200, c=50, k=100，pass@k 估计值是多少？在这种 n 和 k 的取值下，这个估计可靠吗？为什么？

**L3 应用**：(1) 你要为一个内部代码仓库设计评测——仓库主要是 Java Spring Boot 的 CRUD 代码。HumanEval 的评测框架能直接套用吗？如果不能，需要修改哪些部分？(2) 你的代码模型 pass@1=35%，但用户反馈"10次建议才有1次能用"。如果你只有 100ms 的推理时间预算（不能等 200 次采样），有哪些工程手段可以在不增加采样次数的情况下提高"用户首次选中率"？

📅 知识时间锚：2021-07（论文发布），GitHub Copilot 预览版同期上线，AI 编程辅助时代的起点。
