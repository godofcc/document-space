# 本地文档搜索 CLI 工具设计文档

## 项目概述

开发一个基于 Go 的 CLI 工具，用于搜索本地 Markdown 文档。工具使用 grep 进行文本搜索，集成 LLM 对搜索结果进行总结和回答用户问题。

**技术栈：**
- 语言：Go
- 数据库：SQLite
- 搜索引擎：ripgrep
- 文档格式：Markdown
- 更新方式：手动同步 + Git 管理

---

## 核心需求

### 功能需求
1. **文档索引**：扫描 Markdown 文件并建立索引
2. **文档搜索**：支持关键词搜索，返回 LLM 总结的答案
3. **手动同步**：支持手动触发索引更新
4. **配置管理**：支持配置文件管理文档目录、LLM 参数等

### 非功能需求
- 实时性：通过手动 sync 命令保证索引最新
- 性能：利用 SQLite 索引 + grep 加速搜索
- 可扩展性：支持添加更多文件格式、LLM 提供商

---

## 整体架构

### 组件划分

```
┌─────────────────────────────────────────────────────┐
│                      CLI 入口层                        │
│              (Cobra: 命令解析和路由)                   │
└─────────────────────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  sync 命令   │  │ search 命令  │  │  config 命令 │
└──────────────┘  └──────────────┘  └──────────────┘
        │                │
        ▼                ▼
┌──────────────┐  ┌──────────────┐
│ 索引管理器   │  │  搜索引擎    │
└──────────────┘  └──────────────┘
        │                │
        ▼                ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ SQLite DB    │  │   ripgrep    │  │  LLM 客户端  │
└──────────────┘  └──────────────┘  └──────────────┘
```

### 核心组件

1. **CLI 入口层**
   - 使用 Cobra 框架
   - 负责命令解析和路由

2. **索引管理器**
   - SQLite 数据库操作
   - Markdown 文件解析

3. **搜索引擎**
   - 集成 ripgrep 进行文本搜索
   - 收集匹配结果和上下文

4. **LLM 客户端**
   - 调用 LLM API（OpenAI/Anthropic/Ollama）
   - 处理总结请求

5. **配置管理**
   - 读取和管理配置文件

---

## 数据库设计

### documents 表

```sql
CREATE TABLE documents (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    file_path TEXT UNIQUE NOT NULL,           -- 文件路径
    file_name TEXT NOT NULL,                  -- 文件名
    title TEXT,                               -- Markdown 第一级标题
    content_preview TEXT,                     -- 内容摘要（前500字）
    tags TEXT,                                -- 标签（逗号分隔）
    last_modified INTEGER NOT NULL,           -- 文件修改时间戳（秒）
    indexed_at INTEGER NOT NULL,              -- 索引时间戳
    created_at INTEGER NOT NULL               -- 记录创建时间
);

CREATE INDEX idx_file_path ON documents(file_path);
CREATE INDEX idx_file_name ON documents(file_name);
CREATE INDEX idx_title ON documents(title);
CREATE INDEX idx_tags ON documents(tags);
```

---

## 核心流程

### 1. 文档同步流程 (docs sync)

```
用户执行 docs sync
    ↓
[步骤1] 扫描文档目录
    - 递归扫描目录
    - 筛选匹配 include_patterns 的文件
    - 排除 exclude_patterns 的文件
    ↓
[步骤2] 解析每个文件
    - 读取文件内容
    - 提取文件元数据（路径、修改时间）
    - 解析 Markdown 结构（提取标题、标签）
    ↓
[步骤3] 更新索引
    - 查询现有索引
    - 对比文件列表：
      - 新文件：插入数据库
      - 已存在文件：检查 last_modified，有变化则更新
      - 数据库中有但文件不存在：删除记录
    ↓
[步骤4] 输出结果
    - 统计新增/更新/删除的文件数
```

### 2. 文档搜索流程 (docs search)

```
用户执行 docs search "查询词"
    ↓
[步骤1] 从索引获取候选文件
    - 在 SQLite 中全文搜索 file_name, title, content_preview
    - 使用 LIKE 或 FTS（如果启用）
    - 返回匹配的文件列表
    ↓
[步骤2] 对每个候选文件执行 grep
    - 调用 ripgrep 搜索查询词
    - 获取匹配的行号和上下文（前后各 3 行）
    - 收集所有匹配片段
    ↓
[步骤3] 构建提示词上下文
    - 按相关性排序匹配片段
    - 包含文件名、标题、匹配内容
    - 控制总长度在 LLM 上下文限制内
    ↓
[步骤4] 调用 LLM
    - 发送提示词 + 上下文
    - 要求 LLM 总结并回答问题
    ↓
[步骤5] 输出结果
    - 显示 LLM 返回的答案
    - 可选：显示来源文件引用
```

---

## 项目结构

```
local-doc/
├── cmd/
│   ├── root.go           # CLI 入口，初始化 cobra 应用
│   ├── search.go         # search 命令实现
│   ├── sync.go           # sync 命令实现
│   └── config.go         # config 命令（可选）
├── internal/
│   ├── index/
│   │   ├── db.go         # SQLite 数据库操作
│   │   └── parser.go     # Markdown 文件解析
│   ├── search/
│   │   └── grep.go       # ripgrep 集成和结果收集
│   ├── llm/
│   │   └── client.go     # LLM 客户端（OpenAI/Anthropic/Ollama）
│   └── config/
│       └── config.go     # 配置文件读取和管理
├── config.yaml           # 配置文件（示例）
├── go.mod
├── go.sum
└── main.go               # 入口文件
```

---

## 配置文件设计

```yaml
# config.yaml
docs:
  path: "./docs"                    # 文档根目录
  include_patterns:
    - "*.md"
  exclude_patterns:
    - "*/.git/*"
    - "*/node_modules/*"

database:
  path: "./docs.db"                 # SQLite 数据库路径

llm:
  provider: "openai"                # 提供商: openai, anthropic, ollama
  api_key: ""                       # API Key（建议从环境变量读取）
  model: "gpt-4"                    # 模型名称
  base_url: ""                      # 自定义端点（可选）
  max_context: 8000                 # 发送给 LLM 的最大上下文长度（字符数）
  timeout: 30                       # 请求超时时间（秒）
```

**配置加载优先级：**
1. 命令行参数
2. 环境变量
3. 配置文件
4. 默认值

---

## 依赖库

| 组件 | 库 | 用途 |
|------|-----|------|
| CLI | `github.com/spf13/cobra` | 命令行框架 |
| 配置 | `github.com/spf13/viper` | 配置管理 |
| SQLite | `github.com/mattn/go-sqlite3` | 数据库驱动 |
| Markdown | `github.com/yuin/goldmark` | Markdown 解析 |
| 文件遍历 | `github.com/karrick/godirwalk` | 快速文件遍历 |
| HTTP | `net/http` | HTTP 请求（LLM API） |
| 日志 | `go.uber.org/zap` | 结构化日志 |

---

## 命令接口设计

### docs sync

```bash
# 同步文档索引
docs sync

# 可选参数
docs sync --config /path/to/config.yaml
docs sync --verbose  # 输出详细日志
```

**输出示例：**
```
Scanning documents...
  ✓ Found 45 Markdown files
  ✓ Parsed 42 files
  ✓ Indexed 40 files
  ✓ Updated 2 files
  ✓ Removed 3 files
Sync completed in 1.2s
```

### docs search

```bash
# 搜索文档
docs search "休息休息"

# 可选参数
docs search "查询词" --limit 5          # 限制搜索文件数
docs search "查询词" --context-lines 5  # 调整上下文行数
docs search "查询词" --raw              # 输出原始 grep 结果（不调用 LLM）
```

**输出示例：**
```
根据文档内容，关于"休息休息"的建议如下：

1. **番茄工作法**：每工作25分钟，休息5分钟
2. **合理规划**：工作间隙安排10-15分钟的深度休息
3. **睡眠质量**：保证每天7-8小时高质量睡眠

来源：
  - docs/productivity/pomodoro.md:15
  - docs/health/sleep.md:8
```

---

## LLM 提示词设计

### 搜索提示词模板

```
你是一个文档助手，根据以下文档片段回答用户的问题。

用户问题：{{.Question}}

文档片段：
{{range .Fragments}}
### 文件：{{.FileName}} ({{.Title}})
{{.Content}}
---
{{end}}

请基于上述文档内容回答用户问题。只回答问题相关的内容，不要编造信息。如果文档中没有相关信息，请明确说明。
```

---

## 错误处理

### 常见错误场景

1. **配置文件不存在**
   - 使用默认配置
   - 提示用户创建配置文件

2. **文档目录不存在**
   - 提示用户检查配置
   - 退出并返回错误

3. **SQLite 数据库错误**
   - 提示用户检查权限
   - 尝试重新初始化数据库

4. **LLM API 调用失败**
   - 捕获错误并重试（最多 3 次）
   - 失败后返回 grep 原始结果
   - 提示用户检查 API Key 和网络

5. **无搜索结果**
   - 提示用户尝试其他关键词
   - 建议执行 docs sync 更新索引

---

## 测试策略

### 单元测试
- 索引管理器的 CRUD 操作
- Markdown 解析器
- LLM 客户端的请求构建

### 集成测试
- sync 命令端到端测试
- search 命令端到端测试
- 使用测试数据库和测试文档目录

### 测试工具
- Go 标准库 `testing`
- `github.com/stretchr/testify` 断言库

---

## 性能优化

1. **索引查询优化**
   - 为常用字段添加索引
   - 考虑使用 SQLite FTS5 进行全文搜索

2. **grep 性能**
   - ripgrep 本身已优化
   - 限制搜索的文件数和上下文大小

3. **LLM 调用优化**
   - 限制上下文长度
   - 缓存相同查询的结果（可选）

4. **并发处理**
   - 并行解析多个文件（sync 时）
   - 并行执行 grep（search 时）

---

## 未来扩展

1. **支持更多文件格式**
   - PDF
   - Word
   - HTML

2. **更智能的搜索**
   - 支持语义搜索（可选集成向量数据库）
   - 支持正则表达式
   - 支持搜索历史记录

3. **交互式搜索**
   - 支持交互式对话
   - 支持追问和澄清

4. **Web UI**
   - 提供 Web 界面搜索
   - 支持文档预览

5. **Git 集成**
   - 自动检测 git 状态
   - 提示用户拉取最新文档

---

## 开发计划

见后续实现计划文档。