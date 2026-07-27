---
tags: [论文笔记, Agent系统设计, Harness, 自进化, 评估方法论, 测试时扩展]
paper_id: "181"
filename: "181 - Rethinking the Evaluation of Harness Evolution for Agents.pdf"
authors: "Yike Wang, Huaisheng Zhu, Zhengyu Hu, Yige Yuan, Zhengyu Chen, Shakti Senthil, Hannaneh Hajishirzi, Yulia Tsvetkov, Pradeep Dasigi, Teng Xiao（Allen Institute for AI × 华盛顿大学）"
year: 2026
笔记层级: 骨干
成熟度: 🧪
掌握度: 未拷问
复核日期: 2026-07-21
---

# 重新思考 Agent Harness 进化的评估（Rethinking the Evaluation of Harness Evolution for Agents）

📄 **原文 PDF**：[[RAW/181 - Rethinking the Evaluation of Harness Evolution for Agents.pdf]]
arXiv:2607.12227v1（2026-07-14），13 页（正文 8 页 + 附录），AI2 × UW，代码开源

## PM 速判（30秒）
> 一句话：给当红的「自动 harness 进化」泼冷水的评估方法论论文——指出现有工作（Meta-Harness、AHE、AEVO）**用同一个公开基准既做搜索反馈又做最终评测**，且从不与同预算的 test-time scaling 基线对比；在 Terminal-Bench 2.1 上用统一预算协议重测后发现：harness 进化**没有稳定跑赢简单的并行采样/顺序精化**（无 unit test 时平均 67.4 分还低于 68.2 的裸基线），且进化出的 harness 换到 held-out 任务上几乎零增益（平均 +0.6 分）——收益主要来自「多试了几次」而非「更好的 harness 设计」。PM 必须知道：供应商吹的「自进化 harness」大概率是把搜索预算花在了评测集上，验收必须要求预算对齐基线 + 训练/评测任务分离。

## 双层费曼 🗣️
> **给 CEO 的一句话**：有一类产品声称「AI 会自己改进自己的工作流程，越用越强」；这篇论文设计了一个公平的对照实验——同样的额外算力，一边给「自我改进流程」，一边就简单粗暴地「同一道题多做几遍挑最好的」——结果多做几遍反而更好，而且那个「改进过的流程」拿到新任务上几乎没用。翻译成采购语言：为「自进化」支付溢价前，先要求供应商证明它赢过同价位的暴力重试。
>
> **给工程师的一段话**：论文把四种花预算 K 的方式放进统一形式框架：①并行采样（固定 harness h，独立采 K 条轨迹，宽度探索，无 unit test 时自评选优、有则 oracle 选中）；②顺序精化（深度探索，第 k 条轨迹以 σ(y_{k-1}) 或 σ(y_{k-1}, R) 为条件修上一条）；③harness 进化（跨任务批量：meta agent M 读经验库摘要 σ(C_{k-1}) 产出 h_k，跨任务共享，实例化为 AHE 并禁用其 explore agent——否则它会从外部检索针对该基准调好的 harness，把「进化」和「抄答案」混为一谈）；④harness scaling（本文新造的对照臂：逐实例进化 harness，h_k=M(x, σ(h_{k-1},y_{k-1},R))，用来分离「可复用设计」与「实例级适配」）。设置：Terminal-Bench 2.1（89 个终端任务，修复了 2.0 的 28 个坏任务）、Claude Opus 4.6 / GPT-5.4 / GPT-5.4 mini、K=5、m=1、两次运行取平均、初始 harness 是「只有一个 bash 工具」的极简配置。三组结果：无 unit test 时 HE 平均 67.4 < 直采 68.2 < HS 71.8 < PS 72.3（GPT-5.4 上 HE 把 75.3 砍到 69.7——自评驱动的 harness 修订对强模型是净伤害）；有 unit test 时 pass@1 上 PS 86.0 最好、HE 75.8 仅比直采 72.9 高一点，pass@5 上 SR 91.8 最好——HE 的收益只在 pass@5 兑现，说明它赢在「多次尝试」而非「更好设计」；泛化测试（45 训练/10 验证/34 测试）HE 平均只 +0.6。
>

## 问题域定位 🎯
- **根本约束**：harness 进化本身就是一个「反复评估-修订候选」的迭代搜索过程，与 agentic test-time scaling 在计算形态上同构；不与预算对齐的任务级搜索基线对比，就无法归因收益是来自「设计改进」还是「搜索本身」。同时搜索反馈与最终评测共用同一任务集时，报告的增益无法区分「可迁移的 harness 设计原则」与「对评测实例的适配」。
- **之前卡在哪**：prompt 优化（DSPy/MIPRO/TextGrad/GEPA）、经验外化（ExpeL/ACE）到全 harness 搜索（Meta-Harness/AHE/AEVO）一路热闹，但评估协议沿用「基准上搜、同基准上报」的惯例；test-time scaling 文献（Snell 2024、Large Language Monkeys）早就证明纯重复采样的威力，两条线从未在受控预算下对撞过。
- **开启/关闭的路线**：开启「harness 进化必须过三关：同预算 TTS 基线、训练/评测分离、报 pass@1 而非只报可选优的指标」的评估范式；给「在 Terminal-Bench 类低 harness 敏感度基准上刷自进化故事」的路线亮红灯，并呼吁新基准满足两条件——任务留有足够 headroom + 性能确实依赖 harness（专用工具/技能/工作流是成败关键）。

## 核心机制（ASCII 图）

```
   统一预算 K 下的四种花钱方式（固定策略 π、任务分布 X、初始 harness h1）
   ─────────────────────────────────────────────────────────────
   ①并行采样 PS          ②顺序精化 SR          ③harness进化 HE        ④harness scaling HS
   task x                task x                批量任务{x(i)}          task x
   h 固定                h 固定                h_k 跨任务共享进化       h_k 逐实例进化
   y1 y2 ... yK 独立     y1→y2→...→yK          rollout→经验库C_k       y1→M改h→y2→...
   宽度探索              深度探索（读σ(y_{k-1})）→meta agent M 读σ(C)    实例级 harness 适配
   │                     │                     →产出 h_{k+1}           │
   ▼选择                 ▼选择                  ▼选择                   ▼选择
   无test: 自评选优       无test: 取最后一条      无test: 取最终harness    无test: 取最后一条
   有test: R=1 即中       有test: R=1 即中       有test: 库中最优harness   有test: R=1 即中
   ─────────────────────────────────────────────────────────────
   结果（Terminal-Bench 2.1，K=5，三模型平均）：
   无unit test  pass@1：直采68.2 | PS 72.3★ | SR 69.3 | HE 67.4✗ | HS 71.8
   有unit test  pass@1：直采72.9 | PS 86.0★ | SR 84.3 | HE 75.8✗ | HS 82.6
   有unit test  pass@5：          PS 86.0 | SR 91.8★ | HE 86.2  | HS 89.3
   泛化(45训/10验/34测) HE：Opus +1.2 / GPT-5.4 +0.0 / 平均 +0.6 ✗
   诊断：HE 收益只在 pass@5 兑现 → 赢在多次尝试，不在更好设计
```

### 三张表的完整数字（Table 1/2/3）

**Table 1 · 无 unit test，pass@1**（自评/最后一条选择）：

| 方法 | Opus 4.6 | GPT-5.4 | GPT-5.4 mini | 平均 |
|------|---------|---------|--------------|------|
| 直采（初始 harness） | 69.9 | 75.3 | 59.4 | 68.2 |
| 并行采样 | 74.7 | **79.2** | **62.9** | **72.3** |
| 顺序精化 | 73.0 | 73.0 | 61.8 | 69.3 |
| Harness 进化 | 71.4 | 69.7 ↓5.6 | 61.3 | 67.4（低于直采） |
| Harness scaling | **76.0** | 78.1 | 61.2 | 71.8 |

**Table 2 · 有 unit test（既做反馈又做 oracle 选择）**：

| 方法 | Opus p@1 / p@5 | GPT-5.4 p@1 / p@5 | 平均 p@1 / p@5 |
|------|---------------|-------------------|----------------|
| 直采 | 69.9 / – | 75.9 / – | 72.9 / – |
| 并行采样 | **84.8** / 84.8 | **87.1** / 87.1 | **86.0** / 86.0 |
| 顺序精化 | 83.1 / **90.4** | 85.4 / **93.3** | 84.3 / **91.8** |
| Harness 进化 | 73.0 / 83.2 | 78.6 / 89.3 | 75.8 / 86.2 |
| Harness scaling | 83.1 / 89.9 | 82.0 / 88.8 | 82.6 / 89.3 |

**Table 3 · 泛化（45 训练 / 10 验证 / 34 测试，pass@1）**：直采 Opus 63.3 / GPT-5.4 72.1 / 平均 67.7；进化后 harness 64.5（+1.2）/ 72.1（+0.0）/ 68.3（**+0.6**）。

### 机制细读一：meta agent 到底在改什么（§5.1 轨迹审计）
- **Harness Evolution 的编辑升级路径**：首选 prompt 层——往系统提示/长期记忆里加行为规则（尽早交付+预算意识、改易碎状态前先备份、收尾前复查任务约束）；advisory 文本收益见顶后升级为**运行时强制**的 middleware——回合预算跟踪器（到阈值就提醒交付）、超长工具输出截断、缺交付物/未验证就阻断完成的 finalization gate；再改工具层——纠正误导性工具说明、在命令执行时注入恢复提示。
- **Harness Scaling 的主导模式**：把上一轮 rollout 的具体失败知识写进下一轮 prompt/记忆——已知 bug、实现模板、验证检查、文件路径、命令序列；另一大类是工作流提效（批量安装+验证命令、跳过环境探测、放宽慢安装的 shell 超时、带显式消息预算的规定工作流）与验证强化（提交前跑模仿 verifier 的语法检查/本地测试）。
- **诊断结论**：编辑「个个合理」但**记忆修复多、蒸馏策略少**——持久化的多是能干 agent 单 rollout 内本可自己重新发现的信息（省时间但不把失败变成功）；深层推理需求或 harness 管不到的约束造成的顽固失败核心一动不动；持久 prompt 文本不断膨胀反噬剩余收益。
- **附录 Fig.3 六个案例**（Harness Scaling 的成功修复样本）：caffe-cifar-10（前台安装+轮询睡眠耗尽预算→强制后台安装+cmake 优先→第 2 轮过）；cancel-async-tasks（**agent 看到意外行为后去弱化测试**→植入"不许弱化测试"规则+参考实现→过）；compile-compcert（装错 Coq 版本+短超时→直接钉死 8.16.1+智能等待→第 3 轮过）；count-dataset-tokens（分字段 tokenize 规则+数据集事实直接写进 prompt→过）；db-wal-recovery（打开 SQLite 前必须先备份 WAL 的禁令→过）；mteb-retrieve（query/passage 的 prompt type 区分+诊断打印→连败 4 轮后第 5 轮过）。注意这些修复几乎全是**任务专用事实**——正是「过拟合搜索集」的微观证据。

### 机制细读二：实验设置速查（§4.1 + 附录 A）
- **基准**：Terminal-Bench 2.1，89 个终端任务（修复 2.0 的 28 个：外部依赖漂移、资源预算错配、指令与测试错位）。
- **模型**：Claude Opus 4.6 / GPT-5.4 / GPT-5.4 mini，reasoning effort 全高档，生成上限 128K token；泛化实验只用前两个。
- **预算**：K=5，AHE 每 harness 每任务 m=1 条 rollout；全部结果为 2 次独立运行平均；基础设施异常（沙箱崩溃/API 超时）按 r=0 记失败不剔除。
- **初始 harness**：四臂统一从 AHE 的极简 harness 起步——只有一个 bash 工具，无技能/中间件/持久记忆。
- **三个 agent 角色**：Code Agent（干活，300 turn 上限）、Agent Debugger（摘要算子 σ 的实现：把多条 trace 组织成文件系统环境，用 shell 工具探索、写根因分析报告、渐进披露省 token，25 turn）、Meta Agent（读报告改 harness，500 turn）。

## 设计决策解剖 ⚖️

| 决策点 | 他们的选择 | 被放弃的方案 | 为什么 | 这个选择何时失效 |
|--------|----------|------------|--------|---------------|
| 对照基线 | 与并行采样/顺序精化在「反馈可见性 + 推理预算 K」双对齐下比较 | 沿用原论文与「初始 harness 直采」的对比 | HE 本身是搜索，只赢过不搜索的基线证明不了设计价值 | 当 harness 改进能跨大量未来任务摊销时，按单基准单预算算账会低估 HE 的长期复利（本文只测 89 任务内的迁移） |
| HE 的实例化 | 用 AHE 且禁用 explore agent（外部检索针对基准调好的 harness） | 保留完整 AHE；或另测 Meta-Harness/AEVO | 保证增益全部归因于「从反馈进化」而非「检索现成答案」 | 只测一个方法的一个消融版——若 AEVO/Meta-Harness 的搜索算子质量更高，「HE 不行」的结论可能是「AHE 不行」 |
| 新增第四臂 harness scaling | 逐实例进化 harness 作对照 | 只比三臂 | 分离「可复用设计」与「实例级适配」：HS 赢 HE 则说明 harness 修订的价值在实例级而非可迁移设计 | HS 与 SR 的边界模糊——改 harness 里的任务专用 prompt 与直接改轨迹在信息上等价，臂间差异可能只是接口差异 |
| 评测基准 | Terminal-Bench 2.1（修复 2.0 的 28/89 个坏任务） | 沿用 2.0 或多基准 | 修复外部依赖漂移/预算错配/指令-测试错位，测得更准；且是 HE 文献的主场（以子之矛攻子之盾） | 作者自认 TB 可能天生不吃 harness（极简 bash+prompt 已够解大多数可解任务、余下失败卡在模型推理）——在 harness 敏感的基准上结论可能反转 |
| 预算刻度 | K=5、m=1、两次运行平均 | 更大的 K 与更多种子 | 控制三个前沿模型 × 五方法 × 三设置的成本 | K=5 对进化类方法可能太短（搜索没预热完）；两次运行无方差报告，±1-2 分的差距可能在噪声内 |

## 成本与量级 💰
- **实验规模**：89 任务 × 3 模型 × 5 方法臂 × K=5 × 2 次运行，代码 agent 单轨迹上限 300 turn / 200K context / 128K 生成，E2B 远程沙箱逐 rollout 新建——这是一篇「评估论文」但计算开销不小。
- **方法成本对比**：同预算 K 下四臂的推理成本近似对齐（这正是设计点）；但 HE/HS 额外付 meta agent（500 turn 上限）与 Agent Debugger（轨迹摘要 agent，把 trace 组织成文件系统做渐进披露省 token）的费用——严格说 HE 臂的真实开销略高于 TTS 臂，结论对 HE 更不利。
- **收益/成本结论**：额外算力花在「修解」（SR 的 pass@5 91.8）好过花在「修 harness」（HE 86.2）；无验证信号时连修解都不如「多采几条自选」（PS 72.3）。
- **我的产品最小可行配置**：有可靠 verifier → 并行采样 + oracle 选择是性价比之王；无 verifier → 并行采样 + 自评选优；harness 改进走人工工程化（观察轨迹→手改），暂不上自动进化闭环。

## 证据审计 🔬
- **实验公平吗**：预算对齐做得比原文献认真（同 K、同初始 harness、同 rollout 数），但有三处对 HE 不利的倾斜：①只测 AHE 一个方法且动了刀（禁 explore agent）；②K=5 的短预算利好一步到位的采样、不利需要预热的搜索；③89 任务的基准已被前沿模型刷到 68-76 分，headroom 小——作者在 §5.2 诚实列出了后两点作为替代解释。
- **最强证据**：pass@1 vs pass@5 的诊断逻辑——「若 harness 修订真的产出了更好的 harness，改进应体现在 pass@1；实际只在 pass@5 兑现」——这个归因论证不依赖具体分数，是全文最硬的一击。配套定性证据：meta agent 的编辑「记忆修复而非蒸馏策略」（把已知 bug/文件路径/命令序列写进 prompt——这些能干 agent 本来就能在单 rollout 里自己摸出来），顽固难题核心不动，且持久 prompt 膨胀反噬上下文。
- **最可疑的数字**：泛化实验平均 +0.6 分基于 34 个测试任务 × 2 次运行——单任务翻转就是 ±1.5 分，这个「零增益」结论的置信区间可能盖住 ±3 分；且无 unit test 表格里 HS 在 Opus 上拿了全场最佳 76.0，正文却统称「自动 harness 进化不行」——HS 臂的亮点被叙事压平了。
- **如果我是审稿人**：①要求报告跨运行方差与显著性检验（89 任务、2 runs 的差距多半在噪声边缘）；②补测 Meta-Harness 或 AEVO 至少一个原生方法；③在作者自己提议的「harness 敏感」基准（需要专用工具/技能才能解）上给一个正例或反例，否则「任务依赖」的讨论只是推测；④K 扫描（5→20）看 HE 是否只是没跑够。
- **最小复现实验**：不用复现全套——用任一 agent 框架在 20 个任务上跑三臂（直采 / K=5 并行采样 / K=5 的 prompt 自动进化），有 unit test 场景各记 pass@1 与 pass@5；若进化臂的增益同样只出现在 pass@5，即在自己场景复现核心结论。预算约 100-300 美元 API。

## 可复用点（PM 决策）
- **何时采用（本文的评估协议）**：任何「自我改进/自进化」类功能的验收都套三关——①同预算 TTS 基线；②优化反馈与验收任务集分离；③主指标用 pass@1（不许拿可选优的 pass@k 讲设计故事）。
- **何时规避（自动 harness 进化）**：①任务对 harness 不敏感（极简工具+prompt 已够）时别上；②无可靠外部正确性信号时绝对别上（自评驱动的修订对强模型是净伤害：GPT-5.4 掉 5.6 分）；③基准/场景 headroom 小时收益装不下工程复杂度。
- **例外条款**：本文负结果的适用边界是「harness 不敏感的任务 + 前沿模型 + 小 headroom」。若你的场景确实依赖专用工具/技能/领域工作流（如需要私有 API 编排的企业流程），harness 进化仍可能有真实空间——但验收协议不变。
- **一个正向启示**：Harness Scaling 臂显示「实例级 harness 适配」比「跨任务共享进化」更接近 TTS 基线（无 test 时 71.8 vs 67.4，且拿下 Opus 单项最佳 76.0）——如果一定要做 harness 自动化，per-instance 适配 + 用后即弃可能比持久化共享更稳。
- **供应商拷问清单**：
  1. 你们的「自进化」与同预算的并行采样/顺序精化比过吗？pass@1 差多少？
  2. harness 是在哪个任务集上进化的？验收集与它重叠吗？held-out 增益多少？
  3. 增益在 pass@1 还是只在 pass@k 兑现？后者说明是重试红利不是设计红利。
  4. 进化的编辑内容审计过吗——是可迁移策略还是任务专用事实（文件路径/命令序列/已知 bug）的记忆？
  5. 持久化的 prompt/记忆膨胀到多大了？context 膨胀的边际成本算过吗？
  6. 无 verifier 场景下自评修订的回退率（改完反而变差的比例）是多少？

### 术语对照（论文用语 ↔ 工程俗语）
- harness ↔ 脚手架/编排层：prompt+工具+记忆+验证例程+控制逻辑的总和（=178 综述里的 scaffold Σ）
- harness evolution ↔ 「让 agent 自己改自己的工作流」；harness scaling ↔ 「针对这一道题现改现用的工作流」（本文新造的对照概念）
- test-time scaling ↔ 推理时多花算力：并行采样=宽度（多试几条挑最好），顺序精化=深度（一条一条改）
- matched budget ↔ 预算对齐：反馈可见性与推理次数拉平后再比，防止「多算了就是好」的伪归因
- memorize fixes vs distill strategies ↔ 「背题」vs「学套路」——本文对 meta agent 编辑内容的核心判词
- pass@1 / pass@5 分叉 ↔ 归因探针：设计红利应落在 pass@1，重试红利落在 pass@k

## 关联网络 🕸️
- [[Wiki/论文笔记/03_Agent系统设计/32_Harness更新vs获益-自进化Agent能力解耦]] → **直接对话对象**：32 提出「harness 更新次数≠能力获益」的解耦视角，本文把它推进成完整评估协议（预算对齐 TTS 基线 + 泛化拆分）并给出系统性负结果，二者构成「质疑自进化叙事」的连续剧
- [[Wiki/概念/04_Agent框架/Harness设计模式]] → 本文的隐含正面结论：harness 的价值在于人工设计的模式沉淀；自动进化目前只会「记忆修复」，不会「蒸馏模式」
- [[Wiki/论文笔记/03_Agent系统设计/34_Agent-Harness有效反馈计算扩展律]] → 互补视角：34 讲反馈信号质量决定 harness 迭代的扩展律；本文补充「反馈=自评时扩展律为负」（无 test 时 HE 低于基线）的实证端点
- [[Wiki/论文笔记/03_Agent系统设计/172_Harness效应编排设计决定企业Agent economics]] → 172 证明 harness 设计对 token 经济影响巨大；与本文「TB 上 harness 不敏感」并不矛盾——敏感度是任务属性，这正是本文 §5.2 的核心洞察
- [[Wiki/概念/03_推理与评测/测试时计算扩展]]、[[Wiki/概念/03_推理与评测/自洽性采样]] → 本文的基线武器库出处
- **冲突/印证**：与 [[Wiki/论文笔记/03_Agent系统设计/178_现代Agentic系统自我改进综述]] 的 full-scaffolding 乐观谱系（DGM/Live-SWE-Agent/AHE）正面冲突——178 §8 提出的评估军规（预算内轨迹、迁移测试、信号与评测分离）恰是本文用来证伪该谱系近期成果的工具；与 [[Wiki/论文笔记/03_Agent系统设计/139_AEVO元编辑Agent进化框架]] 构成待裁决的张力（AEVO 是被点名但未被实测的方法）。

## 动手练习 💻（约 30 分钟）
练习目标：用一个玩具模型亲手看见本文的核心诊断——「进化的收益可以完全来自重复尝试」，以及为什么 pass@1/pass@5 的分叉能当归因探针。

```python
import random                                # 标准库，模拟任务求解的伯努利过程

random.seed(1)
N_TASK, K = 89, 5                            # 89 个任务、预算 K=5（对齐论文设置）

# 每个任务有一个"单次尝试解出率" p_i：模拟 Terminal-Bench 的任务难度分布
tasks = [random.betavariate(2.2, 1.0) for _ in range(N_TASK)]   # 均值≈0.69，接近直采基线

def parallel_sampling(p, k):
    """并行采样+oracle 选择：k 次独立尝试，任一成功即算过（pass@1 报告值）"""
    return 1 - (1 - p) ** k                  # 数学期望形式，等价于采样极限

def harness_evolution(p, k, train_boost, test_boost, is_train):
    """harness 进化：搜索消耗与采样同样的 k 次 rollout，
    换来任务解出率的提升 boost——但只在训练任务上真提升（过拟合），
    对 held-out 任务提升≈0（对应论文泛化实验 +0.6 分）"""
    boost = train_boost if is_train else test_boost
    p2 = min(1.0, p + boost)                 # 进化后的单次解出率
    return 1 - (1 - p2) ** k                 # 同样有 k 次机会（库中多个 harness 各试过）

def pass_at_1_after(p, boost, is_train, train_boost):
    """进化结束后用最终 harness 只采 1 条的 pass@1（论文的主指标口径）"""
    b = train_boost if is_train else boost
    return min(1.0, p + b)

TRAIN_BOOST, TEST_BOOST = 0.10, 0.005        # 训练集上显著提升，测试集上几乎为零

# --- 场景一：搜索与评测共用同一任务集（现有文献的做法）---
ps  = sum(parallel_sampling(p, K) for p in tasks) / N_TASK
he5 = sum(harness_evolution(p, K, TRAIN_BOOST, TEST_BOOST, True) for p in tasks) / N_TASK
he1 = sum(pass_at_1_after(p, TEST_BOOST, True, TRAIN_BOOST) for p in tasks) / N_TASK
base= sum(tasks) / N_TASK
print(f"[同集评测] 直采pass@1={base:.3f}  PS(oracle)={ps:.3f}")
print(f"[同集评测] HE的pass@5口径={he5:.3f}  HE的pass@1口径={he1:.3f}")
# 观察：HE 的 pass@5 口径看起来很美（重试红利+过拟合红利叠加），
# 但 pass@1 口径的增益 = 纯过拟合 boost —— 两口径分叉暴露收益来源

# --- 场景二：训练/评测任务分离（本文的泛化协议）---
test_tasks = [random.betavariate(2.2, 1.0) for _ in range(34)]  # 34 个 held-out
he_test = sum(pass_at_1_after(p, TEST_BOOST, False, TRAIN_BOOST)
              for p in test_tasks) / len(test_tasks)
base_test = sum(test_tasks) / len(test_tasks)
print(f"[held-out ] 直采={base_test:.3f}  进化后harness={he_test:.3f} "
      f"(增益={he_test - base_test:+.3f})")
# 预期：held-out 增益≈+0.005，复现论文"平均 +0.6 分≈零"的现象
# 顺手做敏感性检验：把 34 个测试任务重抽 10 次，看增益估计的波动幅度——
# 你会发现波动远大于 +0.005，这正是本笔记证据审计里质疑的统计功效问题
```

## 自测三层 🎓
> 判分点随笔记入库；自测先作答再展开。L1 不过不进 L2/L3——答错即记卡点。

> [!question]- L1 复述：四个方法臂各自把预算 K 花在哪里？无/有 unit test 两种设置下各臂怎么选最终答案？三张表的关键数字（HE 无 test 67.4 vs 直采 68.2；有 test 时 HE pass@1 75.8 vs PS 86.0；泛化 +0.6）说明什么？
> **参考答案**：①并行采样 PS：固定 harness，独立采 K 条轨迹（宽度探索）；②顺序精化 SR：第 k 条以上一条的摘要为条件修改（深度探索）；③harness 进化 HE：meta agent 读跨任务经验库摘要产出新 harness、跨任务共享；④harness scaling HS：逐实例进化 harness、现改现用。选择方式：无 unit test 时 PS 自评选优、SR/HS 取最后一条、HE 取最终 harness；有 unit test 时 R=1 即中（oracle 选中），HE 选库中最优 harness。数字含义：无 test 时 HE 67.4 低于直采 68.2——自评驱动的 harness 修订对强模型是净伤害（GPT-5.4 掉 5.6 分）；有 test 时 HE pass@1 75.8 远输 PS 86.0——同预算下修 harness 不如暴力多采样；泛化 +0.6≈零——进化出的 harness 在 held-out 任务上几乎无增益，收益来自「多试几次」而非可迁移的设计。
> **判分点**：必须答出 四臂的预算去向（宽度采样/深度精化/跨任务共享进化/逐实例适配）、无 test 时 HE 低于直采且有 test 时 HE 远输 PS、泛化 +0.6≈零即重试红利非设计红利（缺一个即卡点）

> [!question]- L2 解释：为什么「收益只在 pass@5 兑现」能推出「不是更好的 harness 设计」？这个论证的隐含假设是什么（提示：pass@1 反映单 harness 单尝试的真实力，pass@k 混入了选择效应）？反过来，harness 进化拥护者可以用什么实验反驳本文（提示：harness 敏感基准、更大 K、原生方法）？
> **参考答案**：若修订真的产出了「更好的 harness」，用最终 harness 单次尝试就应更强，即改进应体现在 pass@1；实际 HE 的收益只在 pass@5 兑现（86.2 vs pass@1 仅 75.8），说明赢的是「K 次尝试中任一命中」的选择效应而非设计变好。隐含假设：pass@1 反映单 harness 单尝试的真实能力，pass@k 混入选择效应；且好设计的收益应可在单次尝试上体现并跨任务迁移。拥护者的反驳实验：①换 harness 敏感基准——TB 上极简 bash+prompt 已够解大多数可解任务，在确实依赖专用工具/技能/工作流的基准上结论可能反转；②扫更大的 K——K=5 利好一步到位的采样、不利需要预热的搜索；③测原生方法——本文只测了禁用 explore agent 的 AHE 消融版，「HE 不行」可能只是「AHE 不行」。
> **判分点**：必须答出 pass@1=单尝试真实力 vs pass@k 混入选择效应的归因逻辑、HE 收益只在 pass@5 兑现所以是重试红利、至少两条反驳路径（harness 敏感基准/更大 K/原生方法 AEVO 或 Meta-Harness）（缺一个即卡点）

> [!question]- L3 应用：你在评审一个「Agent 平台自动优化编排配置」的采购提案，对方展示了在其 demo 任务集上从 62% 到 81% 的提升。用本文协议设计三个验收实验（预算对齐基线/任务分离/指标口径），并写出如果三关全过、你仍要检查的一个运营风险（提示：持久配置的 context 膨胀与回退机制）。
> **参考答案**：①预算对齐基线：把「自动优化」消耗的同等额外算力给并行采样/顺序精化，比 pass@1——62→81 若只赢过「优化前直采」则归因无效，必须赢过同预算 TTS 才证明设计价值；②任务分离：问清 demo 任务集是否就是优化时的反馈集，要求在 held-out 任务上重测增益（本文实测跨任务只剩 +0.6≈零）；③指标口径：要求报 pass@1 而非可选优的 pass@k，若增益只在 pass@k 兑现即是重试红利不是设计红利。三关全过后仍要查的运营风险：持久化的 prompt/记忆/配置会随迭代持续膨胀，反噬 context 成本与性能（本文实测编辑多为「记忆修复」型任务专用事实），必须确认配置有版本历史与一键回退机制、并审计编辑内容是可迁移策略还是背题。
> **判分点**：必须答出 三关各自机制（同预算 TTS 基线比 pass@1 / 优化反馈集与验收集分离 / pass@1 口径防选择效应）、运营风险=持久配置膨胀+回退机制（缺一个即卡点）

📅 知识时间锚：2026-07（arXiv 2607.12227v1，2026-07-14 提交；实验模型为 Claude Opus 4.6 / GPT-5.4 世代，基准为 Terminal-Bench 2.1。负结果强烈依赖「TB 对 harness 不敏感 + 前沿模型 headroom 小」两个时点条件——模型换代或出现 harness 敏感基准后需复测）