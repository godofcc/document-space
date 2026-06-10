---
title: "DeepSeek"
type: entity
tags: [AI公司, 大模型, 推理模型]
sources: [Clippings/Bilibili/2026-05-27-AI Agent 推理模型怎么炼成.md]
last_updated: 2026-05-27
---

## 定义

DeepSeek 是中国AI公司深度求索推出的系列大语言模型产品，以开源和推理能力著称。

## 关键信息

- **V3**：DeepSeek 的基础大语言模型（Base Model），可以预测下一个 Token 但不具备推理能力
- **R1-Zero**：从 V3 出发仅使用强化学习（不经过 SFT）得到的实验性推理模型，推理能力较强但思维链可读性差
- **R1**：最终发布的推理模型，通过 Cold Start SFT → 推理导向 RL → 拒绝采样 SFT → 人类偏好对齐的完整流水线训练而成
- **GRPO 算法**：DeepSeek 在推理导向 RL 阶段使用的训练算法
- **核心发现**：仅基于结果奖励（Outcome-based Reward）的方案效果反而最好，不需要过程奖励

### DeepSeek R1 训练流水线

```
V3 (Base Model)
  → R1-Zero (纯RL实验, 可读性差)
  → Cold Start SFT (用R1-Zero数据+人工标注微调V3)
  → 推理导向 RL (GRPO)
  → 拒绝采样 + 非推理数据 SFT
  → 人类偏好对齐 (RLHF)
  → R1
```

## 关联连接

- [[推理模型]] — DeepSeek R1 是推理模型的代表性产品
- [[思维链]] — R1 的核心推理机制
- [[冷启动]] — R1 训练的关键步骤
- [[有监督微调]] — R1 训练流水线中的核心方法
- [[RLHF]] — R1 最终阶段的对齐方法
- [[结果奖励vs过程奖励]] — DeepSeek 的关键实验结论
- [[拒绝采样]] — R1 训练中用于生成高质量数据的方法
- [[摘要-ai-agent-reasoning-model]] — 来源