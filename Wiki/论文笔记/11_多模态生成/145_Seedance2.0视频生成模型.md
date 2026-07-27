---
tags: [论文笔记, 视频生成, DiT, 文生视频, ByteDance, 多模态生成, 骨干精读]
笔记层级: 骨干
paper_id: "145"
复核日期: 2026-07-04
---

# Seedance 2.0：字节跳动旗舰视频生成模型

📄 **原文 PDF**：[[RAW/145 - Seedance 2.0 - ByteDance Seed Official Model Page.pdf]]

## PM 速判（30秒）
> 字节跳动 Seed 团队基于 DiT 架构的视频生成旗舰，采用**统一多模态音视频联合生成架构**，支持文本/图片/音频/视频输入，在运动真实感、提示跟随和音视频同步上达到行业领先。在产品层面，支持 1080p 分辨率、最长约 20 秒的视频生成（后续 Seedance 2.5 推进到 4K 30 秒）。PM 须知：运动一致性是 Seedance 2.0 的核心差异化——光流约束和物理感知训练目标使其在物体一致性和运动平滑度上超越同期竞品。API 通过火山引擎提供，国内通过即梦/豆包使用。

## 双层费曼 🗣️
> **给 CEO 的一句话**：字节的视频生成模型——输入一句话或一张图，自动生成电影级质量的最长 20 秒视频，画面稳定、运动自然、光影一致。可以让用户（比如电商卖家或广告创作者）不需要专业剪辑就能生成产品展示视频。这是目前市面上运动连贯性最好的模型之一。
>
> **给工程师的一段话**：Seedance 2.0 的核心是 DiT（Diffusion Transformer）架构，用 Transformer 替换传统 U-Net 作为扩散模型骨干，支持全注意力时空建模——每一帧与其他所有帧做注意力，避免滑动窗口导致帧间不一致。关键创新包括：(1) 分辨率渐进训练——先在低分辨率学语义再到 1080p 微调；(2) 多模态统一编码——文本/图像/音频/视频输入共用一个语义空间；(3) 运动一致性奖励——专用奖励模型基于光流一致性打分，指导 RLHF 阶段优化；(4) SeedVideoBench-2.0 内部评测体系，覆盖 T2V、I2V、多模态任务多维度雷达图。架构设计上，与 Sora、Kling 同为 DiT 路线，但 Seedance 在运动平滑度和中英双语理解上具独特优势。

## 问题域定位 🎯
- **根本约束**：视频生成需要在分辨率、时长、运动一致性和提示跟随四个维度同时达到商业可用水平——任何一维短板都会使输出不可用。扩散模型在视频上需要同时建模空间（帧内像素关系）和时间（帧间运动关系），计算量远超图像生成。
- **之前的方案卡在哪**：早期模型（Seedance 1.0 等）在单帧质量上可接受，但时序一致性差——物体会在帧间变形、消失或突变。原因在于滑动窗口注意力（只建模邻近帧）和缺乏物理感知的训练目标。此外长视频（>15 秒）随着帧数增加运动漂移累积。
- **开启/关闭的路线**：验证了全时空 DiT + RLHF 运动奖励的技术组合可以将视频生成推进到"商业可用"水平。关闭了对"DiT 在视频上不如 U-Net 高效"的质疑。同时多模态统一架构（音频+视频联合生成）开启了音视频同步生成的产品路径。

## 核心机制

```
┌─────────────────────────────────────────────────────────────┐
│  Seedance 2.0 训练与推理管线                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌────────────────────────────────────────────────────┐     │
│  │  Stage 1: 预训练（低分辨率 256p）                      │     │
│  │  在亿万级视频数据上学习基础语义和运动                        │     │
│  │  使用全帧时空注意力 DiT                                    │     │
│  └─────────────────────┬──────────────────────────────────┘     │
│                        ▼                                       │
│  ┌────────────────────────────────────────────────────┐     │
│  │  Stage 2: 分辨率渐进训练                            │     │
│  │  256p → 512p → 720p → 1080p 逐步微调               │     │
│  │  保持运动先验，提升细节质量                                │     │
│  └─────────────────────┬──────────────────────────────────┘     │
│                        ▼                                       │
│  ┌────────────────────────────────────────────────────┐     │
│  │  Stage 3: SFT + RLHF（运动一致性奖励）                 │     │
│  │  · 光流一致性评分模型作为奖励信号                            │     │
│  │  · 美学质量评分 + 提示跟随评分联合优化                       │     │
│  │  · 物理感知训练：重力、布料、流体先验融入 loss                  │     │
│  └─────────────────────┬──────────────────────────────────┘     │
│                        ▼                                       │
│  ┌────────────────────────────────────────────────────┐     │
│  │  推理：多模态输入 → 统一编码 → DiT 去噪 → 视频输出     │     │
│  │  · 文本/图像/音频/视频 → 双流提示编码器                  │     │
│  │  · 全帧时空注意力 → 逐帧解码 → 视频文件                  │     │
│  │  · 支持 1080p / 最长 ~20s                               │     │
│  └────────────────────────────────────────────────────┘     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 设计决策解剖 ⚖️

| 决策点 | 他们的选择 | 被放弃的方案 | 为什么 | 这个选择何时失效 |
|--------|----------|------------|--------|---------------|
| 模型架构 | 全时空 DiT（全帧注意力） | 滑动窗口注意力（如早期 Video LDM）、U-Net backbone | 全帧注意力让每一帧与其他所有帧直接交互，理论上建模长程时序关系能力最强。全帧注意力消除帧间语义漂移——滑动窗口中远端帧信息逐帧衰减 | 帧数增加时空注意力计算量 O(T²×H×W) 爆炸。当视频超过 30 秒（~900 帧）时全帧注意力的计算成本不可接受 |
| 音频集成 | 统一多模态音视频联合生成架构 | 视频生成后单独配音频 | 联合生成确保唇形同步、音效与动作对齐。独立配音频在时间对齐上有天然缺陷 | 联合生成增加了训练复杂度，音频质量可能受视频生成质量影响 |
| 训练策略 | 分辨率渐进训练 + 数据过滤 | 直接 1080p 端到端训练 | 直接高分辨率训练计算成本极高且运动语义学习困难。低分辨率先学运动先验，再逐步提升细节——类似课程学习 | 当分辨率差距过大（如 256p → 4K）时低分辨率的运动先验可能不完美迁移，需要额外中间步骤 |
| 评测体系 | SeedVideoBench-2.0 内部评测（多维度雷达图） | 仅使用 VBench / EvalCrafter 等公开基准 | 公开基准维度有限，不能覆盖产品级的多模态能力。自建雷达图可跟踪 T2V/I2V/多模态任务各自的多维表现 | 自建基准的外部可比性受限。雷达图维度权重的设定可能偏向已优化的维度 |

## 成本与量级 💰
- **训练成本量级**：未公开。从 DiT 架构 + 亿万级视频数据 + 1080p 渐进训练估算，需要至少数千 GPU·天（H100）。视频模型训练成本通常是同等规模 LLM 的 5-10 倍。
- **推理成本 vs 基线**：官方未公布。推理生成 20 秒 1080p 视频以当前 DiT 模型估计需要 2-10 分钟/clip（A100）。Seedance 2.0 Fast 变体通过优化管线实现约 3x 加速和约 91% 成本降低（$0.022/秒）。
- **我的产品要用**：通过火山引擎 API（Seedance 2.0 Fast 用于快速迭代，Seedance 2.0 用于终版渲染）。Fast 变体成本约 $0.022/秒输出视频，适合高吞吐量场景。国际用户可通过 dreamina.capcut.com 体验。

## 证据审计 🔬
- **实验设计公平吗？** 产品页面形式缺少正式论文的详细实验设计。评测使用 SeedVideoBench-2.0 内部基准，雷达图显示 vs 竞品（Sora/Kling/Gen-3）领先，但竞品的具体评分数值和测试设置未完全透明披露。
- **最强证据**：运动一致性和物体连贯性的用户感知评测（非公开数据），以及雷达图中各维度的领先位置。第三方评测（如 Sitepoint 对比文章）也确认了运动真实感的显著提升。
- **最可疑的数字**：雷达图中 Seedance 2.0 在"中文理解"维度上明确优于 Sora，但未控制提示复杂度——简单中文提示 vs 复杂英文提示差异大。此外 2026 年 3 月 ByteDance 暂停了 Seedance 2.0 的全球发布，可能暗示评测分数与区域实际体验存在差距。
- **最小复现实验**：在 VBench 公开数据集上生成 50 个相同 prompt 的视频，计算帧间 CLIP 一致性得分（Frame Consistency）和运动平滑度（Motion Smoothness），与竞品公开分数比较。预算：约 ¥500（火山引擎 API）+ 2 天评估。

## 可复用点（PM 决策）
- **何时采用**：需要视频生成能力的产品（广告创意、电商展示、短视频制作）；需要中英双语视频生成（中文场景 Seedance 有原生优势）；需要音频-视频同步生成。
- **何时规避**：需要 >30 秒的长视频（一致性下降）；需要实时或近实时生成（当前 2-10 分钟/ clip 延迟）；物理精确性要求极高（液体/布料/复杂交互仍有失真）。
- **供应商拷问清单**：① Fast 变体的质量 vs 标准版的具体差距（哪些场景不建议使用 Fast）？② 是否支持自定义光流一致性奖励模型微调（面向特定场景的垂直优化）？③ 视频生成后是否提供逐帧的 motion consistency score 用于自动质量过滤？

## 关联网络 🕸️
- 相关论文：*Sora视频生成模型*（未收录） — 同属 DiT 路线的视频生成模型，Sora 在叙事连贯性上更强，Seedance 在运动真实感上更强
- 相关概念：[[Wiki/概念/06_多模态生成/扩散模型]]
- 相关概念：[[Wiki/概念/06_多模态生成/视频生成技术]]
- 相关概念：[[Wiki/概念/06_多模态生成/扩散模型]] — DiT架构（全时空注意力），见"DiT / MMDiT 架构变体"小节

## 动手练习 💻（15-45分钟）
练习目标：写代码分析视频帧间一致性指标，模拟 Seedance 2.0 运动一致性奖励模型的核心逻辑。

```python
import numpy as np
from typing import List, Tuple
import math

# ============================================================
# 视频帧间一致性分析
# 模拟 Seedance 2.0 运动一致性奖励模型的核心逻辑
# ============================================================

def simulate_frames(num_frames: int = 30,
                    base_content: str = "static") -> List[np.ndarray]:
    """模拟生成视频帧（用特征向量代替像素，降低计算量）。
    
    参数：
        num_frames: 帧数
        base_content: "static" 静止场景, "smooth" 平滑运动, "erratic" 抖动
    
    返回：每帧的特征向量列表（128维，模拟 CLIP 图像特征）
    """
    np.random.seed(42)
    frames = []
    
    # 用一个 128 维向量模拟每帧的语义特征
    base = np.random.randn(128).astype(np.float32)
    base = base / np.linalg.norm(base)  # 单位向量
    
    for t in range(num_frames):
        if base_content == "static":
            # 静止场景：每帧几乎一样 + 微小噪声
            noise = np.random.randn(128).astype(np.float32) * 0.02
            frame_feat = base + noise
            
        elif base_content == "smooth":
            # 平滑运动：帧间逐渐变化（模拟物体缓慢移动）
            drift = np.sin(t / num_frames * np.pi * 2) * 0.1
            noise = np.random.randn(128).astype(np.float32) * 0.01
            frame_feat = base + drift * np.random.randn(128) / 10 + noise
            
        elif base_content == "erratic":
            # 抖动：帧间随机变化（模拟物体闪烁或突变）
            frame_feat = base + np.random.randn(128).astype(np.float32) * 0.3
            
        else:
            raise ValueError(f"未知场景：{base_content}")
        
        # 归一化到单位向量
        frame_feat = frame_feat / np.linalg.norm(frame_feat)
        frames.append(frame_feat)
    
    return frames


def frame_consistency_score(frames: List[np.ndarray]) -> float:
    """帧间一致性指标（Frame Consistency）。
    
    计算相邻帧之间的余弦相似度均值。
    高一致性 → 物体在帧间保持身份，不发生突然变化。
    
    公式：FC = (1/(N-1)) * Σ_{t=1}^{N-1} cos(f_t, f_{t+1})
    其中 N 为总帧数。
    
    Seedance 2.0 的实际实现更复杂（含光流约束和物体级追踪），
    这里用特征相似度近似。
    """
    N = len(frames)
    similarities = []
    
    for t in range(N - 1):
        # 余弦相似度
        sim = np.dot(frames[t], frames[t + 1]).item()
        similarities.append(sim)
    
    mean_sim = float(np.mean(similarities))
    min_sim = float(np.min(similarities))
    
    return {
        "mean_consistency": mean_sim,
        "min_consistency": min_sim,
        "std_consistency": float(np.std(similarities)),
        # 一致性评分：映射到 0-100 分，方便非技术观众理解
        "score": max(0, min(100, (mean_sim - 0.5) * 200)),
    }


def motion_smoothness_score(frames: List[np.ndarray]) -> float:
    """运动平滑度指标（Motion Smoothness）。
    
    计算帧间差分的二阶导数（加速度）的稳定性。
    平滑运动 → 帧间位移变化缓慢。
    抖动 → 帧间位移剧烈变化。
    
    用特征空间的"速度"（帧间差）和"加速度"（速度的变化）来评估。
    越低的分值表示越平滑（与 Seedance 2.0 的物理感知训练目标对应）。
    """
    N = len(frames)
    
    # "速度" = 帧间特征差
    velocities = []
    for t in range(N - 1):
        vel = frames[t + 1] - frames[t]
        velocities.append(np.linalg.norm(vel))
    
    # "加速度" = 速度的变化
    jerks = []
    for t in range(len(velocities) - 1):
        jerk = abs(velocities[t + 1] - velocities[t])
        jerks.append(jerk)
    
    mean_velocity = float(np.mean(velocities))
    mean_jerk = float(np.mean(jerks)) if jerks else 0.0
    
    # 平滑度评分：低平均速度 + 低加速度变化 = 平滑
    # 归一化到 0-100（100 最平滑）
    smoothness = max(0, min(100, 100 - (mean_velocity * 50 + mean_jerk * 100)))
    
    return {
        "mean_velocity": mean_velocity,
        "mean_jerk": mean_jerk,
        "smoothness_score": smoothness,
    }


def semantic_coherence_score(frames: List[np.ndarray],
                             prompt_embedding: np.ndarray) -> float:
    """语义连贯性（Prompt Adherence）：帧特征与 prompt 的匹配度。
    
    计算所有帧的平均 prompt 匹配度。高 → 视频内容与文本指令一致。
    
    Seedance 2.0 的双流提示编码器将文本和视觉编码到同一语义空间，
    这里简化为帧特征与 prompt 特征的余弦相似度。
    """
    frame_prompt_sims = []
    for feat in frames:
        sim = np.dot(feat, prompt_embedding).item()
        frame_prompt_sims.append(sim)
    
    return {
        "mean_prompt_alignment": float(np.mean(frame_prompt_sims)),
        "min_prompt_alignment": float(np.min(frame_prompt_sims)),
        "score": max(0, min(100, (np.mean(frame_prompt_sims) - 0.3) * 150)),
    }


def comprehensive_evaluation(frames: List[np.ndarray],
                              prompt_embedding: np.ndarray = None) -> dict:
    """综合评测：同时计算一致性 + 平滑度 + 语义匹配
    
    模拟 Seedance 2.0 的 SeedVideoBench-2.0 评估逻辑
    输出雷达图可映射到"运动一致性"
    """
    fc = frame_consistency_score(frames)
    ms = motion_smoothness_score(frames)
    
    result = {
        "frame_consistency": fc,
        "motion_smoothness": ms,
        "composite_quality": (fc["score"] + ms["smoothness_score"]) / 2,
    }
    
    if prompt_embedding is not None:
        sc = semantic_coherence_score(frames, prompt_embedding)
        result["semantic_coherence"] = sc
        result["composite_quality"] = (
            fc["score"] + ms["smoothness_score"] + sc["score"]
        ) / 3
    
    return result


# ============================================================
# 测试：对比三种视频类型的帧间一致性
# ============================================================
if __name__ == "__main__":
    # 模拟 prompt 特征（用于语义一致性评测）
    prompt_feat = np.random.randn(128).astype(np.float32)
    prompt_feat = prompt_feat / np.linalg.norm(prompt_feat)
    
    scenarios = ["static", "smooth", "erratic"]
    
    print(f"{'='*60}")
    print(f"Seedance 2.0 风格帧间一致性分析")
    print(f"{'='*60}")
    
    for scenario in scenarios:
        n_frames = 30
        frames = simulate_frames(n_frames, base_content=scenario)
        
        print(f"\n--- 场景: {scenario} ---")
        eval_result = comprehensive_evaluation(frames, prompt_feat)
        
        fc = eval_result["frame_consistency"]
        ms = eval_result["motion_smoothness"]
        sc = eval_result["semantic_coherence"]
        
        print(f"  帧间一致性:")
        print(f"    均值: {fc['mean_consistency']:.4f}")
        print(f"    最小值: {fc['min_consistency']:.4f}")
        print(f"    评分（0-100）: {fc['score']:.1f}")
        
        print(f"  运动平滑度:")
        print(f"    平均速度: {ms['mean_velocity']:.4f}")
        print(f"    平均抖动: {ms['mean_jerk']:.4f}")
        print(f"    评分（0-100）: {ms['smoothness_score']:.1f}")
        
        print(f"  语义连贯性:")
        print(f"    平均对齐: {sc['mean_prompt_alignment']:.4f}")
        print(f"    评分（0-100）: {sc['score']:.1f}")
        
        print(f"  ▶ 综合质量评分: {eval_result['composite_quality']:.1f}/100")
    
    # 结论：静止 > 平滑 > 抖动
    print(f"\n{'='*60}")
    print(f"结论：静止场景综合质量最高，平滑运动次之，抖动场景最低")
    print(f"Seedance 2.0 的运动一致性奖励模型会优先提升")
    print(f"'平滑运动'场景的质量，使其向'静止'场景靠拢")
    print(f"{'='*60}")
```

**逐行注释说明**：
- `simulate_frames`：用低维特征向量模拟视频帧，三种场景（静止/平滑/抖动）
- `frame_consistency_score`：相邻帧余弦相似度均值，越高表示帧间物体身份越一致
- `motion_smoothness_score`：对特征空间"速度"和"抖动"建模，模拟 Seedance 2.0 物理感知训练目标
- `semantic_coherence_score`：帧与 prompt 的对齐度，模拟提示跟随能力
- `comprehensive_evaluation`：综合评测，输出可映射到雷达图的多维评分
- 读者可以扩展：替换特征为真实 CLIP 嵌入、接入光流算法（OpenCV calcOpticalFlowFarneback）

## 自测三层 🎓
- **L1 复述**：Seedance 2.0 使用什么架构？核心训练策略是什么（列出三个阶段）？
- **L2 解释**：全帧时空注意力相比滑动窗口注意力有什么优缺点？为什么 Seedance 选择全帧注意力？
- **L3 应用**：你的团队正在为电商平台开发"商品展示视频自动生成"功能。商品以静态图片为主，需要生成 10-15 秒展示视频。你选择 Seedance 2.0 的哪个变体（标准/Fast）？在质量、成本和速度之间如何 trade-off？

📅 知识时间锚：2026-02-10（Seedance 2.0 发布日）
