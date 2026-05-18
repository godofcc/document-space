---
title: "Letta"
type: entity
tags: [开源项目, 记忆系统]
sources: [摘要-agent-memory-frameworks-comparison]
last_updated: 2026-05-18
---

## 定义
Letta（原MemGPT，2023年10月论文，2024年9月更名）是将操作系统虚拟内存思想完整搬进Agent架构的记忆系统。

## 关键信息
- **三层记忆架构**：
  - CORE memory：核心内存，直接嵌入系统Prompt，LLM每次推理都能看到（类似CPU直接访问的RAM）
  - archival memory：归档内存，容量无限，走向量检索（类似磁盘）
  - recall memory：召回内存，存放所有历史对话消息（类似操作系统的日志系统）
- **CORE memory结构**：每个block是三元组（label命名空间、description说明、value实际内容、limit字符上限）
- **内存管理**：
  - 当上下文窗口快满时触发SUMMARIZER，默认驱逐30%的消息
  - 被驱逐的消息写进recall memory，仍可通过conversation search检索
  - 真正的虚拟内存无损分层：从in context移到out of context，未真正消失
- **Git版本化记忆**：
  - 双存储：git是真实来源，pg数据库是快速读缓存
  - 每次记忆变更产生git commit（携带agent id、时间戳、变更原因）
  - 好处：不可变性、内容寻址、防静默损坏、完整历史、回溯能力、并发安全、可审计
- **Sleeptime agent**：
  - 主agent只负责推理和回复（低延迟），每五步触发一次sleeptime agent
  - 后台用更大token预算做深度反思和memory block改进
  - 充分利用空闲时间进行自我改进

## 优势
目前开源里记忆自制能力最强的系统。

## 劣势
认知成本高、数据库强依赖（80+依赖包）。

## 关联连接
- [[摘要-agent-memory-frameworks-comparison]] — 来源
- [[Agent记忆]] — 核心概念
- [[虚拟内存]] — 设计思想
- [[Git版本化记忆]] — 核心特性