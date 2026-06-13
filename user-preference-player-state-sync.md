# Navidrome 用户偏好与播放器运行状态同步及跨会话恢复机制

## 概览

Navidrome 采用**前端 localStorage + 后端数据库**双层持久化策略，通过 Redux 状态管理将两者串联。核心要点：

- **用户偏好**（主题、语言、通知等）主要存储在前端 `localStorage`，通过 Redux reducer 驱动变更并节流序列化保存
- **播放器运行状态**（播放队列、当前曲目、音量）同样存储在 `localStorage`，同时可选通过后端 API 将队列保存到数据库
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
- ⚠️ **savedPlayIndex=0 的边界问题**：代码使用 `if (persistedState?.player?.savedPlayIndex)`，当 `savedPlayIndex === 0` 时该判断为 falsy，不会将 `playIndex` 设为 0。而 `PLAYER_REFRESH_QUEUE` reducer 使用 `savedPlayIndex >= 0` 判断可以正确处理 0。因此**从 localStorage 恢复时，如果上次停在第 1 首歌（index=0），`playIndex` 将不会被立即设置，只有当 `refreshQueue` 被 dispatch 后才会被设回 0**。
- 用户登出时（`USER_LOGOUT` action），`resettableAppReducer` 接收 `undefined` 作为 state，触发所有子 reducer 返回其初始值。

### 1.2 状态保存策略（节流间隔与登出写入）

[createAdminStore.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/store/createAdminStore.js#L55-L71) 中使用 `lodash.throttle`（**1000ms** 默认 leading+trailing）包装 `store.subscribe` 回调：

```js
store.subscribe(
  throttle(() => {
    const state = store.getState()
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
  }),
  1000,
)
```

⚠️ **登出后 localStorage 写入**：当 `USER_LOGOUT` 被 dispatch，`resettableAppReducer` 返回所有子 reducer 的初始值，store 状态变为初始值树。`store.subscribe` 会触发 throttle 包装的回调，将初始值序列化写入 `localStorage.state`。例如 `player.savedPlayIndex` 会被写回默认值 `0`，`player.queue` 会被写回 `[]`。因此登出操作**实际上会覆盖 localStorage.state**（将其重置为各子 reducer 的初始状态），而 authProvider 的 `removeItems()` 仅清除认证相关条目。

| 状态切片 | 持久化字段 | 说明 |
|---------|-----------|------|
| `theme` | 完整值 | 当前主题 ID（如 `DarkTheme`、`AUTO_THEME_ID`） |
| `library` | 完整值 | 用户可用库列表 + 已选库 ID 列表 |
| `player` | `queue` / `volume` / `savedPlayIndex` | 播放队列、音量、当前播放索引（**不保存** `current`、`clear`、`playIndex`、`mode` 等瞬态字段） |
| `albumView` | 完整值 | 专辑网格/列表视图模式 |
| `settings` | 完整值 | 通知开关、可切换字段、隐藏字段 |

### 1.3 loadState/saveState 的异常吞没

[persistState.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/store/persistState.js#L1-L20) 中两个函数都使用 `try/catch` 静默吞掉所有异常：

```js
export const loadState = () => {
  try {
    const serializedState = localStorage.getItem('state')
    if (serializedState === null) return undefined
    return JSON.parse(serializedState)
  } catch (err) {
    return undefined   // JSON parse 失败或 localStorage 不可用时，静默返回 undefined
  }
}

export const saveState = (state) => {
  try {
    localStorage.setItem('state', JSON.stringify(state))
  } catch (err) {
    // Ignore write errors — 例如隐私模式下 localStorage 不可写
  }
}
```

没有任何日志或错误上报，若 localStorage 损坏或浏览器隐私模式，状态会静默丢失。

### 1.4 多标签 storage 事件

**代码中不存在任何 `window.addEventListener('storage', ...)` 监听**。这意味着在同一个浏览器的多个标签页中打开 Navidrome 时，**偏好和播放器状态不会跨标签页实时同步**。每个标签页维护独立的 Redux store 和内存状态，仅在各自的 store 变化时写入 localStorage，但不会读取其他标签页写入的变更。

---

## 二、各偏好项的存储与同步

### 2.1 主题偏好 (Theme)

- **Reducer**: [themeReducer.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/reducers/themeReducer.js#L17-L25)
- **Action**: `CHANGE_THEME`，由 [SelectTheme.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/personal/SelectTheme.jsx) 触发
- **存储位置**: 通过 `saveState` 序列化到 `localStorage.state.theme`
- **恢复**: 应用启动时从 `localStorage.state.theme` 加载，无值时使用 `config.defaultTheme`

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
  queue: [],
  current: {},
  clear: false,
  volume: config.defaultUIVolume / 100,
  savedPlayIndex: 0,
}
```

关键状态字段区分：

| 字段 | 是否持久化 | 说明 |
|------|-----------|------|
| `queue` | ✅ | 完整播放队列 |
| `volume` | ✅ | 音量（0-1） |
| `savedPlayIndex` | ✅ | 已确认的播放位置索引 |
| `current` | ❌ | 当前播放曲目瞬态信息 |
| `clear` | ❌ | 队列切换瞬态标志 |
| `playIndex` | ❌ | 待切换的曲目索引（瞬态） |
| `mode` | ❌ | 播放模式（循环/随机等），不持久化 |

### 3.2 播放状态同步流程

[Player.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/audioplayer/Player.jsx) 是播放器核心组件，负责与 `navidrome-music-player` 库交互：

1. **队列同步** (`PLAYER_SYNC_QUEUE`): 当音乐播放器内部队列发生变化时，通过 `onAudioListsChange` 回调触发 `syncQueue` action。[reduceSyncQueue](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/reducers/playerReducer.js#L166-L182) 中的**待切换保护**逻辑——如果 `playIndex` 已设置且（`clear` 为 true 或 `playIndex !== savedPlayIndex`），则保留待切换的 `playIndex` 和 `clear` 标志。此逻辑专门处理 `playIndex === savedPlayIndex === 0` 的边界（例如关闭播放器后从第 1 首播放新专辑）。

2. **当前曲目确认** (`PLAYER_CURRENT`): 当播放器确认切换到新曲目时，通过 `onAudioPlay` 回调触发 `currentPlaying` action。[reduceCurrent](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/reducers/playerReducer.js#L184-L202) 更新 `savedPlayIndex` 并清除待切换标志；若 `savedPlayIndex !== playIndex` 则认为仍有待切换，不覆盖。

3. **音量同步** (`PLAYER_SET_VOLUME`): 通过 `onAudioVolumeChange` 回调触发，注意音量使用 `Math.sqrt` 补偿对数关系。

4. **播放模式** (`PLAYER_SET_MODE`): 通过 `onPlayModeChange` 回调触发 `setPlayMode` action，但**不持久化**。

### 3.3 播放报告 (Playback Reporting)

[Player.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/audioplayer/Player.jsx) 通过 [subsonic/index.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/subsonic/index.js#L44-L58) 的 `reportPlayback` API 向后端 `ReportPlayback` 端点报告播放状态：

| 前端事件 | 报告 state | 说明 |
|---------|-----------|------|
| 新曲目开始（onAudioPlay + isNewTrack） | `starting` → `playing` | 先报告 starting，成功后立即报告 playing |
| 恢复播放（onAudioPlay + !isNewTrack） | `playing` | 同一首歌从暂停恢复 |
| 定时心跳（useInterval） | `playing` | 每 `config.playbackReportIntervalMs`（默认 60000ms = 60 秒）报告一次，仅当有 `heartbeatTrackId` 且未停止时 |
| 暂停（onAudioPause） | `paused` | 清除 heartbeat |
| Seek（onAudioSeeked） | `paused` / `playing` | 根据 audioInstance 当前状态 |
| 曲目自然结束（onAudioEnded） | `stopped` | 使用 `duration * 1000` 作为 positionMs |
| 切换曲目（onAudioPlayTrackChange） | `stopped` | 对旧曲目报告 stopped |
| 页面关闭/隐藏（pagehide） | `stopped` | 通过 `reportPlaybackKeepalive` 发送（见 3.4 节） |
| 销毁播放器（onBeforeDestroy） | `stopped` | 对当前曲目报告 stopped |

#### 后端语义：[play_tracker.go ReportPlayback](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/core/scrobbler/play_tracker.go#L261-L382)

后端按 state 分发：

| state | 行为 |
|-------|------|
| `starting` | 查库获取 `MediaFile`，在 `playMap` 缓存中新建 `PlaybackSession`（TTL = 剩余时长+5秒），入队 `PlaybackReport` 与 `NowPlaying`。`start = now` |
| `playing` / `paused` | 从 `playMap` 按 `clientId` 取现存 session（或重新查库重建），更新 `state`、`positionMs`、`playbackRate`、`lastReport`。`playing` 时 TTL = 剩余时长+5秒；`paused` 时 TTL = 30 分钟。入队 `PlaybackReport`。仅 `starting`/`playing` 状态还会入队 `NowPlaying`（若 player 启用了 scrobble） |
| `stopped` | 关键逻辑：若 `!IgnoreScrobble && player.ScrobbleEnabled`，判断 `positionMs >= min(trackDuration*50%, 240s)`。阈值满足时：<br>1. `incPlay()` 在事务中递增 `media_file`、`album`、参与 artist 的播放计数，并可选写入 `scrobble` 历史表<br>2. `dispatchScrobble()` 将 scrobble 事件分发给已授权的各 scrobbler（Last.fm、ListenBrainz、插件等）<br>最后入队 `PlaybackReport` 并从 `playMap` 移除该 clientId |
| `expired` | 由缓存过期回调触发，state 被改写为 expired，不触发 scrobble/incPlay，仅入队 PlaybackReport |

报告使用 `clientId`（来自 `ClientUniqueId`）作为 `playMap` 缓存 key，用于区分不同客户端/标签页的播放会话。

### 3.4 Keepalive 认证降级

[reportPlaybackKeepalive](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/subsonic/index.js#L50-L58) 与常规 `reportPlayback` 使用不同的认证通道：

```js
const reportPlaybackKeepalive = (mediaId, positionMs, state) => {
  const u = reportPlaybackUrl(mediaId, positionMs, state)
  if (u) {
    fetch(baseUrl(u), {
      keepalive: true,
      headers: { [clientUniqueIdHeader]: clientUniqueId },
    })
  }
}
```

- **常规 reportPlayback**：通过 `httpClient` 走 `/rest/...` 子路径，包含 JWT `Authorization` header + `X-ND-Client-Unique-Id` header，URL 参数同时带有 Subsonic 认证（`u/t/s`）
- **reportPlaybackKeepalive**：使用原生 `fetch` + `keepalive: true`（页面卸载时也能可靠发送），**仅附加 `X-ND-Client-Unique-Id` header**，**不**附加 JWT `Authorization` header，完全依赖 URL 参数中的 Subsonic `u/t/s` 三参数进行认证。这就是所谓**认证降级**——在页面卸载场景下无法保证 JWT header 被正确携带，因此退回到 URL 参数方式。
- 该 fetch 调用被 `try/catch` 包裹，任何异常（包括 fetch/sendBeacon 抛出）被静默忽略。

---

## 四、ClientUniqueId 与 Client 的区别

两者是完全不同层面的概念，在后端通过不同的中间件注入 request context：

### 4.1 Client（客户端名称）

- **来源**：Subsonic API URL 参数 `c=...`；前端固定传入 `"NavidromeUI"`（见 [subsonic/index.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/subsonic/index.js#L22)）
- **注入方式**：[subsonic/middlewares.go checkRequiredParameters](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/server/subsonic/middlewares.go#L64-L98) 从 URL 参数 `c` 解析并调用 `request.WithClient(ctx, client)`
- **用途**：
  - 区分 Subsonic 客户端类型（如 `NavidromeUI`、`DSub`、`Symfonium`）
  - 作为 [playqueue `ChangedBy`](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/server/nativeapi/queue.go#L112) 字段写入数据库，标识哪个客户端修改了播放队列
  - [players.Register](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/core/players.go#L34-L78) 中与 `userId + userAgent` 联合作为查找/创建 player 记录的 key 之一
  - `PlaybackSession.PlayerName` 字段

### 4.2 ClientUniqueId（客户端唯一标识）

- **来源**：前端每次页面加载时用 `uuidv4()` 生成（见 [httpClient.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/dataProvider/httpClient.js#L9-L10)）
- **传输方式**：HTTP header `X-ND-Client-Unique-Id`；前端每个请求都会带（httpClient 中默认 set），keepalive fetch 也手动带上
- **注入方式**：[server/middlewares.go clientUniqueIDMiddleware](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/server/middlewares.go#L130-L166)：
  1. 先读 header；若无，读 cookie `X-ND-Client-Unique-Id`
  2. 若 header 中有值，把该值写回 cookie（HttpOnly, Secure, SameSiteStrict, MaxAge=CookieExpiry）
  3. 最后 `request.WithClientUniqueId(ctx, clientUniqueId)` 注入 context
- **用途**：
  - `playTracker.playMap` 的 key（`PlaybackSession.PlayerId`），区分**同一用户的不同标签页/浏览器实例**的播放会话
  - scrobbler `NowPlaying` 列表按 clientUniqueId 区分
  - 当 Subsonic `ReportPlayback` 请求中缺少 ClientUniqueId 时，回退为 `player.ID`（见 [media_annotation.go](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/server/subsonic/media_annotation.go#L270-L273)）

---

## 五、跨会话恢复机制

### 5.1 前端恢复流程

当用户重新打开浏览器时，恢复流程如下：

```
1. createAdminStore() 启动
   ├→ loadState() 从 localStorage 读取序列化状态
   │   └→ 若 JSON parse 异常或 key 不存在，静默返回 undefined
   ├→ if (savedPlayIndex) playIndex = savedPlayIndex
   │   └→ ⚠️ savedPlayIndex === 0 时不赋值
   └→ createStore(resettableAppReducer, persistedState)

2. Player 组件挂载 (useEffect on [dispatch])
   ├→ detectBrowserProfile() 检测浏览器编解码能力
   ├→ decisionService.setProfile(profile)
   ├→ dispatch(setTranscodingProfile(profile))
   ├→ currentIdx = state.savedPlayIndex || 0   （同样对 0 不精确）
   ├→ 取 queue.slice(currentIdx, currentIdx+4) 中的 trackIds
   ├→ 若 trackIds 为空 → dispatch(refreshQueue())
   └→ 否则用 Promise.allSettled 并发 resolveStreamUrl，dispatch(refreshQueue(resolvedUrls))
       └→ 某首歌 resolve 失败（rejected）则忽略，不写入 resolvedUrls

3. PLAYER_REFRESH_QUEUE reducer 处理
   ├→ 遍历 queue，每首歌：
   │    resolvedUrls[trackId] ? resolvedUrls[trackId] : subsonic.streamUrl(trackId)
   │    └→ 转码决策 URL 未解析/失效时，回落到非转码的原始 Subsonic stream URL
   ├→ clear: true
   ├→ autoPlay: false（不自动播放）
   └→ playIndex: savedPlayIndex >= 0 ? savedPlayIndex : 0 （此处正确处理 0）
```

### 5.2 refreshQueue 的转码回落机制

[playerReducer.js PLAYER_REFRESH_QUEUE](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/reducers/playerReducer.js#L232-L247) 和 [makeMusicSrc](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/reducers/playerReducer.js#L35-L41) 形成**两级转码回落**：

**第一级 - 队列恢复时**（`PLAYER_REFRESH_QUEUE`）：
```js
musicSrc: item.isRadio
  ? item.musicSrc
  : resolvedUrls[item.trackId] || subsonic.streamUrl(trackId)
```
`Promise.allSettled` 只收集 fulfilled 的结果，rejected 的歌曲在 `resolvedUrls` 中没有 key，于是走 `subsonic.streamUrl()`（非转码直链）。

**第二级 - 播放器按需请求时**（`makeMusicSrc` 函数用于新加入队列的曲目）：
```js
const makeMusicSrc = (trackId) =>
  decisionService.getProfile()
    ? () => decisionService.resolveStreamUrl(trackId).catch(() => subsonic.streamUrl(trackId))
    : subsonic.streamUrl(trackId)
```
`musicSrc` 是一个函数（navidrome-music-player 支持函数作为延迟解析的 URL），当播放器实际请求音频时才调用。若 `resolveStreamUrl`（含转码决策请求）失败，`.catch` 回落到非转码直链。此外，当 `onAudioError` 发生时，[Player.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/audioplayer/Player.jsx#L386-L406) 会调用 `decisionService.invalidateAll()` 清空决策缓存，并重新预取后续歌曲的决策。

### 5.3 后端播放队列持久化

后端提供两套 API 保存/恢复播放队列，两者的 `ChangedBy` 字段**对称地**均取自 `request.ClientFrom(ctx)`（即 Subsonic `c` 参数，前端为 `"NavidromeUI"`）：

#### Native API (`/api/queue`)

[nativeapi/queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/server/nativeapi/queue.go#L184-L191) 注册了四个端点：

| 方法 | 路径 | Handler | ChangedBy | 说明 |
|------|------|---------|-----------|------|
| GET | `/api/queue` | `getQueue` | - | 获取当前用户播放队列 |
| POST | `/api/queue` | `saveQueue` | `client` | 整体替换保存队列 |
| PUT | `/api/queue` | `updateQueue` | `client` | 部分更新队列（可只更新 ids / current / position） |
| DELETE | `/api/queue` | `clearQueue` | - | 清空队列 |

`saveQueue` 与 `updateQueue` 的 `ChangedBy` 字段均由 `extractUserAndClient(ctx)` → `request.ClientFrom(ctx)` 提供。

#### Subsonic API (`/rest/savePlayQueue`, `/rest/getPlayQueue`)

[bookmarks.go](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/server/subsonic/bookmarks.go) 提供兼容 Subsonic 协议的播放队列 API：

- `SavePlayQueue`: 通过 `current` ID（歌曲 ID）指定当前曲目，`ChangedBy = client`（[第129行](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/server/subsonic/bookmarks.go#L129)）
- `SavePlayQueueByIndex`: 通过 `currentIndex`（整数索引）指定当前曲目，`ChangedBy = client`（[第204行](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/server/subsonic/bookmarks.go#L204)）
- `GetPlayQueue` / `GetPlayQueueByIndex`: 获取队列，响应中包含 `ChangedBy` 字段（[第99行](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/server/subsonic/bookmarks.go#L99)、[第172行](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/server/subsonic/bookmarks.go#L172)）

两套接口写入和读取 `ChangedBy` 完全对称，使用相同的 `client`（Subsonic `c` 参数）。

#### 数据库持久化

[playqueue_repository.go](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/persistence/playqueue_repository.go#L39-L77) 中 `Store` 方法的逻辑：

- 每个用户只有一条播放队列记录（`user_id` 唯一）
- 无 `colNames` 参数时：先删除旧记录，再插入新记录（整体替换）
- 有 `colNames` 参数时：仅更新指定列（部分更新）
- `items` 字段以逗号分隔的 ID 字符串存储
- `RetrieveWithMediaFiles` 会根据存储的 ID 列表从 `media_file` 表加载完整歌曲信息

### 5.4 播放队列保存为播放列表

[SaveQueueDialog.jsx](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/dialogs/SaveQueueDialog.jsx) 提供了将当前播放队列保存为播放列表的功能：

1. 先通过 `dataProvider.create('playlist', ...)` 创建新播放列表
2. 再通过 `dataProvider.create('playlistTrack', ...)` 添加所有曲目 ID
3. 这是用户主动触发的操作，不是自动同步

### 5.5 播放器配置 (Player) 的持久化

[player.go](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/model/player.go) 定义了 `Player` 模型，包含：

| 字段 | 说明 |
|------|------|
| `TranscodingId` | 关联的转码配置 |
| `MaxBitRate` | 最大比特率限制 |
| `ReportRealPath` | 是否报告真实路径 |
| `ScrobbleEnabled` | 是否启用 Scrobble |
| `Client` / `UserAgent` | 客户端标识，用于匹配 |

[players.go](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/core/players.go#L34-L78) 中的 `Register` 逻辑：

1. 如果请求携带 `playerID`（来自 cookie `nd-player-<username_hash>`），先按 ID 查找
2. 若 ID 对应的 player 的 `Client` 不匹配，则忽略该 ID
3. 尝试按 `(userId, client, userAgent)` 组合查找已有 player
4. 若均未找到，创建新 player 记录
5. 更新 `LastSeen`、`IP`、`UserAgent` 等信息并保存（有频率限制）

### 5.6 用户属性 (UserProps) 的持久化

[user_props_repository.go](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/persistence/user_props_repository.go) 实现了简单的 key-value 存储：

- 表名：`user_props`
- 联合主键：`(user_id, key)`
- `Put` 方法：先尝试 UPDATE，若无匹配行则 INSERT（upsert 语义）
- 用于存储 Last.fm session key、ListenBrainz token 等后端管理的偏好

[session_keys.go](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/core/agents/session_keys.go) 是 `UserPropsRepository` 的封装，为第三方服务（Last.fm、ListenBrainz）提供 session key 的存取。

---

## 六、认证状态与用户信息的持久化

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

**登出行为**：`authProvider.logout()` 调用 `removeItems()` 仅清除上表中的认证相关条目（不含 `locale` 和 `defaultView`）。但如 1.2 节所述，Redux `USER_LOGOUT` 会触发 store.subscribe，将各子 reducer 初始值写回 `localStorage.state`。

[httpClient.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/dataProvider/httpClient.js) 在每次 HTTP 请求中：
- 从 `localStorage` 读取 JWT token 附加到 `X-ND-Authorization` 头
- 附加 `X-ND-Client-Unique-Id` 头（UUID，页面加载时生成）
- 从响应头中检查并更新 JWT token（自动刷新）

---

## 七、实时事件同步

[eventStream.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/eventStream.js) 通过 SSE (Server-Sent Events) 连接 `/api/events`，监听以下事件：

| 事件类型 | 说明 |
|---------|------|
| `serverStart` | 服务器重启 |
| `scanStatus` | 扫描状态更新（100ms throttle） |
| `refreshResource` | 资源变更通知 |
| `nowPlayingCount` | 正在播放计数 |
| `keepAlive` | 心跳 |

当连接断开时自动重连（5秒延迟），重连成功后分发 `streamReconnected` 事件触发数据刷新。

---

## 八、整体数据流图

```
┌──────────────────────────────────────────────────────────────────┐
│                         Browser (localStorage)                   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │  localStorage.key = "state"                              │     │
│  │  {                                                       │     │
│  │    theme: "DarkTheme",                                   │     │
│  │    library: { userLibraries: [...], selectedLibraries: [...] }, │
│  │    player: { queue: [...], volume: 0.8, savedPlayIndex: 0 }, │
│  │    albumView: { grid: true },                            │     │
│  │    settings: { notifications: true, toggleableFields: {}, omittedFields: {} } │
│  │  }                                                       │     │
│  └─────────────────────────────────────────────────────────┘     │
│    ↑ 每 1000ms throttle (lodash.throttle 默认 leading+trailing)  │
│    ↑ USER_LOGOUT 触发后会写回各 reducer 初始值（覆盖）            │
│                                                                  │
│  ┌──────────────────────┐  ┌─────────────────────────┐          │
│  │  localStorage keys:  │  │  localStorage keys:     │          │
│  │  "gainMode"          │  │  "token", "userId",     │          │
│  │  "preAmp"            │  │  "username", "role",    │          │
│  │  "locale"            │  │  "subsonic-token",      │          │
│  │  "defaultView"       │  │  "subsonic-salt",       │          │
│  └──────────────────────┘  │  "is-authenticated"     │          │
│                             └─────────────────────────┘          │
│                                                                  │
│  ⚠️ 无 storage 事件监听 —— 多标签页间状态不实时同步              │
└──────────────────────┬───────────────────────────────────────────┘
                       │ HTTP / SSE
                       │
                       │  请求头:
                       │    X-ND-Client-Unique-Id: <uuidv4, 每页一个>
                       │    X-ND-Authorization: Bearer <jwt>
                       │  URL 参数(Subsonic):
                       │    c=NavidromeUI, u=<user>, t=<token>, s=<salt>
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                      Backend (SQLite Database)                    │
│                                                                  │
│  request context 注入:                                            │
│    ClientUniqueId ← X-ND-Client-Unique-Id header 或 cookie        │
│    Client         ← Subsonic URL 参数 c=...                       │
│    Player         ← (userId, client, userAgent) 查找/注册         │
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
│  │  scrobble    │  │  (client)     │  │                        │  │
│  └─────────────┘  └──────────────┘  └────────────────────────┘  │
│                                                                  │
│  scrobbler.PlayTracker:                                           │
│    playMap[ClientUniqueId] → PlaybackSession                      │
│      state: starting/playing/paused/stopped/expired              │
│      stopped + 阈值(50%/240s) → incPlay + dispatchScrobble       │
└──────────────────────────────────────────────────────────────────┘
```

---

## 九、已知问题与设计局限汇总

| 问题 | 位置 | 影响 |
|------|------|------|
| `savedPlayIndex === 0` 在 store 创建时不恢复为 `playIndex` | [createAdminStore.js#L44](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/store/createAdminStore.js#L44) | 若恰好停在第一首歌，`playIndex` 不会立即赋值，需等 `refreshQueue` 才能修正 |
| `loadState`/`saveState` 静默吞异常 | [persistState.js](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/store/persistState.js) | localStorage 损坏或隐私模式下状态丢失，无任何日志 |
| 无 `storage` 事件监听 | 整个 `ui/src` | 多标签页打开时偏好/队列变化互不可见 |
| USER_LOGOUT 会将偏好写回初始值 | [createAdminStore.js#L55-L71](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/store/createAdminStore.js#L55-L71) | 登出后再登录，主题、库选择、音量等偏好可能丢失（取决于 reducer 初始值 vs 之前保存值的时序） |
| 页面卸载 keepalive 请求退化为 URL 参数认证 | [subsonic/index.js#L50-L58](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/subsonic/index.js#L50-L58) | 依赖 `u/t/s` Subsonic 参数，若 subsonic token 过期则页面卸载时 stopped 报告可能失败 |
| 转码决策解析失败时回落到非转码 URL | [playerReducer.js#L238-L241](file:///d:/fz/0601-1/solo-dogfeeding/code/60-navidrome/ui/src/reducers/playerReducer.js#L238-L241) | 带宽受限场景下可能意外播放无损/高码率文件 |

---

## 十、总结

Navidrome 的偏好与状态同步机制具有以下特点：

1. **前端主导的偏好管理**：大部分用户偏好（主题、语言、通知等）仅存储在浏览器 `localStorage`，不与后端同步。这意味着不同浏览器/设备的偏好相互独立。

2. **双层播放状态持久化**：播放队列和位置既保存在 `localStorage`（用于同浏览器快速恢复），也可通过后端 API 保存到数据库（用于跨设备或通过 Subsonic 兼容客户端访问）。

3. **细粒度状态区分**：Redux 中明确区分持久化字段（`savedPlayIndex`）和瞬态字段（`playIndex`、`clear`），避免将瞬态数据写入 `localStorage` 导致恢复异常。但 `savedPlayIndex === 0` 时因 falsy 判断存在边界缺陷。

4. **URL 时效性处理 + 两级转码回落**：恢复播放队列时重新解析流 URL。若转码决策请求失败，自动回落到非转码 Subsonic `stream` URL，确保播放器可用。

5. **后端管理的偏好**：Last.fm / ListenBrainz 等第三方服务集成状态由后端 `user_props` 表管理，前端仅做展示和触发。

6. **Client vs ClientUniqueId 分层**：`Client`（Subsonic `c` 参数）标识客户端类型，用于 `player` 表匹配和 `playqueue.ChangedBy`；`ClientUniqueId`（每次页面加载的 UUID）用于区分同一用户的不同标签页/浏览器实例，是 `PlayTracker.playMap` 的 key。

7. **两套播放队列接口 ChangedBy 对称**：Native API (`/api/queue`) 的 `saveQueue`/`updateQueue` 和 Subsonic API (`/rest/savePlayQueue*`) 的 `SavePlayQueue`/`SavePlayQueueByIndex` 均使用 `request.ClientFrom(ctx)` 作为 `ChangedBy`，读取时也都返回此字段。

8. **播放报告的后端语义**：`starting` 建立会话、`playing/paused` 更新会话、`stopped` 达到阈值（50% 或 240 秒）时触发 `incPlay`（递增歌曲/专辑/艺人播放计数，写入 scrobble 历史）并向已授权 scrobbler 分发；过期会话自动标记为 `expired` 但不触发 scrobble。
