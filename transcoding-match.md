# Navidrome 转码档位与客户端能力匹配机制分析

## 1. 整体架构概览

Navidrome 的转码系统采用分层设计，从客户端识别到最终转码输出经过以下核心模块：

```
客户端请求 → 客户端识别 → 转码决策 → 流式处理 → ffmpeg 转码 → 输出流
```

主要组件：
- **客户端识别层**：[middlewares.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/server/subsonic/middlewares.go) 中的 `getPlayer` 中间件
- **转码决策层**：[decider.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/stream/decider.go) 中的 `TranscodeDecider` 接口
- **流式处理层**：[media_streamer.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/stream/media_streamer.go) 中的 `MediaStreamer`
- **ffmpeg 执行层**：[ffmpeg.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/ffmpeg/ffmpeg.go) 中的 `FFmpeg` 接口

---

## 2. 客户端识别机制

### 2.1 Player 模型与识别流程

客户端（播放器）识别发生在 Subsonic API 的中间件链中，由 `getPlayer` 中间件处理。

**核心数据结构** - [player.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/model/player.go#L7-L21)：

```go
type Player struct {
    ID              string
    Name            string
    UserAgent       string
    UserId          string
    Client          string    // Subsonic API 的 c 参数
    IP              string
    LastSeen        time.Time
    TranscodingId   string    // 关联的转码配置 ID
    MaxBitRate      int       // 最大比特率限制 (kbps)
    ReportRealPath  bool
    ScrobbleEnabled bool
}
```

### 2.2 客户端识别流程

识别流程在 [players.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/players.go#L34-L78) 的 `Register` 方法中实现：

1. **Cookie 识别**：首先尝试从 Cookie `nd-player-{username_hash}` 中读取 playerId
2. **ID 匹配**：如果有 playerId，直接从数据库获取 Player 记录
3. **特征匹配**：如果 ID 不匹配或不存在，使用 `FindMatch(userID, client, userAgent)` 进行模糊匹配
4. **自动注册**：如果都没匹配到，自动创建新的 Player 记录

User-Agent 规范化在 [middlewares.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/server/subsonic/middlewares.go#L235-L242)：

```go
func canonicalUserAgent(r *http.Request) string {
    u := ua.Parse(r.Header.Get("user-agent"))
    userAgent := u.Name
    if u.OS != "" {
        userAgent = userAgent + "/" + u.OS
    }
    return userAgent
}
```

### 2.3 两种客户端能力描述方式

Navidrome 支持两种方式描述客户端播放能力：

#### 方式一：传统 Subsonic 方式（Legacy）

通过 `format` 和 `maxBitRate` 参数简单指定转码目标。

在 [legacy_client.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/stream/legacy_client.go#L15-L53) 的 `buildLegacyClientInfo` 中构建 `ClientInfo`：

- 如果指定了 `format`，创建对应格式的转码配置
- 如果只有 `maxBitRate` 且源文件比特率更高，使用 `DefaultDownsamplingFormat`（默认 opus）
- 如果都没有，则认为客户端支持所有格式（直接播放）

#### 方式二：OpenSubsonic 方式（Modern）

通过 `getTranscodeDecision` 接口 POST 完整的客户端能力描述。

在 [transcode.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/server/subsonic/transcode.go#L21-L112) 中定义：

**ClientInfo 数据结构** - [types.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/stream/types.go#L31-L41)：

```go
type ClientInfo struct {
    Name                       string
    Platform                   string
    MaxAudioBitrate            int          // 全局最大音频比特率
    MaxTranscodingAudioBitrate int          // 转码时的最大比特率
    DirectPlayProfiles         []DirectPlayProfile  // 直接播放配置
    TranscodingProfiles        []Profile            // 转码目标配置
    CodecProfiles              []CodecProfile       // 编解码器限制
}
```

---

## 3. 转码档位配置

### 3.1 Transcoding 模型

**转码配置数据结构** - [transcoding.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/model/transcoding.go#L3-L9)：

```go
type Transcoding struct {
    ID             string
    Name           string
    TargetFormat   string   // 目标格式（mp3, opus, aac, flac）
    Command        string   // ffmpeg 命令模板
    DefaultBitRate int      // 默认比特率 (kbps)
}
```

### 3.2 内置默认转码配置

默认转码配置在 [consts.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/consts/consts.go#L146-L176) 中定义：

| 格式 | 默认比特率 | 命令模板 |
|------|-----------|----------|
| mp3 | 192 kbps | `ffmpeg -ss %t -i %s -map 0:a:0 -b:a %bk -v 0 -f mp3 -` |
| opus | 128 kbps | `ffmpeg -ss %t -i %s -map 0:a:0 -b:a %bk -v 0 -c:a libopus -f opus -` |
| aac | 256 kbps | `ffmpeg -ss %t -i %s -map 0:a:0 -b:a %bk -v 0 -c:a aac -f adts -` |
| flac | 0（无损） | `ffmpeg -ss %t -i %s -map 0:a:0 -v 0 -c:a flac -f flac -` |

模板变量说明：
- `%s`：源文件路径
- `%t`：起始时间偏移（秒）
- `%b`：目标比特率（kbps）

### 3.3 转码命令查找机制

`LookupTranscodeCommand` 函数在 [decider.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/stream/decider.go#L328-L340) 中实现：

1. **优先从数据库查找**：用户自定义的转码配置
2. **回退到内置默认**：如果数据库中没有，使用 `consts.DefaultTranscodings`
3. **返回空字符串**：如果都没有找到，表示不支持该格式

默认比特率查找 `lookupDefaultBitrate` 遵循相同的优先级。

---

## 4. 转码决策流程

转码决策是整个系统的核心，由 `MakeDecision` 方法实现，位于 [decider.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/stream/decider.go#L40-L119)。

### 4.1 决策总体流程

```
开始
  ↓
构建源音频流详情 (buildSourceStream)
  ↓
检查全局比特率限制
  ├─ 超过 → 跳过直接播放检查
  └─ 未超过 → 逐一检查 DirectPlayProfiles
  ↓
可以直接播放？ ──是──→ 返回 CanDirectPlay=true
  ↓否
逐一检查 TranscodingProfiles
  ↓
找到匹配的转码配置？ ──是──→ 返回 CanTranscode=true
  ↓否
返回 ErrorReason="no compatible playback profile found"
```

### 4.2 源音频流详情构建

`buildSourceStream` 函数在 [decider.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/stream/decider.go#L122-L152)：

源数据优先顺序：
1. **ffprobe 探测数据**（如果可用）：最权威的音频参数
2. **标签元数据**：从音乐文件标签中解析的信息

**ffprobe 探测**（可选，由 `DevEnableMediaFileProbe` 控制）：
- 首次请求时运行 ffprobe 获取精确的编码信息
- 结果序列化为 JSON 存入 `media_file.probe_data` 字段
- 后续请求直接使用缓存的数据

### 4.3 直接播放检查

`checkDirectPlayProfile` 方法在 [decider.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/stream/decider.go#L204-L235)：

检查顺序：
1. **协议检查**：仅支持 `http` 协议
2. **容器检查**：使用 `matchesContainer`，支持别名（如 m4a/mp4/aac 视为同一组）
3. **编解码器检查**：使用 `matchesCodec`，支持别名
4. **声道数检查**：不能超过 `MaxAudioChannels`
5. **编解码器特定限制**：检查 `CodecProfiles` 中的限制

**容器/编解码器别名** 定义在 [aliases.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/stream/aliases.go)：

容器别名组：
- `aac, adts, m4a, mp4, m4b, m4p` → 规范名 `aac`
- `mpeg, mp3, mp2` → 规范名 `mpeg`
- `ogg, oga, opus` → 规范名 `ogg`
- `aif, aiff` → 规范名 `aif`

编解码器别名组：
- `aac, adts` → `aac`
- `ac3, ac-3` → `ac3`
- `eac3, e-ac3, e-ac-3, eac-3` → `eac3`

### 4.4 转码配置匹配

`computeTranscodedStream` 方法在 [decider.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/stream/decider.go#L241-L308)：

匹配流程：
1. **协议检查**：仅支持 `http`
2. **目标格式解析**：`resolveTargetFormat` 确定内部目标格式
   - 优先使用 `AudioCodec`（如 container=mp4 + audioCodec=aac → targetFormat=aac）
   - 没有 AudioCodec 时使用 Container
3. **转码命令存在性检查**：确认有对应的 ffmpeg 命令
4. **无损转换检查**：不允许有损 → 无损的转换
5. **初始化转码流参数**：继承源文件的采样率、声道数等
6. **编解码器固有限制调整**：
   - Opus 固定输出 48000Hz
   - MP3 最大 48000Hz 采样率、最大 2 声道
   - AAC 最大 96000Hz 采样率
   - Opus 最大 8 声道
7. **比特率计算**（见下节）
8. **Profile 声道数限制**：应用 `MaxAudioChannels`
9. **编解码器限制应用**：应用 `CodecProfiles` 中的限制

### 4.5 目标比特率计算

`computeBitrate` 方法在 [decider.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/stream/decider.go#L370-L396)：

根据源文件是否无损，采用不同策略：

**源文件是无损格式：**
- 目标也是无损 → 保留源比特率（不转换）
- 目标是有损 → 优先级：
  1. `MaxTranscodingAudioBitrate`（转码专用上限）
  2. `MaxAudioBitrate`（全局上限）
  3. 格式默认比特率（`lookupDefaultBitrate`）

**源文件是有损格式：**
- 直接使用源文件比特率（不升码）

**最终限制**：使用 `MaxAudioBitrate` 作为最终上限

### 4.6 编解码器限制应用

`applyCodecLimitations` 和 `applyLimitation` 在 [limitations.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/stream/limitations.go)：

支持的限制类型：
- `audioChannels`：声道数限制
- `audioBitrate`：比特率限制
- `audioSamplerate`：采样率限制
- `audioBitdepth`：位深限制
- `audioProfile`：音频配置文件（暂未完全实现）

比较运算符：
- `LessThanEqual`：≤（可向下调整）
- `GreaterThanEqual`：≥（不可向上调整，不满足则拒绝）
- `Equals`：等于（可向下找最接近的允许值）
- `NotEquals`：不等于

调整结果：
- `adjustNone`：已满足，无需调整
- `adjustAdjusted`：已调整到满足限制的值
- `adjustCannotFit`：无法满足，拒绝该配置

### 4.7 服务端强制转码

`applyServerOverride` 函数在 [decider.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/stream/decider.go#L156-L178)：

当 Player 关联了 `TranscodingId` 时，会用服务端配置覆盖客户端能力：
- 将 `MaxAudioBitrate` 和 `MaxTranscodingAudioBitrate` 设为转码配置的默认比特率
- 如果 Player 有 `MaxBitRate`，则使用该值
- 只保留一个 DirectPlayProfile（目标格式）
- 只保留一个 TranscodingProfile（目标格式）

这意味着客户端将被强制转码到指定格式。

---

## 5. 流式处理流水线

### 5.1 流请求入口

**Subsonic 传统流式接口** - [stream.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/server/subsonic/stream.go#L20-L54)：

```go
func (api *Router) Stream(w http.ResponseWriter, r *http.Request) {
    // 1. 解析参数: id, maxBitRate, format, timeOffset
    // 2. 获取媒体文件
    // 3. 调用 ResolveRequest 确定转码参数
    streamReq := api.transcodeDecision.ResolveRequest(ctx, mf, format, maxBitRate, timeOffset)
    // 4. 创建流
    stream, _ := api.streamer.NewStream(ctx, mf, streamReq)
    // 5. 输出到响应
    stream.Serve(ctx, w, r)
}
```

**OpenSubsonic 转码流接口** - [transcode.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/server/subsonic/transcode.go#L329-L404)：

```go
func (api *Router) GetTranscodeStream(w http.ResponseWriter, r *http.Request) {
    // 1. 解析参数: mediaId, mediaType, transcodeParams (JWT token)
    // 2. 从 token 中解析出转码参数
    streamReq, _ := api.transcodeDecision.ResolveRequestFromToken(ctx, token, mf, offset)
    // 3. 创建流并输出
    stream, _ := api.streamer.NewStream(ctx, mf, streamReq)
    stream.Serve(ctx, w, r)
}
```

### 5.2 Request 解析流程

`ResolveRequest` 方法在 [legacy_client.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/stream/legacy_client.go#L57-L113)：

1. 如果 format = "raw"，直接返回原始格式
2. 构建 Legacy ClientInfo
3. 应用服务端 Player 转码覆盖（如果有）
4. 应用 Player 的 MaxBitRate 限制（如果有）
5. 调用 `MakeDecision` 做出决策
6. 如果可以直接播放 → 返回 raw
7. 如果可以转码 → 返回转码参数
8. 如果都不行，尝试回退到 `DefaultDownsamplingFormat`
9. 最终回退到 raw

### 5.3 转码 Token 机制

`CreateTranscodeParams` 和 `ResolveRequestFromToken` 在 [token.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/stream/token.go)：

**Token 编码**：
- 将转码决策编码为 JWT Token
- 包含字段：mid (媒体ID), ua (更新时间), dp (直接播放), f (格式), b (比特率), ch (声道), sr (采样率), bd (位深)
- 有效期 48 小时

**Token 验证**：
- 验证 JWT 签名
- 验证 mediaID 匹配
- 验证源文件更新时间（防止文件变更后使用旧的转码缓存）
- 失效则返回 `ErrTokenStale`

### 5.4 MediaStreamer 流创建

`NewStream` 方法在 [media_streamer.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/stream/media_streamer.go#L66-L133)：

**原始流（Raw）**：
- 直接打开文件
- 支持 seek（文件本身支持随机访问）

**转码流（Transcoded）**：
- 通过转码缓存获取
- 缓存 key 包含：文件ID + 更新时间 + 比特率 + 采样率 + 位深 + 声道 + 格式 + 偏移
- 支持并发限制

### 5.5 转码缓存机制

转码缓存使用 `cache.FileCache`，在 [media_streamer.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/stream/media_streamer.go#L221-L280) 的 `NewTranscodingCache` 中配置：

缓存生产流程：
1. 查找转码命令（`LookupTranscodeCommand`）
2. 获取并发许可（`TranscodeLimiter.Acquire`）
3. 调用 `ffmpeg.Transcode` 启动转码进程
4. 返回包装了 release 函数的 `releasingReadCloser`

**缓存键组成**：
```
{mediaID}.{updatedAt}.{bitrate}.{sampleRate}.{bitDepth}.{channels}.{format}.{offset}
```

### 5.6 并发转码限制

`TranscodeLimiter` 在 [limiter.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/stream/limiter.go)：

两种限制：
- **全局限制**：`Transcoding.MaxConcurrent`（保护服务器）
- **每用户限制**：`Transcoding.MaxConcurrentPerUser`（公平性）

特性：
- 非阻塞：超过限制立即返回 `ErrTooManyTranscodes`
- 匿名请求（公共分享）不占用 per-user 配额
- limiter 启用时，ffmpeg 进程绑定到请求 context，客户端断开即终止

---

## 6. ffmpeg 转码命令生成

### 6.1 两种命令构建方式

`Transcode` 方法在 [ffmpeg.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/ffmpeg/ffmpeg.go#L75-L89)：

根据命令是否为默认命令，选择不同的构建方式：

```go
if isDefaultCommand(opts.Format, opts.Command) {
    args = buildDynamicArgs(opts)  // 动态参数构建
} else {
    args = buildTemplateArgs(opts) // 模板替换构建
}
```

### 6.2 动态参数构建（默认命令）

`buildDynamicArgs` 在 [ffmpeg.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/ffmpeg/ffmpeg.go#L395-L423)：

为已知格式程序化构建参数，能够精确控制所有转码参数：

```
ffmpeg
  -ss {offset}              # 起始偏移（如果有）
  -i {filePath}             # 输入文件
  -map 0:a:0                # 选择第一个音频流
  -c:a {codec}              # 音频编码器
  -b:a {bitrate}k           # 比特率（如果有）
  -ar {sampleRate}          # 采样率（如果有）
  -ac {channels}            # 声道数（如果有）
  -sample_fmt {sample_fmt}  # 采样格式（仅无损格式）
  -v 0                      # 静默输出
  -f {outputFormat}         # 输出格式
  -                         # 输出到 stdout
```

格式与编码器映射：
| 格式 | 编码器 | 输出格式 |
|------|--------|----------|
| mp3 | libmp3lame | mp3 |
| opus | libopus | opus |
| aac | aac | adts |
| flac | flac | flac |

### 6.3 模板参数构建（自定义命令）

`buildTemplateArgs` 在 [ffmpeg.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/ffmpeg/ffmpeg.go#L430-L433)：

用户自定义命令使用模板替换，但仍会注入动态音频参数：

1. 先用 `createFFmpegCommand` 替换 `%s`、`%t`、`%b` 模板变量
2. 再用 `injectDynamicAudioFlags` 在输出前注入 `-ar`、`-ac`、`-sample_fmt`

`injectBeforeOutput` 函数会在末尾的 `-`（stdout 标记）之前插入参数，确保参数生效。

### 6.4 位深与采样格式

`bitDepthToSampleFmt` 在 [ffmpeg.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/ffmpeg/ffmpeg.go#L478-L488)：

仅无损输出格式才设置 `-sample_fmt`：
- 16 位 → `s16`
- 32 位 → `s32`
- 24 位及其他 → `s32`（FLAC 只支持 s16 和 s32，24 位打包在 32 位容器中）

### 6.5 进程管理

`ffCmd` 结构体在 [ffmpeg.go](file:///d:/fz/0601-2/solo-dogfeeding/code/19-navidrome/core/ffmpeg/ffmpeg.go#L297-L341)：

- 使用 `exec.CommandContext` 启动进程，支持 context 取消
- stdout 通过 `io.Pipe` 连接到输出流
- stderr 收集到有限缓冲区（4KB），用于错误诊断
- `wait()` goroutine 等待进程结束，关闭管道

---

## 7. 完整调用链路示例

### 7.1 传统 Subsonic 流请求

```
GET /rest/stream.view?id=xxx&maxBitRate=128&format=mp3
  ↓
getPlayer 中间件
  → players.Register() → 识别/注册 Player
  → 注入 Player 和 Transcoding 到 context
  ↓
Stream handler
  → transcodeDecision.ResolveRequest()
    → buildLegacyClientInfo(mf, "mp3", 128)
    → applyServerOverride() 或 player.MaxBitRate 限制
    → MakeDecision()
      → buildSourceStream()
      → 检查 DirectPlayProfiles（不匹配）
      → 检查 TranscodingProfiles
        → computeTranscodedStream()
          → resolveTargetFormat() → "mp3"
          → LookupTranscodeCommand("mp3") → 找到命令
          → 计算目标比特率 → 128 kbps
          → 应用编解码器限制
      → 返回 decision: CanTranscode=true, TargetFormat="mp3", TargetBitrate=128
    → 返回 Request{Format:"mp3", BitRate:128, ...}
  ↓
streamer.NewStream()
  → 转码缓存 Get(job)
    → 缓存未命中 → 生产
      → limiter.Acquire() → 获取并发许可
      → ffmpeg.Transcode() → 启动 ffmpeg 进程
    → 返回缓存流
  ↓
stream.Serve() → 写入 HTTP 响应
```

### 7.2 OpenSubsonic 转码决策 + 流

```
POST /rest/getTranscodeDecision.view?mediaId=xxx&mediaType=song
Body: { name: "MyClient", directPlayProfiles: [...], transcodingProfiles: [...] }
  ↓
GetTranscodeDecision handler
  → 解析并验证 ClientInfo
  → MakeDecision(mf, clientInfo)
    → 详细决策过程（同上）
  → CreateTranscodeParams(decision) → JWT token
  ↓
返回 decision + token

GET /rest/getTranscodeStream.view?mediaId=xxx&transcodeParams={token}
  ↓
GetTranscodeStream handler
  → ResolveRequestFromToken(token)
    → 验证 JWT
    → 验证 mediaID 匹配
    → 验证源文件未变更
    → 返回 Request
  → NewStream + Serve（同上）
```

---

## 8. 关键设计要点总结

1. **两级决策**：先尝试直接播放，不行再尝试转码，符合客户端能力优先原则
2. **Profile 顺序优先**：DirectPlayProfiles 和 TranscodingProfiles 按顺序匹配，第一个匹配的生效
3. **无损保护**：不允许有损 → 无损的无意义转换
4. **不升码原则**：有损源文件转码时使用源比特率，不提升质量
5. **配置优先级**：数据库自定义 > 内置默认 > 硬编码回退
6. **服务端强制**：Player 关联 Transcoding 时可强制转码到指定格式
7. **并发保护**：全局 + 每用户两级转码并发限制
8. **缓存机制**：转码结果缓存，相同参数重复利用
9. **渐进式降级**：格式不支持时尝试回退到默认下采样格式，最终回退到原始格式
