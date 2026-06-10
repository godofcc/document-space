---
title: "Diffusion Transformer"
type: concept
tags: [AI, 模型架构, 视频生成]
sources: [Clippings/Bilibili/2026-05-27-AI系统设计-Sora_文生视频服务.md]
last_updated: 2026-05-27
---

## 定义

Diffusion Transformer（简称 DiT）是视频生成任务的核心模型架构，结合了扩散模型（Diffusion Model）的去噪能力和 Transformer 的全局注意力机制。与 LLM 的自回归生成不同，DiT 每步同时生成所有 token，且需经过约 50 步去噪迭代，计算量远大于语言模型。

## 关键信息

### 架构组成
- **Text Encoder**（1-3个模型）：将文本编码为 embedding
- **DiT 核心**：30B 参数的密集模型，分布在 8 个 GPU 上，接收 latent 输入进行去噪
- **TAE/VAE Decoder**：将去噪后的 latent 解码为视频帧

### 计算量估算
- 输入经过 TAE 压缩后：sequence length ≈ 73K（48×48×32）
- 计算公式：24 × n × d² × L + 4 × n² × d × L × s
- 简便估算：2 × 30B × 73K × 50步 ≈ 220P FLOPs
- 对比 Llama 3 70B（1K sequence）：140T FLOPs，差约 3 个数量级

### 优化方向
- 模型蒸馏：生产环境使用蒸馏后的小模型
- 并行策略：CP4 + TP2、USP（Unified Sequence Parallelism）
- 精度优化：FP8 量化
- 缓存优化：Cache DiT 减少重复计算

## 关联连接
- [[摘要-ai系统设计-sora-文生视频服务]] — 来源
- [[Sora]] — 使用 DiT 的产品
- [[文生视频]] — 技术应用领域
- [[Gang Scheduling]] — DiT 推理的调度策略