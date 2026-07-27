---
tags: [论文笔记, 搜索Agent, RL训练, 状态管理, 检索, UIUC]
笔记层级: 标准
paper_id: "39"
filename: "39 - Harness-1 - Reinforcement Learning for Search Agents with State-Externalizing Harnesses.pdf"
authors: "Pengcheng Jiang, Zhiyi Shi et al. (UIUC / UC Berkeley / Chroma)"
year: 2026
成熟度: 🔧
---

# Harness-1：状态外化 Harness 中的搜索 Agent RL 训练

📄 **原文 PDF**：[[RAW/39 - Harness-1 - Reinforcement Learning for Search Agents with State-Externalizing Harnesses.pdf]]

## PM 速判（2 分钟）

> **这篇论文给 PM 的一句话**：搜索 Agent 的核心问题是**把"语义搜索决策"和"状态追踪记账"混在一起让 LLM 来做**——Harness-1 把所有记账工作（候选池、证据图、验证记录、预算标记）移入外部 harness，让 20B 策略模型只做语义决策；结果在 8 个检索基准上达到 0.730 均值召回率，比次优开源搜索 Agent **+11.4 点**。

| 项目 | 评估 |
|------|------|
| **核心设计** | 状态外化（harness 管记账）+ 策略（模型管语义）|
| **模型大小** | 20B 参数 |
| **平均检索召回** | 0.730（8 个基准）|
| **vs 次优开源** | +11.4 点；与更大的前沿模型竞争性持平 |
| **成熟度** | 🔧 开源实现，UIUC 2026 |

---

## 核心设计：Harness vs 策略的分工

| 状态组件 | Harness 维护 | 策略决策 |
|---------|-------------|---------|
| 候选池 Pt | 压缩、去重后的候选文档 | 哪些文档要检查/阅读/整理 |
| 精选集 Ct, It | 精选集 + 重要性标签（自动预填充 top-8）| 哪些文档要添加/删除/提升/降级 |
| 全文存储 Dt | 已检索的完整文档（不内联进 prompt）| 哪些文档需要重新查阅 |
| 证据图 Gt | 实体→文档的桥接关系和单例 | 哪个桥接关系/单例值得追踪 |
| 验证缓存 Vt | 声明→文档→验证结论 | 哪些声明需要核查，用哪些文档 |
| 搜索历史 Ht | 已用工具、结果摘要、新文档计数 | 何时多样化/回溯/继续搜索线索 |
| 预算标记 Bt | 上下文预算使用情况（budget-safe 渲染）| 何时搜索/阅读/摘要/停止 |

```
原有方法：policy = 语义搜索决策 + 记账（从长 transcript 重建状态）
  → RL 必须同时优化两件事 → 学习信号模糊 → 难以泛化

Harness-1：policy = 纯语义决策（基于 harness 渲染的显式状态）
  → RL 只需优化语义部分 → 更清晰的信用归因 → 更好的迁移
```

## 核心结论（带时间锚）

1. **状态外化让 RL 信号更清晰**，检索失败可以归因到"搜索策略"而不是"记账失误" 📅 2026
2. **迁移性更强**：在 held-out 迁移基准上的提升最为明显，说明显式搜索状态帮助 RL 学到可泛化的行为 📅 2026

---

## 对 AI PM 的启示

| 场景 | 洞察 |
|------|------|
| **搜索/RAG 产品** | 多跳检索任务中，给 Agent 维护显式"候选池 + 已验证清单 + 证据图"比让模型自己追踪更可靠 |
| **Agent 训练设计** | 把不需要 LLM 做的记账任务外化到 harness；让 RL 只优化语义决策 |
| **成本效率** | 20B 模型 + 好 harness > 更大的前沿模型（+11.4 点）；harness 设计是独立于模型大小的杠杆 |

---

## 权威学习资源

- 📄 论文：Jiang, Shi et al.（UIUC / Berkeley / Chroma），2026
- 🔗 代码：[github.com/pat-jj/harness-1](https://github.com/pat-jj/harness-1)

## 论文精读卡片

**一句话**：把 Agent 的"记账工作"从 LLM 里挖出来交给 Harness，让 20B 策略模型只做语义决策，多跳检索召回率达到 0.730，比次优开源 Agent +11.4 点。

**问题**：搜索 Agent 的 RL 训练信号模糊——因为 LLM 同时做"语义搜索决策"和"状态追踪"，失败时分不清是哪个坏了。

**核心方法**：
- 7 组件状态外化：候选池/精选集/全文存储/证据图/验证缓存/搜索历史/预算标记全部移入 Harness
- Budget-safe 渲染：Harness 根据 context window 大小自动控制状态渲染量
- RL 信号纯化：策略只负责语义决策 → 信用归因清晰 → 泛化更好

**关键图/公式**：
- 状态外化示意：Harness 维护状态 → 渲染成结构化 prompt → LLM 读状态做决策 → 输出动作 → Harness 执行并更新状态

**实验设置**：
- 模型：20B 策略（RL 微调）
- 评测：8 个检索基准（多跳问答类）
- 对比：次优开源搜索 Agent、前沿模型（更大）

**最强证据**：
- 0.730 均值召回率，+11.4 pp vs 次优开源 Agent
- 迁移基准上提升最大：说明学到的是可泛化的搜索策略，不是数据记忆

**最弱证据**：
- 8 个基准全是英文检索任务——中文多跳检索、代码/工具 Agent 是否适用未验证
- 与 GPT-4 类闭源模型的比较结果描述为"竞争性"，但没有精确数字
- 20B 模型 RL 训练成本/时间未披露，难以判断工程可行性

**可复用点**：
- "语义决策 vs 记账分工"框架：任何 Agent 都可以问"哪些记账任务不需要 LLM 做？"
- Budget-safe 渲染：防止 context window 溢出的通用工程模式
- 证据图（Entity→Document）：多跳推理中桥接信息的结构化表示

**和哪些论文相关**：
- [[Wiki/论文笔记/05_推理思维链/73_ReAct推理与行动的协同]] — Harness-1 的 ReAct 基础
- [[Wiki/论文笔记/03_Agent系统设计/45_LIFE-HARNESS适配接口而非模型]] — Harness 设计的完整生命周期视角
- [[Wiki/论文笔记/03_Agent系统设计/49_执行轨迹对齐的Harness推理时设计理论]] — Partial Harness 的设计理论
- [[Wiki/论文笔记/03_Agent系统设计/40_多Agent是否有助于性能提升受控评测]] — 多 Agent 是否能补充单 Agent+Harness？
- [[Wiki/概念/04_Agent框架/Harness设计模式]] — Harness-1 是最典型的状态外化案例
- [[Wiki/概念/05_记忆与检索/RAG检索增强生成]] — 多跳检索的上下文

**我能拿它做什么**：
- 设计搜索/RAG Agent 时，明确哪些状态需要 Harness 维护（候选池、已验证结果）
- 分析 Agent 失败时：是 Harness 状态错了？还是 LLM 决策错了？分层诊断
- RL 训练 Agent 时，先外化状态，让信用归因变清晰，收敛更快

**3天后要回忆的问题**：
1. Harness-1 的 7 个外化状态组件分别是什么？
2. 状态外化如何让 RL 信号更清晰？
3. 结果：均值召回率是多少？相比次优提升多少？
4. 迁移基准上提升最大意味着什么？
5. Budget-safe 渲染解决了什么工程问题？

## 原子概念索引
- [[Wiki/概念/04_Agent框架/Harness设计模式]] — 状态外化 Harness 的典型实现
- [[Wiki/概念/04_Agent框架/ReAct与Agent推理行动框架]] — Harness-1 的底层 ReAct 范式
