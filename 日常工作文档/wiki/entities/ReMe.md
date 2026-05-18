---
title: "ReMe"
type: entity
tags: [开源项目, 记忆框架, 阿里巴巴]
sources: [摘要-agent-memory-frameworks-comparison]
last_updated: 2026-05-18
---

## 定义
ReMe是阿里巴巴AgentScope团队出品的记忆框架，核心理念是"文件即记忆"，把记忆的控制权和透明度还给用户。

## 关键信息
- **核心理念**：记忆直接存成markdown文件，用户打开就能看见、可以直接编辑、可以git版本控制
- **内部两套系统**：
  - Remy light：文件记忆，用于短期工作记忆
  - ReMe本体：向量记忆，用于长期语义记忆
  - 两套系统时间维度错开
- **技术亮点**：
  - Delta file watcher增量监控：检测文件是否纯追加模式，只处理新增部分，节省92%的API调用
  - When to use和content分离：向量嵌入建在when to use上（用户查询和when to use天然语义接近），大幅提升召回率
  - AI自主记忆管理：把文件操作工具给AI，让AI自己决定如何组织记忆
- **性能数据**：千问38B+ReMe后，综合得分超过没有记忆的千问34B

## 设计哲学
好的记忆系统可以让小模型打过大模型；记忆透明、用户可控。

## 关联连接
- [[摘要-agent-memory-frameworks-comparison]] — 来源
- [[AgentScope]] — 所属框架
- [[Agent记忆]] — 核心概念
- [[文件即记忆]] — 核心理念