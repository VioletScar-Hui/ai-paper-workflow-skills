---
tags: [论文笔记, 长上下文推理, 证据回放, 注意力机制, 训练无关方法, 联想记忆, Harness, UIUC]
paper_id: "173"
filename: "173 - ReContext - Recursive Evidence Replay as LLM Harness for Long-Context Reasoning.pdf"
authors: "Yanjun Zhao*、Ruizhong Qiu*、Tianxin Wei*（*共同一作）、Yuanchen Bei、Zhining Liu、Lingjie Chen、Ismini Lourentzou、Hanghang Tong、Jingrui He†（†通讯作者）—— University of Illinois Urbana-Champaign (UIUC)"
year: 2026
笔记层级: 骨干
成熟度: 🧪
复核日期: 2026-07-15
---

# ReContext：递归证据回放作为长上下文推理的 LLM Harness

📄 **原文 PDF**：[[RAW/173 - ReContext - Recursive Evidence Replay as LLM Harness for Long-Context Reasoning.pdf]]

## PM 速判（30秒）
> 不训练、不改模型架构、不外挂检索库，只借用模型自己做一次前向传播时产生的 attention 信号，把 128K 上下文里最相关的约 128 个 token 揪出来、还原成完整句子，在正式回答前"重播"一遍给模型看——在 8 个 128K 长文本数据集 × 3 个开源模型（Qwen3-4B/8B、Llama3.1-8B）上，把平均 Acc 从 0.24 拉到 0.30（相对提升 24.6%，我已按 Table 1 逐格核算复现），且在全部 8 个基准×3 模型上取得最佳平均排名。PM 要关注的是：这是一个可以直接挂到已有开源模型长上下文链路上的"免训练插件"，但代价是必须能拿到模型内部 attention（闭源 API 用不了，论文 Limitations 明确写死），且它优化的是"把已有上下文里的证据榨出来"，不是扩窗口也不是省 token。

## 双层费曼 🗣️
> **给CEO的一句话**：你的 AI 读了 100 页合同/病历/工单，问它一个具体问题时，答案明明就在文档里，它却常常"读了但没用上"、编出别的答案。这篇论文让 AI 在正式回答前，先把它自己认为最相关的几句话挑出来抄一遍、摆在问题前面重新看一遍，再回答——不用重新训练、不用另接检索系统，代价是每次问答要多花约 40% 的推理时间。
>
> **给工程师的一段话**：ReContext 用 prompt 后缀最后 w=8 个 token 作为 query cue，对选定的 layer-head 集合 H 聚合 attention 并按指数衰减（λ=0.75）累积得到相关性分数 r_i，只在原始上下文 token 位置 I_C 内取 Top-K，再把命中 token 映射回其所在完整句子，拼接成证据池 φ(E) 插回 [C; φ(E); q] 重新做前向；该过程递归 R 轮（主实验 R=2），每轮的候选证据只增量追加、不覆盖，工程实现上会 snapshot 原文末尾的 KV 缓存并在 replay 阶段复用，只对新增证据 + 问题部分做增量前向。

## 问题域定位 🎯
- **根本约束**：context access ≠ context utilization——即便上下文窗口做到 128K，模型也不必然会用上已经在输入里的证据（论文引用 Liu et al. 2023a 的 "lost in the middle" 现象作为佐证）。
- **之前卡在哪**：三条既有路线各有代价——① attention 干预类（AttnSharp、DySCO）直接改写 decoding 期间的 attention logits，具有侵入性，且被强调的证据始终留在 latent attention 里，不会以可读文本形式暴露；② 检索/外部记忆类（RAG、A-MEM）额外接一个检索系统，引入检索噪声；③ 压缩类（LLMLingua、DAC）直接缩短有效输入，可能丢掉细粒度信息，在复杂 multi-hop 任务上不稳定（2.1 节、Appendix A）。
- **ReContext 开启的路线**：把"证据组织"和"答案生成"彻底解耦成两个独立推理阶段，全程不裁剪、不压缩原文，只在 prompt 层新增一段"证据脚手架"；不训练、不接外部系统、不做 decoding 期间的 logit 级干预，标准 Transformer 即可用（无需 SSM 等特殊架构）。
- **未解决/关闭的路线**：需要模型开放内部 attention 信号，闭源 API（GPT/Claude/Gemini 等不暴露 attention 权重）无法直接使用（Limitations 明确写明）；也没有解决"证据 selection 本身可能选错"的问题——它做的是把 attention 信号从"隐性影响"变成"显性文本"，而不是提升 attention 本身的准确度。

## 核心机制

```
┌─────────────────────────── 输入 ───────────────────────────┐
│  长上下文 C（128K tokens）              问题 q                │
│  "...LJM 是 Andrew Fastow 用来...的公司..."  "LJM是什么类型的公司?"│
└───────────────────────────┬─────────────────────────────────┘
                             │
      ┌──────────────────────▼──────────────────────┐
      │   ① 证据甄别 Evidence Sifting Pipeline          │
      │                                                │
      │  · 冻结参数 LLM 做一次"部分前向传播"（不生成，只读 attention）│
      │  · 取 prompt 后缀最后 w=8 个 token 作为 cue，       │
      │    对选定 layer-head 集合 H 聚合 attention：        │
      │    a_i^(u) = (1/|H|) Σ_(l,h)∈H A^(l,h)_{tu,i}     │
      │  · 跨 cue 位置指数衰减累积 (λ=0.75) → 相关性分数 r_i │
      │  · 只在原始上下文位置 I_C 内取 Top-K：              │
      │    P = TopK_{i∈I_C}(r_i; K)                       │
      │  · 证据物化：token 位置 → 映射回所在完整句子         │
      │    "240...ghost-entities..." "419...LJM是..."      │
      └──────────────────────┬──────────────────────┘
                             │ 证据池 E（有序、增量追加、不覆盖）
      ┌──────────────────────▼──────────────────────┐
      │   ② 递归证据回放 Recursive Replay                │
      │                                                │
      │  Round1: E⁽⁰⁾=∅         → 打分 → 新增 E₁          │
      │  Round2: [E₁]           → 打分 → 新增 E₂          │
      │  Round3: [E₁,E₂]        → 打分 → 新增 E₃          │
      │  ……（R 轮，主实验 R=2，小且固定，非开放式推理循环）    │
      │                                                │
      │  每轮重新读 x^(j-1)=[C; φ(E^(j-1)); q]             │
      │  → 已回放的证据会改变下一轮 cue token 的隐状态，      │
      │    从而"牵引"出与已选证据关联的新证据                │
      └──────────────────────┬──────────────────────┘
                             │
      ┌──────────────────────▼──────────────────────┐
      │   ③ 最终生成：x^(R) = [C; φ(E^(R)); q]            │
      │   原文 C 全程保留，证据池只做"强调"不做"替代/删减"    │
      │                                                │
      │   工程优化：snapshot 原文末尾 KV 缓存，replay 阶段    │
      │   只对新增证据 + 问题做增量前向（插入证据 <128 token, │
      │   显存开销与 Vanilla 相近）                        │
      └──────────────────────┬──────────────────────┘
                             │
                Final Answer: "independent ghost-entities" ✓
                （Vanilla 直接生成该例会答错）
```

## 设计决策解剖 ⚖️

| 决策点 | 他们的选择 | 被放弃的方案 | 为什么 | 这个选择何时失效 |
|---|---|---|---|---|
| 证据候选来源范围 | 只从原始上下文 I_C 内选取候选 span，即便回放的 scaffold 已经在 prompt 里 | 允许从"整个 replay prompt"（含已插入证据）里继续选新 span | Table 4：Full-prompt 选择 macro-average 0.19 明显低于 Context-only 的 0.23；PopQA Acc 从 0.02 涨到 0.07，NQ Acc 从 0.04 涨到 0.08——说明允许在已选证据里"近亲繁殖"会让候选偏离原文真正的高价值 token | 当证据池已经很大、原始上下文里高价值 token 已被选完（R 轮很多之后），"只从原文选"的边际收益会下降，因为剩余候选质量本身变差——论文没有测这个极限场景 |
| 证据组织粒度 | 把命中 token 映射回其所在完整句子再回放（span-level） | 直接把高分 token 本身（不做句子级还原）塞回 prompt | 3.3 节的定性论证："a selected token may identify an entity, date, or predicate, but the model usually needs the surrounding statement to use it reliably"——但论文并未对 token 级 vs 句子级做单独定量消融，这是一个只有直觉论证、没有数字支撑的决策 | 当命中句子本身很长、夹杂大量无关信息（如法律文书、书籍长难句）时，句子级物化会把"1 个高分 token"的噪声放大成"整句话噪声"；CLIPPER 恰是长书籍语料但论文未分析句长对效果的影响 |
| 递归轮数 R | 固定且小的 R（主实验 R=2），明确写"R is small and fixed...rather than an open-ended reasoning procedure" | 开放式循环直到收敛，或按任务/置信度自适应调整轮数 | Table 6：R 从 1→2，macro-average 从 0.17 跳到 0.22；但继续加轮次收益因任务而异——NQ 在 R=3/R=4 更好（最佳 NQ F1、InfMC Acc 在 R=4），PopQA 却在 R=2 就是峰值（R=3/R=4 上 PopQA F1 从 0.19 掉到 0.18/0.17） | 在 PopQA 这类"证据高度集中、实体中心"的任务上，固定 R=2 之后继续加轮次不但没帮助，还会引入无关证据挤占注意力——固定 R 的"一刀切"超参在这类任务上不是最优 |
| Top-K 候选预算 | 固定 K（消融测了 1/8/16/32），主实验默认 K 值论文正文与附录均未再次明确重申具体数值 | 按任务自适应选择 K | Table 7：K 越大不是单调变好——NQ Acc 在 K=32 时（0.04）反而比 K=8/K=16（都是 0.08）差，但 PopQA、InfMC 在 K=32 时最好（0.10、0.58）；论文自己总结为"recall–noise trade-off" | 在 NQ 这种"答案高度局部化、干扰项多"的数据集上，扩大 K 会把噪声一并选进来，"越大越好"的直觉在这类任务失效 |

## 成本与量级 💰
- **训练成本**：0——全程 training-free，Limitations 明确"it remains training-free and does not maintain a persistent external memory"。
- **推理运行时**（Table 5 / Figure 5，CLIPPER 数据集，Llama3-8B，128K 上下文，wall-clock）：Vanilla 44 分钟，AttnSharp 46 分钟，DAC 34 分钟，A-MEM 50 分钟，DySCO 2 小时 13 分钟，**RECONTEXT 62 分钟**——比 Vanilla 慢约 41%，但比 DySCO 快约 53%。
- **显存开销**：论文只给定性描述——"inserts fewer than 128 additional evidence tokens, resulting in only minimal memory overhead...similar to both Vanilla"（4.7 节），**具体 GB 数值论文未披露**。
- **硬件**：NVIDIA A100 与 H200 GPU，H200 跑在 ARM64（aarch64）架构上（Appendix B.2）。
- **数据规模**：8 个长上下文基准，128K 上下文长度（NQ/TriviaQA/HotpotQA/PopQA/NarrativeQA/InfBench QA/InfBench MC 取自 HELMET 基准，CLIPPER 为独立长书籍 claim verification 基准），另有 64K 上下文的鲁棒性验证（Table 3）。
- **模型规模**：三个 backbone 均为 4B–8B 量级开源模型（Qwen3-4B、Qwen3-8B、Llama3.1-8B），**未测试 30B+ 或闭源模型**，相对提升是否在更强模型上依然成立，论文没有证据支持。
- **最小可行配置**（论文披露的核心超参）：w=8（cue token 数）、λ=0.75（指数衰减系数）、R=2（主实验默认递归轮数）；**Top-K 的主实验默认值论文未披露**（仅在消融表里单独扫描 K=1/8/16/32，未回填 Table 1 用的是哪个 K）。

## 证据审计 🔬

**实验设计是否公平**：5 个 baseline（Vanilla、AttnSharp、DySCO、A-MEM、DAC）覆盖 attention 干预、外部记忆、压缩三条路线，覆盖面合理；4.3 节声明同一 backbone 下所有方法用相同 prompting 格式、解码配置、上下文预算，条件对齐。但"best average rank on all three backbones"这个 headline 结论是靠 **排名聚合** 撑起来的，掩盖了单指标不 dominant 的事实：
- Qwen3-8B 上 ReContext 在 HotpotQA 明显输给 DAC（Acc 0.20 vs 0.22，F1 0.34 vs 0.35）、InfBench MC 被 Vanilla/DAC 并列反超（0.63 vs 0.64）、CLIPPER 被 DySCO 反超（0.33 vs 0.34）——论文 4.4 节自己也承认"with exceptions on HotpotQA, InfBench MC, and CLIPPER"。
- Llama3-8B 上 NQ F1 被 A-MEM 反超（0.31 vs 0.34），NarrativeQA F1 被 DAC 反超（0.29 vs 0.30）。
- Table 2、Table 3 的鲁棒性验证都只抽测 3 个数据集（NQ/PopQA/InfMC），并非完整 8 数据集矩阵，用局部证据支撑"thinking 模式/短上下文下依然有效"的整体性结论，覆盖面偏窄。

**最强证据**：Figure 1/Figure 6 的"top 0.1% token 已占 50–80% 累积相关性"是全文最扎实的前提性证据——覆盖 8 个数据集、3 个模型的均值与方差曲线全部展示，说明"证据高度集中在极少数 token"在不同任务和模型上都稳健成立，直接支撑"只需挑出约 128 个 token"的设计假设。其次，Table 1 的核心结论我逐格核算过：Vanilla 在 8 数据集×3 backbone 共 24 个 Acc 值的平均为 0.2421≈0.24，RECONTEXT 为 0.3017≈0.30，相对提升 24.6%——数字站得住，成立条件是 4B–8B 开源模型 + 128K 上下文这一具体设置。

**最可疑的论证**：我按论文 Appendix E.1 的设定（n 个互相正交的单位向量 token、query q 只对答案方向有正偏好）独立复现了 Theorem 1 的递归更新过程（详见下方动手练习代码），发现当 q 对答案的原始偏好强度 m 较大（如 m=1.5、m=4.0，对应"模型本来就相当确定答案"的场景）时，**第一轮证据回放后余弦相似度反而下降**（m=1.5 时从 0.861 掉到 0.715，m=4.0 时从 0.999 掉到 0.875 再到 0.842，直到第 3、4 轮才重新回升），与 Theorem 1 宣称的"对所有 j≥1 都严格单调上升"直接矛盾。追根溯源：Theorem 1 的证明依赖一个充分条件——"原始 query-token 点积差 ⟨y−ci,q⟩ 必须小于初始 softmax 权重差 a_{i*}^{(0)}−a_i^{(0)}"，但 softmax 是压缩型映射（对二元 softmax 可用标准不等式 tanh(x)<x, x>0 证明：概率差 tanh(m/2) 恒小于原始点积差 m），我的推导显示在对称设定下这一充分条件对任意 m>0 都不成立。这不代表定理在所有配置下都是假的，但说明 Theorem 1 的"每一步都改进"是一个理想化 toy 模型（正交 embedding、单层线性叠加）在特定条件下的结论，并非对真实多层多头 Transformer 的普遍保证——而这恰好和论文自己的真实消融吻合：Table 6 里 PopQA 在 R=2 之后继续加轮次反而下降（F1 从 0.19 到 0.17），说明"模型已经比较确信时继续回放会先降后升"这个 toy 模型里的现象，在真实实验里也留下了痕迹。
- 次要疑点：Table 2 文本"macro-average improves over Vanilla from 28.0 to 32.6"未标注单位（其余处均用 [0,1] 小数），我按行逐项核算（RECONTEXT 五项之和 1.63/5=0.326，Vanilla 1.40/5=0.280）确认数值本身自洽，只是行文里百分号被省略，属于表达不统一而非数字造假。

**审稿人会要求补充**：① 全文没有报告任何多随机种子/标准差/置信区间；② Theorem 1 的正交-单层简化假设在真实（非正交、多层非线性）attention 上是否仍成立，至少应给一个真实模型 hidden state 上的模拟验证；③ 主实验 Table 1 用的具体 Top-K 数值未回填，影响可复现性；④ 更大规模/闭源模型上是否仍有同等相对收益。

**最小复现实验**：只用 InfBench MC（选择题，评测简单、无 F1 对齐歧义）+ 单一 backbone（Qwen3-4B，参数门槛低）对比 Vanilla vs RECONTEXT（R=2, K=8, w=8, λ=0.75）在 128K 上下文下的 Acc 差值，单卡 A100 上可在数小时、几十美元云算力量级内跑完，足以验证"证据回放是否带来正向增量"这个核心论断的方向性。

## 可复用点（PM决策）
**何时采用**：① 你在用能访问 attention 权重的开源模型（Qwen/Llama 等）做长文档 QA/合规检索/客服知识库，发现模型"读了但没用上"证据（关键条款明明在工单里，模型却没引用），且不想为此重新训练模型或搭一整套外部检索基础设施；② 你需要证据可追溯/可解释（合规、法律、医疗场景）——它把选中证据物化成"原文摘录的完整句子"而非 latent attention，便于向审核人员做证据高亮展示（Figure 4 的定性案例正是这种展示形态）。

**何时规避**：① 推理服务调用闭源 API（GPT/Claude/Gemini 等不暴露 attention 的模型）——论文 Limitations 已写死这条路走不通；② 场景对延迟极敏感（如实时对话）——实测慢 Vanilla 约 41%，且这个开销只有在上下文足够长、证据被"稀释"到很小比例时才划算（对应 Figure 1 的前提）；③ 模型规模较大或已经过充分长上下文训练（30B+、或闭源强模型）——论文只在 4B–8B 量级验证过，绝对收益是否等比例迁移未知。

**供应商拷问清单**：
1. "能不能让我看到具体引用了原文的哪几句话，而不是一个黑盒的相关性分数？"（对应证据物化设计——给不出可追溯原文 span，说明可能是纯 attention-rescaling 式黑盒干预，出错无法审计）
2. "我们用的模型是否是闭源 API？如果是，你们打算怎么在拿不到 attention 权重的前提下替代这套机制？"
3. "你们在我们这类数据（证据分散 vs 高度集中）上测过的最优递归轮数和 Top-K 是多少？"（对应论文自己承认"最优 R、K 是任务依赖的，不存在万能配置"）

## 关联网络 🕸️

**相关论文/概念**：
- [[Wiki/概念/04_Agent框架/Harness设计模式]] —— ReContext 全文反复用"LLM Harness"自我定位（2.2 节明确称其为"context harness"），完全对应 Harness 概念"外部框架接管非语义工作、让决策者只需专注语义判断"的定义，只是这里的"外部框架"是 prompt 层的证据脚手架，是 Harness 概念在"证据组织"这一子问题上的具体实例化。
- [[Wiki/概念/05_记忆与检索/上下文压缩与KV缓存]] —— ReContext 在 Related Work 明确与 DAC、LLMLingua 等压缩方法划清界限（"does not...replace the original long context with a shortened version"），自己又在实现上做了 KV 缓存复用优化（snapshot 原文末尾 KV cache，replay 阶段增量前向）——是对该概念"按重要性分层处理"思路的一个变体：不丢弃、不压缩，而是"复制强调"。
- [[Wiki/论文笔记/09_长上下文记忆/36_AdaCoM长视野Agent自适应上下文管理]] —— 同属"上下文管理"问题域，但 AdaCoM 用 RL 训练一个独立小模型主动删除/合并上下文片段（有损操作）；ReContext 训练无关，且明确"selects for emphasis rather than exclusion, unselected context remains available"（不删减）。两者是"上下文治理"的两种哲学：AdaCoM 学"删什么"，ReContext 坚持"不删，只让重要的被看见两次"。
- [[Wiki/论文笔记/09_长上下文记忆/46_效率前沿LLM上下文管理成本性能优化框架]] —— 该框架用预处理复用次数 N 判定策略选型（N≈1 用轻量检索，N≫1 用重量压缩）。ReContext 的证据甄别是每次查询都要重新做的一次性开销、不存在跨 query 复用的预处理结果，按此框架应归为"N≈1"这一端，解释了它为何刻意保持 training-free、无 persistent memory。
- [[Wiki/概念/03_推理与评测/测试时计算扩展]] —— ReContext 的递归轮数 R 正是"测试时计算扩展"里"深度"维度的具体实现（反复对同一输入精化候选证据）。Table 6 的实测规律与该概念"收益随投入呈对数增长、任务不同收益不同"高度吻合：R 从 1→2 大幅增益，但继续加深在 PopQA 上收益转负。
- [[Wiki/论文笔记/09_长上下文记忆/124_MeMo记忆即模型]] / [[Wiki/概念/05_记忆与检索/记忆即模型]] —— 二者是"长上下文/知识管理"问题的两个极端：MEMO 把知识训练进专属小模型参数（内化，需 Full SFT、更新要重训），ReContext 完全不碰参数，每次查询都在推理时从原文现场重新组织证据。是"参数内化 vs prompt 层组织"这一设计光谱的两端。

**冲突/印证**：
- **路线对立**：[[Wiki/论文笔记/09_长上下文记忆/44_LLM是否需要睡眠离线递归的记忆巩固]] 与 ReContext 都用"recursive/recurrence"描述自己的核心机制，但方向几乎相反——44 号论文的"睡眠"机制是当上下文写满时，用 N 次**离线**递归前向传播把上下文**压缩进 SSM 快速权重、随后清空 KV 缓存**（有损，且要求 Attention-SSM 混合架构），睡眠深度 N 越大性能越好；ReContext 的"recursive"则是每轮都**保留 KV 缓存和全部原文**，只是不断扩充一个文本证据池（无损，标准 Transformer 即可用），且论文明确强调 R 要"small and fixed"、不是开放式循环。两篇论文用同一个词描述了对抗长上下文退化的两种几乎对立的路线，值得在选型时对照。
- **印证**：[[Wiki/论文笔记/10_RAG检索/133_Grep-vs-向量检索Agent搜索对比]] 的核心发现是"Agent Harness/框架对最终效果的影响超过检索方法本身选择"（同一模型 GPT-5.4 换框架，准确率从 89.7% 跌到 55.2%，差 34.5 个百分点），这与 ReContext 的问题域定位互相印证——ReContext 的出发点正是"context access 不等于 context utilization"，二者共同指向同一结论：排行榜上"底层算法"的差异，很多时候不如"中间编排层怎么把信息喂给模型"这件事本身影响大。

## 动手练习 💻

**练习目标**：论文 Theorem 1（Appendix E.1–E.2）声称"每一轮证据回放都必然让隐状态更接近答案"，但证明依赖一个不太直观的充分条件。下面用纯 Python（无第三方依赖）严格照抄论文 Eq(14)–(18) 的递归更新规则，构造一个"自然"的 query（对答案方向有正偏好，强度 m 可调），实测：① 余弦相似度是否真的每轮都涨；② 论文自设的充分条件在这个配置下是否成立。这段代码就是本篇"证据审计"里"最可疑的论证"一节的原始复现实验。

```python
import math

def dot(a, b):
    return sum(x * y for x, y in zip(a, b))

def norm(a):
    return math.sqrt(dot(a, a))

def cos_sim(a, b):
    return dot(a, b) / (norm(a) * norm(b))

def softmax(xs):
    m = max(xs)                       # 数值稳定：减去最大值再指数化
    exps = [math.exp(x - m) for x in xs]
    s = sum(exps)
    return [e / s for e in exps]

def add_vec(a, b):
    return [x + y for x, y in zip(a, b)]

def scale_vec(a, s):
    return [x * s for x in a]

def run_recontext_toy(n_tokens, m, rounds):
    """
    完整对应论文 Appendix E.1 的玩具设定：
    n_tokens 个互相正交、单位范数的 token embedding c_1..c_n（对应论文假设）
    c_0 是正确答案 y = c_{i*}；query q 只对答案方向有正偏好，强度为 m（对称设定：<ci,q>=0, i!=0）
    """
    C = [[1.0 if i == j else 0.0 for j in range(n_tokens)] for i in range(n_tokens)]
    y = C[0]
    q = scale_vec(y, m)

    seq = list(C)                                  # x^(0) = [c_1, ..., c_n]，Eq 初始序列
    a = softmax([dot(v, q) for v in seq])           # a^(0)：Eq(14)
    h = [0.0] * n_tokens
    for w, v in zip(a, seq):
        h = add_vec(h, scale_vec(v, w))             # h^(0)：Eq(15)

    trace = [cos_sim(h, y)]
    # 现算论文 Theorem 1 的充分条件：⟨y-ci,q⟩ < a_{i*}^(0) - a_i^(0) 是否对所有 i 成立
    gap_raw = max(dot(y, q) - dot(C[i], q) for i in range(1, n_tokens))      # 原始点积差
    gap_softmax = a[0] - max(a[1:])                                          # softmax权重差
    assumption_holds = gap_raw < gap_softmax

    for j in range(1, rounds + 1):
        scores = [dot(v, h) for v in seq]           # 用上一轮隐状态 h^(j-1) 当作新 query 读 attention
        top = scores.index(max(scores))              # 找当前最受关注的候选证据 token
        seq.append(seq[top])                          # "回放"：把它复制一份拼回序列末尾，Eq(16)
        a = softmax([dot(v, h) for v in seq])         # a^(j)：仍用旧 h 作 query，但在新序列上算，Eq(17)
        h = [0.0] * n_tokens
        for w, v in zip(a, seq):
            h = add_vec(h, scale_vec(v, w))           # h^(j)：Eq(18)
        trace.append(cos_sim(h, y))

    return trace, gap_raw, gap_softmax, assumption_holds

if __name__ == "__main__":
    # m 越大 = query 对答案的原始偏好越强（对应"模型本来就比较确定答案"的场景）
    for m in [0.3, 1.5, 4.0]:
        trace, gap_raw, gap_sm, ok = run_recontext_toy(n_tokens=8, m=m, rounds=4)
        print(f"m={m}  论文充分条件是否满足: {ok}  (原始点积差={gap_raw:.3f}, softmax权重差={gap_sm:.3f})")
        for j, c in enumerate(trace):
            print(f"  round {j}: cos(h,y) = {c:.4f}")
        print("  全程严格单调递增:", all(trace[i] < trace[i+1] for i in range(len(trace)-1)))
```

运行结果（已实测）：m=0.3 时全程单调递增（0.455→0.619→0.788→0.885→0.935）；但 **m=1.5 时 round 0→1 从 0.861 掉到 0.715**，m=4.0 时 round 0→1→2 从 0.999 连续掉到 0.875、0.842，直到 round 3 才回升——且三组配置里论文自设的充分条件全部不成立（gap_raw 恒大于 gap_softmax）。这说明"模型本来就比较自信"时，强行做证据回放反而可能先把隐状态推离答案，第 3-4 轮才追回来，与 R=2 是主实验默认值这一设定放在一起看，恰恰提示 R 太小可能来不及"追回"、R 太大又在别的任务上过犹不及（呼应 Table 6 的 PopQA 现象）。

## 自测三层 🎓
- **L1 复述**：用自己的话说清楚 ReContext 的完整链路——冻结参数做一次前向读 attention → 用后缀 8 个 cue token 聚合打分 → 只在原始上下文内取 Top-K → 映射回完整句子 → 拼回 prompt 重播 → 重复 R 轮（每轮增量追加不覆盖）→ 用带着证据池的最终 prompt 生成答案，全程原文一字不删。
- **L2 解释**（为什么不用别的方案）：为什么不直接学 DySCO 那样在 decoding 期间直接改写 attention logits，而要把证据"物化"成文字重新塞回 prompt？结合 Table 4（context-only 选择优于 full-prompt 选择）、4.6 节定性分析（latent attention 干预留下的证据不可审计、DAC/A-MEM 的压缩视图可能漏句）、Figure 5（DySCO 比 ReContext 慢 2 倍以上）三点，说明"显式文本证据池"在可解释性和工程成本上相对 attention 干预的优势是什么，以及这个优势的边界在哪里（提示：闭源模型拿不到 attention，这个优势直接失效）。
- **L3 应用**：你是一家企业客服系统的 AIPM，Agent 需要在 50 页的历史工单和知识库文档里回答"这个客户之前是否申请过同类退款"这种需要精确定位证据的问题，且要求答案能标注引用出处以便质检抽查。请用"可复用点"里的供应商拷问清单，判断该场景是否适合引入类似 ReContext 的机制，并指出如果你们的模型服务走的是 OpenAI/Anthropic 官方 API 而非自部署开源模型，这个方案会在哪一步直接卡死。

📅 知识时间锚：论文发表于 2026-07-02（arXiv:2607.02509v1），本笔记复核于 2026-07-15。
