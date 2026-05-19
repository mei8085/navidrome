# Navidrome 媒体库扫描器主流程分析

## 一、整体架构

扫描器采用**四阶段流水线（Pipeline）架构**，基于 `go-pipeline` 库实现。整体流程在 `scanner/scanner.go:57-186` 的 `scanFolders` 函数中定义：

```
Phase 1: 文件夹扫描 → Phase 2: 缺失文件处理 → [Parallel: Phase 3: 专辑刷新 | Phase 4: 播放列表导入] → Final Steps (GC + 统计刷新 + 库更新 + DB优化)
```

## 二、增量扫描 vs 全量扫描

### 2.1 触发分支

扫描类型由 `fullScan` 布尔参数控制，在 `scanner.go:57` 传入。扫描开始时会进行一次自动升级检查：

```go
// scanner.go:115-128
if !state.fullScan {
    for _, lib := range state.libraries {
        if lib.FullScanInProgress {
            log.Info(ctx, "Scanner: Interrupted full scan detected", "lib", lib.Name)
            state.fullScan = true
            break
        }
    }
}
```

**触发场景**：
- 全量扫描：手动触发、数据库迁移后、PID 配置变更、中断的全量扫描恢复
- 增量扫描：文件系统监视器触发、定时扫描、常规启动扫描

### 2.2 增量扫描的核心判断逻辑

在 `folder_entry.go:65-70` 的 `isOutdated()` 方法中实现：

```go
func (f *folderEntry) isOutdated() bool {
    // 全量扫描且文件夹在扫描开始后未更新过 → 强制处理
    if f.job.lib.FullScanInProgress && f.updTime.Before(f.job.lib.LastScanStartedAt) {
        return true
    }
    // 增量扫描：比较文件夹哈希（modTime + 文件列表 + 图片更新时间）
    return f.prevHash != f.hash()
}
```

**文件夹哈希计算**（`folder_entry.go:84-118`）包含：
- 目录修改时间
- 播放列表数量
- 子目录数量
- 图片更新时间
- 所有音频文件的名称、大小、修改时间（排序后）
- 所有图片文件的名称、大小、修改时间（排序后）

### 2.3 增量扫描的优化点

1. **文件夹级跳过**：哈希未变化的文件夹直接跳过所有处理 (`phase_1_folders.go:184-186`)
2. **文件级判断**：即使文件夹需要处理，也只重新扫描修改时间晚于数据库记录的文件 (`phase_1_folders.go:243`)
3. **空新文件夹跳过**：增量扫描时，新建的空文件夹不处理 (`phase_1_folders.go:176-177`)

### 2.4 全量扫描的特殊行为

- `scanner.go:67-69`: 全量扫描时 `changesDetected` 初始设为 `true`，确保 GC 等后续步骤执行
- `phase_1_folders.go:234`: 全量扫描时强制导入所有文件，无论修改时间
- `phase_2_missing_tracks.go:348`: 全量扫描时可能触发缺失文件的清理（取决于 `PurgeMissing` 配置）

## 三、封面与歌词副流程

### 3.1 封面预热（Cache Warmer）

封面预热是**异步、非阻塞**的副流程，不影响主扫描流程的完成。

**触发时机**（在数据库事务提交成功后）：
- `phase_1_folders.go:373-376`: 艺术家入库后，预热艺术家封面
- `phase_1_folders.go:385-387`: 专辑入库后，预热专辑封面
- `phase_4_playlists.go:112`: 播放列表导入后，预热播放列表封面

**实现机制**（`core/artwork/cache_warmer.go`）：
1. `PreCache(artID)` 将封面 ID 放入内存缓冲区
2. 后台 goroutine 每 10 秒或缓冲区有数据时唤醒
3. 批量处理（4 并发）调用 `artwork.Get()` 生成并缓存缩略图
4. 支持配置 `EnableArtworkPrecache` 完全禁用

### 3.2 歌词处理

歌词**不在扫描流程中处理**，采用**按需加载**策略：

- 入口：`core/lyrics/lyrics.go:34-59` 的 `GetLyrics()` 方法
- 触发时机：用户在 UI 中打开歌曲详情时首次调用
- 优先级顺序由 `LyricsPriority` 配置控制：
  1. `embedded`: 从音频文件内嵌标签提取
  2. `.lrc` 等后缀：从同目录外部歌词文件读取
  3. 插件提供者：从第三方插件获取

**设计考量**：歌词数据量较大且访问频率低，扫描时批量提取会显著降低扫描速度。

## 四、错误文件归档策略

Navidrome 没有传统意义上的"归档"概念，而是采用**软删除（标记为 Missing）+ 可配置清理**的策略。

### 4.1 错误处理层级

| 错误类型 | 处理方式 | 代码位置 |
|---------|---------|---------|
| 目录读取失败 | 跳过该目录，记录警告 | `walk_dir_tree.go:71-72` |
| 单个文件 stat 失败 | 跳过该文件，记录警告 | `walk_dir_tree.go:144-146` |
| 文件信息读取失败 | 跳过该文件，发送进度警告 | `phase_1_folders.go:237-242` |
| 标签元数据提取失败 | 跳过整个文件夹的标签加载，记录警告 | `phase_1_folders.go:254-260` |
| 数据库写入失败 | 回滚事务，文件夹处理失败 | `phase_1_folders.go:420-422` |

### 4.2 Missing 标记机制

**标记时机**：
1. `phase_1_folders.go:251`: 文件系统中不存在但数据库中存在的轨道 → 标记为 missing
2. `phase_1_folders.go:400-406`: 批量标记缺失轨道
3. `phase_1_folders.go:485-494`: 扫描结束时，所有仍在 `lastUpdates` 中的文件夹（即磁盘上已删除）→ 标记为 missing，其下所有轨道也标记为 missing

### 4.3 缺失文件清理（Purge）策略

由 `Scanner.PurgeMissing` 配置控制，在 `phase_2_missing_tracks.go:348-352` 判断：

```go
if conf.Server.Scanner.PurgeMissing == consts.PurgeMissingAlways || 
   (conf.Server.Scanner.PurgeMissing == consts.PurgeMissingFull && p.state.fullScan) {
    if err = p.purgeMissing(); err != nil { ... }
}
```

**三种策略**（`consts/consts.go:139-141`）：
- `never`（**默认值**，`conf/configuration.go:817`）: 只标记不删除，用户可手动恢复
- `always`: 每次扫描（增量或全量）都删除标记为 missing 的文件
- `full`: 仅在全量扫描时删除 missing 文件

**删除操作**：`phase_2_missing_tracks.go:357-372` 的 `purgeMissing()` 调用 `MediaFile.DeleteAllMissing()` 物理删除数据库记录。

### 4.4 移动文件检测

Phase 2 专门处理文件移动场景，避免删除后重新导入导致播放计数等元数据丢失：

1. 按 PID（Persistent ID）分组缺失文件和新导入文件
2. 三级匹配策略：
   - 精确匹配：`Equals()` 比较所有元数据
   - 单 PID 匹配：同一 PID 下只有一个缺失和一个新增 → 判定为移动
   - 等效匹配：`IsEquivalent()` 比较基础路径等属性
3. 跨库移动检测：`phase_2_missing_tracks.go:195-229`，通过 MBZ Track ID 或文件属性在所有库中搜索匹配

## 五、核心数据流

### 5.1 Phase 1: 文件夹扫描（`phase_1_folders.go`）

```
walkDirTree() → 生成 folderEntry → isOutdated() 判断 → processFolder()
                                                          ↓
                                         加载 DB 现有轨道 → 对比 FS → 确定需导入文件
                                                          ↓
                                         loadTagsFromFiles() 批量读取元数据（200一批）
                                                          ↓
                                         createAlbumsFromMediaFiles() 分组生成专辑
                                                          ↓
                                         createArtistsFromMediaFiles() 生成艺术家
                                                          ↓
                                         persistChanges() 事务写入 DB
                                                          ↓
                                         PreCache() 触发封面预热
```

### 5.2 Phase 2: 缺失文件处理（`phase_2_missing_tracks.go`）

```
GetMissingAndMatching() → 按 PID 分组 → processMissingTracks() 库内匹配
                                                      ↓
                                    processCrossLibraryMoves() 跨库匹配
                                                      ↓
                                    moveMatched() 保留原 ID 更新路径
                                                      ↓
                                    purgeMissing() 可选清理
```

### 5.3 Phase 3: 专辑刷新（`phase_3_refresh_albums.go`）

```
GetTouchedAlbums() → filterUnmodified() 重建对比 → refreshAlbum() 更新
                                                          ↓
                                         RefreshPlayCounts() 刷新播放统计
```

### 5.4 Phase 4: 播放列表导入（`phase_4_playlists.go`）

```
GetTouchedWithPlaylists() → processPlaylistsInFolder() → ImportFromFolder()
                                                                 ↓
                                                          PreCache() 封面预热
```

## 六、关键并发与事务设计

1. **全局互斥**：`controller.go:259` 的 `running` 原子布尔值确保同时只有一个扫描在执行
2. **阶段内并发**：
   - Phase 1: `conf.Server.DevScannerThreads` 并发处理文件夹
   - Phase 3: 5 并发过滤未修改专辑
   - Phase 4: 3 并发处理播放列表
3. **事务边界**：每个文件夹的持久化在独立事务中 (`phase_1_folders.go:336`)，失败不影响其他文件夹
4. **封面预热**：独立 goroutine，不阻塞主流程

## 七、选择性扫描（Selective Scan）

支持仅扫描特定文件夹，通过 `targets` 参数传递：
- `scanner.go:79-94`: 构建 `targets map[libraryID][]folderPaths`
- `walk_dir_tree.go:21-27`: 仅遍历目标文件夹及其子目录
- `scanner.go:238-242`: GC 时仅作用于涉及的库
- 应用场景：文件系统监视器检测到局部变更时触发

## 八、SQLite 落库流程详解

### 8.1 完整调用链（从标签读取到索引写入）

整个 Phase 1 流程分为 **内存处理阶段**（事务外）和 **数据库写入阶段**（事务内），由三个 Pipeline Stage 串行执行：

```
Stage 1: processFolder (内存处理) → Stage 2: persistChanges (DB写入) → Stage 3: logFolder
```

#### 阶段 A：内存处理（事务外，`phase_1_folders.go:208-267`）

```
┌─────────────────────────────────────────────────────────────────────────┐
│  processFolder(entry)                                                    │
│                                                                          │
│  1. 从 DB 加载现有轨道 → dbTracks[path]*MediaFile                        │
│     GetCursor(Filters: folder_id=?)                                      │
│                                                                          │
│  2. 对比 FS 与 DB，确定 filesToImport 和 missingTracks                   │
│     ├─ 全量扫描：所有文件 → filesToImport                                 │
│     └─ 增量扫描：modTime > dbTrack.UpdatedAt → filesToImport             │
│        剩余 dbTracks → missingTracks                                     │
│                                                                          │
│  3. loadTagsFromFiles() 批量读取元数据（200/批）                          │
│     ├─ fs.ReadTags(chunk...) → 从文件系统读取标签                        │
│     ├─ metadata.New() → md.ToMediaFile() → 转换为模型                    │
│     ├─ 收集 uniqueTags                                                  │
│     └─ 记录 albumIDMap（旧专辑ID → 新专辑ID）用于 annotation 迁移       │
│                                                                          │
│  4. createAlbumsFromMediaFiles()                                         │
│     └─ 按 AlbumID 分组 → songs.ToAlbum() → entry.albums                 │
│                                                                          │
│  5. createArtistsFromMediaFiles()                                        │
│     └─ 合并所有 Participants → entry.artists                            │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 阶段 B：数据库写入（事务内，`phase_1_folders.go:329-432`）

```
┌─────────────────────────────────────────────────────────────────────────┐
│  persistChanges(entry)                                                   │
│                                                                          │
│  ┌─ WithTx(...) 开启事务 ──────────────────────────────────────────────┐ │
│  │                                                                      │ │
│  │  1. folderRepo.Put(folder)                                           │ │
│  │     → 更新 folder.hash, num_audio_files, image_files 等              │ │
│  │                                                                      │ │
│  │  2. tagRepo.Add(libID, entry.tags...)                                │ │
│  │     ├─ INSERT INTO tag (ON CONFLICT DO NOTHING)                     │ │
│  │     └─ INSERT INTO library_tag (ON CONFLICT DO NOTHING)             │ │
│  │                                                                      │ │
│  │  3. 遍历 entry.artists:                                              │ │
│  │     ├─ artistRepo.Put(artist, colsToUpdate...)                      │ │
│  │     │  (只更新 name, mbz_artist_id, sort_artist_name 等列)           │ │
│  │     ├─ libraryRepo.AddArtist(libID, artistID)                       │ │
│  │     │  (INSERT INTO library_artist, ON CONFLICT DO NOTHING)         │ │
│  │     └─ 收集 artworkIDs（非 Unknown/Various）                         │ │
│  │                                                                      │ │
│  │  4. 遍历 entry.albums:                                               │ │
│  │     ├─ persistAlbum(album, albumIDMap)                              │ │
│  │     │  ├─ albumRepo.Put(album)                                      │ │
│  │     │  ├─ albumRepo.ReassignAnnotation(prevID, newID)               │ │
│  │     │  └─ albumRepo.CopyAttributes(prevID, newID, "created_at")     │ │
│  │     └─ 收集 artworkIDs                                              │ │
│  │                                                                      │ │
│  │  5. 遍历 entry.tracks:                                               │ │
│  │     └─ mfRepo.Put(track)                                            │ │
│  │        (putByMatch: path + library_id)                              │ │
│  │        → updateParticipants(media_file_artists)                     │ │
│  │                                                                      │ │
│  │  6. if len(missingTracks) > 0:                                      │ │
│  │     ├─ mfRepo.MarkMissing(true, missingTracks...)                   │ │
│  │     └─ albumRepo.Touch(albumIDs) （标记需要刷新的专辑）              │ │
│  │                                                                      │ │
│  └─ 事务提交 ───────────────────────────────────────────────────────────┘ │
│                                                                          │
│  事务成功后：PreCache(artworkIDs) → 异步封面预热                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.2 事务边界与失败影响范围

#### 事务分层设计

| 层级 | 事务范围 | 失败影响 | 代码位置 |
|-----|---------|---------|---------|
| 1. 扫描准备 | `prepareLibrariesForScan()` 每个库独立事务 | 该库跳过扫描，其他库继续 | `scanner.go:131` |
| 2. Phase 1 文件夹 | 每个文件夹独立事务 | 仅该文件夹回滚，其他文件夹不受影响 | `phase_1_folders.go:336` |
| 3. GC 清理 | 所有库在同一事务 | 全部回滚 | `scanner.go:233` |
| 4. 库状态更新 | 每个库独立事务 | 该库 `last_scan_at` 未更新 | `scanner.go:295` |

#### 单文件夹事务内的失败原子性

在 `persistChanges()` 的事务中，**任一步骤失败都会导致整个文件夹回滚**：
- 文件夹写入失败 → 整个文件夹回滚
- 标签写入失败 → 整个文件夹回滚（包括已写入的 folder）
- 艺术家写入失败 → 整个文件夹回滚（包括已写入的 folder、tags）
- 依此类推...

**例外情况**：
- `loadTagsFromFiles()` 在事务外执行，文件读取失败只会跳过该文件夹，不影响其他
- 封面预热在事务提交后执行，失败不影响数据库状态

### 8.3 增量扫描触发更新的条件层级

增量扫描采用 **三级过滤机制**，逐层缩小需要处理的范围：

```
Level 1: 文件夹级过滤 (isOutdated)
    ↓ 只有 outdated 的文件夹才进入下一阶段
Level 2: 文件级过滤 (modTime 比较)
    ↓ 只有修改时间更新的文件才读取标签
Level 3: 字段级更新 (Upsert)
    只有实际变化的字段才被写入
```

#### Level 1: 文件夹级判断 (`folder_entry.go:65-70`)

```go
func (f *folderEntry) isOutdated() bool {
    // 全量扫描 + 文件夹在扫描开始后未更新 → 强制处理
    if f.job.lib.FullScanInProgress && f.updTime.Before(f.job.lib.LastScanStartedAt) {
        return true
    }
    // 增量扫描：比较文件夹哈希
    return f.prevHash != f.hash()
}
```

**哈希因子**（`folder_entry.go:84-118`）：
- 目录修改时间 (modTime)
- 播放列表数量、子目录数量、图片更新时间
- 所有音频文件的名称、大小、修改时间（排序后）
- 所有图片文件的名称、大小、修改时间（排序后）

#### Level 2: 文件级判断 (`phase_1_folders.go:230-248`)

```go
for afPath, af := range entry.audioFiles {
    dbTrack, foundInDB := dbTracks[fullPath]
    if !foundInDB || p.state.fullScan {
        filesToImport[fullPath] = dbTrack  // 新文件或全量扫描 → 导入
    } else {
        info, _ := af.Info()
        // 增量扫描：文件修改时间晚于 DB 记录，或已标记为 missing → 导入
        if info.ModTime().After(dbTrack.UpdatedAt) || dbTrack.Missing {
            filesToImport[fullPath] = dbTrack
        }
    }
    delete(dbTracks, fullPath)  // 从 DB 列表移除，剩余的即为 missing
}
```

#### Level 3: 字段级更新（Upsert 逻辑）

所有实体通过 `sql_base_repository.go:397-452` 的 `put()` 实现 **"先查后更/插"**：

```go
func (r sqlRepository) put(id string, m any, colsToUpdate ...string) (string, error) {
    if id != "" {
        // 有 ID → 先 UPDATE
        update := Update(r.tableName).Where(Eq{"id": id}).SetMap(updateValues)
        count, err := r.executeSQL(update)
        if count > 0 {
            return id, nil  // 更新成功
        }
    }
    // 无 ID 或 UPDATE 影响 0 行 → INSERT
    if id == "" {
        id = id2.NewRandom()
    }
    insert := Insert(r.tableName).SetMap(values)
    _, err = r.executeSQL(insert)
    return id, err
}
```

**增量更新的关键保护**：
- **created_at 保护**：`put()` 中删除 `created_at` 字段，避免覆盖
- **birth_time 保护**：媒体文件的 `birth_time` 也被排除
- **部分列更新**：`colsToUpdate` 参数只更新指定字段（艺术家只更新 6 个字段）

### 8.4 关联表写入机制

#### Participants（艺术家关联）

`sql_participations.go:53-100` 的 `updateParticipants()` 采用 **"先删后插"** 策略：

```go
func (r sqlRepository) updateParticipants(itemID string, participants model.Participants) error {
    // 1. DELETE 所有旧关联（确保角色变更时清理）
    sqd := Delete(r.tableName + "_artists").Where(Eq{r.tableName + "_id": itemID})
    // 2. INSERT 新关联（通过 JOIN artist 过滤不存在的艺术家 ID）
    query := `INSERT INTO media_file_artists (media_file_id, artist_id, role, sub_role)
              SELECT ?, json_extract(value, '$.artist_id'),
                     json_extract(value, '$.role'),
                     COALESCE(json_extract(value, '$.sub_role'), '')
              FROM json_each(?)
              JOIN artist ON artist.id = json_extract(value, '$.artist_id')
              ON CONFLICT (artist_id, media_file_id, role, sub_role) DO NOTHING`
}
```

#### Tags（标签关联）

`tag_repository.go:25-49` 的 `Add()` 采用幂等写入：
- 批量插入 `tag` 表（`ON CONFLICT DO NOTHING`）
- 批量插入 `library_tag` 关联表
- 实际引用存储在 `media_file.tags` JSON 字段中

### 8.5 全量 / 增量 / 选择性扫描对比

| 维度 | 增量扫描 (Incremental) | 全量扫描 (Full) | 选择性扫描 (Selective) |
|-----|------------------------|-----------------|------------------------|
| **触发条件** | 定时、启动、文件监视器 | 手动、迁移后、PID变更 | 文件监视器局部变更 |
| **扫描范围** | 所有库的所有文件夹 | 所有库的所有文件夹 | 仅指定 `targets` 文件夹 |
| **Level 1 判断** | 比较 folder.hash | 强制所有文件夹处理 (FullScanInProgress=true) | 比较 folder.hash（仅目标） |
| **Level 2 判断** | modTime > dbTrack.UpdatedAt | 所有文件强制导入 | modTime > dbTrack.UpdatedAt（仅目标） |
| **lastUpdates 加载** | 所有文件夹 | 所有文件夹 | 仅目标文件夹 + 子目录 (`GetFolderUpdateInfoBatch`) |
| **changesDetected** | 有文件夹处理才设为 true | 初始即为 true | 有文件夹处理才设为 true |
| **GC 范围** | 所有库 | 所有库 | 仅目标库 (`libraryIDs` 参数) |
| **PurgeMissing** | 配置为 always 时执行 | 配置为 always/full 时执行 | 配置为 always 时执行 |
| **统计刷新** | 有变化才刷新 | 必定刷新 | 有变化才刷新 |
| **DB optimize** | 不执行 | 执行 (`PRAGMA optimize`) | 不执行 |

**关键差异代码**：
- 选择性扫描：`scanner.go:79-94` 构建 `targets map`，`folder_repository.go:138-170` 批量查询目标文件夹
- 全量扫描：`scanner.go:67-69` 初始化 `changesDetected=true`，`phase_1_folders.go:234` 强制导入所有文件

### 8.6 GC 清理流程

扫描完成后在 `persistence.go:170-202` 的 `GC()` 中执行，**顺序执行**以下清理：

```
1. album.purgeEmpty(libraryIDs)
   → DELETE FROM album WHERE id NOT IN (SELECT album_id FROM media_file)
   
2. artist.purgeEmpty()
   → DELETE FROM artist WHERE id NOT IN (SELECT artist_id FROM album_artists)
   → 同时清理上传的封面图片文件
   
3. artist.markMissing()
   → UPDATE artist SET missing = true WHERE 所有专辑都 missing
   
4. folder.purgeEmpty(libraryIDs)
   → DELETE FROM folder WHERE num_audio_files=0 AND 无引用
   
5. clean annotations (album/artist/mediafile)
   → 删除无对应实体的 annotation 记录
   
6. tag.purgeUnused()
   → DELETE FROM tag WHERE id 不被任何 album/media_file 引用
   
7. playlist.removeOrphans()
   → 删除无对应 media_file 的 playlist_tracks 记录
```

