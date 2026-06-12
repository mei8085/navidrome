# Navidrome 曲库搜索架构分析报告

## 1. 整体架构概览

Navidrome 的搜索系统采用**多策略、两阶段**的分层架构，核心入口位于 [sql_search.go](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/sql_search.go)。搜索请求从 API 层经过 Repository 层，最终根据查询内容自动选择最优搜索策略。

```
API (searching.go)
    │
    ▼
Repository.Search()  ──►  QueryOptions (Filters/Max/Offset/Sort)
    │
    ▼
doSearch()  [sql_search.go#L52-L83]
    │
    ├── 空查询 → 自然顺序返回
    ├── UUID 查询 → MBID 精确匹配
    ├── 单字符 → 直接返回空 (search3 保护)
    │
    ▼
getSearchStrategy()  [sql_search.go#L38-L46]
    │
    ├── Legacy 模式 ──────────────► newLegacySearch()  [LIKE full_text]
    ├── 含 CJK 字符 ──────────────► newLikeSearch()    [LIKE 核心列]
    ├── 非 CJK / 非 Legacy ───────► newFTSSearch()     [FTS5 + BM25]
    │                                      │
    │                                      └── 退化检测 → fallback 到 LIKE
    ▼
strategy.execute()
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

**BM25 权重配置**（[sql_search_fts.go#L220-L249](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/sql_search_fts.go#L220-L249)）：

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

## 3. 搜索策略选择逻辑

策略路由函数 [getSearchStrategy()](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/sql_search.go#L38-L46) 的决策树：

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

---

## 4. 模糊匹配策略详解

### 4.1 FTS5 模式

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

### 4.2 LIKE 模式（CJK / Fallback）

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

### 4.3 Legacy 模式

Legacy 模式使用 `full_text` 列（所有可搜索字段拼接的去规范化字符串），所有词用 `AND` 连接做 `%word%` 匹配。

---

## 5. 过滤条件叠加的查询路径

### 5.1 两种搜索入口路径

Navidrome 有两条独立的搜索调用路径：

| 路径 | 入口 | 使用场景 |
|-----|-----|---------|
| **Subsonic API Search** | `.Search(q, options...)` | search2 / search3 端点 |
| **REST List Filter** | `fullTextFilter` 通过 `.GetAll(options)` | Web UI 列表过滤、智能播放列表 |

### 5.2 Subsonic API 搜索路径 (两阶段)

以 `mediaFileRepository.Search()` 为例 ([mediafile_repository.go#L441-L452](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/mediafile_repository.go#L441-L452))：

```go
func (r *mediaFileRepository) Search(q string, options ...model.QueryOptions) (model.MediaFiles, error) {
    // sq 已经包含 JOIN (library, annotation, bookmark) + library filter
    sq := r.selectMediaFile(options...)
    return r.doSearch(sq, q, &res, mediaFileSearchConfig, opts)
}
```

**两阶段 FTS 执行** ([ftsSearch.execute()](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/sql_search_fts.go#L292-L339))：

```
Phase 1: Rowid 查询 (轻量)
┌─────────────────────────────────────────────┐
│ SELECT media_file.rowid                      │
│ FROM media_file                              │
│ JOIN media_file_fts ON rowid = rowid        │
│      AND media_file_fts MATCH ?              │
│ WHERE media_file.missing = false             │
│   AND (options.Filters)                      │  ◄── 过滤条件在此叠加
│   AND (library_filter)                       │
│ ORDER BY bm25(media_file_fts, 10,5,3,...)   │
│ LIMIT ? OFFSET ?                             │  ◄── 分页在此应用
└─────────────────────────────────────────────┘
                    │
                    ▼ rowid 集合
Phase 2: 全字段水化
┌─────────────────────────────────────────────┐
│ SELECT media_file.*, library.*, annotation.* │
│ FROM media_file                              │
│ LEFT JOIN library ...                        │
│ LEFT JOIN annotation ...                     │
│ LEFT JOIN bookmark ...                       │
│ JOIN (SELECT rowid as _rid,                  │
│       row_number() OVER () AS _rn            │
│       FROM (Phase1_SQL)) AS _ranked          │
│   ON media_file.rowid = _ranked._rid         │
│ ORDER BY _ranked._rn                         │
└─────────────────────────────────────────────┘
```

**设计原因**：Phase 1 只查 rowid 避免 JOIN 大表排序，Phase 2 用已排序的 rowid 集合做等值 JOIN，效率远高于单条大 SQL。

**过滤条件叠加时机**：
- `options.Filters`（如 `library_id IN (...)`）在 Phase 1 的 `rowidQuery` 上直接 `.Where()` 叠加
- `libraryFilter` 通过 `cfg.LibraryFilter` 钩子或 `r.applyLibraryFilter()` 在 Phase 1 注入
- artist 表因为需要 `library_artist` 关联表过滤，使用自定义 `LibraryFilter` 函数 ([artist_repository.go#L550-L551](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/artist_repository.go#L550-L551))

### 5.3 REST 过滤路径 (单阶段)

Web UI 中在歌曲/专辑列表按名称过滤时，通过 `fullTextFilter` ([helpers.go#L109-L117](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/helpers.go#L109-L117))：

```go
func fullTextFilter(tableName string, mbidFields ...string) func(string, any) Sqlizer {
    return func(field string, value any) Sqlizer {
        v := strings.ToLower(value.(string))
        return cmp.Or[Sqlizer](
            mbidExpr(tableName, v, mbidFields...),  // 先尝试 UUID/MBID 精确匹配
            getSearchStrategy(tableName, v),        // 再走正常搜索策略
        )
    }
}
```

此路径返回的是一个 `Sqlizer` 表达式，直接注入到主查询的 WHERE 中，**不走两阶段优化**。FTS 部分退化为子查询：
```sql
media_file.rowid IN (SELECT rowid FROM media_file_fts WHERE media_file_fts MATCH ?)
```

### 5.4 智能播放列表 Criteria 路径

智能播放列表使用独立的 Criteria DSL ([criteria_sql.go](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/criteria_sql.go))，支持更丰富的操作符：

| 操作符 | SQL 实现 |
|-------|---------|
| `Contains` | `LIKE '%value%'` |
| `StartsWith` | `LIKE 'value%'` |
| `EndsWith` | `LIKE '%value'` |
| `InTheRange` | `BETWEEN` (GtOrEq + LtOrEq) |
| `InTheLast` | `> date(N days ago)` |
| `IsMissing/IsPresent` | `json_tree` EXISTS / NOT EXISTS (仅 Tag/Role) |
| `InPlaylist/NotInPlaylist` | `IN (subquery)` / `NOT IN (subquery)` |

Criteria 通过 Walk 遍历表达式树，按需引入 annotation JOIN（album/artist 评分、收藏）。

---

## 6. 性能取舍点分析

### 6.1 FTS5 vs LIKE 的取舍

| 维度 | FTS5 | LIKE (CJK/Fallback) |
|-----|------|-------------------|
| **查询速度** | 极快（倒排索引 + BM25） | 慢（全表扫描，`%word%` 无法用 B-Tree 索引） |
| **匹配精度** | 词级前缀匹配，支持短语 | 子串匹配，精度更高但召回过大 |
| **CJK 支持** | 差（不分词） | 好（子串匹配天然支持中文） |
| **特殊字符** | 需预处理，可能丢失信息 | 原样保留 (`C++`、`1+` 可精确匹配) |
| **排序质量** | BM25 加权排名 | 只能按列名排序，无语义相关性 |
| **索引体积** | 较大（单独 FTS5 表） | 零额外索引开销 |

**关键优化**：`ftsQueryDegraded()` 检测——当 FTS5 会因特殊字符剥离导致 token 过短时（`≤2` 字符），自动退回 LIKE，牺牲速度换召回正确性。

### 6.2 两阶段查询 vs 单阶段查询

| 维度 | 两阶段 (Search API) | 单阶段 (REST Filter) |
|-----|-------------------|--------------------|
| **大数据量性能** | 优秀（Phase1 排序仅 rowid + FTS） | 较差（排序包含所有 JOIN 列） |
| **分页一致性** | Phase1 `row_number() OVER ()` 保证顺序 | 依赖 SQLite 查询优化器，OFFSET 较大时不稳定 |
| **实现复杂度** | 高（需维护两套查询组装） | 低（直接 WHERE 表达式） |
| **过滤叠加点** | Phase1 的 WHERE（过滤再排序，最优） | 主查询 WHERE（可能排序后再过滤） |

### 6.3 search_normalized 辅助列的取舍

**收益**：
- 解决 FTS5 `unicode61` tokenizer 无法处理名称标点的问题（`R.E.M.`, `a-ha`, `AC/DC`）
- 解决 `remove_diacritics 2` 对原子字母 (`ø`, `æ`, `ß`) 无效的问题

**代价**：
- 主表增加 2 个 TEXT 列（`search_participants` + `search_normalized`），占用额外存储
- 每次写入需要 Go 端 `normalizeForFTS()` 计算
- FTS5 索引中 `search_normalized` 列的 token 冗余

### 6.4 单字符查询限制

[doSearch() L73-L75](file:///d:/fz/0601-1/solo-dogfeeding/code/30-navidrome/persistence/sql_search.go#L73-L75) 对 search3 路径强制 `len(q) < 2` 返回空：

```go
if len(q) < 2 {
    return nil
}
```

**原因**：单字符前缀查询（如 `a*`）在 FTS5 中几乎匹配所有记录，BM25 排序开销极高。REST 路径（`fullTextFilter`）不受此限制，因为 Web UI 的列表过滤场景通常还有其他过滤条件叠加。

### 6.5 CJK 降级的性能权衡

含 CJK 的查询直接降级 LIKE，这意味着：
- **写入无额外开销**：不需要为中文维护 n-gram 或 Jieba 分词索引
- **查询性能随库大小线性下降**：万级歌曲可接受，十万级以上会明显变慢
- **未来可扩展**：架构上通过 `searchStrategy` 接口抽象，后续可接入 ICU 分词器或中文分词插件而不影响上层

### 6.6 库权限过滤的位置

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

### 6.7 分页优化

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

## 7. 关键数据结构汇总

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

## 8. 总结

Navidrome 的搜索架构体现了典型的"**场景分层 + 渐进降级**"设计思想：

1. **分层策略选择**：根据查询内容（CJK/非 CJK、特殊字符占比）和配置（Legacy/FTS5）自动选择最优引擎
2. **两阶段查询**：Subsonic Search API 使用 Phase1 排 + Phase2 水的模式，在大结果集下保持分页性能
3. **索引补偿列**：`search_normalized` + `search_participants` 用存储空间换 FTS5 的召回率
4. **降级兜底**：FTS 退化时退回 LIKE，CJK 直接走 LIKE，保证极端查询不丢结果
5. **过滤前置**：权限和条件过滤在排序前应用，最小化排序和扫描的记录集

性能取舍的核心落点是：**FTS5 的倒排索引速度 vs LIKE 的子串匹配召回完整性**，以及**两阶段查询的工程复杂度 vs 深分页性能**。
