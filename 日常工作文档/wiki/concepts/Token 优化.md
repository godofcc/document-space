---
title: "Token 优化"
type: concept
tags: [概念, 性能优化]
sources: [raw/03-transcripts/2026-05-14-告别一切重复枯燥任务，CLI+Skill搭建浏览器AI自动化框架.md]
last_updated: 2026-05-14
---

## 定义

在 AI Agent 执行任务时，通过优化策略减少 Token 消耗的方法。

## 关键信息

**优化策略：**

1. **按需加载**
   - Playwright CLI 将网页摘要、截图以文件形式存储
   - AI 根据需要选择读取，而非全量塞入上下文
   - 相比 MCP 方案减少 4 倍 Token 消耗

2. **Skill 复用**
   - 将首次执行经验固化为 Skill
   - 避免重复试错和探索
   - 实测 Token 消耗降低约 10 倍（41% → 5%）

3. **脚本固化**
   - 将完全固定的流程编写为脚本
   - 直接执行脚本，实现 0 Token 消耗

**效果对比：**
- Playwright CLI vs PlaywrightMCP：减少 4 倍
- 无 Skill vs 有 Skill：减少约 10 倍
- AI 执行 vs 脚本执行：0 Token 消耗

## 关联连接

- [[摘要-cli-skill-browser-automation]] — 应用实例
- [[Skill]] — 关键优化手段
- [[Playwright CLI]] — 支持按需加载的工具