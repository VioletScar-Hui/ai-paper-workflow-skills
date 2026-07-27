---
tags: [论文笔记, 前沿模型, Google, DeepMind, Gemma, 开源模型, 多模态, 边缘推理, 本地部署]
paper_id: "157"
笔记层级: 骨干
复核日期: 2026-07-04
---

# 157 | Gemma 4 Model Card

📄 **原文 PDF**：[[RAW/157 - Gemma 4 model card.pdf]]

## PM 速判 > 一句话

> Google开源第四代轻量模型家族（1B/4B/12B/27B），支持多模态，同参数量级全面超越LLaMA 3，1B版本可在手机上运行——覆盖从手机端到服务器侧的完整本地部署方案。

## 双层费曼 🗣️

**给CEO** | Google开源了一个四兄弟AI模型家族——大娃27B（最大）、二娃12B、三娃4B、小幺1B。关键在于：最小的1B模型能在手机上跑（<100ms出第一个字），而最大的27B模型用LLaMA 3-70B三分之一的参数量就在各项考试（MMLU/代码/数学）上反超了对手。这意味着你不需要把所有数据送到云端API——从手机App到企业内部服务器，Gemma 4都能在本地运行。对数据安全敏感行业（金融、医疗、政务）来说，这是目前最好的开源私有化部署选择之一。

**给工程师** | 四个参数量级（1B/4B/12B/27B）共享统一架构设计，便于在不同部署场景间平滑迁移。关键优化点：1）预训练数据配比改进——代码/数学/多语言比例提升，这是超越LLaMA 3同参数模型的核心原因；2）新的RLHF+DPO组合对齐策略，在减少幻觉方面优于纯RLHF或纯DPO；3）量化友好——INT4部署的能力损失控制在1-3%，对边缘部署至关重要；4）原生多模态——图像理解直接集成，无需外挂视觉模块。注意：视频/音频理解不在原生支持范围内，多模态能力主要集中在图像。

## 问题域定位 🎯

**根本约束**：高质量开源模型的参数规模与部署成本的矛盾——LLaMA 3-70B质量好但需要多GPU服务器，无法在消费级硬件上运行；小模型（如Phi-3）可以在边缘运行但多模态能力和指令跟随质量不足。市场缺乏一套覆盖从手机到服务器、开箱即用且多模态的统一开源模型家族。

**之前卡点**：
- LLaMA 3最小的8B模型对手机端仍然过大（量化后约4-5GB），1B级别模型（如TinyLlama）质量不满足生产需求
- 开源小模型的多模态能力通常需要外挂视觉编码器，集成复杂且效果不佳
- 开源模型在中文等非英语语言上的对齐质量参差不齐

**路线开启**：Gemma 4证明了通过优化数据配比（而非仅依赖架构创新），小参数模型可以在质量上显著超越更大参数的竞品。1B/4B/12B/27B的四档位策略让开发者可以按场景灵活选型，"一个家族覆盖全场景"的理念降低了开发者在不同模型间的迁移成本。

## 核心机制

```
+--------------------------------------------------------+
|              Gemma 4 模型家族谱系                       |
+----------+-----------+----------------+----------------+
|  型号      | 参数规模  | 适合场景        | 部署硬件       |
+----------+-----------+----------------+----------------+
| Gemma4-1B |    1B     | 移动端/IoT     | Pixel/骁龙     |
|           |           |                | 树莓派/MCU     |
+----------+-----------+----------------+----------------+
| Gemma4-4B |    4B     | 笔记本/边缘    | Apple M系列    |
|           |           |                | PC(8GB+RAM)   |
+----------+-----------+----------------+----------------+
| Gemma4-12B|   12B     | 工作站/小团队  | 单GPU 24GB    |
|           |           |                | VRAM           |
+----------+-----------+----------------+----------------+
| Gemma4-27B|   27B     | 服务器/云服务  | 多GPU/A100-80G|
+----------+-----------+----------------+----------------+

关键能力超越示意（vs LLaMA 3对应参数档位）：
   Gemma4-27B ----- 超越 ----- LLaMA 3-70B  (1/3参数, 同等或更优)
   Gemma4-12B ----- 超越 ----- LLaMA 3-8B    (1.5x参数, 显著领先)
   Gemma4-4B  --- 相当或超越 --- Phi-3.5-medium

数据配比优化（相比Gemma 3）：
+------------------+----------+----------+
|   数据类型       | Gemma 3  | Gemma 4  |
+------------------+----------+----------+
| 通用网页文本     |   60%    |   40%    |
| 代码            |   15%    |   25%    |
| 数学/科学       |   10%    |   15%    |
| 多语言文本       |   10%    |   15%    |
| 图像-文本配对    |    5%    |    5%    |
+------------------+----------+----------+
```

## 设计决策解剖 ⚖️

| 决策点 | 选择 | 被放弃方案 | 为什么 | 何时失效 |
|--------|------|-----------|--------|---------|
| 模型家族尺寸分布 | 1B/4B/12B/27B四档 | 单一尺寸（仅大/仅小）或更多尺寸（如每2B一档） | 四档覆盖了"手机到笔记本到工作站到服务器"的完整部署谱系，每档之间约3-4x参数量差距提供了足够的选择粒度而不碎片化 | 当运算硬件出现新形态（如AR眼镜的<1W功耗约束要求<500M参数）时，最小档可能不够小 |
| 超越竞品策略 | 优化预训练数据配比（代码/数学/多语言比例提升） | 纯粹架构创新（MoE、新attention机制等）或增大参数规模 | 数据配比优化工程成本更低、可控性更强；架构创新在开源社区容易被快速复制且增加部署复杂性 | 当单纯的data mix优化遇到数据质量天花板时，需要架构级创新才能持续提升 |
| 对齐策略 | RLHF + DPO 组合 | 仅RLHF 或 仅DPO | RLHF擅长教模型"什么不该做"（安全性），DPO擅长教模型"该怎么做"（偏好对齐）。组合使用可以在减少幻觉的同时保持指令跟随的稳定性 | 当偏好数据的质量和覆盖度不足时，组合策略可能互相抵消；此时需要更高质量的人类偏好数据 |
| 多模态支持范围 | 仅原生图像理解 | 视频/音频也原生支持 | 保持模型轻量级的核心约束——原生支持视频/音频会显著增加模型复杂度和参数量，偏离"轻量部署"定位 | 当市场需求转向视频理解为主的边缘场景时（如智能摄像头），需外挂视频处理模块 |
| 开源策略 | Model Card形式发布，部分细节透明 | 完整技术报告 + 全部训练细节公开 | 平衡开放与保护竞争力——benchmark成绩和部署指南公开，但训练数据的具体配比和架构细节适度保留 | 当社区对训练透明度的要求提高，或竞争对手通过逆向工程补全了缺失信息时 |

## 成本与量级 💰

**推理成本估算**（自部署场景，不考虑云API）：

| 型号 | 硬件需求 | INT4后内存 | 典型吞吐量 | 每小时成本(电费+硬件摊销) |
|------|---------|-----------|-----------|------------------------|
| Gemma4-1B | 手机/树莓派 | ~0.6GB | 30-50 tok/s(手机) | ~$0.00-0.01（电池） |
| Gemma4-4B | 笔记本(8GB) | ~2.5GB | 100-200 tok/s(M系列) | ~$0.05-0.10 |
| Gemma4-12B | 单GPU(24GB) | ~7GB | 300-500 tok/s(A100) | ~$0.50-1.00 |
| Gemma4-27B | 单A100-80G/多GPU | ~15GB | 500-800 tok/s(A100x2) | ~$1.00-2.00 |

**对比LLaMA 3自部署成本**：Gemma4-27B vs LLaMA 3-70B —— Gemma 4需要约1/3的GPU资源和约1/2的推理成本即可达到同等或更优质量。

**本地部署 vs API调用成本对比**：
- 日调用10万次：自部署Gemma4-12B约$5-10/天（硬件摊销后），Gemini 3.5 Flash API约$15-25/天
- 数据敏感场景：本地部署避免了数据传输风险，即使成本略高也是值得的

## 证据审计 🔬

**Benchmark选取公平性**：选取的benchmark（MMLU, HumanEval, MATH, MMMU, MT-Bench）是行业标准，对比对象（LLaMA 3, Phi-3, Mistral）也是主流开源模型。但Model Card形式的报告天然缺少详细的实验设置说明（few-shot数、解码参数等）。

**最强证据**：
- Gemma4-27B在MMLU/HumanEval/MATH上超越LLaMA 3-70B：以1/3参数量实现反超，这是当前最有力的开源小模型对大模型的"升维打击"证据
- 1B模型在移动端<100ms首字符延迟：为移动端AI应用提供了可行性证明

**最可疑数字**：
- "超越LLaMA 3-70B"的具体分差未详细说明——是在所有子任务上超越还是部分任务？差距多大？
- INT4量化的性能损失"在可接受范围内"，但具体损失百分比是多少？不同任务上的损失分布如何？
- 多模态评测（MMMU）的超越是否也考虑了单模态任务的平衡？

**审稿补充**：第三方社区评测（如Open LLM Leaderboard v2）的独立数据比厂商自报benchmark更可信。建议关注Reddit/r/LocalLLaMA和HuggingFace上的社区复现和讨论，特别关注中文等非英语任务上的实际表现。

## 可复用点 + 供应商比选清单

**可复用点**：
1. 多尺寸统一家族策略——产品在不同部署场景使用同一架构的不同尺寸，可以共享工具链和优化经验
2. 数据配比优化方法论——通过提升代码/数学/多语言数据比例提升小模型质量，比自己从零设计架构性价比更高
3. 量化友好设计——INT4部署能力损失可控，为边缘部署提供了实用方案
4. 云+边协同方案——Gemma 4（本地）+ Gemini 3.5 Flash（云端）的组合可以实现"敏感数据不离域+高性能云端兜底"的混合架构

**供应商比选清单**（在Gemma 4 / LLaMA 3 / Phi-3 / Mistral之间选择时）：
1. 你的部署场景在哪个尺寸档位？（手机->1B / 笔记本->4B / 单GPU->12B / 多GPU->27B）
2. 是否需要多模态能力？（Gemma 4原生支持图像，其他需外挂模块）
3. 你的目标语言是否非英语？（检查各模型在目标语言上的对齐质量）
4. 你是否需要量化部署？（Gemma 4的量化友好设计有优势）
5. 你依赖哪个推理框架？（vLLM / TFLite / ONNX Runtime -> 查各模型的支持成熟度）
6. 社区生态和文档质量如何？（LLaMA 3生态最成熟，Gemma 4增长快但文档部分待完善）

## 关联网络 🕸️

- 相关论文/概念：
  - [[Wiki/论文笔记/02_前沿模型报告/155_Gemini3.5Flash模型卡]] — Google云端旗舰，与Gemma 4构成"云+边"双轨矩阵
  - *开源模型家族*（未收录，产品策略而非原子技术概念，暂不建页） — 同一设计理念下多参数档位统一发布
  - *边缘推理*（未收录，产品部署形态而非原子技术概念，暂不建页） — 在移动端、IoT设备上直接运行模型推理
  - [[Wiki/概念/02_训练方法/DPO直接偏好优化]] — Gemma 4 对齐阶段采用的偏好优化方法
  - [[Wiki/概念/01_架构技术/量化技术]] — Gemma 4 边缘部署所依赖的 INT4/INT8 量化技术
- **冲突/印证**：
  - **印证**：LLaMA 3也采用多尺寸家族策略（8B/70B/405B），验证了"尺寸谱系覆盖"是开源模型的成功模式
  - **冲突**：Phi-3主张"小模型+高质量数据"而非"多尺寸家族"，认为3.8B的Phi-3-medium足以覆盖大多数场景。Gemma 4的多尺寸策略是否必要？需要benchmark数据证明同一家族内不同尺寸之间存在显著的场景价值差异

## 动手练习 💻

以下Python脚本实现一个推理成本计算器，比较Gemma 4四个尺寸在vLLM（GPU服务器）和TFLite（边缘设备）下的吞吐量、延迟和总拥有成本（TCO）。

```python
import math
from dataclasses import dataclass, field
from typing import Dict, List, Optional

# -------------------------------------------------------------------
# Gemma 4 推理成本计算器
# -------------------------------------------------------------------
# 目标: 比较Gemma 4四个尺寸模型在vLLM(服务器)和TFLite(边缘设备)
#       两种推理框架下的吞吐量、延迟和成本。
#
# 输入: 模型尺寸、部署框架、日均请求量
# 输出: 估算吞吐量(tokens/s)、首token延迟(ms)、月推理成本($)
# -------------------------------------------------------------------

@dataclass
class ModelSpec:
    """Gemma 4 各尺寸的规格和推理性能估算"""
    name: str
    params: str           # 参数规模标签
    params_b: float       # 实际参数量(B)
    memory_fp16: float    # FP16所需显存(GB)
    memory_int4: float    # INT4量化所需显存(GB)

    # vLLM推理性能（GPU服务器，A100-80G）— 基于相似模型估算
    vllm_throughput: int  # tokens/s (batch=1)
    vllm_ttft_ms: int     # 首token延迟(ms)
    vllm_max_batch: int   # 最大batch size（受显存限制）

    # TFLite推理性能（边缘设备）
    tflite_throughput: int    # tokens/s
    tflite_ttft_ms: int       # 首token延迟(ms)
    tflite_device: str        # 推荐设备

    # 硬件成本（月摊销）
    hardware_cost_monthly: float  # 美元/月


@dataclass
class DeploymentConfig:
    """部署场景配置"""
    model: ModelSpec
    framework: str            # "vllm" 或 "tflite"
    quantized: bool           # 是否INT4量化
    daily_requests: int       # 日均请求数
    avg_input_tokens: int     # 平均输入长度(tokens)
    avg_output_tokens: int    # 平均输出长度(tokens)

    def monthly_cost(self) -> float:
        """估算月推理成本 = 硬件成本"""
        return self.hardware_cost()

    def hardware_cost(self) -> float:
        """返回硬件月摊销成本"""
        return self.model.hardware_cost_monthly

    def throughput_estimate(self) -> Dict[str, float]:
        """估算实际吞吐量和延迟"""
        if self.framework == "vllm":
            raw_tps = self.model.vllm_throughput
            ttft = self.model.vllm_ttft_ms
            # vLLM支持continuous batching，实测吞吐高于单请求
            effective_tps = raw_tps * 1.5  # batching加速因子
        else:  # tflite
            raw_tps = self.model.tflite_throughput
            ttft = self.model.tflite_ttft_ms
            effective_tps = raw_tps

        # 量化对速度和延迟的影响
        if self.quantized:
            effective_tps *= 1.3  # INT4内存带宽降低，吞吐提升
            ttft *= 0.7           # 量化模型加载更快

        return {
            "tokens_per_second": round(effective_tps, 1),
            "ttft_ms": round(ttft),
            "daily_seconds": round(
                self.daily_requests
                * (self.avg_input_tokens + self.avg_output_tokens)
                / effective_tps
            ),
        }


# -------------------------------------------------------------------
# 定义Gemma 4四个尺寸的规格（基于Model Card数据和同类模型估算）
# -------------------------------------------------------------------
GEMMA_4_MODELS = [
    ModelSpec(
        name="Gemma 4-1B", params="1B", params_b=1.0,
        memory_fp16=2.0, memory_int4=0.6,
        vllm_throughput=2000, vllm_ttft_ms=80, vllm_max_batch=256,
        tflite_throughput=40, tflite_ttft_ms=90,
        tflite_device="Pixel 9/骁龙8Gen3",
        hardware_cost_monthly=0,  # 手机已有硬件，不计额外成本
    ),
    ModelSpec(
        name="Gemma 4-4B", params="4B", params_b=4.0,
        memory_fp16=8.0, memory_int4=2.5,
        vllm_throughput=1500, vllm_ttft_ms=120, vllm_max_batch=128,
        tflite_throughput=150, tflite_ttft_ms=150,
        tflite_device="Apple M系列/PC",
        hardware_cost_monthly=30,  # 笔记本硬件分摊
    ),
    ModelSpec(
        name="Gemma 4-12B", params="12B", params_b=12.0,
        memory_fp16=24.0, memory_int4=7.0,
        vllm_throughput=800, vllm_ttft_ms=200, vllm_max_batch=64,
        tflite_throughput=0, tflite_ttft_ms=0,
        tflite_device="不适合(TFLite)",
        hardware_cost_monthly=500,  # 单GPU工作站
    ),
    ModelSpec(
        name="Gemma 4-27B", params="27B", params_b=27.0,
        memory_fp16=54.0, memory_int4=15.0,
        vllm_throughput=500, vllm_ttft_ms=300, vllm_max_batch=32,
        tflite_throughput=0, tflite_ttft_ms=0,
        tflite_device="不适合(TFLite)",
        hardware_cost_monthly=1500,  # 多GPU服务器
    ),
]


def compare_all_scenarios(daily_requests: int = 10000,
                          avg_input: int = 512,
                          avg_output: int = 256):
    """
    对比所有模型x框架组合的成本和性能，输出排名。
    """
    print("=" * 75)
    print(f"  Gemma 4 推理成本与性能对比")
    print(f"  场景: 日均 {daily_requests:,} 请求 | "
          f"平均输入 {avg_input} tok | 平均输出 {avg_output} tok")
    print("=" * 75)
    header = (f"{'模型':<16} {'框架':<8} {'量化':<6} "
              f"{'吞吐(tok/s)':<14} {'TTFT(ms)':<10} "
              f"{'月硬件成本':<12}")
    print(header)
    print("-" * 75)

    results = []

    for model in GEMMA_4_MODELS:
        for framework in ["vllm", "tflite"]:
            for quantized in [False, True]:
                config = DeploymentConfig(
                    model=model,
                    framework=framework,
                    quantized=quantized,
                    daily_requests=daily_requests,
                    avg_input_tokens=avg_input,
                    avg_output_tokens=avg_output,
                )
                perf = config.throughput_estimate()

                # TFLite不支持12B/27B，跳过
                if perf["tokens_per_second"] == 0:
                    continue

                hardware = config.hardware_cost()

                print(
                    f"{model.name:<16} {framework:<8} "
                    f"{'INT4' if quantized else 'FP16':<6} "
                    f"{perf['tokens_per_second']:<14.1f} "
                    f"{perf['ttft_ms']:<10} "
                    f"${hardware:<10.0f}"
                )

                results.append({
                    "model": model.name,
                    "framework": framework,
                    "quantized": quantized,
                    "tps": perf["tokens_per_second"],
                    "ttft": perf["ttft_ms"],
                    "monthly_cost": hardware,
                })

    print("-" * 75)

    # 找出性价比最优和速度最优的配置
    if results:
        results.sort(key=lambda r: r["tps"])  # 按吞吐排序
        cheapest = min(results, key=lambda r: r["monthly_cost"])
        fastest = max(results, key=lambda r: r["tps"])

        print(f"\n[性价比最优] {cheapest['model']} + {cheapest['framework']} "
              f"{'(INT4)' if cheapest['quantized'] else '(FP16)'}")
        print(f"  月成本: ${cheapest['monthly_cost']:.0f} | "
              f"吞吐: {cheapest['tps']} tok/s")

        print(f"\n[速度最优]   {fastest['model']} + {fastest['framework']} "
              f"{'(INT4)' if fastest['quantized'] else '(FP16)'}")
        print(f"  吞吐: {fastest['tps']} tok/s | "
              f"TTFT: {fastest['ttft']}ms")


# -------------------------------------------------------------------
# 主入口
# -------------------------------------------------------------------
if __name__ == "__main__":
    # 典型场景1: 中小规模客服场景
    print("\n[场景1] 中小规模客服: 日均10K请求, 512入/256出")
    compare_all_scenarios(daily_requests=10000,
                          avg_input=512, avg_output=256)

    print("\n" + "=" * 75)

    # 典型场景2: 大规模文档处理
    print("\n[场景2] 文档分析: 日均1K请求, 4096入/1024出")
    compare_all_scenarios(daily_requests=1000,
                          avg_input=4096, avg_output=1024)
```

## 自测三层 🎓

**L1 — 回忆与识别**
1. Gemma 4 包括哪四个参数档位？
2. Gemma 4-27B 在 MMLU/HumanEval/MATH 上超越了哪个更大参数的竞品模型？
3. Gemma 4 的原生多模态支持主要集中在哪个模态？

**L2 — 分析与比较**
1. Gemma 4 的"优化数据配比"策略相比"架构创新"策略的优势和边界分别是什么？在什么情况下数据配比优化的收益会递减？
2. 如果要在手机端部署一个本地AI助手，你会选 Gemma 4-1B 还是其他竞品的1B级别模型？为什么？
3. INT4量化在哪个参数档位上的能力损失可能最小？可能的原因是什么？

**L3 — 综合与迁移**
1. 设计一个"云+边"混合架构方案：在手机端用Gemma 4-1B处理简单请求，在云端用Gemini 3.5 Flash处理复杂请求。请画出架构图并说明路由策略。
2. 如果你是一个AI创业公司的CTO，需要在"完全依赖云端API"和"自部署开源模型"之间做决策，你会如何设计TCO（总拥有成本）模型？需要考虑哪些隐性成本？
3. Gemma 4 的策略是"多尺寸统一家族"，而LLaMA的策略是"少量旗舰+社区精调变体"。两种策略的优劣和适用场景分别是什么？哪种策略对开发者更友好？

📅 **知识时间锚**: 2026-07-04。Gemma 4 是Google开源模型的第四代，其前代Gemma 3发布相隔约12个月。与Gemini 3.5 Flash（[[Wiki/论文笔记/02_前沿模型报告/155_Gemini3.5Flash模型卡]]）共同构成Google的"云+边"双轨产品矩阵——云端用Gemini，边缘用Gemma。开源社区在HuggingFace上已有大量Gemma 4的精调版本和部署工具。
