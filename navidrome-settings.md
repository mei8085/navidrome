# Navidrome 配置热加载机制深度解析

本文档全面分析 Navidrome 项目中配置项从注册、保存、变更广播到前端消费的完整实现流程。

---

## 一、配置分层架构概述

Navidrome 的配置系统采用 **四层架构**，不同层级的配置具有不同的持久化策略和热加载能力：

| 层级 | 存储位置 | 支持热加载 | 典型配置项 |
|------|---------|-----------|-----------|
| **系统全局配置** | 配置文件 / 环境变量 (Viper) | ❌ 需重启 | 端口、MusicFolder、日志级别 |
| **系统属性** | `property` 数据库表 | ✅ | 上次扫描错误、密码加密标记 |
| **用户偏好设置** | `user_props` 数据库表 | ✅ | Last.fm SessionKey、主题、语言 |
| **运行时实体配置** | `plugin`/`library`/`user`/`transcoding` 等表 | ✅ | 插件配置、媒体库、用户信息 |

---

## 二、配置项的注册与默认值来源

### 2.1 核心配置结构体定义

所有全局配置项集中定义在 `configOptions` 结构体中：

[configuration.go:27-L153](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go#L27-L153)

```go
type configOptions struct {
    ConfigFile                      string
    Address                         string
    Port                            int
    MusicFolder                     string
    DataFolder                      Dir
    LogLevel                        string
    SessionTimeout                  time.Duration
    // ... 约 120+ 个配置字段，包括嵌套子结构体
    Search                          searchOptions
    Scanner                         scannerOptions
    LastFM                          lastfmOptions
    Plugins                         pluginsOptions
    // DevFlags 开发调试选项
    DevEnableProfiler               bool
    DevShowArtistPage               bool
}
```

全局单例访问入口为包级变量 `Server`：

[configuration.go:287-L290](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go#L287-L290)

```go
var (
    Server = &configOptions{}
    hooks  []func()
)
```

### 2.2 默认值注册机制：`setViperDefaults()`

使用 **Viper** 配置管理库，所有默认值在 `setViperDefaults()` 函数中统一注册，该函数通过 `init()` 在包加载时自动调用：

[configuration.go:722-L887](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go#L722-L887)

```go
func setViperDefaults() {
    viper.SetDefault("musicfolder", filepath.Join(".", "music"))
    viper.SetDefault("datafolder", ".")
    viper.SetDefault("loglevel", "info")
    viper.SetDefault("port", 4533)
    viper.SetDefault("sessiontimeout", consts.DefaultSessionTimeout)
    viper.SetDefault("enableartworkprecache", true)
    viper.SetDefault("lastfm.enabled", true)
    viper.SetDefault("plugins.enabled", true)
    // ... 共 160+ 条 SetDefault 调用
}

func init() {
    setViperDefaults()
}
```

**命名约定**：配置键使用小写+点号（如 `scanner.schedule`），对应结构体字段的 PascalCase（`Scanner.Schedule`）。

### 2.3 配置加载流程：`InitConfig()` → `Load()`

[configuration.go:889-L936](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go#L889-L936)

```go
func InitConfig(cfgFile string, loadEnvVars bool) {
    // 1. 注册 INI 编解码器
    codecRegistry := viper.NewCodecRegistry()
    _ = codecRegistry.RegisterCodec("ini", ini.Codec{...})

    // 2. 确定配置文件路径（命令行参数 → ND_CONFIGFILE 环境变量 → 当前目录 navidrome.*）
    cfgFile = getConfigFile(cfgFile)

    // 3. 环境变量绑定：ND_ 前缀 + 点号替换为下划线
    if loadEnvVars {
        viper.SetEnvPrefix("ND")
        replacer := strings.NewReplacer(".", "_")
        viper.SetEnvKeyReplacer(replacer)
        viper.AutomaticEnv()
    }

    // 4. 读取配置文件
    _ = viper.ReadInConfig()
}
```

配置优先级：**环境变量 > 配置文件 > 默认值**

加载时执行的额外处理：
- **废弃选项映射**：如 `ReverseProxyWhitelist` → `ExtAuth.TrustedSources`（详见 2.6 节）
- **INI 格式适配**：将 `[default]` 节合并到根级别
- **计算字段填充**：`BaseURL` 解析为 `BasePath/BaseHost/BaseScheme`
- **Hook 调用**：通过 `AddHook()` 注册的初始化回调

---

## 二点五、Load 阶段延迟校验与裁剪链路

`Load()` 函数在 `InitConfig()` 之后执行，负责将 Viper 中的原始配置反序列化为 `conf.Server` 结构体，并执行一系列校验和裁剪。整个流程分为 **5 个阶段**：

### 阶段 1：键名规范化（前置映射）

[configuration.go:318-L326](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go#L318-L326)

```go
func Load(noConfigDump bool) {
    parseIniFileConfiguration()           // INI 特殊适配
    remapEnvVarKeysFromConfig()           // 修正用户误写的 ND_ 前缀键名

    // Map deprecated options to their new names for backwards compatibility
    mapDeprecatedOption("ReverseProxyWhitelist", "ExtAuth.TrustedSources")
    mapDeprecatedOption("ReverseProxyUserHeader", "ExtAuth.UserHeader")
    mapDeprecatedOption("HTTPSecurityHeaders.CustomFrameOptionsValue", "HTTPHeaders.FrameOptions")
    mapDeprecatedOption("CoverJpegQuality", "CoverArtQuality")
    mapDeprecatedOption("SimilarSongsMatchThreshold", "Matcher.FuzzyThreshold")
}
```

### 阶段 2：反序列化（Unmarshal）

[configuration.go:328-L337](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go#L328-L337)

```go
err := viper.Unmarshal(&Server, viper.DecodeHook(
    mapstructure.ComposeDecodeHookFunc(
        mapstructure.TextUnmarshallerHookFunc(),
        mapstructure.StringToTimeDurationHookFunc(),
        mapstructure.StringToSliceHookFunc(","),
    ),
))
```

> **延迟校验策略**：先反序列化填充整个结构体，再进行校验。这是因为部分校验（如路径存在性、非 root 用户检查）需要结构体字段完整后才能进行。

### 阶段 3：早期校验（Pre-validation）

[configuration.go:339-L342](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go#L339-L342)

```go
// Validate non-root user early, before any filesystem operations
if err := validateEnforceNonRootUser(); err != nil {
    logFatal(err)
}
```

此阶段先校验 `EnforceNonRootUser` 权限，避免后续文件操作在错误的用户上下文中执行。

### 阶段 4：默认值补全与裁剪（Defaulting + Clipping）

[configuration.go:344-L461](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go#L344-L461)

```go
// ---- 4a：路径与目录默认值 ----
if Server.CacheFolder.String() == "" {
    Server.CacheFolder = NewDir(filepath.Join(Server.DataFolder.String(), "cache"))
}
if Server.Plugins.Enabled {
    if Server.Plugins.Folder.String() == "" {
        Server.Plugins.Folder = NewDirWithPerm(
            filepath.Join(Server.DataFolder.String(), "plugins"), 0700)
    }
}
// ... 更多默认值填充：DbPath、日志输出

// ---- 4b：BaseURL 拆解 ----
if Server.BaseURL != "" {
    u, _ := url.Parse(Server.BaseURL)
    Server.BasePath   = u.Path         // 仅路径部分（如 /music）
    u.Path = ""; u.RawQuery = ""
    Server.BaseHost   = u.Host         // 主机:端口（如 navidrome.local:4533）
    Server.BaseScheme = u.Scheme       // http 或 https
}

// ---- 4c：外服务级联禁用 ----
if !Server.EnableExternalServices {
    disableExternalServices()
}

// ---- 4d：PID（Persistent ID）非空兜底 ----
Server.PID.Album = cmp.Or(Server.PID.Album, consts.DefaultAlbumPID)
Server.PID.Track = cmp.Or(Server.PID.Track, consts.DefaultTrackPID)
// DefaultAlbumPID = "musicbrainz_albumid|albumartistid,album,albumversion,releasedate"
// DefaultTrackPID = "musicbrainz_trackid|albumid,discnumber,tracknumber,title"
```

#### 4c：`disableExternalServices()` 级联禁用链路

[configuration.go:563-L574](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go#L563-L574)

```go
func disableExternalServices() {
    Server.EnableInsightsCollector   = false  // 遥测关闭
    Server.EnableM3UExternalAlbumArt = false  // 外部专辑封面禁用
    Server.LastFM.Enabled            = false  // Last.fm 聚合接口
    Server.Deezer.Enabled            = false  // Deezer 元数据
    Server.ListenBrainz.Enabled      = false  // ListenBrainz 元数据
    Server.Agents                    = ""     // 所有外部 Agent 清空

    // 仅当用户未自定义登录背景时，替换为离线内置 PNG（base64 编码）
    if Server.UILoginBackgroundURL == consts.DefaultUILoginBackgroundURL {
        Server.UILoginBackgroundURL = consts.DefaultUILoginBackgroundURLOffline
    }
}
```

> **设计意图**：`EnableExternalServices` 是**最高优先级的总开关**，置为 `false` 时会强制覆盖 6 个子开关，即使它们在配置文件中单独设为 `true` 也无效。

#### 4d：PID 默认值来源

PID 默认值定义在 `consts/consts.go`，在两处均设为默认值（双重保险）：
- **Viper 默认值**：[configuration.go:842-843](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go#L842-L843) — `viper.SetDefault("pid.album", ...)`
- **Load 阶段兜底**：[configuration.go:433-434](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go#L433-L434) — `cmp.Or` 空值回退

### 阶段 5：批量校验、夹断与废弃告警

[configuration.go:384-L467](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go#L384-L467)

```go
// ---- 5a：批量硬校验（失败即退出） ----
err = run.Sequentially(
    validateScanSchedule,
    validateBackupSchedule,
    validatePlaylistsPath,
    validatePurgeMissingOption,
    validateMaxImageUploadSize,
    validateURL("ExtAuth.LogoutURL", Server.ExtAuth.LogoutURL),
)
if err != nil { logFatal(err) }

// ---- 5b：软校验 + 自动夹断（UICoverArtSize） ----
if Server.UICoverArtSize < 200 || Server.UICoverArtSize > 1200 {
    newValue := max(200, min(1200, Server.UICoverArtSize))
    log.Warn("UICoverArtSize must be between 200 and 1200, clamping",
        "value", Server.UICoverArtSize, "newValue", newValue)
    Server.UICoverArtSize = newValue
}

// ---- 5c：废弃选项告警 ----
logDeprecatedOptions("Scanner.GenreSeparators", "")            // 有替代方案
logDeprecatedOptions("Scanner.GroupAlbumReleases", "")         // 有替代方案
logDeprecatedOptions("SearchFullString", "Search.FullString")  // 已迁移
// ... 更多

logRemovedOptions("Spotify.ID", "Spotify.Secret")              // 已彻底删除

// ---- 5d：执行初始化 Hook ----
for _, hook := range hooks { hook() }
```

#### 5b：`UICoverArtSize` 夹断语义

| 约束 | 行为 |
|------|------|
| 默认值 | `consts.DefaultUICoverArtSize = 300` |
| 合法范围 | `[200, 1200]` |
| 越界处理 | 自动夹断到最近边界值 + `Warn` 日志 |
| **不**触发 `logFatal` | 夹断属于"容错性修正"，程序继续启动 |

---

## 二点六、废弃选项映射：Viper Alias 行为与语义差别

Navidrome 使用自定义的 `mapDeprecatedOption()` 实现废弃选项映射，**而非** Viper 原生的 `RegisterAlias`。

### `mapDeprecatedOption()` 的实际行为

[configuration.go:533-L539](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go#L533-L539)

```go
// mapDeprecatedOption is used to provide backwards compatibility for deprecated options.
// It should be called after the config has been read by viper, but before unmarshalling.
func mapDeprecatedOption(legacyName, newName string) {
    if viper.IsSet(legacyName) {
        viper.Set(newName, viper.Get(legacyName))
    }
}
```

**与 Viper `RegisterAlias` 的区别**：

| 特性 | `mapDeprecatedOption` | Viper `RegisterAlias` |
|------|----------------------|----------------------|
| **时机** | 每次 `Load()` 时显式调用 | 在 Viper 初始化时注册一次 |
| **键保留** | 旧键仍保留在 Viper 中 | 别名不会重复存储 |
| **优先级** | 新键值会被旧键覆盖（后调用的 `Set` 生效） | 别名与原键等同优先级 |
| **副作用** | 无警告日志（静默映射） | 无警告日志 |
| **覆盖检测** | 不检测新旧键是否同时设置 | 不适用 |

### 三种"废弃"语义的完整区别

Navidrome 实际存在 **三种不同的废弃处理**，语义完全不同：

#### 语义 A：继续可用但告警（`mapDeprecatedOption` + `logDeprecatedOptions`）

适用于"重命名但功能保留"的场景。**两层处理协同工作**：

| 步骤 | 函数 | 行为 | 执行时机 |
|------|------|------|---------|
| 1. 值迁移 | `mapDeprecatedOption()` | 若旧键被设置，`viper.Set(newName, oldValue)` 把旧值复制到新键 | **反序列化之前**（Load 阶段 1） |
| 2. 输出告警 | `logDeprecatedOptions()` | 检测旧键是否存在（`os.Getenv` + `viper.InConfig`），输出 Warn 日志 | **反序列化之后**（Load 阶段 5c） |

[configuration.go:469-L485](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go#L469-L485)

```go
func logDeprecatedOptions(oldName, newName string) {
    envVar := "ND_" + strings.ToUpper(strings.ReplaceAll(oldName, ".", "_"))
    logWarning := func(oldName, newName string) {
        if newName != "" {
            log.Warn(fmt.Sprintf(
                "Option '%s' is deprecated and will be ignored in a future release. Please use the new '%s'",
                oldName, newName))
        } else {
            log.Warn(fmt.Sprintf(
                "Option '%s' is deprecated and will be ignored in a future release",
                oldName))
        }
    }
    // 分别检查环境变量和配置文件
    if os.Getenv(envVar) != "" { logWarning(envVar, newEnvVar) }
    if viper.InConfig(oldName) { logWarning(oldName, newName) }
}
```

**同设时 legacy 压过 new 的语义**：

由于 `mapDeprecatedOption` 在反序列化前执行，且使用的是 `viper.Set()`（后写入覆盖先写入），最终生效规则为：

| 场景 | 最终生效值 | 说明 |
|------|-----------|------|
| ✅ legacy 设置，❌ new 未设置 | legacy 的值 | 正常迁移场景 |
| ❌ legacy 未设置，✅ new 设置 | new 的值 | 正常新写法 |
| ✅ legacy 设置，✅ new 也设置 | **legacy 的值（压过 new）** | `mapDeprecatedOption` 后执行，Set 覆盖了之前的新键值 |

> **关键差异**：如果用户同时设置了 `ReverseProxyWhitelist=old` 和 `ExtAuth.TrustedSources=new`，最终 **`ExtAuth.TrustedSources=old`**，旧值优先。Viper 原生 `RegisterAlias` 则不存在此问题（别名与原键完全等价，以优先级规则为准）。

**属于语义 A 的完整映射清单**（两层都处理）：
```go
// 阶段 1：mapDeprecatedOption（值迁移）
mapDeprecatedOption("ReverseProxyWhitelist", "ExtAuth.TrustedSources")
mapDeprecatedOption("ReverseProxyUserHeader", "ExtAuth.UserHeader")
mapDeprecatedOption("HTTPSecurityHeaders.CustomFrameOptionsValue", "HTTPHeaders.FrameOptions")
mapDeprecatedOption("CoverJpegQuality", "CoverArtQuality")
mapDeprecatedOption("SimilarSongsMatchThreshold", "Matcher.FuzzyThreshold")

// 阶段 5c：logDeprecatedOptions（告警输出）
logDeprecatedOptions("Scanner.GenreSeparators", "")
logDeprecatedOptions("Scanner.GroupAlbumReleases", "")
logDeprecatedOptions("DevEnableBufferedScrobble", "")
logDeprecatedOptions("SearchFullString", "Search.FullString")
logDeprecatedOptions("ReverseProxyWhitelist", "ExtAuth.TrustedSources")
logDeprecatedOptions("ReverseProxyUserHeader", "ExtAuth.UserHeader")
logDeprecatedOptions("HTTPSecurityHeaders.CustomFrameOptionsValue", "HTTPHeaders.FrameOptions")
logDeprecatedOptions("CoverJpegQuality", "CoverArtQuality")
logDeprecatedOptions("SimilarSongsMatchThreshold", "Matcher.FuzzyThreshold")
```

#### 语义 B：已移除，仅告警（`logRemovedOptions`）

适用于"功能已彻底删除"的场景。旧键名仍会被检测到，但值会被**忽略**，仅输出警告日志。

[configuration.go:487-L502](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go#L487-L502)

```go
func logRemovedOptions(options ...string) {
    for _, option := range options {
        envVar := "ND_" + strings.ToUpper(strings.ReplaceAll(option, ".", "_"))
        logWarning := func(option string) {
            log.Warn(fmt.Sprintf("Option '%s' is not available anymore and will be ignored. Please remove it from your config", option))
        }
        if viper.InConfig(option) { logWarning(option) }
        if os.Getenv(envVar) != "" { logWarning(envVar) }
    }
}

logRemovedOptions("Spotify.ID", "Spotify.Secret")
```

#### 语义 C：结构体字段标记（`// Deprecated:` 注释）

仅为**代码层面的文档标记**，不会触发任何运行时行为。对应字段的值仍然会被正常读取。

[configuration.go:162-L163](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go#L162-L163)

```go
type scannerOptions struct {
    GenreSeparators    string // Deprecated: Use Tags.genre.Split instead
    GroupAlbumReleases bool   // Deprecated: Use PID.Album instead
}
```

**三类废弃语义对比总结**：

| 语义 | 值迁移 | 运行时告警 | 功能仍可用 | 典型场景 |
|-----|-------|----------|----------|---------|
| **A：继续可用但告警** | ✅ `mapDeprecatedOption` | ✅ `logDeprecatedOptions` | ✅ | 重命名（`ReverseProxyWhitelist` → `ExtAuth.TrustedSources`） |
| **B：已移除** | ❌ | ✅ `logRemovedOptions` | ❌ | 功能下线（`Spotify.ID` / `Spotify.Secret`） |
| **C：仅代码注释** | ❌ | ❌ | ✅ | 字段层提示（`GenreSeparators` → `Tags.genre.Split`） |

**与 Viper `RegisterAlias` 的区别**：

---

## 三、前端配置注入机制

全局配置通过 `serve_index.go` 在渲染 `index.html` 时以 JSON 形式注入前端：

[serve_index.go:42-L83](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/serve_index.go#L42-L83)

```go
appConfig := map[string]any{
    "version":                   consts.Version,
    "baseURL":                   conf.Server.BasePath,
    "defaultTheme":              conf.Server.DefaultTheme,
    "enableDownloads":           conf.Server.EnableDownloads,
    "lastFMEnabled":             conf.Server.LastFM.Enabled,
    "listenBrainzEnabled":       conf.Server.ListenBrainz.Enabled,
    "enableReplayGain":          conf.Server.EnableReplayGain,
    "pluginsEnabled":            conf.Server.Plugins.Enabled,
    // ... 约 35 个 UI 相关配置项
}
```

注入方式：将 JSON 序列化为字符串，通过 Go Template 嵌入 `window.__APP_CONFIG__`：

[serve_index.go:103-L114](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/serve_index.go#L103-L114)

前端在 [config.js:50-L58](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/config.js#L50-L58) 解析：

```javascript
const appConfig = JSON.parse(window.__APP_CONFIG__)
config = { ...defaultConfig, ...appConfig }
```

> **关键限制**：此注入发生在页面加载时，**全局配置修改后必须刷新页面才能生效**，实际上等同于需要重启（因为后端 `conf.Server` 本身不支持热修改）。

---

## 四、配置保存接口与持久层写入

### 4.1 系统属性表：`property`

**表结构**（2020 年初始化 Schema）：

[20200130083147_create_schema.go:142-L147](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/db/migrations/20200130083147_create_schema.go#L142-L147)

```sql
create table if not exists property (
    id    varchar(255) not null primary key,
    value varchar(255) default '' not null
);
```

**接口定义**：

[properties.go:3-L8](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/model/properties.go#L3-L8)

```go
type PropertyRepository interface {
    Put(id string, value string) error
    Get(id string) (string, error)
    Delete(id string) error
    DefaultGet(id string, defaultValue string) (string, error)
}
```

**Upsert 实现**：先尝试 UPDATE，影响行数为 0 时再 INSERT：

[property_repository.go:24-L36](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/persistence/property_repository.go#L24-L36)

```go
func (r propertyRepository) Put(id string, value string) error {
    update := Update(r.tableName).Set("value", value).Where(Eq{"id": id})
    count, err := r.executeSQL(update)
    if count > 0 { return nil }
    insert := Insert(r.tableName).Columns("id", "value").Values(id, value)
    _, err = r.executeSQL(insert)
    return err
}
```

**典型使用场景**：
- 密码加密校验和标记（`consts.PasswordsEncryptedKey`）
- 扫描状态与错误信息（`consts.LastScanErrorKey`）

### 4.2 用户偏好表：`user_props`

**接口定义**：

[user_props.go:3-L8](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/model/user_props.go#L3-L8)

```go
type UserPropsRepository interface {
    Put(userId, key string, value string) error
    Get(userId, key string) (string, error)
    Delete(userId, key string) error
    DefaultGet(userId, key string, defaultValue string) (string, error)
}
```

**持久化实现**：复合主键 `(user_id, key)`：

[user_props_repository.go:24-L36](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/persistence/user_props_repository.go#L24-L36)

```go
func (r userPropsRepository) Put(userId, key string, value string) error {
    update := Update(r.tableName).Set("value", value).
        Where(And{Eq{"user_id": userId}, Eq{"key": key}})
    // 同 property 表的 upsert 逻辑
}
```

**典型使用者——SessionKeys 封装**：

[session_keys.go:9-L25](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/core/agents/session_keys.go#L9-L25)

```go
type SessionKeys struct {
    model.DataStore
    KeyName string  // 如 "LastFMSessionKey"、"ListenBrainzSessionKey"
}
```

**Last.fm 链接/取消链接 API**：

[auth_router.go:51-L91](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/adapters/lastfm/auth_router.go#L51-L91)

| HTTP 方法 | 路径 | 操作 |
|-----------|------|------|
| `GET` | `/api/lastfm/link` | 获取当前用户的链接状态 + ApiKey |
| `DELETE` | `/api/lastfm/link` | 删除 SessionKey，取消授权 |
| `GET` | `/api/lastfm/link/callback` | OAuth 回调，保存 SessionKey |

### 4.3 媒体库（Library）配置

**REST API 路由注册**：

[native_api.go:88-L94](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/nativeapi/native_api.go#L88-L94)

```go
r.With(adminOnlyMiddleware).Group(func(r chi.Router) {
    api.RX(r, "/library", api.libs.NewRepository, true)  // GET/POST/PUT/DELETE
})
```

**保存逻辑（含热加载动作）**：

[library.go:162-L192](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/core/library.go#L162-L192)

```go
func (r *libraryRepositoryWrapper) Save(entity any) (string, error) {
    // 1. 校验 + 持久化
    if err := r.validateLibrary(lib); err != nil { ... }
    err := r.LibraryRepository.Put(lib)

    // 2. 启动文件系统监听器（热加载）
    if r.watcher != nil {
        _ = r.watcher.Watch(r.ctx, lib)
    }

    // 3. 异步触发扫描
    if r.scanner != nil {
        go r.triggerScan(lib, "new")
    }

    // 4. 广播刷新事件
    event := &events.RefreshResource{}
    r.broker.SendBroadcastMessage(r.ctx, event.With("library", strconv.Itoa(lib.ID)))
}
```

**Update 操作的差异化处理：路径变更 vs 路径不变**

[library.go:194-L240](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/core/library.go#L194-L240)

```go
func (r *libraryRepositoryWrapper) Update(id string, entity any, _ ...string) error {
    // 1. 校验 + 持久化（所有更新都执行）
    if err := r.validateLibrary(lib); err != nil { return err }
    originalLib, _ := r.Get(libID)
    pathChanged := originalLib.Path != lib.Path
    err = r.LibraryRepository.Put(lib)

    // 2. 仅路径变更时执行的副作用
    if pathChanged {
        if r.watcher != nil {
            if err := r.watcher.Watch(r.ctx, lib); err != nil { ... }
        }
        if r.scanner != nil {
            go r.triggerScan(lib, "updated")  // 全量重扫描
        }
    }

    // 3. 所有更新都执行的副作用
    if r.broker != nil {
        event := &events.RefreshResource{}
        r.broker.SendBroadcastMessage(r.ctx, event.With("library", id))
    }
}
```

**路径变更 vs 路径不变的副作用对比表**：

| 操作 | 路径变更时 | 路径不变时（仅改名称等） |
|------|-----------|-----------------------|
| 校验 `validateLibrary` | ✅ 执行 | ✅ 执行 |
| DB `Put` 持久化 | ✅ 执行 | ✅ 执行 |
| **重启 Watcher** | ✅ `watcher.Watch()` 重新注册文件系统监听 | ❌ 不执行 |
| **触发全量扫描** | ✅ `triggerScan(lib, "updated")` 新协程异步执行 | ❌ 不执行 |
| **广播刷新事件** | ✅ `RefreshResource{library: id}` | ✅ `RefreshResource{library: id}` |

**Delete 操作的副作用**：

[library.go:242-L284](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/core/library.go#L242-L284)

- ✅ `watcher.StopWatching()` 停止监听
- ✅ `triggerScan(lib, "deleted")` 清理孤立数据
- ✅ `RefreshResource{library: id}` 广播刷新
- ✅ `pluginManager.UnloadDisabledPlugins()` 卸载因权限丢失而自动禁用的插件

### 4.4 用户-库关联配置

**专用 API**：

[library.go:15-L22](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/nativeapi/library.go#L15-L22)

```go
r.Route("/user/{id}/library", func(r chi.Router) {
    r.Get("/",  getUserLibraries(api.libs))   // 获取用户已分配库
    r.Put("/", setUserLibraries(api.libs))    // 批量分配库
})
```

**业务逻辑**：

[library.go:69-L105](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/core/library.go#L69-L105)

```go
func (s *libraryService) SetUserLibraries(ctx, userID string, libraryIDs []int) error {
    // 管理员不允许手动分配（自动拥有全部）
    if user.IsAdmin { return error }
    // 至少一个库
    if len(libraryIDs) == 0 { return error }
    // 写入 user_library 关联表
    _ = s.ds.User(ctx).SetUserLibraries(userID, libraryIDs)
    // 广播刷新事件
    s.broker.SendBroadcastMessage(ctx, event.With("user", userID).With("library", libIDs...))
}
```

### 4.5 插件配置（Plugin）

**路由端点**：

[plugin.go:17-L31](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/nativeapi/plugin.go#L17-L31)

```go
r.Route("/plugin", func(r chi.Router) {
    r.Get("/", rest.GetAll(constructor))
    r.Post("/rescan", api.rescanPlugins)
    r.Route("/{id}", func(r chi.Router) {
        r.Get("/", rest.Get(constructor))
        r.Put("/", api.updatePlugin)  // 统一更新入口
    })
})
```

**统一更新函数 `updatePluginSettings()`**：支持配置、用户、库权限的增量更新 + 自动热重载，包含严格的卸载-重载顺序与失败回退机制：

[manager.go:440-L510](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/plugins/manager.go#L440-L510)

```go
func (m *Manager) updatePluginSettings(ctx, id string, updateFn func(*model.Plugin)) error {
    // 1. 读取当前插件 → 应用更新函数
    plugin, _ := repo.Get(id)
    wasEnabled := plugin.Enabled  // 保存原状态用于回退判定
    updateFn(plugin)
    plugin.UpdatedAt = time.Now()

    // 2. 权限不足时自动禁用插件（先卸载，再写回DB）
    if manifest.Permissions.Users != nil && !hasValidUsersConfig(...) {
        if wasEnabled {
            _ = m.unloadPlugin(id)  // 先卸载
        }
        plugin.Enabled = false      // 标记为禁用
        if err := repo.Put(plugin); err != nil { ... }  // 再持久化
        m.sendPluginRefreshEvent(ctx, id)
        return nil
    }

    // 3. 已启用插件：先持久化新配置 → 卸载旧实例 → 用新配置重载
    if err := repo.Put(plugin); err != nil { ... }  // 步骤 1：持久化新配置

    if wasEnabled {
        // 步骤 2：卸载旧实例（即使失败也继续尝试加载）
        if err := m.unloadPlugin(id); err != nil {
            log.Debug(ctx, "Plugin was not loaded", "plugin", id)
        }
        // 步骤 3：用新配置重载
        if err := m.loadPluginWithConfig(plugin); err != nil {
            // ====== 加载失败回退机制 ======
            plugin.LastError = err.Error()  // 记录错误信息
            plugin.Enabled = false          // 标记为禁用
            _ = repo.Put(plugin)            // 写回 DB 持久化回退状态
            return fmt.Errorf("reloading plugin: %w", err)
        }
    }

    // 4. 成功：通知前端刷新
    m.sendPluginRefreshEvent(ctx, id)
    return nil
}
```

**关键顺序保证**：**先写 DB → 再卸载 → 再加载**。这样即使加载失败，DB 中也已保存新配置，不会出现"配置写回部分成功"的不一致状态。

**加载失败回退的完整链路**：

| 阶段 | 操作 | 失败时 |
|------|------|--------|
| 1. 持久化新配置 | `repo.Put(plugin)` | 直接返回错误，不执行后续操作 |
| 2. 卸载旧实例 | `unloadPlugin(id)` | 仅打 Debug 日志，继续加载 |
| 3. 加载新实例 | `loadPluginWithConfig(plugin)` | 设置 `LastError` + `Enabled=false`，**再次写回 DB** 保存回退状态，返回错误 |

### 4.5.1 插件目录文件监听：防抖 + SHA256 哈希自动发现

当 `Plugins.AutoReload=true` 时，启动文件系统监听器自动发现插件变更：

**启动与事件循环**：

[manager_watcher.go:21-L81](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/plugins/manager_watcher.go#L21-L81)

```go
func (m *Manager) startWatcher() error {
    m.watcherEvents = make(chan notify.EventInfo, 10)
    m.watcherDone = make(chan struct{})
    m.debounceTimers = make(map[string]*time.Timer)

    // 监听 CREATE/WRITE/REMOVE/RENAME 事件
    _ = notify.Watch(folder, m.watcherEvents,
        notify.Create, notify.Write, notify.Remove, notify.Rename)

    go m.watcherLoop()  // 单协程事件循环
}
```

**两级防抖机制**：

[manager_watcher.go:83-L109](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/plugins/manager_watcher.go#L83-L109)

```go
func (m *Manager) handleWatcherEvent(event notify.EventInfo) {
    // 只处理 .ndp 插件包文件
    if !strings.HasSuffix(path, PackageExtension) { return }

    // 防抖：取消该插件已有的定时器，启动新的 2s 定时器
    m.debounceMu.Lock()
    if timer, exists := m.debounceTimers[pluginName]; exists {
        timer.Stop()  // 取消上一个待处理事件
    }
    // 2 秒后触发实际处理（防抖窗口）
    m.debounceTimers[pluginName] = time.AfterFunc(debounceDuration, func() {
        m.processPluginEvent(pluginName)
    })
    m.debounceMu.Unlock()
}
```

> **设计意图**：编辑器保存文件时通常会触发多次事件（写入临时文件 → 重命名 → 删除原文件），2 秒防抖窗口确保只处理最终稳定状态。

**基于文件存在性的动作判定**：

[manager_watcher.go:120-L133](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/plugins/manager_watcher.go#L120-L133)

```go
func determinePluginAction(path string) pluginAction {
    if _, err := os.Stat(path); err == nil {
        return actionUpdate  // 文件存在 → 添加或更新
    }
    return actionRemove      // 文件不存在 → 删除
}
```

> **不依赖事件类型的原因**：macOS FSEvents 会合并事件、构建工具的原子写入（写临时文件→重命名）会产生 REMOVE+CREATE 序列，基于最终文件存在性判定更可靠。

**SHA256 哈希去重**：

[manager_watcher.go:158-L203](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/plugins/manager_watcher.go#L158-L203)

```go
func (m *Manager) processPluginEvent(pluginName string) {
    sha256Hash, err := computeFileSHA256(ndpPath)

    // 与 DB 中存储的哈希对比，相同则跳过
    if dbPlugin.SHA256 == sha256Hash {
        return  // 无实际变更
    }

    // 哈希不同 → 提取完整 manifest 并更新 DB
    metadata, err := m.extractManifest(ndpPath)
    if err != nil {
        // 提取失败 → 卸载并禁用
        if dbPlugin.Enabled {
            _ = m.unloadPlugin(pluginName)
            dbPlugin.Enabled = false
        }
        dbPlugin.LastError = err.Error()
        _ = repo.Put(dbPlugin)
        return
    }
    _ = m.updatePluginInDB(ctx, repo, dbPlugin, ndpPath, metadata)
}
```

**SHA256 流式计算实现**：

[manager_sync.go:39-L53](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/plugins/manager_sync.go#L39-L53)

```go
func computeFileSHA256(path string) (string, error) {
    f, err := os.Open(path)
    defer f.Close()
    h := sha256.New()
    if _, err := io.Copy(h, f); err != nil {  // 流式计算，不加载整个文件到内存
        return "", err
    }
    return hex.EncodeToString(h.Sum(nil)), nil
}
```

**完整的监听-处理链路**：

```
文件事件 → handleWatcherEvent()
    → 过滤 .ndp 文件
    → 2s 防抖（取消旧定时器，启动新定时器）
    → 2s 后 processPluginEvent()
        → SHA256 哈希计算
        → 与 DB 对比，相同则跳过
        → 不同则提取 manifest
            → 成功：updatePluginInDB() → 卸载+禁用+写DB
            → 失败：unloadPlugin() + 设置 LastError + Enabled=false
        → sendPluginRefreshEvent()
```

### 4.5.2 `unloadPlugin` 的完整清理流程

[manager.go:512-L544](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/plugins/manager.go#L512-L544)

```go
func (m *Manager) unloadPlugin(name string) error {
    // 1. 加锁从内存映射中移除
    m.mu.Lock()
    plugin, ok := m.plugins[name]
    if !ok { m.mu.Unlock(); return error }
    delete(m.plugins, name)
    m.mu.Unlock()

    // 2. 调用插件 Cleanup 钩子（在锁外执行，避免长时间阻塞）
    err := plugin.Close()

    // 3. 关闭 WASM 运行时实例（5s 超时，允许在途请求完成）
    if plugin.compiled != nil {
        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()
        _ = plugin.compiled.Close(ctx)
    }

    runtime.GC()  // 主动 GC 释放 WASM 内存
    return nil
}
```

### 4.5.3 用户/媒体库删除时的插件级联卸载

当用户或媒体库被删除时，会触发一条完整的 **DB 清理 → 自动禁用 → 内存卸载** 级联链路。

#### 用户删除级联链路

[user.go:62-L76](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/core/user.go#L62-L76)

```go
func (r *userRepositoryWrapper) Delete(id string) error {
    // 步骤 1：底层 DB 删除（含 plugin 用户引用清理）
    err := r.UserRepository.(rest.Persistable).Delete(id)
    if err != nil { return err }

    // 步骤 2：从内存卸载所有因权限丢失而被自动禁用的插件
    r.pluginManager.UnloadDisabledPlugins(r.ctx)
    return nil
}
```

**DB 层自动禁用逻辑**（SQLite JSON 函数处理）：

[plugin_cleanup.go:7-L47](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/persistence/plugin_cleanup.go#L7-L47)

```go
func cleanupPluginUserReferences(db dbx.Builder, userID string) error {
    // 步骤 A：从所有 plugin.users JSON 数组中移除该用户
    _, err := db.NewQuery(`
        UPDATE plugin
        SET users = (
            SELECT json_group_array(value)
            FROM json_each(plugin.users)
            WHERE value != {:userID}
        ), updated_at = CURRENT_TIMESTAMP
        WHERE EXISTS (SELECT 1 FROM json_each(plugin.users) WHERE value = {:userID})
    `).Bind(...).Execute()

    // 步骤 B：自动禁用"只剩空用户列表"的插件
    // 条件：enabled=true AND all_users=false AND manifest.permissions.users 存在 AND users 数组为空
    _, err = db.NewQuery(`
        UPDATE plugin
        SET enabled = false, updated_at = CURRENT_TIMESTAMP
        WHERE enabled = true
          AND all_users = false
          AND json_extract(manifest, '$.permissions.users') IS NOT NULL
          AND (users IS NULL OR users = '' OR users = '[]' OR json_array_length(users) = 0)
    `).Execute()
    return err
}
```

**内存卸载逻辑 `UnloadDisabledPlugins()`**：

[manager.go:546-L590](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/plugins/manager.go#L546-L590)

```go
func (m *Manager) UnloadDisabledPlugins(ctx context.Context) {
    // 1. 从 DB 查询所有 enabled=false 的插件
    plugins, err := repo.GetAll(model.QueryOptions{
        Filters: squirrel.Eq{"enabled": false},
    })

    // 2. 逐个检查：仍在内存中的 → 调用 unloadPlugin()
    var unloaded []string
    for _, p := range plugins {
        m.mu.RLock()
        _, loaded := m.plugins[p.ID]
        m.mu.RUnlock()

        if loaded {
            if err := m.unloadPlugin(p.ID); err == nil {
                unloaded = append(unloaded, p.ID)
            }
        }
    }

    // 3. 有卸载 → 广播刷新事件
    if len(unloaded) > 0 {
        m.sendPluginRefreshEvent(ctx, unloaded...)
    }
}
```

**完整级联时序图（用户删除）**：

```
DELETE /api/user/{id}
  ↓
userRepositoryWrapper.Delete(id)
  ↓
1. persistence.UserRepository.Delete(id)
   ├─ r.delete(Eq{"id": id})               // 删除 user 行
   └─ cleanupPluginUserReferences(db, id)   // 调用 plugin_cleanup.go
      ├─ UPDATE plugin SET users = json_group_array(...) WHERE value != userID
      └─ UPDATE plugin SET enabled = false WHERE users 数组已空
  ↓
2. pluginManager.UnloadDisabledPlugins(ctx)
   ├─ repo.GetAll(Filters: enabled=false)   // 查询所有禁用的插件
   ├─ 对每个仍加载在 m.plugins 中的：unloadPlugin(id)
   │   ├─ 从 m.plugins map 删除
   │   ├─ plugin.Close()                     // 用户 Cleanup 钩子
   │   ├─ compiled.Close(5s timeout)         // 关闭 WASM 运行时
   │   └─ runtime.GC()
   └─ sendPluginRefreshEvent(ctx, ...)       // 广播 RefreshResource{plugin}
```

**媒体库删除的级联链路**与用户删除完全对称（[library.go:279-L281](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/core/library.go#L279-L281) + [plugin_cleanup.go:49-L86](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/persistence/plugin_cleanup.go#L49-L86)），区别在于操作的是 `plugin.libraries` JSON 数组而非 `plugin.users`。

> **关键差异**：`cleanupPluginUserReferences` / `cleanupPluginLibraryReferences` 只处理 DB 层的 `enabled=false`，不操作内存。内存卸载统一由 `UnloadDisabledPlugins()` 完成——该函数是幂等的（先读 DB 再比对内存），因此用户删除、媒体库删除、以及插件配置保存三种触发场景可以复用同一逻辑。

**具体配置保存方法**：

```go
func (m *Manager) UpdatePluginConfig(ctx, id, configJSON string) error {
    return m.updatePluginSettings(ctx, id, func(p *model.Plugin) {
        p.Config = configJSON  // 保存 JSON 序列化后的配置
    })
}

func (m *Manager) UpdatePluginUsers(ctx, id, usersJSON string, allUsers bool) error { ... }
func (m *Manager) UpdatePluginLibraries(ctx, id, librariesJSON string, ...) error { ... }
```

### 4.5.4 插件重载到底重建什么：完整链路详解

插件重载（`loadPluginWithConfig`）并非简单地"重启一个实例"，而是涉及 **10 个步骤**的完整重建链路，从配置解析到 WASM 编译再到能力检测。以下是逐阶段对照代码的详解：

[manager_loader.go:70-L180](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/plugins/manager_loader.go#L70-L180)

---

#### 阶段 1：配置 JSON 解析（parsePluginConfig）

插件配置在 DB 中存为 JSON 字符串，但 Extism 的 Manifest.Config 要求 **所有值必须是字符串**。因此需要做一层类型适配：

```go
func parsePluginConfig(configJSON string) (map[string]string, error) {
    var raw map[string]interface{}
    json.Unmarshal([]byte(configJSON), &raw)
    
    result := make(map[string]string)
    for k, v := range raw {
        switch val := v.(type) {
        case string:
            result[k] = val  // 字符串直接保留
        default:
            // 非字符串值重新序列化为 JSON 字符串
            jsonBytes, _ := json.Marshal(val)
            result[k] = string(jsonBytes)
        }
    }
    return result, nil
}
```

> **设计意图**：Extism 的 WASM 侧只能读取字符串配置，因此布尔、数字、数组、对象等类型都需要序列化成 JSON 字符串，由插件自己反序列化使用。

---

#### 阶段 2：权限 JSON 数组解析

分别解析用户和媒体库的权限配置：

```go
// 用户权限
var allowedUsers []string
if p.AllUsers {
    allUsers = true  // 允许访问所有用户
} else {
    json.Unmarshal([]byte(p.Users), &allowedUsers)  // 指定用户列表
}

// 媒体库权限
var allowedLibraries []int
if p.AllLibraries {
    allLibraries = true  // 允许访问所有库
} else {
    json.Unmarshal([]byte(p.Libraries), &allowedLibraries)  // 指定库列表
}
```

解析结果存入 `serviceContext`，供后续 host services 构造时使用。

---

#### 阶段 3：打开插件包获取 Wasm 字节码 + Manifest

```go
pkg, err := OpenPluginPackage(p.Path)  // .ndp 格式（zip 压缩包）
defer pkg.Close()

wasmBytes, err := pkg.WasmBytes()  // 读取 .wasm 文件
manifest, err := pkg.Manifest()    // 读取 manifest.json
```

---

#### 阶段 4：构造 extism.Manifest

这是插件运行时环境的核心配置，包含 5 类信息：

```go
extismManifest := extism.Manifest{
    Wasm: []extism.Wasm{
        extism.WasmData{
            Data: wasmBytes,  // WASM 字节码
            Hash: "",          // 可选：内容哈希
        },
    },
    Config:         configMap,    // 阶段 1 解析的配置 map
    Timeout:         uint32(consts.PluginTimeout.Seconds()),
    AllowedHosts:    manifest.Http.RequiredHosts,  // 来自插件 manifest 声明
    AllowedPaths:    buildAllowedPaths(...),        // 库文件系统挂载点
}
```

**关键字段说明**：

| 字段 | 来源 | 作用 |
|------|------|------|
| `Wasm` | 插件包内 .wasm 文件 | 实际执行的 WebAssembly 代码 |
| `Config` | DB 中 plugin.config JSON | 插件可读取的配置键值对 |
| `Timeout` | 常量 `PluginTimeout`（30s） | 单次调用最大执行时间，防止死循环 |
| `AllowedHosts` | 插件 manifest.Http.RequiredHosts | 允许 HTTP 请求访问的主机白名单 |
| `AllowedPaths` | `buildAllowedPaths()` 计算 | 允许 WASM 访问的宿主文件系统路径 |

---

#### 阶段 5：库权限 → 允许文件路径计算（buildAllowedPaths）

`buildAllowedPaths` 根据插件的媒体库权限，计算出 WASM 可以访问的宿主文件系统路径映射表：

[manager_loader.go:208-L260](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/plugins/manager_loader.go#L208-L260)

```go
func buildAllowedPaths(perm *LibraryPermission, allowedLibraries []int, allLibraries bool) (map[string]string, error) {
    allowedPaths := make(map[string]string)
    libraries, _ := ds.Library(ctx).GetAll()
    
    for _, lib := range libraries {
        // 权限过滤：allLibraries=true 或 在允许列表中
        if !allLibraries && !slices.Contains(allowedLibraries, lib.ID) {
            continue
        }
        
        hostPath := filepath.Clean(lib.Path)
        mountPoint := fmt.Sprintf("/library/%d", lib.ID)  // WASM 内挂载路径
        
        // 无写权限 → 加 ro: 前缀（只读）
        if perm != nil && !perm.Write {
            mountPoint = "ro:" + mountPoint
        }
        
        allowedPaths[hostPath] = mountPoint
    }
    return allowedPaths, nil
}
```

**挂载规则**：
- **宿主路径**：媒体库的实际文件路径（如 `D:\Music`）
- **WASM 内挂载点**：`/library/{libraryID}`（如 `/library/1`）
- **只读标记**：无写权限时加 `ro:` 前缀，WASM 只能读不能写

> **安全设计**：通过路径映射 + 只读前缀，确保插件只能访问授权的媒体库目录，无法越权访问宿主其他文件。

---

#### 阶段 6：按 manifest 权限重建 host functions

这是插件能力边界的核心——**根据插件 manifest 中声明的权限，动态注册对应的 host 函数**。权限不足时，对应的 host 函数不会被注册，插件尝试调用会直接失败。

采用**表驱动**注册模式，`hostServices` 全局表包含 12 个服务条目：

[manager_loader.go:40-L150](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/plugins/manager_loader.go#L40-L150)

```go
var hostServices = []hostServiceEntry{
    {
        name:          "Config",
        hasPermission: func(p *Permissions) bool { return true },  // 始终可用
        create: func(ctx *serviceContext) ([]extism.HostFunction, io.Closer) {
            service := newConfigService(ctx.pluginName, ctx.config)
            return host.RegisterConfigHostFunctions(service), nil
        },
    },
    {
        name:          "SubsonicAPI",
        hasPermission: func(p *Permissions) bool { return p != nil && p.Subsonicapi != nil },
        create: func(ctx *serviceContext) ([]extism.HostFunction, io.Closer) {
            service := newSubsonicAPIService(..., ctx.allowedUsers, ctx.allUsers)
            return host.RegisterSubsonicAPIHostFunctions(service), nil
        },
    },
    {
        name:          "Library",
        hasPermission: func(p *Permissions) bool { return p != nil && p.Library != nil },
        create: func(ctx *serviceContext) ([]extism.HostFunction, io.Closer) {
            service := newLibraryService(..., ctx.allowedLibraries, ctx.allLibraries)
            return host.RegisterLibraryHostFunctions(service), nil
        },
    },
    // ... 还有 Scheduler / WebSocket / Artwork / Cache / KVStore / Users / HTTP / Task 等
}
```

**重建流程**：

```go
var hostFunctions []extism.HostFunction
var closers []io.Closer

for _, svc := range hostServices {
    // 权限判定：manifest 声明了该权限才注册
    if !svc.hasPermission(manifest.Permissions) {
        continue
    }
    
    // 创建服务实例 + 注册对应的 host functions
    fns, closer := svc.create(&serviceContext{
        pluginName:       id,
        manager:          m,
        permissions:      manifest.Permissions,
        config:           configMap,
        allowedUsers:     allowedUsers,
        allUsers:         allUsers,
        allowedLibraries: allowedLibraries,
        allLibraries:     allLibraries,
    })
    
    hostFunctions = append(hostFunctions, fns...)
    if closer != nil {
        closers = append(closers, closer)  // 卸载时需要清理
    }
}
```

**12 个 Host 服务一览**：

| 服务名 | 权限检查 | 作用 |
|--------|----------|------|
| Config | 始终可用 | 读取插件配置 |
| SubsonicAPI | `permissions.subsonicapi` | 调用 Subsonic API |
| Scheduler | `permissions.scheduler` | 注册定时任务 |
| WebSocket | `permissions.websocket` | 发送 WebSocket 消息 |
| Artwork | `permissions.artwork` | 获取专辑封面 |
| Cache | `permissions.cache` | 使用插件级缓存 |
| Library | `permissions.library` | 查询媒体库数据 |
| KVStore | `permissions.kvstore` | 键值存储持久化 |
| Users | `permissions.users` | 查询用户信息 |
| HTTP | `permissions.http` | 发起 HTTP 请求 |
| Task | `permissions.task` | 后台任务管理 |
| （持续扩展） | | |

> **最小权限原则**：每个插件只能获得 manifest 中声明的权限对应的 host functions。未声明的权限对应的 host function 根本不会被注册到 WASM 运行时中，插件无法调用。

---

#### 阶段 7：wazero RuntimeConfig 构造

配置底层 WASM 运行时（wazero）的编译缓存和运行行为：

```go
runtimeConfig := wazero.NewRuntimeConfig().
    WithCloseOnContextDone(true).  // 上下文取消时自动关闭
    WithCompilationCache(cache)    // 编译缓存（复用已编译模块）

// 实验性线程支持（需要 manifest 声明）
if manifest.Threads {
    runtimeConfig = runtimeConfig.WithCoreFeatures(api.CoreFeaturesV2 | experimental.CoreFeaturesThreads)
}
```

---

#### 阶段 8：编译（NewCompiledPlugin）

这是最重的一步：将 WASM 字节码编译为可执行的机器码，同时注册所有 host functions。

```go
compiledPlugin, err := extism.NewCompiledPlugin(ctx, extismManifest, runtimeConfig, hostFunctions)
```

> **性能注意**：编译是耗时操作，因此配置变更时通过 `unloadPlugin` → `loadPluginWithConfig` 完整重建，而不是尝试"热更新"单个函数。

---

#### 阶段 9：能力检测（detectCapabilities）

创建一个临时实例，扫描插件导出的所有函数，识别其支持的能力：

```go
tempInstance, _ := compiledPlugin.Instance()
defer tempInstance.Close()

capabilities, err := detectCapabilities(tempInstance)  // 扫描导出函数
manifest.ValidateWithCapabilities(capabilities)        // 校验 manifest 声明与实际能力一致
```

---

#### 阶段 10：注册 + 初始化

编译后的插件存入内存 map，并调用插件的初始化函数：

```go
p := &plugin{
    name:           id,
    path:           pluginPath,
    manifest:       manifest,
    compiled:       compiledPlugin,
    capabilities:   capabilities,
    closers:        closers,           // host services 清理句柄
    allowedUserIDs: allowedUsers,
    allUsers:       allUsers,
    libraries:      libraryAccess{...},  // O(1) 库权限查找
}

m.plugins[id] = p
m.callPluginInit(p)  // 调用插件导出的 _start / _initialize 函数
```

---

**plugin 结构体全貌**：

[manager_plugin.go:15-L50](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/plugins/manager_plugin.go#L15-L50)

```go
type plugin struct {
    name           string
    path           string
    manifest       *Manifest          // 插件声明的元数据
    compiled       *extism.CompiledPlugin  // 编译后的 WASM 模块（可复用创建实例）
    capabilities   []Capability       // 检测到的能力
    closers        []io.Closer        // host services 清理句柄
    metrics        PluginMetricsRecorder
    allowedUserIDs []string
    allUsers       bool
    libraries      libraryAccess      // O(1) 库权限查找
}
```

> **关键理解**：`CompiledPlugin` 是编译后的模块，可以快速创建多个实例（轻量级）。因此"重载"的代价主要在编译阶段，实例创建是廉价的。

---

**重载重建清单总结**：

每次 `loadPluginWithConfig` 调用都会完整重建以下内容：

| 重建项 | 说明 | 代价 |
|--------|------|------|
| 配置 map | JSON → `map[string]string` 转换 | 低 |
| 权限列表 | users/libraries JSON 数组解析 | 低 |
| Manifest | Extism 运行时配置（Wasm/Config/Timeout/AllowedHosts/AllowedPaths） | 低 |
| AllowedPaths | 库权限 → 文件路径映射计算 | 低 |
| Host Functions | 按权限遍历 12 个服务，生成 host function 列表 | 中 |
| RuntimeConfig | wazero 运行时配置 + 编译缓存 | 低 |
| CompiledPlugin | WASM 字节码编译为机器码 | **高**（CPU 密集） |
| Capabilities | 导出函数扫描 + manifest 校验 | 中 |
| Closers | 各 host service 的清理句柄 | 低 |
| plugin 实体 | 内存中的插件结构体（存入 m.plugins map） | 低 |

### 4.6 Dev/调试只读配置接口

仅在 `DevUIShowConfig=true` 时启用的 GET 接口（**只读**，无 PUT/POST）：

[native_api.go:234-L238](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/nativeapi/native_api.go#L234-L238)

```go
func (api *Router) addConfigRoute(r chi.Router) {
    if conf.Server.DevUIShowConfig {
        r.Get("/config/*", getConfig)  // 序列化 conf.Server 并脱敏返回
    }
}
```

敏感字段脱敏处理（[config.go:20-L94](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/nativeapi/config.go#L20-L94)）：
- **完全掩码**：`DevAutoCreateAdminPassword`、`PasswordEncryptionKey`、`Prometheus.Password`
- **部分掩码**：`LastFM.ApiKey`、`LastFM.Secret`（首尾字符可见）

---

## 五、配置变更事件广播机制（SSE Broker）

### 5.1 核心数据结构

基于 **Server-Sent Events (SSE)** 协议实现服务端→客户端的实时推送。

**事件接口定义**：

[events.go:21-L36](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/events.go#L21-L36)

```go
type Event interface {
    Name(Event) string   // 事件名：首字母小写的类型名（如 "refreshResource"）
    Data(Event) string   // JSON 序列化后的事件数据
}

type baseEvent struct{}  // 提供默认反射实现
```

**主要事件类型**：

| 事件类型 | 结构 | 用途 |
|---------|------|------|
| `refreshResource` | `RefreshResource` | 资源变更通知（配置/数据修改） |
| `scanStatus` | `ScanStatus` | 扫描进度 |
| `serverStart` | `ServerStart` | 服务启动/重连 |
| `keepAlive` | `KeepAlive` | 每 15s 心跳 |
| `nowPlayingCount` | `NowPlayingCount` | 当前播放人数 |

### 5.2 Broker 单例、单协程调度与反压机制

[sse.go:56-L80](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/sse.go#L56-L80)

```go
type broker struct {
    publish       messageChan     // 待发布事件（buffered=2）
    subscribing   clientsChan     // 新客户端连接（buffered=1）
    unsubscribing clientsChan     // 客户端断开（buffered=1）
}

func GetBroker() Broker {
    return singleton.GetInstance(func() *broker {
        broker := &broker{
            publish:       make(messageChan, 2),
            subscribing:   make(clientsChan, 1),
            unsubscribing: make(clientsChan, 1),
        }
        go broker.listen()  // 启动**唯一的**事件循环协程
        return broker
    })
}
```

**单协程调度设计**：
- 整个 Broker 只有 **一个 `listen()` 协程** 处理所有事件
- 所有对 `clients` map 的读写都在这个协程内完成，**无需加锁**
- 三个 channel（`publish/subscribing/unsubscribing`）作为唯一的并发访问入口
- `select` 语句保证同一时间只处理一个事件，天然串行化

### 5.2.1 反压与通道缓冲设计

**五层缓冲架构**（含发布/订阅/注销三类控制通道 + 两类数据通道）：

| 层级 | 通道 | 缓冲大小 | 用途 | 阻塞行为 |
|------|------|---------|------|---------|
| 1a. 事件发布 | `broker.publish` | **2** | 事件生产者 → Broker 协程 | 满则**阻塞** `SendMessage()` 调用方 |
| 1b. 新客户端注册 | `broker.subscribing` | **1** | HTTP 处理器 → Broker 协程 | 满则**阻塞** SSE 握手协程 |
| 1c. 客户端注销 | `broker.unsubscribing` | **1** | HTTP 处理器 → Broker 协程 | 满则**阻塞** defer 中的注销 |
| 2. 客户端数据 | `client.msgC` | **1** | Broker 协程 → 单个客户端连接 | 满则**丢弃**事件（`sendOrDrop`） |
| 3. 读取端输出 | `pl.ReadOrDone` 输出 | 0（无缓冲） | 客户端协程 → HTTP 写入 | 上下文取消即停止 |

**反压策略 1：发布端阻塞**

[sse.go:82-L99](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/sse.go#L82-L99)

```go
func (b *broker) SendMessage(ctx context.Context, evt Event) {
    b.publish <- b.prepareMessage(ctx, evt)  // 无 select，满则阻塞！
}
```

> **设计意图**：`publish` 通道缓冲仅为 2，当事件产生速度远超处理速度时，调用方会被阻塞，形成自然的反压。这适用于配置变更等低频但重要的事件，确保事件不丢失。

**反压策略 1.5：订阅/退订阻塞调用方**

`subscribing` 和 `unsubscribing` 通道的缓冲均为 1，同样采用**阻塞写入**（无 select+default）：

[sse.go:168-L194](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/sse.go#L168-L194)

```go
func (b *broker) subscribe(r *http.Request) client {
    // ... 构造 client 对象 ...
    b.subscribing <- c       // 缓冲=1，满则阻塞 HTTP 处理器协程
    return c
}

func (b *broker) unsubscribe(c client) {
    b.unsubscribing <- c     // 缓冲=1，满则阻塞 HTTP 处理器协程
}
```

**阻塞对调用方的实际影响**：

| 通道 | 调用方 | 阻塞时行为 |
|------|--------|----------|
| `publish` | 保存/删除配置的业务协程（扫描控制器、用户删除、库更新、插件更新等） | 业务请求被挂起，直到 Broker 消费该事件 |
| `subscribing` | SSE HTTP 处理器协程（`ServeHTTP`） | 新客户端连接等待 Broker 确认注册，超时后由反向代理断连 |
| `unsubscribing` | 同上（HTTP 请求结束触发 defer） | 连接关闭的清理被延迟，但实际 HTTP 响应已发送给客户端 |

**反压策略 2：客户端丢弃**

[sse.go:272-L280](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/sse.go#L272-L280)

```go
func sendOrDrop(client client, msg message) {
    select {
    case client.msgC <- msg:  // 尝试写入
    default:                  // 通道满则直接丢弃
        if log.IsGreaterOrEqualTo(log.LevelTrace) {
            log.Trace("Event dropped because client's channel is full", ...)
        }
    }
}
```

> **设计意图**：客户端侧缓冲仅为 1，当某个客户端网络慢或处理不及时，**直接丢弃事件**而非阻塞整个 Broker。SSE 协议本身只保证最终一致，事件丢失后客户端可通过刷新恢复。

**反压策略 3：优雅关闭**

[pipelines.go:125-L139](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/utils/pl/pipelines.go#L125-L139)

```go
func ReadOrDone[T any](ctx context.Context, in <-chan T) <-chan T {
    valStream := make(chan T)
    go func() {
        defer close(valStream)
        for {
            select {
            case <-ctx.Done():     // HTTP 请求取消时立即退出
                return
            case v, ok := <-in:    // 从客户端 channel 读取
                if !ok { return }  // channel 关闭则退出
                valStream <- v     // 转发到输出 channel
            }
        }
    }()
    return valStream
}
```

[sse.go:157-L164](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/sse.go#L157-L164)

```go
for event := range pl.ReadOrDone(ctx, c.msgC) {
    err := writeEvent(ctx, w, event, writeTimeOut)
    if err != nil {
        return  // 写入失败（客户端断开）则终止连接
    }
}
```

### 5.2.2 单协程调度的完整事件循环

[sse.go:213-L270](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/sse.go#L213-L270)

```go
func (b *broker) listen() {
    keepAlive := time.NewTicker(keepAliveFrequency)  // 15s
    defer keepAlive.Stop()

    clients := map[client]struct{}{}  // 所有客户端状态，单协程内无需锁
    var eventId uint64

    getNextEventId := func() uint64 { eventId++; return eventId }

    for {
        select {
        // 新客户端连接
        case c := <-b.subscribing:
            clients[c] = struct{}{}
            sendOrDrop(c, serverStartEvent)

        // 客户端断开
        case c := <-b.unsubscribing:
            close(c.msgC)
            delete(clients, c)

        // 外部事件发布
        case msg := <-b.publish:
            msg.id = getNextEventId()
            for c := range clients {
                if b.shouldSend(msg, c) {
                    sendOrDrop(c, msg)  // 每个客户端独立判定
                }
            }

        // 定时心跳
        case ts := <-keepAlive.C:
            msg := b.prepareMessage(...)
            msg.id = getNextEventId()
            for c := range clients {
                sendOrDrop(c, msg)
            }
        }
    }
}
```

**单协程的并发安全保证**：
- `clients` map 的所有读写都在同一个 goroutine 内
- 没有互斥锁，但通过 channel 串行化所有访问
- 事件 ID 单调递增，无需原子操作

### 5.3 发送 API：定向 vs 广播

[sse.go:82-L99](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/sse.go#L82-L99)

```go
// 广播：所有连接的客户端（通过 context 标记）
func (b *broker) SendBroadcastMessage(ctx context.Context, evt Event) {
    ctx = broadcastToAll(ctx)
    b.SendMessage(ctx, evt)
}

// 定向：默认发给同用户的其他客户端（排除触发者自身）
func (b *broker) SendMessage(ctx context.Context, evt Event) {
    b.publish <- b.prepareMessage(ctx, evt)
}
```

**分发判定逻辑 `shouldSend()`**：

[sse.go:196-L211](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/sse.go#L196-L211)

```go
func (b *broker) shouldSend(msg message, c client) bool {
    // 1. broadcastToAll=true → 所有人（配置变更场景）
    if broadcastToAll { return true }
    // 2. 非客户端触发 → 所有人（系统事件）
    if !originatedFromClient { return true }
    // 3. 同一 clientUniqueId → 不发（避免回显）
    if c.clientUniqueId == clientUniqueId { return false }
    // 4. 同用户 → 发送（多端同步）
    if username == c.username { return true }
    return true
}
```

### 5.4 核心事件：`RefreshResource`

用于细粒度的资源刷新通知，支持 `*` 通配符：

[events.go:61-L89](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/events.go#L61-L89)

```go
type RefreshResource struct {
    baseEvent
    resources map[string][]string  // resourceType → [ids...]
}

func (rr *RefreshResource) With(resource string, ids ...string) *RefreshResource {
    if len(ids) == 0 {
        rr.resources[resource] = append(rr.resources[resource], Any) // Any = "*"
    }
    rr.resources[resource] = append(rr.resources[resource], ids...)
}
```

**使用示例（广播所有音乐资源变更）**：

[controller.go:234-L237](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/scanner/controller.go#L234-L237)

```go
if s.changesDetected {
    s.broker.SendBroadcastMessage(ctx, &events.RefreshResource{})  // 全量刷新
}
```

#### 5.4.1 Scanner 扫描结束 RefreshResource 的判定与广播范围

扫描结束时是否发送 `RefreshResource` 事件，由 `changesDetected` 标志位决定。这是一个贯穿整个扫描流程的**原子布尔值**，各阶段发现变更时会将其置为 `true`。

##### 判定逻辑总览

整个判定链路分为三层：

```
scanState.changesDetected (scanner.go 内部，atomic.Bool)
        ↓ 各阶段设置
trackProgress() 收集 → s.changesDetected (controller 层，普通 bool)
        ↓ 扫描结束时判定
SendBroadcastMessage(RefreshResource{})
```

##### 第一层：scanState 内的变更标记（atomic.Bool）

[scanner.go:30-L69](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/scanner/scanner.go#L30-L69)

```go
type scanState struct {
    // ...
    changesDetected atomic.Bool  // 原子布尔，各阶段并发安全设置
}

// 全量扫描直接设为 true（强制刷新）
if fullScan {
    state.changesDetected.Store(true)
}
```

**关键规则**：**全量扫描时 `changesDetected` 初始值就是 `true`**，确保所有维护操作都会执行，前端一定会收到刷新事件。增量扫描初始为 `false`，只有检测到实际变更才置为 `true`。

##### 第二层：各阶段检测到变更时置位

以下阶段会检测变更并设置 `changesDetected = true`：

| 阶段 | 触发条件 | 代码位置 |
|------|----------|----------|
| 阶段 1（加载媒体文件） | 检测到新增/修改/删除的文件 | `phase_1_scan.go` |
| 阶段 2（元数据提取） | 专辑/歌曲元数据发生变化 | `phase_2_metadata.go` |
| 阶段 4（播放列表导入） | 播放列表刷新数量 > 0 | [phase_4_playlists.go:124](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/scanner/phase_4_playlists.go#L124) |
| ...（其他阶段） | 检测到数据变更 | 各 phase 的 `finalize()` |

以播放列表阶段为例：

```go
func (p *phasePlaylists) finalize(err error) error {
    refreshed := p.refreshed.Load()
    if refreshed > 0 {
        p.scanState.changesDetected.Store(true)  // 有刷新 → 标记变更
    }
    return err
}
```

##### 第三层：controller 收集并最终判定

[controller.go:271-L310](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/scanner/controller.go#L271-L310)

`trackProgress()` 函数在扫描过程中持续接收进度消息，其中 `ChangesDetected` 标志位会汇总到 controller 层：

```go
func (s *controller) trackProgress(ctx context.Context, progress <-chan *ProgressInfo) {
    s.changesDetected = false  // 初始为 false
    
    for p := range pl.ReadOrDone(ctx, progress) {
        if p.ChangesDetected {  // 收到变更通知
            s.changesDetected = true
            continue
        }
        // ... 处理其他进度消息
    }
}
```

扫描结束后，在 `Rescan()` 函数中做最终判定：

```go
// 如果检测到变更，向所有客户端发送刷新事件
if s.changesDetected {
    log.Debug(ctx, "Library changes imported. Sending refresh event")
    s.broker.SendBroadcastMessage(ctx, &events.RefreshResource{})
}
```

##### 广播范围

**扫描结束发送的是一个空的 `RefreshResource{}`**，这意味着：

- `resources` map 为空 → 在前端被解释为 **通配符 `*`**
- 所有订阅了事件的客户端都会收到
- **所有资源类型都需要刷新**（专辑、歌曲、艺术家、播放列表、用户等）

> **设计考虑**：扫描可能导致跨多种资源类型的级联变更（如新增专辑同时涉及艺术家、歌曲、封面等），为避免细粒度追踪的复杂性，统一发送全量刷新事件，由前端决定哪些页面需要重新拉取数据。

**使用示例（细粒度插件刷新）**：

[manager.go:86-L92](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/plugins/manager.go#L86-L92)

```go
func (m *Manager) sendPluginRefreshEvent(ctx context.Context, pluginIDs ...string) {
    event := (&events.RefreshResource{}).With("plugin", pluginIDs...)
    m.broker.SendBroadcastMessage(ctx, event)
}
```

### 5.5 listen 事件循环

[sse.go:213-L270](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/sse.go#L213-L270)

```go
func (b *broker) listen() {
    clients := map[client]struct{}{}
    for {
        select {
        case c := <-b.subscribing:
            clients[c] = struct{}{}
            sendOrDrop(c, serverStartEvent)  // 新连接立即发送启动事件

        case c := <-b.unsubscribing:
            close(c.msgC); delete(clients, c)

        case msg := <-b.publish:
            msg.id = getNextEventId()
            for c := range clients {
                if b.shouldSend(msg, c) {
                    sendOrDrop(c, msg)  // 非阻塞写入：满则丢弃
                }
            }

        case ts := <-keepAlive.C:
            // 每 15s 向所有客户端发送心跳
        }
    }
}
```

### 5.6 HTTP 处理器（SSE 握手）

[sse.go:133-L166](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/sse.go#L133-L166)

关键响应头：
```
Content-Type: text/event-stream
Cache-Control: no-cache, no-transform
Connection: keep-alive
X-Accel-Buffering: no   // 禁用 Nginx 缓冲
```

---

## 六、前端事件消费链路

### 6.1 SSE 连接建立

[eventStream.js:7-L58](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/eventStream.js#L7-L58)

```javascript
const newEventStream = async () => {
    let url = baseUrl(`${REST_URL}/events?jwt=${token}`)
    return new EventSource(url)  // 原生 EventSource API
}
```

**监听事件类型**：
```javascript
stream.addEventListener('serverStart', handler)
stream.addEventListener('scanStatus', throttledHandler)  // 100ms 节流
stream.addEventListener('refreshResource', handler)
stream.addEventListener('nowPlayingCount', handler)
stream.addEventListener('keepAlive', handler)  // 仅丢弃，不 dispatch
```

**自动重连**：出错后 5 秒重试，重连成功后 dispatch `streamReconnected`。

### 6.2 Redux 状态层：activityReducer

[activityReducer.js:25-L61](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/reducers/activityReducer.js#L25-L61)

```javascript
case EVENT_REFRESH_RESOURCE:
    return {
        ...previousState,
        refresh: {
            lastReceived: Date.now(),  // 时间戳用于去重/顺序判定
            resources: data,           // { library: ['1','2'], user: ['uid'] }
        },
    }
```

### 6.3 组件消费 Hooks

**Hook 1：`useResourceRefresh` — React-Admin 资源刷新**

[useResourceRefresh.jsx:66-L97](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/common/useResourceRefresh.jsx#L66-L97)

```javascript
export const useResourceRefresh = (...visibleResources) => {
    const refreshData = useSelector(state => state.activity.refresh)

    // 收到 '*' 通配符 → 全页面 refresh()
    if (resources['*'] === '*' || anyResourceContainsWildcard) {
        refresh()
        return
    }
    // 指定资源 + 指定 ID → 调用 dataProvider.getMany() 精准刷新
    Object.keys(resources).forEach(r => {
        if (visibleResources.includes(r)) {
            dataProvider.getMany(r, { ids: resources[r] })
        }
    })
}
```

**Hook 2：`useRefreshOnEvents` — 自定义回调**

[useRefreshOnEvents.jsx:69-L109](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/common/useRefreshOnEvents.jsx#L69-L109)

```javascript
export const useRefreshOnEvents = ({ events, onRefresh }) => {
    // 监听指定事件类型，异步执行自定义 onRefresh 回调
    const shouldRefresh = events.some(eventType => resources[eventType])
    if (shouldRefresh) { onRefresh() }
}
```

---

## 七、需要重启才生效的配置提示

### 7.1 转码配置（Transcoding）安全提示与前端渲染链路

**配置项**：`EnableTranscodingConfig`

#### 完整的数据流转链路

**步骤 1：后端注入前端配置**

[serve_index.go:45-L50](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/serve_index.go#L45-L50)

```go
appConfig := map[string]any{
    // ...
    "enableTranscodingConfig":   conf.Server.EnableTranscodingConfig,
    // ...
}
```

**步骤 2：前端配置初始化**

[config.js:50-L58](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/config.js#L50-L58)

```javascript
const appConfig = JSON.parse(window.__APP_CONFIG__)
config = { ...defaultConfig, ...appConfig }
// config.enableTranscodingConfig 可直接访问
```

**步骤 3：列表页面根据开关控制编辑权限**

[TranscodingList.jsx:7-L31](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/transcoding/TranscodingList.jsx#L7-L31)

```javascript
import config from '../config'

const TranscodingList = (props) => {
  return (
    <List
      {...props}
      bulkActionButtons={config.enableTranscodingConfig}  // 批量操作开关
    >
      <Datagrid
        rowClick={config.enableTranscodingConfig ? 'edit' : 'show'}  // 点击行为：编辑 or 只读查看
      >
        {/* ... 字段 */}
      </Datagrid>
    </List>
  )
}
```

**步骤 4：编辑页面 vs 只读页面**

当 `enableTranscodingConfig=false` 时，点击行进入 **只读 Show 页面**：

[TranscodingShow.jsx:10-L25](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/transcoding/TranscodingShow.jsx#L10-L25)

```javascript
const TranscodingShow = (props) => {
  return (
    <>
      <TranscodingNote message={'message.transcodingDisabled'} />  // 禁用提示
      <Show {...props}>
        <SimpleShowLayout>{/* 只读字段 */}</SimpleShowLayout>
      </Show>
    </>
  )
}
```

当 `enableTranscodingConfig=true` 时，点击行进入 **可编辑 Edit 页面**：

[TranscodingEdit.jsx:22-L37](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/transcoding/TranscodingEdit.jsx#L22-L37)

```javascript
const TranscodingEdit = (props) => {
  return (
    <>
      <TranscodingNote message={'message.transcodingEnabled'} />  // 安全警告
      <Edit {...props}>
        <SimpleForm>{/* 可编辑表单字段 */}</SimpleForm>
      </Edit>
    </>
  )
}
```

**步骤 5：提示卡片组件 `TranscodingNote`**

[TranscodingNote.jsx:15-L33](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/transcoding/TranscodingNote.jsx#L15-L33)

```javascript
export const TranscodingNote = ({ message }) => {
  const translate = useTranslate()
  return (
    <Card>
      <CardContent>
        <Typography>
          <Box fontWeight="fontWeightBold">
            {translate('message.note')}:
          </Box>{' '}
          <Interpolate message={translate(message)} field={'config'}>
            <Box fontFamily="Monospace">
              ND_ENABLETRANSCODINGCONFIG=true  {/* 突出显示配置项名 */}
            </Box>
          </Interpolate>
        </Typography>
      </CardContent>
    </Card>
  )
}
```

**辅助组件 `Interpolate`**：用于将 i18n 字符串中的 `%{config}` 占位符替换为高亮的 JSX 元素。

**提示文案定义**（英文 i18n 资源）：

[en.json:569-L570](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/i18n/en.json#L569-L570)

```json
{
  "message": {
    "transcodingDisabled":
      "Changing the transcoding configuration through the web interface is disabled for security reasons. If you would like to change (edit or add) transcoding options, restart the server with the %{config} configuration option.",
    "transcodingEnabled":
      "Navidrome is currently running with %{config}, making it possible to run system commands from the transcoding settings using the web interface. We recommend to disable it for security reasons and only enable it when configuring Transcoding options."
  }
}
```

**生效机制**：此开关通过 `conf.Server.EnableTranscodingConfig` 控制，**修改配置文件后必须重启服务**（`EnableTranscodingConfig` 无运行时热修改 API）。

**重启后效果对比**：

| 状态 | `enableTranscodingConfig=false` | `enableTranscodingConfig=true` |
|------|-------------------------------|-------------------------------|
| 列表点击行为 | 进入只读 Show 页面 | 进入可编辑 Edit 页面 |
| 批量操作按钮 | ❌ 隐藏 | ✅ 显示 |
| 提示卡片 | 黄色禁用提示，引导设置 `ND_ENABLETRANSCODINGCONFIG=true` 后重启 | 红色安全警告，建议用完后关闭 |
| 表单可编辑性 | ❌ 所有字段只读 | ✅ 可编辑 `command` 等危险字段 |

### 7.2 除 EnableTranscodingConfig 外无 UI 重启提示

**明确结论**：在所有需要重启才能生效的配置项中，**只有 `EnableTranscodingConfig` 提供了前端 UI 提示**。其余配置项均不提供任何 UI 层面的重启提醒：

| 配置项 | UI 提示形式 | 提示位置 |
|--------|------------|---------|
| **`EnableTranscodingConfig`** | ✅ 完整提示卡片（`TranscodingNote` 组件） | 转码列表 / 编辑 / 只读页面顶部 |
| `Port` / `Address` | ❌ 无 | — |
| `LogLevel` / `LogFile` | ❌ 无 | — |
| `DataFolder` / `CacheFolder` | ❌ 无 | — |
| `Scanner.Enabled` / `Schedule` | ❌ 无 | — |
| `PasswordEncryptionKey` | ❌ 无 | — |
| `Plugins.Enabled` / `Plugins.Folder` | ❌ 无 | — |
| `EnableExternalServices` | ❌ 无 | — |
| `Prometheus.*` / `Jukebox.*` | ❌ 无 | — |

**提示输出的分布位置**：

- **`EnableTranscodingConfig`**：前端 React 组件（[TranscodingNote.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/transcoding/TranscodingNote.jsx)）
- **其他需重启配置**：仅在启动阶段的服务端日志（`log.Warn` / `log.Info`）中输出部分警告，如废弃选项告警、越界夹断告警，但**不告知需要重启**
- **Dev 模式配置只读接口**：仅在 `DevUIShowConfig=true` 时可查看当前值，不提示修改后需重启

### 7.3 其他需要重启的配置项（隐含约束）

根据代码架构分析，以下全局配置修改后 **必须重启 Navidrome**：

| 配置类别 | 示例配置项 | 原因 |
|---------|-----------|------|
| **网络/监听** | `Address`, `Port`, `TLSCert`, `TLSKey`, `UnixSocketPerm` | 服务启动时创建 Listener，无运行时重建逻辑 |
| **存储路径** | `MusicFolder`, `DataFolder`, `CacheFolder`, `DbPath` | 启动时初始化 DB 连接、文件系统监听器，运行时不可改 |
| **进程控制** | `EnforceNonRootUser`, `PID` | 仅在进程启动时校验/读取 |
| **功能开关（系统级）** | `Scanner.Enabled`, `Prometheus.Enabled`, `Jukebox.Enabled` | 启动时决定是否初始化对应子系统 |
| **认证/安全** | `PasswordEncryptionKey`, `AuthRequestLimit` | 启动时初始化加密器和限流组件 |
| **日志系统** | `LogLevel`, `LogFile`, `EnableLogRedacting`, `DevLogLevels` | 启动时配置 logrus 输出目标和级别 |
| **集成开关** | `EnableExternalServices` | 启动时决定是否注册 Last.fm/Deezer/ListenBrainz 代理 |
| **插件系统** | `Plugins.Enabled`, `Plugins.Folder`, `Plugins.CacheSize` | Manager.Start() 仅运行一次 |

**注意**：`MusicFolder` 在引入多库支持后已非推荐使用方式（改为 `library` 表管理，可热加载）。

### 7.4 配置文件与环境变量：无运行时重载机制

全局配置（`conf.Server`）的读取流程为：
1. **服务启动** → `InitConfig()` → `Load()` → 填充 `conf.Server`
2. **运行期间** → `conf.Server` 结构体字段被各处直接读取（指针值语义）
3. **无 Hot-Reload** → 未提供 `WatchConfig` / `OnConfigChange` 等 Viper 回调注册

因此所有 `conf/configuration.go` 中的配置项，修改配置文件或环境变量后都**需要重启进程**才能生效。

---

## 八、热加载能力总览矩阵

| 配置类型 | 修改方式 | 持久化目标 | 变更广播 | 前端即时生效 | 后端即时生效 |
|---------|---------|-----------|---------|------------|------------|
| **主题/语言（用户级）** | UI 选择器 → localStorage + Redux | 无（客户端存储） | ❌ | ✅ 无需后端 | ❌ N/A |
| **Last.fm 授权** | OAuth 回调 → SessionKeys.Put() | `user_props` 表 | ❌ | ✅ 前端自行轮询 | ✅ 下次 Scrobble 请求读取 |
| **媒体库路径/名称** | 管理员 → `/api/library` | `library` 表 | ✅ `RefreshResource{library}` | ✅ Hook 自动刷新 | ✅ 重启监听器+触发扫描 |
| **用户-库关联** | 管理员 → `/api/user/{id}/library` | `user_library` 表 | ✅ `RefreshResource{user,library}` | ✅ Hook 自动刷新 | ✅ 下次请求校验权限 |
| **插件启停** | 管理员 → `/api/plugin/{id}` PUT | `plugin` 表 | ✅ `RefreshResource{plugin}` | ✅ Hook 自动刷新 | ✅ `unloadPlugin` → `loadPluginWithConfig` |
| **插件配置** | 管理员 → SchemaEditor → PUT config | `plugin.config` 字段 | ✅ 同上 | ✅ 同上 | ✅ 同上（卸载+重载） |
| **扫描计划** | 通过配置文件 | `conf.Server.Scanner.Schedule` | ❌ | ❌ 需刷新 | ❌ 需重启 Scheduler |
| **端口/日志级别** | 通过配置文件/环境变量 | `conf.Server.*` | ❌ | ❌ 刷新也没用 | ❌ 必须重启进程 |

---

## 九、关键文件索引

| 文件路径 | 核心职责 |
|---------|---------|
| [conf/configuration.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/conf/configuration.go) | 全局配置结构体、默认值、Load 五阶段、mapDeprecatedOption |
| [server/serve_index.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/serve_index.go) | 配置注入前端（`__APP_CONFIG__`，含 enableTranscodingConfig） |
| [server/events/sse.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/sse.go) | SSE Broker、单协程事件循环、sendOrDrop 反压 |
| [server/events/events.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/events/events.go) | 事件类型定义（RefreshResource 等） |
| [server/nativeapi/native_api.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/nativeapi/native_api.go) | Native API 路由注册 |
| [server/nativeapi/config.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/nativeapi/config.go) | Dev 模式配置只读接口 + 脱敏 |
| [server/nativeapi/library.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/nativeapi/library.go) | 用户-库关联 API |
| [server/nativeapi/plugin.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/server/nativeapi/plugin.go) | 插件配置更新 API |
| [core/library.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/core/library.go) | Library 保存/更新/删除 + 路径变更副作用差异 |
| [core/user.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/core/user.go) | 用户删除 → 插件级联卸载的业务编排 |
| [plugins/manager.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/plugins/manager.go) | 插件生命周期、配置热重载、unloadPlugin、UnloadDisabledPlugins |
| [plugins/manager_loader.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/plugins/manager_loader.go) | 插件完整加载链路、extism manifest 构造、buildAllowedPaths、hostServices 表驱动注册 |
| [plugins/manager_plugin.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/plugins/manager_plugin.go) | plugin 结构体定义、实例创建、Close 清理 |
| [plugins/manager_watcher.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/plugins/manager_watcher.go) | 插件目录监听、2s 防抖、SHA256 去重 |
| [plugins/manager_sync.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/plugins/manager_sync.go) | 插件 DB 同步、流式 SHA256 计算 |
| [persistence/property_repository.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/persistence/property_repository.go) | 系统属性 Upsert 实现 |
| [persistence/user_props_repository.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/persistence/user_props_repository.go) | 用户偏好 Upsert 实现 |
| [persistence/user_repository.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/persistence/user_repository.go) | 用户 DB 删除 + 调用 cleanupPluginUserReferences |
| [persistence/plugin_cleanup.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/persistence/plugin_cleanup.go) | 用户/库删除时 plugin JSON 数组清理 + 自动禁用 SQL |
| [scanner/controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/scanner/controller.go) | Scanner 控制器、changesDetected 收集、RefreshResource 广播判定 |
| [scanner/scanner.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/scanner/scanner.go) | Scanner 主逻辑、scanState、全量扫描强制标记变更 |
| [scanner/phase_4_playlists.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/scanner/phase_4_playlists.go) | 播放列表扫描阶段、changesDetected 置位示例 |
| [consts/consts.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/consts/consts.go) | PID/CoverArtSize/登录背景 URL 默认值常量 |
| [core/agents/session_keys.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/core/agents/session_keys.go) | UserProps 封装（Last.fm/ListenBrainz Session） |
| [adapters/lastfm/auth_router.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/adapters/lastfm/auth_router.go) | Last.fm 授权 API |
| [utils/pl/pipelines.go](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/utils/pl/pipelines.go) | ReadOrDone、SendOrDone 通道工具 |
| [ui/src/config.js](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/config.js) | 前端配置初始化（从 window.__APP_CONFIG__ 读取） |
| [ui/src/eventStream.js](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/eventStream.js) | 前端 SSE 连接 + 事件分发 |
| [ui/src/reducers/activityReducer.js](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/reducers/activityReducer.js) | Redux 事件状态管理 |
| [ui/src/common/useResourceRefresh.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/common/useResourceRefresh.jsx) | react-admin 资源自动刷新 Hook |
| [ui/src/common/useRefreshOnEvents.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/common/useRefreshOnEvents.jsx) | 自定义回调刷新 Hook |
| [ui/src/transcoding/TranscodingList.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/transcoding/TranscodingList.jsx) | 转码列表（根据开关控制编辑权限） |
| [ui/src/transcoding/TranscodingShow.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/transcoding/TranscodingShow.jsx) | 转码只读页面（禁用状态） |
| [ui/src/transcoding/TranscodingEdit.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/transcoding/TranscodingEdit.jsx) | 转码编辑页面（启用状态） |
| [ui/src/transcoding/TranscodingNote.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/transcoding/TranscodingNote.jsx) | 转码提示卡片 + Interpolate 组件 |
| [ui/src/i18n/en.json](file:///d:/fz/0601-1/solo-dogfeeding/code/93-navidrome/ui/src/i18n/en.json) | 重启提示文案定义 |
