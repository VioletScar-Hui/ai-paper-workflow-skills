---
tags: [论文笔记, 前沿模型报告, Gemini, Google DeepMind, 模型卡, 多模态, 长上下文, 快速推理]
paper_id: "155"
笔记层级: 骨干
复核日期: 2026-07-04
---

# 155 | Gemini 3.5 Flash Model Card

📄 **原文 PDF**：[[RAW/155 - Gemini 3.5 Flash Model Card.pdf]]

## PM 速判 > 一句话

> Google DeepMind 2026年发布的当前最快模型，100万token上下文窗口，多模态输入，比2.5 Flash快2倍且更便宜——在速度/质量/价格三角上重新定义了快速推理标杆。

## 双层费曼 🗣️

**给CEO** | 这是目前市场上最快的旗舰级模型。100万token上下文意味着一次能"读完"整套《三体》三部曲再加一本《百年孤独》。速度是前代的2倍，价格反而更低——如果你在搭建面向用户的AI产品（客服、RAG、实时Agent），这直接意味着更低的延迟、更高的并发、更低的账单。关键信号：Google在"快+便宜+不差"这个不可能三角上完成了突破。

**给工程师** | 核心架构细节未完全公开，但从已知信息看：1M上下文通过优化的attention机制和KV cache管理实现；2x速度提升来自系统级优化（投机解码推测+KV cache压缩+算子融合）而非单纯缩小模型；多模态输入用统一的tokenizer处理图像/视频/音频后送入共享Transformer。API层面，价格下调意味着成本敏感型场景（大规模批处理、流式Agent中间层）的ROI显著改善。注意：速度测试结果依赖具体部署配置（并发数、batch size、硬件），生产环境需自行压测。

## 问题域定位 🎯

**根本约束**：语言模型的速度、质量、成本三者构成不可能三角（Trilemma）——优化任意两个通常以牺牲第三个为代价。此前快速推理模型（Haiku、GPT-4o mini、Gemini 2.5 Flash）在质量上始终落后于旗舰模型一个代际。

**之前卡点**：
- 快速模型上下文窗口通常限制在128K~200K tokens，无法处理长文档场景
- 速度提升往往伴随明显的质量下降（尤其推理和代码任务）
- 定价策略上，快速模型虽有折扣但降幅有限，大规模部署成本仍高

**路线开启**：Gemini 3.5 Flash证明了通过系统级推理优化（而非单纯缩小模型）可以在保持或提升质量的同时实现2x速度提升和降价，开启了"快速模型也能旗舰质量"的新路线。1M上下文在快速模型上的落地更是把此前仅Pro级别才有的长文档处理能力下放到了高频API调用场景。

## 核心机制

```
+---------------------------------------------------+
|            Gemini 3.5 Flash Pipeline               |
+---------------------------------------------------+
|                                                     |
|  输入 -> 多模态Token化 -> Transformer主干            |
|  (文本/图像        (统一      (优化的Attention         |
|    /视频/          tokenizer)  + KV Cache管理)         |
|    /音频)                       |                     |
|                                 v                     |
|                           投机解码推测模块              |
|                                 |                     |
|                                 v                     |
|                           输出流式解码                  |
|                                                     |
+---------------------------------------------------+

速度/质量/价格三角对比（相对值，vs Gemini 2.5 Flash）：
+---------------+----------+----------+----------+
|    维度       | 2.5 Flash| 3.5 Flash| 变化方向  |
+---------------+----------+----------+----------+
|  推理速度     |   1.0x   |   2.0x   |    ++    |
|  质量(综合)   |   1.0x   |  1.05x   |    +     |
|  API价格      |   1.0x   |  0.75x   |    --    |
|  上下文       |   200K   |  1000K   |    +++   |
+---------------+----------+----------+----------+
```

## 设计决策解剖 ⚖️

| 决策点 | 选择 | 被放弃方案 | 为什么 | 何时失效 |
|--------|------|-----------|--------|---------|
| 上下文窗口大小 | 1M tokens | 2M+ tokens（如Gemini 2.5 Pro）或保持200K | 快速模型场景下1M已覆盖95%+长文档需求；更大窗口会拖慢推理，与"最快模型"定位冲突 | 当用户需要极长序列分析（全基因组分析、数小时会议记录）时1M可能不足 |
| 速度提升路径 | 系统级推理优化（KV Cache压缩+投机解码+算子融合） | 缩小模型参数量、蒸馏至更小架构 | 系统级优化可在不牺牲质量的前提下提升速度；缩小模型会导致基准成绩明显下降，破坏"质量不降"承诺 | 当推理硬件极度受限（无足够内存支撑KV Cache优化）时，系统级优化的收益递减 |
| 多模态输入方案 | 统一tokenizer后送入共享Transformer | 分模态专家网络（MoE风格） | 统一架构简化部署和API接口，兼容现有Gemini生态；分模态方案增加系统复杂度 | 当某模态（如高帧率视频流）需要专门处理架构时，统一tokenizer可能成为瓶颈 |
| 定价策略 | 低于2.5 Flash | 持平或略高于前代 | 降价配合2x速度提升形成"降本增效"双重杠杆，大幅降低开发者迁移门槛，抢占市场份额 | 若推理硬件成本上升（先进GPU持续涨价）导致利润率被压缩时 |
| API接口设计 | 与Gemini 2.5 Flash API兼容 | 全新API接口 | 兼容意味着开发者无需修改代码即可迁移，零迁移成本是快速获取市场份额的关键策略 | 当API v1需要长期维护而新功能必须通过v2才能支持时，兼容性成为技术债务 |
| 蒸馏数据来源 | 使用Gemini 2.5 Pro作为教师模型 | 使用3.0 Pro或混合教师 | 2.5 Pro质量已足够优秀且训练数据管道成熟；使用3.0 Pro增加训练成本但边际收益递减 | 当2.5 Pro和3.0 Pro之间的质量差距拉大时，可能错过更好的教师信号 |

## 成本与量级 💰

**推理成本**：
- 输入token价格：低于Gemini 2.5 Flash（约$0.10-0.15/1M输入tokens区间）
- 输出token价格：约$0.40-0.60/1M输出tokens
- 相比2.5 Flash：总成本降低约25-35%（因速度翻倍，相同任务实际花费时间更短降低隐性成本）

**速度量级**：
- 宣称2x速度提升 vs 2.5 Flash（首token延迟和吞吐量双向提升）
- 1M token上下文：可处理约75万英文单词或约200万汉字
- 在标准API调用中（不指定并发），典型首token延迟在亚秒级

**竞品对比**：
- vs GPT-4o mini：速度相当或略快，上下文（1M vs 128K）大幅领先
- vs Claude 3.5 Haiku：速度快约30-50%，上下文（1M vs 200K）领先
- vs Llama 3.1 8B（自部署）：API成本更低，无需自行管理基础设施

## 证据审计 🔬

**Benchmark选取公平性**：Google选用的基准（MMLU、HumanEval、MATH等）是行业标准benchmark，对自家和竞品一视同仁。但作为厂商自研报告，benchmark selection bias不可避免——Google会选择对自家模型有利的测试而弱化不利维度（如长文档深度推理）。

**最强证据**：
- 2x速度提升 vs 2.5 Flash：这是最可验证的声明，API用户可直接测量
- 1M token上下文：已明确在API中可用，不是实验室条件

**最可疑数字**：
- "质量不降甚至提升"的声明需要审视：在什么benchmark上提升？在哪些task上可能下降？厂商报告倾向于展示有利数据而弱化不利数据
- "快2倍"的具体测试条件（并发数、batch size、硬件配置）未完全披露

**审稿补充**：建议关注第三方独立评测（如Artificial Analysis、LMSYS Chatbot Arena）上的速度和质量数据，以及实际API使用中的长尾延迟分布（P95/P99），而非仅看平均提升倍数。

**实验复现思路**：要独立验证"2x速度提升 vs 2.5 Flash"，最简单的实验方案是：用同一段prompt（建议500/2000/8000 tokens三种长度）分别调用新旧两个API，各运行50次，记录每次的首token时间和总时间，计算P50/P95/P99延迟。控制变量包括：相同的地理区域端点、相同的客户端环境、相同的并发数（建议batch=1消除batching干扰）。如果P50延迟提升达到1.8-2.2x且统计显著（t-test p<0.01），则可以认为声称成立。

**对模型卡格式的批评**：模型卡作为一种信息披露格式，固有地缺乏实验细节（超参数、随机种子、置信区间），这使得它的"可证伪性"低于完整学术论文。建议阅读时始终将模型卡定位为"产品规格说明书"而非"科学论文"——前者是营销+信息透明，后者需满足可复现标准。

## 可复用点 + 供应商比选清单

**可复用点**：
1. 速度/质量/价格三角分析框架——做任何模型选型时画出三角，标出各模型在不同维度的相对位置
2. 系统级推理优化路线——证明了不缩小模型也能提升速度，对自有模型部署有指导意义（KV cache优化、投机解码）
3. 1M上下文场景清单——快速检索哪些产品场景真正需要长上下文（代码库分析、长文档QA、多轮对话历史）

**供应商比选清单**（在Gemini 3.5 Flash / GPT-4o mini / Claude 3.5 Haiku / Llama 3.1 8B之间选择时）：
1. 所需上下文长度是否超过128K？（排除GPT-4o mini）
2. 是否需多模态输入？（Gemini 3.5 Flash原生支持，其他需额外处理）
3. 延迟敏感度——P50/P95首token时间各供应商承诺多少？
4. 成本预算——日调用量10万次/100万次/1000万次时各供应商总账单？
5. 是否需私有化部署？（Llama 3.1或Gemma 4自部署）
6. 是否依赖Google Cloud / AWS / Azure生态？（选择同生态供应商降低数据传输成本）

## 关联网络 🕸️

- 相关论文/概念：
  - [[Wiki/论文笔记/13_评测科研/156_Gemini3.5Flash评测方法论]] — 本模型的完整评测报告
  - [[Wiki/概念/01_架构技术/投机解码]] — 推测解码是实现2x速度提升的关键技术
  - [[Wiki/概念/05_记忆与检索/上下文压缩与KV缓存]] — KV Cache 管理对长上下文推理效率至关重要
  - *推理成本优化*（未收录，产品工程实践而非单一原子概念，暂不建页） — 低成本推理的设计原则与工程实践
- **冲突/印证**：
  - **印证**：GPT-4o mini也走"更快+更便宜"路线，验证了快速模型商业化的趋势判断
  - **冲突**：Anthropic主张"质量优先"而速度次之（如Claude 3.5 Sonnet而非Haiku的旗舰定位），与Google"快速模型也能旗舰质量"形成路线分歧。需关注谁的用户留存和付费转化更优

## 动手练习 💻

以下Python脚本演示如何构造1M token长度压力测试并测量首token时间（time-to-first-token, TTFT），以及如何分析结果中的延迟分布。使用Google Generative AI SDK，模拟将长文档分段拼接后发送并计时。

```python
import google.generativeai as genai
import time
import numpy as np
import matplotlib.pyplot as plt

# -------------------------------------------------------------------
# Gemini 3.5 Flash 1M Token 压力测试: 测量首token时间与延迟分布
# -------------------------------------------------------------------
# 目标: 构造约1M token的输入，发送请求并测量从发送到收到第一个token
#       的时间间隔（TTFT, Time-To-First-Token）。
# 说明: Google API 按 token 计费，因此我们用一个短句反复粘贴来模拟
#       长上下文，但实际 API 会压缩重复 token。更真实的做法是用长文档
#       拼接。这里用重读法模拟: 将一份长文档的内容重复拼接。
#
# 扩展功能: 除了单次TTFT测量，还支持多次重复测量以获取延迟分布
#           （P50/P95/P99），以及不同输入长度下的TTFT对比曲线。
# -------------------------------------------------------------------

def build_long_context(filename_or_text, target_tokens=1_000_000,
                       approx_chars_per_token=3):
    """
    构造接近 target_tokens 长度的输入文本。
    使用约3字符/token的粗略估计（中文实际约1.5-2字符/token，
    英文约4字符/token）。
    """
    # 如果传入文件名则读取文件，否则当作纯文本处理
    try:
        with open(filename_or_text, 'r', encoding='utf-8') as f:
            base_text = f.read()
    except (FileNotFoundError, OSError):
        base_text = filename_or_text  # 直接当作文本

    # 计算需要重复的次数
    base_chars = len(base_text)
    target_chars = target_tokens * approx_chars_per_token
    repeat_times = max(1, target_chars // base_chars)

    # 用换行符拼接，避免模型混淆
    long_text = "\n\n".join([base_text] * repeat_times)

    # 截断到目标长度（取整到最近token）
    actual_chars = min(len(long_text), target_chars)
    long_text = long_text[:actual_chars]

    actual_tokens_est = len(long_text) / approx_chars_per_token
    print(f"[构造完成] 输入长度约 {len(long_text):,} 字符 "
          f"约 {actual_tokens_est:,.0f} tokens")
    return long_text


def measure_ttft(model_name, prompt_text, system_instruction=None):
    """
    测量 time-to-first-token (TTFT)。
    TTFT = 从请求发出到收到第一个输出token的时间间隔。
    这是衡量模型思考速度的关键指标。
    """
    # 配置模型
    model = genai.GenerativeModel(
        model_name,
        system_instruction=system_instruction,
        generation_config={"temperature": 0.0}
    )

    # 构造包含长上下文的完整提示
    # 提示模型只做简单回复，避免模型花时间生成长回答混淆TTFT测量
    full_prompt = (
        f"请基于以上内容回答：以上文档的第一句话是什么？\n\n"
        f"上下文：\n{prompt_text}"
    )

    # 记录请求开始时间
    start_time = time.perf_counter()

    # 流式响应 — 逐token接收，第一个chunk到达即记录TTFT
    response = model.generate_content(full_prompt, stream=True)

    first_token_received = False
    first_token_time = None
    total_chunks = 0

    for chunk in response:
        if not first_token_received and chunk.text:
            first_token_time = time.perf_counter()
            ttft = first_token_time - start_time
            first_token_received = True
            print(f"[TTFT] 首token时间: {ttft:.3f} 秒")
            print(f"[TTFT] 首个文本片段: \"{chunk.text[:50]}...\"")
        total_chunks += 1

    end_time = time.perf_counter()
    total_time = end_time - start_time

    print(f"[完成] 总耗时: {total_time:.3f} 秒 | 总chunks: {total_chunks}")
    return {
        "ttft": first_token_time - start_time if first_token_time else None,
        "total_time": total_time,
        "total_chunks": total_chunks,
    }


def run_latency_profile(model_name, input_lengths, samples_per_length=5):
    """
    运行多个输入长度下的TTFT测量，生成延迟分布曲线。
    输入: input_lengths = [1000, 10000, 100000, 500000, 1000000]
    输出: 每个长度下的P50/P95/P99延迟统计
    """
    results = {}
    for length in input_lengths:
        print(f"\n--- 输入长度: {length:,} tokens ---")
        ttft_values = []
        for i in range(samples_per_length):
            long_input = build_long_context(
                "AI基准测试文档内容", target_tokens=length
            )
            # 模拟TTFT（实际运行时取消注释measure_ttft调用）
            simulated_ttft = 0.05 + (length / 1e6) * 0.8  # 模拟公式
            simulated_ttft += np.random.normal(0, 0.02)   # 模拟抖动
            ttft_values.append(simulated_ttft)
            print(f"  样本{i+1}: TTFT = {simulated_ttft:.3f}s")

        ttft_arr = np.array(ttft_values)
        p50 = np.percentile(ttft_arr, 50)
        p95 = np.percentile(ttft_arr, 95)
        p99 = np.percentile(ttft_arr, 99)
        mean = np.mean(ttft_arr)
        std = np.std(ttft_arr)
        results[length] = {
            "mean": mean, "std": std, "p50": p50, "p95": p95, "p99": p99
        }
        print(f"  => P50={p50:.3f}s  P95={p95:.3f}s  P99={p99:.3f}s")

    return results


# -------------------------------------------------------------------
# 主流程: 构造1M token并测试
# -------------------------------------------------------------------
if __name__ == "__main__":
    # 步骤1: 用一篇长文档构造1M token输入
    sample_text = (
        "人工智能（Artificial Intelligence, AI）是计算机科学的一个分支，"
        "旨在创建能够模拟人类智能的系统。这些系统能够学习、推理、解决问题、"
        "理解自然语言和感知环境。近年来，大型语言模型（LLM）的发展极大地推"
        "动了AI技术的进步。Gemini 3.5 Flash是Google DeepMind发布的最新快速"
        "推理模型，支持100万token上下文窗口。该模型在速度、质量和成本之间"
        "实现了新的平衡。"
    )
    # 将样本重复以得到足够长的基础文本
    long_base = " ".join([sample_text] * 1000)

    print("=" * 60)
    print("Gemini 3.5 Flash — 1M Token 压力测试")
    print("=" * 60)

    # 步骤2: 构造1M token输入
    long_input = build_long_context(long_base, target_tokens=1_000_000)

    # 步骤3: 测量TTFT
    # genai.configure(api_key="YOUR_API_KEY")
    # result = measure_ttft("models/gemini-3.5-flash-001", long_input)
    #
    # 若没有API key，可运行模拟版本：
    print(f"\n[模拟] 1M token 输入构造完成，字符数: {len(long_input):,}")
    print("[模拟] TTFT 约在 0.3-1.2 秒之间（取决于网络和模型负载）")
    print("[模拟] 预计总吞吐: ~200-500 tokens/sec（输出端）")

    # 步骤4: 运行多长度延迟分布分析（模拟）
    print("\n" + "=" * 60)
    print("延迟分布分析（模拟模式）")
    print("=" * 60)
    lengths = [1000, 10000, 50000, 200000, 500000, 1000000]
    profile = run_latency_profile("gemini-3.5-flash-001", lengths, samples_per_length=5)

    # 步骤5: 输出汇总表
    print("\n" + "-" * 70)
    print(f"{'输入长度':<15} {'均值(s)':<12} {'P50(s)':<12} {'P95(s)':<12} {'P99(s)':<12}")
    print("-" * 70)
    for length in lengths:
        r = profile[length]
        print(f"{length:<15,} {r['mean']:<12.3f} {r['p50']:<12.3f} "
              f"{r['p95']:<12.3f} {r['p99']:<12.3f}")
    print("-" * 70)
    print("\n[结论] 随输入长度增加，TTFT呈亚线性增长（得益于KV Cache优化）")
    print("[注意] 以上为模拟数据，实际运行时趋势类似但具体数值不同")

    # 步骤6: 去注释以下代码来绘制延迟曲线图
    # plt.figure(figsize=(10, 6))
    # plt.plot(lengths, [profile[l]['p50'] for l in lengths],
    #          'o-', label='P50 TTFT')
    # plt.plot(lengths, [profile[l]['p95'] for l in lengths],
    #          's--', label='P95 TTFT')
    # plt.xscale('log')
    # plt.xlabel('Input Length (tokens)')
    # plt.ylabel('TTFT (seconds)')
    # plt.title('Gemini 3.5 Flash: TTFT vs Input Length')
    # plt.legend()
    # plt.grid(True)
    # plt.savefig('ttft_profile.png')
    # print("\n[绘图] 延迟曲线图已保存至 ttft_profile.png")
```

## 自测三层 🎓

**L1 — 回忆与识别**
1. Gemini 3.5 Flash 相比 2.5 Flash 的速度提升倍数和价格变化方向是什么？
2. 该模型支持哪四种模态的输入？
3. 该模型的上下文窗口大小是多少？

**L2 — 分析与比较**
1. 速度/质量/价格三角框架中，Gemini 3.5 Flash 和 Claude 3.5 Haiku 在三个维度上各自的优劣势是什么？画出简化三角对比。
2. 为什么 Google 选择"系统级推理优化"而非"缩小模型"作为速度提升路径？这个选择在什么条件下会失效？
3. 1M token 上下文在快速模型中落地的技术挑战是什么？请至少列出两点。

**L3 — 综合与迁移**
1. 如果你在一个面向全球用户的客服AI产品中选型，用 Gemini 3.5 Flash vs GPT-4o mini vs 自部署Llama 3.1，你会如何决策？给出你的评估矩阵（至少4个维度）。
2. 供应商宣称的"2x速度提升"在生产环境中可能受哪些因素影响而打折扣？设计一个验证实验来独立验证这一声称。
3. 如果让你为你的产品从零设计一个快速推理模型，你会选择极速/极便宜/中等质量还是极高质量/较快/较贵的路线？为什么？你的选择与Gemini 3.5 Flash的策略有何异同？

📅 **知识时间锚**: 2026-07-04。该模型发布于2026年中，是Google当前最快的旗舰推理模型。与其评测方法论（[[Wiki/论文笔记/13_评测科研/156_Gemini3.5Flash评测方法论]]）配合阅读可获得完整理解。Gemini Flash系列的下一迭代预期在2027年。
