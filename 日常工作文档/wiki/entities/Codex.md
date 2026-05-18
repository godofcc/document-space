---
title: "Codex"
type: entity
tags: [工具, AI Agent]
sources: [raw/03-transcripts/2026-05-14-告别一切重复枯燥任务，CLI+Skill搭建浏览器AI自动化框架.md]
last_updated: 2026-05-14
---

## 定义

AI Agent 框架之一，支持通过 /skills 命令列出和调用已安装的 Skill。

## 关键信息

与 Claude Code 类似，可以配合 Playwright CLI 使用。通过将 Skill 存放在项目目录的 `.codex` 文件夹下即可完成配置。

在视频演示中，Codex 成功使用 Playwright CLI 打开浏览器、自动输入查询并获取结果。

## 关联连接

- [[摘要-cli-skill-browser-automation]] — 应用案例
- [[Claude Code]] — 类似框架
- [[Skill]] — 能力扩展机制