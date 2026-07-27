---
tags: [论文笔记, 前沿模型, ByteDance, Seeduplex, 双模型协作, 路由架构, 推理效率, 2026]
paper_id: "147"
笔记层级: 骨干
复核日期: 2026-07-04
成熟度: ✅
---

# Seeduplex — 字节跳动双模型协作框架：小模型快 + 大模型强

📄 **原文 PDF**：[[RAW/147 - Seeduplex - ByteDance Seed Official Model Page.pdf]]

## PM 速判（30秒）> 一句话
字节跳动提出的双模型路由架构：轻量快速模型处理简单请求、大推理模型处理复杂任务，通过智能路由层动态分配，在一个系统中同时实现低延迟（-60%）和高质量（不降分）。

## 双层费曼 🗣️

> **给CEO**：想象一个客服团队——实习生处理简单问题（秒回），遇到疑难杂症无感转接给专家。用户完全感觉不到切换，因为实习生和专家共享同一份"聊天记录"。Seeduplex 简单问题走小模型（500ms 内响应），复杂问题路由到大模型，整体计算成本降 40%，用户满意度不降。字节已经在内部生产流量中跑通了。

> **给工程师**：双模型服务架构，核心三层——(1) **复杂度评分器**：基于 token 长度、历史轮次、任务类型 NER 计算复杂度分数；(2) **智能路由层**：按分数阈值决策，低于阈值走快速模型（1-3B），高于阈值走 Full 模型（100B+），且支持渐进式升级（先快速响应首 token，检测到超阈值后无感切换）；(3) **上下文共享**：两模型共享 KV Cache 空间，避免切换时上下文丢失。路由阈值通过 contextual bandit 在线学习动态调整。

## 问题域定位 🎯

| 维度 | 说明 |
|------|------|
| **根本约束** | 单一 LLM 无法同时满足高并发简单请求的低延迟（P50<500ms）和复杂推理任务的高质量——这是服务级的速度-质量权衡，与模型级不同 |
| **之前卡点** | (1) 维护两套 API 端点客户端自行选择，切换有损；(2) MoE 模型内部路由无法解耦推理成本；(3) FrugalGPT 在调用层路由但缺少共享上下文和渐进式升级 |
| **开启的路线** | 双模型 + 智能路由 + 上下文共享 + 渐进式升级的四合一方案，重新定义 LLM 服务架构的延迟-质量 Pareto 前沿 |
| **关闭的路线** | 一刀切单一模型在服务场景不可最优；纯客户端路由因缺乏共享上下文和渐进式升级，体验劣于服务端路由 |

## 核心机制（ASCII图）

```
                    ┌─────────────────────────────┐
                    │       用户请求                │
                    │ "解释量子纠缠" vs "写营销方案" │
                    └─────────────┬───────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────┐
│                   智能路由层                              │
│  ┌─────────────────────────────────────────────────┐   │
│  │         复杂度评分器（Complexity Scorer）          │   │
│  │  特征: {token_len, 历史轮次, 任务类型}            │   │
│  │  公式: score = Σ w_i · f_i(x)                    │   │
│  │  权重 w_i 通过 contextual bandit 在线学习更新      │   │
│  └────────┬──────────────────────────┬──────────────┘   │
│           │ score < 阈值            │ score ≥ 阈值      │
│           ▼                          ▼                  │
│  ┌────────────────────┐  ┌──────────────────────┐      │
│  │ 路径 A: 快速模型    │  │ 路径 B: 大推理模型   │      │
│  │ 轻量 LLM (1-3B)    │  │ Full LLM (100B+)    │      │
│  │ 延迟 < 500ms       │  │ 延迟 2-8s           │      │
│  └────────┬───────────┘  └────────┬─────────────┘      │
│           └──────────┬────────────┘                      │
│                      ▼                                   │
│  ┌─────────────────────────────────────────────────┐   │
│  │        上下文共享层（KV Cache）                     │   │
│  │  两模型共享 KV Cache，切换时不丢失对话历史         │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                              ▼
                    ┌─────────────────────────┐
                    │       最终响应           │
                    └─────────────────────────┘
```

## 设计决策解剖 ⚖️

| 决策点 | 选择 | 被放弃方案 | 为什么 | 何时失效 |
|--------|------|-----------|--------|---------|
| 路由位置 | 服务端路由 | 客户端路由 | 客户端路由无法实现无感切换和共享上下文；服务端可统一控制延迟预算 | 客户端有强隐私需求（端侧路由不可泄露请求内容）时不可用 |
| 复杂度评分 | 特征加权评分 + Contextual Bandit 调参 | LLM-as-Judge 语义评分 | LLM-as-Judge 评分本身需调用 LLM，违背低延迟初衷；特征评分 µs 级完成 | 遇分布外新型复杂任务时，特征评分可能严重低估复杂度 |
| 模型关系 | 异构模型共享 KV Cache | 同架构不同尺寸 | 异构模型同等质量下成本更低（可组合不同系列） | tokenizer 和 hidden_size 不一致时 KV Cache 共享实现更复杂 |
| 切换策略 | 渐进式升级（先快速响应后切换） | 路由判断后再调用（串行） | 先快速响应首 token 降低等待感知，切换发生在解码过程中 | 快速模型输出的首段可能被大模型覆盖，浪费算力 |
| 阈值调整 | Contextual Bandit 在线学习 | 固定阈值 / A/B 测试分批 | 在线学习可适应流量分布变化（白天简单对话多，夜间复杂分析多） | Bandit 冷启动收敛前可能频繁误判，需 warm-start |

## 成本与量级 💰

| 指标 | 数值 |
|------|------|
| 系统组成 | 快速模型 1-3B + 大推理模型 100B+ + 路由层 + 共享 KV Cache 服务 |
| 延迟改善 | P50 降低 ~60%（与纯大模型对比） |
| 计算成本降低 | ~40% |
| 路由层开销 | < 5ms（特征提取 + 评分 + 决策） |
| 切换感知时间 | < 50ms（终端用户无感） |
| 路由准确率目标 | 假阴性（复杂→小模型）< 1%，假阳性（简单→大模型）< 10% |

## 证据审计 🔬

| 证据类型 | 内容 | 可信度 |
|---------|------|--------|
| **最强证据** | 保持纯大模型相近质量前提下，P50 延迟降 60%，计算成本降 40% | ★★★★☆ — 内部高流量 A/B 测试，数据来自真实流量，统计效力充足；但"主观评分不降"需披露评分者间信度 |
| **最可疑数字** | "用户满意度不低于纯大模型"——渐进式升级后质量不连贯（风格断裂）可能被平均效应掩盖 |
| **实验公平性** | 对比了单独快速模型、单独大模型、用户手动选择三种方案，缺少与 FrugalGPT 和 RouteLLM 的外部公开对比 |
| **审稿补充** | 需要补充：(1) 路由误差 task-level breakdown；(2) 不同负载下延迟 P50/P95/P99 分布；(3) Contextual Bandit 对抗稳定性 |
| **最小复现设计** | 取 Qwen2.5-1.5B + Qwen2.5-72B，HuggingFace Transformers 搭建；复杂度评分用 token 数 + 关键词匹配；阈值用 Bayesian Optimization |

## 可复用点 + 供应商拷问清单

**可复用点：**
1. **复杂度评分特征**：token 长度 + 历史轮次 + 任务类型 NER 的轻量组合可移植到任何模型路由系统
2. **渐进式升级**：首 token 快速响应 + 无感切换大模型，对实时聊天/客服类产品极有价值
3. **Contextual Bandit 调阈值**：将路由决策建模为多臂赌博机，在延迟-质量 Pareto 前沿上自动平衡

**供应商拷问清单：**
- [ ] 客户能否自行设定延迟/质量的优先级权重？
- [ ] 渐进式升级下快速模型已输出 token 会丢弃还是保留？拼接处平滑处理？
- [ ] 两模型使用同一 tokenizer 吗？hidden_size 不同时 KV Cache 怎么映射？
- [ ] Contextual Bandit 冷启动时间多长？有防止早期误判的保护机制吗？
- [ ] 大模型宕机时降级为纯小模型模式还是拒绝服务？

## 关联网络 🕸️

- [[Wiki/论文笔记/02_前沿模型报告/144_Seed2.1字节跳动Agentic生产力模型]] — Seed 系列基础模型，Seeduplex 是生产部署的服务架构补充
- [[Wiki/概念/04_Agent框架/模型路由与CAF循环]] — Seeduplex 是模型路由的具体工程实现典范，CAF 循环思路一致
- [[Wiki/论文笔记/02_前沿模型报告/83_DeepSeek-V3.2开放大模型新前沿]] — DeepSeek V3 的 MoE 与 Seeduplex 的"服务层 MoE"形成模型内 vs 系统级路由对比
- [[Wiki/概念/01_架构技术/MoE混合专家架构]] — Seeduplex 可视为服务层面 MoE：Expert 是独立模型，Router 是复杂度评分决策器
- [[Wiki/概念/01_架构技术/量化技术]] — 量化 + 双模型路由可进一步叠加
- **冲突/印证**：与 FrugalGPT（Chen 2023）对比——FrugalGPT 在 API 调用层路由且无共享上下文，更适合第三方 API 聚合；Seeduplex 自有模型+共享上下文的方案在体验一致性上更优但灵活性更低。路由策略的选择取决于是否控制模型权重。

## 动手练习 💻

```python
"""练习：模拟 Seeduplex 双模型路由决策，在不同请求分布下评估延迟-质量权衡。"""

import numpy as np
from typing import List, Tuple

class ComplexityScorer:
    """复杂度评分：基于 token 数、对话轮次、任务类型计算 [0,1] 分数"""
    def __init__(self, weights: dict = None):
        self.w = weights or {'token': 0.4, 'turns': 0.3, 'type': 0.3}
        # 任务类型 → 复杂度映射
        self.type_map = {'simple_qa': 0.1, 'translation': 0.3, 'summarization': 0.4,
                         'code': 0.6, 'analysis': 0.7, 'creative': 0.8, 'reasoning': 0.9}

    def score(self, token_count: int, history_turns: int, task_type: str) -> float:
        f_tok = min(token_count / 4096, 1.0)
        f_turn = min(history_turns / 20, 1.0)
        f_type = self.type_map.get(task_type, 0.5)
        return min(self.w['token'] * f_tok + self.w['turns'] * f_turn + self.w['type'] * f_type, 1.0)


class Router:
    """路由决策：基于阈值决定 fast / progressive / full"""
    def __init__(self, threshold: float = 0.5):
        self.threshold = threshold
        self.scorer = ComplexityScorer()

    def decide(self, token_c: int, turns: int, task: str) -> Tuple[str, float]:
        s = self.scorer.score(token_c, turns, task)
        if s < self.threshold * 0.8: return ('fast', s)
        elif s < self.threshold:      return ('progressive', s)
        else:                         return ('full', s)


def simulate(requests: List[dict], router: Router):
    """运行模拟：统计延迟、质量、路由分布"""
    delay_map = {'fast': 0.3, 'full': 3.0, 'progressive': 0.5}
    latencies, qualities, routes = [], [], {'fast': 0, 'full': 0, 'progressive': 0}

    for req in requests:
        route, _ = router.decide(req['tok'], req['turns'], req['task'])
        routes[route] += 1
        latencies.append(delay_map[route])
        # fast 模型质量随真实复杂度下降，full 保持 0.9
        q = 0.7 * req['true_cplx'] + 0.3 if route == 'fast' else 0.9
        qualities.append(q)

    print(f"路由: fast={routes['fast']/len(requests)*100:.0f}% "
          f"progressive={routes['progressive']/len(requests)*100:.0f}% "
          f"full={routes['full']/len(requests)*100:.0f}%")
    print(f"延迟: avg={np.mean(latencies):.2f}s  P50={np.percentile(latencies,50):.2f}s  "
          f"P99={np.percentile(latencies,99):.2f}s")
    print(f"质量: avg={np.mean(qualities):.3f}  纯大模型对比={np.mean(qualities)/0.9*100:.1f}%")


if __name__ == '__main__':
    np.random.seed(42)
    # 生成 1000 个请求：80% 简单 / 20% 复杂
    reqs = []
    for _ in range(1000):
        if np.random.random() < 0.8:
            reqs.append({'tok': int(np.random.exponential(200)+10), 'turns': int(np.random.poisson(2)),
                         'task': np.random.choice(['simple_qa','translation','summarization']),
                         'true_cplx': np.random.beta(2, 5)})
        else:
            reqs.append({'tok': int(np.random.exponential(500)+200), 'turns': int(np.random.poisson(5)),
                         'task': np.random.choice(['analysis','code','reasoning','creative']),
                         'true_cplx': np.random.beta(5, 2)})

    for thresh in [0.3, 0.5, 0.7]:
        print(f"\n—— 阈值 = {thresh} ——")
        simulate(reqs, Router(threshold=thresh))
```

## 自测三层 🎓

**L1 — 记忆与理解：**
1. Seeduplex 的复杂度评分器使用了哪几类特征？权重如何调整？
2. "渐进式升级"是什么意思？为什么它能提升用户体验？
3. 两模型之间的上下文共享通过什么机制实现？解决了什么问题？

**L2 — 分析与比较：**
1. 对比 Seeduplex（服务端路由）和 FrugalGPT（客户端路由）的设计差异。什么场景下客户端路由反而更优？
2. 路由阈值的 Contextual Bandit 在线学习与固定阈值的 Tradeoff 是什么？冷启动如何保护用户体验？
3. 渐进式升级中快速模型输出的首段内容最终被大模型覆盖——这浪费了多少计算量？能否将快速模型输出作为大模型的条件输入（类似 speculative decoding）？

**L3 — 应用与迁移：**
1. 设计一个三模型路由系统（小/中/大），复杂度评分器需新增哪些特征才能使三个层级有意义？
2. 如果要将 Seeduplex 应用到 B2B API 产品（各客户有不同延迟/质量 SLA），路由层需要扩展什么能力？
3. 安全考虑：攻击者可能构造"语义简单但实际恶意"的请求绕过路由直接命中大模型。如何增强路由层安全性？

📅 **知识时间锚：2026-06 字节跳动 Seeduplex 发布，同期业内关注 FrugalGPT（Chen 2023）和 RouteLLM，但 Seeduplex 是首个在生产流量验证的双模型共享上下文方案。**