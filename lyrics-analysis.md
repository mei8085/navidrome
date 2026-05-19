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

## 4. 代码架构总结

### 4.1 模块职责划分

| 模块 | 职责 | 文件 |
|------|------|------|
| 数据模型 | 歌词结构定义、LRC 格式解析 | `model/lyrics.go` |
| 服务层 | 来源路由、优先级控制 | `core/lyrics/lyrics.go` |
| 来源实现 | 内嵌、外部文件、插件三种来源 | `core/lyrics/sources.go` |
| 插件适配 | WASM 插件调用与结果解析 | `plugins/lyrics_adapter.go` |
| API 层 | Subsonic API 响应构建、元数据补充 | `server/subsonic/helpers.go` |

### 4.2 设计特点

1. **可扩展的来源系统**: 通过配置字符串动态支持新的来源类型
2. **统一解析入口**: 所有来源最终都通过 `model.ToLyrics()` 解析
3. **容错性强**: 单个来源失败不影响整体流程
4. **插件友好**: WASM 插件只需返回原始文本，格式解析由主程序统一处理
5. **元数据降级**: 歌词元数据缺失时自动从媒体文件补充
