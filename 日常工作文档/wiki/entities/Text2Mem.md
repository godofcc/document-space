---
title: "Text2Mem"
type: entity
tags: [开源项目, 记忆框架]
sources: [摘要-agent-memory-frameworks-comparison]
last_updated: 2026-05-18
---

## 定义
Text2Mem 是一个为记忆系统定义通用操作语言的框架，不是做记忆系统本身，而是给所有记忆系统提供一套标准化的指令集。

## 关键信息
- **12个原子操作**：将记忆操作收敛到12个原子操作，分成三个阶段
  - ENC阶段：encode负责记忆诞生
  - RET阶段：retrieve（纯数据取回）+ summarize（让LLM再生成摘要）
  - STO阶段：9个操作覆盖整个生命周期
- **五元JSON契约**：结构包含阶段、操作名、目标、参数、原数据五个字段
- **双层验证**：
  - 外层：JSON schema格式和结构验证
  - 里层：Pydantic业务逻辑拦截
- **安全机制**：
  - dryrun模拟执行
  - confirmation确认字段（二选一机制）
  - lock操作四种模式：read only、no delete、append only、custom
- **存储实现**：SQLite参考实现（目前短板），无HTTP API，向量以JSON文本存在text列里

## 设计哲学
不信任LLM但能兜住LLM，通过约束和验证确保记忆操作的安全性和可靠性。

## 关联连接
- [[摘要-agent-memory-frameworks-comparison]] — 来源
- [[Agent记忆]] — 核心概念
- [[记忆框架]] — 所属类别