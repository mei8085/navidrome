# Navidrome 曲库搜索架构分析报告

## 1. 整体架构概览

Navidrome 的搜索系统采用**三路径、共享引擎、分层叠加**的架构设计。三条独立的查询路径共享底层搜索策略选择器，但在调用入口、过滤叠加层级和执行流程上有本质区别。

核心搜索策略选择和执行逻辑位于 [sql_search.go](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/sql_search.go)。

```
                    ┌─────────────────────────────────┐
                    │   getSearchStrategy()          │
                    │   (FTS5 / LIKE / Legacy)       │
                    └───────────────┬─────────────────┘
                                    │
           ┌────────────────────────┼────────────────────────┐
           │                        │                        │
  ┌────────▼──────┐       ┌─────────▼────────┐       ┌───────▼──────────┐
  │  Search API   │       │  REST List Filter│       │ Smart Playlist   │
  │ (search2/3)   │       │  (Web UI 列表)   │       │   Criteria       │
  └────────┬──────┘       └─────────┬────────┘       └───────┬──────────┘
           │                        │                        │
  ┌────────▼──────┐       ┌─────────▼────────┐       ┌───────▼──────────┐
  │  doSearch()   │       │ parseRestFilters()│       │ smartPlaylist    │
  │ (两阶段执行)  │       │ fullTextFilter() │       │ Criteria         │
  └────────┬──────┘       └─────────┬────────┘       └───────┬──────────┘
           │                        │                        │
  ┌────────▼──────┐       ┌─────────▼────────┐       ┌───────▼──────────┐
  │strategy.execute│       │    GetAll()      │       │ INSERT INTO ...  │
  │(Phase1+Phase2) │       │(单阶段查询)      │       │ SELECT ...       │
  └───────────────┘       └──────────────────┘       └──────────────────┘
```

---

## 2. 索引结构设计

### 2.1 FTS5 虚拟表索引

Navidrome 使用 SQLite FTS5 作为主要全文搜索引擎，索引定义在迁移文件 [20260220173400_add_fts5_search.go](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/db/migrations/20260220173400_add_fts5_search.go) 中。

**三张核心 FTS5 虚拟表：**

| 虚拟表名 | 对应主表 | 索引列定义 (按顺序) |
|---------|---------|-------------------|
| `media_file_fts` | `media_file` | title(10), album(5), artist(3), album_artist(3), sort_title(1), sort_album_name(1), sort_artist_name(1), sort_album_artist_name(1), disc_subtitle(1), search_participants(2), search_normalized(1) |
| `album_fts` | `album` | name(10), sort_album_name(1), album_artist(3), search_participants(2), discs(1), catalog_num(1), album_version(1), search_normalized(1) |
| `artist_fts` | `artist` | name(10), sort_artist_name(1), search_normalized(1) |

**分词器配置：**
```sql
tokenize='unicode61 remove_diacritics 2'
```
- `unicode61`：基于 Unicode 6.1 标准的分词器，按空格和标点分词
- `remove_diacritics 2`：移除 NFKD 可分解的变音符号（如 `é` → `e`）

> ⚠️ **注意**：`unicode61` 对 CJK 文本效果极差——整段中文会被当作单个 token，因此 Navidrome 对 CJK 查询自动降级到 LIKE 策略。

**BM25 权重配置** ([sql_search_fts.go#L220-L249](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/sql_search_fts.go#L220-L249))：

列的权重通过 `ftsColumnDefs` 预计算，标题/名称列权重最高（10.0），排序辅助列权重最低（1.0）：

```go
var ftsColumnDefs = map[string][]ftsColumn{
    "media_file": {
        {"title", 10.0},          // 歌曲标题权重最高
        {"album", 5.0},
        {"artist", 3.0},
        {"album_artist", 3.0},
        {"search_participants", 2.0},  // 参与者（JSON 提取的名字）
        {"search_normalized", 1.0},    // 规范化后的文本
        // ...
    },
}
```

### 2.2 辅助索引列

主表中额外维护了两个搜索专用列：

- **`search_participants`**：从 `participants` JSON 字段中提取的所有参与者姓名，空格分隔。避免搜索时走 `json_tree` 子查询。
- **`search_normalized`**：存储 `normalizeForFTS()` 输出——标点剥离后的 ASCII 连接形式（`R.E.M.` → `REM`，`a-ha` → `aha`）以及重音移除变体。使 FTS5 能通过规范化 token 匹配含特殊字符的名字。

### 2.3 触发器同步机制

FTS5 采用**外部内容表**模式 (`content=''`)，通过 AFTER INSERT/UPDATE/DELETE 触发器保持与主表同步（[迁移文件 L191-L364](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/db/migrations/20260220173400_add_fts5_search.go#L191-L364)）。UPDATE 触发器有 `WHEN` 子句，仅当搜索相关列发生变化时才重索引。

---

## 3. 三条查询路径详解

### 3.1 路径一：Subsonic 搜索接口（Search API）

**入口**：Subsonic API 的 `search2` / `search3` 端点，参数 `query`。

**调用链**：

```
[server/subsonic/searching.go]
    │
    ▼ search2() / search3()
    ├─► albumRepo.Search(query, options)
    ├─► artistRepo.Search(query, options)
    └─► mediaFileRepo.Search(query, options)
                                   │
                                   ▼
    [persistence/mediafile_repository.go#L441-L452]
                                   │
    func Search(q, options) {      │
        sq := selectMediaFile(options...)
        return doSearch(sq, q, results, mediaFileSearchConfig, options)
    }
                                   │
                                   ▼
    [persistence/sql_search.go#L52-L83]
    doSearch(sq, q, results, cfg, options)
        │
        ├─► q == "" → 自然顺序返回
        ├─► uuid.Validate(q) → MBID 精确匹配
        ├─► len(q) < 2 → return nil  ═══════╗
        │                                   ║  ⟵ 单字符限制只在此处
        ▼                                   ║
    getSearchStrategy(tableName, q)          ║
        │                                   ║
        ├─► newFTSSearch() ────► fallback? ─╣───► newLikeSearch()
        ├─► newLikeSearch()                 ║
        └─► newLegacySearch()               ║
        │                                   ║
        ▼                                   ║
    strategy.execute(r, sq, results, ...)   ║
        │                                   ║
        └─► 两阶段 FTS 执行（见 4.1 节）    ║
                                            ║
    ═════════════════════════════════════════╝
```

**关键特征**：
- 经过 `doSearch()` 函数，因此受单字符查询限制
- 使用两阶段 FTS 优化（Phase1: rowid 排序；Phase2: 全字段水化）
- 过滤条件在 Phase 1 的 WHERE 中叠加（排序前过滤，最优性能）

### 3.2 路径二：普通列表筛选（REST List Filter）

**入口**：Web UI 的歌曲/专辑/艺术家列表页，URL 查询参数如 `?title=love&genre=rock`。

**调用链**：

```
[REST API Handler]
    │
    ▼ rest.QueryOptions { Filters: {"title": "love", "genre": "rock"} }
    │
    ▼
    [persistence/sql_restful.go#L54-L67]
    parseRestOptions(ctx, restOptions) → model.QueryOptions
        │
        ▼ parseRestFilters() [sql_restful.go#L20-L52]
            │
            ├─► 遍历每个 filter key
            ├─► 匹配 filterMappings 中的自定义 filter 函数
            │   │
            │   └─► "title" → fullTextFilter("media_file", ...)
            │           │
            │           └─► [sql_restful.go#L109-L117]
            │               func(field, value) Sqlizer {
            │                   v := strings.ToLower(value.(string))
            │                   return cmp.Or[Sqlizer](
            │                       mbidExpr(tableName, v, mbidFields...),
            │                       getSearchStrategy(tableName, v),  ← 直接调用！
            │                   )
            │               }
            │
            ├─► "id" 结尾字段 → eqFilter()
            └─► 其他字段 → startsWithFilter()
        │
        ▼
    model.QueryOptions {
        Filters: And{
            fullTextFilter 返回的 Sqlizer,  // FTS5 或 LIKE 表达式
            genre_id tagIDFilter,
        }
    }
        │
        ▼
    [persistence/mediafile_repository.go#L199-L207]
    GetAll(options)
        │
        ├─► sq := selectMediaFile(options...)
        │   │
        │   └─► newSelect(options...) [sql_base_repository.go#L94-L102]
        │       │
        │       ├─► applyOptions(sq, options)   // LIMIT/OFFSET/ORDER
        │       └─► applyFilters(sq, options)   // ◄── 在此叠加！
        │           └─► sq.Where(options.Filters)  // 全量条件注入主查询 WHERE
        │
        └─► queryAll(sq, results, options...)
```

**关键特征**：
- `fullTextFilter` **直接调用** `getSearchStrategy()`，**绕过 `doSearch()`**
- **不受**单字符查询限制（Web UI 列表筛选允许单字符）
- 单阶段查询，搜索表达式作为子查询注入主查询 WHERE
- FTS5 退化为 `rowid IN (SELECT rowid FROM xxx_fts WHERE MATCH ?)` 形式
- 所有过滤条件（文本搜索 + 其他筛选器）在同一 WHERE 层叠加

### 3.3 路径三：智能播放列表筛选（Smart Playlist Criteria）

**入口**：智能播放列表的 `Rules` 字段，JSON 格式的 criteria 表达式树。

**调用链**：

```
[persistence/smart_playlist_repository.go#L19-L72]
refreshSmartPlaylist(pls)
    │
    ├─► rulesSQL := newSmartPlaylistCriteria(*pls.Rules, withSmartPlaylistOwner(*usr))
    │   │
    │   └─► [persistence/criteria_sql.go#L34-L45]
    │       type smartPlaylistCriteria struct {
    │           criteria.Criteria
    │           owner model.User
    │       }
    │
    ├─► refreshChildPlaylists(pls, rulesSQL)  // 递归刷新依赖的子播放列表
    ├─► resolvePercentageLimit(pls, &rulesSQL, usr.ID)
    │
    ├─► sq := buildSmartPlaylistQuery(pls, rulesSQL, usr.ID)
    │   │
    │   ├─► SELECT row_number() over (...) as id, ?, media_file.id
    │   │   FROM media_file
    │   ├─► addMediaFileAnnotationJoin(sq, userID)    // LEFT JOIN annotation
    │   ├─► addSmartPlaylistAnnotationJoins(sq, ...)  // LEFT JOIN album_annotation / artist_annotation
    │   └─► applyLibraryFilter(sq, "media_file")      // 库权限过滤
    │
    ├─► sq, err := addCriteria(sq, rulesSQL)
    │   │
    │   └─► [criteria_sql.go#L190-L202]
    │       func addCriteria(sql, cSQL) {
    │           cond, _ := cSQL.Where()
    │           sql = sql.Where(cond)  // ◄── 在此叠加！
    │           // criteria.Walk 遍历表达式树构建条件
    │           // 见 [criteria_sql.go#L129-L198] exprSQL()
    │       }
    │
    └─► INSERT INTO playlist_tracks SELECT ...
```

**Criteria 表达式遍历** ([criteria_sql.go#L129-L198](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/criteria_sql.go#L129-L198))：

| Criteria 操作符 | SQL 实现 |
|----------------|---------|
| `All` (AND) | `squirrel.And{}` |
| `Any` (OR) | `squirrel.Or{}` |
| `Is` / `Gt` / `Lt` | `Eq{}` / `Gt{}` / `Lt{}` |
| `Contains` / `NotContains` | `LIKE '%value%'` / `NOT LIKE '%value%'` |
| `StartsWith` / `EndsWith` | `LIKE 'value%'` / `LIKE '%value'` |
| `InTheRange` | `GtOrEq{} AND LtOrEq{}` |
| `InTheLast` / `NotInTheLast` | `> date(N days ago)` / `(< date OR NULL)` |
| `InPlaylist` / `NotInPlaylist` | `IN (subquery)` / `NOT IN (subquery)` |
| `IsMissing` / `IsPresent` | `NOT EXISTS json_tree` / `EXISTS json_tree` |

**关键特征**：
- 完全独立的查询构建路径，**不经过 `doSearch()`，也不调用 `getSearchStrategy()`**
- **不使用 FTS5**，模糊匹配用原生 `LIKE` 实现
- 不经过 Repository 层的 `Search()` 或 `GetAll()` 方法
- 直接构建 `INSERT INTO ... SELECT ...` 查询
- 通过 `criteria.Walk()` 遍历表达式树，按需引入 annotation JOIN
- **不受**单字符查询限制

---

## 4. 搜索策略选择与模糊匹配

### 4.1 策略路由逻辑

三条路径中，**只有 Search API 和 REST List Filter 经过** [getSearchStrategy()](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/sql_search.go#L38-L46)。智能播放列表 Criteria 不使用此路由。

```
输入 Query
    │
    ├─► conf.Search.Backend == "legacy" ────► Legacy (LIKE full_text)
    │   或 conf.Search.FullString == true
    │
    ├─► containsCJK(query) == true ────────► LIKE (核心列匹配)
    │   (Han / Hiragana / Katakana / Hangul)
    │
    └─► 其他 ──────────────────────────────► FTS5
                                              │
                                              └─► ftsQueryDegraded()
                                                    │
                                                    ├─ true  → fallback 到 LIKE
                                                    └─ false → 正常 FTS5
```

**CJK 检测** ([containsCJK()](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/sql_search_fts.go#L19-L29))：

```go
func containsCJK(s string) bool {
    for _, r := range s {
        if unicode.Is(unicode.Han, r) ||       // 中日韩统一表意文字
           unicode.Is(unicode.Hiragana, r) ||  // 平假名
           unicode.Is(unicode.Katakana, r) ||  // 片假名
           unicode.Is(unicode.Hangul, r) {     // 谚文
            return true
        }
    }
    return false
}
```

**FTS 退化检测** ([ftsQueryDegraded()](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/sql_search_fts.go#L360-L405))：

当查询主要由 FTS5 tokenizer 会剥离的特殊字符组成时（如 `"1+"`、`"C++"`、`"!!!"`、`"C#"`），FTS 会丢失大部分区分度，此时自动降级到 LIKE 搜索。

### 4.2 FTS5 模糊匹配

FTS5 查询构建由 [buildFTS5Query()](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/sql_search_fts.go#L146-L208) 完成，处理流程如下：

**Step 1：引号短语保护**
- 提取双引号包裹的短语为占位符 `\x00PHRASEn\x00`，后续处理不触碰短语内部

**Step 2：重音音译**
- 对引号外部分调用 `sanitize.Accents()`，将 `ø→o, æ→ae, œ→oe, ß→ss` 等 Unicode 字母转为 ASCII 变体

**Step 3：操作符中性化**
- 将 `AND, OR, NOT, NEAR` 转为小写（FTS5 操作符大小写敏感，小写后变成普通 token）

**Step 4：标点词处理** ([processPunctuatedWords()](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/sql_search_fts.go#L95-L124))

| 输入 | 模式 | 输出 |
|-----|-----|-----|
| `R.E.M.` | 点缩写（单字母+点） | `"R E M"`（短语查询） |
| `a-ha` / `AC/DC` / `Jay-Z` | 其他连接符 | `("a ha" OR aha*)`（短语 OR 连接前缀） |

**Step 5：特殊字符剥离 + 前缀通配**
- 移除非字母/数字/空白/`*`/`"`/`\x00` 的所有字符
- 为普通 token 追加 `*` 实现前缀匹配（`love` → `love*`）
- token 间使用显式 `AND` 连接

**Step 6：列过滤器注入**

最终 MATCH 表达式格式：
```
{title album artist album_artist sort_title ...} : (love* AND "r e m" AND ("a ha" OR aha*))
```

### 4.3 LIKE 模糊匹配

LIKE 搜索由 [likeSearchExpr()](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/sql_search_like.go#L84-L106) 实现：

```
词间关系：AND（每个词必须出现在某列中）
词内关系：OR（每个词可匹配任意核心列）
```

**搜索列范围**（[likeSearchColumns](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/sql_search_like.go#L74-L78)）：

| 表 | 列 |
|---|---|
| media_file | title, album, artist, album_artist |
| album | name, album_artist |
| artist | name |

生成的 SQL 模式示例：
```sql
-- 查询 "周杰伦 晴天"
(media_file.title LIKE '%周杰伦%' OR media_file.album LIKE '%周杰伦%' OR ...)
AND
(media_file.title LIKE '%晴天%' OR media_file.album LIKE '%晴天%' OR ...)
```

---

## 5. 过滤条件叠加层级对比

### 5.1 三条路径过滤叠加对比表

| 维度 | Search API (search2/3) | REST List Filter (Web UI) | Smart Playlist Criteria |
|-----|-----------------------|--------------------------|------------------------|
| **入口函数** | `.Search(q, options)` | `parseRestOptions()` → `.GetAll(options)` | `newSmartPlaylistCriteria()` → `addCriteria()` |
| **经过 doSearch()** | ✅ 是 | ❌ 否 | ❌ 否 |
| **调用 getSearchStrategy()** | ✅ 是（通过 strategy.execute） | ✅ 是（通过 fullTextFilter 直接调用） | ❌ 否（自实现 LIKE） |
| **使用 FTS5** | ✅ 是（两阶段优化） | ✅ 是（子查询形式） | ❌ 否（原生 LIKE） |
| **单字符限制** | ✅ 有（`len(q)<2` 返回空） | ❌ 无 | ❌ 无 |
| **过滤叠加点** | Phase 1 `rowidQuery` 的 WHERE | 主查询 WHERE | `buildSmartPlaylistQuery` 的 WHERE |
| **过滤时机** | 排序前过滤（最优） | 排序后或同时过滤（取决于查询计划） | 与排序同时（子查询一次性构建） |
| **库权限过滤** | Phase 1 叠加（applyLibraryFilter） | newSelect 内 applyLibraryFilter | buildSmartPlaylistQuery 内 applyLibraryFilter |
| **其他筛选器叠加** | Phase 1 WHERE 叠加 options.Filters | parseRestFilters 构建 And{}，GetAll 时注入主 WHERE | criteria.Walk 遍历表达式树，按需 JOIN |
| **执行阶段数** | 2 阶段（rowid 排序 + 字段水化） | 1 阶段（单条 SQL） | 1 阶段（INSERT SELECT） |

### 5.2 搜索接口（Search API）过滤叠加详解

**两阶段 FTS 执行** ([ftsSearch.execute()](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/sql_search_fts.go#L292-L339))：

```
Phase 1: Rowid 查询 (轻量)
┌──────────────────────────────────────────────────────────┐
│ SELECT media_file.rowid                                   │
│ FROM media_file                                           │
│ JOIN media_file_fts ON rowid = rowid                     │
│      AND media_file_fts MATCH ?                           │  ◄── 搜索条件
│ WHERE media_file.missing = false                          │
│   AND (library_id IN (SELECT ul.library_id ...))          │  ◄── 库权限过滤
│   AND (options.Filters)                                   │  ◄── 其他过滤条件（如 genre_id）
│ ORDER BY bm25(media_file_fts, 10,5,3,3,1,1,1,1,1,2,1)    │  ◄── BM25 排序
│ LIMIT ? OFFSET ?                                          │  ◄── 分页
└──────────────────────────────────────────────────────────┘
                    │
                    ▼ rowid 集合
Phase 2: 全字段水化
┌──────────────────────────────────────────────────────────┐
│ SELECT media_file.*, library.*, annotation.*, bookmark.*  │
│ FROM media_file                                           │
│ LEFT JOIN library ON library.id = media_file.library_id   │
│ LEFT JOIN annotation ON ...                               │
│ LEFT JOIN bookmark ON ...                                 │
│ JOIN (SELECT rowid as _rid,                               │
│       row_number() OVER () AS _rn                         │
│       FROM (Phase1_SQL)) AS _ranked                       │
│   ON media_file.rowid = _ranked._rid                      │
│ ORDER BY _ranked._rn                                      │
└──────────────────────────────────────────────────────────┘
```

> **关键设计**：所有过滤条件（搜索、权限、其他筛选器）都在 **Phase 1** 叠加，先过滤再排序，最小化排序记录集。

### 5.3 普通列表筛选（REST List Filter）过滤叠加详解

```
主查询 WHERE 层
┌──────────────────────────────────────────────────────────┐
│ SELECT media_file.*, library.*, annotation.*, bookmark.*  │
│ FROM media_file                                           │
│ LEFT JOIN library ON ...                                  │
│ LEFT JOIN annotation ON ...                               │
│ LEFT JOIN bookmark ON ...                                 │
│ WHERE                                                     │
│   media_file.missing = false                              │
│   AND (library_id IN (SELECT ul.library_id ...))          │  ◄── 库权限过滤
│   AND (                                                   │
│     media_file.mbz_recording_id = ?                       │  ◄── fullTextFilter: MBID 精确匹配
│     OR media_file.mbz_release_track_id = ?                │
│     OR media_file.rowid IN (                              │
│          SELECT rowid FROM media_file_fts                 │  ◄── fullTextFilter: FTS5 子查询
│          WHERE media_file_fts MATCH ?                     │
│        )                                                  │
│   )                                                       │
│   AND (                                                   │  ◄── parseRestFilters 构建
│     EXISTS (SELECT 1 FROM json_tree(media_file.tags, ...) │  ◄── genre_id 过滤
│   )                                                       │
│   AND (starred = ?)                                       │  ◄── starred 过滤
│ ORDER BY order_title ASC                                  │
│ LIMIT ? OFFSET ?                                          │
└──────────────────────────────────────────────────────────┘
```

> **特点**：所有条件在同一 WHERE 层通过 `And{}` 连接，SQLite 查询优化器决定执行顺序。FTS5 退化为子查询形式，不保证先过滤后排序。

### 5.4 智能播放列表（Smart Playlist）过滤叠加详解

```
INSERT INTO playlist_tracks (id, playlist_id, media_file_id)
SELECT
    row_number() over (order by play_count desc, title asc) as id,
    'playlist-xxx' as playlist_id,
    media_file.id as media_file_id
FROM media_file
LEFT JOIN annotation ON (annotation.item_id = media_file.id AND annotation.item_type = 'media_file' AND annotation.user_id = ?)
LEFT JOIN annotation AS album_annotation ON (album_annotation.item_id = media_file.album_id AND ...)
WHERE
    media_file.library_id IN (SELECT ul.library_id FROM user_library ul WHERE ul.user_id = ?)
    AND (                                                     ◄───┐
        COALESCE(annotation.play_count, 0) > 0                  │
        AND media_file.year >= 2000                              ├─ criteria.Walk 遍历构建
        AND media_file.title LIKE '%love%'                       │
        AND media_file.id NOT IN (SELECT media_file_id FROM ...)  │
    )                                                         ◄───┘
ORDER BY play_count desc, title asc
LIMIT 100
```

> **特点**：通过 `criteria.Walk()` 遍历表达式树，按需引入 annotation JOIN。所有条件直接构建在主 WHERE 中，完全不经过 Repository 层的搜索逻辑。

---

## 6. 单字符查询限制的影响范围

**限制代码位置**：[sql_search.go#L70-L75](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/sql_search.go#L70-L75)

```go
// Min-length guard: single-character queries are too broad for search3.
// This check lives here (not in the strategies) so that fullTextFilter
// (REST filter path) can still use single-character queries.
if len(q) < 2 {
    return nil
}
```

**受影响的路径**：✅ **只影响 Search API 路径**（search2/search3）

**不受影响的路径**：
- ❌ REST List Filter：`fullTextFilter` 直接调用 `getSearchStrategy()`，绕过 `doSearch()`
- ❌ Smart Playlist Criteria：完全独立的路径，不使用 `doSearch()`

**设计原因**：
- search3 接口面向全局搜索场景，单字符前缀查询（如 `a*`）在 FTS5 中几乎匹配所有记录，BM25 排序开销极高
- REST List Filter 通常配合其他过滤条件（如 genre、year 等），即使单字符也不会召回全库
- 代码注释明确说明此检查放在 `doSearch()` 而不是 strategy 中，就是为了让 `fullTextFilter` 能继续使用单字符

---

## 7. 三条路径的相互关系

### 7.1 共享组件

| 组件 | Search API | REST List Filter | Smart Playlist |
|-----|------------|------------------|----------------|
| `getSearchStrategy()` 策略选择器 | ✅ 是 | ✅ 是 | ❌ 否 |
| FTS5 虚拟表索引 | ✅ 是（两阶段） | ✅ 是（子查询） | ❌ 否 |
| LIKE 搜索实现 | ✅ 是（fallback） | ✅ 是（fallback） | ❌ 否（自实现） |
| `applyLibraryFilter()` 库权限 | ✅ 是（Phase 1） | ✅ 是（newSelect） | ✅ 是（buildQuery） |
| `QueryOptions` 结构 | ✅ 是 | ✅ 是 | ❌ 否 |

### 7.2 调用关系图

```
                  User Query
                      │
         ┌────────────┼────────────┐
         │            │            │
    Subsonic      REST URL   Smart Playlist
    search?q=    ?title=     Rules JSON
         │            │            │
         ▼            ▼            ▼
  .Search(q, options) .GetAll(options) refreshSmartPlaylist()
         │            │            │
         ├────────────┘            │
         │                         │
         ▼                         ▼
  doSearch()              newSmartPlaylistCriteria()
         │                         │
         ▼                         ▼
  getSearchStrategy()        criteria.Walk()
         │                         │
         ├─────────────────────────┤
         │                         │
         ▼                         ▼
  strategy.execute()    INSERT INTO playlist_tracks SELECT...
         │
         ▼
  Phase 1 (rowid + filters + sort + limit)
         │
         ▼
  Phase 2 (full field hydration)
```

### 7.3 设计意图

1. **Search API** 面向全局搜索，性能要求最高 → 两阶段 FTS 优化，严格单字符保护
2. **REST List Filter** 面向列表页筛选，需要灵活组合条件 → 直接 WHERE 叠加，允许单字符
3. **Smart Playlist** 面向后台异步刷新，需要完整的 Criteria DSL 表达能力 → 独立查询构建，不依赖搜索引擎

---

## 8. 性能取舍点分析

### 8.1 FTS5 vs LIKE 的取舍

| 维度 | FTS5 | LIKE (CJK/Fallback) |
|-----|------|-------------------|
| **查询速度** | 极快（倒排索引 + BM25） | 慢（全表扫描，`%word%` 无法用 B-Tree 索引） |
| **匹配精度** | 词级前缀匹配，支持短语 | 子串匹配，精度更高但召回过大 |
| **CJK 支持** | 差（不分词） | 好（子串匹配天然支持中文） |
| **特殊字符** | 需预处理，可能丢失信息 | 原样保留 (`C++`、`1+` 可精确匹配) |
| **排序质量** | BM25 加权排名 | 只能按列名排序，无语义相关性 |
| **索引体积** | 较大（单独 FTS5 表） | 零额外索引开销 |

**关键优化**：`ftsQueryDegraded()` 检测——当 FTS5 会因特殊字符剥离导致 token 过短时（`≤2` 字符），自动退回 LIKE，牺牲速度换召回正确性。

### 8.2 两阶段查询 vs 单阶段查询

| 维度 | 两阶段 (Search API) | 单阶段 (REST Filter / Smart Playlist) |
|-----|-------------------|-------------------------------------|
| **大数据量性能** | 优秀（Phase1 排序仅 rowid + FTS） | 较差（排序包含所有 JOIN 列） |
| **分页一致性** | Phase1 `row_number() OVER ()` 保证顺序 | 依赖 SQLite 查询优化器，OFFSET 较大时不稳定 |
| **实现复杂度** | 高（需维护两套查询组装） | 低（直接 WHERE 表达式） |
| **过滤叠加点** | Phase1 的 WHERE（过滤再排序，最优） | 主查询 WHERE（可能排序后再过滤） |

### 8.3 search_normalized 辅助列的取舍

**收益**：
- 解决 FTS5 `unicode61` tokenizer 无法处理名称标点的问题（`R.E.M.`, `a-ha`, `AC/DC`）
- 解决 `remove_diacritics 2` 对原子字母 (`ø`, `æ`, `ß`) 无效的问题

**代价**：
- 主表增加 2 个 TEXT 列（`search_participants` + `search_normalized`），占用额外存储
- 每次写入需要 Go 端 `normalizeForFTS()` 计算
- FTS5 索引中 `search_normalized` 列的 token 冗余

### 8.4 单字符查询限制的取舍

**收益**：
- 保护 search3 接口免受 FTS5 前缀匹配全库的性能冲击
- `a*` 前缀查询在万级曲库可能召回 30%+ 记录，BM25 排序 + 多表 JOIN 代价极高

**代价**：
- 限制了搜索功能的使用场景（如用户想搜 `"a"` 开头的歌曲）
- 两条路径行为不一致可能导致用户困惑

**缓解**：限制只作用于全局搜索接口，列表筛选路径仍然允许单字符。

### 8.5 CJK 降级的性能权衡

含 CJK 的查询直接降级 LIKE，这意味着：
- **写入无额外开销**：不需要为中文维护 n-gram 或 Jieba 分词索引
- **查询性能随库大小线性下降**：万级歌曲可接受，十万级以上会明显变慢
- **未来可扩展**：架构上通过 `searchStrategy` 接口抽象，后续可接入 ICU 分词器或中文分词插件而不影响上层

### 8.6 库权限过滤的位置

**Phase 1 先过滤再排序**是性能关键设计：

```go
// Phase 1 中
if cfg.LibraryFilter != nil {
    rowidQuery = cfg.LibraryFilter(rowidQuery)  // 先过滤
} else {
    rowidQuery = r.applyLibraryFilter(rowidQuery)
}
if options.Filters != nil {
    rowidQuery = rowidQuery.Where(options.Filters)
}
// 然后才 LIMIT/OFFSET
```

如果先排序再过滤，需要对全库 BM25 排序后才丢弃无权访问的记录，O(N log N) 中的 N 是全库而不是用户可见子集。

### 8.7 分页优化

[optimizePagination()](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/sql_base_repository.go#L367-L376) 在 OFFSET 超过阈值（`conf.Server.DevOffsetOptimize`）时，使用 `rowid NOT IN (subquery LIMIT offset)` 替代原生 `OFFSET`：

```go
func (r sqlRepository) optimizePagination(sq SelectBuilder, options model.QueryOptions) SelectBuilder {
    if options.Offset > conf.Server.DevOffsetOptimize {
        sq = sq.RemoveOffset()
        rowidSq := sq.RemoveColumns().Columns(r.tableName + ".rowid")
        rowidSq = rowidSq.Limit(uint64(options.Offset))
        rowidSql, args, _ := rowidSq.ToSql()
        sq = sq.Where(r.tableName+".rowid not in ("+rowidSql+")", args...)
    }
    return sq
}
```

适用于深分页场景，避免 SQLite 跳过前 N 行时仍需扫描并丢弃它们。

---

## 9. 关键数据结构汇总

### QueryOptions ([datastore.go#L10-L17](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/model/datastore.go#L10-L17))

```go
type QueryOptions struct {
    Sort    string
    Order   string
    Max     int
    Offset  int
    Filters squirrel.Sqlizer  // 任意复杂的 WHERE 条件树
    Seed    string            // 随机排序种子
}
```

### searchConfig ([sql_search.go#L19-L26](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/sql_search.go#L19-L26))

```go
type searchConfig struct {
    NaturalOrder  string                       // 空查询时的 ORDER BY
    OrderBy       []string                     // 文本搜索时的辅助排序列
    MBIDFields    []string                     // UUID 查询时匹配的 MBID 列
    LibraryFilter func(SelectBuilder) SelectBuilder  // 自定义库过滤钩子
}
```

### searchStrategy 接口 ([sql_search.go#L31-L34](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/sql_search.go#L31-L34))

```go
type searchStrategy interface {
    Sqlizer                                          // REST 路径：单表达式注入
    execute(r sqlRepository, sq SelectBuilder, dest any,
            cfg searchConfig, options model.QueryOptions) error  // Search API 路径：两阶段执行
}
```

三种实现：`ftsSearch`、`likeSearch`（包含 legacy 和 CJK/fallback 两种模式）。

---

## 10. 总结

Navidrome 的搜索架构体现了典型的"**场景分层 + 渐进降级 + 路径专用**"设计思想：

1. **三条独立路径**：Search API（全局搜索，两阶段优化，单字符保护）、REST List Filter（列表筛选，灵活组合，允许单字符）、Smart Playlist（后台异步，Criteria DSL，不依赖搜索引擎）

2. **共享搜索策略选择器**：`getSearchStrategy()` 根据查询内容（CJK/非 CJK、特殊字符占比）和配置自动选择 FTS5 或 LIKE，但只有前两条路径使用它

3. **过滤叠加层级差异**：
   - Search API：Phase 1 `rowidQuery` WHERE 先过滤再排序（最优）
   - REST List Filter：主查询 WHERE 层统一叠加（SQLite 优化器决定顺序）
   - Smart Playlist：Criteria 表达式树 Walk 构建，按需 JOIN

4. **单字符限制只作用于 Search API**：通过将检查放在 `doSearch()` 而不是 `getSearchStrategy()`，实现了路径差异化

5. **两阶段查询**：Subsonic Search API 使用 Phase1 排 + Phase2 水的模式，在大结果集下保持分页性能

6. **索引补偿列**：`search_normalized` + `search_participants` 用存储空间换 FTS5 的召回率

7. **降级兜底**：FTS 退化时退回 LIKE，CJK 直接走 LIKE，保证极端查询不丢结果

性能取舍的核心落点是：**FTS5 的倒排索引速度 vs LIKE 的子串匹配召回完整性**，以及**两阶段查询的工程复杂度 vs 深分页性能**，以及**全局搜索的保护性限制 vs 列表筛选的灵活性**。
