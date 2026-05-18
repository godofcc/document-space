---
title: "MCP"
type: concept
tags: [概念, 协议]
sources: [raw/03-transcripts/2026-05-14-告别一切重复枯燥任务，CLI+Skill搭建浏览器AI自动化框架.md]
last_updated: 2026-05-14
---

## 定义

Model Context Protocol，模型上下文协议，用于将工具和数据接入 AI Agent 的标准协议。

## 关键信息

**传统方案（PlaywrightMCP）：**
- 将网页内容全部塞入 AI 上下文
- 将截图直接传入上下文
- Token 消耗较大

**对比方案（Playwright CLI）：**
- 按需加载，AI 决定是否读取详细信息
- 截图以本地文件形式存储
- 比 MCP 方案减少约 4 倍 Token 消耗

**趋势：**
CLI+Skill 的搭配使用正在成为技术发展趋势，可以取代传统的 MCP 方式。

## 关联连接

- [[摘要-cli-skill-browser-automation]] — 对比案例
- [[Playwright CLI]] — 替代方案
- [[Token 优化]] — 性能优势