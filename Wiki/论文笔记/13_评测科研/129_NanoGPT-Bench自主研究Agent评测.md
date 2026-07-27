---
tags: [论文笔记, AI科研, Agent评测, 自主研究, 基准测试]
笔记层级: 标准
paper_id: "129"
filename: "129 - NanoGPT-Bench - Evaluating Autonomous Research Agents on the NanoGPT Speedrun.pdf"
authors: "Intology AI"
year: 2026
成熟度: 🧪
---

# NanoGPT-Bench：自主研究 Agent 的评测基准

📄 **原文 PDF**：[[RAW/129 - NanoGPT-Bench - Evaluating Autonomous Research Agents on the NanoGPT Speedrun.pdf]]

## PM 速判（3 分钟）

> **这篇论文给 PM 的一句话**：Claude Code、Codex、Autoresearch 三种前沿 Agent 在 512 H100 GPU 小时的算力预算下，只能恢复人类 5 个月研究进展的 **<10%**——Agent 主要做超参调优，几乎不能做真正的算法创新。

| 项目 | 评估 |
|------|------|
| **最高分 Agent** | Autoresearch（Karpathy harness + Claude Opus 4.6）= 9.3% |
| **Claude Code 成绩** | Opus 4.6 Max = 8.2% |
| **Codex 成绩** | GPT-5.4 xhigh = 8.6% |
| **人类对比** | 5个月内累计提速 63.2 秒；Agent 5个月内 <10% |
| **成熟度** | 🧪 评测研究，揭示当前 Agent AI 科研能力上限 |

---

## 评测设置

```
任务：NanoGPT Speedrun 挑战
  目标：从 2025年9月人类世界纪录出发
        自主优化 GPT-2 预训练速度
  衡量：相对于 2026年1月19日人类世界纪录的提速百分比
  
关键设计：
  ✓ 从已优化的人类纪录出发（排除低悬果实）
  ✓ 长视野人类对比（5个月人类记录）
  ✓ 开放式研究任务（非结构化指令）
  ✓ 自动提交验证（LLM judge + 统计显著性检验）
  
算力预算：512 H100 GPU 小时/Agent
周期：最长运行 1 周实际时间

强制跑完：用"Ralph Wiggum"循环阻止 Agent 提前终止
```

---

## 核心发现：Agent 只会调参，不会创新

```
各 Agent 行为分析（跨度分类）：
  Codex：几乎完全避免算法研究
  Claude Code：推理讨论算法研究 > Codex，但仍回避实现
  Autoresearch：最多算法研究尝试，但成功率低
  
对比人类成功路径：
  Muon 优化器（算法创新）
  Polar Express 矩阵乘法（算法创新）
  NorMuon 优化器（算法创新）
  
Agent 最终选择：调超参（学习率、batch size、epoch 数）
```

---

## 核心结论（带时间锚）

1. **当前 Coding Agent 无法在有竞争力的 AI 研究基准上匹配人类** 📅 2026
   最优 Agent（Autoresearch，9.3%）对应约 7 天人类研究进展；实际人类 5 个月累计进展 100%——差距约 10 倍。

2. **算法研究能力是当前 Agent 的核心瓶颈，不是算力** 📅 2026
   给了 512 H100 小时（超过大多数个人研究者能动用的算力）仍只有 <10% 进展——瓶颈是"提出新算法想法并实现"的能力，不是计算资源。

3. **NanoGPT-Bench 设计为低污染、高性能余量的基准** 📅 2026
   相比 RE-Bench、PaperBench 等，NanoGPT-Bench 有优化初始化 + 长视野人类参考 + 开放研究性质三合一，更能测出真实 AI R&D 能力上限。

---

## 对 AI PM 的启示

| 洞察 | 产品决策影响 |
|------|------------|
| **AI 做不了前沿研究（目前）** | 不要过度依赖 AI 自主发现新方法；AI 科研 Agent 适合加速人类已知方向，不适合开拓未知 |
| **超参优化 vs 算法创新** | AI Agent 在超参搜索上已经相当有用（快速遍历大量变体）；算法创新仍需人类主导 |
| **基准设计：避免低悬果实** | 如果在内部测试 AI 辅助研究效率，需要从已优化基线出发——否则结果被容易的改进虚高 |
| **递归自我改进的当前局限** | 结合 Paper 125（AIRA），Agent 在有约束搜索空间内能超越人工设计；但在开放研究任务中仍大幅落后 |

---

## 权威学习资源

- 📄 论文（博客）：Intology AI，2026年5月
- 🔗 项目：[NanoGPT-Bench GitHub](https://github.com/IntologyAI/NanoGPT-Bench)
- 🔗 参考：Karpathy Autoresearch harness（最强 Agent 的脚手架）
- 🔗 相关：[[Wiki/论文笔记/05_推理思维链/125_AIRA智能体神经架构搜索]] — 有约束空间内 AI 自主架构发现

## 论文精读卡片

**一句话**：Claude Code（Opus 4.6 Max，8.2%）、Codex（GPT-5.4 xhigh，8.6%）和 Autoresearch（9.3%）在 512 H100 GPU 小时预算下只能恢复人类 5 个月 NanoGPT 优化进展的不到 10%——算力不是瓶颈，Agent 无法提出新算法想法（Muon 优化器等）才是核心限制。

**问题**：当前最强的 Coding Agent 在开放式 AI 研究任务中的真实能力上限是什么？如何设计评测基准才能避免"低悬果实"污染结果，真实反映 AI 研究能力？

**核心方法**：
- 基准设计：从 2025年9月人类世界纪录出发（排除低悬果实），衡量相对 2026年1月19日纪录的提速百分比，5个月人类进展作为100%参考
- 控制变量：512 H100 GPU 小时/Agent（超过大多数个人研究者资源）；最长1周实际时间；Ralph Wiggum 循环强制 Agent 持续尝试不提前终止
- 三个 Agent 对比：Autoresearch（Karpathy harness + Claude Opus 4.6）、Claude Code（Opus 4.6 Max）、Codex CLI（GPT-5.4 xhigh）

**关键图/公式**：行为分类对比——Codex 几乎完全避免算法研究，Claude Code 讨论算法研究但回避实现，Autoresearch 最多算法研究尝试但成功率低；人类成功路径（Muon/Polar Express/NorMuon）全部是算法创新，Agent 选择的全是调超参。

**实验设置**：
- 规模/数据：NanoGPT Speedrun 挑战，GPT-2 预训练优化；512 H100 GPU 小时/Agent，最长1周
- 对比：Autoresearch 9.3%、Codex 8.6%、Claude Code 8.2% vs 人类5个月累计 100%（63.2秒绝对提速）

**最强证据**：三个 Agent 使用了超过大多数个人研究者能动用的算力（512 H100小时）仍只有 <10% 进展——排除了"算力不够"的解释，明确指向"提出新算法想法并实现"的能力缺口。

**最弱证据**：仅有三个 Agent 的单点测试，没有统计误差棒；NanoGPT Speedrun 是特定类型的研究任务（训练优化），不代表所有 AI 科研能力；512 H100 GPU 小时的算力使用方式可能未被 Agent 优化。

**可复用点**：内部测试 AI 辅助研究效率时，从已优化基线出发（避免低悬果实）的评测设计原则；Oracle 最优选择作为 Agent 覆盖范围上限的测量工具（结合 Paper 126）；Ralph Wiggum 强制持续尝试机制用于 Agent 评测的行为完整性保证。

**和哪些论文相关**：
- [[Wiki/论文笔记/05_推理思维链/125_AIRA智能体神经架构搜索]] — 直接互补：AIRA 在有约束搜索空间内超越人类，NanoGPT-Bench 在开放研究任务中大幅落后——两者共同描绘 AI 科研能力边界
- [[Wiki/论文笔记/03_Agent系统设计/126_弱推理模型的Agentic增强]] — 结合理解：多 Agent 在代码场景有效（有验证信号），但开放研究任务的验证信号（发现新算法有效）本身就是 Agent 的能力瓶颈
- [[Wiki/概念/02_训练方法/递归自我改进]] — NanoGPT-Bench 展示了 RSI 在开放任务中的当前局限

**我能拿它做什么**：
- 向内部决策者解释"AI 科研 Agent"现状时，用"<10% 进展 vs 5个月人类进展"作为锚点
- 规划 AI 辅助研究产品时，定位到 Agent 真正适合的范围（加速已知方向的超参搜索）而非过度承诺（自主算法创新）
- 设计内部 AI 能力评测时，参考 NanoGPT-Bench 的三要素（优化初始化基线+长视野人类参考+开放研究性质）

**3天后要回忆的问题**：
1. 三个 Agent 的 SWE-bench 分数分别是多少？最高是哪个？
2. 为什么说算力不是 Agent 科研能力的瓶颈？实验如何排除这个解释？
3. Agent 的行为与人类成功路径有何本质差异？具体表现是什么？
4. NanoGPT-Bench 基准设计中"从已优化基线出发"解决了什么问题？
5. Ralph Wiggum 循环是什么？为什么需要它？

## 原子概念索引

- 相关：[[Wiki/概念/02_训练方法/递归自我改进]] — NanoGPT-Bench 展示了 RSI 在开放任务中的当前局限
- 相关：[[Wiki/概念/04_Agent框架/Harness设计模式]] — Karpathy Autoresearch harness 是最高分 Agent 的脚手架基础
