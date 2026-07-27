---
tags: [论文笔记, 思维链, CoT, 推理, 基础论文, Google]
paper_id: "68"
filename: "68 - Chain-of-Thought Prompting Elicits Reasoning in Large Language Models.pdf"
authors: "Jason Wei et al. (Google)"
year: 2022
成熟度: "✅"
笔记层级: 骨干
复核日期: 2026-07-04
---

# Chain-of-Thought Prompting Elicits Reasoning in Large Language Models

📄 **原文 PDF**：[[RAW/68 - Chain-of-Thought Prompting Elicits Reasoning in Large Language Models.pdf]]

## PM 速判（30秒）
> 一句话：只要在 few-shot 示例里加上"中间推理步骤"，PaLM 540B 在 GSM8K 数学题上的准确率就从 17.9% 跳到 56.9%（外接计算器 58.6%），超过当时的有监督 SOTA——这篇 2022 年论文奠定了"先思考再回答"范式，是 o1/R1 等一切推理模型的源头。你产品里每个"思考模式"开关、每条 step-by-step prompt 的成本与延迟逻辑，都从这里来。

## 双层费曼 🗣️
> **给 CEO 的一句话**：不用重新训练模型，只要在提问时先示范"怎么一步步想"，同一个模型做复杂题的正确率就能翻三倍——换个问法解锁隐藏能力，几乎零成本。
> **给工程师的一段话**：CoT prompting 把 few-shot exemplar 从 (question, answer) 改写成 (question, rationale, answer) 三段式，推理时模型自回归地先生成自然语言中间步骤再输出答案。效果是涌现的：约 100B 参数以下加 CoT 无效甚至更差；PaLM 540B 上 GSM8K 从 17.9%→56.9%。消融证明关键在"答案前的自然语言推理"：只给公式（equation only）、只加占位符 token（dots）、把推理放答案后面，三者全部无效。费用全在输出 token——rationale 长度是答案的 3-10 倍。

## 问题域定位 🎯
- 这篇论文在回应什么**根本约束**？Transformer 每个 token 的前向计算量固定，答案 token 处能分配的"思考算力"与问题难度无关；多步推理需要把中间状态显式写到上下文里（externalize），让后续 token 以它们为条件。
- 之前的方案卡在哪里？标准 few-shot 在 GSM8K 这类多步数学题上 scaling 曲线平坦（越大越好但极慢）；有监督路线（Cobbe 2021 微调 GPT-3 + verifier）需要几千条人工标注的解题过程，且一个模型只能做一个任务。
- 它开启/关闭了哪条技术路线？开启了"prompting 即可激发推理"→ Zero-shot CoT → Self-Consistency → PRM 过程监督 → o1/R1 的 test-time scaling 整条主线；关闭了"每个推理任务必须微调专用模型"的假设。

## 核心机制

```
标准 few-shot                          CoT few-shot
┌───────────────────────┐             ┌────────────────────────────────┐
│ Q1: Roger 有 5 个网球， │             │ Q1: Roger 有 5 个网球，又买了    │
│     又买了 2 罐每罐 3 个│             │     2 罐每罐 3 个…              │
│ A: 11                 │             │ A: Roger 一开始有 5 个。2 罐    │
│                       │             │    每罐 3 个共 6 个。5+6=11。   │
│ Q_test: 食堂有 23 个   │             │    答案是 11。   ← rationale    │
│ 苹果，用掉 20 个又买 6 │             │                                │
└──────────┬────────────┘             │ Q_test: 食堂有 23 个苹果…       │
           ▼                          └───────────────┬────────────────┘
    A: 27 ✗（直接猜答案）                              ▼
                                      A: 原有 23 个，用掉 20 个剩 3 个，
                                         又买 6 个，3+6=9。答案是 9。 ✓

规模涌现（GSM8K 准确率示意）
 60%│                                        ● CoT 提示
    │                                      ╱
 40%│                                    ╱
    │                                  ╱
 20%│              ●─────●──────────●╱     ○ 标准提示
    │    ●────────╱          ○───────○──────○
  0%└────┴────────┴──────────┴──────────────▶ 模型规模
         8B       62B       ~100B      540B
    ←── 100B 以下：CoT 无效甚至更差 ──→│←── 涌现区 ──→
```

**关键数据**（PaLM 540B，标准提示 → CoT 提示）：

| 基准 | 标准提示 | CoT | 增益 | 备注 |
|------|---------|-----|------|------|
| GSM8K | 17.9% | **56.9%** | +39.0 | 超当时 SOTA 55%（微调 GPT-3+verifier）；外接计算器 58.6% |
| SVAMP | 69.4% | **79.0%** | +9.6 | 新 SOTA |
| MAWPS | 79.1% | **93.3%** | +14.2 | 新 SOTA |
| AQuA | 25.2% | **35.8%** | +10.6 | 距 SOTA 2% 内 |
| StrategyQA | — | **75.6%** | — | 超此前 SOTA 69.4% |

## 设计决策解剖 ⚖️
| 决策点 | 他们的选择 | 被放弃的方案 | 为什么 | 这个选择何时失效 |
|--------|----------|------------|--------|---------------|
| 中间步骤的形式 | 自然语言推理链 | 只输出数学公式（equation only）；只加占位 token（dots） | GSM8K 题面语义太复杂，无法直接翻成公式；自然语言承载语义分解，占位符证明"多算力"本身没用 | 一两步的简单题（MAWPS 的 SingleOp 子集）上公式已足够，CoT 增益趋近 0 甚至为负 |
| 激发方式 | few-shot prompting | 标注 rationale 做微调（Cobbe 2021 verifier 路线） | 不需训练集，一个 checkpoint 通吃数学/常识/符号三类任务 | <100B 模型上 prompting 激发不出推理；需要格式稳定的生产场景纯 prompt 脆弱——后来被 SFT+RL（o1/R1）取代 |
| 示例数量与来源 | 8 个人工手写 exemplar | 自动生成/检索示例 | 最直接可控；作者验证换标注者、换成 GSM8K 训练集示例，增益依然显著（鲁棒性消融） | 任务分布远离示例领域时增益缩水；对示例风格敏感，不同标注者间有明显方差 |
| 解码策略 | 贪心解码单条链 | 采样多条链投票 | 保持方法最简、成本最低 | 高方差任务上单链不稳——一年内被 Self-Consistency 取代（GSM8K 再 +17.9 点） |

## 成本与量级 💰
- 训练成本量级：**0**（纯 prompting，复用现成 checkpoint）——这正是它 2022 年引爆的原因
- 推理成本 vs 基线：输出 token 增加约 3-10 倍（rationale 长度），延迟同比上升；无额外训练但把成本转嫁到每次调用
- 我的产品要用：最小可行配置——任何现代指令模型 + 3-8 个带推理过程的示例即可；注意 2026 年的模型已在训练中内化 CoT，多数场景直接开推理模式或写一句 "think step by step" 即可，手工 few-shot CoT 主要用于格式控制和领域定制

## 证据审计 🔬
- 实验设计公平吗？与同模型标准提示的对比公平；但与"prior supervised SOTA"的对比隐含不公——监督方法用的模型小几个数量级，CoT 的胜利很大程度是 540B 规模的胜利，论文用"涌现"叙事把这一点讲成了 feature。
- 最强证据：GSM8K 17.9%→56.9%（PaLM 540B，8 个手写示例），且跨 LaMDA/GPT-3/PaLM 三个模型家族、跨换示例/换标注者的消融都成立。成立条件：≥100B 模型 + 多步任务 + 基线 scaling 平坦。
- 最可疑的数字：LaMDA 137B 的人工检查称"50 个答对样本中推理链几乎全部逻辑正确"——样本只有 50 例、单一模型，后续研究（unfaithful CoT）已证明推理链可以与模型真实计算过程脱节，"链对答案对"不代表因果。
- 如果我是审稿人，会要求补充：① 为什么 <100B 失效——是推理能力缺失还是格式跟随失败？② rationale 长度 vs 准确率的关系（dots 消融太弱，不足以排除"更多计算"混淆变量）。
- 最小复现实验：OpenAI 兼容 API + GSM8K 抽 50 题，直接回答 vs CoT 对比，预算 <1 美元（见动手练习）。历史校准：2026 年模型差距会远小于 2022 年，因为 CoT 已被训进模型。

## 可复用点（PM 决策）
- **何时采用**：多步数学/逻辑/规划任务；可以用延迟和 token 成本换准确率；需要可审计的中间过程（合规场景额外受益）。
- **何时规避**：简单抽取/分类任务（加 CoT 平添成本甚至"过度思考"引入错误）；延迟敏感的实时链路；已在用推理模型时再叠 step-by-step 指令属于重复施工。
- **供应商拷问清单**：
  1. "关闭思维链/推理模式后，你们模型的 GSM8K、MATH 掉多少？"——探明能力是训练内化的还是靠 prompt 撑的
  2. "推理 token 怎么计费？平均 rationale 多长？"——CoT 的真实成本全在输出 token
  3. "你们的 CoT 中间步骤和最终答案的一致率有测过吗？"——考察 unfaithful reasoning 风险

## 关联网络 🕸️
- 相关论文：
  - [[Wiki/论文笔记/05_推理思维链/71_LetsVerifyStepByStep过程奖励模型]] → PRM 对 CoT 每一步打分验证，是 CoT 的过程监督延伸
  - [[Wiki/论文笔记/01_LLM基础架构/69_PaLM用Pathways扩展语言建模]] → CoT 最强结果的底座模型，两文同期同厂
  - [[Wiki/论文笔记/05_推理思维链/73_ReAct推理与行动的协同]] → 把 Thought 与外部 Action 交错，是 CoT 在 Agent 场景的扩展
  - [[Wiki/论文笔记/05_推理思维链/84_DeepSeek-R1强化学习激励推理能力]] → CoT 的 RL 化终点
- 相关概念：
  - [[Wiki/概念/03_推理与评测/思维链与推理模型]] → 本论文是该概念的直接源头
  - [[Wiki/概念/03_推理与评测/测试时计算扩展]] → o1/R1 的 test-time scaling 本质是把 CoT 步骤大规模 RL 化
  - [[Wiki/概念/03_推理与评测/自洽性采样]] → 本文"设计决策解剖"已指出：贪心单链解码一年内即被 Self-Consistency（多链采样+多数投票）取代，GSM8K 再 +17.9 点
- **冲突/印证**：
  - **印证**——[[Wiki/论文笔记/05_推理思维链/84_DeepSeek-R1强化学习激励推理能力]] 证明 CoT 能力可从纯 RL 结果奖励中自发涌现，把本文"需要人工示例激发"推进为"无需人工示例"
  - **冲突**——[[Wiki/论文笔记/05_推理思维链/73_ReAct推理与行动的协同]] 的人工分析显示纯 CoT 的成功案例中 14% 是"答案对但事实是编的"（假阳性），直接挑战本文"推理链基本正确"的 50 例抽查结论

## 动手练习 💻（15-45分钟）
练习目标：亲手复现论文核心对比——同一个模型、同样 8 道 GSM8K 样题，"只许给数字" vs "先推理再作答"的准确率差异。

```python
# ============================================================
# 实验：直接回答 vs CoT 提示，在 8 道 GSM8K 真题上对比准确率
# 依赖：pip install openai   （其余全是标准库）
# 运行前：设置环境变量 OPENAI_API_KEY（兼容任何 OpenAI 格式服务）
# ============================================================
import re                      # 正则表达式库，用来从模型回复里抽取数字
from openai import OpenAI      # OpenAI 官方客户端

client = OpenAI()              # 创建客户端，自动读取 OPENAI_API_KEY 环境变量
MODEL = "gpt-4o-mini"          # 故意用小模型：模型越小，CoT 差距越明显

# 8 道 GSM8K 真题，每项是 (题目, 正确答案) 元组
PROBLEMS = [
    ("Natalia sold clips to 48 of her friends in April, and then she sold "
     "half as many clips in May. How many clips did Natalia sell altogether "
     "in April and May?", 72),
    ("Weng earns $12 an hour for babysitting. Yesterday, she just did 50 "
     "minutes of babysitting. How much did she earn?", 10),
    ("Betty is saving money for a wallet which costs $100. She has only half "
     "of the money she needs. Her parents give her $15, and her grandparents "
     "twice as much as her parents. How much more money does she need?", 5),
    ("James writes a 3-page letter to 2 different friends twice a week. "
     "How many pages does he write a year?", 624),
    ("Mark planted flowers: ten are yellow, and there are 80% more purple "
     "ones. Green flowers are 25% as many as yellow and purple combined. "
     "How many flowers does Mark have?", 35),
    ("Albert buys 2 large pizzas (16 slices each) and 2 small pizzas "
     "(8 slices each). If he eats it all, how many slices is that?", 48),
    ("A robe takes 2 bolts of blue fiber and half that much white fiber. "
     "How many bolts in total does it take?", 3),
    ("Tina makes $18.00 an hour. Overtime (beyond 8 hours per shift) is paid "
     "at hourly wage + 1/2 hourly wage. If she works 10 hours every day for "
     "5 days, how much money does she make?", 990),
]

# 两种系统指令：唯一的区别就是"许不许先写推理过程"
DIRECT_SYS = "You are a math solver. Reply with ONLY the final number. No explanation."
COT_SYS = ("You are a math solver. Think step by step: write out your reasoning "
           "first, then end with the line 'Answer: <number>'.")

def ask(system_prompt, question):
    """调用一次 LLM，返回回复文本"""
    resp = client.chat.completions.create(
        model=MODEL,
        temperature=0,                                        # 温度 0，让结果可复现
        messages=[
            {"role": "system", "content": system_prompt},     # 系统指令决定"直接答"还是"先推理"
            {"role": "user", "content": question},            # 用户消息就是题目本身
        ],
    )
    return resp.choices[0].message.content                    # 取出回复正文

def extract_number(text):
    """从回复中抽取最后一个数字作为最终答案（论文的标准评测做法）"""
    nums = re.findall(r"-?\d+(?:\.\d+)?", text.replace(",", ""))  # 找出所有数字（先去掉千分位逗号）
    return float(nums[-1]) if nums else None                      # 取最后一个；没有数字则返回 None

def run(mode_name, system_prompt):
    """用指定模式跑完 8 题，返回答对数量"""
    correct = 0
    for q, gold in PROBLEMS:                                  # 逐题遍历
        reply = ask(system_prompt, q)                         # 问模型
        pred = extract_number(reply)                          # 抽取模型给出的数字
        ok = pred is not None and abs(pred - gold) < 1e-6     # 与标准答案比对
        correct += ok                                          # True 会被当作 1 累加
        print(f"[{mode_name}] 预测={pred} 标准答案={gold} {'对' if ok else '错'}")
    return correct

if __name__ == "__main__":
    n = len(PROBLEMS)
    direct = run("直接回答", DIRECT_SYS)                       # 模式一：只许给数字
    cot = run("CoT 提示", COT_SYS)                             # 模式二：先写推理再作答
    print(f"\n直接回答准确率: {direct}/{n} = {direct/n:.0%}")
    print(f"CoT  提示准确率: {cot}/{n} = {cot/n:.0%}")
    # 延伸实验：① 换更强的模型，观察差距缩小（CoT 已被内化）
    # ② 把 COT_SYS 改成只输出公式（equation only），复现论文的关键消融
```

**预期观察**：小模型上 CoT 模式应明显高于直接回答；若差距很小，说明该模型已把推理内化——这本身就是 2022→2026 年技术演进的活证据。

## 自测三层 🎓
- **L1 复述**：① CoT 把 PaLM 540B 的 GSM8K 从多少提到多少？外接计算器后多少？② 涌现阈值约多少参数？100B 以下加 CoT 会怎样？
- **L2 解释**：① 为什么"只输出公式"在 GSM8K 上无效、在 SVAMP 上却有效？这揭示了自然语言推理链的什么作用？② 当时已有"微调+verifier"路线拿到 55%，为什么 CoT prompting 仍是更重要的贡献？它相对微调赢在哪、输在哪？
- **L3 应用**：你负责一款 K12 数学辅导 App，团队想用 7B 端侧模型省成本。本论文哪个结论直接警告了这个方案？结合 R1-Distill 的存在，你会给出什么替代路线？

📅 知识时间锚：论文 2022-01（arXiv）/ NeurIPS 2022；本笔记 2026-07-04 对照 PDF 复核，GSM8K 数字确认原文 Table 值为 17.9→56.9（计算器 58.6）。
