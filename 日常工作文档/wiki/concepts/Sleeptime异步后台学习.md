---
title: "Sleeptime异步后台学习"
type: concept
tags: [Agent学习, 异步处理]
sources: [摘要-agent-memory-frameworks-comparison]
last_updated: 2026-05-18
---

## 定义
Sleeptime agent是在用户不交互的时候持续自我改进的agent，利用空闲时间进行深度反思和记忆优化。

## 关键信息
- **实现框架**：Letta
- **工作机制**：
  - 主agent只负责推理和回复（低延迟）
  - 每五步触发一次sleeptime agent
  - 后台用更大token预算做深度反思和memory block改进
- **三大好处**：
  - 主路径低延迟
  - 后台可以用更大token预算做深度反思
  - 充分利用空闲时间

## 关联连接
- [[摘要-agent-memory-frameworks-comparison]] — 来源
- [[Letta]] — 应用框架