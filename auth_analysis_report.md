# Navidrome 认证机制与公开访问路径分析报告

## 1. 架构概览

Navidrome 采用 **JWT (JSON Web Token)** 作为核心认证机制，同时支持多种认证路径和公开访问模式。系统中并行存在三种主要访问模式：

| 访问模式 | 认证方式 | 典型路径 | 权限范围 |
|---------|---------|---------|---------|
| 登录用户访问 | JWT 会话令牌 | `/api/*`, `/rest/*` | 完整用户权限 |
| 公开分享访问 | 公开 JWT 令牌 | `/share/s/*`, `/share/img/*` | 仅限指定资源 |
| 公开页面访问 | 分享 ID | `/share/{id}` | 分享元数据访问 |

---

## 2. 核心认证实现

### 2.1 JWT 基础设施

**核心文件**: `core/auth/auth.go:26-46`

```go
func Init(ds model.DataStore) {
    once.Do(func() {
        secret, err := ds.Property(ctx).Get(consts.JWTSecretKey)
        if err != nil || secret == "" {
            secret = createNewSecret(ctx, ds)  // 生成并存储新密钥
        }
        TokenAuth = jwtauth.New("HS256", []byte(secret), nil)
    })
}
```

**关键特性**:
- 算法：HS256 (HMAC-SHA256)
- 密钥存储：数据库 `property` 表，加密存储
- 签发者：`ND` (Navidrome)
- 密钥轮换：更换密钥会使所有现有令牌失效

### 2.2 Claims 结构定义

**核心文件**: `core/auth/claims.go:9-104`

```go
type Claims struct {
    // 标准 JWT 声明
    Issuer    string    // iss
    Subject   string    // sub - 用户名（会话令牌）
    IssuedAt  time.Time // iat
    ExpiresAt time.Time // exp
    
    // 自定义声明
    UserID   string // uid - 用户 ID
    IsAdmin  bool   // adm - 管理员标记
    ID       string // id - 资源 ID（公开令牌）
    Format   string // f  - 音频格式
    BitRate  int    // b  - 比特率
    ShareID  string // sid - 分享 ID
}
```

---

## 3. 登录用户认证流程

### 3.1 Native API 认证链

**路由配置**: `server/nativeapi/native_api.go:63-95`

```
请求 → JWTVerifier → Authenticator → JWTRefresher → UpdateLastAccessMiddleware → 业务处理器
```

#### 阶段 1: JWTVerifier (`server/auth.go:174-176`)

```go
func JWTVerifier(next http.Handler) http.Handler {
    return jwtauth.Verify(auth.TokenAuth, 
        tokenFromHeader,        // X-ND-Authorization: Bearer <token>
        jwtauth.TokenFromCookie,
        jwtauth.TokenFromQuery, // ?auth=<token>
    )(next)
}
```

**令牌提取优先级**:
1. 自定义请求头 `X-ND-Authorization`
2. Cookie
3. 查询参数

#### 阶段 2: Authenticator (`server/auth.go:260-272`)

```go
func Authenticator(ds model.DataStore) func(next http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx, err := authenticateRequest(ds, r,
                UsernameFromConfig,       // 开发模式自动登录
                UsernameFromToken,        // 从 JWT 提取
                UsernameFromExtAuthHeader // 反向代理认证
            )
            // ... 错误处理
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

**认证来源优先级**:
1. 配置自动登录（开发环境）
2. JWT 令牌
3. 外部认证头（需配置可信代理）

#### 阶段 3: JWTRefresher (`server/auth.go:275-293`)

```go
func JWTRefresher(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token, _, err := jwtauth.FromContext(ctx)
        if err == nil {
            newTokenString, _ := auth.TouchToken(token)
            w.Header().Set(consts.UIAuthorizationHeader, newTokenString)
        }
        next.ServeHTTP(w, r)
    })
}
```

**滑动会话机制**:
- 每次有效请求都会更新 `ExpiresAt`
- 新令牌通过响应头 `X-ND-Authorization` 返回
- 会话超时：默认 48 小时（`DefaultSessionTimeout`）

### 3.2 Subsonic API 认证

**核心文件**: `server/subsonic/middlewares.go:100-156`

Subsonic API 支持四种认证方式：

| 方式 | 参数 | 说明 |
|------|------|------|
| 明文密码 | `u` + `p` | `p` 支持 `enc:` 前缀的十六进制编码 |
| 令牌认证 | `u` + `t` + `s` | `t = md5(password + s)` |
| JWT 认证 | `u` + `jwt` | 复用 Navidrome 会话令牌 |
| 内部/代理 | 头信息 | 插件调用或反向代理 |

---

## 4. 公开访问路径分析

### 4.1 公开路由配置

**核心文件**: `server/public/public.go:38-60`

```go
func (pub *Router) routes() http.Handler {
    r := chi.NewRouter()
    r.Group(func(r chi.Router) {
        r.Use(server.URLParamsMiddleware)
        r.HandleFunc("/img/{id}", pub.handleImages)    // 公开图片
        if conf.Server.EnableSharing {
            r.HandleFunc("/s/{id}", pub.handleStream)  // 公开流媒体
            r.HandleFunc("/d/{id}", pub.handleDownloads) // 公开下载
            r.HandleFunc("/{id}/m3u", pub.handleM3U)   // M3U 播放列表
            r.HandleFunc("/{id}", pub.handleShares)    // 分享页面
        }
    })
    return r
}
```

> **重要**: 公开路由 **不经过** `Authenticator` 和 `JWTRefresher` 中间件，也不调用 `UpdateLastAccessMiddleware`。

### 4.2 公开图片访问 (`/share/img/{id}`)

**核心文件**: `server/public/handle_images.go:17-88`

```go
func decodeArtworkID(tokenString string) (model.ArtworkID, error) {
    token, err := auth.TokenAuth.Decode(tokenString)  // 仅解码，不验证过期
    if err != nil {
        return model.ArtworkID{}, err
    }
    c := auth.ClaimsFromToken(token)
    if c.ID == "" {
        return model.ArtworkID{}, errors.New("required claim \"id\" not found")
    }
    return model.ParseArtworkID(c.ID)
}
```

**访问控制特点**:
- 令牌验证：仅验证签名，**不检查过期时间**
- 权限：只能访问令牌中 `id` 声明指定的资源
- 缓存：响应头设置 `Cache-Control: public, max-age=315360000`（10年）

### 4.3 公开流媒体访问 (`/share/s/{id}`)

**核心文件**: `server/public/handle_streams.go:17-97`

```go
func decodeStreamInfo(tokenString string) (shareTrackInfo, error) {
    c, err := auth.Validate(tokenString)  // 完整验证（包括过期）
    if err != nil {
        return shareTrackInfo{}, err
    }
    return shareTrackInfo{
        id:      c.ID,
        format:  c.Format,
        bitrate: c.BitRate,
        shareID: c.ShareID,
    }, nil
}
```

**双重验证机制**:
1. JWT 令牌验证（检查过期）
2. 分享记录验证（如果令牌包含 `ShareID`）

```go
if info.shareID != "" {
    share, err := pub.ds.Share(ctx).Get(info.shareID)
    if err != nil {
        checkShareError(ctx, w, err, info.shareID)
        return
    }
    if expiresAt := V(share.ExpiresAt); !expiresAt.IsZero() && expiresAt.Before(time.Now()) {
        checkShareError(ctx, w, model.ErrExpired, info.shareID)
        return
    }
}
```

### 4.4 分享页面访问 (`/share/{id}`)

**核心文件**: `server/public/handle_shares.go:21-44`

```go
func (pub *Router) handleShares(w http.ResponseWriter, r *http.Request) {
    s, err := pub.share.Load(r.Context(), id)
    if err != nil {
        checkShareError(r.Context(), w, err, id)
        return
    }
    // 增加访问计数
    s = pub.mapShareInfo(r, *s)
    server.IndexWithShare(pub.ds, ui.BuildAssets(), s)(w, r)
}
```

**分享加载逻辑**: `core/share.go:34-52`

```go
func (s *shareService) Load(ctx context.Context, id string) (*model.Share, error) {
    share, err := repo.Get(id)
    if err != nil {
        return nil, err
    }
    // 检查分享是否过期
    expiresAt := V(share.ExpiresAt)
    if !expiresAt.IsZero() && expiresAt.Before(time.Now()) {
        return nil, model.ErrExpired
    }
    // 更新访问统计
    share.LastVisitedAt = P(time.Now())
    share.VisitCount++
    repo.(rest.Persistable).Update(id, share, "last_visited_at", "visit_count")
    return share, nil
}
```

---

## 5. 请求上下文字段详解

### 5.1 上下文键定义

**核心文件**: `model/request/request.go:9-21`

```go
const (
    User           = contextKey("user")           // model.User - 完整用户对象
    Username       = contextKey("username")       // string - 用户名
    Client         = contextKey("client")         // string - 客户端标识（Subsonic）
    Version        = contextKey("version")        // string - API 版本
    Player         = contextKey("player")         // model.Player - 播放器信息
    Transcoding    = contextKey("transcoding")    // model.Transcoding - 转码配置
    ClientUniqueId = contextKey("clientUniqueId") // string - 客户端唯一ID
    ReverseProxyIp = contextKey("reverseProxyIp") // string - 反向代理IP
    InternalAuth   = contextKey("internalAuth")   // string - 内部认证用户名
)
```

### 5.2 字段来源对比

| 字段 | 登录用户请求 | 公开分享请求 | 设置位置 |
|------|-------------|-------------|---------|
| `User` | ✅ 完整对象 | ❌ 不存在 | `Authenticator` / Subsonic `authenticate` |
| `Username` | ✅ 用户名 | ❌ 不存在 | `Authenticator` / Subsonic `checkRequiredParameters` |
| `Client` | ✅ (Subsonic) | ❌ 不存在 | Subsonic `checkRequiredParameters` |
| `Player` | ✅ (Subsonic) | ❌ 不存在 | Subsonic `getPlayer` |
| `Transcoding` | ✅ (Subsonic) | ❌ 不存在 | Subsonic `getPlayer` |
| `ClientUniqueId` | ✅ 可选 | ❌ 不存在 | `clientUniqueIDMiddleware` |
| `ReverseProxyIp` | ✅ 可选 | ✅ 可选 | `realIPMiddleware`（仅配置时） |
| `InternalAuth` | ✅ 可选 | ❌ 不存在 | 插件调用时设置 |

### 5.3 业务层使用示例

**管理员权限检查**: `server/nativeapi/native_api.go:258-267`

```go
func adminOnlyMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        user, ok := request.UserFrom(r.Context())
        if !ok || !user.IsAdmin {
            http.Error(w, "Access denied: admin privileges required", http.StatusForbidden)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

**用户最后访问时间更新**: `server/middlewares.go:304-329`

```go
func UpdateLastAccessMiddleware(ds model.DataStore) func(next http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx := r.Context()
            usr, ok := request.UserFrom(ctx)
            if ok {  // 只有登录用户才会更新
                userAccessLimiter.Do(usr.ID, func() {
                    ds.User(ctx).UpdateLastAccessAt(usr.ID)
                })
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

---

## 6. 令牌类型与权限差异

### 6.1 令牌类型对比

| 特性 | 会话令牌 (Session) | 分享令牌 (Share) | 转码令牌 (Transcode) |
|------|-------------------|-----------------|---------------------|
| 创建函数 | `CreateToken(u)` | `CreateExpiringPublicToken(exp, claims)` | `EncodeToken(decision.toClaimsMap())` |
| 包含声明 | `sub`, `uid`, `adm`, `iat`, `exp` | `id`, `f`, `b`, `sid`, `exp`(可选) | `mid`, `ua`, `dp`, `f`, `b`, `ch`, `sr`, `bd`, `exp` |
| 过期时间 | 48 小时（滑动） | 分享过期时间或永久 | 48 小时 |
| 可访问路径 | 所有受保护 API | `/share/s/*`, `/share/img/*` | 转码流端点 |
| 权限范围 | 用户完整权限 | 单个资源 | 单个资源转码 |

### 6.2 分享令牌生成

**核心文件**: `server/public/handle_shares.go:100-109`

```go
func encodeMediafileShare(s model.Share, id string) string {
    claims := auth.Claims{
        ID:      id,        // 媒体文件 ID
        Format:  s.Format,  // 允许的格式
        BitRate: s.MaxBitRate, // 最大比特率
        ShareID: s.ID,      // 关联的分享记录
    }
    token, _ := auth.CreateExpiringPublicToken(V(s.ExpiresAt), claims)
    return token
}
```

---

## 7. 令牌过期与撤销边界

### 7.1 过期机制

| 令牌类型 | 过期检查点 | 过期后行为 |
|---------|-----------|-----------|
| 会话令牌 | `JWTVerifier` (jwtauth 库) | 返回 401 Unauthorized |
| 分享流令牌 | `auth.Validate()` + 分享记录检查 | 返回 400 Bad Request 或 410 Gone |
| 分享页面 | `share.Load()` 中检查 | 返回 410 Gone |
| 转码令牌 | `parseTranscodeParams()` | 返回错误，需要重新获取 |
| 公开图片 | **不检查过期** | 永久可访问（只要签名有效） |

### 7.2 撤销机制

#### 7.2.1 显式撤销（分享）

分享令牌可以通过删除分享记录实现"软撤销"：

```go
// 分享流访问时的二次检查
if info.shareID != "" {
    share, err := pub.ds.Share(ctx).Get(info.shareID)
    if err != nil {  // 分享已删除 → 拒绝访问
        checkShareError(ctx, w, err, info.shareID)
        return
    }
}
```

#### 7.2.2 隐式撤销

1. **JWT 密钥轮换**：更换数据库中的 JWT 密钥会使所有令牌失效
2. **用户密码变更**：不影响现有 JWT 令牌（无令牌黑名单机制）
3. **用户删除**：下次请求时 `Authenticator` 会找不到用户，返回 401

### 7.3 访问控制边界总结

| 场景 | 会话令牌 | 分享令牌 | 公开图片令牌 |
|------|---------|---------|-------------|
| 用户删除后 | ❌ 拒绝（Authenticator） | ✅ 仍可访问（无用户检查） | ✅ 仍可访问 |
| 分享删除后 | N/A | ❌ 拒绝（DB检查） | ✅ 仍可访问 |
| 令牌过期后 | ❌ 拒绝 | ❌ 拒绝 | ✅ 仍可访问（不检查） |
| 密钥轮换后 | ❌ 拒绝 | ❌ 拒绝 | ❌ 拒绝 |
| 用户权限变更 | ✅ 立即生效（每次从DB加载） | N/A | N/A |

---

## 8. 安全设计分析

### 8.1 优点

1. **最小权限原则**：公开令牌只包含必要的资源声明，不携带用户信息
2. **分层验证**：分享流访问同时验证 JWT 和数据库记录，提供双重保障
3. **滑动会话**：自动续期机制提升用户体验
4. **密钥加密存储**：JWT 签名密钥在数据库中加密存储

### 8.2 潜在风险

1. **无令牌黑名单**：用户登出或密码变更后，已签发令牌在过期前仍然有效
2. **图片令牌永不过期**：`/share/img/*` 路径不检查令牌过期时间，一旦泄露永久有效
3. **分享令牌权限过大**：分享令牌中不包含用户信息，无法追溯访问来源
4. **转码令牌可复用**：转码令牌 48 小时内可重复使用，无使用次数限制

---

## 9. 关键代码路径索引

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| JWT 初始化 | `core/auth/auth.go` | 26-46 |
| Claims 定义 | `core/auth/claims.go` | 9-104 |
| 登录处理 | `server/auth.go` | 36-172 |
| JWT 验证中间件 | `server/auth.go` | 174-185 |
| 用户认证中间件 | `server/auth.go` | 260-272 |
| 令牌刷新中间件 | `server/auth.go` | 275-293 |
| 公开路由配置 | `server/public/public.go` | 38-60 |
| 公开图片处理 | `server/public/handle_images.go` | 17-88 |
| 公开流处理 | `server/public/handle_streams.go` | 17-97 |
| 分享页面处理 | `server/public/handle_shares.go` | 21-109 |
| 分享服务核心 | `core/share.go` | 34-52 |
| 请求上下文定义 | `model/request/request.go` | 1-127 |
| Native API 路由 | `server/nativeapi/native_api.go` | 56-98 |
| Subsonic 认证中间件 | `server/subsonic/middlewares.go` | 100-181 |
| 转码令牌处理 | `core/stream/token.go` | 15-148 |
