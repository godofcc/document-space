---
title: "摘要-AI系统设计-Sora文生视频服务"
type: source
tags: [来源, B站, 系统设计, AI, 视频生成]
sources: [Clippings/Bilibili/2026-05-27-AI系统设计-Sora_文生视频服务.md]
last_updated: 2026-05-27
---

## 核心摘要

这是 s09g 在 B 站发布的一期视频，详细讲解 OpenAI 面试题"Design Text-to-Video System"的完整系统设计。视频从功能性与非功能性需求（scalability、latency、efficiency、合规）出发，逐层展开：准入控制（三层：限流→GPU资源检查→流量整形与优先级排队）、预处理（Safety Check → Prompt Enhancement → 二次 Check）、模型推理（Text Encoder → DiT去噪 → TAE/VAE解码）、后处理（超分辨率 → Safety Check → Watermark）、交付（CDN上传 + 推送通知 + 自适应下载）。重点讲解了 DAG 工作流与状态机设计、是否支持 Preemption 的权衡（结论：DiT推理阶段不建议中断，应在准入层就做控制）、故障恢复与服务降级策略（单GPU→实例级→区域级→召回离线GPU）、以及 GPU 并行方案计算（DiT 比 LLM 计算量大数个数量级，USP/CP+TP/GFG 并行、模型蒸馏、FP8量化等优化方向）。

## 关键要点

### 需求分析
- **功能需求**：文字提示词生成视频，15秒720p起步，企业级60秒1080p，异步交付
- **非功能需求**：日500万请求（峰值QPS 200）、端到端延迟<5分钟、GPU效率优化、合规（无公众人物/IP侵权、添加水印）

### 准入控制（三层）
1. Rate Limiter — 按用户subscription查quota，不足则upsell
2. Resource Check — 查线上GPU可用性，考虑多team资源分配和BYOC场景
3. Traffic Shaping — Queue + 优先级排队（free/paid/enterprise）

### 模型推理流水线
- Text Encoder（1-3个模型）→ Diffusion Transformer（30B参数，8 GPU gang scheduling）→ TAE/VAE Decoder
- 中间产物大小估算：text embedding ~0.7MB，720p 256帧 uint8 ~500MB，1080p ~2GB

### 故障降级梯度
- 单GPU故障 → 新开实例retry
- GPU实例故障（8卡全挂）→ 新实例 + load balancer failover
- 流量>容量 → 延长等待时间，Queue缓冲，降级免费用户
- 区域级故障 → 跨区域调度（利用时差）
- 极端容量不足 → 召回离线GPU（训练/批处理任务让路）

### GPU并行与优化
- DiT计算量估算：220P FLOPs vs Llama 3 70B的140T FLOPs，差约3个数量级
- 并行策略：CP4+TP2、USP（Unified Sequence Parallelism）、CFG Parallelism
- 优化：模型蒸馏、FP8量化、Cache DiT、减少生成分辨率/时长

## 关联连接
- [[Sora]] — OpenAI 文生视频产品
- [[OpenAI]] — 视频面试题来源公司
- [[s09g]] — 视频创作者
- [[Bilibili]] — 视频发布平台
- [[文生视频]] — 文字到视频生成的技术领域
- [[准入控制]] — 系统设计中多层准入策略
- [[DAG工作流]] — 有向无环图任务编排
- [[Gang Scheduling]] — 多GPU联合调度策略
- [[服务降级]] — 资源不足时保护核心用户
- [[Diffusion Transformer]] — 视频生成核心模型架构
- [[提示词增强]] — 提升输入质量的改写技术
- [[超分辨率]] — 提升视频分辨率的技术