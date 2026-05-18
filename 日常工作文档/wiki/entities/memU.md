---
title: "memU"
type: entity
tags: [开源项目, 主动记忆系统]
sources: [摘要-agent-memory-frameworks-comparison]
last_updated: 2026-05-18
---

## 定义
memU是范式最激进的Agent记忆系统，让记忆本身变成一个24/7后台主动运行的Agent。

## 关键信息
- **核心创新**：从agent有记忆到记忆本身是一个agent，记忆从被动响应变为主动出击
- **双agent架构**：
  - Main agent：听用户说话、调工具、生成回复
  - memu bot：只负责记忆，持续盯着每次交互，后台提取整理分类
  - 靠共享conversation messages列表通信
- **文件系统**：概念领域，底层是数据库（文件夹对应category、文件对应memory item、符号链接对应交叉引用、挂载点对应resource）
- **Salient感知记忆**（V1.4引入）：
  - 每条memory item带reinforcement counter计数器
  - 被检索一次计数+1次
  - 排序时根据强化次数加权，越常用的记忆越容易被再次召回
  - 模拟人类记忆的本质特性：越经常想到的事情越容易想起（给记忆加了一层肌肉记忆）
- **性能数据**：在OMEO基准上跨所有推理任务的平均准确率12.09%（与Mem0同一把尺子）

## 适用场景
需要长期陪伴、长期学习的场景：个人AI助手、企业级客服、DevOps agent、研究型助手。

## 关联连接
- [[摘要-agent-memory-frameworks-comparison]] — 来源
- [[Agent记忆]] — 核心概念
- [[主动Agent记忆]] — 核心创新