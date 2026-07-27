---
tags: [MOC, Agent, 系统架构, 导航]
created: 2026-07-04
updated: 2026-07-04
---

# 🏗️ MOC：Agent 系统架构全景

> **从 Agent 设计模式到生产部署的完整视图。** 本 MOC 跨「04_Agent框架」和「05_记忆与检索」两大类别（见 [[Wiki/概念导航索引]]），串联 Harness、记忆、工具、多 Agent 协作等子领域。

---

## 一、核心架构范式

### 1.1 Harness 设计模式（地基）
Agent 系统的"操作系统"——负责状态管理、工具调用、错误恢复等非语义工作。

- [[Wiki/概念/04_Agent框架/Harness设计模式]] — 核心概念：5 功能层 + 类型谱系
- [[Wiki/概念/04_Agent框架/ReAct与Agent推理行动框架]] — Thought→Action→Observation 循环
- [[Wiki/概念/04_Agent框架/动态智能体脚手架]] — 推理时按需生成的 Agent 协作结构

**关键论文：**
- [[Wiki/论文笔记/03_Agent系统设计/39_Harness-1状态外化搜索Agent]] — 状态外化：7 组件设计
- [[Wiki/论文笔记/03_Agent系统设计/45_LIFE-HARNESS适配接口而非模型]] — 4 层生命周期，90% 失败是接口问题
- [[Wiki/论文笔记/03_Agent系统设计/49_执行轨迹对齐的Harness推理时设计理论]] — 部分 Harness 最优原则
- [[Wiki/论文笔记/03_Agent系统设计/22_Self-Harness-Agent自我改进运行框架]] — Agent 自动优化自身 Harness
- [[Wiki/论文笔记/03_Agent系统设计/32_Harness更新vs获益-自进化Agent能力解耦]] — Harness vs 模型权重贡献解耦
- [[Wiki/论文笔记/03_Agent系统设计/34_Agent-Harness有效反馈计算扩展律]] — 有效反馈计算（EFC）扩展律
- [[Wiki/论文笔记/04_Agent技能工具/42_将Agent工作流编译进LLM权重]] — Harness 的极端：编译进权重

### 1.2 Agent 记忆系统
跨 session 持续积累的信息存储，从短期上下文到长期记忆。

- [[Wiki/概念/05_记忆与检索/Agent记忆系统四模块]] — R（表示）→E（抽取）→Q（检索）→U（维护）
- [[Wiki/概念/05_记忆与检索/Agent长期记忆]] — 跨 session 持久化
- [[Wiki/概念/05_记忆与检索/原子事实记忆]] — 基于原子事实的精细粒度记忆
- [[Wiki/概念/05_记忆与检索/记忆即模型]] — 记忆作为模型参数的理论

**关键论文：**
- [[Wiki/论文笔记/09_长上下文记忆/02_Agent记忆系统综述]] — 系统综述
- [[Wiki/论文笔记/09_长上下文记忆/18_AtomMem基于原子事实的LLM-Agent长期记忆]] — 原子事实记忆实现
- [[Wiki/论文笔记/09_长上下文记忆/124_MeMo记忆即模型]] — 记忆即模型
- [[Wiki/论文笔记/09_长上下文记忆/94_TaskMem多模态Agent的任务聚焦记忆]] — 任务聚焦记忆
- [[Wiki/论文笔记/09_长上下文记忆/135_delta-mem轻量在线记忆机制]] — 轻量在线记忆
- [[Wiki/论文笔记/09_长上下文记忆/36_AdaCoM长视野Agent自适应上下文管理]] — 长视野上下文管理
- [[Wiki/论文笔记/09_长上下文记忆/44_LLM是否需要睡眠离线递归的记忆巩固]] — 离线记忆巩固

### 1.3 工具使用与技能系统
Agent 如何发现、学习、迁移和使用工具。

- [[Wiki/概念/04_Agent框架/函数调用与工具使用]] — Agent 工具调用的基础机制
- [[Wiki/概念/04_Agent框架/Computer Use与GUI Agent]] — 图形界面交互 Agent
- [[Wiki/概念/04_Agent框架/模型路由与CAF循环]] — Context→Action→Feedback 路由闭环

**关键论文：**
- [[Wiki/论文笔记/04_Agent技能工具/74_Toolformer语言模型自学使用工具]] — 自监督工具学习范式
- [[Wiki/论文笔记/04_Agent技能工具/11_SpatialClaw空间推理动作接口]] — 空间推理动作接口
- [[Wiki/论文笔记/04_Agent技能工具/13_PreAct重复任务加速的计算机使用Agent]] — 重复任务加速
- [[Wiki/论文笔记/04_Agent技能工具/12_SKILLWEAVER组合技能路由]] — 组合技能路由
- [[Wiki/论文笔记/04_Agent技能工具/16_OpenClaw-Skill集体技能树搜索Agent技能构建]] — 集体技能树
- [[Wiki/论文笔记/04_Agent技能工具/19_SKILLMIGRATOR跨域Web技能迁移]] — Web 技能迁移
- [[Wiki/论文笔记/04_Agent技能工具/41_SkillOpt-Agent技能的文本空间优化器]] — 技能文本优化

---

## 二、多 Agent 与集体智能

### 2.1 多 Agent 协作
- [[Wiki/概念/04_Agent框架/LLM集体智能]] — 多模型互补优势
- [[Wiki/概念/04_Agent框架/并发Agent管理]] — 多 Agent 并发与资源管理
- [[Wiki/概念/04_Agent框架/学习型编排器]] — 训练模型动态编排 Agent

**关键论文：**
- [[Wiki/论文笔记/03_Agent系统设计/136_多Agent系统协作失败归因自进化综述]] — LIFE 框架综述
- [[Wiki/论文笔记/03_Agent系统设计/40_多Agent是否有助于性能提升受控评测]] — 公平对比：多 Agent vs 单 Agent
- [[Wiki/论文笔记/03_Agent系统设计/25_Agentopia-长期生命仿真和Agent社会学习]] — 100 Agent 仿真 10 年
- [[Wiki/论文笔记/03_Agent系统设计/43_AutoScientists自组织Agent团队科学发现]] — 自组织 Agent 科研团队
- [[Wiki/论文笔记/03_Agent系统设计/01_Sakana-Fugu技术报告]] — 编排器模型动态路由
- [[Wiki/论文笔记/03_Agent系统设计/139_AEVO元编辑Agent进化框架]] — 元 Agent 编辑进化机制
- [[Wiki/论文笔记/03_Agent系统设计/140_记忆诅咒扩展记忆损害多Agent协作]] — 记忆诅咒
- [[Wiki/论文笔记/04_Agent技能工具/08_Skill-MAS元技能自动多智能体]] — 元技能多 Agent

### 2.2 Agent 通信协议
- [[Wiki/概念/04_Agent框架/MCP]] — Model Context Protocol
- [[Wiki/概念/04_Agent框架/A2A]] — Agent-to-Agent 协议
- [[Wiki/概念/04_Agent框架/分层Agent协议栈]] — 协议分层架构
- [[Wiki/概念/04_Agent框架/Agent通信三元困境]] — 效率/质量/成本三元权衡
- [[Wiki/概念/04_Agent框架/Agent发现机制]] — Agent 服务发现

**关键论文：**
- [[Wiki/论文笔记/03_Agent系统设计/06_LLM-Agent通信协议分类]] — 通信协议技术分类

---

## 三、Agent 生命周期与可靠性

### 3.1 生命周期管理
- [[Wiki/概念/04_Agent框架/Agent寿命工程]] — 生产 Agent 的维护与演化
- [[Wiki/概念/04_Agent框架/任务视野]] — Agent 能处理的任务时长
- [[Wiki/论文笔记/13_评测科研/48_AgingBench-你的Agent也在老化]] — Agent 老化评测

### 3.2 安全与边界
- [[Wiki/概念/04_Agent框架/Agentic意图越界]] — Agent 执行未授权操作
- [[Wiki/概念/04_Agent框架/一致性幻觉]] — 多 Agent 辩论达成共识 ≠ 正确
- [[Wiki/概念/04_Agent框架/随机确定边界]] — LLM 适合语义决策 vs 确定性计算
- [[Wiki/概念/04_Agent框架/Agentic与Agentive区分]] — 外部脚手架 vs 内部涌现

**关键论文：**
- [[Wiki/论文笔记/03_Agent系统设计/29_一致性幻觉-多Agent辩论隐藏推理不对齐]] — 一致性幻觉的实证发现
- [[Wiki/论文笔记/03_Agent系统设计/128_生产LLM-Agent运行架构模式方法论]] — 生产架构模式：71% 问题是架构问题
- [[Wiki/论文笔记/03_Agent系统设计/04_Agent模型批判]] — Agentic vs Agentive 的哲学辨析
- [[Wiki/论文笔记/04_Agent技能工具/14_LLM-Agent世界模型推断能力评测]] — 随机-确定边界实证

---

## 四、实用导航

### 按成熟度筛选
| 成熟度 | 概念 | 说明 |
|--------|------|------|
| ✅ 成熟 | [[Wiki/概念/04_Agent框架/Harness设计模式]] | 理论基础+工程实践均完备 |
| 🔧 发展中 | [[Wiki/概念/04_Agent框架/动态智能体脚手架]] | 前沿研究活跃 |
| 🔧 发展中 | [[Wiki/概念/04_Agent框架/LLM集体智能]] | 多 Agent 协作方案仍在演进 |
| ⚪ 待补充 | [[Wiki/概念/04_Agent框架/Agent寿命工程]] | 概念待填充 |

### 按生产就绪度
```
立即可用 ───────────────── 研究前沿
   ReAct        Harness-1      Self-Harness
   MCP/A2A      LIFE-HARNESS   动态脚手架
   Toolformer   Memory-model   元编辑进化
```

---

📅 **最后更新**：2026-07-04
