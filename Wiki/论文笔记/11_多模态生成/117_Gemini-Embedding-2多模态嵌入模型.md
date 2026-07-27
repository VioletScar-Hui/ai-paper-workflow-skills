---
tags: [论文笔记, 嵌入模型, 多模态, 向量检索, RAG, Google]
笔记层级: 参考
paper_id: "117"
filename: "117 - Gemini Embedding 2 - A Native Multimodal Embedding Model from Gemini.pdf"
authors: "Madhuri Shanbhogue, Zhe Li, Shanfeng Zhang et al. (Google Gemini Embedding Team)"
year: 2026
arxiv: "2026.05.27"
成熟度: 🏭
---

# Gemini Embedding 2：原生多模态嵌入模型

📄 **原文 PDF**：[[RAW/117 - Gemini Embedding 2 - A Native Multimodal Embedding Model from Gemini.pdf]]

## PM 速判（3 分钟）

> **这篇论文给 PM 的一句话**：Google 发布的 Gemini Embedding 2 是目前最强的通用多模态嵌入模型——把视频/音频/图片/文本**统一**映射到同一向量空间，不需要先转成文本，直接支持"拿着视频片段搜文档"这类跨模态检索；整体 Benchmark 得分 77.2，领先 Amazon Nova MME（68.2）和 Voyage-3.5（70.0）。

| 项目 | 评估 |
|------|------|
| **机构** | Google（Gemini Embedding 团队） |
| **发布时间** | 2026.05.27 |
| **部署状态** | 🏭 可通过 Google Vertex AI API 调用 |
| **支持模态** | Text / Image / Audio / Video（及任意组合） |
| **向量维度** | 最高 3072 维 |

---

## 核心能力：原生多模态 vs. 传统方案

```
传统方案（CLIP/ALIGN 式）：
  Text ─────────────────→ [Encoder A] → 向量
  Image ────────────────→ [Encoder B] → 向量
  
  局限：双塔分开，跨模态只能"文本↔图片"，
        无法处理"图片+文本 → 视频片段"的混合查询

Gemini Embedding 2（原生多模态）：
  Text ┐
  Image ┼──→ [Gemini backbone + 双向注意力] → 统一向量
  Audio ┤   (最高 3072 维)
  Video ┘
  
  优势：交错输入（如"文字+图片作为一个查询"）→ 直接检索视频片段
        音频直接 embed（无需 ASR 转录）
```

---

## 技术架构（工程解释）

```
训练流程（3步）：

1. Pre-Fine-Tuning (PFT)：
   用大规模噪声 query-target 对微调 Gemini backbone
   → 把自回归生成能力转换成编码能力
   → 阶段1只用 text/image/code 任务

2. Fine-Tuning (FT)：
   用 text+code+document+image+audio+video 全模态任务微调
   → 加入 Hard Negative（困难负样本）提高检索精度
   
3. Model Souping（消融实验发现）：
   将基础模型 + 特定域微调模型按比例混合参数
   → 在保持通用性的同时提升垂类性能（e.g., 视频检索）

损失函数：NCE（噪声对比估计）+ In-Batch Negatives
输出：Mean Pooling → Linear Projection → 目标维度向量
```

---

## Benchmark 对比

### 多模态检索

| 任务 | Gemini Embedding 2 | Amazon Nova MME | Voyage-3.5 |
|------|-------------------|----------------|------------|
| Image→Image（GUIEC R@1） | **79.4** | 68.6 | 69.4 |
| Text→Image（MSCOCO R@1） | **62.9** | 57.2 | 58.1 |
| Text→Video（Vatex NDCG@10） | **68.8** | 60.3 | 55.2 |
| Document Retrieval（NDCG@10） | **64.9** | 60.6 | 65.5 |
| **整体均值** | **77.2** | 68.2 | 70.0 |

### 文本嵌入（MTEB）

| 任务 | Gemini Embedding 2 | Gemini Embedding V1 | Voyage-3.5 |
|------|-------------------|---------------------|------------|
| MTEB Multilingual | **69.9** | 68.4 | 58.5 |
| MTEB Code | **84.0** | 76.0 | – |

### 音频检索（MSEB，原生 vs. 先转文字）

```
Gemini Embedding 2 + ASR转录:  mrr@10 = 68.98
Gemini Embedding 2 + 原生音频: mrr@10 = 73.99（+5.01pp）

跨语言检索差异更显著：
  ASR 版: 67.55
  原生音频版: 72.56（+5.01pp）

→ 原生音频保留了语调/重音/发音模糊性等信息，比文字转录更准
```

---

## 产品场景应用

| 场景 | 用法 | 价值 |
|------|------|------|
| **多模态 RAG** | 文档库包含 PDF+视频+图片，query 可以是混合模态 | 不需要把所有内容转换成文字 |
| **视频检索** | "用这张截图找相关视频片段" | 视频推荐、素材搜索 |
| **音频搜索** | 直接用语音查找文档（多语言场景） | 无需 ASR，跨语言检索更精准 |
| **跨模态推荐** | 用户浏览的图片 → 推荐相关视频 | 电商/娱乐平台 |
| **专业域检索** | 医学图像（显微镜）/科学图（天文）搜索 | R@5 在显微/天文领域接近翻倍 |

---

## 与竞品关键差异

| 维度 | Gemini Embedding 2 | Amazon Nova MME | Voyage-3.5-multimodal |
|------|-------------------|----------------|----------------------|
| 模态支持 | V+A+I+T | V+A+I+T | V+I+T（无音频） |
| 交错输入 | ✅ 任意组合 | ❌ 需分开 | ❌ 需分开 |
| 原生音频 | ✅（无需ASR） | 未说明 | 不支持 |
| API可用性 | Google Vertex AI | Amazon Bedrock | Voyage API |
| 开源 | ❌ | ❌ | ❌ |

---

## 核心结论（带时间锚）

1. **多模态 RAG 从"先转文字"进入"原生多模态"时代** 📅 2026.05
   Gemini Embedding 2 证明：视频/音频可以不经 ASR/Caption 直接 embed，降低了构建多模态检索系统的工程复杂度，RAG 基础设施开始向模态无关的统一向量库方向发展。

2. **交错输入（Interleaved Input）是关键差异** 📅 2026.05
   支持"图片+文字"作为一个查询去检索视频——这是传统双塔模型无法做到的，开启了新的检索范式（如：用截图+描述找视频片段）。

3. **通用 vs. 特化：Model Souping 是折中方案** 📅 2026.05
   纯微调垂类任务会损害通用性，Model Souping（混合基础模型权重）是在不损失通用性的前提下提升特定场景的实用策略，可借鉴于自家模型部署。

---

## 工程部署注意

- **API 调用**：Vertex AI / Google API（非开源）
- **向量维度**：可配置到 3072，RAG 场景建议先测 768 是否足够
- **成本**：多模态 embed 的 token 消耗（视频帧数）需评估
- **竞品选择**：若需要音频或交错输入 → Gemini Embedding 2；若只需图文 → Voyage-3.5-multimodal 也可

---

## 权威学习资源

- 📄 论文：Google，2026.05.27
- 🔗 API：https://cloud.google.com/vertex-ai/docs/generative-ai/embeddings/get-multimodal-embeddings
- 🔗 参考：CLIP (Radford et al., 2021) — 多模态嵌入的原始双塔方案
- 🔗 参考：MTEB/MMTEB — 嵌入模型评测基准
- 🔗 相关：Amazon Nova MME — 直接竞品

## 论文精读卡片

**一句话**：Gemini Embedding 2 将视频/音频/图片/文本统一映射到同一向量空间（3072 维），整体 Benchmark 均值 77.2，领先 Amazon Nova MME（68.2）和 Voyage-3.5（70.0），原生音频检索比先 ASR 转录再检索高 5.01pp。

**问题**：传统双塔嵌入模型（如 CLIP）只能处理"文本↔图片"的对齐，无法支持"图片+文字"作为混合查询去检索视频，也无法直接处理音频（须先 ASR 转录），限制了多模态 RAG 系统的能力上限。

**核心方法**：
- 原生多模态架构：基于 Gemini backbone + 双向注意力，支持文本/图片/音频/视频交错输入，统一映射到同一向量空间
- 三步训练流程：Pre-Fine-Tuning（大规模噪声数据）→ Fine-Tuning（全模态+困难负样本）→ Model Souping（混合基础模型和垂类微调模型权重）
- 损失函数：NCE（噪声对比估计）+ In-Batch Negatives；输出：Mean Pooling → Linear Projection

**关键图/公式**：`传统双塔：Text→Encoder A→向量，Image→Encoder B→向量（无法交叉）；Gemini Embedding 2：[Text/Image/Audio/Video]→Gemini backbone+双向注意力→统一向量`。交错输入是核心差异。

**实验设置**：
- 规模/数据：MTEB Multilingual（文本）、MTEB Code、GUIEC（图图检索）、MSCOCO（文图检索）、Vatex（文视频检索）、MSEB（音频检索）等多套 Benchmark
- 对比：Amazon Nova MME、Voyage-3.5-multimodal（主要竞品）；Gemini Embedding V1（上代对比）

**最强证据**：原生音频 mrr@10 = 73.99 vs. 先 ASR 转录再检索 68.98（+5.01pp），证明"原生保留语调/重音信息"有实测价值；视频检索 NDCG@10 = 68.8 vs. Nova 60.3（+8.5pp）。

**最弱证据**：所有 Benchmark 均由 Google 选择，未包含可能对竞品更有利的评测；Model Souping 的混合比例选择方法未详细说明，难以独立复现；API 为闭源商业服务，无法外部验证模型架构。

**可复用点**：Model Souping 策略——在不损失通用性的前提下提升特定场景性能，将基础模型权重和垂类微调模型权重按比例混合，可借鉴于任何需要兼顾通用性和特化性能的模型部署场景。

**和哪些论文相关**：
- [[Wiki/论文笔记/02_前沿模型报告/118_MedGemma-1.5医学AI基础模型]] — 同期 Google 医疗多模态模型，多模态处理技术路线可对比
- [[Wiki/概念/01_架构技术/原生多模态嵌入]] — 统一向量空间的多模态 embed（vs. 双塔方案）
- [[Wiki/概念/05_记忆与检索/RAG检索增强生成]] — 多模态 RAG 的基础设施升级

**我能拿它做什么**：
- 构建包含 PDF+视频+图片+音频的多模态文档库时，Gemini Embedding 2 是当前最强的统一 embed 选项
- 音频检索场景（多语言），选择原生音频 embed 而非先 ASR 转录，可获得 5pp 以上精度提升
- 在内部多模态 RAG 评测时，用 GUIEC/Vatex/MSEB 等真实 Benchmark 衡量实际性能，不只看 MTEB

**3天后要回忆的问题**：
1. Gemini Embedding 2 与传统双塔嵌入模型（CLIP）的最大架构区别是什么？
2. 原生音频检索比先 ASR 转录再检索高多少？（mrr@10 高 5.01pp）
3. 整体 Benchmark 均值与主要竞品相比如何？（77.2 vs. Nova 68.2 vs. Voyage 70.0）
4. Model Souping 是什么，解决了什么问题？
5. Gemini Embedding 2 和竞品在模态支持上的关键差异是什么？（支持交错输入和原生音频）

## 原子概念索引

- 新概念：[[Wiki/概念/01_架构技术/原生多模态嵌入]] — 统一向量空间的多模态 embed（vs. 双塔方案）
- [[Wiki/概念/05_记忆与检索/嵌入模型]] — 本论文统一文本/图像/音频/视频至同一向量空间（3072 维），是嵌入模型概念在原生多模态方向的最新前沿
