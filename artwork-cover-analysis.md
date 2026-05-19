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
│  │ mediafileReader │  │ playlistReader  │  │ radioReader │ │
│  │ resizedReader   │  │                 │  │             │ │
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

**特性开关**：`EnableMediaFileCoverArt`，默认 `true`（`conf/configuration.go:752`）

**决策链构建逻辑**（`core/artwork/reader_mediafile.go:66-81`）：

```go
func (a *mediafileArtworkReader) Reader(ctx context.Context) (io.ReadCloser, string, error) {
    var ff []sourceFunc
    // 仅当媒体文件有独立内嵌封面时，才尝试读取自身的内嵌封面
    if a.mediafile.CoverArtID().Kind == model.KindMediaFileArtwork {
        ff = []sourceFunc{
            fromTag(ctx, a.lib.FS, a.mediafile.Path),       // 1. taglib 读取媒体文件内嵌封面
            fromFFmpegTag(ctx, a.a.ffmpeg, a.lib.Abs(a.mediafile.Path)),  // 2. ffmpeg 回退
        }
    }
    // 多碟专辑 → 回退到光碟封面；单碟专辑 → 回退到专辑封面
    if len(a.album.Discs) > 1 {
        ff = append(ff, fromAlbum(ctx, a.a, a.mediafile.DiscCoverArtID()))
    } else {
        ff = append(ff, fromAlbum(ctx, a.a, a.mediafile.AlbumCoverArtID()))
    }
    return selectImageReader(ctx, a.artID, ff...)
}
```

**关键判断条件**（`model/mediafile.go:117-124`）：
```go
func (mf MediaFile) CoverArtID() ArtworkID {
    // 仅当 HasCoverArt=true 且 EnableMediaFileCoverArt=true 时，才使用媒体文件自身的封面
    if mf.HasCoverArt && conf.Server.EnableMediaFileCoverArt {
        return artworkIDFromMediaFile(mf)
    }
    // 否则回退到光碟或专辑封面
    return mf.DiscCoverArtID()
}
```

**完整决策链**：

```
┌─ 前提条件：mediafile.CoverArtID().Kind == KindMediaFileArtwork
│    即：mf.HasCoverArt == true 且 conf.Server.EnableMediaFileCoverArt == true
│
├─ 1. fromTag(媒体文件路径)         — taglib 读取媒体文件内嵌封面
│       ├─ 成功 → 返回
│       └─ 失败 → 继续
│
├─ 2. fromFFmpegTag(媒体文件路径)   — ffmpeg 读取媒体文件内嵌封面
│       ├─ 成功 → 返回
│       └─ 失败 → 继续
│
└─ 3. 回退逻辑（始终存在，不受前提条件影响）
       ├─ 多碟专辑 → fromAlbum(光碟封面ID)
       │       ├─ 成功 → 返回光碟封面
       │       └─ 失败 → 光碟内部继续回退到专辑封面
       └─ 单碟专辑 → fromAlbum(专辑封面ID)
               ├─ 成功 → 返回专辑封面
               └─ 失败 → 返回 ErrUnavailable
```

**设计意图**：
- 允许同一专辑内不同音轨有不同封面（如合集专辑）
- 但默认行为是所有音轨共享专辑封面（通过回退实现）
- `EnableMediaFileCoverArt` 开关允许用户完全禁用媒体文件级别的封面

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
   - HTTP URL 受 `EnableM3UExternalAlbumArt` 配置控制，默认 `false`（`conf/configuration.go:751`）
4. **自动拼图**：随机取 4 张专辑封面拼成 2x2 网格
5. **占位图**：`resources/album-art-placeholder.png`（编译时嵌入的资源）

**兜底条件修正**：

之前的结论"播放列表永不失败"不完全准确。实际情况是：

- **理论上永不失败**：`fromAlbumPlaceholder()` 调用 `resources.FS().Open(consts.PlaceholderAlbumArt)`，该资源是编译时嵌入的，理论上不会失败
- **实际上可能失败**：如果嵌入式资源在构建时出错或损坏，`Open()` 可能返回 error
- **代码层面没有特殊保证**：`fromAlbumPlaceholder()` 没有捕获或忽略错误，错误会正常返回
- **错误传递路径**：如果占位图打开失败，错误会返回给 `selectImageReader`，最终返回 `ErrUnavailable` 给调用方

**占位图实现**（`core/artwork/sources.go:184-189`）：
```go
func fromAlbumPlaceholder() sourceFunc {
    return func() (io.ReadCloser, string, error) {
        r, _ := resources.FS().Open(consts.PlaceholderAlbumArt)  // 忽略错误，r 可能为 nil
        return r, consts.PlaceholderAlbumArt, nil
    }
}
```

> **注意**：代码中使用 `r, _ := ...` 忽略了 `Open()` 的错误。如果资源不存在，`r` 将为 nil，`selectImageReader` 会认为该 sourceFunc 失败。
> 但由于这是列表中的最后一个 sourceFunc，失败后会返回 `ErrUnavailable`。
> 实际上，由于资源是编译时嵌入的，正常构建的程序中该资源一定存在。

**HTTP 外部图片的开关限制**（`core/artwork/reader_playlist.go:96-100`）：
```go
if parsed.Scheme == "http" || parsed.Scheme == "https" {
    if !conf.Server.EnableM3UExternalAlbumArt {
        return nil, "", nil  // 开关关闭时，直接返回"不适用"，继续下一个来源
    }
    return fromURL(ctx, parsed)
}
```

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

## 三、缓存键构成逻辑详解

### 3.1 缓存键设计目标

缓存键必须满足：
1. **实体唯一性**：不同实体的封面不能冲突
2. **内容新鲜性**：实体更新后缓存自动失效
3. **配置感知**：配置变更后缓存自动失效
4. **尺寸区分**：不同尺寸/质量的图片分开缓存

### 3.2 基础缓存键（cacheKey）

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

### 3.3 各类型缓存键扩展

不同类型在基础键上添加配置相关的哈希后缀，确保配置变更时缓存失效。

#### 3.3.1 专辑封面缓存键（`core/artwork/reader_album.go:64-76`）

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

#### 3.3.2 艺术家封面缓存键（`core/artwork/reader_artist.go:103-111`）

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

#### 3.3.3 媒体文件封面缓存键（`core/artwork/reader_mediafile.go:55-61`）

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

#### 3.3.4 光碟封面缓存键（`core/artwork/reader_disc.go:116-123`）

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

#### 3.3.5 播放列表封面缓存键

播放列表使用**基础键**，无额外配置字段。

**完整示例**：
```
pl-pl-101.1716000000000
```

---

#### 3.3.6 缩放后封面缓存键（`core/artwork/reader_resized.go:63-69`）

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

### 3.4 LastUpdate 时间戳来源

`lastUpdate` 字段取多个时间戳的最大值，确保任何相关内容变更都能使缓存失效。

| 类型 | 时间戳来源（取最大值） | 代码位置 |
|------|------------------------|----------|
| 专辑 | `album.UpdatedAt`、`album.ImportedAt`、所有关联文件夹的 `ImagesUpdatedAt` | `reader_album.go:57-60` |
| 艺术家 | 所有专辑的 `ImagesUpdatedAt`、`artist.UpdatedAt`、艺术家文件夹更新时间、图片文件夹文件修改时间 | `reader_artist.go:84-97` |
| 媒体文件 | `mediafile.UpdatedAt`、`album.UpdatedAt`、`imagesUpdatedAt` | `reader_mediafile.go:45-51` |
| 播放列表 | `playlist.UpdatedAt`、边车文件修改时间、外部图片文件修改时间 | `reader_playlist.go:44-59` |
| 光碟 | 同专辑 | `reader_disc.go:109-112` |

**设计意图**：
- 不仅考虑实体本身的更新时间
- 还考虑关联图片文件的更新时间（如用户替换了目录下的 cover.jpg）
- 确保用户替换图片后缓存立即失效

---

## 四、外部来源优先顺序的配置驱动机制

### 4.1 配置项说明

外部来源的优先顺序完全由 `conf.Server.Agents` 配置项驱动。

**默认值**：无显式默认值，空字符串（`conf/configuration.go` 中未设置默认值）

**配置示例**：
```ini
Agents = "lastfm,deezer,spotify"
```

### 4.2 Agent 列表构建逻辑

Agent 列表由 `getEnabledAgentNames()` 方法构建（`core/agents/agents.go:58-97`）：

```go
func (a *Agents) getEnabledAgentNames() []enabledAgent {
    // 1. 如果没有配置 Agent，仅使用本地 Agent
    if conf.Server.Agents == "" {
        return []enabledAgent{{name: LocalAgentName, isPlugin: false}}
    }

    // 2. 获取可用的插件 Agent 列表
    var availablePlugins []string
    if a.pluginLoader != nil {
        availablePlugins = a.pluginLoader.PluginNames("MetadataAgent")
    }

    // 3. 分割配置字符串
    configuredAgents := strings.Split(conf.Server.Agents, ",")

    // 4. 确保 LocalAgentName 始终在列表中（如果配置中没有）
    hasLocalAgent := slices.Contains(configuredAgents, LocalAgentName)
    if !hasLocalAgent {
        configuredAgents = append(configuredAgents, LocalAgentName)
    }

    // 5. 过滤出有效的 Agent（内置或插件）
    var validAgents []enabledAgent
    for _, name := range configuredAgents {
        isBuiltIn := Map[name] != nil
        isPlugin := slices.Contains(availablePlugins, name)
        if isBuiltIn {
            validAgents = append(validAgents, enabledAgent{name: name, isPlugin: false})
        } else if isPlugin {
            validAgents = append(validAgents, enabledAgent{name: name, isPlugin: true})
        } else {
            log.Debug("Unknown agent ignored", "name", name)
        }
    }
    return validAgents
}
```

### 4.3 配置驱动的优先级规则

**规则总结**：

| 配置场景 | 生效的 Agent 列表（按优先级） |
|----------|------------------------------|
| `Agents = ""`（空） | `[local]` |
| `Agents = "lastfm"` | `[lastfm, local]` |
| `Agents = "lastfm,deezer"` | `[lastfm, deezer, local]` |
| `Agents = "lastfm,deezer,local"` | `[lastfm, deezer, local]`（不重复） |
| `Agents = "unknown,lastfm"` | `[lastfm, local]`（unknown 被忽略） |

**关键特性**：
1. **配置顺序即优先级**：配置字符串中 Agent 出现的顺序决定了调用顺序
2. **LocalAgent 始终存在**：无论配置如何，`local` Agent 始终在列表末尾
3. **未知 Agent 静默忽略**：配置中不存在的 Agent 名会被忽略，仅记录 Debug 日志
4. **插件 Agent 支持**：除了内置 Agent，还支持 WASM 插件 Agent

### 4.4 Agent 调用流程

`callAgentMethod` 泛型函数按顺序调用 Agent（`core/agents/agents.go:322-345`）：

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
            return result, nil  // 第一个返回有效结果的 Agent 获胜
        }
    }
    return zero, ErrNotFound  // 全部失败
}
```

**示例流程**（配置 `Agents = "lastfm,deezer"`）：

```
调用 ArtistImage(artistID)
    │
    ├─ 调用 Last.fm Agent
    │       ├─ 成功返回 URL → 返回结果，结束
    │       └─ 失败（网络错误/无结果）→ 继续
    │
    ├─ 调用 Deezer Agent
    │       ├─ 成功返回 URL → 返回结果，结束
    │       └─ 失败 → 继续
    │
    └─ 调用 Local Agent
            ├─ 成功返回 URL → 返回结果，结束
            └─ 失败 → 返回 ErrNotFound
```

### 4.5 配置变更的缓存失效

Agent 配置变更会导致缓存自动失效，因为：
- 专辑缓存键包含 `md5(Agents + CoverArtPriority)`（`reader_album.go:70-75`）
- 艺术家缓存键包含 `md5(Agents)`（`reader_artist.go:105-109`）

这意味着修改 `Agents` 配置后，所有相关的缓存键都会变化，旧缓存自动失效。

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

#### 5.2.1 媒体文件封面降级路径

```
检查前提条件：CoverArtID().Kind == KindMediaFileArtwork
    │
    ├─ 条件满足
    │       ├─ fromTag(媒体文件)
    │       │     ├─ 成功 → 返回
    │       │     └─ 失败 → fromFFmpegTag(媒体文件)
    │       │                  ├─ 成功 → 返回
    │       │                  └─ 失败 → 继续回退
    │       └─ 继续回退
    │
    └─ 条件不满足 → 直接回退
           │
           ▼
    回退逻辑（始终执行）
        ├─ 多碟专辑 → 回退到光碟封面
        │       ├─ 光碟封面成功 → 返回
        │       └─ 光碟封面失败 → 光碟内部回退到专辑封面
        │                ├─ 专辑封面成功 → 返回
        │                └─ 专辑封面失败 → 返回 ErrUnavailable
        │
        └─ 单碟专辑 → 回退到专辑封面
                ├─ 专辑封面成功 → 返回
                └─ 专辑封面失败 → 返回 ErrUnavailable
```

**关键代码**（`core/artwork/reader_mediafile.go:68-80`）：
- `fromAlbum(ctx, a.a, a.mediafile.DiscCoverArtID())` — 回退到光碟封面
- `fromAlbum(ctx, a.a, a.mediafile.AlbumCoverArtID())` — 回退到专辑封面

这两个调用会递归进入对应类型的完整降级链。

---

#### 5.2.2 专辑封面降级路径

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
            ├─ 查 DB 缓存 → 有 URL → fromURL()
            │                       ├─ 200 OK → 返回
            │                       └─ 失败 → 继续
            │
            └─ 无缓存 → 同步调用 Agent（按配置顺序）
                      ├─ Agent 1 成功 → 保存到 DB → fromURL()
                      │                              ├─ 200 OK → 返回
                      │                              └─ 失败 → 继续
                      ├─ Agent 1 失败 → Agent 2
                      │                ├─ 成功 → 保存到 DB → fromURL()
                      │                │                              ├─ 200 OK → 返回
                      │                │                              └─ 失败 → 继续
                      │                └─ 失败 → 继续
                      └─ 全部 Agent 失败 → 返回 ErrUnavailable
                                                    │
                                                    ▼
                                          调用方 GetOrPlaceholder()
                                                    │
                                                    └─► 返回专辑占位图
```

---

#### 5.2.3 艺术家封面降级路径

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
            └─ 无缓存 → 同步调用 Agent（按配置顺序）
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

**外部来源缓存与落盘机制**

**两层缓存架构**：

Navidrome 的外部封面有两层独立的缓存机制：

| 缓存层级 | 存储位置 | 内容 | 失效机制 |
|----------|----------|------|----------|
| **文件缓存** | `cache/images/` 目录 | 实际的图片二进制文件 | LRU 策略 + 缓存键中的时间戳 |
| **元数据缓存** | 数据库 `artist`/`album` 表 | 图片 URL 字符串 | TTL 过期 + 后台刷新 |

---

**艺术家图片（ArtistImage）落盘时机**（`core/external/provider.go:373-402`）：

```
封面请求 → ArtistImage()
    │
    ├─ DB 中有 LargeImageUrl → 直接返回 URL（不落盘）
    │       │
    │       └─ 检查过期：time.Since(ExternalInfoUpdatedAt) > TTL
    │               ├─ 未过期 → 返回
    │               └─ 已过期 → 入队列后台刷新（异步落盘）
    │
    └─ DB 中无 LargeImageUrl → 同步调用 Agent 获取 URL
            │
            ├─ 获取成功 → 更新内存结构体 → 返回 URL（**不落盘**）
            └─ 获取失败 → 返回 ErrNotFound
```

**落盘时机**：仅在以下场景写入数据库：
1. 调用 `UpdateArtistInfo()` API 时，首次获取或后台刷新
2. 后台刷新队列处理时（`populateArtistInfo`）

---

**专辑图片（AlbumImage）落盘时机**（`core/external/provider.go:404-441`）：

```
封面请求 → AlbumImage()
    │
    └─ 每次都同步调用 Agent 获取（不检查 DB 缓存）
            │
            ├─ 获取成功 → 返回 URL（**永不落盘**）
            └─ 获取失败 → 返回 ErrNotFound
```

**落盘时机**：仅在调用 `UpdateAlbumInfo()` API 时写入数据库

---

**文件缓存（FileCache）落盘时机**（`utils/cache/file_caches.go:165-181`）：

```
GetCoverArt API → GetOrPlaceholder() → Get() → cache.Get()
    │
    ├─ 缓存命中 → 直接返回缓存文件
    │
    └─ 缓存未命中 → 调用 artworkReader.Reader() 获取图片
            │
            ├─ 获取成功 → 异步写入文件缓存（后台 goroutine）
            └─ 获取失败 → 返回错误
```

**文件缓存特性**：
- 缓存目录：`{CacheFolder}/images/`
- 最大条目：`consts.DefaultImageCacheMaxItems`
- 清理策略：LRU（最近最少使用）
- 无 TTL：依赖缓存键中的时间戳实现逻辑失效

---

**缓存时效默认值**（`consts/consts.go:61-62`）：
- `DevArtistInfoTimeToLive`：**24 小时**
- `DevAlbumInfoTimeToLive`：**7 天**

**刷新触发条件**：

| 触发场景 | 检查条件 | 行为 |
|----------|----------|------|
| `UpdateArtistInfo()` 调用 | `ExternalInfoUpdatedAt` 为零值 | 同步调用 `populateArtistInfo()` 落盘 |
| `UpdateArtistInfo()` 调用 | `time.Since(updatedAt) > TTL` | 入队列后台异步刷新 |
| `ArtistImage()` 调用 | DB 有 URL 且过期 | 入队列后台异步刷新 |
| `UpdateAlbumInfo()` 调用 | `ExternalInfoUpdatedAt` 为零值 | 同步调用 `populateAlbumInfo()` 落盘 |
| `UpdateAlbumInfo()` 调用 | `time.Since(updatedAt) > TTL` | 入队列后台异步刷新 |
| 后台刷新队列 | 每 5 秒处理一个任务 | 调用 `populateXxxInfo()` 落盘 |

**后台刷新队列参数**（`core/external/provider.go:28-30`）：
- `refreshDelay`：5 秒（处理间隔）
- `refreshTimeout`：15 秒（单个任务超时）
- `refreshQueueLength`：2000（队列最大长度）

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
    ├─ HTTP URL
    │     ├─ EnableM3UExternalAlbumArt = false → 跳过（不适用）
    │     └─ EnableM3UExternalAlbumArt = true → fromURL()
    │                                              ├─ 200 OK → 返回
    │                                              └─ 失败 → 继续
    ├─ 本地路径 → 读取文件
    │     ├─ 存在 → 返回
    │     └─ 失败 → 继续
    └─ 空 → 继续
        │
        ▼
自动生成 2x2 拼图
    ├─ 随机取 4 张专辑封面
    ├─ 解码成功 → 拼接 → 返回
    └─ 全部失败（0 张可用）→ 继续
        │
        ▼
占位图（嵌入式资源）
    ├─ 资源存在 → 返回占位图
    └─ 资源不存在 → 返回 ErrUnavailable
```

**自动拼图降级**（`core/artwork/reader_playlist.go:187-195`）：
- 取到 1 张 → 直接返回该图
- 取到 2 张 → [A, B, B, A] 对称排列
- 取到 3 张 → [A, B, C, A] 排列
- 取到 4 张 → [A, B, C, D] 正常排列
- 取到 0 张 → 失败，继续到占位图

**兜底修正**：
- `fromAlbumPlaceholder()` 忽略了 `Open()` 的错误（`r, _ := ...`）
- 如果嵌入式资源不存在，`r` 为 nil，`selectImageReader` 会认为失败
- 由于这是最后一个 sourceFunc，失败后返回 `ErrUnavailable`
- 正常构建的程序中，资源一定存在，因此**实际运行中不会失败**

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
| **光碟封面**有一个硬编码的最低优先级：回退到专辑封面 | `core/artwork/reader_disc.go:131-133` |
| **媒体文件封面**的前置条件：`HasCoverArt=true` 且 `EnableMediaFileCoverArt=true` | `model/mediafile.go:119` |
| **媒体文件封面**的回退：多碟→光碟封面，单碟→专辑封面 | `core/artwork/reader_mediafile.go:76-80` |
| **播放列表封面**的 HTTP 外部图片受 `EnableM3UExternalAlbumArt` 控制，默认关闭 | `core/artwork/reader_playlist.go:97`、`conf/configuration.go:751` |
| **播放列表占位图**理论上永不失败（编译时嵌入资源），但代码层面没有强保证 | `core/artwork/sources.go:184-189` |
| **内嵌封面**会尝试两次：先 taglib 后 ffmpeg | `core/artwork/reader_album.go:93-96` |
| **电台封面**仅支持用户上传，无其他来源 | `core/artwork/reader_radio.go:32-36` |
| **EnableMediaFileCoverArt** 默认值为 `true` | `conf/configuration.go:752` |

### 6.2 缓存键结论

| 结论 | 代码依据 |
|------|----------|
| 所有缓存键都包含时间戳，确保实体更新后缓存自动失效 | `core/artwork/image_cache.go:26` |
| 专辑缓存键包含 `CoverArtPriority` 和 `Agents` 的 MD5，配置变更自动失效 | `core/artwork/reader_album.go:64-76` |
| 媒体文件缓存键包含 `EnableMediaFileCoverArt`，开关变更自动失效 | `core/artwork/reader_mediafile.go:55-61` |
| 缩放后的图片有独立的缓存键，包含尺寸和质量参数 | `core/artwork/reader_resized.go:63-69` |
| `lastUpdate` 取多个时间戳的最大值，包括图片文件的更新时间 | `reader_album.go:57-60`、`reader_artist.go:84-97` |

### 6.3 外部来源配置驱动结论

| 结论 | 代码依据 |
|------|----------|
| Agent 优先级完全由 `conf.Server.Agents` 配置字符串的顺序决定 | `core/agents/agents.go:71` |
| `LocalAgent` 始终被追加到 Agent 列表末尾，即使配置中没有 | `core/agents/agents.go:74-77` |
| 配置为空时，仅使用 `LocalAgent` | `core/agents/agents.go:60-62` |
| 未知的 Agent 名称会被静默忽略，仅记录 Debug 日志 | `core/agents/agents.go:92-94` |
| Agent 配置变更会导致专辑和艺术家缓存自动失效 | `reader_album.go:70-75`、`reader_artist.go:105-109` |

### 6.4 降级路径结论

| 结论 | 代码依据 |
|------|----------|
| 所有来源失败最终返回 `ErrUnavailable`，由调用方决定是否返回占位图 | `core/artwork/sources.go:41` |
| `GetOrPlaceholder()` 会捕获 `ErrUnavailable` 并返回占位图，`Get()` 不会 | `core/artwork/artwork.go:44-58` |
| Agent 调用是静默降级的，单个失败仅记录 Trace 日志 | `core/agents/agents.go:335` |
| 艺术家图片有 DB 缓存，首次同步获取，后续从缓存读取 | `core/external/provider.go:373-402` |
| 媒体文件封面的回退是递归的，会进入光碟/专辑的完整降级链 | `core/artwork/reader_mediafile.go:76-80` |
| 图片缩放失败时会降级返回原图 | `core/artwork/reader_resized.go:90-96` |
| HTTP 请求超时时间为 5 秒 | `core/artwork/sources.go:213` |

### 6.5 设计权衡结论

| 决策 | 优点 | 缺点 |
|------|------|------|
| sourceFunc 返回 nil 表示失败 | 接口简洁，易于扩展 | 无法区分"不适用"和"失败" |
| Agent 失败静默降级 | 单个服务不可用不影响整体功能 | 问题排查困难，需开启 Trace 日志 |
| 内嵌封面两次尝试（taglib + ffmpeg） | 兼容性好 | 开销较大，某些场景下重复工作 |
| 配置变更通过哈希使缓存失效 | 无需手动清理缓存 | 配置微小变化也会导致全量缓存失效 |
| 播放列表占位图忽略 Open 错误 | 代码简洁 | 如果构建出错可能返回 ErrUnavailable |
| 媒体文件封面递归回退 | 逻辑统一，避免重复代码 | 调用链较深，调试困难 |

---

## 七、关键代码索引

| 功能 | 文件位置 | 行号 |
|------|----------|------|
| 核心调度器 selectImageReader | `core/artwork/sources.go` | 27-42 |
| sourceFunc 类型定义 | `core/artwork/sources.go` | 44 |
| 媒体文件封面 Reader | `core/artwork/reader_mediafile.go` | 66-81 |
| 媒体文件 CoverArtID 判断 | `model/mediafile.go` | 117-124 |
| 专辑优先级构建 | `core/artwork/reader_album.go` | 86-104 |
| 艺术家优先级构建 | `core/artwork/reader_artist.go` | 127-145 |
| 光碟优先级构建（含专辑回退） | `core/artwork/reader_disc.go` | 129-158 |
| 播放列表固定优先级链 | `core/artwork/reader_playlist.go` | 68-76 |
| 播放列表 HTTP 图片开关 | `core/artwork/reader_playlist.go` | 96-100 |
| 播放列表自动拼图失败处理 | `core/artwork/reader_playlist.go` | 187-195 |
| 占位图实现 | `core/artwork/sources.go` | 184-189 |
| 基础缓存键定义 | `core/artwork/image_cache.go` | 16-28 |
| 专辑缓存键 | `core/artwork/reader_album.go` | 64-76 |
| 艺术家缓存键 | `core/artwork/reader_artist.go` | 103-111 |
| 媒体文件缓存键 | `core/artwork/reader_mediafile.go` | 55-61 |
| 缩放缓存键 | `core/artwork/reader_resized.go` | 63-69 |
| GetOrPlaceholder 占位图逻辑 | `core/artwork/artwork.go` | 44-58 |
| 外部图片 HTTP 客户端 | `core/artwork/sources.go` | 212-225 |
| Agent 列表构建 | `core/agents/agents.go` | 58-97 |
| Agent 聚合器核心循环 | `core/agents/agents.go` | 322-345 |
| 艺术家图片获取（不落盘） | `core/external/provider.go` | 373-402 |
| 专辑图片获取（不缓存） | `core/external/provider.go` | 404-441 |
| 艺术家信息落盘（含图片） | `core/external/provider.go` | 247-281 |
| 专辑信息落盘（含图片） | `core/external/provider.go` | 143-189 |
| 缓存时效默认值 | `consts/consts.go` | 61-62 |
| 默认配置值 | `conf/configuration.go` | 750-772、870-871 |
