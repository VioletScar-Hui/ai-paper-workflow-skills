---
tags: [论文笔记, 投机解码, 效率量化, 半自回归, 置信度调度, 骨干精读]
笔记层级: 骨干
paper_id: "142"
复核日期: 2026-07-04
---

# DSpark：置信度调度投机解码与半自回归生成

📄 **原文 PDF**：[[RAW/142 - DSpark - Confidence-Scheduled Speculative Decoding with Semi-Autoregressive Generation.pdf]]

## PM 速判（30秒）
> DeepSeek-AI 与北大联合提出的投机解码框架，用半自回归 draft（并行骨干 + 轻量序列头）解决并行 draft 的 token 间依赖缺失问题，再用置信度调度按请求动态裁剪验证长度。部署在 DeepSeek-V4 生产系统后，每用户生成速度提升 60-85%，且在高并发下不出现吞吐悬崖。PM 须知：这是行业内首个将"draft 质量"和"验证负载感知"统一优化的生产级加速方案，开源（DeepSpec 仓库），可直接复用于自建推理服务。

## 双层费曼 🗣️
> **给 CEO 的一句话**：把大模型生成提速 60-85%，同时让服务器在高峰期不卡顿——核心思路是不在所有 token 上花同样的力气，对模型有把握的段落大批量生成，对没把握的段落逐字谨慎处理。
>
> **给工程师的一段话**：DSpark 由两个互补组件构成。(1) 半自回归 draft：保留并行 draft backbone（单次前向生成所有候选 token）保证低延迟，附加轻量序列头（Markov 头或 RNN 头）注入局部 token 间依赖，缓解并行 draft 的"后缀衰变"（多模态碰撞，如同时采样出 "of problem"）。(2) 置信度调度验证：confidence head 逐位置估计前缀存活概率，经 Sequential Temperature Scaling (STS) 校准后，硬件感知调度器按当前引擎负载（SPS 曲线）动态决定每个请求的验证长度——低负载全验证，高负载只验证最有把握的前缀。在 Qwen3-14B 上对比 Eagle3 接受长度提升 30.0%，对比 DFlash 提升 18.3%。生产部署 DeepSeek-V4 后 60-85% 速度提升，且消除了固定长度投机解码在高并发下的吞吐悬崖。

## 问题域定位 🎯
- **根本约束**：投机解码的加速比受制于三个杠杆——draft 延迟、接受长度、验证开销，三者互相牵制。并行 draft 压低 draft 延迟但牺牲接受长度（后缀衰变），固定长度验证在高并发下浪费 batch 容量。
- **之前卡在哪**：并行 drafter（Medusa、DFlash）虽能一次生成 16+ 候选 token，但因无 token 间依赖，长块尾部接受率暴跌；同时现有工作均忽略"验证低质量 token 在高并发下的机会成本"——验证一个必然被拒的 token 所消耗的算力本可用于服务其他活跃请求。
- **开启的技术路线**：首次将"draft 质量"与"系统负载"联合建模，打开了投机解码从"硬件无关优化"走向"负载感知优化"的大门。后续工作可沿置信度估计校准、调度策略学习、以及半自回归架构搜索三个方向深入。

## 核心机制

```
                                          ┌─────────────────────────────┐
                                          │   Target Model 推理引擎      │
                                          │  (负载感知 SPS 曲线)          │
                                          └──────────┬──────────────────┘
                                                     │ 实时吞吐 profile
┌──────────┐    ┌─────────────────────────────┐      │
│ 当前 token│───▶│   Step 1: 半自回归 Draft     │      │
│  (anchor) │    │                             │      │
└──────────┘    │  ┌─────────────────────┐    │      │
                │  │ Parallel Backbone    │    │      │
                │  │ (DFlash variant,     │    │      │
                │  │  单次前向, 共享嵌入)  │    │      │
                │  └──────────┬──────────┘    │      │
                │             ▼               │      │
                │  ┌─────────────────────┐    │      │
                │  │ Sequential Head      │    │      │
                │  │ (Markov / RNN)       │    │      │
                │  │ 注入 token 间依赖    │    │      │
                │  └──────────┬──────────┘    │      │
                │             ▼               │      │
                │  draft tokens + hidden      │      │
                └─────────────┬───────────────┘      │
                              │                      │
                              ▼                      │
                ┌─────────────────────────────┐      │
                │   Step 2: Confidence Head    │      │
                │   σ(W·[h_t; emb(x_{t-1})])   │◀─────┤
                │   → 逐位置存活概率 p_t       │      │
                └──────────────┬──────────────┘      │
                              │                      │
                              ▼                      │
                ┌─────────────────────────────┐      │
                │   Step 3: Hardware-Aware     │──────┘
                │   Prefix Scheduler           │
                │   按 p_t 和引擎负载裁剪前缀   │
                │   保留 EFG, 丢弃 H           │
                └──────────────┬──────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │   Step 4: Target 并行验证     │
                │   → 接受 E, F; 拒绝 G        │
                │   → 从 target 采样修正 G*    │
                └─────────────────────────────┘
```

**数据流**：anchor token → parallel backbone 生成所有候选 hidden → sequential head 逐位置注入依赖并采样 → confidence head 估存活概率 → scheduler 裁剪 → target 并行验证。

## 设计决策解剖 ⚖️

| 决策点 | 他们的选择 | 被放弃的方案 | 为什么 | 这个选择何时失效 |
|--------|----------|------------|--------|---------------|
| Draft 架构 | 半自回归（并行 backbone + 轻量序列头） | 纯并行（Medusa/DFlash）或纯自回归（EAGLE3） | 纯并行无 token 间依赖→后缀衰变；纯自回归 draft 延迟随块长线性增长。半自回归在几乎不增加延迟的前提下提升接受长度 16-30% | 序列头本身仍然是顺序采样，当块长极大（>32）时序列头延迟占比上升，优势被侵蚀 |
| 置信度估计 | 独立 confidence head + STS 校准 | 直接用 target softmax 熵（零开销）或阈值启发式 | 熵是间接信号，神经估计有偏差。confidence head 直接拟合每步接受率（TV 距离），STS 校准绝对值使调度器能做精确吞吐-收益计算 | 分布外任务下 confidence head 本身的校准可能失效；STS 依赖 hold-out 验证集 |
| 调度策略 | 贪心排序 + 吞吐模型选最优 | 固定长度（γ=4/8/16）、或贪心直到置信度低于阈值 | 固定长度无视负载变化；阈值策略不考虑验证的机会成本。贪心排序 + 吞吐曲线（SPS）在有 batch 维度时接近全局最优 | 实时 SPS 曲线估计有延迟时可能调度次优；极端 burst 下所有请求都被裁到极短前缀，流水线气泡增加 |

## 成本与量级 💰
- **训练成本量级**：需要额外训练 parallel backbone（基于 DFlash 修改）、sequential head（Markov 或 RNN）、confidence head。训练数据与 target model 一致（通用语料），以 Qwen3-14B 规模估算约需 500-1000 GPU·时。
- **推理成本 vs 基线**：draft 阶段 vs 固定 γ 基线增加约 5-10% overhead（序列头 + confidence head 的前向），但验证阶段大幅减少被拒 token 验证。端到端：在 DeepSeek-V4 生产环境下每用户速度提升 60-85%。
- **我的产品要用**：最小可行配置——基于现有投机解码实现（如 Medusa 或 EAGLE），附加一个轻量 confidence head（单层线性 + sigmoid）和固定阈值裁剪（γ_t = max(1, floor(γ_max · c_t))），无需完整调度器即可获得 5-15% 额外加速。

## 证据审计 🔬
- **实验设计公平吗？** 总体公平。控制实验在同一 target model（Qwen3-4B/8B/14B）下比较所有 drafter（Eagle3、DFlash、DSpark），且评估覆盖了数学、代码、对话多领域。唯一可能有偏的是 sequential head 的参数量——论文未披露序列头的精确参数量 vs backbone 的比例。
- **最强证据**：Qwen3-14B 上 macro-average accepted length 30.0% 优于 Eagle3、18.3% 优于 DFlash。置信区间合理，跨 3 个模型规模趋势一致。
- **最可疑的数字**：生产部署 "60-85% per-user speed improvement" —— 该数字未给出置信区间，且生产环境对比基线是 MTP-1（非 DFlash 或 Eagle3），不是最先进的对比。此外生产环境存在许多不可控变量（请求混合、缓存命中率、网络延迟）。
- **最小复现实验**：在 NVIDIA A100 上基于开源 DeepSpec 仓库复现 Qwen3-4B 离线评测。代理指标：accepted length / block size ratio。预算：约 50 GPU·时 + 50 美元云服务费。

## 可复用点（PM 决策）
- **何时采用**：推理服务在 100+ QPS 高并发下吞吐瓶颈明显；当前使用固定长度投机解码且观察到高并发时有效加速比下降。
- **何时规避**：draft model 本身已接近 target model 质量（无需改进 draft）；低并发场景（验证机会成本忽略不计）；模型迭代频繁导致 confidence head 需频繁重训。
- **供应商拷问清单**：① confidence head 校准频率？是否支持在线校准？② 调度器是否感知请求优先级（SLA 分层）？③ 序列头参数是否与 backbone 一起更新？重训练成本多少？

## 关联网络 🕸️
- 相关论文：*EAGLE投机解码*（未收录） — DSpark 的序列头借鉴 EAGLE 的 draft token 间依赖建模思路
- 相关论文：*Medusa多头解码*（未收录） — 纯并行 drafter 的代表，DSpark 的对比基线和 backbone 起点
- 相关概念：[[Wiki/概念/01_架构技术/投机解码]]
- 相关概念：[[Wiki/概念/01_架构技术/投机解码]] — 半自回归生成（Semi-Autoregressive Draft）机制，见"半自回归生成（DSpark，论文142深化）"小节
- **冲突/印证**：与 EAGLE3 实证互补——EAGLE3 证明深自回归 draft 的高质量，DSpark 证明用更轻量的序列头可以达到接近质量但大幅降低成本。

## 动手练习 💻（15-45分钟）
练习目标：用 numpy 实现 DSpark 的置信度调度 draft 长度选择逻辑（confidence head → scheduler 核心）。
核心逻辑：给定一组逐位置置信度分数，结合引擎负载曲线，选择使吞吐最优的验证长度。

```python
import numpy as np
from typing import List

# ============================================================
# DSpark 置信度调度 draft 长度选择逻辑
# ============================================================

def simulate_acceptance(confidence: float, num_samples: int = 1000) -> bool:
    """模拟单步接受/拒绝过程。
    真实场景下这取决于 draft/target 分布的 TV 距离，
    这里用 confidence 作为接受概率（经 STS 校准后近似成立）。"""
    return np.random.random() < confidence

def compute_throughput(prefix_len: int, batch_size_estimate: int,
                       sps_curve: dict) -> float:
    """硬件感知调度器的核心：给定当前验证长度和 batch 大小，
    预测每秒生成的 token 数（SPS）。
    
    sps_curve = {
        "idle": [sps_at_len_1, sps_at_len_2, ...],   # 空闲时
        "loaded": [sps_at_len_1, sps_at_len_2, ...],  # 负载高时
    }
    """
    # 随 prefix 长度增加，SPS 先升后降（过长时被拒 token 浪费算力）
    if batch_size_estimate < 16:
        curve = np.array(sps_curve["idle"])
    else:
        curve = np.array(sps_curve["loaded"])
    idx = min(prefix_len - 1, len(curve) - 1)
    return curve[idx]

def select_verification_length(
    confidences: List[float],
    batch_size_estimate: int,
    max_draft_len: int = 8,
    sps_curve: dict = None,
) -> int:
    """DSpark 硬件感知调度器的简化版本。
    
    输入：
        confidences: confidence head 输出的逐位置存活概率 [p1, p2, ..., pN]
        batch_size_estimate: 当前 engine 排队的请求数（负载信号）
        max_draft_len: 最大 draft 长度
        sps_curve: 引擎 SPS 曲线
    
    输出：
        selected_len: 被选中的验证长度 γ_t
    
    思想：对每个候选长度 k ∈ [1, max_draft_len]，
    计算 expected_accepted_tokens = sum(p1 + p2 + ... + pk)
    然后 expected_throughput = expected_accepted_tokens / (draft_latency + verify_latency)
    选使 throughput 最大的 k。
    """
    if sps_curve is None:
        sps_curve = {
            "idle": [8, 12, 14, 15, 14, 13, 11, 9],
            "loaded": [8, 11, 12, 11, 9, 7, 5, 3],
        }
    
    # 1) 取前 max_draft_len 个 confidence 值
    probs = np.array(confidences[:max_draft_len])
    assert len(probs) == max_draft_len, f"需要 {max_draft_len} 个confidence值，传入 {len(probs)}"
    
    # 2) 对每个候选长度 k，计算累计期望接受数
    #    注意：真实场景中一旦某个位置被拒，后面全部丢弃
    #    因此前缀生存概率 = Π(直到该位置所有接受概率)
    #    期望接受数 = Σ(survival_prob_at_i)
    best_len, best_tp = 1, 0.0
    
    for k in range(1, max_draft_len + 1):
        # 前缀生存概率 = 前 k 个全部接受的概率 ≈ Π(p_i)
        # 更精确：第 i 个位置被接受的期望 = Π_{j=1}^{i} p_j
        prefix_probs = probs[:k]
        survival = np.cumprod(prefix_probs)  # [p1, p1*p2, p1*p2*p3, ...]
        expected_accepted = survival.sum()
        
        # 计算 throughput：expected_accepted / (draft时间 + 验证时间)
        # 验证时间由 prefix_len 和当前负载决定
        verify_latency = 1.0 / compute_throughput(k, batch_size_estimate, sps_curve)
        draft_latency = 0.005 * k  # 简化：draft 时间 ≈ 0.5ms/token
        total_latency = draft_latency + verify_latency
        
        tp = expected_accepted / total_latency
        
        if tp > best_tp:
            best_tp = tp
            best_len = k
    
    return best_len

# ============================================================
# 测试
# ============================================================
if __name__ == "__main__":
    np.random.seed(42)
    
    # 场景1：高置信度（如代码生成）
    high_conf = [0.95, 0.93, 0.90, 0.88, 0.85, 0.82, 0.78, 0.75]
    batch = 8  # 轻度负载
    
    print("=== 高置信度场景（代码生成），轻负载 ===")
    for b in [4, 16, 64]:
        length = select_verification_length(high_conf, batch_size_estimate=b)
        print(f"  batch≈{b:3d} → 选中验证长度 γ_t = {length}")
    
    # 场景2：低置信度（如开放对话）
    low_conf = [0.75, 0.60, 0.45, 0.35, 0.28, 0.22, 0.18, 0.15]
    
    print("\n=== 低置信度场景（开放对话），轻负载 ===")
    for b in [4, 16, 64]:
        length = select_verification_length(low_conf, batch_size_estimate=b)
        print(f"  batch≈{b:3d} → 选中验证长度 γ_t = {length}")
    
    # 场景3：高负载下置信度调度效果（骨架，可自扩展）
    print("\n=== 高负载时的调度行为比较 ===")
    for conf_list, label in [(high_conf, "高置信度"), (low_conf, "低置信度")]:
        idle_len = select_verification_length(conf_list, batch_size_estimate=4)
        busy_len = select_verification_length(conf_list, batch_size_estimate=64)
        print(f"  {label}: 空闲时 γ={idle_len}, 繁忙时 γ={busy_len} "
              f"({'裁减' if busy_len < idle_len else '不变'})")
```

**逐行注释说明**：
- `simulate_acceptance`：模拟单步接受/拒绝，用于后续扩展完整仿真
- `compute_throughput`：模拟不同负载下的 SPS 曲线，体现"验证低质量 token 在繁忙时的机会成本"
- `select_verification_length`：核心调度函数，对每个候选长度 k 计算期望吞吐，选择最优
- `cumprod`（累积乘积）是关键操作：第 i 个位置的实际生存概率依赖前面所有位置全部被接受
- 读者可以自行扩展：替换 SPS 曲线为真实 profiling 数据，或用 Monte Carlo 仿真替代确定性计算

## 自测三层 🎓
- **L1 复述**：DSpark 的两个核心组件分别解决什么问题？半自回归架构的"半"体现在哪个维度？
- **L2 解释**：为什么并行 drafter 会产生"多模态碰撞"（如 "of problem"）？Markov head 如何缓解？如果不缓解，对端到端加速比的影响有多大？
- **L3 应用**：你的团队正在为电商客服搭建 LLM 推理服务，峰值 QPS 高达 500，目前使用 EAGLE 固定 γ=5 的投机解码。你观察到峰值时加速比从 2.5x 降到 1.3x。你认为最可能的原因是什么？按照 DSpark 的思路你会做哪两项最小改动来验证？

📅 知识时间锚：2026-06-16（DSpark arXiv 发布日）
