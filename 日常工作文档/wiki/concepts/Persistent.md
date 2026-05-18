---
title: "Persistent"
type: concept
tags: [概念, 持久化]
sources: [raw/03-transcripts/2026-05-14-告别一切重复枯燥任务，CLI+Skill搭建浏览器AI自动化框架.md]
last_updated: 2026-05-14
---

## 定义

将浏览器状态（如 cookie、登录信息、本地存储）持久化到磁盘，供后续会话复用。

## 关键信息

**作用：**
- 避免每次使用时重新登录
- 保持浏览器会话状态
- 提升自动化效率

**Playwright CLI 中的使用：**
- 使用 `--persistent` 参数
- 将登录状态写入磁盘
- 下次使用时继续读取

**应用场景：**
- 需要登录才能操作的服务
- 保持会话的自动化流程

## 关联连接

- [[摘要-cli-skill-browser-automation]] — 使用场景
- [[Playwright CLI]] — 支持此功能
- [[浏览器自动化]] — 应用领域