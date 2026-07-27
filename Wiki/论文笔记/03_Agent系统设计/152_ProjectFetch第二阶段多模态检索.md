---
tags: [论文笔记, 多模态Agent, 网页检索, 视觉理解, RAG, 信息提取, ProjectFetch]
paper_id: "152"
笔记层级: 骨干
复核日期: 2026-07-04
---

# Project Fetch Phase 2：多模态Agent网页检索

📄 **原文 PDF**：[[RAW/152 - Project Fetch - Phase two.pdf]]

## PM 速判 > 一句话

纯文本Agent在网页检索中丢失截图/表格/图表中的视觉信息，Phase 2通过双通道（HTML文本+全页截图）VLM融合架构，将视觉信息获取准确率提升18-34%。

## 双层费曼 🗣️

**给CEO：**
网页上大量关键信息藏在图片、表格和图表里——财报数据、产品参数、科学图表。普通AI只能读文字，抓不到这些东西。Phase 2让AI不仅能读网页文字，还能"看"网页截图，像人一样把文字和图片的信息合起来理解。实测在金融数据场景下准确率提升34%。

**给工程师：**
双通道架构：对每个目标页面同时提取HTML→Markdown文本流 + 截取全页截图作为图像流。VLM对截图做视觉元素定位（表格边界识别、图表类型分类、数值OCR），两路token在同一LLM context中拼接做跨模态注意力融合。关键工程细节：截图质量（viewport尺寸、懒加载时机）直接影响表格OCR精度；token消耗约2-3倍于纯文本；迭代检索策略让Agent在信息不足时自主翻页/点击。

## 问题域定位 🎯

- **宏观问题**：信息检索（Information Retrieval）中非文本模态被系统性忽略
- **具体问题**：网页环境下的多模态信息提取与融合推理
- **边界条件**：仅关注网页截图（含表格、图表、UI元素），不支持视频/音频；不涉及网页操控（点击/表单填写）本身，仅关注信息提取
- **失败案例**：纯文本Agent查询"Q2财报毛利率趋势"，HTML仅包含"详见下图"而无截图中折线图的数值，返回空或错误答案
- **对抗场景**：大量使用图片展示数据的网站（金融数据平台、科学论文数据库、电商产品参数页）

## 核心机制

```
┌────────────────────────────────────────────────┐
│                 用户查询 Q                       │
└────────────┬───────────────────────────────────┘
             ↓
┌──────────────────────────────┐
│     Agent 检索规划器          │
│  （判断需要哪些页面/动作）     │
└────────────┬──────────────────┘
             ↓
┌──────────────────────────────┐
│        目标网页获取            │
└──────┬──────────────┬────────┘
       ↓              ↓
┌──────────┐   ┌──────────────┐
│ 通道1    │   │ 通道2         │
│ HTML解析  │   │ 全页截图      │
│ →Markdown│   │ →VLM 视觉     │
│ 文本流   │   │ 元素定位       │
└────┬─────┘   │ - 表格边界识别  │
     │         │ - 图表类型分类  │
     │         │ - 数值OCR      │
     │         │ - UI元素标注    │
     │         └──────┬─────────┘
     ↓                ↓
┌──────────────────────────────┐
│  跨模态融合（LLM Context）     │
│  文本token + 视觉patch token  │
│  → 联合注意力计算             │
│  → 跨模态推理                 │
└────────────┬──────────────────┘
             ↓
      ┌──────┴──────┐
      │ 信息充足？    │
      ├── 是 ────→ 结构化答案
      └── 否 ────→ 迭代检索（翻页/点击/新搜索词）
```

## 设计决策解剖 ⚖️

| 决策点 | 选择 | 被放弃方案 | 为什么 | 何时失效 |
|--------|------|-----------|--------|---------|
| 截图时机 | 全页渲染后一次性截图 | 逐元素截图/流式截图 | 平衡工程复杂度与覆盖度；DOM加载完成即可截 | 动态加载内容（无限滚动、延迟渲染图表）可能漏截 |
| 视觉编码方式 | VLM端到端（如GPT-4V） | 独立OCR引擎+图表解析器pipeline | 端到端减少工程耦合，VLM利用预训练视觉知识 | VLM对密集表格OCR精确度低于专用OCR引擎 |
| 文本通道格式 | HTML→Markdown降级 | 保留完整DOM Tree | Markdown减少token，保留关键语义结构 | 丢失CSS布局信息（如表格行列视觉对齐） |
| 迭代策略 | 规则+LLM判断混合 | 纯LLM自驱动/纯规则 | LLM判断何时信息不足，规则约束动作空间防止偏差 | LLM"幻觉"出信息充足结论时，迭代提前终止 |
| token预算控制 | 截图压缩至适配合力窗口 | 高分辨率截图+滑动窗口 | 避免上下文窗口溢出，控制成本 | 压缩后小字体/密集图表细节丢失 |

## 成本与量级 💰

- **Token消耗**：每个页面截图约占用2-3倍于纯文本的token量（基于GPT-4V pricing：图像token ~85-170 per 512x512 tile）
- **延迟**：截图获取+传输 ~0.5-2s；VLM推理 ~2-5s per page；迭代检索累加
- **API成本**：假设每查询访问3页，每页截图+文本 ~3000 token纯文本 + ~6000 token图像 → ~9000 token/页，×3 = ~27K token/查询，GPT-4级别~ $0.30-0.60/查询
- **规模瓶颈**：大规模索引场景（百万级页面）下截图存储量级为TB级，需预计算索引缓存
- **与纯文本对比**：精度提升18-34%，成本增加200-300%，ROI取决于任务对视觉信息敏感度

## 证据审计 🔬

**最强证据：**
- FinFetch金融数据基准上提升34%——这是真实场景的高难度任务（图表数值提取），效果差距显著
- VisualWebArena提升18%——独立公开基准，可复现可验证

**最可疑：**
- 迭代检索策略的22%提升是否来自更长的推理链而非多模态本身？应看消融实验：迭代策略×单模态 vs 迭代×多模态
- 截图时机对动态页面（地图、实时图表）的影响未量化

**审稿人可能追问：**
- 基线中SeeAct也使用视觉，为何Phase 2更好？差异在哪——视觉分辨率？prompt工程？融合方式？
- 截图质量对精度的敏感性分析（viewport大小、DPI、懒加载时间）

**最小复现实验：**
拿10个含表格/图表的网站，对比纯文本GPT-4 vs GPT-4V+截图：统计在多模态增强前后能正确回答的比例差。

## 可复用点 + 供应商拷问清单

**可复用组件：**
- 双通道网页抓取pipeline：Playwright截图 + BeautifulSoup Markdown解析，可独立复用
- VLM视觉元素定位prompt模板（"找出截图中所有数据表，提取其中数值并标注行列表头"）
- 迭代检索终止条件判断prompt模板

**供应商拷问清单：**
1. 你们的VLM在密集表格OCR上精确度如何？（基准测试：表格单元格级别，非行级别）
2. 截图压缩到多少分辨率？小字体图表能识别吗？
3. 动态加载/懒加载页面如何保证截图完整性？
4. 双通道方案token消耗预算是多少？能否控制在总token预算内？
5. 多模态结果的置信度如何判断？能否输出"来自文本/来自视觉/不确定性高"的归因信息？

## 关联网络 🕸️

- *145_ProjectFetch第一阶段文本检索*（未收录）
- *148_MultiModalRAG多模态检索增强生成*（未收录）
- [[Wiki/概念/05_记忆与检索/RAG检索增强生成]] — 多模态检索（MMR）是标准 RAG 的文本+视觉双通道扩展，见该概念页"多模态检索：文本+视觉双通道"小节
- [[Wiki/概念/01_架构技术/原生多模态嵌入]] — 视觉语言模型（VLM）在本文中承担截图的视觉元素定位/OCR"精理解"角色，与原生多模态嵌入的"粗召回"角色形成分工，见该概念页"VLM 视角"小节

**冲突/印证：**
- **印证** SeeAct（Zheng 2024）同样使用截图作为视觉输入，但Phase 2在信息提取而非网页操控，印证"截图+文本双通道是视觉网页Agent的基础模式"
- **冲突** 纯文本RAG范式认为"HTML语义结构足以提取网页信息"，Phase 2结果表明该假设在含图表现代网页上不成立——RAG pipeline应考虑混合模态

## 动手练习 💻

```python
# 多模态网页信息提取模拟器
# 模拟：对含文本+表格+图表的网页截图进行多模态提取
# 用Python模拟伪VLM输出，对比双通道 vs 纯文本效果

import json
import random

# 模拟网页数据：文本段落、表格、图表
mock_page = {
    "title": "ACME Corp Q2 2026 Earnings",
    "text_paragraphs": [
        "ACME Corp reported strong Q2 results with revenue up 15% year-over-year.",
        "The growth was driven by enterprise segment expansion.",
        "For detailed financial breakdown, see the chart below."
    ],
    "table": {
        "headers": ["Metric", "Q2 2025", "Q2 2026", "YoY Change"],
        "rows": [
            ["Revenue ($M)", "850", "978", "+15.1%"],
            ["Gross Margin", "62.3%", "64.1%", "+1.8pp"],
            ["Operating Income ($M)", "210", "255", "+21.4%"],
            ["EPS ($)", "2.15", "2.48", "+15.3%"]
        ]
    },
    "chart_data": {
        "chart_type": "bar_chart",
        "title": "Revenue by Segment (Q2 2026)",
        "series": [
            {"segment": "Enterprise", "revenue_m": 520},
            {"segment": "SMB", "revenue_m": 280},
            {"segment": "Consumer", "revenue_m": 178}
        ]
    }
}

# 模拟VLM视觉元素定位输出
def simulate_vlm_extraction(screenshot_desc):
    """接收截图描述，返回VLM识别的表格和图表内容"""
    if "table" in screenshot_desc or "chart" in screenshot_desc:
        # VLM从图像中提取的表格数值（模拟，可能有OCR误差）
        extracted_table = mock_page["table"]["rows"][:2]  # 只提取到前2行
        extracted_chart = mock_page["chart_data"]["series"][:2]
        return {"table": extracted_table, "chart": extracted_chart}
    return {"table": [], "chart": []}

def text_only_retrieve(page):
    """纯文本通道：仅从HTML/Markdown文本中提取信息"""
    # 模拟：纯文本可能丢失表格/图表数值
    result = {"summary": page["text_paragraphs"]}
    # 纯文本Agent可能错失表格中的具体数值
    # 因为它看到的markdown表格可能渲染成"see chart below"
    return result

def multimodal_retrieve(page):
    """双通道：文本+截图VLM融合"""
    text_result = text_only_retrieve(page)
    # VLM从截图提取表格和图表
    vlm_result = simulate_vlm_extraction("page contains table and chart")
    
    # 融合：将VLM提取的结构化信息加入结果
    fused = {
        "summary": text_result["summary"],
        "table_data": vlm_result["table"],
        "chart_data": vlm_result["chart"],
        "fusion_note": "Table rows extracted: Q2 2025/Q2 2026 comparison; "
                       "Chart series: Enterprise=$520M, SMB=$280M"
    }
    return fused

# 模拟查询
query = "What was ACME Corp revenue in Q2 2026?"

# 纯文本结果
text_result = text_only_retrieve(mock_page)
print("=== 纯文本Agent结果 ===")
# 纯文本：只有"revenue up 15%"这样的文字摘要，没有精确值
# 所以会返回近似或模糊答案
print(f"Query: {query}")
# 模拟失败：纯文本可以回答"increased 15%"，但不能回答精确数值
print(f"Response: Revenue increased year-over-year (from text: 'up 15%')")
print(f"Can extract exact $ value? {'Yes' if '978' in str(text_result) else 'NO — table not found'}")

print()

# 多模态结果
multi_result = multimodal_retrieve(mock_page)
print("=== 多模态双通道结果 ===")
print(f"Query: {query}")
print(f"Response: ACME Corp Q2 2026 revenue was $978M")
print(f"(Text: Growth 15% → Figure: Q2 2025=$850M, Q2 2026=$978M)")
print(f"Tables extracted: {multi_result['table_data']}")
```

## 自测三层 🎓

**L1 — 概念回忆：**
Q: Phase 2的双通道是哪两路？为什么需要两路？
A: HTML→Markdown文本流 + 全页截图视觉流。因为网页中大量关键信息以图像形式存在（表格、图表），单一文本通道会系统性丢失。

**L2 — 迁移应用：**
Q: 如果要在PDF文档上实现类似的"文本+截图"双通道检索，需要做哪些调整？
A: PDF面临的问题：1）没有HTML结构，无法直接文本解析，需OCR或PDF解析库（如PyMuPDF）；2）PDF多页，需逐页截图；3）PDF中表格定位难度更高（缺乏HTML的table标签），需表格检测模型（如CascadeTabNet）；4）PDF书签可辅助确定页面组织。工程架构类似，但文本通道改为PDF文本提取，截图通道改为逐页渲染截图。

**L3 — 批判性质疑：**
Q: Phase 2的双通道架构是否一定优于"仅用VLM处理截图"的单通道方案？在什么条件下单通道反而更好？
A: 不一定。如果VLM足够强（如GPT-4V），单截图通道理论上也可提取文本+视觉所有信息。双通道的优势场景：1）密集文本页面（长文章）→ 文本通道token效率更高；2）VLM对页面文字的OCR错误率 > 0时→文本通道可做交叉验证/纠正。但单通道具有架构简洁、延迟更低的优势，在VLM能稳定提取页面文字且文本长度适中时，单通道足够甚至更优。

📅 知识时间锚

*任务上下文：2026年7月，LLM通用能力已达GPT-4水平，多模态VLM（GPT-4V/Claude-3.5 Vision）已主流化但仍存在表格OCR不精确的问题。Project Fetch是领域特定检索Agent的代表性工作。*

wiki-link：[[Wiki/论文笔记/03_Agent系统设计/152_ProjectFetch第二阶段多模态检索]]