---
title: "Sora"
type: entity
tags: [产品, AI, 视频生成, OpenAI]
sources: [Clippings/Bilibili/2026-05-27-AI系统设计-Sora_文生视频服务.md]
last_updated: 2026-05-27
---

## 定义

Sora 是 OpenAI 推出的文生视频（Text-to-Video）产品，能够根据文字提示词生成高质量视频内容。原为 OpenAI 系统设计面试题，后发展为实际产品。

## 关键信息

- 开发商：OpenAI
- 功能：从文字提示词生成视频
- 竞品方向：Meta MovieGen 等视频生成模型
- 核心架构：Diffusion Transformer（DiT）+ TAE/VAE 解码器
- 生成规格：基础 720p/15秒，企业级可达 1080p/60秒
- 资源需求：模型推理需 8 GPU 并行（gang scheduling）
- 计算量：单次生成约 220P FLOPs，远超传统 LLM 推理

## 关联连接
- [[摘要-ai系统设计-sora-文生视频服务]] — 来源
- [[OpenAI]] — 开发公司
- [[文生视频]] — 所属技术领域
- [[Diffusion Transformer]] — 核心模型架构