# CLI 工具架构设计文档

## 概述

基于插件化架构的 CLI 工具，用于团队内部 API 的数据查询和任务执行。设计目标是为 AI 助手提供机器可解析的接口，同时保持人类可读性。

## 设计原则

1. **AI 友好**：结构化输出，清晰的错误信息，便于 AI 解析
2. **插件化**：核心功能模块化，易于扩展和维护
3. **可测试**：依赖注入，便于单元测试和集成测试
4. **可扩展**：支持新功能插件式添加，不影响现有代码

## 架构图

```
┌─────────────────────────────────────────────────────┐
│                    用户/AI 助手                        │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│                  CLI 入口层 (cmd/)                     │
│  root.go → profile/ → config/ → auth/ → update/      │
└─────────────────────┬───────────────────────────────┘
                      │
          ┌───────────┴───────────┐
          │   核心插件注册系统       │
          │  shortcuts/register.go │
          └───────────┬───────────┘
                      │
    ┌─────────────────┼─────────────────┐
    │                 │                 │
┌───▼────┐       ┌────▼────┐      ┌────▼────┐
│ Query  │       │  Task   │      │  File   │
│Plugin  │       │ Plugin  │      │ Plugin  │
└───┬────┘       └────┬────┘      └────┬────┘
    │                 │                 │
┌───▼─────────────────▼─────────────────▼────┐
│          共享基础设施层 (internal/)            │
│  client/, credential/, output/, vfs/,        │
│  cmdutil/, core/, validate/                  │
└───────────────────────────────────────────────┘
                      │
                      ▼
              ┌───────────────┐
              │  内部 API     │
              └───────────────┘
```

## 目录结构

```
cli-tool/
├── cmd/                          # CLI 入口层
│   ├── root.go                   # 根命令，命令注册
│   ├── bootstrap.go              # 启动逻辑
│   ├── build.go                  # 命令构建器
│   ├── global_flags.go           # 全局标志定义
│   ├── profile/                  # 多配置文件管理
│   │   ├── add.go
│   │   ├── list.go
│   │   ├── use.go
│   │   └── remove.go
│   ├── config/                   # 配置管理
│   │   ├── init.go
│   │   ├── show.go
│   │   └── remove.go
│   ├── auth/                     # 认证管理
│   │   ├── login.go
│   │   └── logout.go
│   └── update/                   # 自动更新
│       └── update.go
│
├── shortcuts/                    # 插件系统（核心功能）
│   ├── register.go               # 插件注册入口
│   ├── common/                   # 插件通用功能
│   │   └── runner.go             # 执行上下文和运行时
│   ├── query/                    # 数据查询插件
│   │   ├── query.go              # 查询命令实现
│   │   ├── database.go           # 数据库查询
│   │   └── log.go                # 日志查询
│   ├── task/                     # 任务执行插件
│   │   ├── task.go               # 任务命令实现
│   │   ├── compile.go            # 编译任务
│   │   └── monitor.go            # 任务状态监听
│   └── file/                     # 文件操作插件
│       ├── file.go               # 文件命令实现
│       └── download.go           # 文件下载
│
├── internal/                     # 内部基础设施
│   ├── client/                   # API 客户端
│   │   └── client.go             # HTTP 请求封装
│   ├── credential/               # 凭证管理
│   │   ├── provider.go           # 凭证提供者接口
│   │   └── apikey.go             # API Key 实现
│   ├── output/                   # 输出格式化
│   │   ├── format.go             # 格式化工具
│   │   └── json_envelope.go      # JSON 输出封装
│   ├── cmdutil/                  # 命令工具
│   │   ├── factory.go            # 工厂模式
│   │   └── completion.go         # 自动补全
│   ├── core/                     # 核心定义
│   │   ├── config.go             # 配置结构
│   │   └── profile.go            # 配置文件管理
│   ├── vfs/                      # 虚拟文件系统
│   │   └── fs.go                 # 文件系统抽象
│   └── validate/                 # 输入验证
│       └── path.go               # 路径验证
│
├── extension/                    # 扩展点
│   └── credential/               # 外部凭证扩展
│       └── env.go                # 环境变量凭证
│
├── main.go                       # 程序入口
├── go.mod
├── go.sum
├── Makefile                      # 构建脚本
├── README.md
└── AGENTS.md                     # AI 助手使用指南
```

## 命令结构

```
cli-tool                           # 根命令
├── profile                        # 配置文件管理
│   ├── add <name> --api-key      # 添加配置
│   ├── list                       # 列出所有配置
│   ├── use <name>                 # 切换配置
│   └── remove <name>              # 删除配置
├── config                         # 配置管理
│   ├── init                       # 初始化配置
│   └── show                       # 显示当前配置
├── auth                           # 认证
│   ├── login --api-key            # 登录（设置 API Key）
│   └── logout                     # 登出
├── query                          # 数据查询
│   ├── database <table>           # 数据库查询
│   │   ├── --filter <json>        # 过滤条件
│   │   ├── --fields <csv>         # 指定字段
│   │   └── --limit <n>            # 限制返回数量
│   └── log <service>              # 日志查询
│       ├── --time-range <range>   # 时间范围
│       └── --level <level>        # 日志级别
├── task                           # 任务执行
│   ├── compile <project>         # 创建编译任务
│   │   ├── --branch <branch>      # 分支
│   │   └── --params <json>        # 编译参数
│   ├── status <task-id>           # 查询任务状态
│   │   └── --watch                # 实时监听
│   └── list                       # 列出所有任务
│       ├── --status <status>      # 按状态筛选
│       └── --limit <n>            # 限制数量
└── file                           # 文件操作
    ├── download <url>             # 下载文件
    │   └── --output <path>        # 输出路径
    └── list <path>                # 列出文件
        └── --recursive             # 递归列出

全局选项:
  --format <json|table|csv|pretty>  输出格式
  --profile <name>                   使用指定配置
  --debug                            调试模式
  --help                             显示帮助
```

## 核心组件

### 1. 插件系统

**插件接口定义**
```go
type Plugin interface {
    Name() string                          // 插件名称
    Description() string                   // 插件描述
    Register(root *cobra.Command)          // 注册命令
    Dependencies() []string               // 依赖的其他插件
    Validate(ctx *RuntimeContext) error    // 上下文验证
}
```

**执行上下文**
```go
type RuntimeContext struct {
    ctx        context.Context
    Config     *core.CliConfig
    Cmd        *cobra.Command
    Format     string
    APIClient  *client.APIClient
    Credential credential.Credentials
    Output     io.Writer
    ErrOutput  io.Writer
}
```

### 2. API 客户端

```go
type APIClient struct {
    baseURL    string
    httpClient *http.Client
    credential credential.Credentials
    timeout    time.Duration
}

func (c *APIClient) DoRequest(ctx context.Context, method, path string, body interface{}) (*Response, error)
func (c *APIClient) DoStream(ctx context.Context, method, path string) (io.ReadCloser, error)
```

### 3. 凭证管理

```go
type Credentials interface {
    GetAPIKey() (string, error)
    Validate() error
}

// API Key 实现
type APIKeyCredentials struct {
    apiKey string
}
```

### 4. 输出格式化

```go
type Formatter interface {
    Format(data interface{}) ([]byte, error)
}

// JSON 输出封装
type JSONEnvelope struct {
    Ok      bool        `json:"ok"`
    Data    interface{} `json:"data,omitempty"`
    Error   *ErrorDetail `json:"error,omitempty"`
    Notice  *Notice     `json:"_notice,omitempty"`
}

type ErrorDetail struct {
    Type    string `json:"type"`
    Code    string `json:"code"`
    Message string `json:"message"`
    Hint    string `json:"hint,omitempty"`
}
```

## 数据流

```
┌─────────┐    ┌─────────┐    ┌──────────┐    ┌────────┐    ┌─────────┐
│  用户   │───▶│  Cobra  │───▶│  Plugin  │───▶│ Client │───▶│  API    │
│  输入   │    │ 参数解析 │    │  业务逻辑 │    │ HTTP   │    │  服务   │
└─────────┘    └────┬────┘    └────┬─────┘    └───┬────┘    └────┬────┘
                    │              │              │             │
                    ▼              ▼              ▼             ▼
                 全局验证       权限检查        请求发送      返回响应
                    │              │              │             │
                    └──────────────┴──────────────┴─────────────┘
                                         │
                                         ▼
                                   ┌──────────┐
                                   │  Output  │
                                   │ 格式化   │
                                   └────┬─────┘
                                        │
                              ┌─────────┴─────────┐
                              │                   │
                         ┌────▼────┐          ┌───▼────┐
                         │ stdout  │          │ stderr │
                         │ 数据输出│          │ 错误/日志│
                         └─────────┘          └────────┘
```

## 配置管理

### 配置文件结构

**位置**: `~/.cli-tool/config.yaml`

```yaml
profiles:
  default:
    api_key: "your-api-key"
    api_endpoint: "https://api.example.com"
  staging:
    api_key: "staging-api-key"
    api_endpoint: "https://staging-api.example.com"
current_profile: default

settings:
  timeout: 30s
  max_retries: 3
  default_format: json
```

### 配置加载流程

1. 读取配置文件
2. 验证配置完整性
3. 应用当前配置
4. 初始化凭证

## 错误处理策略

### 错误分类

1. **用户错误**（4xx）
   - 参数错误：`{"type": "invalid_argument", "code": "INVALID_PARAM", "message": "...", "hint": "..."}`
   - 认证失败：`{"type": "auth_error", "code": "UNAUTHORIZED", "message": "...", "hint": "run 'cli-tool auth login'"}`

2. **系统错误**（5xx）
   - 网络错误：`{"type": "network_error", "code": "NETWORK_ERROR", "message": "...", "hint": "check network connection"}`
   - API 错误：`{"type": "api_error", "code": "INTERNAL_ERROR", "message": "...", "hint": "retry later"}`

3. **配置错误**
   - 缺少配置：`{"type": "config_error", "code": "MISSING_CONFIG", "message": "...", "hint": "run 'cli-tool config init'"}`

### 错误处理规范

- 所有命令必须返回结构化错误
- 错误消息包含：类型、代码、描述、解决建议
- AI 助手可通过 `code` 判断错误类型并自动重试

## AI 助手集成

### AI 友好特性

1. **结构化输出**
   - 统一 JSON 格式
   - 明确的字段命名
   - 类型一致的返回值

2. **清晰的错误信息**
   - 机器可解析的错误码
   - 明确的错误分类
   - 可执行的解决建议

3. **幂等操作**
   - 查询操作可重复执行
   - 明确的状态管理
   - 支持条件查询

4. **进度反馈**
   - 长任务支持状态查询
   - 异步任务返回 task_id
   - 支持 watch 模式

### AI 助手使用示例

```bash
# 查询数据
cli-tool query database users --filter '{"status":"active"}' --limit 10

# 创建任务
cli-tool task compile my-project --branch main --params '{"target":"release"}'

# 监控任务
cli-tool task status task-123 --watch

# 下载文件
cli-tool file download https://example.com/file.zip --output ./file.zip
```

### AI 解析建议

AI 助手应遵循以下解析策略：

1. **成功响应**：检查 `ok: true`，读取 `data` 字段
2. **错误响应**：检查 `ok: false`，根据 `error.code` 决定操作
3. **异步操作**：检查返回的 `task_id`，使用 `task status` 查询状态
4. **分页数据**：检查 `pagination` 字段，按需请求下一页

## 测试策略

### 单元测试

- 测试每个插件的命令处理逻辑
- Mock API 客户端，测试错误处理
- 测试配置加载和验证

### 集成测试

- Dry-run 模式：验证请求结构，不调用真实 API
- Live 模式：需要测试环境凭证，验证完整流程

### E2E 测试

```
tests/cli_e2e/
├── dryrun/                       # Dry-run 测试
│   ├── query_test.go
│   ├── task_test.go
│   └── file_test.go
└── live/                         # Live 测试（需要凭证）
    ├── query_workflow_test.go
    ├── task_workflow_test.go
    └── file_workflow_test.go
```

## 性能考虑

- 使用连接池复用 HTTP 连接
- 支持并发请求（对于批量操作）
- 缓存常用查询结果
- 支持请求超时配置

## 安全考虑

- API Key 加密存储
- 支持环境变量覆盖
- 敏感信息不输出到日志
- 验证输入路径，防止路径遍历

## 扩展性

### 添加新插件

1. 在 `shortcuts/` 下创建新目录
2. 实现 `Plugin` 接口
3. 在 `shortcuts/register.go` 中注册
4. 编写测试用例

### 添加新凭证类型

1. 在 `internal/credential/` 下实现新类型
2. 在 `extension/credential/` 中注册
3. 更新配置结构

## 开发规范

- 遵循 Go 标准项目布局
- 使用 Cobra 管理命令
- 错误处理必须使用结构化错误
- 所有公共接口添加注释
- 遵循 Conventional Commits 规范

## 构建和发布

### Makefile 目标

```makefile
build:          # 构建二进制文件
test:           # 运行所有测试
unit-test:      # 运行单元测试
lint:           # 代码检查
fmt:            # 代码格式化
```

### 发布流程

1. 更新版本号
2. 生成 CHANGELOG
3. 构建 Multi-platform 二进制
4. 发布到 GitHub Releases
5. 更新文档

## 文档

- `README.md`: 用户文档
- `AGENTS.md`: AI 助手使用指南
- `docs/`: 开发者文档
- 代码注释：API 文档

---

**文档版本**: 1.0
**创建日期**: 2026-05-11
**作者**: CLI 工具开发团队
