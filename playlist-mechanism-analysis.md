# Navidrome 播放列表实现机制分析

## 事务口径与位置ID稳定性最终总结

### 一、写路径事务边界汇总

| 操作 | 事务模式 | 并发风险 | 说明 |
|------|----------|----------|------|
| Create | `WithTxImmediate` | 低 | Service 层包裹整个 Put 操作 |
| Update（先删后加） | `WithTxImmediate` | 低 | Service 层包裹 Delete + Add |
| Update（仅元数据） | 无事务 | 极低 | 单条 UPDATE，原子性由 SQLite 保证 |
| Delete（播放列表） | 无事务 | 低 | 单条 DELETE + 文件系统操作 |
| RemoveTracks | `WithTx` | 中 | Service 层包裹 Delete + renumber |
| ReorderTrack | `WithTx` | 中 | Service 层包裹四条 UPDATE |
| **AddTracks/AddAlbums/AddArtists/AddDiscs** | **无事务** | **高** | `SELECT max(id)` + 分块 INSERT，完全非原子 |
| **Native 批量添加四类来源** | **无外层事务** | **极高** | 四类 AddXxx 顺序独立执行，部分失败不回滚 |
| GC/removeOrphans | `WithTx` | 低 | Scanner 层包裹整个 GC 过程 |

### 二、位置ID稳定性边界

| 操作类型 | 位置ID稳定性 | 说明 |
|----------|-------------|------|
| 添加歌曲 | ✅ 稳定 | 仅追加新ID，不改变已有ID |
| 重排序 | ⚠️ 部分稳定 | 仅调整相关位置，其他位置ID不变 |
| 更新元数据 | ✅ 稳定 | 完全不影响 playlist_tracks |
| **删除歌曲（任何形式）** | ❌ 不稳定 | **触发 renumber，所有位置ID全部重新分配** |
| **孤儿清理（GC）** | ❌ 不稳定 | **触发 renumber，所有位置ID全部重新分配** |
| **先删后加 Update** | ❌ 不稳定 | **删除触发 renumber，所有位置ID全部重新分配** |

### 三、核心结论

1. **Add 操作无事务是最大风险点**：`SELECT max(id)` 和分块 INSERT 之间没有事务保护，并发添加会导致唯一约束违反
2. **位置ID是临时标识**：任何删除操作都会触发 renumber，导致所有位置ID重新分配，客户端绝不能缓存
3. **Subsonic 和 Native API 并发安全性都低**：无论是索引还是位置ID，在并发修改场景下都可能错位
4. **设计权衡**：Navidrome 假设播放列表并发修改概率低，优先保证性能和实现简单，牺牲了强一致性

---

## 1. 整体架构分层

播放列表在 Navidrome 中横跨三个协作层：

| 层级 | 职责 | 核心文件 |
|------|------|----------|
| API层 | REST接口、请求处理 | `server/nativeapi/playlists.go` |
| 业务逻辑层 | 权限校验、业务编排 | `core/playlists/playlists.go` |
| 持久层 | 数据读写、事务管理 | `persistence/playlist_repository.go`、`persistence/playlist_track_repository.go` |

**调用链路**：
```
HTTP Request
    ↓
server/nativeapi/playlists.go (Handler)
    ↓
core/playlists/playlists.go (Service)
    ↓
persistence/playlist_repository.go (DAO)
    ↓
SQLite Database
```

---

## 2. 播放列表创建流程

### 2.1 创建入口

创建操作的入口是 `core/playlists/playlists.go:105-134` 的 `Create` 方法，它同时兼容 Subsonic API 的"创建"和"替换"语义：

```go
func (s *playlists) Create(ctx context.Context, playlistId string, name string, ids []string) (string, error) {
    usr, _ := request.UserFrom(ctx)
    err := s.ds.WithTxImmediate(func(tx model.DataStore) error {
        var pls *model.Playlist
        var err error

        if playlistId != "" {
            // 替换现有播放列表
            pls, err = tx.Playlist(ctx).Get(playlistId)
            if err != nil {
                return err
            }
            if pls.IsSmartPlaylist() {
                return model.ErrNotAuthorized
            }
            if !usr.IsAdmin && pls.OwnerID != usr.ID {
                return model.ErrNotAuthorized
            }
        } else {
            // 创建新播放列表
            pls = &model.Playlist{Name: name}
            pls.OwnerID = usr.ID
        }
        pls.Tracks = nil
        pls.AddMediaFilesByID(ids)  // 构建内存中的Tracks列表

        err = tx.Playlist(ctx).Put(pls)  // 持久化
        playlistId = pls.ID
        return err
    })
    return playlistId, err
}
```

### 2.2 数据模型

播放列表的数据结构定义在 `model/playlist.go:12-33`：

```go
type Playlist struct {
    ID               string         `structs:"id" json:"id"`
    Name             string         `structs:"name" json:"name"`
    Comment          string         `structs:"comment" json:"comment"`
    Duration         float32        `structs:"duration" json:"duration"`
    Size             int64          `structs:"size" json:"size"`
    SongCount        int            `structs:"song_count" json:"songCount"`
    OwnerName        string         `structs:"-" json:"ownerName"`
    OwnerID          string         `structs:"owner_id" json:"ownerId"`
    Public           bool           `structs:"public" json:"public"`
    Tracks           PlaylistTracks `structs:"-" json:"tracks,omitempty"`
    Path             string         `structs:"path" json:"path"`
    Sync             bool           `structs:"sync" json:"sync"`
    UploadedImage    string         `structs:"uploaded_image" json:"uploadedImage"`
    ExternalImageURL string         `structs:"external_image_url" json:"externalImageUrl,omitempty"`
    CreatedAt        time.Time      `structs:"created_at" json:"createdAt"`
    UpdatedAt        time.Time      `structs:"updated_at" json:"updatedAt"`

    // SmartPlaylist attributes
    Rules       *criteria.Criteria `structs:"rules" json:"rules"`
    EvaluatedAt *time.Time         `structs:"evaluated_at" json:"evaluatedAt"`
}

type PlaylistTrack struct {
    ID          string `json:"id"`           // 位置序号（1-based）
    MediaFileID string `json:"mediaFileId"`  // 关联的媒体文件ID
    PlaylistID  string `json:"playlistId"`   // 所属播放列表ID
    MediaFile
}
```

**关键设计**：
- `Tracks` 字段标记 `structs:"-"`，不直接映射到数据库字段
- `PlaylistTrack.ID` 存储位置序号，从1开始连续递增
- 智能播放列表通过 `Rules` 字段区分，其歌曲列表由规则动态生成

### 2.3 持久化入口

`playlist_repository.go:100-129` 的 `Put` 方法是持久层的核心入口：

```go
func (r *playlistRepository) Put(p *model.Playlist, cols ...string) error {
    pls := dbPlaylist{Playlist: *p}
    if len(cols) > 0 {
        if pls.ID == "" {
            return errors.New("playlist id is required for partial update")
        }
        _, err := r.put(pls.ID, pls, cols...)
        return err
    }
    if pls.ID == "" {
        pls.CreatedAt = time.Now()
    }
    pls.UpdatedAt = time.Now()

    id, err := r.put(pls.ID, pls)  // 保存playlist元数据
    if err != nil {
        return err
    }
    p.ID = id

    if p.IsSmartPlaylist() {
        // 智能播放列表不立即更新tracks，避免长时间锁定数据库
        return nil
    }
    // 只有指定了tracks才更新关联表
    if len(pls.Tracks) > 0 {
        return r.updateTracks(id, p.MediaFiles())
    }
    return r.refreshCounters(&pls.Playlist)
}
```

---

## 3. 歌曲顺序持久化机制

### 3.1 存储设计

歌曲顺序通过 `playlist_tracks` 表的 `id` 字段实现：

| 字段 | 类型 | 作用 |
|------|------|------|
| `playlist_id` | TEXT | 外键，关联播放列表 |
| `media_file_id` | TEXT | 外键，关联媒体文件 |
| `id` | INTEGER | **位置序号**，从1开始连续递增 |

**联合唯一约束**：`(playlist_id, id)` —— 同一播放列表内位置唯一

### 3.2 新增歌曲时的顺序分配

`playlist_repository.go:229-246` 的 `addTracks` 方法负责批量插入并分配顺序：

```go
func (r *playlistRepository) addTracks(playlistId string, startingPos int, mediaFileIds []string) error {
    // 分块插入避免SQLITE_MAX_VARIABLE_NUMBER限制
    pos := startingPos
    for chunk := range slices.Chunk(mediaFileIds, 200) {
        ins := Insert("playlist_tracks").Columns("playlist_id", "media_file_id", "id")
        for _, t := range chunk {
            ins = ins.Values(playlistId, t, pos)
            pos++
        }
        _, err := r.executeSQL(ins)
        if err != nil {
            return err
        }
    }

    return r.refreshCounters(&model.Playlist{ID: playlistId})
}
```

**获取下一个位置**（`playlist_track_repository.go:150-159`）：
```go
func (r *playlistTrackRepository) Add(mediaFileIds []string) (int, error) {
    // Get next pos (ID) in playlist
    sq := r.newSelect().Columns("max(id) as max").Where(Eq{"playlist_id": r.playlistId})
    var res struct{ Max sql.NullInt32 }
    err := r.queryOne(sq, &res)
    if err != nil {
        return 0, err
    }

    return len(mediaFileIds), r.playlistRepo.addTracks(r.playlistId, int(res.Max.Int32+1), mediaFileIds)
}
```

**关键点**：
- 起始位置 = `max(id) + 1`
- 每批次200条，避免SQLite参数限制（默认999）
- 插入完成后自动刷新播放列表统计信息（时长、大小、歌曲数）

### 3.3 全量替换流程

当需要完全替换播放列表内容时（`playlist_repository.go:218-227`）：

```go
func (r *playlistRepository) updatePlaylist(playlistId string, mediaFileIds []string) error {
    // Remove old tracks
    del := Delete("playlist_tracks").Where(Eq{"playlist_id": playlistId})
    _, err := r.executeSQL(del)
    if err != nil {
        return err
    }

    return r.addTracks(playlistId, 1, mediaFileIds)
}
```

**流程**：先删除所有旧记录 → 重新从位置1开始插入

### 3.4 歌曲重排序

`playlist_track_repository.go:210-248` 的 `Reorder` 方法实现单条歌曲位置调整，采用**四步算法**避免唯一约束冲突：

```go
func (r *playlistTrackRepository) Reorder(pos int, newPos int) error {
    if pos == newPos {
        return nil
    }
    pid := r.playlistId

    // Step 1: 将源位置标记为临时值 -999999
    _, err := r.executeSQL(Expr(
        `UPDATE playlist_tracks SET id = -999999 WHERE playlist_id = ? AND id = ?`, pid, pos))
    if err != nil {
        return err
    }

    // Step 2: 移动受影响范围（使用负值避免冲突）
    if pos < newPos {
        // 下移：pos+1 ~ newPos 整体减1
        _, err = r.executeSQL(Expr(
            `UPDATE playlist_tracks SET id = -(id - 1) WHERE playlist_id = ? AND id > ? AND id <= ?`,
            pid, pos, newPos))
    } else {
        // 上移：newPos ~ pos-1 整体加1
        _, err = r.executeSQL(Expr(
            `UPDATE playlist_tracks SET id = -(id + 1) WHERE playlist_id = ? AND id >= ? AND id < ?`,
            pid, newPos, pos))
    }
    if err != nil {
        return err
    }

    // Step 3: 负号翻转回正值
    _, err = r.executeSQL(Expr(
        `UPDATE playlist_tracks SET id = -id WHERE playlist_id = ? AND id < 0 AND id != -999999`, pid))
    if err != nil {
        return err
    }

    // Step 4: 源位置放到目标位置
    _, err = r.executeSQL(Expr(
        `UPDATE playlist_tracks SET id = ? WHERE playlist_id = ? AND id = -999999`, newPos, pid))
    return err
}
```

**示例**：将位置2的歌曲移到位置4
1. 原顺序：[1, 2, 3, 4, 5]
2. Step 1: [1, -999999, 3, 4, 5]
3. Step 2 (pos < newPos): [1, -999999, -2, -3, 5]  (3→-2, 4→-3)
4. Step 3: [1, -999999, 2, 3, 5]
5. Step 4: [1, 4, 2, 3, 5] → 结果正确

---

## 4. 持久层更新入口汇总

| 操作 | 入口方法 | 事务边界（Service层） | 核心逻辑 |
|------|----------|----------------------|----------|
| 创建/替换 | `Put(pls *Playlist)` | `WithTxImmediate` | 先存元数据，再调用 `updateTracks` |
| 添加歌曲 | `Tracks().Add(ids)` | **无事务** | `SELECT max(id)` + 分块 `INSERT`，完全无事务 |
| 按专辑添加 | `Tracks().AddAlbums(ids)` | **无事务** | 查询专辑歌曲后调用 `Add` |
| 按艺术家添加 | `Tracks().AddArtists(ids)` | **无事务** | 查询艺术家歌曲后调用 `Add` |
| 按碟片添加 | `Tracks().AddDiscs(ids)` | **无事务** | 查询碟片歌曲后调用 `Add` |
| 删除歌曲 | `Tracks().Delete(ids)` | `WithTx` | 删除后调用 `renumber()` 重编号 |
| 重排序 | `Tracks().Reorder(pos, newPos)` | `WithTx` | 四步算法原地调整 |
| 更新元数据 | `Put(pls, cols...)` | 无事务 | 仅更新指定字段（名称、注释、公开状态） |
| 先删后加更新 | `Update(playlistID, ...)` | `WithTxImmediate` | 先 Delete 再 Add，中间自动 renumber |
| 统计刷新 | `refreshCounters(pls)` | 无事务 | 聚合查询更新 duration/size/song_count |
| 孤儿清理 | `removeOrphans()` | `WithTx` (外层GC事务) | 删除无效引用 + renumber |

---

## 5. 事务边界详解

### 5.1 两种事务模式

Navidrome 提供两种事务模式，定义在 `persistence/persistence.go:131-168`：

**普通事务 `WithTx`**：
```go
func (s *SQLStore) WithTx(block func(tx model.DataStore) error, scope ...string) error {
    conn, inTx := s.db.(*dbx.DB)
    if !inTx {
        log.Trace("Nested Transaction started", "scope", msg)
        conn = dbx.NewFromDB(db.Db(), db.Driver)
    }
    return conn.Transactional(func(tx *dbx.Tx) error {
        newDb := &SQLStore{db: tx}
        err := block(newDb)
        return err
    })
}
```

**立即事务 `WithTxImmediate`**：
```go
func (s *SQLStore) WithTxImmediate(block func(tx model.DataStore) error, scope ...string) error {
    ctx := context.Background()
    return s.WithTx(func(tx model.DataStore) error {
        // Workaround to force the transaction to be upgraded to immediate mode to avoid deadlocks
        _ = tx.Property(ctx).Put("tmp_lock_flag", "")
        defer func() {
            _ = tx.Property(ctx).Delete("tmp_lock_flag")
        }()
        return block(tx)
    }, scope...)
}
```

**区别**：
- `WithTxImmediate` 通过执行一个写操作（`Put` 临时属性）强制 SQLite 将事务升级为 IMMEDIATE 模式，避免死锁
- `WithTx` 使用默认的 DEFERRED 模式，适用于只读或轻量写操作

### 5.2 各操作的事务边界（最终统一口径）

| 操作 | 入口文件 | 事务模式 | 事务位置 | 并发风险 |
|------|----------|----------|----------|----------|
| Create | `core/playlists/playlists.go:107` | `WithTxImmediate` | Service 层包裹整个 Put 操作 | 低（立即事务避免死锁） |
| Update（先删后加） | `core/playlists/playlists.go:166` | `WithTxImmediate` | Service 层包裹 Delete + Add | 低（原子执行） |
| Update（仅元数据） | `core/playlists/playlists.go:197` | 无事务 | 直接调用 Put(pls, cols...) | 极低（单条 UPDATE） |
| Delete（播放列表） | `core/playlists/playlists.go:149` | 无事务 | 直接调用 Delete(id) | 低（单条 DELETE + 文件操作） |
| RemoveTracks | `core/playlists/playlists.go:278` | `WithTx` | Service 层包裹 Delete | 中（事务内 renumber） |
| ReorderTrack | `core/playlists/playlists.go:287` | `WithTx` | Service 层包裹 Reorder | 中（事务内四条 UPDATE） |
| **AddTracks** | `core/playlists/playlists.go:252` | **无事务** | 直接调用 Add(ids) | **高（SELECT max(id) 和 INSERT 非原子）** |
| AddAlbums | `core/playlists/playlists.go:258` | **无事务** | 直接调用 AddAlbums(ids) | **高** |
| AddArtists | `core/playlists/playlists.go:265` | **无事务** | 直接调用 AddArtists(ids) | **高** |
| AddDiscs | `core/playlists/playlists.go:271` | **无事务** | 直接调用 AddDiscs(ids) | **高** |
| **Native 批量添加四类来源** | `server/nativeapi/playlists.go:121-167` | **无外层事务** | Handler 内顺序调用四类 AddXxx | **极高（四类操作独立，部分失败不回滚）** |
| GC/removeOrphans | `scanner/scanner.go:233` | `WithTx` | Scanner 层包裹整个 GC | 低（整个GC在一个事务中） |

#### 5.2.1 Add 操作的并发风险详解

`AddTracks` 内部的 `SELECT max(id)` 和 `INSERT` 之间没有事务保护：

```go
// persistence/playlist_track_repository.go:143-159
func (r *playlistTrackRepository) Add(mediaFileIds []string) (int, error) {
    // Step 1: 查询 max(id) - 自动提交
    sq := r.newSelect().Columns("max(id) as max").Where(Eq{"playlist_id": r.playlistId})
    var res struct{ Max sql.NullInt32 }
    err := r.queryOne(sq, &res)
    
    // Step 2: 分块插入 - 每块自动提交
    return len(mediaFileIds), r.playlistRepo.addTracks(r.playlistId, int(res.Max.Int32+1), mediaFileIds)
}
```

**并发冲突时序**：
```
时间点 | 请求A | 请求B
-------|-------|-------
  T1   | SELECT max(id) → 100 |
  T2   |       | SELECT max(id) → 100
  T3   | INSERT 从 101 开始 |
  T4   |       | INSERT 也从 101 开始 → UNIQUE constraint failed!
```

#### 5.2.2 事务模式选择的设计逻辑

| 事务模式 | 使用场景 | 设计考量 |
|----------|----------|----------|
| `WithTxImmediate` | Create、Update（先删后加） | 涉及多个表修改，避免 SQLite 死锁 |
| `WithTx` | RemoveTracks、ReorderTrack、GC | 多条 SQL 需要原子性，但并发概率低 |
| 无事务 | AddTracks、AddAlbums 等 | 性能优先，假设并发添加概率低 |

### 5.3 事务嵌套支持

`WithTx` 方法检测当前是否已在事务中：
```go
conn, inTx := s.db.(*dbx.DB)
if !inTx {
    // 已在事务中，创建新连接（嵌套事务）
    conn = dbx.NewFromDB(db.Db(), db.Driver)
}
```
SQLite 不支持真正的嵌套事务，这里通过新建连接模拟。

---

## 6. Native 批量添加四类来源的事务边界与部分成功风险

### 6.1 四类添加来源

Native API 支持同时从四个来源批量添加歌曲，入口在 `server/nativeapi/playlists.go:121-167` 的 `addToPlaylist` 方法：

```go
type addTracksPayload struct {
    Ids       []string       `json:"ids"`       // 直接指定歌曲ID
    AlbumIds  []string       `json:"albumIds"`  // 按专辑添加
    ArtistIds []string       `json:"artistIds"` // 按艺术家添加
    Discs     []model.DiscID `json:"discs"`     // 按碟片添加
}

func addToPlaylist(pls playlists.Playlists) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // ... 解析 payload
        
        count, c := 0, 0
        if c, err = pls.AddTracks(ctx, playlistId, payload.Ids); err != nil {
            http.Error(w, err.Error(), http.StatusBadRequest)
            return
        }
        count += c
        if c, err = pls.AddAlbums(ctx, playlistId, payload.AlbumIds); err != nil {
            http.Error(w, err.Error(), http.StatusBadRequest)
            return
        }
        count += c
        if c, err = pls.AddArtists(ctx, playlistId, payload.ArtistIds); err != nil {
            http.Error(w, err.Error(), http.StatusBadRequest)
            return
        }
        count += c
        if c, err = pls.AddDiscs(ctx, playlistId, payload.Discs); err != nil {
            http.Error(w, err.Error(), http.StatusBadRequest)
            return
        }
        count += c
        
        // 返回成功添加的总数
        _, err = fmt.Fprintf(w, `{"added":%d}`, count)
    }
}
```

### 6.2 各来源的解析与添加流程

四类来源最终都通过 `addMediaFileIds` 方法统一处理（`persistence/playlist_track_repository.go:161-189`）：

```go
func (r *playlistTrackRepository) addMediaFileIds(cond Sqlizer) (int, error) {
    // 先根据条件查询出所有符合的媒体文件ID
    sq := Select("id").From("media_file").Where(cond).
        OrderBy("album_artist, album, release_date, disc_number, track_number")
    var ids []string
    err := r.queryAllSlice(sq, &ids)
    if err != nil {
        return 0, err
    }
    // 再调用 Add 方法批量添加
    return r.Add(ids)
}

func (r *playlistTrackRepository) AddAlbums(albumIds []string) (int, error) {
    return r.addMediaFileIds(Eq{"album_id": albumIds})
}

func (r *playlistTrackRepository) AddArtists(artistIds []string) (int, error) {
    return r.addMediaFileIds(Eq{"album_artist_id": artistIds})
}

func (r *playlistTrackRepository) AddDiscs(discs []model.DiscID) (int, error) {
    var clauses Or
    for _, d := range discs {
        clauses = append(clauses, And{
            Eq{"album_id": d.AlbumID},
            Eq{"release_date": d.ReleaseDate},
            Eq{"disc_number": d.DiscNumber},
        })
    }
    return r.addMediaFileIds(clauses)
}
```

**添加顺序**：
1. Ids → 按传入顺序添加
2. AlbumIds → 按 `album_artist, album, release_date, disc_number, track_number` 排序后添加
3. ArtistIds → 同上排序
4. Discs → 同上排序

### 6.3 追加写入的实际 SQL 执行路径与事务边界

#### 6.3.1 完整执行路径

`Add` 方法的完整执行路径如下（`persistence/playlist_track_repository.go:143-159`）：

```go
func (r *playlistTrackRepository) Add(mediaFileIds []string) (int, error) {
    // Step 1: 查询当前最大位置ID
    sq := r.newSelect().Columns("max(id) as max").Where(Eq{"playlist_id": r.playlistId})
    var res struct{ Max sql.NullInt32 }
    err := r.queryOne(sq, &res)  // 第一条SQL: SELECT max(id)
    if err != nil {
        return 0, err
    }

    // Step 2: 分块插入，每块200条
    return len(mediaFileIds), r.playlistRepo.addTracks(r.playlistId, int(res.Max.Int32+1), mediaFileIds)
}
```

`addTracks` 内部实现（`persistence/playlist_repository.go:229-246`）：

```go
func (r *playlistRepository) addTracks(playlistId string, startingPos int, mediaFileIds []string) error {
    pos := startingPos
    for chunk := range slices.Chunk(mediaFileIds, 200) {
        ins := Insert("playlist_tracks").Columns("playlist_id", "media_file_id", "id")
        for _, t := range chunk {
            ins = ins.Values(playlistId, t, pos)
            pos++
        }
        _, err := r.executeSQL(ins)  // 每块一条独立的 INSERT SQL
        if err != nil {
            return err  // 失败立即返回，前面已插入的块不会回滚
        }
    }
    return r.refreshCounters(&model.Playlist{ID: playlistId})  // 最后一条SQL: UPDATE 统计信息
}
```

#### 6.3.2 真实事务边界分析

**关键发现：完全没有事务包裹！**

| 操作 | SQL 语句 | 事务状态 |
|------|----------|----------|
| 查询 max(id) | `SELECT max(id) FROM playlist_tracks WHERE playlist_id = ?` | 自动提交 |
| 插入第1块 | `INSERT INTO playlist_tracks ...` (200条) | 自动提交 |
| 插入第2块 | `INSERT INTO playlist_tracks ...` (200条) | 自动提交 |
| ... | ... | ... |
| 刷新统计 | `UPDATE playlist SET duration=?, size=?, song_count=? WHERE id=?` | 自动提交 |

**并发安全问题：**

`SELECT max(id)` 和后续 `INSERT` 之间没有事务，也没有锁：
1. 请求A：`SELECT max(id)` → 得到 100
2. 请求B：`SELECT max(id)` → 也得到 100
3. 请求A：`INSERT` 从 101 开始
4. 请求B：`INSERT` 也从 101 开始 → **唯一约束违反！**

#### 6.3.3 Service 层的事务边界

四类添加操作在 Service 层也没有外层事务（`core/playlists/playlists.go:246-272`）：

```go
func (s *playlists) AddTracks(ctx context.Context, playlistID string, ids []string) (int, error) {
    if _, err := s.checkTracksEditable(ctx, playlistID); err != nil {
        return 0, err
    }
    return s.ds.Playlist(ctx).Tracks(playlistID, false).Add(ids)  // 无事务
}

func (s *playlists) AddAlbums(ctx context.Context, playlistID string, albumIds []string) (int, error) {
    if _, err := s.checkTracksEditable(ctx, playlistID); err != nil {
        return 0, err
    }
    return s.ds.Playlist(ctx).Tracks(playlistID, false).AddAlbums(albumIds)  // 无事务
}
// AddArtists 和 AddDiscs 同理
```

### 6.4 部分成功风险

#### 6.4.1 单 Add 调用内的部分成功

**风险场景**：一次添加 500 首歌，分 3 块插入
1. 第1块（200条）：成功提交
2. 第2块（200条）：成功提交
3. 第3块（100条）：失败（如数据库连接断开）
4. 结果：前 400 首歌已永久添加，后 100 首丢失

#### 6.4.2 Native 批量添加四类来源的部分成功

**风险场景**：
1. 请求同时包含 `ids: ["track1", "track2"]` 和 `albumIds: ["album1"]`
2. `AddTracks` 成功添加了2首歌
3. `AddAlbums` 查询 `album1` 的歌曲时出错（如数据库连接问题）
4. 整个请求返回错误，但 **track1 和 track2 已经永久添加到播放列表**

**风险级别**：高
- 用户看到错误提示，以为全部失败
- 实际上部分歌曲已添加
- 再次请求会导致重复添加

**设计意图分析**：
- 追求性能：避免大事务锁定数据库
- 简化实现：四类来源独立处理
- 但牺牲了原子性，属于典型的"性能 vs 一致性"权衡

---

## 7. 先删后加路径对最终顺序的影响

### 7.1 先删后加的代码路径

Subsonic API 的 `UpdatePlaylist` 方法支持同时删除和添加歌曲，核心逻辑在 `core/playlists/playlists.go:152-199`：

```go
func (s *playlists) Update(ctx context.Context, playlistID string,
    name *string, comment *string, public *bool,
    idsToAdd []string, idxToRemove []int) error {
    
    return s.ds.WithTxImmediate(func(tx model.DataStore) error {
        repo := tx.Playlist(ctx)

        if len(idxToRemove) > 0 {
            tracksRepo := repo.Tracks(playlistID, false)
            // 将0-based索引转换为1-based位置ID
            positions := make([]string, len(idxToRemove))
            for i, idx := range idxToRemove {
                positions[i] = strconv.Itoa(idx + 1)
            }
            // 第一步：删除指定位置的歌曲
            if err := tracksRepo.Delete(positions...); err != nil {
                return err
            }
            // 第二步：删除后自动 renumber（在 Delete 内部调用）
            // 第三步：添加新歌曲（追加到末尾）
            if len(idsToAdd) > 0 {
                if _, err := tracksRepo.Add(idsToAdd); err != nil {
                    return err
                }
            }
            return s.updateMetadata(ctx, tx, pls, name, comment, public)
        }

        // 只有添加没有删除的情况
        if len(idsToAdd) > 0 {
            if _, err := repo.Tracks(playlistID, false).Add(idsToAdd); err != nil {
                return err
            }
        }
        // ...
    })
}
```

### 7.2 对最终顺序的影响

**关键观察**：删除后立即 renumber，然后添加的歌曲追加到末尾。

**示例**：
假设播放列表原有顺序：[A, B, C, D, E]（位置1-5）

请求：`idxToRemove=[1, 3]`（删除B和D，0-based索引），`idsToAdd=[X, Y]`

执行流程：
1. 转换索引：`idxToRemove=[1,3]` → `positions=["2","4"]`（位置ID）
2. 删除位置2和4 → 临时状态：[A, C, E]（位置1,3,5，有间隙）
3. **Delete 内部调用 renumber** → 重编号为：[A(1), C(2), E(3)]
4. **Add 追加 X, Y** → 查询 `max(id)=3`，从位置4开始添加
5. 最终顺序：[A, C, E, X, Y]

**重要结论**：
- 删除的位置基于**操作前**的播放列表状态
- 添加的歌曲总是在**删除并修复后**的列表末尾追加
- 新增歌曲永远不会插入到被删除的位置

### 7.3 索引转换的正确性

代码注释明确说明了转换逻辑：
```go
// Convert 0-based indices to 1-based position IDs and delete them directly,
// avoiding the need to load all tracks into memory.
```

**为什么不需要先加载所有 tracks？**
- Subsonic API 约定 `songIndexToRemove` 是基于当前播放列表的 0-based 索引
- 直接转换为 1-based 位置ID即可删除
- 但这要求调用方确保索引是基于最新状态的

---

## 8. Subsonic 与 Native 在删除参数语义上的差异及其与重编号的关系

### 8.1 两种 API 的删除参数对比

| API | 参数名称 | 语义 | 类型 |
|-----|----------|------|------|
| **Subsonic** | `songIndexToRemove` | 0-based **索引**（基于当前列表顺序） | `[]int` |
| **Native** | `id` | 1-based **位置ID**（数据库中的 `playlist_tracks.id`） | `[]string` |

### 8.2 Subsonic 删除流程

**入口**：`server/subsonic/playlists.go:97-128` 的 `UpdatePlaylist`

```go
func (api *Router) UpdatePlaylist(r *http.Request) (*responses.Subsonic, error) {
    p := req.Params(r)
    playlistId, _ := p.String("playlistId")
    songsToAdd, _ := p.Strings("songIdToAdd")
    songIndexesToRemove, _ := p.Ints("songIndexToRemove")  // 0-based 索引数组
    
    // 传递给 service 层
    err = api.playlists.Update(r.Context(), playlistId, plsName, comment, public, songsToAdd, songIndexesToRemove)
    // ...
}
```

**Service 层处理**：`core/playlists/playlists.go:169-176`

```go
if len(idxToRemove) > 0 {
    tracksRepo := repo.Tracks(playlistID, false)
    // 0-based 索引 → 1-based 位置ID
    positions := make([]string, len(idxToRemove))
    for i, idx := range idxToRemove {
        positions[i] = strconv.Itoa(idx + 1)
    }
    if err := tracksRepo.Delete(positions...); err != nil {
        return err
    }
    // ...
}
```

### 8.3 Native 删除流程

**入口**：`server/nativeapi/playlists.go:101-119` 的 `deleteFromPlaylist`

```go
func deleteFromPlaylist(pls playlists.Playlists) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        p := req.Params(r)
        playlistId, _ := p.String(":playlistId")
        ids, _ := p.Strings("id")  // 直接是位置ID字符串
        
        err := pls.RemoveTracks(r.Context(), playlistId, ids)
        // ...
    }
}
```

**Service 层直接传递**：`core/playlists/playlists.go:274-281`

```go
func (s *playlists) RemoveTracks(ctx context.Context, playlistID string, trackIds []string) error {
    if _, err := s.checkTracksEditable(ctx, playlistID); err != nil {
        return err
    }
    return s.ds.WithTx(func(tx model.DataStore) error {
        return tx.Playlist(ctx).Tracks(playlistID, false).Delete(trackIds...)  // 直接用传入的 ids
    })
}
```

### 8.4 与重编号的真实关系

#### 8.4.1 renumber 会改变所有位置ID！

**关键纠正**：之前错误地认为 Native API 的位置ID是稳定的。实际上，`renumber` 方法会**重新分配所有位置ID**！

`renumber` 的核心逻辑（`persistence/playlist_repository.go:389-411`）：
```go
func (r *playlistRepository) renumber(id string) error {
    // Step 1: 将所有ID取反
    UPDATE playlist_tracks SET id = -id WHERE playlist_id = ? AND id > 0
    
    // Step 2: 重新分配从1开始的连续ID
    WITH new_ids AS (
        SELECT rowid as rid, ROW_NUMBER() OVER (ORDER BY id DESC) as new_id
        FROM playlist_tracks WHERE playlist_id = ?
    )
    UPDATE playlist_tracks SET id = new_ids.new_id
    FROM new_ids
    WHERE playlist_tracks.rowid = new_ids.rid AND playlist_tracks.playlist_id = ?
}
```

**结论**：任何触发 renumber 的操作（删除歌曲、孤儿清理）都会导致**所有歌曲的位置ID被重新分配**。

#### 8.4.2 Subsonic 索引的脆弱性

Subsonic 的 `songIndexToRemove` 是基于"播放列表当前顺序"的索引，但这个索引只在**调用时**有效。

**并发风险示例**：
1. 用户A看到播放列表：[A, B, C]（索引0,1,2，对应位置ID 1,2,3）
2. 用户B删除了歌曲A → 触发 renumber
3. renumber 后：[B(1), C(2)]（所有位置ID都变了！）
4. 用户A发送请求删除索引1（想删B）
5. 服务端转换：索引1 → 位置ID "2"
6. 删除位置2 → 实际删除了C，而不是B！

#### 8.4.3 Native API 位置ID的真实风险

Native API 直接使用位置ID，但位置ID会被 renumber 改变，所以**同样不稳定**！

**并发风险示例**：
1. 用户A看到播放列表：[A(1), B(2), C(3)]
2. 用户B删除了歌曲A → 触发 renumber
3. renumber 后：[B(1), C(2)]（B的位置ID从2变成1，C从3变成2）
4. 用户A发送请求删除 `id=2`（想删B）
5. 此时位置ID 2 是C → 删除了C，而不是B！

**更隐蔽的风险**：
- 如果 renumber 发生在用户获取列表和发送删除请求之间
- 用户想删的歌曲可能已经移动到其他位置ID
- 结果：要么删除了错误的歌曲，要么返回"not found"（如果位置ID已不存在）

### 8.5 批量删除的顺序问题

**Subsonic 批量删除多个索引**：

假设列表：[A, B, C, D, E]（索引0-4，位置ID 1-5）
请求删除索引：[1, 3]（想删B和D）

```go
idxToRemove = [1, 3]
// 转换为 positions = ["2", "4"]
// SQL: DELETE FROM playlist_tracks WHERE playlist_id = ? AND id IN ("2", "4")
```

这是正确的，因为：
- 删除操作是基于**操作前**的原始位置
- `IN` 操作不关心参数顺序
- 删除后 renumber 会修复间隙

**但如果索引是乱序的**：
`idxToRemove = [3, 1]` → `positions = ["4", "2"]` → 结果相同，不影响正确性

### 8.6 关键差异总结

| 维度 | Subsonic API | Native API |
|------|-------------|------------|
| 参数类型 | 0-based 索引数组 | 1-based 位置ID字符串数组 |
| 转换层 | Service 层 `idx+1` 转换 | 无转换，直接传递 |
| **并发安全性** | **低（索引可能错位）** | **同样低（位置ID会被renumber改变）** |
| 与 renumber 关系 | 依赖 renumber 保持索引连续 | 受 renumber 直接影响，位置ID动态变化 |
| 批量删除顺序 | 不影响结果 | 不影响结果 |
| 风险表现 | 索引错位删除错误歌曲 | 位置ID重分配删除错误歌曲 |

### 8.7 位置 ID 稳定性边界（最终结论）

#### 8.7.1 哪些操作会改变所有位置ID？

**renumber 触发场景**（完整列表）：

| 场景 | 触发点 | 位置ID变化 |
|------|--------|------------|
| 用户主动删除歌曲 | `playlist_track_repository.go:197` | ✅ 全部重新分配 |
| 清空播放列表 | `playlist_track_repository.go:206` | ✅ 全部重新分配 |
| 孤儿清理（GC） | `playlist_repository.go:379` | ✅ 全部重新分配 |
| 先删后加 Update | `Delete` 内部调用 | ✅ 全部重新分配 |
| **仅添加歌曲** | `addTracks` | ❌ 不改变已有ID |
| **仅重排序** | `Reorder` | ❌ 仅调整相关位置 |
| **仅更新元数据** | `Put(pls, cols...)` | ❌ 不影响 |

#### 8.7.2 稳定性边界总结

**位置ID在以下场景是稳定的**：
- 播放列表没有任何删除操作（用户删除、GC孤儿清理）
- 仅进行添加、重排序、元数据更新操作

**位置ID在以下场景会失效**：
- 任何删除操作（用户主动删除、GC清理）
- 先删后加的 Update 操作

#### 8.7.3 客户端最佳实践

1. **不要缓存位置ID**：每次操作前重新获取播放列表
2. **操作原子性**：获取列表后立即执行删除/修改操作
3. **并发操作提示**：如果播放列表可能被多人修改，使用 `updated_at` 字段检测冲突
4. **幂等性设计**：删除失败时不要盲目重试，先重新获取列表

**重要结论**：
- Subsonic 和 Native API 在并发场景下都不安全
- 根本原因：`renumber` 会动态改变所有位置ID
- 位置ID是**临时的顺序标识**，不是**永久的唯一标识**

---

## 9. 删除歌曲后的列表修复机制

### 9.1 两种删除场景

Navidrome 有两种歌曲删除场景，修复机制不同：

#### 场景 A：用户主动从播放列表移除歌曲

**入口**：`core/playlists/playlists.go:274-281`

```go
func (s *playlists) RemoveTracks(ctx context.Context, playlistID string, trackIds []string) error {
    if _, err := s.checkTracksEditable(ctx, playlistID); err != nil {
        return err
    }
    return s.ds.WithTx(func(tx model.DataStore) error {
        return tx.Playlist(ctx).Tracks(playlistID, false).Delete(trackIds...)
    })
}
```

**删除后立即修复**（`playlist_track_repository.go:191-198`）：
```go
func (r *playlistTrackRepository) Delete(ids ...string) error {
    err := r.delete(And{Eq{"playlist_id": r.playlistId}, Eq{"id": ids}})
    if err != nil {
        return err
    }

    return r.playlistRepo.renumber(r.playlistId)  // 立即重编号
}
```

#### 场景 B：媒体文件从库中删除（孤儿清理）

当媒体文件被扫描器检测到不存在并删除时，`playlist_tracks` 表中会产生**孤儿记录**（`media_file_id` 指向不存在的记录）。

### 9.2 孤儿清理触发时机

孤儿清理是垃圾回收（GC）的一部分，触发链如下：

```
扫描完成 (scanner/controller.go)
    ↓
scanner.runGC() (scanner.go:230-256) 检测到 changesDetected
    ↓
SQLStore.GC() (persistence.go:170-202)
    ↓
按顺序执行清理任务（顺序执行）
    ↓
playlistRepository.removeOrphans() (playlist_repository.go:353-384)
```

**GC 任务序列**（`persistence.go:186-197`）：
```go
err := run.Sequentially(
    trace(ctx, "purge empty albums", ...),
    trace(ctx, "purge empty artists", ...),
    trace(ctx, "mark missing artists", ...),
    trace(ctx, "purge empty folders", ...),
    trace(ctx, "clean album annotations", ...),
    trace(ctx, "clean artist annotations", ...),
    trace(ctx, "clean media file annotations", ...),
    trace(ctx, "clean media file bookmarks", ...),
    trace(ctx, "purge non used tags", ...),
    trace(ctx, "remove orphan playlist tracks", ...),  // 播放列表孤儿清理
)
```

### 9.3 孤儿清理实现

`playlist_repository.go:353-384` 的 `removeOrphans` 方法：

```go
func (r *playlistRepository) removeOrphans() error {
    // Step 1: 找出所有包含孤儿track的播放列表
    sel := Select("playlist_tracks.playlist_id as id", "p.name").From("playlist_tracks").
        Join("playlist p on playlist_tracks.playlist_id = p.id").
        LeftJoin("media_file mf on playlist_tracks.media_file_id = mf.id").
        Where(Eq{"mf.id": nil}).
        GroupBy("playlist_tracks.playlist_id")

    var pls []struct{ Id, Name string }
    err := r.queryAll(sel, &pls)
    if err != nil {
        return fmt.Errorf("fetching playlists with orphan tracks: %w", err)
    }

    for _, pl := range pls {
        log.Debug(r.ctx, "Cleaning-up orphan tracks from playlist", "id", pl.Id, "name", pl.Name)
        del := Delete("playlist_tracks").Where(And{
            ConcatExpr("media_file_id not in (select id from media_file)"),
            Eq{"playlist_id": pl.Id},
        })
        n, err := r.executeSQL(del)
        if n == 0 || err != nil {
            return fmt.Errorf("deleting orphan tracks from playlist %s: %w", pl.Name, err)
        }
        log.Debug(r.ctx, "Deleted tracks, now reordering", "id", pl.Id, "name", pl.Name, "deleted", n)

        // Renumber the playlist if any track was removed
        if err := r.renumber(pl.Id); err != nil {
            return fmt.Errorf("renumbering playlist %s: %w", pl.Name, err)
        }
    }
    return nil
}
```

### 9.4 重编号（renumber）算法

`playlist_repository.go:389-411` 的 `renumber` 方法采用**两步CTE算法**避免唯一约束冲突：

```go
func (r *playlistRepository) renumber(id string) error {
    // Step 1: 将所有ID取反，清空正整数空间
    _, err := r.executeSQL(Expr(
        `UPDATE playlist_tracks SET id = -id WHERE playlist_id = ? AND id > 0`, id))
    if err != nil {
        return err
    }
    // Step 2: 使用CTE生成新的连续序号并更新
    // The CTE is fully materialized before the UPDATE begins, avoiding self-referencing issues.
    // ORDER BY id DESC restores original order since IDs are now negative.
    _, err = r.executeSQL(Expr(
        `WITH new_ids AS (
            SELECT rowid as rid, ROW_NUMBER() OVER (ORDER BY id DESC) as new_id
            FROM playlist_tracks WHERE playlist_id = ?
        )
        UPDATE playlist_tracks SET id = new_ids.new_id
        FROM new_ids
        WHERE playlist_tracks.rowid = new_ids.rid AND playlist_tracks.playlist_id = ?`, id, id))
    if err != nil {
        return err
    }
    return r.refreshCounters(&model.Playlist{ID: id})
}
```

**算法说明**：

假设播放列表有5首歌，删除位置2和4：
1. 原顺序（删除后）：[1, 3, 5]
2. Step 1（取反）：[-1, -3, -5]
3. Step 2 CTE 计算（`ORDER BY id DESC`）：
   - -1 → new_id = 1
   - -3 → new_id = 2
   - -5 → new_id = 3
4. 最终结果：[1, 2, 3]（连续无空洞）

**关键技巧**：
- 取反后 `ORDER BY id DESC` 等价于原顺序的 `ORDER BY id ASC`
- CTE 先完全计算再执行 UPDATE，避免自引用问题
- 使用 SQLite 的 `rowid` 作为关联键，不依赖业务字段

#### 9.4.1 renumber 的副作用：所有位置ID被重新分配

**重要提示**：renumber 虽然解决了编号不连续的问题，但带来了严重的副作用——**所有歌曲的位置ID都会被重新分配**！

**完整的 renumber 触发场景**：

| 场景 | 触发点 | 位置ID变化 |
|------|--------|------------|
| 用户主动删除歌曲 | `playlist_track_repository.go:197` | ✅ 全部重新分配 |
| 清空播放列表 | `playlist_track_repository.go:206` | ✅ 全部重新分配 |
| 孤儿清理（GC） | `playlist_repository.go:379` | ✅ 全部重新分配 |
| 先删后加 Update | `Delete` 内部调用 | ✅ 全部重新分配 |
| **仅添加歌曲** | `addTracks` | ❌ 不改变已有ID |
| **仅重排序** | `Reorder` | ❌ 仅调整相关位置 |
| **仅更新元数据** | `Put(pls, cols...)` | ❌ 不影响 |

**对客户端的影响**：
- 客户端缓存的位置ID在 renumber 后全部失效
- 基于旧位置ID的删除请求可能删除错误的歌曲
- 客户端必须在每次操作前重新获取播放列表

**设计权衡**：
- 简单的设计选择了简单的 renumber 算法
- 牺牲了并发安全性，换取实现简单
- 假设播放列表很少并发修改的概率较低（单用户场景）

---

## 10. 统计信息刷新机制

`playlist_repository.go:249-279` 的 `refreshCounters` 方法：

```go
func (r *playlistRepository) refreshCounters(pls *model.Playlist) error {
    statsSql := Select(
        "coalesce(sum(duration), 0) as duration",
        "coalesce(sum(size), 0) as size",
        "count(*) as count",
    ).
        From("media_file").
        Join("playlist_tracks f on f.media_file_id = media_file.id").
        Where(Eq{"playlist_id": pls.ID})
    var res struct{ Duration, Size, Count float32 }
    err := r.queryOne(statsSql, &res)
    if err != nil {
        return err
    }

    // Update playlist's total duration, size and count
    upd := Update("playlist").
        Set("duration", res.Duration).
        Set("size", res.Size).
        Set("song_count", res.Count).
        Set("updated_at", time.Now()).
        Where(Eq{"id": pls.ID})
    _, err = r.executeSQL(upd)
    if err != nil {
        return err
    }
    pls.SongCount = int(res.Count)
    pls.Duration = res.Duration
    pls.Size = int64(res.Size)
    return nil
}
```

**触发时机**：
- `addTracks` 完成后
- `renumber` 完成后
- `Put` 但没有 tracks 更新时（仅刷新已有数据）

---

## 11. 关键设计决策总结

### 11.1 用 id 存储顺序的优缺点

**优点**：
- 简单直接，查询时 `ORDER BY playlist_tracks.id` 即可获得正确顺序
- 无需额外的 `position` 字段，节省存储空间
- 天然支持唯一约束，避免同一位置重复

**缺点**：
- 删除中间元素会产生空洞，需要 `renumber` 修复
- 重排序需要复杂的多步操作避免唯一约束冲突
- 批量删除后重编号成本 O(n)

### 11.2 性能优化策略

| 优化点 | 实现方式 |
|--------|----------|
| 批量插入分块 | 每200条一批，避免SQLITE_MAX_VARIABLE_NUMBER |
| 孤儿清理延迟 | 扫描后统一GC，不在删除媒体文件时立即执行 |
| 智能播放列表延迟更新 | Put 时不更新 tracks，由后台进程异步刷新 |
| 统计信息聚合 | 使用 SQL SUM/COUNT 一次性计算，避免内存遍历 |

### 11.3 权限模型

- 播放列表有 `OwnerID` 和 `Public` 字段
- 非管理员用户只能修改自己的播放列表
- 智能播放列表的 tracks 不可直接修改（`IsSmartPlaylist()` 检查）
- `checkWritable` 和 `checkTracksEditable` 统一权限校验

### 11.4 并发安全

- 写操作使用 `WithTxImmediate` 避免 SQLite 死锁
- 所有修改操作在事务中执行，保证原子性
- 重排序算法通过临时负值避免唯一约束冲突

---

## 12. 核心代码路径汇总

| 功能 | 文件位置 | 行号 |
|------|----------|------|
| 创建播放列表 | `core/playlists/playlists.go` | 105-134 |
| 持久化入口 Put | `persistence/playlist_repository.go` | 100-129 |
| 批量添加歌曲 addTracks | `persistence/playlist_repository.go` | 229-246 |
| 全量替换 updatePlaylist | `persistence/playlist_repository.go` | 218-227 |
| 歌曲重排序 Reorder | `persistence/playlist_track_repository.go` | 210-248 |
| 删除歌曲 Delete | `persistence/playlist_track_repository.go` | 191-198 |
| 孤儿清理 removeOrphans | `persistence/playlist_repository.go` | 353-384 |
| 重编号算法 renumber | `persistence/playlist_repository.go` | 389-411 |
| 统计刷新 refreshCounters | `persistence/playlist_repository.go` | 249-279 |
| GC 入口 | `persistence/persistence.go` | 170-202 |
| 扫描器 GC 触发 | `scanner/scanner.go` | 230-256 |
| 事务 WithTx | `persistence/persistence.go` | 131-154 |
| 立即事务 WithTxImmediate | `persistence/persistence.go` | 156-168 |
| 数据模型 Playlist | `model/playlist.go` | 12-33 |
| 数据模型 PlaylistTrack | `model/playlist.go` | 136-141 |
| **Update 先删后加逻辑** | `core/playlists/playlists.go` | 152-199 |
| **Native 批量添加入口** | `server/nativeapi/playlists.go` | 121-167 |
| **四类来源 addMediaFileIds** | `persistence/playlist_track_repository.go` | 161-189 |
| **Subsonic UpdatePlaylist** | `server/subsonic/playlists.go` | 97-128 |
| **Native deleteFromPlaylist** | `server/nativeapi/playlists.go` | 101-119 |
| **RemoveTracks Service** | `core/playlists/playlists.go` | 274-281 |
| **AddTracks/AddAlbums Service** | `core/playlists/playlists.go` | 246-272 |

---

## 13. 数据库表结构（简化）

```sql
CREATE TABLE playlist (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    comment TEXT,
    duration REAL DEFAULT 0,
    size INTEGER DEFAULT 0,
    song_count INTEGER DEFAULT 0,
    owner_id TEXT NOT NULL,
    public BOOLEAN DEFAULT 0,
    path TEXT,
    sync BOOLEAN DEFAULT 0,
    uploaded_image TEXT,
    external_image_url TEXT,
    created_at DATETIME,
    updated_at DATETIME,
    rules TEXT,
    evaluated_at DATETIME
);

CREATE TABLE playlist_tracks (
    id INTEGER NOT NULL,
    playlist_id TEXT NOT NULL,
    media_file_id TEXT NOT NULL,
    PRIMARY KEY (playlist_id, id),
    FOREIGN KEY (playlist_id) REFERENCES playlist(id) ON DELETE CASCADE,
    FOREIGN KEY (media_file_id) REFERENCES media_file(id)
);

CREATE INDEX idx_playlist_tracks_media_file ON playlist_tracks(media_file_id);
```

**注意**：`playlist_tracks.id` 是位置序号，不是自增主键。真正的主键是 `(playlist_id, id)` 联合主键。
