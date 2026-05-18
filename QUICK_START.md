# Code Review Graph 使用指南

## 📌 项目概述

**Code Review Graph** 是一个基于 Tree-sitter 和 MCP 的代码审查知识图谱系统，通过构建代码的结构化图谱，帮助 AI 编程助手在代码审查时只读取必要文件，减少 8.2 倍的 token 消耗。

## 🏗️ 核心架构

```
┌──────────────────────────────────────────────────────────┐
│                      AI 编程工具层                         │
│  (Claude Code | Cursor | Windsurf | Zed | Codex | ...)   │
└───────────────────┬──────────────────────────────────────┘
                    │ MCP 协议
        ┌───────────┴──────────────────┐
        ▼                              ▼
┌─────────────────────┐      ┌─────────────────────┐
│  MCP Server (stdio) │      │   Skills & Hooks    │
│  - 28个工具         │◄─────│ - build-graph       │
│  - 5个工作流模板    │      │ - review-delta      │
│  - 语义搜索         │      │ - review-pr         │
└──────────┬──────────┘      └─────────────────────┘
           │
    ┌──────┴──────┬─────────────────┬─────────────┐
    ▼             ▼                 ▼             ▼
┌─────────┐  ┌─────────┐    ┌──────────┐  ┌──────────┐
│ Parser  │  │  Graph  │    │Incremental│  │ Analysis │
│         │  │  Store  │    │  Engine   │  │          │
│Tree-sitter│ │ SQLite  │    │git diff  │  │Blaster-  │
│AST 解析   │ │.cr-g/   │    │hash check│  │Radius    │
└─────────┘  └─────────┘    └──────────┘  └──────────┘
```

## 🚀 快速开始

### 安装
```bash
pip install code-review-graph
code-review-graph install          # 自动检测所有平台
code-review-graph build            # 首次构建图谱 (~10s for 500 files)
```

### 核心流程
```bash
# 1. 构建图谱（首次）
/code-review-graph:build-graph

# 2. 审查变更（日常）
/code-review-graph:review-delta

# 3. 审查 PR
/code-review-graph:review-pr

# 4. 可视化
code-review-graph visualize
open .code-review-graph/graph.html
```

---

## 🔄 是否需要 LLM？

### ✅ 无需 LLM，直接用 CLI 的功能

这些功能**可以直接命令行使用**，不依赖 AI：

#### 1. 基础管理
```bash
# 构建图谱
code-review-graph build

# 增量更新
code-review-graph update

# 自动监控文件变更
code-review-graph watch

# 查看图谱状态
code-review-graph status
```

#### 2. 可视化
```bash
# 生成交互式 HTML 图谱
code-review-graph visualize
open .code-review-graph/graph.html

# 导出为其他格式
code-review-graph visualize --format graphml   # Gephi
code-review-graph visualize --format svg       # 静态图
code-review-graph visualize --format cypher    # Neo4j
code-review-graph visualize --format obsidian  # Obsidian 笔记

# 启动本地服务器
code-review-graph visualize --serve
# 浏览器访问 http://localhost:8765
```

#### 3. 文档生成
```bash
# 自动生成 Wiki 文档
code-review-graph wiki
# 输出到 .code-review-graph/wiki/
```

#### 4. 变更检测
```bash
# 检测变更影响（简单输出）
code-review-graph detect-changes

# 简洁模式
code-review-graph detect-changes --brief

# 自定义对比基准
code-review-graph detect-changes --base HEAD~3
```

#### 5. 多仓库管理
```bash
# 注册仓库
code-review-graph register ~/project-a --alias mylib

# 列出已注册仓库
code-review-graph repos

# 取消注册
code-review-graph unregister mylib
```

#### 6. 多仓库监控守护进程
```bash
# 启动多仓库监控（后台运行）
crg-daemon start

# 查看状态
crg-daemon status

# 查看日志
crg-daemon logs --repo mylib -f

# 停止
crg-daemon stop
```

---

### 🤖 需要 LLM（通过 MCP）的智能功能

这些功能需要 AI 来理解代码和做推理：

#### 1. 智能代码审查
```
"Review my recent changes with risk scoring"
```
- 理解代码语义
- 识别潜在 bug
- 生成审查建议

#### 2. 自然语言查询图谱
```
"Who calls the authenticate function?"
"What are the dependencies of AuthService?"
"Show tests for src/auth/login.py"
```
- 语义理解
- 跨语言查询
- 上下文推理

#### 3. 架构洞察
```
"Show me the architecture of this project"
"Find architectural chokepoints"
"Identify unexpected coupling"
```
- 高级模式识别
- 抽象层级理解

#### 4. 重构建议
```
"Suggest refactoring opportunities"
"If I rename User class, what files are affected?"
```
- 影响范围分析
- 重构风险评估

---

## 📊 功能对比

| 功能类别 | CLI 独立使用 | 需要 LLM |
|---------|------------|---------|
| **构建/更新图谱** | ✅ `build`, `update`, `watch` | - |
| **查看统计信息** | ✅ `status` | - |
| **可视化图谱** | ✅ `visualize` (HTML/GraphML/SVG) | - |
| **生成文档** | ✅ `wiki` | - |
| **变更检测** | ✅ `detect-changes` (简单输出) | - |
| **多仓库管理** | ✅ `register/unregister/repos` | - |
| **守护进程** | ✅ `crg-daemon` | - |
| **智能审查** | - | ✅ 自然语言审查 |
| **语义搜索** | ✅ 部分支持（embeddings） | ✅ 智能结果解释 |
| **自然语言查询** | - | ✅ "Who calls X?" |
| **架构洞察** | - | ✅ 抽象分析 |
| **重构建议** | ✅ 部分支持（refactor_tool） | ✅ 上下文理解 |

---

## 💡 实际使用建议

### 纯命令行工作流（无 LLM）
```bash
# 1. 构建图谱
code-review-graph build

# 2. 启动自动更新
code-review-graph watch &

# 3. 修改代码后查看影响
code-review-graph detect-changes

# 4. 可视化查看
open .code-review-graph/graph.html

# 5. 生成文档
code-review-graph wiki
```

### LLM 增强工作流
```bash
# 1. 启动 MCP 服务器
code-review-graph serve

# 2. 在 AI 助手中自然语言交互
"Review my changes with risk scoring"
"Show me callers of authenticate_user"
"Find architectural hotspots"
```

---

## 🔍 直接查询 SQLite 图谱

如果你想直接查询图谱（无需 LLM），可以：
```bash
# 进入 SQLite
sqlite3 .code-review-graph/graph.db

# 查询示例
SELECT name, qualified_name, file_path FROM nodes WHERE kind = 'Function' LIMIT 10;

# 查看调用关系
SELECT e.kind, n1.name, n2.name
FROM edges e
JOIN nodes n1 ON e.source_qualified = n1.qualified_name
JOIN nodes n2 ON e.target_qualified = n2.qualified_name
WHERE e.kind = 'CALLS' LIMIT 20;
```

---

## 📋 CLI 命令速查

| 命令 | 说明 |
|------|------|
| `install` | 配置 AI 编程工具 |
| `build` | 完整构建图谱 |
| `update` | 增量更新（仅变更文件） |
| `watch` | 自动监控文件变更 |
| `status` | 显示图谱统计信息 |
| `visualize` | 生成可视化图表 |
| `wiki` | 生成 Wiki 文档 |
| `detect-changes` | 分析变更影响 |
| `register` | 注册仓库到多仓库注册表 |
| `unregister` | 从注册表移除仓库 |
| `repos` | 列出已注册仓库 |
| `serve` | 启动 MCP 服务器 |

---

## 🎯 核心特性

| 特性 | 说明 |
|------|------|
| **Token 优化** | 平均 8.2x 减少 |
| **增量更新** | <2秒完成 |
| **多语言支持** | 24+ 语言 + Jupyter |
| **爆炸半径分析** | 精准定位影响范围 |
| **社区检测** | Leiden 算法自动分组 |
| **执行流追踪** | 从入口点追踪调用链 |
| **向量搜索** | 语义搜索 (可选) |
| **多仓库管理** | 注册多个仓库跨库搜索 |
| **离线支持** | 核心功能完全离线 |

---

## 📝 总结

**不需要 LLM 也可以用大部分功能！**

- ✅ **可以独立用 CLI**：构建、可视化、文档、变更检测、多仓库管理
- ✅ **需要 LLM**：智能审查、自然语言查询、架构洞察、重构建议

**如果你只想做可视化和管理，不需要 LLM 完全够用！**

---

## 🔗 相关文档

- [项目架构文档](ARCHITECTURE.md)
- [使用指南](docs/USAGE.md)
- [命令参考](docs/COMMANDS.md)
- [故障排除](docs/TROUBLESHOOTING.md)