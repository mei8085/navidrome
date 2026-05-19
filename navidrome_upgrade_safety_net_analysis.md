# Navidrome 升级安全网分析：数据库迁移与配置校验

## 1. 数据库迁移设计

### 1.1 迁移框架与执行流程

Navidrome 使用 [goose](https://github.com/pressly/goose) 作为数据库迁移框架，构建了一套完整的 SQLite 迁移体系。

**核心执行流程**（`db/db.go:75-117`）：

```go
func Init(ctx context.Context) func() {
    db := Db()
    // 1. 临时禁用外键约束，允许迁移中重建表
    _, err := db.ExecContext(ctx, "PRAGMA foreign_keys=off")
    
    // 2. 检查是否有待执行的迁移
    hasSchemaChanges := hasPendingMigrations(ctx, db, migrationsFolder)
    
    // 3. 执行所有迁移
    err = goose.UpContext(ctx, db, migrationsFolder)
    
    // 4. 迁移后优化（可选）
    if hasSchemaChanges && conf.Server.DevOptimizeDB {
        _, err = db.ExecContext(ctx, "PRAGMA optimize")
    }
    
    // 5. 恢复外键约束（defer）
}
```

**设计亮点**：
- **安全关闭**：迁移前后自动管理外键约束，避免迁移时的参照完整性错误
- **静默初始化**：新数据库（无 `goose_db_version` 表）静默执行迁移，不输出日志
- **优化时机**：仅在有实际 schema 变更时执行 `PRAGMA optimize`
- **原子性**：每个迁移在事务中执行，失败自动回滚

### 1.2 迁移顺序与版本管理

迁移文件采用 **时间戳前缀命名** 保证执行顺序：

```
20200130083147_create_schema.go          # 初始 schema
20200131183653_standardize_item_type.go  # 标准化字段
20200208222418_add_defaults_to_annotations.go
...
20260220173400_add_fts5_search.go        # FTS5 全文搜索
20260513173954_move_ss_before_input.go   # 最新迁移
```

**迁移注册机制**：每个迁移文件通过 `init()` 函数注册到 goose：

```go
func init() {
    goose.AddMigrationContext(Up20200130083147, Down20200130083147)
}
```

### 1.3 默认值约束设计思路

Navidrome 对默认值约束的设计经历了明显的演进，体现了对数据完整性的逐步重视。

**阶段一：初始设计（2020年）**

在 `20200130083147_create_schema.go` 中，核心表采用 **NOT NULL + DEFAULT** 模式：

```sql
create table if not exists album (
    id varchar(255) not null primary key,
    name varchar(255) default '' not null,          -- 字符串默认空串
    year integer default 0 not null,                -- 数值默认 0
    compilation bool default FALSE not null,        -- 布尔默认 false
    created_at datetime,                            -- 时间戳允许 NULL
    updated_at datetime
);
```

**设计原则**：
- 业务字段强制 NOT NULL，避免 NULL 带来的三值逻辑复杂性
- 字符串类型默认 `''`，数值类型默认 `0`，布尔类型默认 `FALSE`
- 时间戳字段允许 NULL，表示"未设置"状态

**阶段二：遗留数据修复（2024年）**

`20240122223340_add_default_values_to_null_columns.go.go` 是一个里程碑式的迁移，对历史原因产生的 NULL 字段进行全面修复。采用 **"新增-迁移-替换"** 三步法：

```sql
-- 1. 新增带默认值的新列
alter table album add image_files_new varchar not null default '';

-- 2. 将旧列数据迁移到新列（NULL 转为默认值）
update album set image_files_new = image_files where image_files is not null;

-- 3. 删除旧列，重命名新列
alter table album drop image_files;
alter table album rename image_files_new to image_files;
```

该迁移修复了 `album`（13列）、`artist`（3列）、`media_file`（23列）、`share`（7列）共 46 个字段的 NULL 问题。

**阶段三：安全添加 NOT NULL 列**

`migration.go:85-123` 提供了 `createAddColumnFunc` 工具函数，专门处理 SQLite 中添加 NOT NULL 列的限制：

```go
func createAddColumnFunc(ctx context.Context, tx *sql.Tx) addColumnFunc {
    return func(tableName, columnName, columnType, defaultValue, initialValue string) execFunc {
        return func() error {
            // 1. 先添加为 nullable（SQLite 限制）
            _, err := tx.ExecContext(ctx, fmt.Sprintf(
                `alter table %s add column %s %s %s;`, 
                tableName, columnName, columnType, tempDefault))
            
            // 2. 更新现有行的初始值
            _, err = tx.ExecContext(ctx, fmt.Sprintf(
                `update %s set %s = %s where %[2]s is null;`, 
                tableName, columnName, initialValue))
            
            // 3. 直接修改 sqlite_master 元数据，改为 NOT NULL
            _, err = tx.ExecContext(ctx, `
                PRAGMA writable_schema = on;
                UPDATE sqlite_master
                SET sql = replace(sql, '%[1]s %[2]s %[5]s', '%[1]s %[2]s default %[3]s not null')
                WHERE type = 'table' AND name = '%[4]s';
                PRAGMA writable_schema = off;
            `, columnName, columnType, defaultValue, tableName, tempDefault)
        }
    }
}
```

**技术要点**：
- 利用 SQLite 的 `writable_schema` 直接修改系统表
- 通过空格填充技巧避免字符串截断
- 支持 SQL 表达式作为初始值（如 `current_timestamp`）

### 1.4 数据迁移与重建的协作

重大 schema 变更时，迁移与重建机制协同工作：

**强制全量扫描标志**（`migration.go:23-35`）：

```go
func forceFullRescan(tx *sql.Tx) error {
    if conf.Server.DevOptimizeDB {
        _, err := tx.Exec(`ANALYZE;`)
    }
    _, err := tx.Exec(fmt.Sprintf(`
        INSERT OR REPLACE into property (id, value) values ('%s', '1');
    `, consts.FullScanAfterMigrationFlagKey))
    return err
}
```

有 23 个迁移调用了 `forceFullRescan`，主要场景包括：
- 搜索字段变更（如 FTS5 引入、搜索规范化算法变更）
- 新增元数据字段（如 BPM、声道数、ReplayGain）
- 数据格式转换（如歌词从纯文本转为 JSON）
- 表结构重构（如参与者信息 JSON 化）

**用户通知机制**（`migration.go:15-20`）：

```go
func notice(tx *sql.Tx, msg string) {
    if isDBInitialized(tx) {
        line := strings.Repeat("*", len(msg)+8)
        fmt.Printf("\n%s\nNOTICE: %s\n%s\n\n", line, msg, line)
    }
}
```

用于向用户通知重要变更（如破坏性变更、需要重建索引等）。

---

## 2. 配置校验体系

### 2.1 配置加载与验证流程

配置系统采用 **"默认值 → 加载 → 验证 → 修正"** 四阶段流水线：

**加载流程**（`conf/configuration.go:317-467`）：

```go
func Load(noConfigDump bool) {
    // 1. 特殊格式解析（INI）
    parseIniFileConfiguration()
    
    // 2. 环境变量键名修正
    remapEnvVarKeysFromConfig()
    
    // 3. 废弃选项映射（向后兼容）
    mapDeprecatedOption("ReverseProxyWhitelist", "ExtAuth.TrustedSources")
    
    // 4. 反序列化为结构体
    err := viper.Unmarshal(&Server, viper.DecodeHook(...))
    
    // 5. 安全性预检查（非 root 用户）
    if err := validateEnforceNonRootUser(); err != nil {
        logFatal(err)
    }
    
    // 6. 路径推导与默认值补全
    Server.CacheFolder = NewDir(filepath.Join(Server.DataFolder.String(), "cache"))
    Server.DbPath = filepath.Join(Server.DataFolder.String(), consts.DefaultDbPath)
    
    // 7. 批量验证
    err = run.Sequentially(
        validateScanSchedule,
        validateBackupSchedule,
        validatePlaylistsPath,
        validatePurgeMissingOption,
        validateMaxImageUploadSize,
        validateURL("ExtAuth.LogoutURL", Server.ExtAuth.LogoutURL),
    )
    
    // 8. 值域修正（clamping）
    if Server.UICoverArtSize < 200 || Server.UICoverArtSize > 1200 {
        newValue := max(200, min(1200, Server.UICoverArtSize))
        Server.UICoverArtSize = newValue
    }
    
    // 9. 触发初始化钩子
    for _, hook := range hooks {
        hook()
    }
}
```

### 2.2 默认值体系

`setViperDefaults()` 函数（`configuration.go:722-883`）定义了超过 **150 个** 默认值，覆盖：

| 类别 | 示例 |
|------|------|
| 网络配置 | `address: 0.0.0.0`, `port: 4533` |
| 路径配置 | `musicfolder: ./music`, `datafolder: .` |
| 功能开关 | `enabledownloads: true`, `enablesharing: false` |
| 性能调优 | `transcodingcachesize: 100MB`, `imagecachesize: 100MB` |
| 外部集成 | `lastfm.enabled: true`, `listenbrainz.enabled: true` |
| 扫描策略 | `scanner.scanonstartup: true`, `scanner.purgemissing: never` |
| 开发调试 | `devoptimizedb: true`, `devactivitypanel: true` |

### 2.3 验证函数设计

**调度表达式验证**：

```go
func validateSchedule(schedule, field string) (string, error) {
    _, err := scheduler.ParseCrontab(schedule)
    if err != nil {
        return schedule, fmt.Errorf("invalid %s %q: %w", field, schedule, err)
    }
    return schedule, nil
}
```

**URL 验证**（高阶函数，返回验证器）：

```go
func validateURL(optionName, optionURL string) func() error {
    return func() error {
        if optionURL == "" { return nil }
        u, err := url.Parse(optionURL)
        if err != nil { return fmt.Errorf("invalid %s: %w", optionName, err) }
        if u.Scheme != "http" && u.Scheme != "https" {
            return fmt.Errorf("invalid scheme for %s: '%s'", optionName, u.Scheme)
        }
        return nil
    }
}
```

**枚举值验证**：

```go
func validatePurgeMissingOption() error {
    allowedValues := []string{"never", "always", "full"}
    if !slices.Contains(allowedValues, Server.Scanner.PurgeMissing) {
        Server.Scanner.PurgeMissing = "never"  // 自动回退到安全默认值
        return fmt.Errorf("invalid value, must be one of: %v", allowedValues)
    }
    return nil
}
```

### 2.4 敏感字段保护

`server/nativeapi/config.go` 实现了配置 API 输出时的敏感信息脱敏：

```go
var sensitiveFieldsPartialMask = []string{
    "LastFM.ApiKey", "LastFM.Secret", 
    "Prometheus.MetricsPath", "DevAutoLoginUsername",
}

var sensitiveFieldsFullMask = []string{
    "DevAutoCreateAdminPassword", 
    "PasswordEncryptionKey", 
    "Prometheus.Password",
}

func redactValue(key string, value string) string {
    if len(value) < 7 { return "****" }
    return string(value[0]) + strings.Repeat("*", len(value)-2) + string(value[len(value)-1])
}
```

**脱敏策略**：
- **完全掩码**：密码类字段全部显示为 `****`
- **部分掩码**：API Key 等保留首尾字符，中间用 `*` 替代
- **递归处理**：支持嵌套结构中的敏感字段

### 2.5 插件配置验证

`plugins/config_validation.go` 实现了基于 **JSON Schema** 的插件配置验证：

```go
func ValidateConfig(manifest *Manifest, configJSON string) error {
    // 1. 检查插件是否定义了配置 schema
    if !manifest.HasConfigSchema() {
        return fmt.Errorf("plugin has no configurable options")
    }
    
    // 2. 解析配置 JSON
    var configData any
    if err := json.Unmarshal([]byte(configJSON), &configData); err != nil {
        return &ConfigValidationErrors{...}
    }
    
    // 3. 编译并验证 schema
    compiler := jsonschema.NewCompiler()
    compiler.AddResource("schema.json", manifest.Config.Schema)
    schema, _ := compiler.Compile("schema.json")
    
    // 4. 执行验证
    if err := schema.Validate(configData); err != nil {
        return convertValidationError(err)
    }
    return nil
}
```

**错误收集**：递归遍历 `jsonschema.ValidationError` 树，收集所有叶子错误，支持字段级错误路径显示。

---

## 3. 配置测试覆盖范围

### 3.1 核心验证逻辑测试

`conf/configuration_test.go` 提供了全面的测试覆盖：

| 测试场景 | 覆盖要点 |
|----------|----------|
| **语言解析** | 单语言、多语言逗号分隔、空白处理、空值回退 |
| **URL 验证** | HTTP/HTTPS 协议、无协议、不支持协议、空 URL、无主机 |
| **搜索后端归一化** | 大小写、空白、未知值回退、空字符串 |
| **键名转换** | 简单键、嵌套键、空字符串 |
| **环境变量键重映射** | 自动修正 ND_ 前缀、冲突检测、正常配置不影响 |
| **致命错误触发** | 无效配置文件、日志目录不可写、BaseURL 格式错误 |
| **图片大小验证** | 10MB/1GB/raw bytes/MiB/小写格式、无效字符串 |
| **非 root 用户强制** | root 用户阻止启动、非 root 放行、Windows 豁免 |
| **多格式配置加载** | TOML/YAML/INI/JSON 四种格式的正确解析 |

### 3.2 边界条件与错误处理

**测试设计亮点**：

```go
It("exits when enabled and running as root without having created a data folder", func() {
    // 创建不存在的路径
    nonExistentDataFolder := filepath.Join(tempBase, "nonexistent", "data")
    DeferCleanup(conf.SetRuntimeInfoForTest("linux", 0))  // 模拟 root 用户
    viper.Set("enforcenonrootuser", true)
    viper.Set("datafolder", nonExistentDataFolder)
    
    // 验证：加载失败（panic）
    Expect(func() { conf.Load(true) }).To(PanicWith(ContainSubstring("running as root")))
    
    // 验证：副作用未发生（目录未创建）
    Expect(nonExistentDataFolder).ToNot(BeAnExistingFile())
})
```

该测试验证了"验证失败时不产生副作用"这一重要安全属性。

### 3.3 配置快照与测试隔离

`conf/configuration.go:294-306` 提供的 `SnapshotConfig` 是测试基础设施的关键组件：

```go
func SnapshotConfig() func() {
    snapshot, _ := json.Marshal(Server)
    return func() {
        var restored configOptions
        json.Unmarshal(snapshot, &restored)
        Server = &restored
    }
}
```

**设计意图**：
- 通过 JSON 序列化/反序列化创建深拷贝
- 确保 `Dir` 等包含 `sync.Once` 的字段获得新鲜实例
- 测试结束后自动恢复配置，避免测试间污染

---

## 4. 持久层端到端验证路径

### 4.1 E2E 测试架构设计

`server/e2e/e2e_suite_test.go` 构建了完整的端到端验证体系：

**套件初始化流程**：

```
BeforeSuite
├─ 配置测试数据库路径（内存/文件）
├─ db.Init(ctx) → 执行所有迁移
├─ 创建测试用户（admin/regular）
├─ 创建测试 Library
├─ 构建测试文件系统（fake FS）
├─ 执行完整扫描（scanner.ScanAll）
├─ Checkpoint WAL 并创建 DB 快照
└─ 保存快照用于测试恢复
```

**单测试执行流程**：

```
BeforeEach
├─ configtest.SetupConfig() → 配置快照
├─ restoreDB() → 从快照恢复数据库
│   ├─ PRAGMA foreign_keys = OFF
│   ├─ ATTACH DATABASE 快照文件
│   ├─ 清空所有表
│   ├─ 从快照插入数据
│   └─ PRAGMA foreign_keys = ON
├─ 初始化真实 DataStore（persistence.New）
├─ 初始化 auth 模块
├─ 构建 Subsonic Router（集成所有依赖）
└─ 执行测试用例
```

### 4.2 模块协作关系图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         E2E 测试协调层                               │
│  server/e2e/e2e_suite_test.go                                       │
└─────────────┬───────────────────────────────────────────────────────┘
              │
    ┌─────────┼─────────────────────────────────────────┐
    │         │                                         │
    ▼         ▼                                         ▼
┌───────┐ ┌─────────┐  ┌──────────────────────┐  ┌──────────────┐
│  DB   │ │  Conf   │  │  Persistence Layer   │  │  Core Layer  │
│  Init │ │  Load   │  │  persistence.New()   │  │  scanner     │
└───┬───┘ └────┬────┘  └──────────┬───────────┘  │  artwork     │
    │           │                  │              │  playlists   │
    │           │                  │              │  auth        │
    │           ▼                  ▼              └──────┬───────┘
    │    ┌──────────────┐  ┌───────────────┐             │
    │    │  goose.Up()  │  │  SQLStore     │             │
    │    │  迁移执行    │  │  仓储实现      │             │
    │    └──────┬───────┘  └───────┬───────┘             │
    │           │                  │                     │
    └───────────┼──────────────────┼─────────────────────┘
                │                  │
                ▼                  ▼
        ┌──────────────────────────────────┐
        │          SQLite 数据库            │
        │  - schema（迁移创建）             │
        │  - 测试数据（扫描器生成）         │
        │  - 快照（测试隔离）               │
        └──────────────────────────────────┘
```

### 4.3 关键协作路径

**路径1：迁移验证 → 数据完整性**

```
db.Init(ctx)
├─ 执行所有迁移（按时间戳顺序）
├─ 验证 goose_db_version 版本号
└─ scanner.ScanAll(ctx, true)
   ├─ 解析测试文件系统中的元数据
   ├─ 通过 persistence 写入数据库
   └─ 验证：所有字段正确填充、无 NULL 违反约束
```

**路径2：API 请求 → 持久层操作**

```
doReq("getAlbumList2", "type", "recent")
└─ subsonic.Router.ServeHTTP
   ├─ 参数解析与认证
   ├─ core/albums 业务逻辑
   │  └─ persistence.AlbumRepository
   │     └─ dbx.Builder 生成 SQL
   │        └─ SQLite 执行查询
   └─ 结果序列化返回
```

**路径3：测试隔离机制**

```
Test A 执行
├─ restoreDB() → 从快照恢复
├─ 修改数据（如创建播放列表）
├─ 验证修改成功
└─ 测试结束 → 配置自动恢复
   
Test B 执行
├─ restoreDB() → 再次从快照恢复（Test A 的修改已消失）
└─ 在干净状态下执行测试
```

### 4.4 依赖注入与可测试性

E2E 测试中使用了精心设计的测试替身（Test Double）策略：

| 组件 | 实现 | 用途 |
|------|------|------|
| Artwork | `noopArtwork` | 避免真实图片处理 |
| Streamer | `spyStreamer` | 捕获请求参数，返回模拟音频 |
| FFmpeg | `noopFFmpeg` | 避免依赖外部二进制 |
| External Provider | `noopProvider` | 避免网络请求 |
| Archiver | `noopArchiver` | 避免文件压缩操作 |
| Event Broker | `events.NoopBroker` | 静默事件处理 |
| Metrics | `metrics.NewNoopInstance()` | 禁用指标收集 |

**设计原则**：
- 持久层使用 **真实实现** 验证数据库交互
- 跨进程/网络依赖使用 **空实现** 或 **间谍对象**
- 核心业务逻辑使用 **真实实现** 验证行为正确性

---

## 5. 总结与设计评价

### 5.1 迁移系统的设计智慧

1. **渐进式约束收紧**：从允许 NULL 到全面 NOT NULL，体现了对数据完整性认识的深化
2. **SQLite 适配层**：通过 `createAddColumnFunc` 等工具函数，抹平了 SQLite 的 DDL 限制
3. **迁移-重建闭环**：`forceFullRescan` 机制确保 schema 变更后数据一致性
4. **用户友好**：`notice` 函数在破坏性变更时提供清晰的用户通知

### 5.2 配置系统的安全设计

1. **纵深防御**：默认值 → 验证 → 修正 → 脱敏，四层防护
2. **容错而非失败**：对无效配置采用"记录警告 + 使用安全默认值"策略
3. **向后兼容**：通过 `mapDeprecatedOption` 平滑过渡配置键名变更
4. **可测试性**：`logFatal` 可替换、`SnapshotConfig` 测试隔离

### 5.3 E2E 验证的工程价值

1. **迁移正确性担保**：通过真实扫描 → 真实查询的闭环，验证迁移后数据可正常访问
2. **回归防护**：快照机制确保每个测试在已知状态下运行，快速发现回归
3. **集成点验证**：覆盖了从 HTTP 路由到 SQLite 的完整调用链
4. **性能保障**：数据库快照恢复比重新扫描快一个数量级，使 E2E 测试可频繁运行

### 5.4 可改进空间

1. **迁移回滚支持**：多数 `Down` 函数为空实现，降级能力有限
2. **迁移测试**：缺乏对迁移本身的单元测试（如数据转换逻辑）
3. **配置验证可扩展性**：当前验证函数为硬编码，插件式验证框架可能更灵活
4. **性能基准**：E2E 测试未包含迁移执行时间的基准监控

---

## 附录：关键文件速查表

| 模块 | 文件 | 核心功能 |
|------|------|----------|
| 迁移框架 | `db/db.go` | goose 集成、迁移执行流程 |
| 迁移工具 | `db/migrations/migration.go` | `forceFullRescan`、`notice`、`createAddColumnFunc` |
| 初始 schema | `db/migrations/20200130083147_create_schema.go` | 核心表结构定义 |
| NULL 修复 | `db/migrations/20240122223340_add_default_values_to_null_columns.go.go` | 46 个字段的 NOT NULL 修复 |
| 配置核心 | `conf/configuration.go` | 加载、验证、默认值、钩子机制 |
| 配置测试 | `conf/configuration_test.go` | 验证逻辑的全面测试 |
| 敏感字段 | `server/nativeapi/config.go` | API 输出时的脱敏处理 |
| 插件配置 | `plugins/config_validation.go` | JSON Schema 验证 |
| E2E 套件 | `server/e2e/e2e_suite_test.go` | 端到端测试基础设施 |
| 持久层 | `persistence/persistence.go` | SQLStore 仓储工厂 |
