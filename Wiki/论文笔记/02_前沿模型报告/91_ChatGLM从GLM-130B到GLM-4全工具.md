---
tags: [论文笔记, ChatGLM, GLM-4, 工具调用, 中文大模型, ZhipuAI]
笔记层级: 参考
paper_id: "91"
filename: "91 - ChatGLM - A Family of Large Language Models from GLM-130B to GLM-4 All Tools.pdf"
authors: "Team GLM (ZhipuAI & Tsinghua)"
year: 2024
成熟度: ✅
---

# ChatGLM：从 GLM-130B 到 GLM-4 全工具的大语言模型家族

📄 **原文 PDF**：[[RAW/91 - ChatGLM - A Family of Large Language Models from GLM-130B to GLM-4 All Tools.pdf]]

## PM 速判（30秒）

> **国内最早大规模落地的中英文 LLM 家族，GLM-4 对标 GPT-4 并在中文评测超越。** ZhipuAI/清华 2024 年发布 ChatGLM 系列综述，重点介绍 GLM-4 系列（GLM-4、GLM-4-Air、GLM-4-9B）：10T tokens 预训练（主要中英文），多阶段对齐；GLM-4 在 MMLU/MATH/GSM8K 等指标紧追 GPT-4，中文对齐测评（AlignBench）超越 GPT-4；GLM-4 All Tools 集成网页浏览/Python 解释器/文生图/自定义工具，媲美 GPT-4 All Tools。

| 年份 | 成熟度 | PM 相关性 |
|------|--------|---------|
| 2024 | ✅ | **高**（理解国内 LLM 生态发展，中文 AI 产品基础参考）|

## 核心机制

```
GLM-4 系列：
  GLM-4: 旗舰推理/对话模型
  GLM-4-Air: 轻量高效版
  GLM-4-9B: 开源小模型（128K/1M 上下文版本）
  GLM-4V-9B: 视觉版本
  
预训练：
  ~10T tokens（中英文为主 + 24 种语言）
  
后训练（多阶段）：
  ① SFT（监督微调）
  ② RLHF（人类反馈强化学习）
  多阶段对齐 → 高质量中文对齐
  
GLM-4 All Tools（Agent 模式）：
  网页浏览器 → 在线信息获取
  Python 解释器 → 数学/代码执行
  文生图模型 → 多模态生成
  用户自定义函数 → 可扩展工具调用
  
开源贡献：
  ChatGLM-6B（三代）
  GLM-4-9B（128K, 1M）
  WebGLM, CodeGeeX
  2023 年 HuggingFace 下载量超 1000 万次
```

## 关键数据 📅 2024

| 基准 | GLM-4 | GPT-4 | GPT-4-Turbo |
|------|-------|-------|-------------|
| MMLU | ~86% | 86% | 87% |
| GSM8K | 竞争 | 竞争 | — |
| AlignBench（中文）| **超越 GPT-4** | — | — |
| IFEval | 接近 GPT-4-Turbo | — | ✅ |
| 长文本（128K）| 媲美 Claude 3 | — | 媲美 |

## PM 结论

- 📅 2024 年，ChatGLM 是中文大模型中最广泛部署的系列（10M+ 下载，智谱 API 商业化）
- **中文对齐优势**是核心差异化：AlignBench 超越 GPT-4——国内企业对话产品首选
- GLM-4-9B 是开源可部署的首选小模型（128K 上下文，本地推理友好）
- All Tools 模式 = 中国版 ChatGPT Plugins，集成度高，适合做垂直 AI 助手产品

## 论文精读卡片

**一句话**：GLM-4 在 AlignBench 中文对齐评测上超越 GPT-4，HumanEval 在 10T token 训练后达到竞争水平，GLM-4-9B 开源支持 128K/1M 上下文——ChatGLM 是国内下载量最高（HuggingFace 1000 万次+）的中文 LLM 系列。

**问题**：如何在中英双语场景下训练一个在中文理解/对话质量上超越 GPT-4、同时具备完整 Agent 工具调用能力的商业级 LLM，并在国内开源生态中建立统治地位？

**核心方法**：
- 大规模双语预训练——约 10T tokens（中英文为主，含 24 种语言），比 GPT-3.5 时代训练量提升约 3-5x
- 多阶段对齐——SFT（监督微调）+ RLHF（人类反馈强化学习）两阶段，中文对齐数据比例高于主流英文模型
- GLM-4 All Tools——集成四类工具：网页浏览器/Python 解释器/文生图/用户自定义函数，形成完整 Agent 闭环

**关键图/公式**：GLM 系列演进：GLM-130B（2022）→ ChatGLM-6B（2023）→ ChatGLM2/3 → GLM-4/4-9B（2024），核心路线是规模扩张 + 中文对齐精细化 + 工具调用能力注入。

**实验设置**：
- 规模/数据：GLM-4（旗舰，参数未公开），GLM-4-Air（高效），GLM-4-9B（开源，128K/1M 上下文），~10T tokens 预训练
- 对比：GPT-4、GPT-4 Turbo、Claude 3 在 MMLU/GSM8K/AlignBench/IFEval/长文本任务

**最强证据**：AlignBench（中文综合对齐）超越 GPT-4——这是中文模型首次在系统性中文评测上超越 GPT-4；MMLU ~86%（与 GPT-4 并列）；GLM-4-9B 128K 上下文性能媲美 Claude 3；HuggingFace 下载量超 1000 万次（2023 年统计）。

**最弱证据**：MMLU 并列而非超越（GPT-4 也是 86%）；评测主要集中在中文，英文推理能力相比 GPT-4 Turbo 有差距；参数量和架构细节同样不完全公开，复现困难；All Tools 性能仅声称"媲美 GPT-4 All Tools"，无统一基准数字。

**可复用点**：All Tools = 四类工具的最小闭环集合（搜索+代码执行+生成+自定义），这是构建通用 AI 助手的最小工具集定义，可直接作为 Agent 产品 MVP 的工具选择框架。

**和哪些论文相关**：
- [[Wiki/论文笔记/02_前沿模型报告/89_GLM-5从氛围编码到Agent工程]] — GLM-4 是 GLM-5 的前置版本，能力路线的直接传承
- [[Wiki/论文笔记/06_训练对齐RL/79_RLAIF对比RLHF用AI反馈扩展强化学习]] — ChatGLM 使用 RLHF 对齐，RLAIF 是其潜在替代方案
- [[Wiki/概念/02_训练方法/RLHF与RLAIF]] — ChatGLM 多阶段对齐的技术基础

**我能拿它做什么**：
- 国内中文对话产品选型：GLM-4 是中文对齐最强商业 API，AlignBench 超过 GPT-4
- GLM-4-9B 开源版是国内私有化部署的标准小模型选择（128K 上下文，轻量推理）
- All Tools 的四类工具集定义可直接用作自己 Agent 产品工具层的设计蓝图

**3天后要回忆的问题**：
1. GLM-4 在 AlignBench 和 MMLU 上与 GPT-4 的具体对比结果？
2. GLM-4 All Tools 集成了哪四类工具？
3. ChatGLM 系列从 GLM-130B 到 GLM-4 经历了哪些主要演进节点？
4. GLM-4-9B 开源版支持的最大上下文长度是多少？
5. ChatGLM 在 HuggingFace 的下载量数字说明了什么？

## 原子概念索引
- [[Wiki/概念/02_训练方法/RLHF与RLAIF]] — ChatGLM 多阶段对齐（SFT+RLHF）是该技术的典型工业应用
- [[Wiki/概念/04_Agent框架/ReAct与Agent推理行动框架]] — GLM-4 All Tools 是 Agent 工具调用框架的产品化实现
