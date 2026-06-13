# Navidrome 用户偏好与播放器运行状态同步及跨会话恢复机制

## 概览

Navidrome 采用**前端 localStorage + 后端数据库**双层持久化策略，通过 Redux 状态管理将两者串联。核心思想是：

- **用户偏好**（主题、语言、通知等）主要存储在前端 `localStorage`，通过 Redux reducer 驱动变更并定期序列化保存
- **播放器运行状态**（播放队列、当前曲目、音量）同样存储在 `localStorage`，同时可选地将队列保存为后端数据库记录
- **后端数据库**存储用户属性（`user_props` 表）、播放器配置（`player` 表）、播放队列（`playqueue` 表）等持久数据
- 跨会话恢复时，先从 `localStorage` 加载序列化状态重建 Redux store，再按需从后端获取补充数据

---

## 一、前端状态持久化机制

### 1.1 Redux Store 创建与状态加载

入口文件 [createAdminStore.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/store/createAdminStore.js#L14-L75) 负责创建 Redux store：

```js
const persistedState = loadState()
if (persistedState?.player?.savedPlayIndex) {
  persistedState.player.playIndex = persistedState.player.savedPlayIndex
}
const store = createStore(resettableAppReducer, persistedState, ...)
```

- 调用 [persistState.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/store/persistState.js#L1-L20) 中的 `loadState()` 从 `localStorage.getItem('state')` 反序列化整个状态树
- 特别处理：如果存在 `savedPlayIndex`，则将其恢复为 `playIndex`，使播放器知道上次播放到了哪首歌
- 用户登出时（`USER_LOGOUT` action），reducer 返回 `undefined`，清除所有状态

### 1.2 状态保存策略

[createAdminStore.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/store/createAdminStore.js#L55-L71) 中使用 `lodash.throttle` 节流保存（间隔 1000ms），仅保存指定的状态切片：

```js
saveState({
  theme: state.theme,
  library: state.library,
  player: (({ queue, volume, savedPlayIndex }) => ({
    queue,
    volume,
    savedPlayIndex,
  }))(state.player),
  albumView: state.albumView,
  settings: state.settings,
})
```

| 状态切片 | 持久化字段 | 说明 |
|---------|-----------|------|
| `theme` | 完整值 | 当前主题 ID（如 `DarkTheme`、`AUTO_THEME_ID`） |
| `library` | 完整值 | 用户可用库列表 + 已选库 ID 列表 |
| `player` | `queue` / `volume` / `savedPlayIndex` | 播放队列、音量、当前播放索引（**不保存** `current`、`clear`、`playIndex` 等瞬态字段） |
| `albumView` | 完整值 | 专辑网格/列表视图模式 |
| `settings` | 完整值 | 通知开关、可切换字段、隐藏字段 |

**注意**：`replayGain`、`transcoding`、`dialog`、`activity` 等状态切片**不在此处保存**，它们有各自独立的持久化方式。

---

## 二、各偏好项的存储与同步

### 2.1 主题偏好 (Theme)

- **Reducer**: [themeReducer.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/reducers/themeReducer.js#L17-L25)
- **Action**: `CHANGE_THEME`，由 [SelectTheme.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/personal/SelectTheme.jsx) 触发
- **存储位置**: 通过 `saveState` 序列化到 `localStorage.state.theme`
- **恢复**: 应用启动时从 `localStorage.state.theme` 加载，无值时使用 `config.defaultTheme` 决定默认值

### 2.2 语言偏好 (Language)

- **存储位置**: 独立存储在 `localStorage.getItem('locale')`
- **设置**: [SelectLanguage.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/personal/SelectLanguage.jsx) 中 `setLocale()` 后手动写入 `localStorage.setItem('locale', ...)`
- **恢复**: [App.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/App.jsx#L168) 中 `let language = localStorage.getItem('locale') || 'en'`

### 2.3 默认视图偏好 (Default View)

- **存储位置**: 独立存储在 `localStorage.getItem('defaultView')`
- **设置**: [SelectDefaultView.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/personal/SelectDefaultView.jsx) 中直接 `localStorage.setItem('defaultView', ...)`
- **恢复**: 同组件读取 `localStorage.getItem('defaultView') || defaultAlbumList`

### 2.4 通知偏好 (Notifications)

- **Reducer**: [settingsReducer.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/reducers/settingsReducer.js#L13-L20)
- **存储位置**: 通过 `saveState` 序列化到 `localStorage.state.settings.notifications`
- **设置**: [NotificationsToggle.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/personal/NotificationsToggle.jsx) 调用 `dispatch(setNotificationsState(...))`
- **特殊逻辑**: 如果浏览器权限被拒绝或不是安全上下文，自动关闭通知

### 2.5 ReplayGain 偏好

- **Reducer**: [replayGainReducer.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/reducers/replayGainReducer.js#L1-L48)
- **存储位置**: **不通过** `saveState` 统一保存，而是 reducer 内部自行读写 `localStorage`：
  - `gainMode` → `localStorage.getItem('gainMode')` / `localStorage.setItem('gainMode', ...)`
  - `preAmp` → `localStorage.getItem('preAmp')` / `localStorage.setItem('preAmp', ...)`
- **恢复**: reducer 的 `initialState` 直接从 `localStorage` 读取

### 2.6 音乐库选择偏好 (Library Selection)

- **Reducer**: [libraryReducer.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/reducers/libraryReducer.js#L8-L52)
- **存储位置**: 通过 `saveState` 序列化到 `localStorage.state.library`
- **恢复**: 启动时加载，并通过 [wrapperDataProvider.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/dataProvider/wrapperDataProvider.js#L12-L33) 中 `getSelectedLibraries()` 从 `localStorage.state` 读取，在数据请求时附加 `library_id` 过滤

### 2.7 专辑视图模式 (Album View)

- **Reducer**: [albumView.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/reducers/albumView.js#L3-L17)
- **存储位置**: 通过 `saveState` 序列化到 `localStorage.state.albumView`
- **恢复**: 启动时加载

### 2.8 可切换/隐藏字段偏好 (Toggleable/Omitted Fields)

- **Reducer**: [settingsReducer.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/reducers/settingsReducer.js#L21-L39)
- **存储位置**: 通过 `saveState` 序列化到 `localStorage.state.settings.toggleableFields` / `omittedFields`
- **恢复**: 启动时加载

### 2.9 Last.fm / ListenBrainz Scrobble 开关

这两个偏好**不存储在前端**，而是通过后端 API 管理：
- **Last.fm**: [LastfmScrobbleToggle.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/personal/LastfmScrobbleToggle.jsx) 调用 `/api/lastfm/link` API，链接状态存储在后端 `user_props` 表
- **ListenBrainz**: [ListenBrainzScrobbleToggle.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/personal/ListenBrainzScrobbleToggle.jsx) 调用 `/api/listenbrainz/link` API，状态同样存储在后端
- **恢复**: 每次打开 Personal 页面时通过 `useEffect` 向后端查询当前链接状态

---

## 三、播放器运行状态同步

### 3.1 前端播放器状态管理

[playerReducer.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/reducers/playerReducer.js#L18-L24) 的初始状态：

```js
const initialState = {
  queue: [],          // 当前播放队列
  current: {},        // 当前播放曲目信息
  clear: false,       // 是否需要清空旧队列
  volume: config.defaultUIVolume / 100,  // 音量
  savedPlayIndex: 0,  // 已确认的播放位置索引
}
```

关键状态字段区分：

| 字段 | 是否持久化 | 说明 |
|------|-----------|------|
| `queue` | ✅ | 完整播放队列 |
| `volume` | ✅ | 音量（0-1） |
| `savedPlayIndex` | ✅ | 上次确认播放的曲目索引 |
| `current` | ❌ | 当前播放曲目瞬态信息 |
| `clear` | ❌ | 队列切换瞬态标志 |
| `playIndex` | ❌ | 待切换的曲目索引（瞬态） |
| `mode` | ❌ | 播放模式（循环/随机等） |

### 3.2 播放状态同步流程

[Player.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/audioplayer/Player.jsx) 是播放器核心组件，负责与 `navidrome-music-player` 库交互：

1. **队列同步** (`PLAYER_SYNC_QUEUE`): 当音乐播放器内部队列发生变化时，通过 `onAudioListsChange` 回调触发 `syncQueue` action，将播放器内部状态同步回 Redux。特别注意 [playerReducer.js#L166-L182](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/reducers/playerReducer.js#L166-L182) 中的**待切换保护**逻辑——如果用户已请求切换曲目但播放器尚未确认，则保留 `playIndex` 和 `clear` 标志。

2. **当前曲目确认** (`PLAYER_CURRENT`): 当播放器确认切换到新曲目时，通过 `onAudioPlay` 回调触发 `currentPlaying` action。[reduceCurrent](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/reducers/playerReducer.js#L184-L202) 更新 `savedPlayIndex` 并清除待切换标志。

3. **音量同步** (`PLAYER_SET_VOLUME`): 通过 `onAudioVolumeChange` 回调触发，注意音量使用 `Math.sqrt` 补偿对数关系。

4. **播放模式** (`PLAYER_SET_MODE`): 通过 `onPlayModeChange` 回调触发 `setPlayMode` action，但**不持久化**。

### 3.3 播放报告 (Playback Reporting)

[Player.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/audioplayer/Player.jsx) 通过 [subsonic/index.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/subsonic/index.js#L44-L58) 的 `reportPlayback` API 向后端报告播放状态：

| 事件 | 报告状态 | 说明 |
|------|---------|------|
| 新曲目开始 | `starting` → `playing` | 首次播放某曲目 |
| 正在播放（心跳） | `playing` | 每 `playbackReportIntervalMs`（默认60秒）报告一次 |
| 暂停 | `paused` | 用户暂停 |
| 跳转完成 | `paused`/`playing` | Seek 后报告当前位置 |
| 曲目结束 | `stopped` | 自然播放结束 |
| 曲目切换 | `stopped` | 切换到下一首时报告前一首结束 |
| 页面关闭 | `stopped` | 通过 `pagehide` 事件 + `keepalive: true` 的 fetch 发送 |

---

## 四、跨会话恢复机制

### 4.1 前端恢复流程

当用户重新打开浏览器时，恢复流程如下：

```
1. App.jsx 启动
   └→ createAdminStore() 
      ├→ loadState() 从 localStorage 读取序列化状态
      ├→ 恢复 savedPlayIndex → playIndex
      └→ 创建 Redux store（preloadedState = persistedState）

2. Player 组件挂载
   ├→ 从 playerStateRef.current 读取恢复的 queue 和 savedPlayIndex
   ├→ detectBrowserProfile() 检测浏览器编解码能力
   ├→ 为当前及后续 3 首曲目预解析 transcode URL
   └→ dispatch(refreshQueue(resolvedUrls)) 刷新队列中的流 URL

3. PLAYER_REFRESH_QUEUE reducer 处理
   ├→ 用解析后的 URL 替换队列中各曲目的 musicSrc
   ├→ 设置 clear: true（触发播放器重新加载队列）
   ├→ 设置 autoPlay: false（不自动播放）
   └→ 设置 playIndex = savedPlayIndex（恢复到上次位置）
```

关键代码在 [Player.jsx#L81-L111](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/audioplayer/Player.jsx#L81-L111)：

```js
useEffect(() => {
  const profile = detectBrowserProfile()
  decisionService.setProfile(profile)
  dispatch(setTranscodingProfile(profile))

  const state = playerStateRef.current
  const currentIdx = state.savedPlayIndex || 0
  const trackIds = state.queue
    .slice(currentIdx, currentIdx + 4)
    .filter((item) => !item.isRadio && item.trackId)
    .map((item) => item.trackId)

  if (trackIds.length === 0) {
    dispatch(refreshQueue())
    return
  }

  Promise.allSettled(
    trackIds.map((id) =>
      decisionService.resolveStreamUrl(id).then((url) => [id, url])
    )
  ).then((results) => {
    const resolvedUrls = {}
    results.forEach((r) => {
      if (r.status === 'fulfilled') {
        resolvedUrls[r.value[0]] = r.value[1]
      }
    })
    dispatch(refreshQueue(resolvedUrls))
  })
}, [dispatch])
```

这里的关键设计：localStorage 中保存的队列包含 `trackId` 但 **不包含有效的流 URL**（因为 URL 含有时效性 token），所以恢复时必须重新解析。

### 4.2 后端播放队列持久化

后端提供两套 API 保存/恢复播放队列：

#### Native API (`/api/queue`)

[nativeapi/queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/server/nativeapi/queue.go#L184-L191) 注册了四个端点：

| 方法 | 路径 | Handler | 说明 |
|------|------|---------|------|
| GET | `/api/queue` | `getQueue` | 获取当前用户播放队列（含完整 MediaFile 信息） |
| POST | `/api/queue` | `saveQueue` | 整体替换保存队列 |
| PUT | `/api/queue` | `updateQueue` | 部分更新队列（可只更新 ids / current / position） |
| DELETE | `/api/queue` | `clearQueue` | 清空队列 |

`updateQueue` 支持**部分更新**：只需传递要更新的字段（`ids`、`current`、`position`），未传递的字段保持不变，通过 `colNames` 参数控制数据库更新哪些列。

#### Subsonic API (`/rest/savePlayQueue`, `/rest/getPlayQueue`)

[bookmarks.go](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/server/subsonic/bookmarks.go) 提供兼容 Subsonic 协议的播放队列 API：

- `SavePlayQueue`: 通过 `current` ID（歌曲 ID）指定当前曲目
- `SavePlayQueueByIndex`: 通过 `currentIndex`（整数索引）指定当前曲目
- `GetPlayQueue` / `GetPlayQueueByIndex`: 获取队列

#### 数据库持久化

[playqueue_repository.go](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/persistence/playqueue_repository.go#L39-L77) 中 `Store` 方法的逻辑：

- 每个用户只有一条播放队列记录（`user_id` 唯一）
- 无 `colNames` 参数时：先删除旧记录，再插入新记录（整体替换）
- 有 `colNames` 参数时：仅更新指定列（部分更新）
- `items` 字段以逗号分隔的 ID 字符串存储
- `RetrieveWithMediaFiles` 会根据存储的 ID 列表从 `media_file` 表加载完整歌曲信息

### 4.3 播放队列保存为播放列表

[SaveQueueDialog.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/dialogs/SaveQueueDialog.jsx) 提供了将当前播放队列保存为播放列表的功能：

1. 先通过 `dataProvider.create('playlist', ...)` 创建新播放列表
2. 再通过 `dataProvider.create('playlistTrack', ...)` 添加所有曲目 ID
3. 这是用户主动触发的操作，不是自动同步

### 4.4 播放器配置 (Player) 的持久化

[player.go](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/model/player.go) 定义了 `Player` 模型，包含：

| 字段 | 说明 |
|------|------|
| `TranscodingId` | 关联的转码配置 |
| `MaxBitRate` | 最大比特率限制 |
| `ReportRealPath` | 是否报告真实路径 |
| `ScrobbleEnabled` | 是否启用 Scrobble |
| `Client` / `UserAgent` | 客户端标识，用于匹配 |

[players.go](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/core/players.go#L34-L78) 中的 `Register` 逻辑：

1. 如果请求携带 `playerID`，先按 ID 查找
2. 若 ID 对应的 player 的 `Client` 不匹配，则忽略该 ID
3. 尝试按 `(userId, client, userAgent)` 组合查找已有 player
4. 若均未找到，创建新 player 记录
5. 更新 `LastSeen`、`IP`、`UserAgent` 等信息并保存（有频率限制）

### 4.5 用户属性 (UserProps) 的持久化

[user_props_repository.go](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/persistence/user_props_repository.go) 实现了简单的 key-value 存储：

- 表名：`user_props`
- 联合主键：`(user_id, key)`
- `Put` 方法：先尝试 UPDATE，若无匹配行则 INSERT（upsert 语义）
- 用于存储 Last.fm session key、ListenBrainz token 等后端管理的偏好

[session_keys.go](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/core/agents/session_keys.go) 是 `UserPropsRepository` 的封装，为第三方服务（Last.fm、ListenBrainz）提供 session key 的存取。

---

## 五、认证状态与用户信息的持久化

[authProvider.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/authProvider.js) 管理 LDAP 认证相关信息的持久化：

| localStorage Key | 说明 |
|-----------------|------|
| `token` | JWT 认证令牌 |
| `userId` | 用户 ID |
| `name` | 用户显示名称 |
| `username` | 用户名 |
| `avatar` | 头像 URL |
| `role` | 角色（`admin` / `regular`） |
| `subsonic-salt` | Subsonic API 认证盐 |
| `subsonic-token` | Subsonic API 认证令牌 |
| `is-authenticated` | 认证标志 |
| `locale` | 语言偏好 |
| `defaultView` | 默认视图 |

[httpClient.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/dataProvider/httpClient.js) 在每次 HTTP 请求中：
- 从 `localStorage` 读取 JWT token 附加到 `X-ND-Authorization` 头
- 附加 `X-ND-Client-Unique-Id` 头（UUID，页面加载时生成）
- 从响应头中检查并更新 JWT token（自动刷新）

---

## 六、实时事件同步

[eventStream.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/eventStream.js) 通过 SSE (Server-Sent Events) 连接 `/api/events`，监听以下事件：

| 事件类型 | 说明 |
|---------|------|
| `serverStart` | 服务器重启 |
| `scanStatus` | 扫描状态更新 |
| `refreshResource` | 资源变更通知 |
| `nowPlayingCount` | 正在播放计数 |
| `keepAlive` | 心跳 |

当连接断开时自动重连（5秒延迟），重连成功后分发 `streamReconnected` 事件触发数据刷新。

---

## 七、整体数据流图

```
┌──────────────────────────────────────────────────────────────────┐
│                         Browser (localStorage)                   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │  localStorage.key = "state"                              │     │
│  │  {                                                       │     │
│  │    theme: "DarkTheme",                                   │     │
│  │    library: { userLibraries: [...], selectedLibraries: [...] }, │
│  │    player: { queue: [...], volume: 0.8, savedPlayIndex: 3 }, │
│  │    albumView: { grid: true },                            │     │
│  │    settings: { notifications: true, toggleableFields: {}, omittedFields: {} } │
│  │  }                                                       │     │
│  └─────────────────────────────────────────────────────────┘     │
│                                                                  │
│  ┌──────────────────────┐  ┌─────────────────────────┐          │
│  │  localStorage keys:  │  │  localStorage keys:     │          │
│  │  "gainMode"          │  │  "token", "userId",     │          │
│  │  "preAmp"            │  │  "username", "role",    │          │
│  │  "locale"            │  │  "subsonic-token",      │          │
│  │  "defaultView"       │  │  "subsonic-salt",       │          │
│  └──────────────────────┘  │  "is-authenticated"     │          │
│                             └─────────────────────────┘          │
└──────────────────────┬───────────────────────────────────────────┘
                       │ HTTP / SSE
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                      Backend (SQLite Database)                    │
│                                                                  │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────────┐  │
│  │  player      │  │  playqueue    │  │  user_props            │  │
│  │  ──────────  │  │  ───────────  │  │  ──────────────────    │  │
│  │  id          │  │  id           │  │  user_id + key (PK)    │  │
│  │  user_id     │  │  user_id      │  │  value                 │  │
│  │  client      │  │  current      │  │                        │  │
│  │  user_agent  │  │  position     │  │  (Last.fm session,     │  │
│  │  transcoding │  │  items (CSV)  │  │   ListenBrainz token,  │  │
│  │  max_bit_rate│  │  changed_by   │  │   ...)                 │  │
│  │  scrobble    │  │               │  │                        │  │
│  └─────────────┘  └──────────────┘  └────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 八、总结

Navidrome 的偏好与状态同步机制具有以下特点：

1. **前端主导的偏好管理**：大部分用户偏好（主题、语言、通知等）仅存储在浏览器 `localStorage`，不与后端同步。这意味着不同浏览器/设备的偏好相互独立。

2. **双层播放状态持久化**：播放队列和位置既保存在 `localStorage`（用于同浏览器快速恢复），也可通过后端 API 保存到数据库（用于跨设备或通过 Subsonic 兼容客户端访问）。

3. **细粒度状态区分**：Redux 中明确区分持久化字段（`savedPlayIndex`）和瞬态字段（`playIndex`、`clear`），避免将瞬态数据写入 `localStorage` 导致恢复异常。

4. **URL 时效性处理**：恢复播放队列时，由于流 URL 含有时效性 token，会重新通过 `decisionService.resolveStreamUrl()` 解析，确保可以正常播放。

5. **后端管理的偏好**：Last.fm / ListenBrainz 等第三方服务集成状态由后端 `user_props` 表管理，前端仅做展示和触发，确保敏感信息不暴露在 `localStorage`。

6. **登出清除**：`USER_LOGOUT` action 通过 `resettableAppReducer` 将所有 Redux 状态重置为 `undefined`，同时 `authProvider.logout()` 清除认证相关的 `localStorage` 条目，但**不清除** `localStorage.state`（下次登录仍可恢复偏好）。
