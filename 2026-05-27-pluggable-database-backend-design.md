# CodeGraph 可插拔数据库后端 + REST Query API Server 设计文档

**日期**: 2026-05-27
**状态**: 已确认，待实现

---

## 1. 需求背景

CodeGraph 当前仅支持 SQLite 作为存储后端。SQLite 作为本地文件数据库，在单用户场景下表现优秀，但存在以下局限：

1. **无法团队共享**：索引数据锁在 `.codegraph/codegraph.db` 文件中，每个开发者需独立索引
2. **大型项目性能瓶颈**：超大型 Android 项目（数万文件）在 SQLite 下的并发读取能力有限
3. **无法集中管理**：团队无法统一维护一份索引数据供多人查询

目标：保留 SQLite 作为零配置本地后端，新增 PostgreSQL 作为可配置远程后端，支持单机索引写入 + 团队只读共享查询。

---

## 2. 决策记录

| 决策 | 选择 | 理由 |
|------|------|------|
| 使用场景 | 团队共享，读多写少 | 一台机器索引，多台机器查询 |
| SQLite 定位 | 保留作为默认本地后端 | 保持 `codegraph init` 零配置体验 |
| 写入模型 | 单机索引写入，远程只读 | CodeGraph 写入模式是全量替换性质，不适合多写者并发 |
| 远程数据库 | PostgreSQL 优先 | `ts_vector` 替代 FTS5，JSONB 替代 JSON 列，纯 JS 客户端无原生依赖 |
| 架构方案 | 方案 A：Core + Dialect 插件 | 改动最小，一个包，增量交付 |
| 隔离层 | 薄 Dialect 层（非 Repository） | 只抽离约 25 个 SQL 方言差异点，不重写整个查询层 |
| Query 服务协议 | REST API | 最通用，Swagger 文档自动生成，任何客户端可调用 |

---

## 3. 架构设计

### 3.1 整体架构

```
codegraph (单 npm 包)
├── CLI 入口
│   ├── codegraph index/sync  (写入: 扫描 → Dialect → SQLite/Postgres)
│   ├── codegraph serve --mcp (本地 MCP，读 SQLite)
│   └── codegraph serve --api (远程 REST API，读 Postgres)
│
├── src/db/
│   ├── dialect.ts            (新增：方言接口 ~30个方法)
│   ├── sqlite-dialect.ts     (实现：包装现有 QueryBuilder)
│   ├── postgres-dialect.ts   (实现：新写)
│   ├── connection.ts         (新增：工厂方法，配置选择后端)
│   ├── queries.ts            (现有，逐步迁入 dialect)
│   └── schema.sql            (现有)
│
├── src/api/                  (新增：REST API Server)
│   ├── server.ts             (Fastify 入口)
│   ├── routes/               (按资源拆路由)
│   └── middleware/            (auth, rate-limit)
│
└── src/mcp/                  (现有 MCP，保持不变)
```

### 3.2 部署形态

```
开发者机器                     服务器
┌──────────────┐          ┌──────────────────┐
│ codegraph     │          │ Query API Server  │
│ index/sync    │──写────▶│ (Fastify+pg)      │
│               │          │                   │
│ codegraph     │          │                   │
│ serve --mcp   │──读────▶│                   │
└──────────────┘          └───────┬───────────┘
                                  │
                          ┌───────▼───────────┐
                          │   PostgreSQL       │
                          └───────────────────┘
```

- **Indexer** 在开发者机器上运行，通过 PostgresDialect 写入远程 PostgreSQL
- **Query API Server** 部署在服务器上，通过 Fastify 暴露 REST API 供团队查询
- **本地 MCP** 继续读 SQLite，零配置体验不变

---

## 4. Dialect 抽象层设计

### 4.1 核心接口

```typescript
// src/db/dialect.ts

interface DatabaseDialect {
  // 连接管理
  connect(): Promise<void>;
  close(): Promise<void>;
  isInTransaction(): boolean;
  transaction<T>(fn: () => Promise<T>): Promise<T>;

  // Schema 管理
  initializeSchema(): Promise<void>;
  runMigrations(fromVersion: number): Promise<void>;
  getCurrentVersion(): Promise<number>;

  // Node CRUD (批量优先)
  insertNodes(nodes: Node[]): Promise<void>;
  updateNode(node: Node): Promise<void>;
  deleteNodesByFile(filePath: string): Promise<void>;
  getNodeById(id: string): Promise<Node | null>;
  getNodesByFile(filePath: string): Promise<Node[]>;
  getNodesByKind(kind: NodeKind): Promise<Node[]>;
  getNodesByName(name: string): Promise<Node[]>;
  getNodesByQualifiedNameExact(name: string): Promise<Node[]>;
  getNodesByLowerName(name: string): Promise<Node[]>;
  findNodesByExactName(names: string[]): Promise<Node[]>;
  findNodesByNameSubstring(query: string, limit: number): Promise<Node[]>;

  // Edge CRUD
  insertEdges(edges: Edge[]): Promise<void>;
  deleteEdgesBySource(source: string): Promise<void>;
  deleteEdgesByTarget(target: string): Promise<void>;
  getOutgoingEdges(source: string, kinds?: EdgeKind[]): Promise<Edge[]>;
  getIncomingEdges(target: string, kinds?: EdgeKind[]): Promise<Edge[]>;
  findEdgesBetweenNodes(sources: string[], targets: string[]): Promise<Edge[]>;

  // File CRUD
  upsertFile(file: FileRecord): Promise<void>;
  deleteFile(filePath: string): Promise<FileRecord | null>;
  getFileByPath(filePath: string): Promise<FileRecord | null>;
  getAllFiles(): Promise<FileRecord[]>;
  getAllFilePaths(): Promise<string[]>;

  // Unresolved refs
  insertUnresolvedRefsBatch(refs: UnresolvedReference[]): Promise<void>;
  deleteUnresolvedByNode(nodeId: string): Promise<void>;
  getUnresolvedByName(name: string): Promise<UnresolvedReference[]>;
  getUnresolvedCount(): Promise<number>;
  getUnresolvedBatch(offset: number, limit: number): Promise<UnresolvedReference[]>;
  getAllNodeNames(): Promise<string[]>;

  // 全文搜索 (方言核心差异点)
  searchNodes(query: string, options: SearchOptions): Promise<SearchResult[]>;

  // 统计与维护
  getStats(): Promise<GraphStats>;
  runMaintenance(): Promise<void>;
  getJournalMode(): Promise<string>;
}
```

### 4.2 方言差异点（约 25 个）

| 类别 | SQLite | PostgreSQL |
|------|--------|------------|
| 批量插入 | `INSERT OR REPLACE` + 事务批量 | `INSERT ... ON CONFLICT` + `UNNEST` 或批量 VALUES |
| 布尔值 | `0/1 INTEGER` | `BOOLEAN` |
| 自增主键 | `INTEGER PRIMARY KEY AUTOINCREMENT` | `SERIAL` / `BIGSERIAL` |
| FTS | FTS5 虚拟表 + 触发器同步 | `ts_vector` 列 + GIN 索引 |
| 大小写不敏感 | `lower(name)` 表达式索引 | `LOWER(name)` 函数索引或 `CITEXT` |
| JSON 列 | `TEXT` 存 JSON，`safeJsonParse` 读 | `JSONB` 原生支持 |
| PRAGMA | `PRAGMA journal_mode=WAL` 等 | 无（Postgres 自身管理） |
| 连接池 | 无（单文件锁） | `pg.Pool` 连接池 |
| Schema 初始化 | `schema.sql` 一次性执行 | 迁移表 + `IF NOT EXISTS` |
| 参数绑定 | `@named` 参数 | `$1, $2` 位置参数 |
| 事务 | 同步 `db.transaction()}` | 异步 `client.query('BEGIN/COMMIT')` |
| 批量行数 | 内存允许即可 | 单条 INSERT 最多 65535 参数（需分批） |
| 维护操作 | `PRAGMA optimize` + `VACUUM` | `ANALYZE` + `VACUUM` |
| WAL 检查点 | `PRAGMA wal_checkpoint(PASSIVE)` | 无（Postgres 自动管理） |

### 4.3 PostgreSQL Schema

```sql
-- 与 SQLite schema 对齐，但使用 PostgreSQL 特性

-- nodes: 布尔值用 BOOLEAN，JSON 列用 JSONB
CREATE TABLE IF NOT EXISTS nodes (
    id TEXT PRIMARY KEY,
    kind TEXT NOT NULL,
    name TEXT NOT NULL,
    qualified_name TEXT NOT NULL,
    file_path TEXT NOT NULL,
    language TEXT NOT NULL,
    start_line INTEGER NOT NULL,
    end_line INTEGER NOT NULL,
    start_column INTEGER NOT NULL,
    end_column INTEGER NOT NULL,
    docstring TEXT,
    signature TEXT,
    visibility TEXT,
    is_exported BOOLEAN DEFAULT FALSE,
    is_async BOOLEAN DEFAULT FALSE,
    is_static BOOLEAN DEFAULT FALSE,
    is_abstract BOOLEAN DEFAULT FALSE,
    decorators JSONB,
    type_parameters JSONB,
    updated_at BIGINT NOT NULL
);

-- 全文搜索：ts_vector 列 + GIN 索引替代 FTS5
ALTER TABLE nodes ADD COLUMN IF NOT EXISTS search_vector tsvector
    GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(name, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(qualified_name, '')), 'B') ||
        setweight(to_tsvector('english', coalesce(docstring, '')), 'C') ||
        setweight(to_tsvector('english', coalesce(signature, '')), 'D')
    ) STORED;

CREATE INDEX IF NOT EXISTS idx_nodes_search ON nodes USING GIN(search_vector);

-- edges: 自增主键用 BIGSERIAL
CREATE TABLE IF NOT EXISTS edges (
    id BIGSERIAL PRIMARY KEY,
    source TEXT NOT NULL REFERENCES nodes(id) ON DELETE CASCADE,
    target TEXT NOT NULL REFERENCES nodes(id) ON DELETE CASCADE,
    kind TEXT NOT NULL,
    metadata JSONB,
    line INTEGER,
    col INTEGER,
    provenance TEXT DEFAULT NULL
);

-- files
CREATE TABLE IF NOT EXISTS files (
    path TEXT PRIMARY KEY,
    content_hash TEXT NOT NULL,
    language TEXT NOT NULL,
    size INTEGER NOT NULL,
    modified_at BIGINT NOT NULL,
    indexed_at BIGINT NOT NULL,
    node_count INTEGER DEFAULT 0,
    errors JSONB
);

-- unresolved_refs
CREATE TABLE IF NOT EXISTS unresolved_refs (
    id BIGSERIAL PRIMARY KEY,
    from_node_id TEXT NOT NULL REFERENCES nodes(id) ON DELETE CASCADE,
    reference_name TEXT NOT NULL,
    reference_kind TEXT NOT NULL,
    line INTEGER NOT NULL,
    col INTEGER NOT NULL,
    candidates JSONB,
    file_path TEXT NOT NULL DEFAULT '',
    language TEXT NOT NULL DEFAULT 'unknown'
);

-- project_metadata
CREATE TABLE IF NOT EXISTS project_metadata (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL,
    updated_at BIGINT NOT NULL
);

-- schema_versions
CREATE TABLE IF NOT EXISTS schema_versions (
    version INTEGER PRIMARY KEY,
    applied_at BIGINT NOT NULL,
    description TEXT
);
```

### 4.4 连接配置

```typescript
// src/db/connection.ts

interface DatabaseConfig {
  type: 'sqlite' | 'postgres';
  // SQLite
  sqlitePath?: string;  // 默认 .codegraph/codegraph.db
  // PostgreSQL
  postgresUrl?: string;  // postgresql://user:pass@host:5432/codegraph
  poolSize?: number;     // 默认 10
}
```

配置来源优先级：
1. 环境变量 `CODEGRAPH_DB_TYPE` / `CODEGRAPH_DB_URL`
2. `.codegraph/config.json` 中的 `database` 字段
3. 默认值：SQLite（零配置保持不变）

```typescript
// 配置文件示例
// .codegraph/config.json
{
  "database": {
    "type": "postgres",
    "postgresUrl": "postgresql://codegraph:secret@db.example.com:5432/codegraph",
    "poolSize": 10
  }
}
```

### 4.5 SQLite Dialect

对现有 `QueryBuilder` + `DatabaseConnection` 的薄封装，不改变任何现有行为：

- `DatabaseDialect` 接口方法内部调用现有 `QueryBuilder` 对应方法
- 返回值类型相同（`Promise<>` 包装，因为 Postgres 是异步的，接口统一为 `Promise`）
- 无性能回归——SQLite Dialect 内部本质是同步调用 + Promise.resolve 包装

### 4.6 PostgreSQL Dialect

新实现，使用 `pg` 库：

- 批量操作用多行 INSERT（最多 500 行/批，受参数上限约束）
- FTS 用 `ts_vector` 生成列 + GIN 索引，查询用 `to_tsquery`
- JSON 列用 `JSONB`，查询用 `@>` / `->` / `->>` 操作符
- 连接池用 `pg.Pool`，配置化 `poolSize`
- 异步全链路——所有方法返回 `Promise`

---

## 5. REST API Server 设计

### 5.1 入口

```bash
codegraph serve --api --port 3000 --db-url postgresql://...
```

### 5.2 路由设计

```
GET    /api/v1/status                    # 索引状态 + 统计
GET    /api/v1/search?q=&kind=&limit=   # 全文搜索
GET    /api/v1/nodes/:id                 # 单节点详情
GET    /api/v1/nodes?file=&kind=&name=   # 节点列表
GET    /api/v1/files                      # 文件列表
GET    /api/v1/files/:path/nodes          # 文件内节点
GET    /api/v1/graph/callers/:id?depth=  # 调用者链
GET    /api/v1/graph/callees/:id?depth=   # 被调用者链
GET    /api/v1/graph/impact/:id?depth=    # 影响半径
GET    /api/v1/graph/trace?from=&to=      # 端到端路径
GET    /api/v1/explore?symbols=&budget=   # 多符号探索
GET    /api/v1/context?query=&format=    # 上下文构建
```

### 5.3 技术选型

- **框架**: Fastify（比 Express 快 2-3 倍，内置 JSON Schema 校验和 TypeScript 支持）
- **数据库**: `pg` 库 + `pg.Pool` 连接池
- **认证**: Bearer Token（初始版本简单鉴权，通过环境变量 `CODEGRAPH_API_KEY` 配置）
- **限流**: Fastify 内置 rate-limit 插件

### 5.4 响应格式

```json
{
  "ok": true,
  "data": { ... },
  "meta": {
    "dbType": "postgres",
    "nodeCount": 9289,
    "stale": false
  }
}
```

错误响应：

```json
{
  "ok": false,
  "error": {
    "code": "NODE_NOT_FOUND",
    "message": "Node 'xxx' not found"
  }
}
```

---

## 6. 现有代码影响分析

### 6.1 需要修改的文件

| 文件 | 改动内容 |
|------|----------|
| `src/db/index.ts` | 新增 `DatabaseConfig` 读取，工厂方法改为按配置选择 Dialect |
| `src/db/queries.ts` | 方法签名逐步从同步转为异步，或先加异步包装 |
| `src/index.ts` | 构造函数接受 `DatabaseConfig`，所有 DB 调用点加 `await` |
| `src/extraction/index.ts` | 批量写入调用改为 `await` |
| `src/resolution/index.ts` | 批量写入调用改为 `await` |
| `src/graph/traversal.ts` | 查询调用改为 `await` |
| `src/graph/queries.ts` | 查询调用改为 `await` |
| `src/context/index.ts` | 查询调用改为 `await` |
| `src/mcp/tools.ts` | 工具处理器改为 `async` |
| `src/bin/codegraph.ts` | 新增 `serve --api` 子命令 |

### 6.2 新增文件

| 文件 | 内容 |
|------|------|
| `src/db/dialect.ts` | `DatabaseDialect` 接口定义 |
| `src/db/sqlite-dialect.ts` | SQLite Dialect 实现（薄封装 QueryBuilder） |
| `src/db/postgres-dialect.ts` | PostgreSQL Dialect 实现 |
| `src/db/connection.ts` | `createDialect(config)` 工厂方法 |
| `src/db/postgres-schema.sql` | PostgreSQL 建表语句 |
| `src/db/postgres-migrations.ts` | PostgreSQL 迁移逻辑 |
| `src/api/server.ts` | Fastify 入口 |
| `src/api/routes/*.ts` | REST 路由 |
| `src/api/middleware/auth.ts` | Bearer Token 鉴权 |
| `src/api/middleware/rate-limit.ts` | 限流 |

### 6.3 不需要修改的文件

| 文件/目录 | 原因 |
|-----------|------|
| `src/extraction/languages/*` | 提取器只产出 Node/Edge，不涉及存储 |
| `src/extraction/tree-sitter.ts` | 解析层完全与存储无关 |
| `src/extraction/svelte-extractor.ts` 等 | 同上 |
| `src/resolution/frameworks/*` | 框架解析器只产出 UnresolvedRef → Edge，不涉及存储 |
| `src/sync/watcher.ts` | 文件监听逻辑不变 |
| `src/installer/*` | 安装器不涉及存储 |
| `src/types.ts` | 类型定义不变 |

---

## 7. 交付计划

### Phase 1：Dialect 抽象层（SQLite Dialect）

1. 定义 `DatabaseDialect` 接口
2. 实现 `SqliteDialect`（薄封装现有 `QueryBuilder`，方法签名加 `Promise` 包装）
3. 实现 `createDialect(config)` 工厂方法
4. 替换 `src/index.ts` 中直接使用 `DatabaseConnection` 的地方
5. 全量测试通过，行为无变化

### Phase 2：PostgreSQL Dialect

1. 编写 PostgreSQL schema（与 SQLite 对齐）
2. 实现 `PostgresDialect`
3. 实现 `DatabaseConfig` 环境变量 + 配置文件读取
4. 索引 + 同步流程端到端测试
5. 全量测试通过

### Phase 3：REST API Server

1. Fastify 服务器骨架
2. 实现全部 REST 路由（代理到 Dialect 方法）
3. Bearer Token 鉴权
4. `serve --api` CLI 子命令
5. API 文档（Swagger/OpenAPI）
6. 集成测试

### Phase 4：性能验证

1. 大型项目（10000+ 文件）索引性能基准
2. 并发读取压力测试
3. FTS 查询准确性对比（PostgreSQL ts_vector vs SQLite FTS5）
4. 连接池配置调优

---

## 8. 风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| SQLite→PostgreSQL 全文搜索结果不一致 | 查询召回率下降 | 维护搜索等价测试套件，确保 ts_vector 和 FTS5 行为对齐 |
| 异步改造引入 bug | 现有功能回归 | Phase 1 先对 SQLite 做 Promise 包装，全量测试无变化后再加 Postgres |
| `pg` 库增加包体积 | npm 包变大约 300KB | `pg` 仅在配置为 Postgres 时才 require，SQLite 路径零影响 |
| 大型项目批量写入性能 | 索引速度慢 | 批量操作使用 multi-row INSERT + 分批（500行/批） |
| 网络延迟影响本地索引 | 索引变慢 | 索引写入使用异步批量提交，减少 RTT；本地场景继续用 SQLite |