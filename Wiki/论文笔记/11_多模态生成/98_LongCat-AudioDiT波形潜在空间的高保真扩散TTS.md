---
tags: [论文笔记, LongCat-AudioDiT, TTS, 扩散模型, 语音合成, 零样本克隆, Meituan]
笔记层级: 参考
paper_id: "98"
filename: "98 - LongCat-AudioDiT - High-Fidelity Diffusion Text-to-Speech in the Waveform Latent Space.pdf"
authors: "Meituan LongCat Team"
year: 2025
成熟度: 🏭
---

# LongCat-AudioDiT：波形潜在空间的高保真扩散 TTS

📄 **原文 PDF**：[[RAW/98 - LongCat-AudioDiT - High-Fidelity Diffusion Text-to-Speech in the Waveform Latent Space.pdf]]

## PM 速判（30秒）

> **TTS 直接在波形潜在空间建模，消除中间表示（Mel 频谱图）带来的误差累积。** 美团 LongCat 团队发布 LongCat-AudioDiT：非自回归扩散 TTS，核心创新是跳过 Mel 频谱图直接在 Wav-VAE 的潜在空间做扩散，只需两个组件（Wav-VAE + DiT）。LongCat-AudioDiT-3.5B 在 Seed 基准的零样本声音克隆上超越 Seed-TTS（SIM: Seed-ZH 0.809→0.818，Seed-Hard 0.776→0.797），同时开源代码和权重。

| 年份 | 成熟度 | PM 相关性 |
|------|--------|---------|
| 2025 | 🏭 | **中**（TTS 产品技术路线选型参考，音色克隆能力基准）|

## 核心机制

```
架构对比：
  传统扩散 TTS：文本 → 中间声学表示（Mel频谱图）→ 声码器 → 波形
                           ↑ 误差累积两次
                           
  LongCat-AudioDiT：文本 → Wav-VAE 潜在空间 → 解码 → 波形
                              ↑ 只有一步转换，无中间误差
  
两个组件：
  1. Wav-VAE（波形变分自编码器）
     编码器：波形 → 连续潜在向量（训练时用）
     解码器：潜在向量 → 波形（推理时用）
     注：VAE 重建质量与 TTS 最终质量的关系是非平凡的（需要系统研究）
     
  2. DiT（扩散 Transformer）
     在 Wav-VAE 潜在空间做去噪扩散
     
两项关键推理改进：
  1. 训练-推理不匹配修正
     发现并修复了长期存在的配置不一致问题
  2. 自适应投影引导（Adaptive Projection Guidance）
     替代传统分类器自由引导（CFG）
     → 更高生成质量
```

## 关键数据 📅 2025

| 基准 | LongCat-AudioDiT-3.5B | Seed-TTS（之前 SOTA）|
|------|----------------------|---------------------|
| Seed-ZH SIM | **0.818** | 0.809 |
| Seed-Hard SIM | **0.797** | 0.776 |
| 开源状态 | ✅ 代码 + 权重全开源 | ❌ 闭源 |

## PM 结论

- 📅 2025 年，TTS 的两条技术路线：AR（GLM-TTS 路线）vs NAR 扩散（AudioDiT 路线）各有优势
- **关键技术洞察**：Mel 频谱图是人为引入的中间表示 → 直接在连续潜在空间建模消除两次转换误差
- 声音克隆场景下 SIM 分数是核心指标：0.797 vs 0.776 = 约 2.7% 提升，在用户感知上是可察觉的
- 对 PM：开源权重降低了语音克隆产品的入门门槛，要注意版权/伦理合规

## 论文精读卡片

**一句话**：LongCat-AudioDiT-3.5B 跳过 Mel 频谱图直接在 Wav-VAE 潜在空间做扩散，在零样本声音克隆 Seed 基准超越闭源 Seed-TTS（SIM：Seed-ZH 0.809→0.818，Seed-Hard 0.776→0.797），且完全开源。

**问题**：传统扩散 TTS 需要两次转换：文本→Mel 频谱图→声码器→波形，每次转换都引入误差累积；而端到端在波形空间建模的方法因复杂度高一直缺乏实用的系统性实现。

**核心方法**：
- Wav-VAE（波形变分自编码器）：将波形编码为连续潜在向量，训练时用编码器，推理时用解码器；系统研究 VAE 重建质量与 TTS 最终质量的非平凡关系
- DiT（扩散 Transformer）：直接在 Wav-VAE 潜在空间做去噪扩散，消除 Mel 频谱图中间表示
- 自适应投影引导（Adaptive Projection Guidance）：替代传统 CFG，提升生成质量

**关键图/公式**：系统架构对比——传统：文本→Mel→声码器→波形（两次误差）；AudioDiT：文本→Wav-VAE 潜在→解码→波形（一次误差）。消除中间表示是核心设计原则。

**实验设置**：
- 规模/数据：LongCat-AudioDiT-3.5B；Seed-TTS 基准（Seed-ZH 和 Seed-Hard 两个测试集）；SIM（说话人相似度）作为核心指标
- 对比：Seed-TTS（之前闭源 SOTA）；SIM 分数对比

**最强证据**：Seed-Hard（更难的零样本克隆集合）SIM 从 0.776→0.797，提升 2.7%——在声音克隆任务中这是可被用户感知的改善，且超越了闭源系统。

**最弱证据**：只报告了 SIM 分数，缺少 WER（词错误率）、自然度 MOS 等完整评估；Wav-VAE 重建质量瓶颈的分析是定性的；3.5B 参数量较大，推理延迟未报告。

**可复用点**：训练-推理不匹配修正（发现并修复长期存在的配置不一致问题）——这是工程实现中容易被忽视的细节，对所有基于扩散模型的音频生成系统都值得检查。

**和哪些论文相关**：
- [[Wiki/论文笔记/11_多模态生成/97_LongCat-Video-Avatar-1.5音频驱动视频生成]] — 同属 LongCat 系列，AudioDiT 是 Video-Avatar 的音频质量基础
- [[Wiki/概念/01_架构技术/原生多模态嵌入]] — 在潜在空间统一建模多模态的设计思路

**我能拿它做什么**：
- 基于开源权重构建高质量零样本声音克隆应用
- 在扩散 TTS 工程实践中，系统检查训练-推理配置的一致性
- 用自适应投影引导（APG）替代 CFG 作为改善生成质量的低成本方案

**3天后要回忆的问题**：
1. 为什么直接在 Wav-VAE 潜在空间建模比经过 Mel 频谱图的路径更优？
2. Wav-VAE 和普通 VAE 的区别是什么？重建质量如何影响 TTS 最终质量？
3. Seed-ZH 和 Seed-Hard 测试集的区别是什么？为什么 Hard 上的提升更有意义？
4. 自适应投影引导（APG）相比 CFG 的改进机制是什么？
5. LongCat-AudioDiT 和 AR（自回归）路线的 TTS 各有什么优缺点？

## 原子概念索引
- [[Wiki/概念/01_架构技术/原生多模态嵌入]] — 在波形潜在空间统一建模是原生模态表示的音频实例
- [[Wiki/概念/06_多模态生成/音频Token化与TTS]] — AudioDiT 是该概念在扩散 TTS 路线的代表性实现，直接在 Wav-VAE 潜在空间扩散跳过 Mel 频谱图
- [[Wiki/概念/06_多模态生成/扩散模型]] — AudioDiT 是扩散模型在 TTS 领域的前沿应用
- [[Wiki/概念/01_架构技术/多模态离散Token化]] — AudioDiT 的连续波形潜在空间方案是离散 Token 路线的对比参照
