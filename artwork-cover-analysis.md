# Navidrome 封面管理系统详细分析报告

> 本报告基于代码可复核，所有结论均附代码位置引用。
> 核心代码目录：`core/artwork/`

---

## 一、核心架构与设计思想

### 1.1 核心设计模式

Navidrome 封面管理采用 **责任链模式 + 策略模式** 的组合设计：

```
┌─────────────────────────────────────────────────────────────┐
│                   Artwork 接口                               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Get(ctx, artID, size, square)                        │  │
│  │  GetOrPlaceholder(ctx, id, size, square)              │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   artwork 结构体                            │
│  - ds: DataStore        (数据访问)                          │
│  - cache: FileCache     (图片缓存)                          │
│  - ffmpeg: FFmpeg       (音视频处理)                        │
│  - provider: Provider   (外部元数据)                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              artworkReader 接口族（策略模式）                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │ albumReader     │  │ artistReader    │  │ discReader  │ │
│  │ playlistReader  │  │ radioReader     │  │ resizedReader│ │
│  │ mediafileReader │  │                 │  │             │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
│  共同接口：Key() / LastUpdated() / Reader()                 │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│            selectImageReader（责任链模式）                    │
│  按顺序尝试 sourceFunc 列表，第一个成功即返回                 │
└─────────────────────────────────────────────────────────────┘
```

**代码位置**：
- `Artwork` 接口：`core/artwork/artwork.go:22-25`
- `artworkReader` 接口：`core/artwork/artwork.go:38-42`
- `selectImageReader` 核心调度：`core/artwork/sources.go:27-42`

### 1.2 sourceFunc 抽象

所有封面来源都统一为 `sourceFunc` 函数类型：

```go
type sourceFunc func() (r io.ReadCloser, path string, err error)
```

**契约规则**（`core/artwork/sources.go:44`）：
- 返回非 nil `ReadCloser` 表示成功
- 返回 nil `ReadCloser` + 非 nil error 表示失败（继续尝试下一个）
- 返回 nil `ReadCloser` + nil error 表示"不适用"（继续尝试下一个）

---

## 二、来源优先级决策链详解

### 2.1 优先级决策的核心机制

优先级决策分两步完成：

**Step 1: 构建 sourceFunc 列表** — 根据配置字符串按顺序生成来源函数列表
**Step 2: 责任链执行** — `selectImageReader` 按顺序尝试，第一个成功即返回

**核心代码**（`core/artwork/sources.go:27-42`）：
```go
func selectImageReader(ctx context.Context, artID model.ArtworkID, extractFuncs ...sourceFunc) (io.ReadCloser, string, error) {
    for _, f := range extractFuncs {
        if ctx.Err() != nil {
            return nil, "", ctx.Err()
        }
        r, path, err := f()
        if r != nil {
            return r, path, nil  // 成功，短路返回
        }
        // 失败，继续下一个
    }
    return nil, "", fmt.Errorf("could not get ...: %w", ErrUnavailable)
}
```

---

### 2.2 专辑封面（Album）决策链

**默认配置**（`conf/configuration.go:766`）：
```
cover.*, folder.*, front.*, embedded, external
```

**决策链构建逻辑**（`core/artwork/reader_album.go:86-104`）：

```go
func (a *albumArtworkReader) fromCoverArtPriority(ctx context.Context, ffmpeg ffmpeg.FFmpeg, priority string) []sourceFunc {
    var ff []sourceFunc
    for pattern := range strings.SplitSeq(strings.ToLower(priority), ",") {
        pattern = strings.TrimSpace(pattern)
        switch {
        case pattern == "embedded":
            // 一个模式生成两个 sourceFunc：taglib -> ffmpeg
            ff = append(ff,
                fromTag(ctx, a.lib.FS, embedRel),        // 先试 taglib
                fromFFmpegTag(ctx, ffmpeg, a.lib.Abs(embedRel)),  // 再试 ffmpeg
            )
        case pattern == "external":
            ff = append(ff, fromAlbumExternalSource(ctx, a.album, a.provider))
        case len(a.imgFiles) > 0:
            ff = append(ff, fromExternalFile(ctx, a.lib.FS, a.imgFiles, pattern))
        }
    }
    return ff
}
```

**完整决策链（默认配置展开后）**：

```
1. fromExternalFile(..., "cover.*")    — 匹配 cover.jpg, cover.png 等
2. fromExternalFile(..., "folder.*")   — 匹配 folder.jpg, folder.png 等
3. fromExternalFile(..., "front.*")    — 匹配 front.jpg, front.png 等
4. fromTag(...)                        — taglib 读取内嵌封面
5. fromFFmpegTag(...)                  — ffmpeg 读取内嵌封面（taglib 失败后的回退）
6. fromAlbumExternalSource(...)        — 外部 Agent 获取（Last.fm / Deezer）
```

**内嵌封面的二次尝试设计**（`core/artwork/reader_album.go:93-96`）：
- `fromTag` 使用 `taglib` 库，轻量快速，但兼容性有限
- `fromFFmpegTag` 使用 `ffmpeg` 子进程，兼容性更好但开销较大
- 两者顺序执行，形成"快速路径优先，兼容路径兜底"的策略

**内嵌图片选择策略**（`core/artwork/sources.go:79-137`）：
当音频文件包含多张内嵌图片时，按以下正则优先级选择：
1. `.*cover.*front.*|.*front.*cover.*` — 同时包含 front 和 cover
2. `.*front.*` — 包含 front
3. `.*cover.*` — 包含 cover
4. 都不匹配时返回第一张

---

### 2.3 艺术家封面（Artist）决策链

**默认配置**（`conf/configuration.go:769`）：
```
artist.*, album/artist.*, external
```

**决策链构建逻辑**（`core/artwork/reader_artist.go:117-145`）：

```go
func (a *artistReader) Reader(ctx context.Context) (io.ReadCloser, string, error) {
    // 最高优先级：用户上传的图片（硬编码，不在配置中）
    ff := []sourceFunc{a.fromArtistUploadedImage()}
    // 然后才是配置的优先级
    ff = append(ff, a.fromArtistArtPriority(ctx, conf.Server.ArtistArtPriority)...)
    return selectImageReader(ctx, a.artID, ff...)
}
```

**完整决策链（默认配置展开后）**：

```
1. fromArtistUploadedImage()          — 用户上传的艺术家图片（最高优先级，硬编码）
2. fromArtistFolder(..., "artist.*")  — 艺术家目录匹配 artist.jpg 等
3. fromExternalFile(..., "artist.*")  — 专辑目录匹配 artist.jpg 等（album/ 前缀）
4. fromArtistExternalSource(...)      — 外部 Agent 获取
```

**模式说明**：
- `image-folder`：在 `ArtistImageFolder` 配置的目录中，按 MBID 或艺术家名称匹配
- `album/xxx`：在艺术家的专辑目录中匹配 `xxx` 模式
- 其他模式：在艺术家目录及其父目录（最多 3 层）中匹配

**目录遍历限制**（`core/artwork/reader_artist.go:28`）：
```go
const maxArtistFolderTraversalDepth = 3  // 最多向上查找 3 层父目录
```

---

### 2.4 媒体文件封面（MediaFile）决策链

**无独立配置项**，优先级逻辑硬编码在 `reader_mediafile.go:66-82`：

```go
func (a *mediafileArtworkReader) Reader(ctx context.Context) (io.ReadCloser, string, error) {
    var ff []sourceFunc
    // 如果媒体文件自身的 CoverArtID 是媒体文件类型，先尝试读取内嵌封面
    if a.mediafile.CoverArtID().Kind == model.KindMediaFileArtwork {
        ff = []sourceFunc{
            fromTag(ctx, a.lib.FS, a.mediafile.Path),      // taglib 读取
            fromFFmpegTag(ctx, a.a.ffmpeg, a.lib.Abs(a.mediafile.Path)),  // ffmpeg 回退
        }
    }
    // 回退逻辑：多碟专辑回退到光碟封面，单碟专辑回退到专辑封面
    if len(a.album.Discs) > 1 {
        ff = append(ff, fromAlbum(ctx, a.a, a.mediafile.DiscCoverArtID()))
    } else {
        ff = append(ff, fromAlbum(ctx, a.a, a.mediafile.AlbumCoverArtID()))
    }
    return selectImageReader(ctx, a.artID, ff...)
}
```

**完整决策链**：

```
1. fromTag(mediafile.Path)             — taglib 读取媒体文件自身的内嵌封面
2. fromFFmpegTag(mediafile.Path)       — ffmpeg 读取内嵌封面（taglib 失败后）
3. 回退逻辑：
   ├─ 多碟专辑 → fromAlbum(DiscCoverArtID)   — 回退到对应光碟的封面
   └─ 单碟专辑 → fromAlbum(AlbumCoverArtID)  — 回退到专辑封面
```

**关键特性**：
- 只有当 `mediafile.CoverArtID().Kind == KindMediaFileArtwork` 时才会尝试读取自身的内嵌封面
- 如果媒体文件没有独立的内嵌封面（CoverArtID 指向专辑或光碟），直接跳过后备逻辑
- 这是唯一**不依赖用户配置字符串**的封面类型，优先级完全由代码逻辑决定

---

### 2.5 光碟封面（Disc）决策链

**默认配置**（`conf/configuration.go:771`）：
```
disc*.*, cd*.*, cover.*, folder.*, front.*, discsubtitle, embedded
```

**决策链构建逻辑**（`core/artwork/reader_disc.go:129-158`）：

```go
func (d *discArtworkReader) Reader(ctx context.Context) (io.ReadCloser, string, error) {
    var ff = d.fromDiscArtPriority(ctx, d.a.ffmpeg, conf.Server.DiscArtPriority)
    // 硬编码的最终回退：专辑封面
    albumArtID := model.NewArtworkID(model.KindAlbumArtwork, d.album.ID, &d.album.UpdatedAt)
    ff = append(ff, fromAlbum(ctx, d.a, albumArtID))
    return selectImageReader(ctx, d.cacheKey.artID, ff...)
}
```

**特殊模式**：
- `discsubtitle`：匹配文件名与光碟副标题相同的图片
- 数字模式匹配：如 `disc*.*` 会提取文件名中的数字，匹配对应碟片编号

**完整决策链（默认配置展开后）**：

```
1. fromExternalFile(..., "disc*.*")    — 匹配 disc01.jpg, disc2.png 等
2. fromExternalFile(..., "cd*.*")      — 匹配 cd1.jpg, cd02.png 等
3. fromExternalFile(..., "cover.*")    — 匹配 cover.jpg 等
4. fromExternalFile(..., "folder.*")   — 匹配 folder.jpg 等
5. fromExternalFile(..., "front.*")    — 匹配 front.jpg 等
6. fromDiscSubtitle(...)               — 匹配文件名与光碟副标题相同
7. fromTag(...)                        — taglib 读取第一首音轨的内嵌封面
8. fromFFmpegTag(...)                  — ffmpeg 读取第一首音轨的内嵌封面
9. fromAlbum(...)                      — 回退到专辑封面（硬编码兜底）
```

---

### 2.6 播放列表封面（Playlist）决策链

**固定优先级**（无配置项，硬编码）（`core/artwork/reader_playlist.go:68-76`）：

```go
func (a *playlistArtworkReader) Reader(ctx context.Context) (io.ReadCloser, string, error) {
    return selectImageReader(ctx, a.artID,
        a.fromPlaylistUploadedImage(),    // 1. 用户上传
        a.fromPlaylistSidecar(ctx),       // 2. 同目录同名边车文件
        a.fromPlaylistExternalImage(ctx), // 3. 播放列表的 ExternalImageURL
        a.fromGeneratedTiledCover(ctx),   // 4. 自动生成 2x2 拼图
        fromAlbumPlaceholder(),           // 5. 占位图
    )
}
```

**各来源说明**：
1. **用户上传**：`data/cache/images/pl-<id>_cover.*`
2. **边车文件**：播放列表文件同目录下同名图片（如 `my.m3u8` → `my.jpg`）
3. **外部 URL**：`playlist.ExternalImageURL` 字段，支持 http/https 或本地路径
   - HTTP URL：需要 `EnableM3UExternalAlbumArt = true` 才会尝试
   - 本地路径：直接读取文件
4. **自动拼图**：随机取 4 张专辑封面拼成 2x2 网格
5. **占位图**：`resources/album-art-placeholder.png`（嵌入式资源）

**修正结论**：播放列表**理论上永不失败**，因为最后有 `fromAlbumPlaceholder()` 兜底。
但 `fromAlbumPlaceholder()` 忽略了 `resources.FS().Open()` 的错误返回（`sources.go:184-188`），如果嵌入式资源损坏，极端情况下仍可能返回 nil。

**自动拼图降级逻辑**（`core/artwork/reader_playlist.go:187-195`）：
- 取到 1 张 → 直接返回该图
- 取到 2 张 → [A, B, B, A] 对称排列
- 取到 3 张 → [A, B, C, A] 排列
- 取到 4 张 → [A, B, C, D] 正常排列
- 取到 0 张 → 返回错误，继续到占位图

---

### 2.7 电台封面（Radio）决策链

**仅支持用户上传**（`core/artwork/reader_radio.go:32-36`）：

```go
func (a *radioArtworkReader) Reader(ctx context.Context) (io.ReadCloser, string, error) {
    return selectImageReader(ctx, a.artID,
        a.fromRadioUploadedImage(),  // 仅支持用户上传
    )
}
```

---

## 三、外部来源优先顺序的配置驱动机制

### 3.1 Agent 优先级配置

外部元数据获取的优先级完全由 `conf.Server.Agents` 配置决定。

**默认配置**（`conf/configuration.go` 中 `Agents` 默认为空字符串）：
- 当 `Agents` 为空时，仅启用 `local` Agent（本地数据，不调用外部 API）

**典型配置示例**：
```toml
Agents = "lastfm,deezer"
```

这表示：
1. 先尝试 Last.fm 获取元数据
2. Last.fm 失败后尝试 Deezer
3. 都失败则最后尝试 `local` Agent（自动追加，无需显式配置）

### 3.2 Agent 列表构建逻辑

**代码位置**：`core/agents/agents.go:58-97`

```go
func (a *Agents) getEnabledAgentNames() []enabledAgent {
    // 1. 无配置时仅用 local
    if conf.Server.Agents == "" {
        return []enabledAgent{{name: LocalAgentName, isPlugin: false}}
    }

    configuredAgents := strings.Split(conf.Server.Agents, ",")

    // 2. 自动追加 local（如果未在配置中）
    hasLocalAgent := slices.Contains(configuredAgents, LocalAgentName)
    if !hasLocalAgent {
        configuredAgents = append(configuredAgents, LocalAgentName)
    }

    // 3. 过滤有效 Agent（内置或插件）
    var validAgents []enabledAgent
    for _, name := range configuredAgents {
        isBuiltIn := Map[name] != nil
        isPlugin := slices.Contains(availablePlugins, name)
        if isBuiltIn {
            validAgents = append(validAgents, enabledAgent{name: name, isPlugin: false})
        } else if isPlugin {
            validAgents = append(validAgents, enabledAgent{name: name, isPlugin: true})
        }
    }
    return validAgents
}
```

**关键规则**：
1. `local` Agent **始终**被包含（自动追加到末尾）
2. 配置为空时，**仅**使用 `local` Agent
3. 配置顺序决定调用顺序
4. 无效的 Agent 名称会被静默过滤

### 3.3 Agent 执行顺序

**代码位置**：`core/agents/agents.go:322-345`

```go
func callAgentMethod[T comparable](ctx context.Context, agents *Agents, methodName string, fn func(Interface) (T, error)) (T, error) {
    for _, enabledAgent := range agents.getEnabledAgentNames() {
        ag := agents.getAgent(enabledAgent)
        if ag == nil {
            continue
        }
        result, err := fn(ag)
        if err != nil {
            log.Trace(ctx, "Agent method call error", ...)
            continue  // 单个 Agent 失败，静默继续
        }
        if result != zero {
            return result, nil  // 第一个成功即返回
        }
    }
    return zero, ErrNotFound  // 全部失败
}
```

**示例流程**（配置 `Agents = "lastfm,deezer"`）：
```
调用 ArtistImage
    │
    ├─► lastfm Agent
    │     ├─ 成功 → 返回 URL
    │     └─ 失败 → 继续
    │
    ├─► deezer Agent
    │     ├─ 成功 → 返回 URL
    │     └─ 失败 → 继续
    │
    └─► local Agent（自动追加）
          └─ 失败（local 不提供图片）→ 返回 ErrNotFound
```

### 3.4 可用的内置 Agent

| Agent 名称 | 能力 | 代码位置 |
|-----------|------|----------|
| `lastfm` | 艺术家头像、专辑封面、艺术家简介、相似艺术家 | `adapters/lastfm/agent.go` |
| `deezer` | 艺术家头像、专辑封面 | `adapters/deezer/deezer.go` |
| `local` | 本地热门歌曲（无图片能力） | `core/agents/local_agent.go` |

---

## 四、缓存键构成逻辑详解

### 4.1 缓存键设计目标

缓存键必须满足：
1. **实体唯一性**：不同实体的封面不能冲突
2. **内容新鲜性**：实体更新后缓存自动失效
3. **配置感知**：配置变更后缓存自动失效
4. **尺寸区分**：不同尺寸/质量的图片分开缓存

### 4.2 基础缓存键（cacheKey）

所有类型共享的基础结构（`core/artwork/image_cache.go:16-28`）：

```go
type cacheKey struct {
    artID      model.ArtworkID
    lastUpdate time.Time
}

func (k *cacheKey) Key() string {
    return fmt.Sprintf(
        "%s-%s.%d",
        k.artID.Kind,      // [1] 类型前缀
        k.artID.ID,        // [2] 实体 ID
        k.lastUpdate.UnixMilli(),  // [3] 时间戳（毫秒）
    )
}
```

**字段含义**：

| 字段 | 含义 | 示例值 |
|------|------|--------|
| `artID.Kind` | 封面类型的前缀缩写 | `al` (专辑), `ar` (艺术家), `mf` (媒体文件), `pl` (播放列表), `dc` (光碟), `ra` (电台) |
| `artID.ID` | 实体的数据库 ID | `al-123`, `ar-456` |
| `lastUpdate` | 实体相关的最新更新时间戳（毫秒） | `1716000000000` |

**基础键示例**：
```
al-al-123.1716000000000
```

---

### 4.3 各类型缓存键扩展

不同类型在基础键上添加配置相关的哈希后缀，确保配置变更时缓存失效。

#### 4.3.1 专辑封面缓存键（`core/artwork/reader_album.go:64-76`）

```go
func (a *albumArtworkReader) Key() string {
    hashInput := conf.Server.CoverArtPriority
    if conf.Server.EnableExternalServices {
        hashInput = conf.Server.Agents + hashInput
    }
    hash := md5.Sum([]byte(hashInput))
    return fmt.Sprintf(
        "%s.%x.%t",
        a.cacheKey.Key(),           // 基础键
        hash,                       // CoverArtPriority + Agents 的 MD5
        conf.Server.EnableExternalServices,  // 是否启用外部服务
    )
}
```

**字段含义**：

| 字段 | 含义 | 目的 |
|------|------|------|
| 基础键 | `{kind}-{id}.{timestamp}` | 实体唯一标识 + 内容新鲜性 |
| `hash` | `md5(Agents + CoverArtPriority)` | 优先级配置或 Agent 列表变更时失效 |
| `EnableExternalServices` | `true`/`false` | 外部服务开关变更时失效 |

**完整示例**：
```
al-al-123.1716000000000.a1b2c3d4e5f6.true
```

---

#### 4.3.2 艺术家封面缓存键（`core/artwork/reader_artist.go:103-111`）

```go
func (a *artistReader) Key() string {
    hash := md5.Sum([]byte(conf.Server.Agents))
    return fmt.Sprintf(
        "%s.%t.%x",
        a.cacheKey.Key(),           // 基础键
        conf.Server.EnableExternalServices,  // 是否启用外部服务
        hash,                       // Agents 配置的 MD5
    )
}
```

**字段含义**：

| 字段 | 含义 | 目的 |
|------|------|------|
| 基础键 | `{kind}-{id}.{timestamp}` | 实体唯一标识 + 内容新鲜性 |
| `EnableExternalServices` | `true`/`false` | 外部服务开关变更时失效 |
| `hash` | `md5(Agents)` | Agent 列表变更时失效 |

**完整示例**：
```
ar-ar-456.1716000000000.true.a1b2c3d4e5f6
```

---

#### 4.3.3 媒体文件封面缓存键（`core/artwork/reader_mediafile.go:55-61`）

```go
func (a *mediafileArtworkReader) Key() string {
    return fmt.Sprintf(
        "%s.%t",
        a.cacheKey.Key(),
        conf.Server.EnableMediaFileCoverArt,
    )
}
```

**字段含义**：

| 字段 | 含义 | 目的 |
|------|------|------|
| 基础键 | `{kind}-{id}.{timestamp}` | 实体唯一标识 + 内容新鲜性 |
| `EnableMediaFileCoverArt` | `true`/`false` | 媒体文件封面开关变更时失效 |

**完整示例**：
```
mf-mf-789.1716000000000.true
```

---

#### 4.3.4 光碟封面缓存键（`core/artwork/reader_disc.go:116-123`）

```go
func (d *discArtworkReader) Key() string {
    hash := md5.Sum([]byte(conf.Server.DiscArtPriority))
    return fmt.Sprintf("%s.%x", d.cacheKey.Key(), hash)
}
```

**字段含义**：

| 字段 | 含义 | 目的 |
|------|------|------|
| 基础键 | `{kind}-{id}.{timestamp}` | 实体唯一标识 + 内容新鲜性 |
| `hash` | `md5(DiscArtPriority)` | 光碟封面优先级配置变更时失效 |

**完整示例**：
```
dc-al-123:1.1716000000000.a1b2c3d4e5f6
```

---

#### 4.3.5 缩放后封面缓存键（`core/artwork/reader_resized.go:63-69`）

```go
func (a *resizedArtworkReader) Key() string {
    baseKey := fmt.Sprintf("%s.%d", a.cacheKey, a.size)
    if a.square {
        return baseKey + ".square"
    }
    return fmt.Sprintf("%s.%d", baseKey, conf.Server.CoverArtQuality)
}
```

**字段含义**：

| 字段 | 含义 | 目的 |
|------|------|------|
| `cacheKey` | 原始封面的完整缓存键 | 关联到原始图片 |
| `size` | 请求的尺寸（像素） | 不同尺寸分开缓存 |
| `square` | 是否裁剪为正方形 | 正方形与非正方形分开缓存 |
| `CoverArtQuality` | JPEG/WebP 质量（0-100） | 质量配置变更时失效 |

**完整示例**：
```
# 非正方形，尺寸 300，质量 75
al-al-123.1716000000000.a1b2c3.300.75

# 正方形，尺寸 300
al-al-123.1716000000000.a1b2c3.300.square
```

---

### 4.4 LastUpdate 时间戳来源

`lastUpdate` 字段取多个时间戳的最大值，确保任何相关内容变更都能使缓存失效。

| 类型 | 时间戳来源（取最大值） | 代码位置 |
|------|------------------------|----------|
| 专辑 | `album.UpdatedAt`、`album.ImportedAt`、所有关联文件夹的 `ImagesUpdatedAt` | `reader_album.go:57-60` |
| 艺术家 | 所有专辑的 `ImagesUpdatedAt`、`artist.UpdatedAt`、艺术家文件夹更新时间、图片文件夹文件修改时间 | `reader_artist.go:84-97` |
| 媒体文件 | `mediafile.UpdatedAt`、`album.UpdatedAt`、`ImagesUpdatedAt` | `reader_mediafile.go:45-51` |
| 播放列表 | `playlist.UpdatedAt`、边车文件修改时间、外部图片文件修改时间 | `reader_playlist.go:44-59` |
| 光碟 | 同专辑 | `reader_disc.go:109-112` |

**设计意图**：
- 不仅考虑实体本身的更新时间
- 还考虑关联图片文件的更新时间（如用户替换了目录下的 cover.jpg）
- 确保用户替换图片后缓存立即失效

---

## 五、外部来源失败后的降级路径

### 5.1 降级路径总览

整个封面获取过程是一条多层级的降级链：

```
HTTP 请求
    │
    ▼
GetOrPlaceholder()  ──  ErrUnavailable  ──► 返回占位图
    │
    ▼
Get()
    │
    ├─► 查缓存 ── 命中 ──► 返回
    │
    ▼
getArtworkReader()  ──  创建对应类型的 Reader
    │
    ▼
cache.Get()  ── 未命中 ──► 调用 Reader.Reader()
    │
    ▼
selectImageReader()  ── 按顺序尝试 sourceFunc 列表
    │
    ├─► sourceFunc 1 成功 ──► 返回
    ├─► sourceFunc 1 失败 ──► sourceFunc 2
    ├─► sourceFunc 2 成功 ──► 返回
    └─► ... 全部失败 ──► 返回 ErrUnavailable
```

---

### 5.2 各类型降级路径详解

#### 5.2.1 专辑封面降级路径

```
配置优先级列表
    │
    ├─► cover.* 匹配文件
    │       ├─ 找到 → 返回
    │       └─ 未找到 → 继续
    │
    ├─► folder.* 匹配文件
    │       ├─ 找到 → 返回
    │       └─ 未找到 → 继续
    │
    ├─► front.* 匹配文件
    │       ├─ 找到 → 返回
    │       └─ 未找到 → 继续
    │
    ├─► embedded (taglib)
    │       ├─ 找到 → 返回
    │       └─ 失败 → fromFFmpegTag
    │                  ├─ 找到 → 返回
    │                  └─ 失败 → 继续
    │
    └─► external (外部 Agent)
            ├─ 查 DB 缓存（仅艺术家有，专辑无）
            │
            └─ 同步调用 Agent
                      ├─ Agent 1 (Last.fm)
                      │     ├─ 成功 → 返回
                      │     └─ 失败 → Agent 2 (Deezer)
                      │                ├─ 成功 → 返回
                      │                └─ 失败 → Agent 3 (local)
                      │                           └─ 失败 → 继续
                      │
                      └─ 全部 Agent 失败 → 返回 ErrUnavailable
                                                    │
                                                    ▼
                                          调用方 GetOrPlaceholder()
                                                    │
                                                    └─► 返回专辑占位图
```

**关键代码路径**：
- 外部文件匹配：`core/artwork/sources.go:56-77`
- 内嵌封面读取：`core/artwork/sources.go:86-166`
- 外部来源调用：`core/artwork/sources.go:201-225`

---

#### 5.2.2 艺术家封面降级路径

```
用户上传图片
    ├─ 存在 → 返回
    └─ 不存在 → 继续
        │
        ▼
配置优先级列表
    ├─► artist.* (艺术家目录)
    │       ├─ 找到 → 返回
    │       └─ 未找到 → 继续
    │
    ├─► album/artist.* (专辑目录)
    │       ├─ 找到 → 返回
    │       └─ 未找到 → 继续
    │
    └─► external (外部 Agent)
            ├─ 查 DB 缓存 → 有 URL → 返回
            │
            └─ 无缓存 → 同步调用 Agent
                      ├─ Agent 1 成功 → 保存到 DB → 返回
                      ├─ Agent 1 失败 → Agent 2
                      │                ├─ 成功 → 保存到 DB → 返回
                      │                └─ 失败 → 继续
                      └─ 全部失败 → 返回 ErrUnavailable
                                                    │
                                                    ▼
                                          调用方 GetOrPlaceholder()
                                                    │
                                                    └─► 返回艺术家占位图
```

**外部来源缓存机制**（`core/external/provider.go:373-402`）：
- 首次调用：同步调用 Agent 获取 URL，保存到 `artist.LargeImageUrl`
- 后续调用：直接从 DB 读取 URL，过期后后台异步刷新
- 过期时间：`conf.Server.DevArtistInfoTimeToLive`（默认 7 天）

---

#### 5.2.3 媒体文件封面降级路径

```
mediafile.CoverArtID.Kind == KindMediaFileArtwork?
    ├─ 是 → 尝试读取内嵌封面
    │       ├─ fromTag(taglib)
    │       │     ├─ 成功 → 返回
    │       │     └─ 失败 → fromFFmpegTag
    │       │                ├─ 成功 → 返回
    │       │                └─ 失败 → 回退
    │       │
    │       └─ 回退逻辑
    │             ├─ 多碟专辑 → fromAlbum(DiscCoverArtID) → 光碟封面
    │             └─ 单碟专辑 → fromAlbum(AlbumCoverArtID) → 专辑封面
    │
    └─ 否 → 直接回退
          ├─ 多碟专辑 → fromAlbum(DiscCoverArtID) → 光碟封面
          └─ 单碟专辑 → fromAlbum(AlbumCoverArtID) → 专辑封面
```

**代码位置**：`core/artwork/reader_mediafile.go:66-82`

---

#### 5.2.4 光碟封面降级路径

```
配置优先级列表
    ├─► disc*.*
    ├─► cd*.*
    ├─► cover.*
    ├─► folder.*
    ├─► front.*
    ├─► discsubtitle
    └─► embedded (taglib → ffmpeg)
    │
    ├─ 任一成功 → 返回
    └─ 全部失败 → 硬编码回退
        │
        ▼
fromAlbum(专辑封面)
    ├─ 成功 → 返回专辑封面
    └─ 失败 → 返回 ErrUnavailable
```

**硬编码回退**（`core/artwork/reader_disc.go:131-133`）：
```go
// Fallback to album cover art
albumArtID := model.NewArtworkID(model.KindAlbumArtwork, d.album.ID, &d.album.UpdatedAt)
ff = append(ff, fromAlbum(ctx, d.a, albumArtID))
```

这是唯一在 Reader 层面硬编码回退到其他类型的设计。

---

#### 5.2.5 播放列表封面降级路径

```
用户上传图片
    ├─ 存在 → 返回
    └─ 不存在 → 继续
        │
        ▼
同目录同名边车文件
    ├─ 存在 → 返回
    └─ 不存在 → 继续
        │
        ▼
播放列表 ExternalImageURL
    ├─ 空 → 继续
    ├─ HTTP URL
    │     ├─ EnableM3UExternalAlbumArt = false → 继续
    │     └─ true → 网络请求
    │           ├─ 200 OK → 返回
    │           └─ 失败 → 继续
    └─ 本地路径 → 读取文件
          ├─ 存在 → 返回
          └─ 失败 → 继续
        │
        ▼
自动生成 2x2 拼图
    ├─ 随机取 4 张专辑封面
    ├─ 取到 ≥1 张 → 拼接 → 返回
    └─ 取到 0 张 → 失败 → 继续
        │
        ▼
占位图（嵌入式资源）→ 返回
```

**自动拼图降级**（`core/artwork/reader_playlist.go:187-195`）：
- 取到 1 张 → 直接返回该图
- 取到 2 张 → [A, B, B, A] 对称排列
- 取到 3 张 → [A, B, C, A] 排列
- 取到 4 张 → [A, B, C, D] 正常排列
- 取到 0 张 → 失败，继续到占位图

---

### 5.3 Agent 内部的降级路径

外部元数据获取通过 `Agents` 聚合器实现多 Agent 依次尝试（`core/agents/agents.go:322-369`）：

```go
func callAgentMethod[T comparable](ctx context.Context, agents *Agents, methodName string, fn func(Interface) (T, error)) (T, error) {
    for _, enabledAgent := range agents.getEnabledAgentNames() {
        ag := agents.getAgent(enabledAgent)
        if ag == nil {
            continue
        }
        result, err := fn(ag)
        if err != nil {
            log.Trace(ctx, "Agent method call error", ...)
            continue  // 单个 Agent 失败，静默继续
        }
        if result != zero {
            return result, nil  // 第一个成功即返回
        }
    }
    return zero, ErrNotFound  // 全部失败
}
```

**Agent 调用特性**：
- 按配置顺序尝试：`conf.Server.Agents`（如 `lastfm,deezer`）
- 单个 Agent 失败仅记录 Trace 日志，不中断流程
- 失败原因不向上传递，仅最终返回 `ErrNotFound`
- 问题排查难度较高，需开启 Trace 日志

**代码位置**：`core/agents/agents.go:322-345`

---

### 5.4 HTTP 请求失败降级

外部图片 URL 获取（`core/artwork/sources.go:212-225`）：

```go
func fromURL(ctx context.Context, imageUrl *url.URL) (io.ReadCloser, string, error) {
    hc := http.Client{Timeout: 5 * time.Second}
    req, _ := http.NewRequestWithContext(ctx, http.MethodGet, imageUrl.String(), nil)
    resp, err := hc.Do(req)
    if err != nil {
        return nil, "", err  // 网络错误
    }
    if resp.StatusCode != http.StatusOK {
        resp.Body.Close()
        return nil, "", fmt.Errorf("error retrieving artwork from %s: %s", imageUrl, resp.Status)
    }
    return resp.Body, imageUrl.String(), nil
}
```

**失败处理**：
- 超时：5 秒超时后返回错误
- 非 200 状态码：返回错误
- 任何错误都导致该 `sourceFunc` 失败，继续下一个来源

---

### 5.5 图片缩放失败降级

缩放处理失败时返回原图（`core/artwork/reader_resized.go:75-103`）：

```go
resized, origSize, err := a.resizeImage(ctx, orig)
if err != nil {
    log.Warn(ctx, "Could not resize image. Will return image as is", ...)
}
if err != nil || resized == nil {
    // 缩放失败，返回原图
    orig, _, err = a.a.Get(ctx, a.artID, 0, false)
    return orig, "", err
}
```

**缩放失败场景**：
- 图片格式不支持
- 图片损坏
- 内存不足
- 编码错误

---

## 六、可复核的结论

### 6.1 优先级决策结论

| 结论 | 代码依据 |
|------|----------|
| **专辑封面**的默认优先级是：`cover.*` > `folder.*` > `front.*` > `embedded` > `external` | `conf/configuration.go:766` |
| **艺术家封面**有一个硬编码的最高优先级：用户上传图片 | `core/artwork/reader_artist.go:118` |
| **媒体文件封面**无配置项，优先级硬编码：自身内嵌封面 → 光碟/专辑封面 | `core/artwork/reader_mediafile.go:66-82` |
| **光碟封面**有一个硬编码的最低优先级：回退到专辑封面 | `core/artwork/reader_disc.go:131-133` |
| **播放列表封面**理论上永不失败，最后有 `fromAlbumPlaceholder()` 兜底 | `core/artwork/reader_playlist.go:74` |
| **内嵌封面**会尝试两次：先 taglib 后 ffmpeg | `core/artwork/reader_album.go:93-96` |
| **电台封面**仅支持用户上传，无其他来源 | `core/artwork/reader_radio.go:32-36` |
| **外部 Agent 顺序**由 `conf.Server.Agents` 配置决定，`local` 始终自动追加 | `core/agents/agents.go:58-77` |

### 6.2 缓存键结论

| 结论 | 代码依据 |
|------|----------|
| 所有缓存键都包含时间戳，确保实体更新后缓存自动失效 | `core/artwork/image_cache.go:26` |
| 专辑缓存键包含 `CoverArtPriority` 和 `Agents` 的 MD5，配置变更自动失效 | `core/artwork/reader_album.go:64-76` |
| 媒体文件缓存键包含 `EnableMediaFileCoverArt` 开关 | `core/artwork/reader_mediafile.go:55-61` |
| 缩放后的图片有独立的缓存键，包含尺寸和质量参数 | `core/artwork/reader_resized.go:63-69` |
| `lastUpdate` 取多个时间戳的最大值，包括图片文件的更新时间 | `reader_album.go:57-60`、`reader_artist.go:84-97` |

### 6.3 降级路径结论

| 结论 | 代码依据 |
|------|----------|
| 所有来源失败最终返回 `ErrUnavailable`，由调用方决定是否返回占位图 | `core/artwork/sources.go:41` |
| `GetOrPlaceholder()` 会捕获 `ErrUnavailable` 并返回占位图，`Get()` 不会 | `core/artwork/artwork.go:44-58` |
| Agent 调用是静默降级的，单个失败仅记录 Trace 日志 | `core/agents/agents.go:335` |
| 艺术家图片有 DB 缓存，首次同步获取，后续从缓存读取 | `core/external/provider.go:373-402` |
| 专辑图片无 DB 缓存，每次都同步调用 Agent | `core/external/provider.go:404-441` |
| 媒体文件封面会根据专辑碟数回退到光碟或专辑封面 | `core/artwork/reader_mediafile.go:76-80` |
| 图片缩放失败时会降级返回原图 | `core/artwork/reader_resized.go:90-96` |
| HTTP 请求超时时间为 5 秒 | `core/artwork/sources.go:213` |
| 播放列表 HTTP 外部封面需 `EnableM3UExternalAlbumArt = true` | `core/artwork/reader_playlist.go:97-98` |

### 6.4 设计权衡结论

| 决策 | 优点 | 缺点 |
|------|------|------|
| sourceFunc 返回 nil 表示失败 | 接口简洁，易于扩展 | 无法区分"不适用"和"失败" |
| Agent 失败静默降级 | 单个服务不可用不影响整体功能 | 问题排查困难，需开启 Trace 日志 |
| 内嵌封面两次尝试（taglib + ffmpeg） | 兼容性好 | 开销较大，某些场景下重复工作 |
| 配置变更通过哈希使缓存失效 | 无需手动清理缓存 | 配置微小变化也会导致全量缓存失效 |
| 播放列表永不失败 | 用户体验好 | 可能掩盖真实问题 |
| Agent 配置自动追加 local | 确保始终有可用的 Agent | 用户可能未意识到 local 始终启用 |
| 媒体文件封面无配置 | 逻辑简单，不易出错 | 灵活性较低 |

---

## 七、关键代码索引

| 功能 | 文件位置 | 行号 |
|------|----------|------|
| 核心调度器 selectImageReader | `core/artwork/sources.go` | 27-42 |
| sourceFunc 类型定义 | `core/artwork/sources.go` | 44 |
| 专辑优先级构建 | `core/artwork/reader_album.go` | 86-104 |
| 艺术家优先级构建 | `core/artwork/reader_artist.go` | 127-145 |
| 媒体文件优先级构建 | `core/artwork/reader_mediafile.go` | 66-82 |
| 光碟优先级构建（含专辑回退） | `core/artwork/reader_disc.go` | 129-158 |
| 播放列表固定优先级链 | `core/artwork/reader_playlist.go` | 68-76 |
| 基础缓存键定义 | `core/artwork/image_cache.go` | 16-28 |
| 专辑缓存键 | `core/artwork/reader_album.go` | 64-76 |
| 艺术家缓存键 | `core/artwork/reader_artist.go` | 103-111 |
| 媒体文件缓存键 | `core/artwork/reader_mediafile.go` | 55-61 |
| 缩放缓存键 | `core/artwork/reader_resized.go` | 63-69 |
| GetOrPlaceholder 占位图逻辑 | `core/artwork/artwork.go` | 44-58 |
| 外部图片 HTTP 客户端 | `core/artwork/sources.go` | 212-225 |
| Agent 聚合器核心循环 | `core/agents/agents.go` | 322-345 |
| Agent 列表构建（含 local 自动追加） | `core/agents/agents.go` | 58-97 |
| 艺术家图片获取（含缓存） | `core/external/provider.go` | 373-402 |
| 专辑图片获取（无缓存） | `core/external/provider.go` | 404-441 |
| 默认配置值 | `conf/configuration.go` | 766-772 |
