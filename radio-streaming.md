# Navidrome 网络电台流式播放协同机制

## 概述

Navidrome 的网络电台功能由三大模块协同工作：**订阅源管理**、**流式代理**和**播放器对接**。与本地曲库不同，电台流采用"直连播放"模式，音频数据不经过 Navidrome 服务器中转，只有元数据（名称、封面等）由服务器管理。

---

## 一、订阅源管理（Subscription Management）

### 1.1 数据模型

核心数据结构定义在 [radio.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/model/radio.go#L9-L26)：

```go
type Radio struct {
    ID            string    // 电台唯一标识
    StreamUrl     string    // 音频流地址（核心）
    Name          string    // 电台名称
    HomePageUrl   string    // 电台主页
    UploadedImage string    // 用户上传的封面图文件名
    CreatedAt     time.Time
    UpdatedAt     time.Time
}
```

Radio 实现了 `CoverArtID()` 方法，返回 `ra-` 前缀的 ArtworkID，与专辑（`al-`）、艺术家（`ar-`）等实体的封面图系统统一。

### 1.2 数据持久化

[radio_repository.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/persistence/radio_repository.go#L15-L125) 实现了 RadioRepository 接口：

- **权限控制**：只有管理员用户可以创建、修改、删除电台
- **查询能力**：支持按名称模糊搜索（`containsFilter`）
- **CRUD 操作**：Get / GetAll / CountAll / Put / Delete
- **REST 适配**：实现了 `rest.Repository` 和 `rest.Persistable` 接口

### 1.3 API 层

#### Native API（[radios.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/server/nativeapi/radios.go#L16-L70)）
- 路径：`/api/radio`
- 标准 RESTful 接口：GET / POST / PUT / DELETE
- 额外支持封面图上传：`POST /{id}/image`、`DELETE /{id}/image`

#### Subsonic API（[radio.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/server/subsonic/radio.go#L14-L127)）
- `getInternetRadioStations` - 获取所有电台
- `createInternetRadioStation` - 创建电台
- `updateInternetRadioStation` - 更新电台
- `deleteInternetRadioStation` - 删除电台

Subsonic 响应结构在 [responses.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/server/subsonic/responses/responses.go#L517-L531) 中定义，包含 OpenSubsonic 扩展的 `CoverArt` 字段（非 legacy 客户端才返回）。

### 1.4 封面图管理

电台封面图通过统一的 Artwork 系统提供服务，在 [reader_radio.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/core/artwork/reader_radio.go#L11-L40) 中实现：

- `radioArtworkReader` 实现 `artworkReader` 接口
- 封面来源：用户上传的图片（`UploadedImage`）
- 没有上传图片时返回 `ErrUnavailable`
- ArtworkID 前缀：`ra-`（在 [artwork_id.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/model/artwork_id.go#L26) 中定义）

---

## 二、流式代理（Streaming Proxy）

### 2.1 核心发现：无代理模式

**Navidrome 对电台音频流不做任何代理或中转。**

这是理解整个系统最关键的一点：
- 电台的 `StreamUrl` 直接暴露给客户端
- 播放器直接连接电台服务器获取音频流
- Navidrome 服务器不参与音频数据的传输

对比本地歌曲的播放路径：
```
本地歌曲: 播放器 → Navidrome /rest/stream → 读取本地文件 → 转码(可选) → 播放器
网络电台: 播放器 → 电台服务器 StreamUrl → 播放器 (完全不经过 Navidrome)
```

### 2.2 为什么没有代理？——从转码架构看技术原因

外部流不经过 Navidrome 转码代理，不仅仅是设计选择，更是现有架构的客观限制。让我们从转码系统的内部结构来理解：

#### 2.2.1 转码管道的输入假设

整个转码系统的入口是 `MediaStreamer.NewStream()`，定义在 [media_streamer.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/core/stream/media_streamer.go#L26-L27)：

```go
type MediaStreamer interface {
    NewStream(ctx context.Context, mf *model.MediaFile, req Request) (*Stream, error)
}
```

**输入参数是 `*model.MediaFile`**，而不是 `*model.Radio`。这意味着：
- 转码系统从设计上就只认识本地媒体文件
- Radio 实体根本无法进入转码管道
- 没有任何代码路径能把电台的 `StreamUrl` 喂给 ffmpeg

#### 2.2.2 ffmpeg 的输入约束

深入到 [ffmpeg.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/core/ffmpeg/ffmpeg.go#L75-L89) 的 `Transcode()` 方法：

```go
func (e *ffmpeg) Transcode(ctx context.Context, opts TranscodeOptions) (io.ReadCloser, error) {
    if _, err := ffmpegCmd(); err != nil {
        return nil, err
    }
    if err := fileExists(opts.FilePath); err != nil {  // 关键点：检查本地文件是否存在
        return nil, err
    }
    // ...
    args = append(args, "-i", opts.FilePath)       // -i 后面是本地文件路径
    // ...
}
```

`TranscodeOptions` 中的 `FilePath` 字段（定义在 [ffmpeg.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/core/ffmpeg/ffmpeg.go#L25-L34)）是整个转码过程的输入源，它被假定为**本地文件系统路径**：
- `fileExists(opts.FilePath)` 调用 `os.Stat()` 检查本地文件
- `-i` 参数直接传递给 ffmpeg 作为输入文件
- 完全没有 URL 输入的处理逻辑

ffmpeg 本身确实支持 HTTP 输入（`ffmpeg -i http://...`），但 Navidrome 的封装层没有暴露这个能力，也没有相关的错误处理、超时控制、重定向处理等机制。

#### 2.2.3 转码决策依赖的元数据

转码决策器 `TranscodeDecider`（定义在 [decider.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/core/stream/decider.go#L21-L26)）的 `MakeDecision()` 方法需要 `*model.MediaFile` 来获取：

| 元数据 | 来源 | 电台是否可用 |
|--------|------|-------------|
| 容器格式（Suffix） | 文件扩展名 | ❌ 只有 URL |
| 编码格式（AudioCodec） | 标签/ffprobe | ❌ 不可用 |
| 比特率（BitRate） | 标签/ffprobe | ❌ 不可用 |
| 采样率（SampleRate） | 标签/ffprobe | ❌ 不可用 |
| 声道数（Channels） | 标签/ffprobe | ❌ 不可用 |
| 时长（Duration） | 标签/ffprobe | ❌ 直播流无固定时长 |
| 文件大小（Size） | 文件系统 | ❌ 不可用 |
| ProbeData | ffprobe 结果缓存 | ❌ 不可用 |

电台流在播放前无法可靠获取这些元数据（除非先连接并探测），而转码决策需要在播放开始前完成。

#### 2.2.4 缓存与 seek 架构不匹配

本地文件转码缓存的 Key（定义在 [media_streamer.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/core/stream/media_streamer.go#L59-L61)）是：

```go
func (j *streamJob) Key() string {
    return fmt.Sprintf("%s.%s.%d.%d.%d.%d.%s.%d", 
        j.mf.ID, j.mf.UpdatedAt.Format(time.RFC3339Nano), 
        j.bitRate, j.sampleRate, j.bitDepth, j.channels, 
        j.format, j.offset)
}
```

这个缓存机制的前提是**内容固定**：同样的 ID + 同样的参数 → 同样的输出。但电台直播流是实时的：
- 内容随时间变化，缓存没有意义
- `UpdatedAt` 对直播流没有意义
- `offset`（秒级跳转）对直播流不可用

Seek 功能同样不成立：本地文件可以通过 `-ss` 参数跳到指定位置（[media_streamer.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/core/stream/media_streamer.go#L158-L162) 用 `http.ServeContent` 支持 Range 请求），但直播流是顺序的，无法回退。

#### 2.2.5 设计层面的综合考量

除了上述架构限制，还有明确的设计选择：

1. **带宽成本**：电台流是持续的直播流，代理会消耗大量服务器带宽和流量
2. **协议复杂性**：网络电台使用多种协议（HTTP progressive、HLS、ICY/SHOUTcast），每种都需要专门处理
3. **Subsonic 标准对齐**：Subsonic API 规范中 Internet Radio 的 streamUrl 就是供客户端直接播放的，服务器不参与
4. **功能边界**：Navidrome 定位是个人音乐服务器，而非电台代理/重流服务器
5. **故障责任**：直连模式下，电台不可用是电台的问题；代理模式下，用户会归咎于 Navidrome

### 2.3 经过服务器的唯一数据：封面图

封面图是唯一经过 Navidrome 服务器的电台相关媒体资源：

```
getCoverArt?id=ra-xxx → Artwork 系统 → radioArtworkReader → 本地文件系统
```

---

## 三、播放器对接（Player Integration）

### 3.1 电台 → 歌曲 的适配层

UI 端通过 [helper.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/radio/helper.jsx#L5-L33) 中的 `songFromRadio()` 函数将电台转换为"伪歌曲"格式：

```javascript
function songFromRadio(radio) {
  return {
    ...radio,
    title: radio.name,        // 电台名称作为标题
    album: radio.homePageUrl, // 主页 URL 作为专辑名
    artist: radio.name,       // 电台名称作为艺术家
    cover,                     // 封面图 URL
    isRadio: true,             // 关键标记：区分电台与普通歌曲
  }
}
```

封面图优先级：
1. 用户上传的封面图（通过 Navidrome artwork 接口）
2. 电台主页的 favicon.ico（前端探测，失败则使用占位图）

### 3.2 播放队列中的电台处理

在 [playerReducer.js](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/reducers/playerReducer.js#L43-L99) 的 `mapToAudioLists()` 中：

```javascript
if (item.isRadio) {
  return {
    trackId,
    musicSrc: item.streamUrl,  // 直接使用原始流地址
    cover: item.cover,
    isRadio: true,
    // ...
  }
} else {
  return {
    musicSrc: makeMusicSrc(trackId),  // 经过 Navidrome 代理
    // ...
  }
}
```

**关键差异**：
- 普通歌曲：`musicSrc` 是一个函数，调用 `decisionService.resolveStreamUrl()` 获取转码后的播放地址
- 电台：`musicSrc` 直接就是 `streamUrl` 字符串，浏览器直连

### 3.3 播放器行为差异

在 [Player.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/audioplayer/Player.jsx) 中，`isRadio` 标记驱动了多种 UI 行为变化：

| 功能 | 本地歌曲 | 网络电台 | 代码位置 |
|------|----------|----------|----------|
| 进度条显示 | 显示 | 隐藏 | [styles.js](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/audioplayer/styles.js#L83-L86) |
| Media Session | 启用 | 禁用 | [Player.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/audioplayer/Player.jsx#L257) |
| 播放报告 | 发送 | 不发送 | [Player.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/audioplayer/Player.jsx#L437) |
| 页面离开时上报 | 上报 | 不上报 | [Player.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/audioplayer/Player.jsx#L183) |
| 保存队列按钮 | 可用 | 禁用 | [PlayerToolbar.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/audioplayer/PlayerToolbar.jsx#L60) |

### 3.4 播放队列持久化

**服务端 PlayQueue 不支持电台**：

- 服务端 [PlayQueue](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/model/playqueue.go#L7-L16) 的 `Items` 字段类型是 `MediaFiles`，只能存储本地歌曲
- [queue.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/core/playback/queue.go#L12-L15) 的 playback Queue 也是 `model.MediaFiles` 类型
- Native API 的 [queue.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/server/nativeapi/queue.go) 只处理 MediaFile ID

因此：
- 电台只存在于前端 Redux store 的内存队列中
- 刷新页面或重新登录后，电台不会保留在播放队列中
- 服务端同步的播放队列只包含本地歌曲

### 3.5 Jukebox 模式

Jukebox（点唱机）模式由 [jukebox.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/server/subsonic/jukebox.go) 处理，使用 `core/playback` 模块驱动服务器端播放器（如 mpv）。

当前 Jukebox 只支持 `MediaFile`（本地歌曲），**不支持网络电台**，因为：
- `pb.Set(ctx, ids)` 接收的是媒体文件 ID
- 内部 Queue 存储的是 `model.MediaFiles`
- mpv 播放需要本地文件路径或 Navidrome 代理的流地址

---

## 四、三者协同机制全景

### 4.1 架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                        客户端 (浏览器)                       │
│  ┌──────────┐    ┌──────────────┐    ┌─────────────────┐   │
│  │ 电台列表  │ →  │ songFromRadio │ →  │  音乐播放器      │   │
│  └──────────┘    └──────────────┘    └─────────────────┘   │
│        │                │                   │               │
│        │ 元数据请求      │ 封面图请求         │ 音频流请求    │
└────────┼────────────────┼───────────────────┼───────────────┘
         │                │                   │
         ▼                ▼                   ▼
┌──────────────────────────────────┐    ┌─────────────┐
│       Navidrome 服务器           │    │  电台服务器  │
│  ┌─────────┐  ┌──────────────┐  │    │             │
│  │ Radio   │  │   Artwork    │  │    │  StreamUrl  │
│  │ 存储    │  │   封面服务   │  │    │             │
│  └─────────┘  └──────────────┘  │    └─────────────┘
└──────────────────────────────────┘
```

### 4.2 完整播放流程

**步骤 1：订阅管理（管理员）**
```
管理员 → Native API /radio → RadioRepository → SQLite
              (创建/修改/删除电台)
```

**步骤 2：获取电台列表**
```
播放器 → GET /rest/getInternetRadioStations
           → RadioRepository.GetAll()
           → 组装 Radio 响应（含 streamUrl 和 coverArt）
           → 返回给客户端
```

**步骤 3：触发播放**
```
用户点击电台 → handleRowClick()
             → dispatch(setTrack(songFromRadio(record)))
             → mapToAudioLists() 标记 isRadio=true
             → musicSrc = item.streamUrl (直连地址)
```

**步骤 4：音频播放**
```
浏览器音频元素 → 直接连接电台 StreamUrl
                (完全不经过 Navidrome 服务器)
```

**步骤 5：封面展示**
```
播放器 → GET /rest/getCoverArt?id=ra-xxx
         → Artwork.Get()
         → radioArtworkReader
         → 读取本地上传的图片文件
         → 返回封面图
```

### 4.3 与本地曲库的对比

| 维度 | 本地曲库 | 网络电台 |
|------|----------|----------|
| **音频流路径** | Navidrome 代理（支持转码） | 直连电台服务器 |
| **音频来源** | 本地文件系统 | 外部网络流 |
| **转码支持** | 完整支持（ffmpeg） | 不支持 |
| **进度控制** | 支持 seek | 不支持（直播流） |
| **封面图** | 经过服务器 | 经过服务器 |
| **元数据存储** | SQLite | SQLite |
| **播放队列持久化** | 服务端存储 | 仅前端内存 |
| **播放统计** | 完整 scrobble | 无 |
| **搜索** | 多字段全文搜索 | 仅名称搜索 |
| **Jukebox 支持** | 支持 | 不支持 |
| **分享功能** | 支持 | 不支持 |

### 4.4 关键设计模式

1. **统一封面系统**：通过 `Kind` 前缀（`ra-`、`al-`、`ar-` 等）实现不同实体封面的统一访问，在 [artwork_id.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/model/artwork_id.go) 中定义

2. **适配器模式**：`songFromRadio()` 将电台对象适配成播放器可识别的"歌曲"接口，通过 `isRadio` 标记处理差异行为

3. **REST Repository 模式**：`RadioRepository` 同时实现领域接口和 `rest.Repository` 接口，一套实现同时服务于业务逻辑和 HTTP API

4. **直连播放**：规避服务器带宽成本，将流式传输的责任交给客户端和电台服务器

---

## 五、关键代码文件索引

### 5.1 订阅源管理

| 模块 | 文件 | 核心作用 |
|------|------|----------|
| 数据模型 | [model/radio.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/model/radio.go) | Radio 结构体定义 |
| 数据模型 | [model/artwork_id.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/model/artwork_id.go) | ArtworkID 与 Kind 定义 |
| 数据持久化 | [persistence/radio_repository.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/persistence/radio_repository.go) | Radio 数据库操作 |
| 封面服务 | [core/artwork/reader_radio.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/core/artwork/reader_radio.go) | 电台封面读取器 |
| Native API | [server/nativeapi/radios.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/server/nativeapi/radios.go) | 电台 REST API |
| Subsonic API | [server/subsonic/radio.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/server/subsonic/radio.go) | Subsonic 电台接口 |
| Subsonic 响应 | [server/subsonic/responses/responses.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/server/subsonic/responses/responses.go#L517-L531) | Radio 响应结构定义 |

### 5.2 转码架构（电台不可达）

| 模块 | 文件 | 核心作用 |
|------|------|----------|
| 流媒体入口 | [core/stream/media_streamer.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/core/stream/media_streamer.go) | MediaStreamer 接口（仅支持 MediaFile） |
| 转码决策 | [core/stream/decider.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/core/stream/decider.go) | TranscodeDecider 转码决策服务 |
| 转码类型 | [core/stream/types.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/core/stream/types.go) | 转码请求/响应类型定义 |
| ffmpeg 封装 | [core/ffmpeg/ffmpeg.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/core/ffmpeg/ffmpeg.go) | ffmpeg 转码封装（仅本地文件输入） |
| Subsonic 转码 API | [server/subsonic/transcode.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/server/subsonic/transcode.go) | getTranscodeDecision 接口（仅 song 类型） |
| CORS 中间件 | [server/middlewares.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/server/middlewares.go#L87-L102) | 服务器 CORS 配置 |

### 5.3 播放器对接

| 模块 | 文件 | 核心作用 |
|------|------|----------|
| UI 适配层 | [ui/src/radio/helper.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/radio/helper.jsx) | 电台→歌曲适配（songFromRadio） |
| UI 电台列表 | [ui/src/radio/index.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/radio/index.jsx) | 电台列表页面 |
| UI 播放器状态 | [ui/src/reducers/playerReducer.js](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/reducers/playerReducer.js) | 播放队列状态与 isRadio 处理 |
| UI 主播放器 | [ui/src/audioplayer/Player.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/audioplayer/Player.jsx) | 主播放器组件与 crossOrigin 设置 |
| UI 转码决策 | [ui/src/transcode/decisionService.js](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/transcode/decisionService.js) | 前端转码决策服务（电台绕过） |
| 服务端播放队列 | [core/playback/queue.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/core/playback/queue.go) | 服务端播放队列（仅 MediaFile） |
| Jukebox | [server/subsonic/jukebox.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/server/subsonic/jukebox.go) | 点唱机模式（不支持电台） |

---

## 六、播放失败诊断指南

外部电台流播放失败时，由于音频数据完全不经过 Navidrome 服务器，传统的服务端日志手段无法定位问题。本节从代码出发，梳理 Web 播放器和第三方客户端各自的错误路径和诊断方法。

### 6.1 Web 播放器的错误处理路径

#### 6.1.1 onAudioError 回调：电台与本地歌曲的差异处理

在 [Player.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/audioplayer/Player.jsx#L374-L394) 中，`onAudioError` 回调对电台和本地歌曲做了**关键区分**：

```javascript
const onAudioError = useCallback(
  (error, currentPlayId, audioLists, audioInfo) => {
    // Invalidate all cached decisions — token may be stale
    decisionService.invalidateAll()

    // Pre-fetch decisions for upcoming songs with fresh tokens
    const currentIdx = playerState.queue.findIndex(
      (item) => item.uuid === currentPlayId,
    )
    if (currentIdx >= 0) {
      const nextSongIds = playerState.queue
        .slice(currentIdx + 1, currentIdx + 4)
        .filter((item) => !item.isRadio)    // ← 关键：跳过电台
        .map((item) => item.trackId)
      if (nextSongIds.length > 0) {
        decisionService.prefetchDecisions(nextSongIds)
      }
    }
  },
  [playerState.queue],
)
```

**诊断要点**：
- 当播放失败触发 `onAudioError` 时，代码首先清除所有缓存的转码决策（`invalidateAll()`）
- 然后尝试为接下来的歌曲预取转码决策，但**跳过电台项**（`!item.isRadio`）
- **电台失败不会触发任何重试逻辑**——没有转码 token 可刷新，因为电台本身就不走转码
- 电台失败后，由于 `loadAudioErrorPlayNext: false`，播放器**不会自动跳到下一首**，用户看到的是一个停止的播放器，没有明确的错误提示

#### 6.1.2 本地歌曲失败 vs 电台失败的对比

| 维度 | 本地歌曲失败 | 电台失败 |
|------|------------|---------|
| 错误回调 | `onAudioError` 触发 | `onAudioError` 触发 |
| 重试策略 | `invalidateAll()` 后重取转码 token | 无重试（电台无 token） |
| 预取逻辑 | 失败后为后续歌曲预取决策 | 后续电台被跳过 |
| 自动跳过 | `loadAudioErrorPlayNext` 控制 | `false`，不自动跳过 |
| 错误可见性 | `react-jinke-music-player` 内置提示 | 同左，但错误信息被 CORS 屏蔽 |
| 服务端日志 | Navidrome 有请求日志 | **无**，Navidrome 不参与 |

#### 6.1.3 刷新队列时的电台保留逻辑

在 [playerReducer.js](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/reducers/playerReducer.js#L232-L247) 的 `PLAYER_REFRESH_QUEUE` 处理中：

```javascript
case PLAYER_REFRESH_QUEUE: {
  const resolvedUrls = payload.data || {}
  return {
    ...previousState,
    queue: previousState.queue.map((item) => ({
      ...item,
      musicSrc: item.isRadio
        ? item.musicSrc                              // ← 电台保持原 streamUrl
        : resolvedUrls[item.trackId] || subsonic.streamUrl(item.trackId),  // 歌曲刷新
    })),
    clear: true,
    autoPlay: false,
    playIndex: previousState.savedPlayIndex >= 0 ? previousState.savedPlayIndex : 0,
  }
}
```

**诊断要点**：
- 当用户刷新播放队列时，本地歌曲的 `musicSrc` 会被重新解析（获取新的转码 token）
- 电台的 `musicSrc` **永远保持为原始 `streamUrl`**，不会被刷新
- 这意味着：如果电台 URL 本身有效但暂时不可用，刷新队列不会解决电台的播放问题
- 反过来，如果电台 URL 是动态的（如带 token 的临时链接），刷新也不会更新它

### 6.2 浏览器跨域失败的诊断

#### 6.2.1 跨域错误的两种形态

浏览器对跨域电台流的错误处理有两种截然不同的表现：

**形态 1：音频元素级别失败（CORS 完全阻止）**

当电台服务器设置了严格的 CORS 策略（如 `Access-Control-Allow-Origin: https://specific-site.com`），且 Navidrome UI 的 origin 不在允许列表中：
- `<audio>` 元素触发 `error` 事件
- `MediaError.code` = `4`（`MEDIA_ERR_SRC_NOT_SUPPORTED`）
- `MediaError.message` 通常为空或只显示笼统的错误描述
- **浏览器安全策略禁止暴露跨域错误的详细信息**

这是最难诊断的场景：错误消息被浏览器主动隐藏了。打开浏览器开发者工具的 Network 面板，看到的可能是：
- 请求根本没有发出去（preflight 失败）
- 或者请求发出但 response 被标记为 "blocked by CORS policy"

**形态 2：Web Audio API 级别失败（CORS 阻止音频处理）**

当电台服务器没有 CORS 头，但音频数据本身可以播放时：
- `<audio>` 播放正常（no-cors 模式）
- 但 `createMediaElementSource()` 抛出 `InvalidStateError` 或静默失败
- 代码中设置了 `audioInstance.crossOrigin = 'anonymous'`（[Player.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/audioplayer/Player.jsx#L152)），这反而可能导致原本能播放的电台失败

**关键诊断问题**：设置 `crossOrigin = 'anonymous'` 后，浏览器会以 CORS 模式发起请求。如果电台服务器不返回 CORS 头，连原本可以播放的音频也会失败。

#### 6.2.2 crossOrigin 设置的副作用链

```
启用 ReplayGain
  → audioInstance.crossOrigin = 'anonymous'
    → 浏览器以 CORS 模式请求电台流
      → 电台服务器无 CORS 头
        → 浏览器阻止播放（原本不设 crossOrigin 时能播放）
          → 用户困惑：不启用 ReplayGain 能播，启用了反而不能播
```

这个副作用链在代码中没有被处理。设置 `crossOrigin` 的条件是启用 ReplayGain（[Player.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/audioplayer/Player.jsx#L142-L162)），但代码没有检查当前播放的是否是电台、电台流是否支持 CORS。

#### 6.2.3 诊断跨域问题的操作方法

| 步骤 | 操作 | 预期结果 |
|------|------|----------|
| 1 | 打开浏览器 DevTools → Network 面板 | 查看电台流请求 |
| 2 | 查看请求的 Response Headers | 是否有 `Access-Control-Allow-Origin` |
| 3 | 查看请求状态 | `200` 但被 CORS 阻止？还是 `4xx/5xx`？ |
| 4 | 查看控制台错误 | "blocked by CORS policy" 或 "InvalidStateError" |
| 5 | 禁用 ReplayGain 后重试 | 如果能播，说明是 `crossOrigin` 设置的副作用 |
| 6 | 用 `curl` 或 Postman 测试电台 URL | 确认 URL 本身是否有效 |
| 7 | 检查电台 URL 协议 | HTTPS 页面不能播放 HTTP 流（mixed content） |

**HTTPS 混合内容问题**：如果 Navidrome 运行在 HTTPS 上，浏览器会阻止 `<audio>` 元素加载 HTTP 的电台流。这是一个常见的失败原因，但错误信息可能与 CORS 错误混淆。Network 面板中会显示 "mixed content" 警告。

### 6.3 播放器设置对失败的影响

#### 6.3.1 loadAudioErrorPlayNext: false 的设计意图

在 [Player.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/audioplayer/Player.jsx#L211) 中：

```javascript
loadAudioErrorPlayNext: false,
```

这个设置对电台播放有**双重影响**：
1. **正面**：电台失败时不会自动跳到下一首（本地歌曲），保留了用户的电台选择意图
2. **负面**：没有"重试"机制——电台直播流可能只是暂时不可用，但用户无法自动恢复

#### 6.3.2 showMediaSession: !isRadio 的副作用

在 [Player.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/audioplayer/Player.jsx#L257) 中：

```javascript
showMediaSession: !current.isRadio,
```

禁用 Media Session 意味着：
- 系统媒体控制栏（锁屏、通知栏）不显示电台信息
- 硬件播放/暂停键不控制电台播放
- 用户无法通过系统级 UI 控制电台

**诊断场景**：用户反馈"电台播放时锁屏没有控制按钮"——这不是 bug，是设计选择，因为直播流的时长和进度不确定，Media Session API 的行为会不一致。

#### 6.3.3 PlayerToolbar 中电台的功能禁用

在 [PlayerToolbar.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/audioplayer/PlayerToolbar.jsx#L58-L100) 中，`isRadio` 标记禁用了两个按钮：

```javascript
const PlayerToolbar = ({ id, isRadio }) => {
  const { data, loading } = useGetOne('song', id, { enabled: !!id && !isRadio })
  // ...
  <IconButton disabled={isRadio}>           // 保存队列按钮
  <LoveButton disabled={... || isRadio} />  // 收藏按钮
}
```

**诊断场景**：
- "保存队列"按钮灰显 → 因为电台不能持久化到服务端 PlayQueue
- "收藏"按钮灰显 → 因为电台不是 `model.MediaFile`，没有 star/unstar 接口
- `useGetOne('song', id, { enabled: !!id && !isRadio })` → 电台不会触发歌曲数据获取，避免 404 错误

### 6.4 第三方 Subsonic 客户端的失败场景

#### 6.4.1 服务器端不参与流传输——所有客户端的共性

Navidrome 的 `getInternetRadioStations` API 返回的 `streamUrl` 是原始 URL，服务器不做任何代理或包装。所有客户端拿到这个 URL 后，播放成败完全取决于：

1. 电台服务器是否可达
2. 电台服务器返回的 HTTP 头（CORS、Content-Type 等）
3. 音频编码格式是否被客户端支持
4. 网络连接是否稳定

**Navidrome 服务端日志无法帮助诊断电台播放失败**——因为服务器根本没有参与音频传输。

#### 6.4.2 Legacy 客户端的 CoverArt 丢失

在 [radio.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/server/subsonic/radio.go#L73-L84) 中：

```go
player, _ := request.PlayerFrom(ctx)
if strings.Contains(conf.Server.Subsonic.LegacyClients, player.Client) {
    continue   // ← Legacy 客户端跳过 CoverArt 字段
}
var coverArt string
if g.UploadedImage != "" {
    coverArt = g.CoverArtID().String()
}
res[i].OpenSubsonicRadio = &responses.OpenSubsonicRadio{
    CoverArt: coverArt,
}
```

**诊断场景**：
- Legacy 客户端（在 `LegacyClients` 配置中的客户端）获取电台列表时，`OpenSubsonicRadio` 字段被跳过
- 这些客户端的电台没有封面图，但**这不是 bug，是兼容性处理**
- 检查方法：查看 Navidrome 配置中的 `Subsonic.LegacyClients` 列表

#### 6.4.3 客户端尝试转码电台的失败

支持 OpenSubsonic 的客户端可能尝试对电台调用 `getTranscodeDecision`：

```
POST /rest/getTranscodeDecision
{
  "mediaId": "ra-xxxxxxxx",
  "mediaType": "radio",
  ...
}
```

服务端响应：
```json
{
  "subsonic-response": {
    "status": "failed",
    "error": {
      "code": 0,
      "message": "mediaType 'radio' is not yet supported"
    }
  }
}
```

来自 [transcode.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/server/subsonic/transcode.go#L171-L178) 和 [transcode.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/server/subsonic/transcode.go#L256-L258) 的限制。

**诊断场景**：
- 客户端日志显示 "mediaType not supported" 错误 → 不是 bug，是架构限制
- 客户端应该直接播放 `streamUrl`，不尝试转码

#### 6.4.4 电台管理的权限限制

在 [radio_repository.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/persistence/radio_repository.go#L29-L32) 中：

```go
func (r *radioRepository) isPermitted() bool {
    user := loggedUser(r.ctx)
    return user.IsAdmin
}
```

**诊断场景**：
- 非管理员用户通过 Subsonic API 调用 `createInternetRadioStation` → 返回权限错误
- 非管理员用户通过 Native API 调用 `POST /api/radio` → 返回 403
- 但 `getInternetRadioStations` 对所有用户开放——**读取是全局的，写入是管理员专属**

### 6.5 失败场景速查表

| 症状 | 可能原因 | 诊断方法 | 代码位置 |
|------|----------|----------|----------|
| 电台无法播放，无错误提示 | 电台服务器不可达 | `curl` 测试 URL | - |
| 电台无法播放，控制台 CORS 错误 | 电台服务器无 CORS 头 + ReplayGain 启用 | 禁用 ReplayGain 重试 | [Player.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/audioplayer/Player.jsx#L152) |
| 电台无法播放，mixed content 警告 | Navidrome HTTPS + 电台 HTTP 流 | 改用 HTTPS 电台 URL | - |
| 启用 ReplayGain 后电台播放失败 | `crossOrigin='anonymous'` 触发 CORS 预检 | 确认电台 CORS 头 | [Player.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/audioplayer/Player.jsx#L142-L162) |
| 电台播放正常但无 ReplayGain | 电台 CORS 不支持 Web Audio | 检查 Network 面板 CORS 头 | [Player.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/audioplayer/Player.jsx#L153) |
| 播放失败后播放器停止不动 | `loadAudioErrorPlayNext: false` | 手动切换曲目 | [Player.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/audioplayer/Player.jsx#L211) |
| 刷新页面后电台从队列消失 | 队列不持久化电台 | 正常行为，非 bug | [playerReducer.js](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/reducers/playerReducer.js#L43-L57) |
| 电台无封面图（第三方客户端） | Legacy 客户端跳过 CoverArt | 检查 LegacyClients 配置 | [radio.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/server/subsonic/radio.go#L73-L76) |
| 电台无法转码 | `mediaType: "radio"` 不支持 | 直接播放 streamUrl | [transcode.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/server/subsonic/transcode.go#L171-L178) |
| 非管理员无法添加电台 | `isPermitted()` 检查 | 使用管理员账号 | [radio_repository.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/persistence/radio_repository.go#L29-L32) |
| 电台无进度条/无 Media Session | `isRadio` 标记禁用 | 正常行为，非 bug | [Player.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/audioplayer/Player.jsx#L257) |
| 电台不能收藏/不能保存队列 | 电台不是 MediaFile | 正常行为，非 bug | [PlayerToolbar.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/audioplayer/PlayerToolbar.jsx#L84-L97) |
| 电台流 URL 无法被 Navidrome 日志追踪 | 服务器不参与音频传输 | 检查浏览器 Network 面板 | - |
