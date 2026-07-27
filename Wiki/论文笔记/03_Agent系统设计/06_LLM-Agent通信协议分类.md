# LLM Agent 通信协议分类——AI PM 总结

📄 **原文 PDF**：[[RAW/06 - A Technical Taxonomy of LLM Agent Communication Protocols.pdf]]

> **来源论文**: A Technical Taxonomy of LLM Agent Communication Protocols
> **作者**: Linus Sander et al.，Technische Universität München
> **发表**: arXiv 2025.06（[arxiv.org/abs/2506.19153](https://arxiv.org)）
> **类型**: 综述 / 分类（Taxonomy）论文
> **成熟度**: 🔧 论文本身是分析框架，覆盖的协议成熟度各异
> **开源实现**: N/A（分类研究，非实现）
> **商业 API**: 覆盖的协议中，MCP ✅ 已可直接使用

---

## PM 速判（10 分钟结论）

**三个产品问题**：

| 问题 | 答案 |
|------|------|
| 解锁了什么产品能力？ | 提供 Agent 通信协议的系统地图——选型时有框架，不再凭感觉跟风 |
| 我的团队能用上吗？ | MCP 现在就能用；A2A 可以开始布局；其他协议暂时不依赖 |
| 现在能落地吗？ | MCP ✅，A2A 🏭，其他 🔬/🧪 |

**论文做了什么**：分析了 9 个开源、有实际落地的 Agent 通信协议，用 5 个维度建立了分类体系，并预测了协议生态的演进方向。

---

## 分类框架：5 个维度

论文用以下 5 个维度区分不同协议：

| 维度 | 含义 | 典型取值 |
|------|------|---------|
| **Counterparty（交互对象）** | 连的是谁 | Agent / 工具数据（Context）/ 两者都有（Hybrid）|
| **Payload（载荷类型）** | 传什么 | 结构化数据 / 会话消息 / 两者混合（Hybrid）|
| **Interaction State（状态）** | 有无会话记忆 | 无状态 / 有 Session State |
| **Discovery（发现机制）** | 怎么找对方 | 静态 / 集中式 / 去中心化 / 混合 |
| **Schema Flexibility（格式灵活性）** | 通信格式有多固定 | 单一固定 / 多种预定义 / 运行时协商（Evolving）|

**两个规律**：
- 所有 Agent-to-Agent 协议都有 Session State + Hybrid Payload（必须支持多轮、多格式）
- 绝大多数协议用静态或集中式发现，真正的去中心化发现是最大空白

---

## 工程算法解释

> 这不是让你去实现这些协议，而是让你理解"底层在做什么"——这样你在 Review 工程方案时能辨别合理性，和工程师讨论时不完全依赖他们翻译。

### MCP 的工作原理

**类比**：USB-C 标准接口。各种设备（工具/数据库/API）只要实现了 MCP Server，所有支持 MCP 的 LLM Host（MacBook）都能统一接入，不用为每个设备配专用驱动。

**底层机制**：

- **通信协议**：JSON-RPC 2.0——一种远程函数调用标准。每条请求包含 `method`（调什么）、`params`（参数）、`id`（编号）；响应返回 `result` 或 `error`
- **传输方式**：本地工具用 **stdio**（进程间管道，延迟 < 1ms）；远程服务用 **HTTP + SSE**（服务器推送，支持流式输出）
- **三种能力类型**：

| 类型 | 作用 | 举例 |
|------|------|------|
| **Resources** | LLM 可以"读"的数据 | 文件内容、数据库查询结果 |
| **Tools** | LLM 可以"调"的函数 | 发邮件、搜索、提交代码 |
| **Prompts** | 可复用的提示词模板 | 标准代码审查 prompt |

**一次 Tool 调用的数据流**：
```
LLM 决定需要"搜索"工具
 → Client 发 JSON-RPC 请求：
   {"method": "tools/call", "params": {"name": "search", "arguments": {"query": "..."}}}
 → MCP Server 执行搜索
 → 返回结果：{"content": [{"type": "text", "text": "结果内容..."}]}
 → LLM 读取结果，继续生成
```

**PM 工程直觉**：MCP Server 就是给现有系统（API、数据库、内部工具）包一层 JSON-RPC 外壳，暴露给 LLM。**任何现有内部系统，写一个 MCP Server 就能让 AI 用上，不需要改原系统**。

---

### A2A 的工作原理

**类比**：外卖平台接单系统。你（主 Agent）发布订单（Task），平台（注册中心）知道哪些商家（子 Agent）能接单，商家接单后持续更新状态，你实时收到进展通知。

**底层机制**：

- **通信协议**：REST API（HTTP + JSON）——比 JSON-RPC 更通用，任何语言任何框架都能实现
- **Agent Card**：每个 Agent 的"能力说明书"，JSON-LD 格式，固定发布在：

```
GET https://your-agent.com/.well-known/agent.json
```

```json
{
  "name": "Research Agent",
  "description": "搜索和总结论文",
  "skills": [{"id": "search", "name": "搜索论文", "inputModes": ["text"]}],
  "url": "https://your-agent.com"
}
```

- **Task 状态机**（核心）：

```
POST /tasks  →  submitted
                  ↓ Agent 开始处理
               working（SSE 流式推送中间进度）
                  ↓ 如需追加信息
           input-required  →  (补充后) → working
                  ↓ 结束
         completed / failed / canceled
```

**PM 工程直觉**：Task 的异步状态机意味着 A2A 天然支持**长时间运行的任务**——主 Agent 发出任务后不需要等待阻塞，子 Agent 完成时主动推送结果。这对复杂多步骤 AI 工作流至关重要。

---

### 运行时协议协商（Agora）的工作原理

**类比**：两个外国人初次见面，先用简单英语协商好怎么沟通（"我们说英文，语速慢一点"），再正式开始对话。

**底层机制**：

核心概念是 **Protocol Document**——一段**自然语言文本**，描述两个 Agent 之间约定好的通信规则。

```
第一阶段：协商（自然语言，消耗 token）
  Agent A → Agent B: "我要传一批新闻文章，你能处理吗？"
  Agent B → Agent A: "可以，用这个格式：title（标题）+ content（正文）+ date（ISO 8601）"

第二阶段：执行（按协商格式，高效传输）
  Agent A → Agent B: [{"title": "...", "content": "...", "date": "2025-06-01"}]
```

**代价所在**：第一阶段协商本身需要 LLM 推理，消耗 token 和时间——这是它与 MCP/A2A（固定 Schema）的效率差距根源（参见 [[Wiki/概念/04_Agent框架/Agent通信三元困境|Agent通信三元困境]]）。

**价值所在**：MCP 和 A2A 都需要工程师提前写好固定的 Schema，而 Agora 让 Agent 自己决定通信格式——这是"Agent 真正自主"的关键一步，但当前 LLM 可靠性尚不足以在生产中依赖。

---

## 9 个协议速览

| 协议 | 发布方 | 连谁 | 成熟度 | PM 关注度 |
|------|-------|------|:------:|:--------:|
| **[[MCP]]** | Anthropic | LLM ↔ 工具/数据 | ✅ | ⭐⭐⭐ 必须了解 |
| **[[A2A]]** | Google | Agent ↔ Agent | 🏭 | ⭐⭐⭐ 值得布局 |
| LAP | LangChain | Agent ↔ Agent | 🔧 | ⭐⭐ 关注 |
| agents.json | Wildcard | LLM ↔ API | 🔧 | ⭐ 了解 |
| Agora | Oxford | Agent ↔ Agent | 🧪 | ⭐ 代表未来方向 |
| ANP | 社区 | Agent ↔ Agent | 🧪 | ⭐ 代表未来方向 |
| LMOS | Eclipse | Agent ↔ Agent | 🔧 | ⭐ 唯一去中心化 |
| ACP | BeeAI/IBM | Agent ↔ Agent | 🔧 | ⭐ 关注 |
| agntcy | 社区 | Agent ↔ Agent | 🔧 | ⭐ 关注 |

---

## 五条 AI PM 核心结论

### 结论 1：MCP 已是行业标配——现在就该在用
MCP 是工具/数据接入的事实标准，Anthropic 出品，Claude、Cursor、VS Code 等都已支持。如果你的 AI 产品需要接入任何外部工具，MCP 是默认选择，不要再手写 function calling 了。

> 📅 **知识时间锚**：MCP 发布 **2024.11**（Anthropic）· 本结论成立自 2024.11 · 论文验证 2025.06 · 核查 2026.07 ✅ 仍成立

→ 详见原子概念页：[[Wiki/概念/04_Agent框架/MCP|MCP]]

### 结论 2：A2A 是 Agent 间通信的新兴标准——值得提前布局
Google 出品，定位为 MCP 的互补协议。**MCP 管"AI 能用什么工具"，A2A 管"AI Agent 怎么找到并委托另一个 Agent"**——两者组合是当前多 Agent 产品架构的主流方向。

> 📅 **知识时间锚**：A2A 发布 **2025.04**（Google）· 本结论成立自 2025.04 · 论文分析 2025.06 · 核查 2026.07 ✅ 仍成立，生态在持续完善

→ 详见原子概念页：[[Wiki/概念/04_Agent框架/A2A|A2A]]

### 结论 3：不会有"赢家通吃"的单一协议——分层协议栈是终局
[[Wiki/概念/04_Agent框架/Agent通信三元困境|Agent通信三元困境]]决定了没有单一协议能同时做到多功能、高效率、强可移植性。类比 OSI 模型，Agent 通信也将走向分层：工具接入层（MCP）+ Agent 协作层（A2A）+ 发现层（待成熟）。

**对 PM 的含义**：不要押注某个协议"赢定了"，要关注你的场景在哪一层、选那一层最成熟的标准。

> 📅 **知识时间锚**：⚠️ 预测性结论，来自本论文分析 **2025.06** · 逻辑基础（三元困境）稳定 · 核查 2026.07 ✅ 预测方向仍成立，MCP+A2A 组合已成主流

→ 详见原子概念页：[[Wiki/概念/04_Agent框架/分层Agent协议栈|分层Agent协议栈]]

### 结论 4：当前最大技术空白是去中心化 Agent 发现——别被"Internet of Agents"愿景迷惑
9 个协议里 8 个靠静态配置或集中注册中心。所有宣称的"AI Agent 自主协作"，背后都有人工提前配置了 Agent 的连接关系。真正的去中心化 Agent 发现（像 DNS 解析网站一样找到任意 Agent）仍是研究空白。

**对 PM 的含义**：产品规划里不要假设 Agent 能"自动发现"未预先配置的服务；"Internet of Agents"是 3-5 年后的愿景，不是现在的现实。

> 📅 **知识时间锚**：研究发现，来自本论文 **2025.06** · 覆盖 9 个协议（截至 2025.07） · 核查 2026.07 ✅ 仍成立，去中心化发现仍无主流实现

→ 详见原子概念页：[[Wiki/概念/04_Agent框架/Agent发现机制|Agent发现机制]]

### 结论 5：运行时协议协商是长期方向——现在不能依赖
Agora 和 ANP 实现的"Agent 之间协商通信格式"代表了真正自主 Agent 网络的方向，但目前是 🔬 研究阶段，不具备产品落地条件。了解这个方向，但现在的产品规划里不要依赖它。

> 📅 **知识时间锚**：研究发现，来自本论文 **2025.06** · Agora 原型 2024，ANP 白皮书 2025 · 核查 2026.07 ✅ 仍为 🔬，无重大产品落地

→ 概念解释：[[Wiki/概念/04_Agent框架/Agent通信三元困境|Agent通信三元困境]]（运行时协商与效率的取舍见此页）

---

## 权威学习资源

> 以下资源来自论文引用 + 各协议官方渠道，按可信度分层。⭐ = 一手权威源，🔧 = 社区工具，🔬 = 研究阶段。

### MCP（优先级最高，现在就该看）

| 资源 | 类型 | 地址 | 说明 |
|------|:----:|------|------|
| MCP 核心架构文档 | ⭐ | [modelcontextprotocol.io/docs/concepts/architecture](https://modelcontextprotocol.io/docs/concepts/architecture) | Anthropic 维护，最权威，覆盖 Transport/Server/Client 设计 |
| MCP 官方 GitHub | ⭐ | [github.com/modelcontextprotocol](https://github.com/modelcontextprotocol) | 官方 SDK（Python / TypeScript）+ 示例 Server |
| Anthropic 发布公告 | ⭐ | [anthropic.com/news/model-context-protocol](https://www.anthropic.com/news/model-context-protocol) | 2024.11 发布公告，解释设计动机 |
| MCP Server 目录 | 🔧 | [smithery.ai](https://smithery.ai) | 社区最大索引，找现成 Server 省开发时间 |

### A2A（布局阶段，了解方向）

| 资源 | 类型 | 地址 | 说明 |
|------|:----:|------|------|
| A2A 核心概念 | ⭐ | [a2aproject.github.io/A2A/latest/topics/key-concepts](https://a2aproject.github.io/A2A/latest/topics/key-concepts/) | Task / Agent Card / 状态机官方解释 |
| A2A 协议规范 | ⭐ | [a2aproject.github.io/A2A/latest/specification](https://a2aproject.github.io/A2A/latest/specification/) | JSON Schema 定义，工程师查询用 |
| A2A 与 MCP 的关系 | ⭐ | [a2aproject.github.io/A2A/latest/topics/a2a-and-mcp](https://a2aproject.github.io/A2A/latest/topics/a2a-and-mcp/) | Google 官方解释两者分工，PM 必读 |

### 学术参考（理解背景、判断趋势）

| 论文 | 发表时间 | arXiv | 价值 |
|------|:--------:|-------|------|
| 本论文（Taxonomy，TU München）| 2025.06 | 2506.19153 | 最完整的协议分类框架，覆盖 9 个协议 |
| A Survey of AI Agent Protocols（Yang et al.）| 2025.04 | 2504.16736 | 互补视角的协议综述，与本论文对照读 |
| ANP Technical White Paper | 2025 | 2508.00007 | 去中心化 Agent 网络最完整方案，了解未来方向 |
| Multi-agent Collaboration Mechanisms（Tran et al.）| 2025.01 | 2501.06322 | MAS 整体协作机制综述，理解 Agent 通信的背景 |

### Agora / ANP（了解未来方向，不需要现在深入）

| 资源 | 类型 | 地址 |
|------|:----:|------|
| Agora Protocol 规范 | 🔬 | [agoraprotocol.org/docs/protocol/specification](https://agoraprotocol.org/docs/protocol/specification) |
| ANP 官方 | 🔬 | [agent-network-protocol.com](https://agent-network-protocol.com) |

---

## 论文精读卡片

**一句话**：TU München 对 9 个真实落地的 Agent 通信协议建立首个五维分类框架，发现"不会有赢家通吃的单一协议"——三元困境（功能/效率/可移植性）使分层协议栈成为必然终局，MCP（工具层）+ A2A（Agent 协作层）组合已是主流架构。

**问题**：Agent 通信协议生态碎片化（MCP/A2A/LAP/Agora 等），PM 和工程师选型时没有统一框架；产品规划中过度依赖"去中心化 Agent 自主发现"等尚未成立的假设，造成架构错误。

**核心方法**：
- 五维分类框架：Counterparty（连谁）/ Payload（传什么）/ Interaction State（有无状态）/ Discovery（发现机制）/ Schema Flexibility（格式灵活性）
- 覆盖 9 个协议（MCP/A2A/LAP/agents.json/Agora/ANP/LMOS/ACP/agntcy），按 GitHub stars + 维护状态筛选
- 三元困境（Agent 通信三元困境）：功能完整性、传输效率、格式可移植性三者无法同时最优，决定了协议分层的必然性

**关键图/公式**：Agent 通信三元困境是全文最核心的洞察：任何协议都无法同时做到多功能（处理复杂 Agent-to-Agent 对话）+ 高效率（低 token 开销）+ 强可移植性（任意格式）。Agora 用自然语言协商格式实现了高可移植性，但协商本身消耗大量 token，效率差；MCP 格式固定效率高，但扩展性差——这个三角形解释了为什么协议必须分层。

**实验设置**：
- 规模/数据：9 个协议的文档 + GitHub 分析；覆盖 2024.11-2025.06 发布的主要 Agent 通信协议
- 对比：各协议在五维度上的定性对比矩阵

**最强证据**：规律性发现——所有 Agent-to-Agent 协议都有 Session State + Hybrid Payload（必须支持多轮、多格式），而工具接入协议（MCP）无需状态。这个规律性区分有力支撑了"分层协议栈"的结论，且在 9 个协议中无一例外。

**最弱证据**：9 个协议的筛选基于 GitHub stars 和维护状态，可能遗漏了企业级私有协议（如大厂内部 Agent 通信方案）；分类是定性的，没有性能测试数据；"真正的去中心化 Agent 发现仍是空白"的结论基于 2025.06 的快照，可能已过时。

**可复用点**：MCP + A2A 双层架构决策：凡涉及"AI 调用工具/数据" → 用 MCP；凡涉及"AI Agent 委托另一个 Agent" → 用 A2A。不要为同一问题创建自定义协议。MCP Server 的最简实现：给现有系统包一层 JSON-RPC 外壳，无需改原系统。

**和哪些论文相关**：
- [[Wiki/论文笔记/03_Agent系统设计/01_Sakana-Fugu技术报告]] — Fugu 的 Worker Pool 通信是 MCP+A2A 组合的典型场景
- [[Wiki/概念/04_Agent框架/MCP]] — Model Context Protocol 详解
- [[Wiki/概念/04_Agent框架/A2A]] — A2A Agent 通信协议详解
- [[Wiki/概念/04_Agent框架/Agent通信三元困境]] — 为什么无法有一个协议统一一切
- [[Wiki/概念/04_Agent框架/分层Agent协议栈]] — MCP+A2A 分层架构

**我能拿它做什么**：
- 在产品架构评审中，用五维框架快速评估工程方案选择的协议是否合适
- 在路线图规划中，将 MCP 集成列为 P0（现在），A2A 布局列为 P1（下季度），去中心化发现不纳入 2 年内计划
- 向业务方解释为什么"AI Agent 无法自动发现任意服务"：去中心化 Agent 发现是技术空白，背后的"自动协作"需要提前人工配置

**3天后要回忆的问题**：
1. 论文的五维分类框架是哪五个维度？
2. 为什么所有 A2A 协议都需要 Session State + Hybrid Payload？
3. Agent 通信三元困境的三个角分别是什么？
4. MCP 和 A2A 的分工是什么？（各管哪一层）
5. 当前 9 个协议中，唯一支持去中心化发现的是哪个？

## 原子概念索引

| 概念 | 文件 | 成熟度 | 优先级 |
|------|------|:------:|:-----:|
| MCP（模型上下文协议） | [[MCP]] | ✅ | 必读 |
| A2A（智能体间通信协议） | [[A2A]] | 🏭 | 必读 |
| Agent 通信三元困境 | [[Agent通信三元困境]] | ✅ | 选型决策框架 |
| Agent 发现机制 | [[Agent发现机制]] | ⚠️ | 理解技术现实 |
| 分层 Agent 协议栈 | [[分层Agent协议栈]] | ⚠️ | 架构趋势 |

---

## 这篇论文对 PM 的局限

- 论文 2025 年 6 月发表，AI 领域变化极快，协议成熟度在持续更新
- 论文是学术分类，实际产品选型还需要结合工程师对各协议 SDK 成熟度的评估
- 9 个协议的选样基于 GitHub stars 和维护状态，可能遗漏了部分企业级私有协议

> **时效状态**: ⚠️ 框架有效，但具体协议生态需要每 3-6 个月关注更新
> **最后核查**: 2026-07
