# Subsonic 兼容回归风险最终分析

本文档以 `getArtists`、`getAlbumList`、`getAlbumList2`、`getInternetRadioStations` 四个样本端点为基础，深入分析 `players.Register` 失败后的回归风险，并给出可落地的测试清单和修复顺序。

---

## 一、四个样本端点的链路与行为分析

### 1.1 公共前提：getPlayer 中间件的降级逻辑

```go
// server/subsonic/middlewares.go:192-211
player, trc, err := players.Register(ctx, playerId, client, userAgent, ip)
if err != nil {
    log.Error(ctx, "Could not register player", "username", userName, "client", client, err)
    // ⚠️  失败时仅打日志，context 中没有 player
} else {
    ctx = request.WithPlayer(ctx, *player)  // 成功才存入 context
    r = r.WithContext(ctx)
}
next.ServeHTTP(w, r)  // 无论成功失败，都继续处理请求
```

**关键事实：** `players.Register` 失败时，`request.PlayerFrom(ctx)` 返回 `(model.Player{}, false)`，即零值 player 和 `false`。

---

### 1.2 样本 1：getArtists

**调用链路：**
```
getArtists (api.go:115-126)
    ↓
getArtistIndexID3 (browsing.go:81-98)
    ↓
toArtistID3 (helpers.go:115-132)  // 对每个 artist 调用
    ↓
toOSArtistID3 (helpers.go:134-145)  // 兼容判定点
```

**兼容判定代码：**
```go
// helpers.go:134-138
func toOSArtistID3(ctx context.Context, a model.Artist) *responses.OpenSubsonicArtistID3 {
    player, _ := request.PlayerFrom(ctx)  // 忽略 ok！
    if strings.Contains(conf.Server.Subsonic.LegacyClients, player.Client) {
        return nil  // legacy：不返回 OpenSubsonic 扩展
    }
    // ... 返回 OpenSubsonic 扩展
}
```

**player 注册失败后的行为：**
- `player` 是零值 `model.Player{}`，`player.Client` = `""`（空字符串）
- `strings.Contains(conf.Server.Subsonic.LegacyClients, "")` → **永远返回 true**（任何字符串包含空字符串）
- **无条件进入 legacy 分支**，返回 `nil`
- 最终响应中 **所有 artist 都缺少 OpenSubsonic 扩展字段**

**字段变化对比：**

| 字段 | 正常（有 player） | 异常（无 player） |
|------|------------------|------------------|
| `id` | ✓ | ✓ |
| `name` | ✓ | ✓ |
| `albumCount` | ✓ | ✓ |
| `coverArt` | ✓ | ✓ |
| `musicBrainzId` | ✓（现代客户端） | ❌ 缺失 |
| `sortName` | ✓（现代客户端） | ❌ 缺失 |
| `roles` | ✓（现代客户端） | ❌ 缺失 |

---

### 1.3 样本 2：getAlbumList

**调用链路：**
```
GetAlbumList (album_lists.go:90-103)
    ↓
childFromAlbum (helpers.go:337-364)  // 对每个 album 调用
    ↓
osChildFromAlbum (helpers.go:366-387)  // 兼容判定点
```

**兼容判定代码：**
```go
// helpers.go:366-370
func osChildFromAlbum(ctx context.Context, al model.Album) *responses.OpenSubsonicChild {
    player, _ := request.PlayerFrom(ctx)  // 忽略 ok！
    if strings.Contains(conf.Server.Subsonic.LegacyClients, player.Client) {
        return nil  // legacy：不返回 OpenSubsonic 扩展
    }
    // ... 返回 OpenSubsonic 扩展
}
```

**player 注册失败后的行为：**
- 同样触发空字符串匹配 bug
- **无条件进入 legacy 分支**，返回 `nil`
- 最终响应中 **所有 album 都缺少 OpenSubsonic 扩展字段**

**字段变化对比：**

| 字段 | 正常（有 player） | 异常（无 player） |
|------|------------------|------------------|
| `id` | ✓ | ✓ |
| `title` | ✓ | ✓ |
| `album` | ✓ | ✓ |
| `artist` | ✓ | ✓ |
| `year` | ✓ | ✓ |
| `genre` | ✓ | ✓ |
| `coverArt` | ✓ | ✓ |
| `musicBrainzId` | ✓（现代客户端） | ❌ 缺失 |
| `sortName` | ✓（现代客户端） | ❌ 缺失 |
| `originalReleaseDate` | ✓（现代客户端） | ❌ 缺失 |
| `isCompilation` | ✓（现代客户端） | ❌ 缺失 |

---

### 1.4 样本 3：getAlbumList2

**调用链路：**
```
GetAlbumList2 (album_lists.go:105-116)
    ↓
buildAlbumID3 (helpers.go:433-451)  // 对每个 album 调用
    ↓
buildOSAlbumID3 (helpers.go:453-486)  // 兼容判定点
```

**兼容判定代码：**
```go
// helpers.go:453-457
func buildOSAlbumID3(ctx context.Context, album model.Album) *responses.OpenSubsonicAlbumID3 {
    player, _ := request.PlayerFrom(ctx)  // 忽略 ok！
    if strings.Contains(conf.Server.Subsonic.LegacyClients, player.Client) {
        return nil  // legacy：不返回 OpenSubsonic 扩展
    }
    // ... 返回 OpenSubsonic 扩展
}
```

**player 注册失败后的行为：**
- 同样触发空字符串匹配 bug
- **无条件进入 legacy 分支**，返回 `nil`
- 最终响应中 **所有 album 都缺少 OpenSubsonic 扩展字段**

**字段变化对比：**

| 字段 | 正常（有 player） | 异常（无 player） |
|------|------------------|------------------|
| `id` | ✓ | ✓ |
| `name` | ✓ | ✓ |
| `artist` | ✓ | ✓ |
| `artistId` | ✓ | ✓ |
| `coverArt` | ✓ | ✓ |
| `songCount` | ✓ | ✓ |
| `duration` | ✓ | ✓ |
| `playCount` | ✓ | ✓ |
| `musicBrainzId` | ✓（现代客户端） | ❌ 缺失 |
| `sortName` | ✓（现代客户端） | ❌ 缺失 |
| `originalReleaseDate` | ✓（现代客户端） | ❌ 缺失 |
| `isCompilation` | ✓（现代客户端） | ❌ 缺失 |
| `totalDiscCount` | ✓（现代客户端） | ❌ 缺失 |
| `totalTrackCount` | ✓（现代客户端） | ❌ 缺失 |
| `releaseTypes` | ✓（现代客户端） | ❌ 缺失 |

---

### 1.5 样本 4：getInternetRadioStations

**调用链路：**
```
GetInternetRadios (radio.go:57-93)  // 直接在函数内做兼容判定
```

**兼容判定代码：**
```go
// radio.go:72-78
for _, radio := range radios {
    player, _ := request.PlayerFrom(ctx)  // 忽略 ok！
    if strings.Contains(conf.Server.Subsonic.LegacyClients, player.Client) {
        continue  // legacy：跳过 OpenSubsonicRadio 赋值
    }
    radio.OpenSubsonicRadio = &responses.OpenSubsonicRadio{CoverArt: radio.CoverArt}
}
```

**player 注册失败后的行为：**
- 同样触发空字符串匹配 bug
- **无条件进入 legacy 分支**，`continue` 跳过赋值
- 最终响应中 **所有 radio 都缺少 coverArt 字段**

**字段变化对比：**

| 字段 | 正常（有 player） | 异常（无 player） |
|------|------------------|------------------|
| `id` | ✓ | ✓ |
| `name` | ✓ | ✓ |
| `streamUrl` | ✓ | ✓ |
| `homepageUrl` | ✓ | ✓ |
| `coverArt` | ✓（现代客户端） | ❌ 缺失 |

---

### 1.6 四个样本行为汇总

| 样本端点 | 是否经过 getPlayer | 兼容判定点 | player 缺失时判定 | 字段缺失 |
|---------|------------------|-----------|-----------------|---------|
| getArtists | ✅ | toOSArtistID3 | legacy（空字符串匹配） | musicBrainzId、sortName、roles |
| getAlbumList | ✅ | osChildFromAlbum | legacy（空字符串匹配） | musicBrainzId、sortName、originalReleaseDate、isCompilation |
| getAlbumList2 | ✅ | buildOSAlbumID3 | legacy（空字符串匹配） | 7 个 OpenSubsonic 扩展字段 |
| getInternetRadioStations | ✅ | GetInternetRadios 内联 | legacy（空字符串匹配） | coverArt |

**一致性结论：** 四个样本端点在 player 注册失败时的行为**完全一致**——都因为空字符串匹配 bug 被错误地判定为 legacy 客户端，导致 OpenSubsonic 扩展字段缺失。

---

## 二、最小可落地测试清单

### 2.1 测试设计原则

1. **最小可行**：每个测试只验证一个行为
2. **可落地**：使用现有测试框架（Ginkgo/Gomega），无需新增依赖
3. **高价值**：覆盖回归风险最高的场景
4. **可自动化**：可直接集成到 CI 流水线

### 2.2 测试清单

#### 测试组 1：getArtists 端点（3 个测试）

**测试 1.1：有 legacy client player → 不返回 OpenSubsonic 扩展**
- **前置条件：**
  - `conf.Server.Subsonic.LegacyClients = "legacy-client"`
  - context 中放入 `model.Player{Client: "legacy-client"}`
- **核心断言：**
  ```go
  osArtist := toOSArtistID3(ctx, artist)
  Expect(osArtist).To(BeNil())
  ```
- **预期差异：** 修复前后行为一致（都是 nil）

**测试 1.2：有 modern client player → 返回 OpenSubsonic 扩展**
- **前置条件：**
  - `conf.Server.Subsonic.LegacyClients = "legacy-client"`
  - context 中放入 `model.Player{Client: "modern-client"}`
- **核心断言：**
  ```go
  osArtist := toOSArtistID3(ctx, artist)
  Expect(osArtist).ToNot(BeNil())
  Expect(osArtist.MusicBrainzId).To(Equal("mbz-123"))
  ```
- **预期差异：** 修复前后行为一致（都返回扩展）

**测试 1.3：无 player（注册失败） → 返回 OpenSubsonic 扩展**
- **前置条件：**
  - `conf.Server.Subsonic.LegacyClients = "legacy-client"`
  - context 中**不放入** player（模拟注册失败）
- **核心断言：**
  ```go
  osArtist := toOSArtistID3(ctx, artist)
  Expect(osArtist).ToNot(BeNil())  // 修复前失败，修复后通过
  Expect(osArtist.MusicBrainzId).To(Equal("mbz-123"))
  ```
- **预期差异：** 修复前返回 nil（bug），修复后返回扩展（正确）

---

#### 测试组 2：getAlbumList 端点（3 个测试）

**测试 2.1：有 legacy client player → 不返回 OpenSubsonic 扩展**
- **前置条件：**
  - `conf.Server.Subsonic.LegacyClients = "legacy-client"`
  - context 中放入 `model.Player{Client: "legacy-client"}`
- **核心断言：**
  ```go
  osChild := osChildFromAlbum(ctx, album)
  Expect(osChild).To(BeNil())
  ```
- **预期差异：** 修复前后行为一致

**测试 2.2：有 modern client player → 返回 OpenSubsonic 扩展**
- **前置条件：**
  - `conf.Server.Subsonic.LegacyClients = "legacy-client"`
  - context 中放入 `model.Player{Client: "modern-client"}`
- **核心断言：**
  ```go
  osChild := osChildFromAlbum(ctx, album)
  Expect(osChild).ToNot(BeNil())
  Expect(osChild.MusicBrainzId).To(Equal("mbz-456"))
  ```
- **预期差异：** 修复前后行为一致

**测试 2.3：无 player（注册失败） → 返回 OpenSubsonic 扩展**
- **前置条件：**
  - `conf.Server.Subsonic.LegacyClients = "legacy-client"`
  - context 中**不放入** player
- **核心断言：**
  ```go
  osChild := osChildFromAlbum(ctx, album)
  Expect(osChild).ToNot(BeNil())  // 修复前失败，修复后通过
  ```
- **预期差异：** 修复前返回 nil（bug），修复后返回扩展（正确）

---

#### 测试组 3：getAlbumList2 端点（3 个测试）

**测试 3.1：有 legacy client player → 不返回 OpenSubsonic 扩展**
- **前置条件：**
  - `conf.Server.Subsonic.LegacyClients = "legacy-client"`
  - context 中放入 `model.Player{Client: "legacy-client"}`
- **核心断言：**
  ```go
  osAlbum := buildOSAlbumID3(ctx, album)
  Expect(osAlbum).To(BeNil())
  ```
- **预期差异：** 修复前后行为一致

**测试 3.2：有 modern client player → 返回 OpenSubsonic 扩展**
- **前置条件：**
  - `conf.Server.Subsonic.LegacyClients = "legacy-client"`
  - context 中放入 `model.Player{Client: "modern-client"}`
- **核心断言：**
  ```go
  osAlbum := buildOSAlbumID3(ctx, album)
  Expect(osAlbum).ToNot(BeNil())
  Expect(osAlbum.MusicBrainzId).To(Equal("mbz-789"))
  ```
- **预期差异：** 修复前后行为一致

**测试 3.3：无 player（注册失败） → 返回 OpenSubsonic 扩展**
- **前置条件：**
  - `conf.Server.Subsonic.LegacyClients = "legacy-client"`
  - context 中**不放入** player
- **核心断言：**
  ```go
  osAlbum := buildOSAlbumID3(ctx, album)
  Expect(osAlbum).ToNot(BeNil())  // 修复前失败，修复后通过
  ```
- **预期差异：** 修复前返回 nil（bug），修复后返回扩展（正确）

---

#### 测试组 4：getInternetRadioStations 端点（1 个测试更新）

**测试 4.1：更新现有测试预期**
- **现有测试：** `radio_test.go:101-114`
- **当前（错误）预期：**
  ```go
  Expect(response.InternetRadioStations.Radios[0].OpenSubsonicRadio).To(BeNil())
  ```
- **修复后（正确）预期：**
  ```go
  Expect(response.InternetRadioStations.Radios[0].OpenSubsonicRadio).ToNot(BeNil())
  Expect(response.InternetRadioStations.Radios[0].CoverArt).To(Equal("ra-rd-1_0"))
  ```
- **预期差异：** 测试预期反转

---

#### 测试组 5：子串误判防回归（2 个测试）

**测试 5.1：客户端名是子串不应匹配**
- **前置条件：**
  - `conf.Server.Subsonic.LegacyClients = "foobar2000"`
  - context 中放入 `model.Player{Client: "foo"}`
- **核心断言：**
  ```go
  osArtist := toOSArtistID3(ctx, artist)
  Expect(osArtist).ToNot(BeNil())  // 修复前失败（strings.Contains 误判），修复后通过
  ```
- **预期差异：** 修复前误判为 legacy，修复后正确判定为非 legacy

**测试 5.2：客户端名精确匹配才应匹配**
- **前置条件：**
  - `conf.Server.Subsonic.LegacyClients = "foobar2000"`
  - context 中放入 `model.Player{Client: "foobar2000"}`
- **核心断言：**
  ```go
  osArtist := toOSArtistID3(ctx, artist)
  Expect(osArtist).To(BeNil())  // 修复前后都通过
  ```
- **预期差异：** 修复前后行为一致

---

### 2.3 测试执行矩阵

| 测试编号 | 修复前执行结果 | 修复后执行结果 | 回归验证 |
|---------|--------------|--------------|---------|
| 1.1 | ✅ 通过 | ✅ 通过 | legacy 场景不变 |
| 1.2 | ✅ 通过 | ✅ 通过 | modern 场景不变 |
| 1.3 | ❌ 失败 | ✅ 通过 | 无 player 场景修复 |
| 2.1 | ✅ 通过 | ✅ 通过 | legacy 场景不变 |
| 2.2 | ✅ 通过 | ✅ 通过 | modern 场景不变 |
| 2.3 | ❌ 失败 | ✅ 通过 | 无 player 场景修复 |
| 3.1 | ✅ 通过 | ✅ 通过 | legacy 场景不变 |
| 3.2 | ✅ 通过 | ✅ 通过 | modern 场景不变 |
| 3.3 | ❌ 失败 | ✅ 通过 | 无 player 场景修复 |
| 4.1 | ✅ 通过（错误预期） | ✅ 通过（正确预期） | 测试预期更新 |
| 5.1 | ❌ 失败 | ✅ 通过 | 子串误判修复 |
| 5.2 | ✅ 通过 | ✅ 通过 | 精确匹配不变 |

---

## 三、修复顺序推荐与理由

### 3.1 推荐顺序：先改实现，再修测试

**具体步骤：**

```
步骤 1：修改 4 处实现代码（helpers.go × 3 + radio.go × 1）
    ↓
步骤 2：运行现有测试，验证回归（大部分测试应继续通过）
    ↓
步骤 3：更新 radio_test.go 中不一致的测试预期
    ↓
步骤 4：（可选）新增 toOSArtistID3、osChildFromAlbum、buildOSAlbumID3 的单元测试
    ↓
步骤 5：运行完整测试套件，全绿提交
```

### 3.2 为什么不先修测试？

**反对"测试先行"的理由：**

1. **测试预期本身是错误的**
   - `radio_test.go:101-114` 的测试描述 `"empty client matches legacy list"` 实际上是在描述 bug，而不是预期行为
   - 先修测试意味着要先接受错误的行为作为基准，然后再改回来，会造成两次语义反转

2. **正常场景测试不受影响**
   - 有 legacy client player 的测试（1.1、2.1、3.1）修复前后都通过
   - 有 modern client player 的测试（1.2、2.2、3.2）修复前后都通过
   - 只有无 player 的异常场景测试行为会变化

3. **改动量极小，风险可控**
   - 4 处实现改动，每处只改 2 行代码
   - 改动模式完全一致：`player, _` → `player, ok`，`strings.Contains` → `ok && isClientInList`
   - 可以通过代码评审轻松验证正确性

4. **符合"安全修复"原则**
   - 先改实现，运行测试，观察哪些测试失败
   - 失败的测试就是需要更新预期的测试（目前只有 radio_test.go:101-114）
   - 这种方式不会遗漏任何需要更新的测试

### 3.3 执行细节

**步骤 1：修改实现（4 处，每处 2 行）**

```go
// 统一模式
player, ok := request.PlayerFrom(ctx)
if ok && isClientInList(conf.Server.Subsonic.LegacyClients, player.Client) {
    return nil
}
```

**步骤 2：运行现有测试**

```bash
# 应该只有 radio_test.go:101-114 失败
go test ./server/subsonic/... -run "TestRadio" -v
```

**步骤 3：更新 radio_test.go 预期**

将 `Expect(...OpenSubsonicRadio).To(BeNil())` 改为 `Expect(...OpenSubsonicRadio).ToNot(BeNil())`

**步骤 4：（可选）新增单元测试**

在 `helpers_test.go` 中添加测试组 1、2、3、5 的测试用例。

**步骤 5：完整验证**

```bash
go test ./server/subsonic/... -v
```

### 3.4 风险控制

| 风险 | 发生概率 | 影响 | 缓解措施 |
|------|---------|------|---------|
| 改动引入新 bug | 低 | 4 处改动模式完全一致，代码评审可覆盖 | 要求至少 1 人评审 |
| 其他未发现的测试依赖 bug | 中 | 可能有其他测试隐含依赖空字符串匹配 | 运行完整测试套件，检查失败用例 |
| 部署后用户报告字段变化 | 极低 | 只有 player 注册失败的异常场景会变化 | 发布说明中提及 |

---

## 四、结论

### 4.1 回归风险总结

1. **四个样本端点行为一致**：player 注册失败时都会因为空字符串匹配 bug 被错误判定为 legacy
2. **正常场景零影响**：有 player 的请求（无论 legacy 还是 modern）修复前后行为完全一致
3. **异常场景功能增强**：player 注册失败时，从"不返回 OpenSubsonic 扩展"变为"返回扩展"，是功能增强而非功能破坏
4. **测试缺口大**：3 个 OpenSubsonic 扩展函数完全没有单元测试，只有 radio 有测试但预期错误

### 4.2 落地建议

✅ **立即修复**
- 改动量极小（8 行代码）
- 风险极低（仅影响异常路径）
- 收益明确（修复两个 bug：空字符串匹配 + 子串误判）
- 可验证性强（现有测试框架可覆盖）

❌ **不建议延期**
- bug 已经存在，只是未被触发（player 注册失败概率低）
- 越早修复，测试债务越少
- 后续新增类似函数时，可能会复制粘贴错误的模式

### 代码位置速查

| 改动点 | 文件 | 行号 |
|--------|------|------|
| toOSArtistID3 | `server/subsonic/helpers.go` | 134-138 |
| osChildFromAlbum | `server/subsonic/helpers.go` | 366-370 |
| buildOSAlbumID3 | `server/subsonic/helpers.go` | 453-457 |
| GetInternetRadios | `server/subsonic/radio.go` | 73-77 |
| getPlayer 降级逻辑 | `server/subsonic/middlewares.go` | 192-211 |
| radio 测试（需更新） | `server/subsonic/radio_test.go` | 101-114 |
| 参考实现（osChildFromMediaFile） | `server/subsonic/helpers.go` | 243-247 |
