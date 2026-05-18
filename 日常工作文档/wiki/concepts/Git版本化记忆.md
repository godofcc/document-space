---
title: "Git版本化记忆"
type: concept
tags: [版本控制, 记忆管理]
sources: [摘要-agent-memory-frameworks-comparison]
last_updated: 2026-05-18
---

## 定义
将Git的版本控制思想应用于记忆管理，每次记忆变更都产生一个commit，实现记忆的完整历史追踪、可回溯、并发安全。

## 关键信息
- **实现框架**：Letta
- **双存储设计**：
  - git：真实来源
  - pg数据库：快速读缓存
- **commit信息**：携带agent id、时间戳、变更原因
- **核心优势**：
  - 不可变性：内容寻址存储
  - 防静默损坏：任何修改都留有记录
  - 完整历史：每次记忆修改都能回溯
  - 并发安全：多个sleep time agent可以用git worktree隔离修改
  - 可审计：每个commit就是一条审计记录

## 关联连接
- [[摘要-agent-memory-frameworks-comparison]] — 来源
- [[Letta]] — 应用框架