---
tags: [论文笔记, Agent框架, 自我改进, 运行装置, 提示工程, 自动化]
笔记层级: 标准
paper_id: "22"
filename: "22 - Self-Harness - Harnesses That Improve Themselves.pdf"
authors: "Hangfan Zhang, Shao Zhang et al. (Shanghai AI Lab)"
year: 2026
成熟度: 🔧
---

# Self-Harness：Agent 自我改进运行框架

📄 **原文 PDF**：[[RAW/22 - Self-Harness - Harnesses That Improve Themselves.pdf]]

## PM 速判（3 分钟）

> **这篇论文给 PM 的一句话**：Agent 的表现由基础模型和"运行装置（harness）"共同决定——Self-Harness 让 **Agent 通过分析自身失败轨迹来改进自己的 harness**（提示/工具/策略/记忆机制），无需人工工程师或更强的外部模型指导；在 Terminal-Bench-2.0 上三个模型分别提升 +21.4pp、+14.3pp、+14.2pp，且不同模型自动发展出不同的 harness 修改。

| 项目 | 评估 |
|------|------|
| **三步循环** | 弱点挖掘 → Harness 提案 → 回归验证 |
| **关键设计** | 最小化修改（targeted, not generic）+ 双分割验证（held-in + held-out）|
| **测试平台** | Terminal-Bench-2.0（终端/CLI 复杂任务） |
| **效果** | 三个不同系列模型均持续改进，最高 +21.4pp |
| **成熟度** | 🔧 框架实现完整，SAIL，可复现 |

---

## 核心机制

```
背景：什么是 Agent Harness？
  = 围绕模型的运行系统：提示词/工具定义/记忆策略/输出策略/执行策略
  ≠ 模型权重本身
  重要性：同一模型配不同 harness 性能差异巨大

Self-Harness 循环（每轮 t）：
  1. 弱点挖掘（Weakness Mining）
     - 在 held-in 任务集上运行当前 harness
     - 收集所有执行轨迹（成功 + 失败）
     - 聚类失败轨迹 → 识别反复出现的失败模式
     - 产出：结构化的"失败证据包"（Evidence Bundle）
  
  2. Harness 提案（Harness Proposal）
     - 同一模型作为"提案者"角色
     - 基于失败证据，生成 K 个候选 harness 修改
     - 约束：每个修改必须"最小化"且"针对特定失败机制"
     - 禁止替换整体控制架构（只做局部修改）
  
  3. 提案验证（Proposal Validation）
     - 对每个候选修改做回归测试（held-in + held-out 双分割）
     - 接受条件：两个分割上都不下降，且至少一个有改进
     - 通过的修改合并进当前 harness → 下一轮
     - 拒绝的修改记录但不更改 harness
  
  → 循环 T 轮，返回最终 harness（模型权重全程不变）

GLM-5 实际发现的 harness 修改（案例）：
  ✓ 会话级工具（persist PATH and verify）
  ✓ 外部计算限制（bound, stage, inspect downloads）
  ✓ 早期输出工件（write artifacts early, force cleanup）
  ✓ 实现方向建议（shift to build/test pattern）
  → 这些修改是模型特异性的，Qwen 和 MiniMax 得到了不同修改
```

---

## 关键数据

| 模型 | 初始 Pass Rate | Self-Harness 后 | 提升 |
|------|----------------|-----------------|------|
| MiniMax M2.5 | 40.5% | **61.9%** | +21.4pp |
| Qwen3.5-35B-A3B | 23.8% | **38.1%** | +14.3pp |
| GLM-5 | 42.9% | **57.1%** | +14.2pp |

基准：Terminal-Bench-2.0（64 个终端复杂任务）

---

## 核心结论（带时间锚）

1. **Agent 可以分析自身失败并有效改进运行框架，无需人工介入** 📅 2026
   三个不同系列模型均在 Terminal-Bench-2.0 上持续改进——说明 Self-Harness 循环不依赖模型特殊能力，具有普适性。

2. **不同模型产生不同的 harness 修改：模型特异性确实存在** 📅 2026
   Qualitative 分析确认：GLM-5、Qwen、MiniMax 自动发展出不同的 harness 修改——这证明了手工设计统一 harness 的局限性。

3. **最小化修改 + 回归验证是防止过拟合的关键** 📅 2026
   不加约束的 harness 修改（通用指令 vs 针对特定失败）会导致 held-out 退化；最小化约束 + 双分割验证是让改进泛化的核心机制。

---

## 对 AI PM 的启示

| 场景 | 洞察 |
|------|------|
| **Agent 产品维护** | Self-Harness 框架是减少人工 harness 调优成本的重要路径——可以让 Agent 自己发现提示缺陷 |
| **多模型部署** | 当产品同时支持多个底座模型时，为每个模型维护单独 harness 成本高——Self-Harness 式自动化是解法 |
| **生产 Agent 迭代** | 失败轨迹聚类 → 针对性修改 → 回归验证：这是比"人工看日志改提示"更系统的迭代方法 |
| **注意事项** | 需要高质量的失败判断器（verifier）——如果 pass/fail 评估不准确，Self-Harness 会学到错误的改进方向 |

---

## 权威学习资源

- 📄 论文：Zhang, Zhang et al.（上海人工智能实验室），2026
- 🔗 基准：Terminal-Bench-2.0

## 论文精读卡片

**一句话**：Self-Harness 让 Agent 通过三步循环（弱点挖掘→Harness 提案→回归验证）分析自身失败轨迹并改进运行框架（提示/工具/策略），在 Terminal-Bench-2.0 上三个不同系列模型分别提升 +21.4pp、+14.3pp、+14.2pp，且不同模型自动发展出模型特异性的不同修改。

**问题**：Agent 的表现由基础模型和"运行装置（harness）"共同决定，当前 harness 优化需要人工工程师反复查看失败日志、修改提示——成本高、速度慢，且只能针对观察到的失败，无法覆盖所有边角情况；同时不同基础模型需要不同的最优 harness，人工维护多份 harness 成本极高。

**核心方法**：
- 弱点挖掘：在 held-in 任务集上运行当前 harness，收集失败轨迹，聚类识别反复出现的失败模式，产出结构化"失败证据包"
- Harness 提案：同一模型作为提案者，基于失败证据生成 K 个候选 harness 修改，约束每个修改必须"最小化且针对特定失败机制"
- 双分割回归验证：在 held-in 和 held-out 两个分割上测试，两者都不退化且至少一个提升才接受修改；拒绝过于通用的修改

**关键图/公式**：三步循环示意图——循环 T 轮，每轮接受通过验证的修改并合并进 harness，被拒绝的修改记录但不影响当前 harness；核心洞察是"最小化修改 + 双分割验证"同时解决了过拟合（held-in 过强）和泛化不足（只测 held-in）两个问题。

**实验设置**：
- 规模/数据：Terminal-Bench-2.0（64 个终端 CLI 复杂任务）
- 对比：各模型的初始 harness 基线（MiniMax M2.5: 40.5%，Qwen3.5-35B-A3B: 23.8%，GLM-5: 42.9%）

**最强证据**：三个完全不同系列的模型（MiniMax/Qwen/GLM）均显著提升（+14pp~+21pp），且提升幅度在同一量级——这排除了"某个模型特别适合自我修改"的解释，说明 Self-Harness 机制本身有普适效果。

**最弱证据**：Terminal-Bench-2.0 只有 64 个任务，统计功效有限，+14pp 的置信区间未报告；自我修改的循环轮数（T）对最终效果的影响未系统分析；验证器（判断任务 pass/fail）的准确性是整个系统的关键假设，但在实际产品部署中终端命令的 pass/fail 判断比代码测试更难自动化。

**可复用点**：将 Self-Harness 的三步循环应用于 Agent 产品的持续改进管线：定期收集失败会话→聚类失败模式→让 LLM 生成针对性的提示修改候选→用历史成功案例做回归测试→只接受不退化的修改；这比"人工看日志改提示"系统化且可扩展。

**和哪些论文相关**：
- [[Wiki/概念/04_Agent框架/Harness设计模式]] — Self-Harness 是 Harness 架构的自动化改进机制
- [[Wiki/概念/02_训练方法/递归自我改进]] — Self-Harness 是递归自我改进在运行框架层（而非权重层）的实例
- [[Wiki/论文笔记/06_训练对齐RL/15_从学员到教练LLM设计自身训练环境]] — 同为"模型分析失败后自动改进"，一个改 harness，一个改训练环境设计

**我能拿它做什么**：
- 为 Agent 产品设计"自动 harness 优化"管线：每周从失败会话中抽取样本，用 LLM 分析失败模式并提出提示修改，用 A/B 测试验证修改效果
- 向团队说明：不同底座模型部署时需要不同的 harness 优化策略（GLM vs Qwen vs MiniMax 发展出不同修改就是证据），推动模型特异性的 harness 维护
- 用 Self-Harness 三步框架（失败聚类→针对性修改→回归测试）替代当前的"人工看日志改提示"流程，降低 Agent 运维成本

**3天后要回忆的问题**：
1. Self-Harness 的三步循环是什么？每步的输入和输出分别是什么？
2. 为什么"最小化修改"约束和"双分割验证"是防止过拟合的关键？
3. 三个模型（MiniMax/Qwen/GLM）在 Terminal-Bench-2.0 上分别提升了多少？
4. 为什么不同模型产生了不同的 harness 修改？这对"统一 harness 设计"的实践意味着什么？
5. Self-Harness 系统最关键的依赖假设是什么？在实际产品部署中这个假设成立的条件是什么？

## 原子概念索引

- [[Wiki/概念/04_Agent框架/Harness设计模式]] — Self-Harness 是 Harness 架构的自动化迭代改进机制
- [[Wiki/概念/02_训练方法/递归自我改进]] — Self-Harness 是递归自我改进在运行框架层（非权重层）的实现
