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
- `never`（默认？需确认）: 只标记不删除，用户可手动恢复
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
