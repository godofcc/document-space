---
title: "Mem0"
type: entity
tags: [开源项目, 记忆中间件]
sources: [摘要-agent-memory-frameworks-comparison]
last_updated: 2026-05-18
---

## 定义
Mem0 是目前开源社区热度最高的Agent记忆中间件，由VC孵化，GitHub Star数达几万级别。

## 关键信息
- **架构**：三层（memory API、LLM推理+向量检索逻辑层、存储层）
- **五大工厂模式**：
  - LLM factory：支持17+个LLM提供商
  - Embedding factory：支持11+个embedding模型
  - Vector store factory：支持22+种向量存储
  - Graph store factory：支持4种图存储
  - Reranker factory：支持5种reranker
- **三种记忆类型**：
  - 语义记忆：抽象的事实性知识
  - 情景记忆：具体事件
  - 程序记忆：agent执行的完整步骤（逐字保留，用于崩溃后恢复执行状态）
- **核心设计**：
  - UUID幻觉处理：将UUID临时映射成简单整数012，LLM决定后再映射回真实UUID
  - 双存储并行：向量存储（语义相似搜索）+图存储（关系推理），用thread executor并行执行
  - 双prompt策略：User memory extraction + Agent memory extraction分离
  - 多层级作用域隔离：用户隔离、agent隔离、运行时隔离
- **性能数据**：相比直接用OpenAI memory，准确率高26%、响应快91%、token省90%

## 成本瓶颈
每次调用完整模式会调用2-5次LLM，token数随记忆规模线性增长，不适合高频实时写入场景，更适合用户维度、对话力度、中低频写入场景（如AI对话助手、客服系统、医疗健康追踪）。

## 关联连接
- [[摘要-agent-memory-frameworks-comparison]] — 来源
- [[Agent记忆]] — 核心概念
- [[记忆框架]] — 所属类别
- [[语义记忆]] — 记忆类型
- [[情景记忆]] — 记忆类型
- [[程序记忆]] — 记忆类型