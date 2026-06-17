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

### 2.2 为什么没有代理？

从代码中可以推断出以下设计考量：

1. **带宽成本**：电台流通常是持续的直播流，代理会消耗大量服务器带宽
2. **转码复杂性**：直播流转码需要特殊处理（HLS/ICY 等协议），与本地文件转码架构不同
3. **Subsonic 标准**：Subsonic API 规范中 Internet Radio 的 streamUrl 就是供客户端直接播放的
4. **功能边界**：Navidrome 定位是个人音乐服务器，而非电台代理服务器

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

| 模块 | 文件 | 核心作用 |
|------|------|----------|
| 数据模型 | [model/radio.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/model/radio.go) | Radio 结构体定义 |
| 数据模型 | [model/artwork_id.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/model/artwork_id.go) | ArtworkID 与 Kind 定义 |
| 数据持久化 | [persistence/radio_repository.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/persistence/radio_repository.go) | Radio 数据库操作 |
| 封面服务 | [core/artwork/reader_radio.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/core/artwork/reader_radio.go) | 电台封面读取器 |
| Native API | [server/nativeapi/radios.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/server/nativeapi/radios.go) | 电台 REST API |
| Subsonic API | [server/subsonic/radio.go](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/server/subsonic/radio.go) | Subsonic 电台接口 |
| UI 适配层 | [ui/src/radio/helper.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/radio/helper.jsx) | 电台→歌曲适配 |
| UI 播放器 | [ui/src/reducers/playerReducer.js](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/reducers/playerReducer.js) | 播放队列状态管理 |
| UI 播放器 | [ui/src/audioplayer/Player.jsx](file:///d:/fz/0601-2/solo-dogfeeding/code/20-navidrome/ui/src/audioplayer/Player.jsx) | 主播放器组件 |
