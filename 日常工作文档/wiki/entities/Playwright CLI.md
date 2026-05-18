---
title: "Playwright CLI"
type: entity
tags: [工具, 浏览器自动化, 微软]
sources: [raw/03-transcripts/2026-05-14-告别一切重复枯燥任务，CLI+Skill搭建浏览器AI自动化框架.md]
last_updated: 2026-05-14
---

## 定义

微软在 2026 年初开源的命令行浏览器自动化工具，通过 CLI 方式提供浏览器操作能力。

## 关键信息

相比传统的 PlaywrightMCP 方案，Playwright CLI 能够减少约 4 倍的 Token 消耗。核心优势在于按需加载：网页摘要、截图等内容都以文件形式存储在本地磁盘，由 AI 按需读取，而非全量塞入上下文。

支持 `--headed` 参数使用有头浏览器（可视化），默认为无头浏览器（后台静默运行）。支持 `--persistent` 参数将 cookie 和本地存储持久化，避免重复登录。

需要 Node.js 环境和 Chrome/Edge 浏览器配合使用。

## 关联连接

- [[摘要-cli-skill-browser-automation]] — 使用教程
- [[浏览器自动化]] — 应用领域
- [[MCP]] — 对比方案
- [[Node.js]] — 依赖环境