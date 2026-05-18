---
title: "主动Agent记忆"
type: concept
tags: [记忆范式, Agent主动性]
sources: [摘要-agent-memory-frameworks-comparison]
last_updated: 2026-05-18
---

## 定义
主动Agent记忆是让记忆系统不再等待用户指令，而是主动运行的agent范式，在用户不说话时也在看，甚至在用户下一句话还没发出来之前就预测用户大概会问什么并提前加载相关上下文。

## 关键信息
- **实现框架**：memU
- **核心创新**：从agent有记忆到记忆本身是一个agent
- **双agent架构**：
  - Main agent：听用户说话、调工具、生成回复
  - memu bot：只负责记忆，持续盯着每次交互，后台提取整理分类
  - 靠共享conversation messages列表通信
- **显著性感知记忆**（V1.4引入）：
  - 每条memory item带reinforcement counter计数器
  - 被检索一次计数+1次
  - 排序时根据强化次数加权，越常用的记忆越容易被再次召回
  - 模拟人类记忆的本质特性：越经常想到的事情越容易想起（给记忆加了一层肌肉记忆）
- **适用场景**：需要长期陪伴、长期学习的场景（个人AI助手、企业级客服、DevOps agent、研究型助手）

## 关联连接
- [[摘要-agent-memory-frameworks-comparison]] — 来源
- [[memU]] — 应用框架