---
title: "Gang Scheduling"
type: concept
tags: [分布式系统, GPU, 调度]
sources: [Clippings/Bilibili/2026-05-27-AI系统设计-Sora_文生视频服务.md]
last_updated: 2026-05-27
---

## 定义

Gang Scheduling 是一种多资源联合调度策略，要求一组资源（如多个 GPU）必须同时分配给同一个任务并同时释放，避免部分资源空闲等待导致浪费。在文生视频系统中，DiT 模型推理需要 8 个 GPU 并行执行，必须采用 gang scheduling。

## 关键信息

### 为什么需要 Gang Scheduling
- DiT 模型参数量约 30B（60GB 显存），需要张量并行分布到多 GPU
- 如果只分配部分 GPU，其他已分配的 GPU 会空闲等等待，浪费昂贵算力
- 8 GPU 必须同时开始计算、同时结束，确保通信同步

### 与其他并行策略的配合
- **CP4 + TP2**：4路上下文并行 + 2路张量并行，适合 8 GPU 配置
- **USP（Unified Sequence Parallelism）**：合并 self-attention、cross-attention 和 FFN 的三次 reduce 为一次 unified ring communication

### 与 Preemption 的关系
- Gang scheduling 的任务不建议被抢占（preempt）
- 抢占应发生在 gang scheduling 之前，即准入控制阶段
- 30 秒推理时间不长，可做 draining 而非抢占

## 关联连接
- [[摘要-ai系统设计-sora-文生视频服务]] — 来源
- [[Diffusion Transformer]] — 需要 Gang Scheduling 的核心模型
- [[文生视频]] — 应用场景