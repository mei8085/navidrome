# Subsonic 兼容策略落地可行性分析

本文档基于端点链路梳理和测试矩阵评估，回答 Subsonic 兼容策略能否安全落地的核心疑问，并给出可执行的最小修复方案。

---

## 一、端点链路梳理：哪些请求经过 getPlayer

### 1.1 路由中间件挂载全景

```go
// server/subsonic/api.go:86-233
func (api *Router) routes() http.Handler {
    r := chi.NewRouter()
    r.Use(postFormToQueryParams)
    
    // Public: getOpenSubsonicExtensions (不经过 getPlayer)
    
    r.Group(func(r chi.Router) {
        r.Use(checkRequiredParameters)
        r.Use(authenticate(api.ds))
        
        // 分组 1: ping, getLicense → 经过 getPlayer
        r.Group(func(r chi.Router) {
            r.Use(getPlayer(api.players))
            h(r, "ping", api.Ping)
            h(r, "getLicense", api.GetLicense)
        })
        
        // 分组 2: 浏览类接口 → 经过 getPlayer
        r.Group(func(r chi.Router) {
            r.Use(getPlayer(api.players))
            h(r, "getArtists", api.GetArtists)      // 调用 toArtistID3 → toOSArtistID3
            h(r, "getArtist", api.GetArtist)        // 调用 toArtistID3 → toOSArtistID3
            h(r, "getAlbum", api.GetAlbum)          // 调用 buildAlbumID3 → buildOSAlbumID3
            h(r, "getAlbumInfo", api.GetAlbumInfo)
            h(r, "getArtistInfo", api.GetArtistInfo)
            h(r, "getArtistInfo2", api.GetArtistInfo2)  // 调用 toArtistID3 → toOSArtistID3
            // ...
        })
        
        // 分组 3: 列表类接口 → 经过 getPlayer
        r.Group(func(r chi.Router) {
            r.Use(getPlayer(api.players))
            hr(r, "getAlbumList", api.GetAlbumList)    // 调用 childFromAlbum → osChildFromAlbum
            hr(r, "getAlbumList2", api.GetAlbumList2)  // 调用 buildAlbumID3 → buildOSAlbumID3
            h(r, "getStarred", api.GetStarred)
            h(r, "getStarred2", api.GetStarred2)       // 调用 buildAlbumID3 → buildOSAlbumID3
            h(r, "getRandomSongs", api.GetRandomSongs)
            h(r, "getSongsByGenre", api.GetSongsByGenre)
        })
        
        // 分组 4: 搜索接口 → 经过 getPlayer
        r.Group(func(r chi.Router) {
            r.Use(getPlayer(api.players))
            h(r, "search2", api.Search2)
            h(r, "search3", api.Search3)          // 调用 toArtistID3, buildAlbumID3
        })
        
        // 分组 5: 网络电台 → 经过 getPlayer
        r.Group(func(r chi.Router) {
            r.Use(getPlayer(api.players))
            h(r, "getInternetRadioStations", api.GetInternetRadios)  // 直接做 legacy 判定
        })
        
        // ... 其他分组均经过 getPlayer
    })
}
```

### 1.2 四个目标函数的调用链路

#### toOSArtistID3 调用链

```
端点（经过 getPlayer）
    ↓
toArtistID3() → helpers.go:115-132
    ↓
toOSArtistID3() → helpers.go:134-145  [不安全模式 B]
```

**涉及端点：**
- `getArtists` → `getArtistIndexID3` → `toArtistID3`
- `getArtist` → `buildArtist` → `toArtistID3`
- `getArtistInfo2` → `toArtistID3`
- `getStarred2` → `toArtistID3`
- `search3` → `toArtistID3`

#### osChildFromAlbum 调用链

```
端点（经过 getPlayer）
    ↓
childFromAlbum() → helpers.go:337-364
    ↓
osChildFromAlbum() → helpers.go:366-387  [不安全模式 B]
```

**涉及端点：**
- `getAlbumList` → `childFromAlbum`
- `getStarred` → `childFromAlbum`
- `getMusicDirectory` → `childFromAlbum`
- `search2` → `childFromAlbum`
- `getShares` → `childFromAlbum`

#### buildOSAlbumID3 调用链

```
端点（经过 getPlayer）
    ↓
buildAlbumID3() → helpers.go:433-451
    ↓
buildOSAlbumID3() → helpers.go:453-486  [不安全模式 B]
```

**涉及端点：**
- `getAlbumList2` → `buildAlbumID3`
- `getAlbum` → `buildAlbum` → `buildAlbumID3`
- `getStarred2` → `buildAlbumID3`
- `search3` → `buildAlbumID3`

#### GetInternetRadios 调用链

```
端点（经过 getPlayer）
    ↓
GetInternetRadios() → radio.go:57-93  [不安全模式 B，直接判定]
```

**涉及端点：**
- `getInternetRadioStations` → 直接在函数内做 legacy 判定

### 1.3 哪些场景会出现无 player

**所有经过 `getPlayer` 中间件的端点都可能遇到无 player 场景**，触发条件：

```go
// middlewares.go:192-195
player, trc, err := players.Register(ctx, playerId, client, userAgent, ip)
if err != nil {
    log.Error(ctx, "Could not register player", ...)
    // ⚠️  不返回错误，继续执行，但 context 中没有 player
} else {
    ctx = request.WithPlayer(ctx, *player)
    r = r.WithContext(ctx)
}
next.ServeHTTP(w, r)  // 无论成功失败都继续
```

**players.Register 可能失败的场景：**
1. 数据库临时不可用（连接池耗尽、网络分区）
2. 数据库事务冲突
3. Player 表数据异常
4. 并发写入冲突

**无 player 场景的实际行为：**

| 函数 | 当前行为（模式 B） | 预期行为（模式 A） |
|------|------------------|------------------|
| toOSArtistID3 | 不返回 OpenSubsonic 扩展 | 返回 OpenSubsonic 扩展 |
| osChildFromAlbum | 不返回 OpenSubsonic 扩展 | 返回 OpenSubsonic 扩展 |
| buildOSAlbumID3 | 不返回 OpenSubsonic 扩展 | 返回 OpenSubsonic 扩展 |
| GetInternetRadios | 不返回 coverArt | 返回 coverArt |

**影响范围估算：** 至少 **15+ 个 API 端点**会受到影响，覆盖浏览、搜索、列表、收藏等核心功能。

---

## 二、测试矩阵覆盖评估

### 2.1 现有测试覆盖情况

#### helpers_test.go 覆盖情况

| 函数 | 有 player 测试 | 无 player 测试 | legacy client 测试 |
|------|--------------|--------------|-----------------|
| childFromMediaFile (Minimal) | ✅ | ✅（helpers_test.go:305-311） | ✅ |
| osChildFromMediaFile | ✅ | ✅（helpers_test.go:376-381） | ✅（helpers_test.go:336-347） |
| childFromAlbum | ⚠️ 仅测试 averageRating | ❌ | ❌ |
| toArtistID3 | ⚠️ 仅测试 averageRating | ❌ | ❌ |
| buildAlbumID3 | ⚠️ 仅测试 Created fallback | ❌ | ❌ |
| toOSArtistID3 | ❌ | ❌ | ❌ |
| osChildFromAlbum | ❌ | ❌ | ❌ |
| buildOSAlbumID3 | ❌ | ❌ | ❌ |

#### radio_test.go 覆盖情况

| 场景 | 测试 | 当前预期 | 正确预期 |
|------|------|---------|---------|
| 有 legacy client player | ✅（radio_test.go:80-99） | 不返回 coverArt | 不返回 coverArt |
| 有 modern client player | ✅（radio_test.go:57-78） | 返回 coverArt | 返回 coverArt |
| 无 player | ✅（radio_test.go:101-114） | 不返回 coverArt ❌ | 返回 coverArt |
| LegacyClients 为空 | ✅（radio_test.go:116-134） | 返回 coverArt | 返回 coverArt |

#### playlists_test.go 覆盖情况

| 函数 | 有 player 测试 | 无 player 测试 | legacy client 测试 |
|------|--------------|--------------|-----------------|
| buildPlaylist (Minimal) | ✅ | ❌ | ✅ |
| buildOSPlaylist | ✅ | ❌ | ✅ |

### 2.2 遗漏的回归点

**高优先级遗漏：**

1. **toOSArtistID3 无任何测试**
   - 无 player 场景测试
   - legacy client 场景测试
   - 子串误判场景测试（如 "foo" vs "foobar2000"）

2. **osChildFromAlbum 无任何测试**
   - 无 player 场景测试
   - legacy client 场景测试
   - 子串误判场景测试

3. **buildOSAlbumID3 无任何测试**
   - 无 player 场景测试
   - legacy client 场景测试
   - 子串误判场景测试

4. **子串误判场景完全未覆盖**
   - 配置 `LegacyClients = "foobar2000"`，客户端 `"foo"` 应不匹配
   - 配置 `LegacyClients = "dsub,isub"`，客户端 `"sub"` 应不匹配

5. **端到端测试缺失**
   - 没有测试模拟 `players.Register` 失败的场景
   - 没有验证整个请求链路在无 player 时的行为一致性

**中优先级遗漏：**

6. **childFromAlbum 缺少 legacy/minimal 测试**
7. **toArtistID3 缺少 legacy client 测试**
8. **buildAlbumID3 缺少 legacy client 测试**

---

## 三、最小修复方案与风险分级

### 3.1 修复方案选择

**推荐方案：统一采用"安全模式 A"**

```
player, ok := request.PlayerFrom(ctx)
if ok && isClientInList(conf.Server.Subsonic.LegacyClients, player.Client) {
    return nil
}
```

**理由：**
1. 与 `osChildFromMediaFile`、`buildOSPlaylist` 等已有的正确实现一致
2. `isClientInList` 精确匹配，避免子串误判
3. 检查 `ok` 返回值，无 player 时正确判定为非 legacy
4. 改动量极小，风险可控

### 3.2 代码改动清单

#### 改动 1：toOSArtistID3（helpers.go:134-145）

```go
// 改动前：
func toOSArtistID3(ctx context.Context, a model.Artist) *responses.OpenSubsonicArtistID3 {
    player, _ := request.PlayerFrom(ctx)
    if strings.Contains(conf.Server.Subsonic.LegacyClients, player.Client) {
        return nil
    }
    // ...
}

// 改动后：
func toOSArtistID3(ctx context.Context, a model.Artist) *responses.OpenSubsonicArtistID3 {
    player, ok := request.PlayerFrom(ctx)
    if ok && isClientInList(conf.Server.Subsonic.LegacyClients, player.Client) {
        return nil
    }
    // ...
}
```

#### 改动 2：osChildFromAlbum（helpers.go:366-387）

```go
// 改动前：
func osChildFromAlbum(ctx context.Context, al model.Album) *responses.OpenSubsonicChild {
    player, _ := request.PlayerFrom(ctx)
    if strings.Contains(conf.Server.Subsonic.LegacyClients, player.Client) {
        return nil
    }
    // ...
}

// 改动后：
func osChildFromAlbum(ctx context.Context, al model.Album) *responses.OpenSubsonicChild {
    player, ok := request.PlayerFrom(ctx)
    if ok && isClientInList(conf.Server.Subsonic.LegacyClients, player.Client) {
        return nil
    }
    // ...
}
```

#### 改动 3：buildOSAlbumID3（helpers.go:453-486）

```go
// 改动前：
func buildOSAlbumID3(ctx context.Context, album model.Album) *responses.OpenSubsonicAlbumID3 {
    player, _ := request.PlayerFrom(ctx)
    if strings.Contains(conf.Server.Subsonic.LegacyClients, player.Client) {
        return nil
    }
    // ...
}

// 改动后：
func buildOSAlbumID3(ctx context.Context, album model.Album) *responses.OpenSubsonicAlbumID3 {
    player, ok := request.PlayerFrom(ctx)
    if ok && isClientInList(conf.Server.Subsonic.LegacyClients, player.Client) {
        return nil
    }
    // ...
}
```

#### 改动 4：GetInternetRadios（radio.go:57-93）

```go
// 改动前（第 73-76 行）：
player, _ := request.PlayerFrom(ctx)
if strings.Contains(conf.Server.Subsonic.LegacyClients, player.Client) {
    continue
}

// 改动后：
player, ok := request.PlayerFrom(ctx)
if ok && isClientInList(conf.Server.Subsonic.LegacyClients, player.Client) {
    continue
}
```

### 3.3 测试改动清单

#### 改动 5：更新 radio_test.go 无 player 场景预期

```go
// radio_test.go:101-114 改动前：
Context("when no player in context", func() {
    It("does not include coverArt (empty client matches legacy list)", func() {
        DeferCleanup(configtest.SetupConfig())
        conf.Server.Subsonic.LegacyClients = "legacy-client"

        r := httptest.NewRequest("GET", "/rest/getInternetRadios", nil)
        r = r.WithContext(ctx)

        response, err := api.GetInternetRadios(r)

        Expect(err).ToNot(HaveOccurred())
        Expect(response.InternetRadioStations.Radios[0].OpenSubsonicRadio).To(BeNil())
    })
})

// 改动后：
Context("when no player in context", func() {
    It("includes coverArt (no player treated as modern client)", func() {
        DeferCleanup(configtest.SetupConfig())
        conf.Server.Subsonic.LegacyClients = "legacy-client"

        r := httptest.NewRequest("GET", "/rest/getInternetRadios", nil)
        r = r.WithContext(ctx)

        response, err := api.GetInternetRadios(r)

        Expect(err).ToNot(HaveOccurred())
        Expect(response.InternetRadioStations.Radios[0].OpenSubsonicRadio).ToNot(BeNil())
        Expect(response.InternetRadioStations.Radios[0].CoverArt).To(Equal("ra-rd-1_0"))
    })
})
```

#### 新增测试：helpers_test.go 中添加 toOSArtistID3 测试

```go
Describe("toOSArtistID3", func() {
    var a model.Artist
    
    BeforeEach(func() {
        a = model.Artist{
            ID: "ar-1",
            Name: "Test Artist",
            MbzArtistID: "mbz-123",
            SortArtistName: "Artist, Test",
            OrderArtistName: "Test Artist",
        }
    })
    
    Context("with legacy client", func() {
        BeforeEach(func() {
            conf.Server.Subsonic.LegacyClients = "legacy-client"
            player := model.Player{Client: "legacy-client"}
            ctx = request.WithPlayer(ctx, player)
        })
        
        It("returns nil", func() {
            osArtist := toOSArtistID3(ctx, a)
            Expect(osArtist).To(BeNil())
        })
    })
    
    Context("with non-legacy client", func() {
        BeforeEach(func() {
            conf.Server.Subsonic.LegacyClients = "legacy-client"
            player := model.Player{Client: "modern-client"}
            ctx = request.WithPlayer(ctx, player)
        })
        
        It("returns OpenSubsonic artist fields", func() {
            osArtist := toOSArtistID3(ctx, a)
            Expect(osArtist).ToNot(BeNil())
            Expect(osArtist.MusicBrainzId).To(Equal("mbz-123"))
            Expect(osArtist.SortName).To(Equal("Artist, Test"))
        })
    })
    
    Context("when no player in context", func() {
        It("returns OpenSubsonic artist fields", func() {
            osArtist := toOSArtistID3(ctx, a)
            Expect(osArtist).ToNot(BeNil())
        })
    })
    
    Context("with substring client name", func() {
        BeforeEach(func() {
            conf.Server.Subsonic.LegacyClients = "foobar2000"
            player := model.Player{Client: "foo"}  // 子串，不应匹配
            ctx = request.WithPlayer(ctx, player)
        })
        
        It("does not match substring and returns fields", func() {
            osArtist := toOSArtistID3(ctx, a)
            Expect(osArtist).ToNot(BeNil())  // 不应被误判为 legacy
        })
    })
})
```

### 3.4 风险分级

| 风险项 | 等级 | 说明 | 缓解措施 |
|--------|------|------|---------|
| 正常请求行为变更 | 🟢 极低 | 只影响无 player 的异常路径，正常有 player 的请求行为完全不变 | 无需额外措施 |
| 端到端行为一致性 | 🟡 低 | 修复后 4 个函数与其他 4 个函数行为一致，整体更一致 | 运行完整测试套件 |
| 子串误判修复 | 🟡 中 | 修复了潜在的误判问题，但可能影响依赖此 bug 的部署 | 文档说明，建议配置检查 |
| 测试预期变更 | 🟡 中 | radio 测试预期反转，需要确认业务合理性 | 代码评审确认 |

### 3.5 回归验证清单

执行修复后，按以下顺序验证：

1. **单元测试通过**
   ```bash
   go test ./server/subsonic/... -run "TestHelpers|TestRadio" -v
   ```

2. **验证修复的 4 个函数行为一致**
   - 有 legacy client player → 返回 nil
   - 有 modern client player → 返回扩展字段
   - 无 player → 返回扩展字段
   - 子串客户端名 → 不匹配，返回扩展字段

3. **集成测试（可选）**
   - 模拟 `players.Register` 失败，验证端点响应
   - 验证 `getArtists`、`getAlbumList2`、`getInternetRadioStations` 等端点返回 OpenSubsonic 扩展

4. **手动验证**
   - 配置 `LegacyClients = "foobar2000"`
   - 使用客户端名 `"foo"` 访问 API
   - 验证返回 OpenSubsonic 扩展（之前会错误地不返回）

---

## 四、落地可行性结论

### 4.1 技术可行性

✅ **完全可行**
- 改动量极小：4 个源文件，8 行代码修改
- 测试改动：1 个测试文件更新预期，可选新增单元测试
- 风险可控：仅影响异常路径（player 注册失败）
- 行为一致：修复后与 osChildFromMediaFile 等已有正确实现保持一致

### 4.2 业务影响

**对用户的影响：**
- 正常场景（player 注册成功）：**完全无影响**
- 异常场景（player 注册失败）：之前不返回 OpenSubsonic 扩展，**修复后返回扩展字段**，功能增强

**对运维的影响：**
- 如果有部署依赖 "无 player 时不返回扩展" 的行为（极不可能），需要调整
- 建议在配置中检查 LegacyClients 是否包含可能被子串误判的名称

### 4.3 落地建议

| 步骤 | 操作 | 负责人 |
|------|------|--------|
| 1 | 提交代码改动（4 个源文件） | 开发 |
| 2 | 更新 radio_test.go 测试预期 | 开发 |
| 3 | （可选）新增 toOSArtistID3 等单元测试 | 开发 |
| 4 | 运行完整测试套件验证 | CI |
| 5 | 代码评审，确认行为变更合理性 | 评审人 |
| 6 | 发布，在变更日志中说明 | 发布负责人 |

---

### 代码位置速查

| 改动点 | 文件 | 行号 |
|--------|------|------|
| toOSArtistID3 模式 B | `server/subsonic/helpers.go` | 134-137 |
| osChildFromAlbum 模式 B | `server/subsonic/helpers.go` | 366-369 |
| buildOSAlbumID3 模式 B | `server/subsonic/helpers.go` | 453-456 |
| GetInternetRadios 模式 B | `server/subsonic/radio.go` | 73-76 |
| osChildFromMediaFile 模式 A（参考） | `server/subsonic/helpers.go` | 243-246 |
| getPlayer 降级逻辑 | `server/subsonic/middlewares.go` | 192-211 |
| radio 不一致的测试 | `server/subsonic/radio_test.go` | 101-114 |
| helpers 一致的测试（参考） | `server/subsonic/helpers_test.go` | 305-311, 376-381 |
| 路由挂载 | `server/subsonic/api.go` | 86-233 |
