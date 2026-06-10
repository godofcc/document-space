---
title: "DAG工作流"
type: concept
tags: [系统设计, 工作流, 分布式系统]
sources: [Clippings/Bilibili/2026-05-27-AI系统设计-Sora_文生视频服务.md]
last_updated: 2026-05-27
---

## 定义

DAG 工作流（Directed Acyclic Graph Workflow）是将复杂的多阶段任务建模为有向无环图的编排方式，每个节点代表一个处理阶段，边表示依赖关系。对于文生视频系统，DAG 明确了从准入到交付的完整执行顺序和依赖。

## 关键信息

### 文生视频系统的 DAG 阶段
1. **Admission Control** — 准入控制，决定请求是否进入系统
2. **Preprocessing** — 预处理（Safety Check → Prompt Enhancement → 二次 Check）
3. **Model Inference** — 模型推理（Text Encoder → DiT → TAE/VAE Decoder）
4. **Post-Processing** — 后处理（超分辨率 → Safety Check → Watermark）
5. **Deliver** — 交付（CDN 上传 → 通知用户）

### 与状态机结合
- 每个 DAG 节点内部包含多个 subtask
- 每个 subtask 有自己的状态机：ready → scheduling → running → complete / retry → failure
- 计算资源需求因阶段而异：Preprocessing 用 CPU/小GPU，Inference 用 8 GPU gang，Post-processing 用单 GPU

### 断点续传判断
- Preprocessing 阶段中间产物为文本，容易保存和恢复
- Inference 阶段模型的激活状态（30B参数 ≈ 60GB 显存）太大不值得保存
- 结论：Inference 阶段一旦开始不应中断，应通过 draining 让已完成任务交付

## 关联连接
- [[摘要-ai系统设计-sora-文生视频服务]] — 来源
- [[文生视频]] — DAG 的应用场景
- [[Gang Scheduling]] — Inference 阶段的调度策略