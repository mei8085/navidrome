# Navidrome 播放列表实现机制分析

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

| 操作 | 入口方法 | 事务边界 | 核心逻辑 |
|------|----------|----------|----------|
| 创建/替换 | `Put(pls *Playlist)` | `WithTxImmediate` | 先存元数据，再调用 `updateTracks` |
| 添加歌曲 | `Tracks().Add(ids)` | 无（单条SQL） | 查询 `max(id)`，调用 `addTracks` 追加 |
| 删除歌曲 | `Tracks().Delete(ids)` | `WithTx` | 删除后调用 `renumber()` 重编号 |
| 重排序 | `Tracks().Reorder(pos, newPos)` | `WithTx` | 四步算法原地调整 |
| 更新元数据 | `Put(pls, cols...)` | 无 | 仅更新指定字段（名称、注释、公开状态） |
| 统计刷新 | `refreshCounters(pls)` | 无 | 聚合查询更新 duration/size/song_count |
| 孤儿清理 | `removeOrphans()` | `WithTx` (GC内) | 删除无效引用 + renumber |

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

### 5.2 各操作的事务边界

| 操作 | 事务模式 | 原因 |
|------|----------|------|
| Create | `WithTxImmediate` | 可能同时写入 playlist 和 playlist_tracks，避免并发创建死锁 |
| Update (track changes) | `WithTxImmediate` | 涉及多个表修改 |
| Update (metadata only) | 无事务 | 单条 UPDATE，原子性由SQLite保证 |
| Delete (playlist) | 无事务 | 单条 DELETE + 文件系统操作（非事务） |
| RemoveTracks | `WithTx` | 删除 + renumber 需要原子性 |
| ReorderTrack | `WithTx` | 四条 UPDATE 必须原子执行 |
| AddTracks | 无事务 | 单条 INSERT（分块时多条，但无状态依赖） |
| GC/removeOrphans | `WithTx` (外层) | 整个GC过程在一个大事务中 |

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

## 6. 删除歌曲后的列表修复机制

### 6.1 两种删除场景

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

### 6.2 孤儿清理触发时机

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

### 6.3 孤儿清理实现

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

### 6.4 重编号（renumber）算法

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

---

## 7. 统计信息刷新机制

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

## 8. 关键设计决策总结

### 8.1 用 id 存储顺序的优缺点

**优点**：
- 简单直接，查询时 `ORDER BY playlist_tracks.id` 即可获得正确顺序
- 无需额外的 `position` 字段，节省存储空间
- 天然支持唯一约束，避免同一位置重复

**缺点**：
- 删除中间元素会产生空洞，需要 `renumber` 修复
- 重排序需要复杂的多步操作避免唯一约束冲突
- 批量删除后重编号成本 O(n)

### 8.2 性能优化策略

| 优化点 | 实现方式 |
|--------|----------|
| 批量插入分块 | 每200条一批，避免SQLITE_MAX_VARIABLE_NUMBER |
| 孤儿清理延迟 | 扫描后统一GC，不在删除媒体文件时立即执行 |
| 智能播放列表延迟更新 | Put 时不更新 tracks，由后台进程异步刷新 |
| 统计信息聚合 | 使用 SQL SUM/COUNT 一次性计算，避免内存遍历 |

### 8.3 权限模型

- 播放列表有 `OwnerID` 和 `Public` 字段
- 非管理员用户只能修改自己的播放列表
- 智能播放列表的 tracks 不可直接修改（`IsSmartPlaylist()` 检查）
- `checkWritable` 和 `checkTracksEditable` 统一权限校验

### 8.4 并发安全

- 写操作使用 `WithTxImmediate` 避免 SQLite 死锁
- 所有修改操作在事务中执行，保证原子性
- 重排序算法通过临时负值避免唯一约束冲突

---

## 9. 核心代码路径汇总

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

---

## 10. 数据库表结构（简化）

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
