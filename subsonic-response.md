# Subsonic 协议响应对象构造分析

本文档深入分析 Navidrome 中 Subsonic 协议响应对象的构造机制，涵盖三个核心方面：领域模型映射、客户端版本兼容、错误与分页适配。

---

## 一、领域模型到 Subsonic 响应字段的映射

### 核心设计思想

Navidrome 采用**分层映射**策略：领域模型（`model.Artist`、`model.Album`、`model.MediaFile`）通过一系列 `toXxx` 和 `buildXxx` 函数转换为 Subsonic 协议定义的响应结构体。所有响应结构体定义在 `server/subsonic/responses/responses.go` 中，映射逻辑集中在 `server/subsonic/helpers.go`。

### 1.1 艺人（Artist）映射

#### 映射函数与目标结构

| 函数 | 目标结构 | 使用场景 |
|------|----------|----------|
| `toArtist()` | `responses.Artist` | 传统 API（如 `getIndexes`、`getStarred`） |
| `toArtistID3()` | `responses.ArtistID3` | ID3 风格 API（如 `getArtists`、`getArtist`、`getStarred2`） |

#### 字段映射示例（`toArtistID3`）

```go
// server/subsonic/helpers.go:115-132
func toArtistID3(r *http.Request, a model.Artist) responses.ArtistID3 {
    artist := responses.ArtistID3{
        Id:             a.ID,              // 直接映射
        Name:           a.Name,            // 直接映射
        AlbumCount:     getArtistAlbumCount(&a),  // 统计字段计算
        CoverArt:       a.CoverArtID().String(),   // 封面ID序列化
        ArtistImageUrl: publicurl.ImageURL(r, a.CoverArtID(), 600), // 图片URL生成
        UserRating:     int32(a.Rating),
    }
    // 可选字段条件映射
    if conf.Server.Subsonic.EnableAverageRating {
        artist.AverageRating = a.AverageRating
    }
    if a.Starred {
        artist.Starred = a.StarredAt  // 时间指针，nil 时 XML/JSON 省略
    }
    artist.OpenSubsonicArtistID3 = toOSArtistID3(r.Context(), a)
    return artist
}
```

#### 关键映射模式

1. **直接映射**：`Id`、`Name` 等字段一一对应
2. **计算字段**：`AlbumCount` 通过 `getArtistAlbumCount()` 计算，支持不同统计策略
3. **序列化转换**：`CoverArtID()` 方法将内部封面标识转换为字符串形式
4. **URL 生成**：`publicurl.ImageURL()` 生成可访问的图片 URL
5. **可选字段**：使用指针类型（`*time.Time`），零值自动省略
6. **配置驱动**：`EnableAverageRating` 控制平均评分字段是否返回

### 1.2 专辑（Album）映射

#### 映射函数与目标结构

| 函数 | 目标结构 | 使用场景 |
|------|----------|----------|
| `childFromAlbum()` | `responses.Child`（IsDir=true） | 传统 API（如 `getAlbumList`、`getMusicDirectory`） |
| `buildAlbumID3()` | `responses.AlbumID3` | ID3 风格 API（如 `getAlbumList2`、`getAlbum`） |

#### 字段映射示例（`childFromAlbum`）

```go
// server/subsonic/helpers.go:337-364
func childFromAlbum(ctx context.Context, al model.Album) responses.Child {
    child := responses.Child{}
    child.Id = al.ID
    child.IsDir = true  // 专辑作为目录
    child.Title = al.FullName()  // 方法调用获取完整名称
    child.Album = fullName
    child.Artist = al.AlbumArtist
    child.Year = int32(cmp.Or(al.MaxOriginalYear, al.MaxYear))  // 优先级选择
    child.Genre = al.Genre
    child.CoverArt = al.CoverArtID().String()
    child.Created = P(albumCreatedAt(al))  // 多字段 fallback 策略
    child.Duration = int32(al.Duration)
    child.SongCount = int32(al.SongCount)
    child.OpenSubsonicChild = osChildFromAlbum(ctx, al)
    return child
}
```

#### 专辑创建时间 fallback 策略

```go
// server/subsonic/helpers.go:327-335
func albumCreatedAt(al model.Album) time.Time {
    if !al.CreatedAt.IsZero() {
        return al.CreatedAt    // 优先使用创建时间
    }
    if !al.UpdatedAt.IsZero() {
        return al.UpdatedAt    // 其次使用更新时间
    }
    return al.ImportedAt       // 最后使用导入时间
}
```

### 1.3 曲目（MediaFile）映射

#### 映射函数与目标结构

| 函数 | 目标结构 | 使用场景 |
|------|----------|----------|
| `childFromMediaFile()` | `responses.Child`（IsDir=false） | 所有曲目标相关 API |

#### 字段映射示例（简化版）

```go
// server/subsonic/helpers.go:190-241
func childFromMediaFile(ctx context.Context, mf model.MediaFile) responses.Child {
    child := responses.Child{}
    child.Id = mf.ID
    child.Title = mf.FullTitle()
    child.IsDir = false
    child.Parent = mf.AlbumID
    child.Album = mf.FullAlbumName()
    child.Year = int32(mf.Year)
    child.Artist = mf.Artist
    child.Genre = mf.Genre
    child.Track = int32(mf.TrackNumber)
    child.Duration = int32(mf.Duration)
    child.Size = mf.Size
    child.Suffix = mf.Suffix
    child.BitRate = int32(mf.BitRate)
    child.CoverArt = mf.CoverArtID().String()
    child.ContentType = mf.ContentType()
    
    // 路径处理：真实路径 or 伪造路径
    if ok && player.ReportRealPath {
        child.Path = mf.AbsolutePath()
    } else {
        child.Path = fakePath(mf)  // 构造虚拟路径保护隐私
    }
    
    // 转码信息
    format, _ := getTranscoding(ctx)
    if mf.Suffix != "" && format != "" && mf.Suffix != format {
        child.TranscodedSuffix = format
        child.TranscodedContentType = mime.TypeByExtension("." + format)
    }
    
    child.OpenSubsonicChild = osChildFromMediaFile(ctx, mf)
    return child
}
```

#### 伪造路径生成（隐私保护）

```go
// server/subsonic/helpers.go:305-317
func fakePath(mf model.MediaFile) string {
    builder := strings.Builder{}
    builder.WriteString(fmt.Sprintf("%s/%s/", sanitizeSlashes(mf.AlbumArtist), sanitizeSlashes(mf.FullAlbumName())))
    if mf.DiscNumber != 0 {
        builder.WriteString(fmt.Sprintf("%02d-", mf.DiscNumber))
    }
    if mf.TrackNumber != 0 {
        builder.WriteString(fmt.Sprintf("%02d - ", mf.TrackNumber))
    }
    builder.WriteString(fmt.Sprintf("%s.%s", sanitizeSlashes(mf.FullTitle()), mf.Suffix))
    return builder.String()
}
```

### 1.4 组合响应结构

Subsonic 协议中许多响应是组合结构，通过嵌套和集合映射实现：

```go
// 获取单个艺人（含专辑列表）
func (api *Router) buildArtist(r *http.Request, artist *model.Artist) (*responses.ArtistWithAlbumsID3, error) {
    a := &responses.ArtistWithAlbumsID3{}
    a.ArtistID3 = toArtistID3(r, *artist)                    // 艺人基础信息
    albums, _ := api.ds.Album(ctx).GetAll(...)
    a.Album = slice.MapWithArg(albums, ctx, buildAlbumID3)    // 专辑列表映射
    return a, nil
}

// 获取单个专辑（含曲目列表）
func (api *Router) buildAlbum(ctx context.Context, album *model.Album, mfs model.MediaFiles) *responses.AlbumWithSongsID3 {
    dir := &responses.AlbumWithSongsID3{}
    dir.AlbumID3 = buildAlbumID3(ctx, *album)                 // 专辑基础信息
    dir.Song = slice.MapWithArg(mfs, ctx, childFromMediaFile) // 曲目列表映射
    return dir
}
```

---

## 二、不同客户端版本字段的兼容

### 2.1 版本兼容架构

Navidrome 实现了**多维度的客户端兼容策略**，通过配置、客户端识别、条件返回三层机制实现：

```
配置层 (conf.Server.Subsonic)
    ├─ LegacyClients    // 旧客户端，不返回 OpenSubsonic 扩展
    ├─ MinimalClients   // 极简客户端，只返回核心字段
    └─ EnableAverageRating  // 功能开关
客户端识别层 (request.PlayerFrom(ctx))
    └─ Client 字段标识客户端类型
条件返回层 (映射函数中的 if 判断)
    ├─ 早期返回（极简客户端）
    ├─ nil 返回（旧客户端，不添加扩展字段）
    └─ 完整返回（现代客户端）
```

### 2.2 极简客户端兼容（MinimalClients）

对于配置在 `MinimalClients` 列表中的客户端，`childFromMediaFile` 提前返回，只包含最基本的字段：

```go
// server/subsonic/helpers.go:190-199
func childFromMediaFile(ctx context.Context, mf model.MediaFile) responses.Child {
    child := responses.Child{}
    child.Id = mf.ID
    child.Title = mf.FullTitle()
    child.IsDir = false

    player, ok := request.PlayerFrom(ctx)
    if ok && isClientInList(conf.Server.Subsonic.MinimalClients, player.Client) {
        return child  // 直接返回，跳过后续所有字段
    }
    // ... 其余字段仅对非极简客户端返回
}
```

**返回字段差异对比**：

| 字段 | 极简客户端 | 普通客户端 |
|------|-----------|-----------|
| Id | ✓ | ✓ |
| Title | ✓ | ✓ |
| IsDir | ✓ | ✓ |
| Parent | ✗ | ✓ |
| Album | ✗ | ✓ |
| Artist | ✗ | ✓ |
| ... | ... | ... |
| OpenSubsonicChild | ✗ | ✓ |

### 2.3 旧客户端兼容（LegacyClients）

对于配置在 `LegacyClients` 列表中的客户端，**不返回 OpenSubsonic 扩展字段**。这通过将扩展结构体指针设为 `nil` 实现：

```go
// server/subsonic/helpers.go:134-145
func toOSArtistID3(ctx context.Context, a model.Artist) *responses.OpenSubsonicArtistID3 {
    player, _ := request.PlayerFrom(ctx)
    if strings.Contains(conf.Server.Subsonic.LegacyClients, player.Client) {
        return nil  // 返回 nil，XML/JSON 序列化时自动省略
    }
    // 构建并返回 OpenSubsonic 扩展字段
    return &artist
}
```

#### OpenSubsonic 扩展字段设计

OpenSubsonic 扩展通过**结构体嵌入 + `omitempty` 标签**实现条件序列化：

```go
// server/subsonic/responses/responses.go:229-239
type ArtistID3 struct {
    Id                     string     `xml:"id,attr" json:"id"`
    Name                   string     `xml:"name,attr" json:"name"`
    // ... 基础字段
    *OpenSubsonicArtistID3 `xml:",omitempty" json:",omitempty"`  // 匿名嵌入，指针类型
}

type OpenSubsonicArtistID3 struct {
    MusicBrainzId string        `xml:"musicBrainzId,attr,omitempty" json:"musicBrainzId"`
    SortName      string        `xml:"sortName,attr,omitempty" json:"sortName"`
    Roles         Array[string] `xml:"roles,omitempty" json:"roles"`
}
```

**序列化规则**：
- 当 `OpenSubsonicArtistID3` 指针为 `nil` 时，整个嵌入部分在 XML/JSON 中被省略
- 当指针非 nil 时，其内部字段按各自的 `omitempty` 规则序列化

### 2.4 API 版本声明

Navidrome 声明支持 Subsonic API 版本 `1.16.1`，同时标识为 OpenSubsonic 兼容服务器：

```go
// server/subsonic/api.go:31
const Version = "1.16.1"

// server/subsonic/helpers.go:27-35
func newResponse() *responses.Subsonic {
    return &responses.Subsonic{
        Status:        responses.StatusOK,
        Version:       Version,           // 协议版本
        Type:          consts.AppName,    // 服务器类型
        ServerVersion: consts.Version,    // Navidrome 自身版本
        OpenSubsonic:  true,              // OpenSubsonic 支持标识
    }
}
```

响应根结构（`responses.Subsonic`）包含完整的版本元数据：

```go
// server/subsonic/responses/responses.go:9-66
type Subsonic struct {
    XMLName       xml.Name `xml:"http://subsonic.org/restapi subsonic-response" json:"-"`
    Status        string   `xml:"status,attr" json:"status"`
    Version       string   `xml:"version,attr" json:"version"`
    Type          string   `xml:"type,attr" json:"type"`
    ServerVersion string   `xml:"serverVersion,attr" json:"serverVersion"`
    OpenSubsonic  bool     `xml:"openSubsonic,attr,omitempty" json:"openSubsonic,omitempty"`
    // ... 各种响应字段
}
```

---

## 三、错误与分页的协议适配

### 3.1 错误处理机制

#### 错误码定义

Subsonic 协议定义了标准错误码，在 `server/subsonic/responses/errors.go` 中集中管理：

```go
// server/subsonic/responses/errors.go:3-23
const (
    ErrorGeneric            int32 = 0
    ErrorMissingParameter   int32 = 10
    ErrorClientTooOld       int32 = 20
    ErrorServerTooOld       int32 = 30
    ErrorAuthenticationFail int32 = 40
    ErrorAuthorizationFail  int32 = 50
    ErrorTrialExpired       int32 = 60
    ErrorDataNotFound       int32 = 70
)

var errors = map[int32]string{
    ErrorGeneric:            "A generic error",
    ErrorMissingParameter:   "Required parameter is missing",
    // ...
}
```

#### 自定义错误类型

Navidrome 实现了 `subError` 类型，携带错误码和自定义消息：

```go
// server/subsonic/helpers.go:37-64
type subError struct {
    code     int32
    messages []any
}

func newError(code int32, message ...any) error {
    return subError{code: code, messages: message}
}

func (e subError) Error() string {
    if len(e.messages) == 0 {
        return responses.ErrorMsg(e.code)  // 使用默认消息
    }
    return fmt.Sprintf(e.messages[0].(string), e.messages[1:]...)  // 格式化自定义消息
}
```

#### 错误映射（内部错误 → Subsonic 错误）

`mapToSubsonicError` 函数将 Navidrome 内部错误转换为 Subsonic 协议错误：

```go
// server/subsonic/api.go:293-310
func mapToSubsonicError(err error) subError {
    switch {
    case errors.Is(err, errSubsonic):      // 已经是 Subsonic 错误，直接返回
    case errors.Is(err, req.ErrMissingParam):
        err = newError(responses.ErrorMissingParameter, err.Error())
    case errors.Is(err, req.ErrInvalidParam):
        err = newError(responses.ErrorGeneric, err.Error())
    case errors.Is(err, model.ErrNotFound):
        err = newError(responses.ErrorDataNotFound, "data not found")
    case errors.Is(err, model.ErrNotAuthorized):
        err = newError(responses.ErrorAuthorizationFail)
    default:
        err = newError(responses.ErrorGeneric, fmt.Sprintf("Internal Server Error: %s", err))
    }
    var subErr subError
    errors.As(err, &subErr)
    return subErr
}
```

#### 错误响应发送

`sendError` 构造标准的 Subsonic 错误响应：

```go
// server/subsonic/api.go:312-319
func sendError(w http.ResponseWriter, r *http.Request, err error) {
    subErr := mapToSubsonicError(err)
    response := newResponse()
    response.Status = responses.StatusFailed  // 状态标记为 failed
    response.Error = &responses.Error{
        Code:    subErr.code,                 // 错误码
        Message: subErr.Error(),              // 错误消息
    }
    sendResponse(w, r, response)
}
```

### 3.2 分页适配

#### 分页参数解析

Subsonic 协议使用 `offset` 和 `size` 参数进行分页，Navidrome 在业务函数中解析并应用：

```go
// server/subsonic/album_lists.go:19-88
func (api *Router) getAlbumList(r *http.Request) (model.Albums, int64, error) {
    // ... 类型过滤逻辑
    
    opts.Offset = p.IntOr("offset", 0)                  // 偏移量，默认 0
    opts.Max = min(p.IntOr("size", 10), 500)            // 每页大小，默认 10，最大 500
    
    albums, err := api.ds.Album(r.Context()).GetAll(opts)  // 分页查询
    count, err := api.ds.Album(r.Context()).CountAll(opts) // 总数统计
    
    return albums, count, nil
}
```

#### 总数响应头

分页总数通过自定义 HTTP 头 `x-total-count` 返回给客户端：

```go
// server/subsonic/album_lists.go:90-118
func (api *Router) GetAlbumList(w http.ResponseWriter, r *http.Request) (*responses.Subsonic, error) {
    albums, count, err := api.getAlbumList(r)
    if err != nil {
        return nil, err
    }
    
    w.Header().Set("x-total-count", strconv.Itoa(int(count)))  // 总数写入响应头
    
    response := newResponse()
    response.AlbumList = &responses.AlbumList{
        Album: slice.MapWithArg(albums, r.Context(), childFromAlbum),
    }
    return response, nil
}
```

#### 分页安全限制

- `size` 参数有上限（500），防止恶意请求导致服务器压力过大
- `offset` 无显式上限，但数据库查询本身会处理边界情况
- 不同接口可能有不同的 `size` 默认值和上限

### 3.3 响应序列化格式

Subsonic 协议支持三种响应格式，通过 `f` 参数控制：

```go
// server/subsonic/api.go:321-379
func sendResponse(w http.ResponseWriter, r *http.Request, payload *responses.Subsonic) {
    p := req.Params(r)
    f, _ := p.String("f")
    
    switch f {
    case "json":
        w.Header().Set("Content-Type", "application/json")
        wrapper := &responses.JsonWrapper{Subsonic: *payload}
        response, err = json.Marshal(wrapper)  // JSON 格式需要外层包装
    
    case "jsonp":
        callback, _ := p.String("callback")
        // ... 回调名称验证
        wrapper := &responses.JsonWrapper{Subsonic: *payload}
        response, err = json.Marshal(wrapper)
        response = fmt.Appendf(nil, "%s(%s)", callback, response)  // JSONP 包装
    
    default:  // xml
        w.Header().Set("Content-Type", "application/xml")
        response, err = xml.Marshal(payload)  // XML 直接序列化
    }
    
    w.Write(response)
}
```

#### JSON 包装结构

JSON 格式要求响应外层包裹 `subsonic-response` 键：

```go
// server/subsonic/responses/responses.go:73-75
type JsonWrapper struct {
    Subsonic Subsonic `json:"subsonic-response"`
}
```

---

## 总结

### 设计优点

1. **关注点分离**：响应结构定义、映射逻辑、错误处理、序列化各自独立
2. **可扩展性**：OpenSubsonic 扩展通过指针嵌入实现优雅的条件返回
3. **兼容性强**：通过配置驱动的客户端白名单机制，灵活支持新旧客户端
4. **隐私保护**：伪造路径机制避免暴露服务器真实文件系统结构
5. **类型安全**：所有映射都是编译时类型检查的 Go 代码

### 核心代码位置

| 功能 | 文件位置 |
|------|---------|
| 响应结构定义 | `server/subsonic/responses/responses.go` |
| 错误码定义 | `server/subsonic/responses/errors.go` |
| 映射辅助函数 | `server/subsonic/helpers.go` |
| API 路由与响应发送 | `server/subsonic/api.go` |
| 专辑列表分页 | `server/subsonic/album_lists.go` |
| 浏览接口实现 | `server/subsonic/browsing.go` |
