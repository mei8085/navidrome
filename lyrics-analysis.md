# Navidrome 歌词系统代码分析报告

## 1. 同步与非同步歌词格式的区别

### 1.1 格式识别机制

**文件位置**: `model/lyrics.go:29-36, 52`

Navidrome 通过正则表达式自动检测歌词类型，无需用户手动指定：

```go
// 时间戳正则: 支持 [mm:ss.mm], [hh:*], [*.mmm] 格式
const timeRegexString = `\[([0-9]{1,2}:)?([0-9]{1,2}):([0-9]{1,2})(.[0-9]{1,3})?\]`

var (
    syncRegex  = regexp.MustCompile(`(^|\n)\s*` + timeRegexString)
    timeRegex  = regexp.MustCompile(timeRegexString)
    lrcIdRegex = regexp.MustCompile(`\[(ar|ti|offset|lang):([^]]+)]`)
)

// 检测逻辑
synced := syncRegex.MatchString(text)
```

**识别规则**:
- 时间戳必须出现在行首（允许前面有空白字符）或文件开头
- 行中间的时间戳不会触发同步歌词识别（见 `model/lyrics.go:98-105`）

### 1.2 格式对比表

| 特性 | 同步歌词 (Synced) | 非同步歌词 (Unsynced) |
|------|------------------|----------------------|
| 时间戳标记 | 每行开头有 `[mm:ss.xx]` 格式 | 无时间戳 |
| `Line.Start` 字段 | 非 nil，存储毫秒级时间戳 | nil |
| `Synced` 布尔字段 | `true` | `false` |
| LRC 元数据标签 | 支持 `[ar:]`, `[ti:]`, `[offset:]`, `[lang:]` | 不支持 |
| 多行文本处理 | 同一时间戳可包含多行（用换行分隔） | 每行独立为一个 Line |

### 1.3 同步歌词特殊处理逻辑

**文件位置**: `model/lyrics.go:93-170`

#### 多时间戳重复行
一行中多个时间戳会被展开为多行，值相同：
```
[00:00.00][00:10.00]Repeated
```
解析为两行，时间戳分别为 0 和 10000ms，值都是 "Repeated"。

#### 时间戳自动排序
当存在多时间戳行时，最终结果会按时间戳排序：
```go
if repeated {
    slices.SortFunc(structuredLines, func(a, b Line) int {
        return cmp.Compare(*a.Start, *b.Start)
    })
}
```

#### 时间格式支持
- `[00:00.001]` → 1ms
- `[00:00.01]` → 10ms
- `[00:00.1]` → 100ms
- `[01:00:00]` → 3600000ms（支持小时格式）

#### LRC 元数据标签映射
- `[ar:Artist Name]` → `DisplayArtist` 字段
- `[ti:Track Title]` → `DisplayTitle` 字段
- `[offset:1551]` → `Offset` 字段（毫秒）
- `[lang:eng]` → `Lang` 字段

---

## 2. 元数据字段从读取到返回的关联路径

### 2.1 数据模型定义

**文件位置**: `model/lyrics.go:14-26`

```go
type Line struct {
    Start *int64 `structs:"start,omitempty" json:"start,omitempty"`
    Value string `structs:"value"           json:"value"`
}

type Lyrics struct {
    DisplayArtist string `structs:"displayArtist,omitempty" json:"displayArtist,omitempty"`
    DisplayTitle  string `structs:"displayTitle,omitempty"  json:"displayTitle,omitempty"`
    Lang          string `structs:"lang"                    json:"lang"`
    Line          []Line `structs:"line"                    json:"line"`
    Offset        *int64 `structs:"offset,omitempty"        json:"offset,omitempty"`
    Synced        bool   `structs:"synced"                  json:"synced"`
}

type LyricList []Lyrics
```

### 2.2 存储层：MediaFile 中的歌词

**文件位置**: `model/mediafile.go:73, 139-146`

```go
type MediaFile struct {
    // ... 其他字段
    Lyrics string `structs:"lyrics" json:"lyrics"`  // JSON 序列化后的 LyricList
}

func (mf MediaFile) StructuredLyrics() (LyricList, error) {
    lyrics := LyricList{}
    err := json.Unmarshal([]byte(mf.Lyrics), &lyrics)
    return lyrics, err
}
```

**关键点**:
- 内嵌歌词以 JSON 字符串形式存储在 `MediaFile.Lyrics` 字段
- 扫描时已完成解析和序列化，请求时只需反序列化

### 2.3 服务层：来源获取与解析

**文件位置**: `core/lyrics/sources.go`

#### 内嵌歌词来源
```go
func fromEmbedded(ctx context.Context, mf *model.MediaFile) (model.LyricList, error) {
    if mf.Lyrics != "" {
        return mf.StructuredLyrics()
    }
    return nil, nil
}
```

#### 外部文件来源
```go
func fromExternalFile(ctx context.Context, mf *model.MediaFile, suffix string) (model.LyricList, error) {
    basePath := mf.AbsolutePath()
    ext := path.Ext(basePath)
    externalLyric := basePath[0:len(basePath)-len(ext)] + suffix
    
    contents, err := ioutils.UTF8ReadFile(externalLyric)
    lyrics, err := model.ToLyrics("xxx", string(contents))
    return model.LyricList{*lyrics}, nil
}
```

**文件查找规则**: 与音频文件同名，后缀替换为配置的后缀（如 `.lrc` 或 `.txt`）

#### 插件来源
**文件位置**: `plugins/lyrics_adapter.go:36-62`

```go
func (l *LyricsPlugin) GetLyrics(ctx context.Context, mf *model.MediaFile) (model.LyricList, error) {
    resp, err := callPluginFunction[...](ctx, l.plugin, FuncLyricsGetLyrics, req)
    
    for _, lt := range resp.Lyrics {
        lang := lt.Lang
        if lang == "" {
            lang = "xxx"
        }
        parsed, err := model.ToLyrics(lang, lt.Text)
        if parsed != nil && !parsed.IsEmpty() {
            result = append(result, *parsed)
        }
    }
    return result, nil
}
```

### 2.4 API 层：响应构建与元数据补充

**文件位置**: `server/subsonic/helpers.go:505-538`

```go
func buildStructuredLyric(mf *model.MediaFile, lyrics model.Lyrics) responses.StructuredLyric {
    structured := responses.StructuredLyric{
        DisplayArtist: lyrics.DisplayArtist,
        DisplayTitle:  lyrics.DisplayTitle,
        Lang:          lyrics.Lang,
        Offset:        lyrics.Offset,
        Synced:        lyrics.Synced,
        Line:          make([]responses.Line, len(lyrics.Line)),
    }
    
    // 关键：元数据降级填充
    if structured.DisplayArtist == "" {
        structured.DisplayArtist = mf.Artist
    }
    if structured.DisplayTitle == "" {
        structured.DisplayTitle = mf.Title
    }
    
    return structured
}
```

**元数据填充逻辑**:
- 如果歌词本身没有 `DisplayArtist`，使用 `MediaFile.Artist`
- 如果歌词本身没有 `DisplayTitle`，使用 `MediaFile.Title`
- 这保证了即使歌词文件中没有元数据标签，API 响应也能返回正确的艺术家和标题

### 2.5 完整链路图

```
MediaFile 存储层
    ↓ (Lyrics: JSON string)
model/mediafile.go: StructuredLyrics()
    ↓ (LyricList)
core/lyrics/lyrics.go: GetLyrics()
    ├─→ fromEmbedded() → 直接反序列化
    ├─→ fromExternalFile() → 读取文件 → model.ToLyrics()
    └─→ fromPlugin() → 插件调用 → model.ToLyrics()
    ↓ (LyricList)
server/subsonic/helpers.go: buildLyricsList()
    ├─→ buildStructuredLyric()
    │   ├─→ 复制 Lyrics 字段
    │   └─→ 补充 DisplayArtist/DisplayTitle（如果为空）
    ↓ (responses.LyricsList)
API 响应
```

---

## 3. 多来源冲突时的优先级和中断条件

### 3.1 优先级配置

**文件位置**: `conf/configuration.go:81`

```go
type Server struct {
    // ...
    LyricsPriority string  // 示例值: "embedded,.lrc,.txt,my-plugin"
}
```

### 3.2 核心调度逻辑

**文件位置**: `core/lyrics/lyrics.go:34-58`

```go
func (l *lyricsService) GetLyrics(ctx context.Context, mf *model.MediaFile) (model.LyricList, error) {
    var lyricsList model.LyricList
    var err error

    for pattern := range strings.SplitSeq(conf.Server.LyricsPriority, ",") {
        pattern = strings.TrimSpace(pattern)
        switch {
        case strings.EqualFold(pattern, "embedded"):
            lyricsList, err = fromEmbedded(ctx, mf)
        case strings.HasPrefix(pattern, "."):
            lyricsList, err = fromExternalFile(ctx, mf, strings.ToLower(pattern))
        default:
            lyricsList, err = l.fromPlugin(ctx, mf, pattern)
        }

        if err != nil {
            log.Error(ctx, "error getting lyrics", "source", pattern, err)
        }

        if len(lyricsList) > 0 {
            return lyricsList, nil  // 中断条件：第一个返回非空结果的来源
        }
    }

    return nil, nil
}
```

### 3.3 中断条件详解

**立即中断返回**: 当某个来源返回的 `lyricsList` 长度大于 0 时，立即返回该结果，后续来源不再尝试。

**不中断的情况**:
1. 来源返回空列表（`len(lyricsList) == 0`）
2. 来源返回错误（仅记录日志，继续下一个）
3. 插件不存在或加载失败（静默跳过）
4. 歌词解析失败（返回空，继续下一个）

### 3.4 错误处理策略

**文件位置**: `core/lyrics/lyrics.go:49-51`

```go
if err != nil {
    log.Error(ctx, "error getting lyrics", "source", pattern, err)
}
```

**容错设计**:
- 单个来源出错不会导致整个请求失败
- 错误仅记录日志，流程继续
- 所有来源都失败或为空时，返回 `nil, nil`

### 3.5 测试用例验证的行为

**文件位置**: `core/lyrics/lyrics_test.go:75-193`

| 配置 | 场景 | 结果 |
|------|------|------|
| `embedded,.lrc,.txt` | 内嵌有内容 | 返回内嵌，忽略 .lrc 和 .txt |
| `.lrc,embedded,.txt` | .lrc 文件存在 | 返回 .lrc，忽略内嵌 |
| `.txt,.lrc,embedded` | .txt 文件存在 | 返回 .txt，忽略 .lrc 和内嵌 |
| `test-lyrics-plugin,embedded` | 插件返回错误 | 记录错误，返回内嵌 |
| `nonexistent-plugin,embedded` | 插件不存在 | 跳过插件，返回内嵌 |

### 3.6 关键设计决策

**1. 不合并多来源结果**
- 简化实现，避免复杂的冲突解决逻辑
- 用户通过配置优先级来控制偏好

**2. 非空即返回**
- 不验证歌词内容质量，只要有内容就返回
- 依赖用户合理配置优先级顺序

**3. 错误隔离**
- 单个来源问题不影响整体可用性
- 提供最大的容错性

---

## 4. 同名歌曲多记录选取规则

### 4.1 过滤器实现

**文件位置**: `server/subsonic/filter/filters.go:111-124`

```go
func SongsByArtistTitleWithLyricsFirst(artist, title string) Options {
    return addDefaultFilters(Options{
        Sort:  "lyrics, updated_at",
        Order: "desc",
        Max:   1,
        Filters: And{
            Eq{"title": title},
            Or{
                persistence.Exists("json_tree(participants, '$.albumartist')", Eq{"value": artist}),
                persistence.Exists("json_tree(participants, '$.artist')", Eq{"value": artist}),
            },
        },
    })
}
```

### 4.2 选取规则详解

#### 过滤条件
1. **标题精确匹配**: `Eq{"title": title}` - 歌曲标题必须完全匹配
2. **艺术家匹配**: 通过 `OR` 条件匹配两种参与者类型
   - `albumartist`: 专辑艺术家
   - `artist`: 歌曲艺术家
   - 使用 SQLite 的 `json_tree` 函数解析 JSON 格式的参与者字段

#### 排序优先级（从高到低）
**文件位置**: `persistence/mediafile_repository.go:83-93`

`mediaFileRepository` 的 `sortMappings` 中 **没有** 对 `lyrics` 字段的特殊映射，因此 `Sort: "lyrics, updated_at"` 直接生成 SQL：

```sql
ORDER BY lyrics DESC, updated_at DESC
```

**排序语义修正**:
- **不是** 通过 `(lyrics != '[]')` 布尔表达式排序
- 而是直接对 `lyrics` 列进行**字符串字典序降序**排序
- 利用 SQLite 字符串比较特性：空歌词 `"[]"` < 有歌词 `"[{...}]"`（因为 `]` 的 ASCII 值 93 < `{` 的 123）
- 这恰好实现了「有歌词的排在前面」的效果

#### 选取结果
- **Max: 1** - 只返回排序后的第一条记录
- 即：**优先选择有歌词的，歌词条件相同时选择更新时间最新的**

### 4.3 测试用例验证

**文件位置**: `server/subsonic/media_retrieval_test.go:113-151`

测试场景：三条同名同艺术家记录，按更新时间排序，但有歌词的记录优先

| ID | Lyrics | UpdatedAt | 排序结果 |
|----|--------|-----------|----------|
| 2 | 空（"[]"） | 2小时后（最新） | 第2名 |
| 1 | 有歌词 | 1小时后 | 第1名（胜出） |
| 3 | 空（"[]"） | 3小时后 | 第3名 |

**结果**: 即使 ID 1 不是最新的，但因为有歌词，仍然被选中。

---

## 5. 两个歌词接口的差异对比

Navidrome 提供两个 Subsonic API 端点获取歌词，它们的设计目标和返回格式有显著差异。

### 5.1 接口概览

| 特性 | GetLyrics | GetLyricsBySongId |
|------|-----------|-------------------|
| **API 路径** | `/rest/getLyrics` | `/rest/getLyricsBySongId` |
| **参数** | `artist`, `title` | `id` (歌曲ID) |
| **响应结构** | `Lyrics` (传统格式) | `LyricsList` (结构化格式) |
| **多条歌词** | ❌ 只返回第一条 | ✅ 返回全部 |
| **语言信息** | ❌ 丢失 | ✅ 保留 |
| **时间轴信息** | ❌ 丢失 | ✅ 保留 |
| **同步标记** | ❌ 丢失 | ✅ 保留 |

### 5.2 GetLyrics 接口实现

**文件位置**: `server/subsonic/media_retrieval.go:94-131`

```go
func (api *Router) GetLyrics(r *http.Request) (*responses.Subsonic, error) {
    p := req.Params(r)
    artist, _ := p.String("artist")   // 从请求参数获取
    title, _ := p.String("title")     // 从请求参数获取
    response := newResponse()
    lyricsResponse := responses.Lyrics{}
    response.Lyrics = &lyricsResponse
    
    mediaFiles, err := api.ds.MediaFile(r.Context()).GetAll(
        filter.SongsByArtistTitleWithLyricsFirst(artist, title))
    
    if len(mediaFiles) == 0 {
        return response, nil
    }
    
    structuredLyrics, err := api.lyrics.GetLyrics(r.Context(), &mediaFiles[0])
    
    if len(structuredLyrics) == 0 {
        return response, nil
    }
    
    // ⚠️  关键：Artist 和 Title 直接使用请求参数，不是从 MediaFile 获取
    lyricsResponse.Artist = artist
    lyricsResponse.Title = title
    
    // 关键：只取第一条歌词，丢弃时间轴，拼接成纯文本
    var lyricsText strings.Builder
    for _, line := range structuredLyrics[0].Line {
        lyricsText.WriteString(line.Value + "\n")
    }
    
    lyricsResponse.Value = lyricsText.String()
    return response, nil
}
```

**响应结构**: `server/subsonic/responses/responses.go:511-515`
```go
type Lyrics struct {
    Artist string `xml:"artist,omitempty,attr"  json:"artist,omitempty"`
    Title  string `xml:"title,omitempty,attr"   json:"title,omitempty"`
    Value  string `xml:",chardata"              json:"value"`
}
```

**字段取值来源（重点修正）**:
- **Artist**: 直接来自 `?artist=` 请求参数，**不是** 从数据库 `MediaFile.Artist` 获取
- **Title**: 直接来自 `?title=` 请求参数，**不是** 从数据库 `MediaFile.Title` 获取
- **Value**: 从 `structuredLyrics[0].Line` 拼接纯文本

**潜在问题**: 如果请求参数与实际数据库中的元数据不一致，响应会返回用户传入的值而非实际值。

**信息丢失点**:
1. **多条歌词**: 只取 `structuredLyrics[0]`，其他语言/版本被丢弃
2. **时间轴**: `Line.Start` 字段完全被忽略
3. **语言**: `Lang` 字段丢失
4. **同步标记**: `Synced` 字段丢失
5. **Offset**: 偏移量丢失
6. **真实元数据**: 不使用数据库中实际的 Artist/Title

### 5.3 GetLyricsBySongId 接口实现

**文件位置**: `server/subsonic/media_retrieval.go:133-153`

```go
func (api *Router) GetLyricsBySongId(r *http.Request) (*responses.Subsonic, error) {
    id, err := req.Params(r).String("id")
    
    mediaFile, err := api.ds.MediaFile(r.Context()).Get(id)
    
    structuredLyrics, err := api.lyrics.GetLyrics(r.Context(), mediaFile)
    
    // 关键：保留所有歌词，使用 buildLyricsList 构建结构化响应
    response.LyricsList = buildLyricsList(mediaFile, structuredLyrics)
    
    return response, nil
}
```

**响应结构**: `server/subsonic/responses/responses.go:545-562`
```go
type Line struct {
    Start *int64 `xml:"start,attr,omitempty" json:"start,omitempty"`
    Value string `xml:",chardata"            json:"value"`
}

type StructuredLyric struct {
    DisplayArtist string `xml:"displayArtist,attr,omitempty" json:"displayArtist,omitempty"`
    DisplayTitle  string `xml:"displayTitle,attr,omitempty"  json:"displayTitle,omitempty"`
    Lang          string `xml:"lang,attr"                    json:"lang"`
    Line          []Line `xml:"line"                         json:"line"`
    Offset        *int64 `xml:"offset,attr,omitempty"        json:"offset,omitempty"`
    Synced        bool   `xml:"synced,attr"                  json:"synced"`
}

type LyricsList struct {
    StructuredLyrics []StructuredLyric `xml:"structuredLyrics,omitempty" json:"structuredLyrics,omitempty"`
}
```

**信息完整性**:
1. **多条歌词**: 全部保留在 `StructuredLyrics` 数组中
2. **时间轴**: `Line.Start` 字段完整保留
3. **语言**: `Lang` 字段保留
4. **同步标记**: `Synced` 布尔字段保留
5. **Offset**: 偏移量保留
6. **元数据**: `DisplayArtist`、`DisplayTitle` 保留（缺失时从 MediaFile 补充）

### 5.4 差异影响分析

#### 对客户端的影响

| 场景 | GetLyrics | GetLyricsBySongId |
|------|-----------|-------------------|
| 显示纯文本歌词 | ✅ 可用 | ✅ 可用（需提取文本） |
| 卡拉OK歌词滚动 | ❌ 不可用（无时间轴） | ✅ 可用 |
| 多语言切换 | ❌ 不可用（只返回第一条） | ✅ 可用 |
| 显示歌词来源元数据 | ❌ 不可用 | ✅ 可用 |

#### 向后兼容性

- `GetLyrics` 是传统 Subsonic API，兼容性最好
- `GetLyricsBySongId` 是 OpenSubsonic 扩展 API，支持更丰富的功能
- 客户端需要根据能力选择合适的接口

#### 数据获取路径差异

```
GetLyrics 路径:
    artist/title 参数
        ↓
    SongsByArtistTitleWithLyricsFirst 过滤
        ↓ (排序后取第一条)
    MediaFile[0]
        ↓
    lyrics.GetLyrics()
        ↓ (取 LyricList[0])
    丢弃时间轴，拼接纯文本
        ↓
    Lyrics 响应

GetLyricsBySongId 路径:
    id 参数
        ↓
    MediaFile.Get(id)
        ↓
    lyrics.GetLyrics()
        ↓ (保留完整 LyricList)
    buildLyricsList() 构建结构化响应
        ↓ (元数据降级填充)
    LyricsList 响应
```

---

## 6. 三层决策链完整整合结论

### 6.1 决策链全景图

```
用户请求
    ↓
┌─────────────────────────────────────────┐
│ 第一层：选歌决策 (Song Selection)        │
│ ┌─────────────────────────────────────┐ │
│ │ 输入: artist, title (请求参数)      │ │
│ │ 过滤: title 精确匹配 + artist 匹配  │ │
│ │ 排序: lyrics DESC, updated_at DESC  │ │
│ │ 输出: MediaFile[0] (取第一条)       │ │
│ └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘
    ↓ (MediaFile)
┌─────────────────────────────────────────┐
│ 第二层：来源决策 (Source Selection)      │
│ ┌─────────────────────────────────────┐ │
│ │ 输入: MediaFile                     │ │
│ │ 配置: LyricsPriority (如 embedded,.lrc)│ │
│ │ 策略: 按顺序遍历，先到先得          │ │
│ │ 中断: len(lyricsList) > 0 立即返回  │ │
│ │ 输出: LyricList (第一条非空来源)    │ │
│ └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘
    ↓ (LyricList)
┌─────────────────────────────────────────┐
│ 第三层：接口返回决策 (API Response)      │
│ ┌─────────────────────────────────────┐ │
│ │ 接口: GetLyrics                     │ │
│ │   • Artist/Title: 请求参数回显      │ │
│ │   • 歌词: 取 LyricList[0] 纯文本    │ │
│ │   • 丢失: 时间轴、语言、同步标记    │ │
│ │                                     │ │
│ │ 接口: GetLyricsBySongId             │ │
│ │   • Artist/Title: MediaFile 降级填充 │ │
│ │   • 歌词: 完整保留 LyricList        │ │
│ │   • 保留: 所有元数据和时间轴        │ │
│ └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘
    ↓
API 响应
```

### 6.2 第一层：选歌决策详解

**触发条件**: 仅 `GetLyrics` 接口需要选歌；`GetLyricsBySongId` 直接通过 ID 定位。

| 决策点 | 规则 | 代码位置 |
|--------|------|----------|
| 匹配字段 | `title` 精确匹配，`artist` 通过 `json_tree` 匹配参与者 | `filter/filters.go:111-124` |
| 排序逻辑 | `lyrics DESC, updated_at DESC` | `filter/filters.go:113` |
| 排序语义 | 字符串字典序，利用 `"[]"` < `"[{...}]"` 特性 | `persistence/mediafile_repository.go:83-93` |
| 结果数量 | `Max: 1`，只取第一条 | `filter/filters.go:115` |

### 6.3 第二层：来源决策详解

**两个接口共享同一来源决策逻辑**。

| 决策点 | 规则 | 代码位置 |
|--------|------|----------|
| 来源顺序 | 按 `LyricsPriority` 配置逗号分割顺序遍历 | `core/lyrics/lyrics.go:38` |
| 来源类型 | `embedded` → `.lrc/.txt` → 插件名称 | `core/lyrics/lyrics.go:40-46` |
| 中断条件 | `len(lyricsList) > 0` 立即返回 | `core/lyrics/lyrics.go:53-55` |
| 错误处理 | 仅记录日志，继续下一个来源 | `core/lyrics/lyrics.go:49-51` |
| 最终输出 | 第一个非空来源的 `LyricList` | `core/lyrics/lyrics.go:54` |

### 6.4 第三层：接口返回决策对比

| 决策点 | GetLyrics | GetLyricsBySongId |
|--------|-----------|-------------------|
| Artist 取值 | 请求参数 `?artist=` 回显 | 优先 `Lyrics.DisplayArtist`，降级 `MediaFile.Artist` |
| Title 取值 | 请求参数 `?title=` 回显 | 优先 `Lyrics.DisplayTitle`，降级 `MediaFile.Title` |
| 歌词条数 | 只取 `LyricList[0]` | 保留全部 `LyricList` |
| 时间轴信息 | 丢弃，纯文本拼接 | 完整保留 `Line.Start` |
| 语言信息 | 丢弃 | 保留 `Lang` 字段 |
| 同步标记 | 丢弃 | 保留 `Synced` 字段 |
| Offset | 丢弃 | 保留 `Offset` 字段 |

### 6.5 关键核对点

**核对点 1：排序语义**
- ❌ 错误理解：通过 `(lyrics != '[]')` 布尔表达式排序
- ✅ 正确理解：直接 `lyrics DESC` 字符串排序，利用 `"[]"` < `"[{...}]"` 特性
- 验证：`persistence/mediafile_repository.go:83-93` 中无 `lyrics` 的 sortMapping

**核对点 2：旧接口元数据来源**
- ❌ 错误理解：从查询到的 MediaFile 中获取
- ✅ 正确理解：直接回显请求参数 `artist` 和 `title`
- 验证：`server/subsonic/media_retrieval.go:96-97, 120-121`

**核对点 3：三层决策独立性**
- 选歌决策与来源决策相互独立
- 选歌只决定使用哪个 MediaFile，来源决策决定该 MediaFile 使用哪个歌词来源
- 接口返回决策只影响输出格式，不影响数据获取

---

## 7. 代码架构总结

### 7.1 模块职责划分

| 模块 | 职责 | 文件 |
|------|------|------|
| 数据模型 | 歌词结构定义、LRC 格式解析 | `model/lyrics.go` |
| 服务层 | 来源路由、优先级控制 | `core/lyrics/lyrics.go` |
| 来源实现 | 内嵌、外部文件、插件三种来源 | `core/lyrics/sources.go` |
| 插件适配 | WASM 插件调用与结果解析 | `plugins/lyrics_adapter.go` |
| API 层 | Subsonic API 响应构建、元数据补充 | `server/subsonic/helpers.go` |
| 过滤器 | 同名歌曲选取规则 | `server/subsonic/filter/filters.go` |

### 7.2 设计特点

1. **可扩展的来源系统**: 通过配置字符串动态支持新的来源类型
2. **统一解析入口**: 所有来源最终都通过 `model.ToLyrics()` 解析
3. **容错性强**: 单个来源失败不影响整体流程
4. **插件友好**: WASM 插件只需返回原始文本，格式解析由主程序统一处理
5. **元数据降级**: 歌词元数据缺失时自动从媒体文件补充
6. **双接口设计**: 同时支持传统纯文本接口和现代结构化接口

### 7.3 关键技术决策总结

| 决策 | 说明 |
|------|------|
| **优先级胜出** | 多来源不合并，按配置顺序先到先得 |
| **歌词优先排序** | 同名歌曲通过 `lyrics DESC` 字符串排序优先选择有歌词的记录 |
| **请求参数回显** | 旧接口 `GetLyrics` 的 Artist/Title 直接回显请求参数 |
| **信息分层暴露** | 旧接口简化输出，新接口完整保留全部信息 |
| **插件原始文本协议** | 插件只需返回文本，解析逻辑集中处理 |
