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

Navidrome 有两条独立的登录用户认证路径，**各自实现过期检查**，过期判断并非集中在 `Authenticator` 一处完成。

### 3.1 Native API 认证链

**路由配置**: `server/nativeapi/native_api.go:63-95`

```
请求 → [全局中间件] JWTVerifier → [路由组中间件] Authenticator → JWTRefresher → UpdateLastAccessMiddleware → 业务处理器
```

#### 阶段 1: JWTVerifier（全局中间件，`server/auth.go:174-176`）

```go
func JWTVerifier(next http.Handler) http.Handler {
    return jwtauth.Verify(auth.TokenAuth, 
        tokenFromHeader,        // X-ND-Authorization: Bearer <token>
        jwtauth.TokenFromCookie,
        jwtauth.TokenFromQuery, // ?auth=<token>
    )(next)
}
```

**职责边界**：
- 从请求头、Cookie、查询参数提取令牌
- 验证令牌签名，将解析后的令牌存入上下文
- **不检查过期**（jwtauth 库会检测但不拒绝，仅将错误存入上下文）
- 无令牌或验证失败的请求继续执行，不终止请求链

#### 阶段 2: Authenticator（Native API 专属，`server/auth.go:260-272`）

```go
func Authenticator(ds model.DataStore) func(next http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx, err := authenticateRequest(ds, r,
                UsernameFromConfig,       // 开发模式自动登录
                UsernameFromToken,        // 从 JWT 提取
                UsernameFromExtAuthHeader // 反向代理认证
            )
            if err != nil {
                _ = rest.RespondWithError(w, http.StatusUnauthorized, "Not authenticated")
                return
            }
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

**过期检查点**：`UsernameFromToken()` → `jwtauth.FromContext()`

```go
func UsernameFromToken(r *http.Request) string {
    token, _, err := jwtauth.FromContext(r.Context())
    if err != nil || token == nil {
        return ""  // 令牌过期或无效，返回空字符串
    }
    sub, _ := token.Subject()
    return sub
}
```

**关键边界**：
- 这是 **Native API 的过期检查点**
- 如令牌已过期，`jwtauth.FromContext()` 返回错误 → `UsernameFromToken()` 返回空
- 无法获取有效用户名 → `authenticateRequest()` 返回 `ErrUnauthenticated` → 返回 401
- 此中间件**仅用于 Native API**，Subsonic API 不使用它

#### 阶段 3: JWTRefresher（Native API 专属，`server/auth.go:275-293`）

```go
func JWTRefresher(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()
        token, _, err := jwtauth.FromContext(ctx)
        if err != nil {
            next.ServeHTTP(w, r)  // 令牌有问题直接跳过
            return
        }
        newTokenString, err := auth.TouchToken(token)  // 延长过期时间
        if err == nil {
            w.Header().Set(consts.UIAuthorizationHeader, newTokenString)
        }
        next.ServeHTTP(w, r)
    })
}
```

**滑动会话机制**：
- 仅当令牌有效（未过期）时才刷新
- 新令牌通过响应头 `X-ND-Authorization` 返回
- 会话超时：默认 48 小时（`DefaultSessionTimeout`）

---

### 3.2 Subsonic API 认证链

**路由配置**: `server/subsonic/api.go:98-233`

```
请求 → [全局中间件] JWTVerifier → [路由组中间件] checkRequiredParameters → authenticate(Subsonic专属) → UpdateLastAccessMiddleware → 业务处理器
```

> **重要**：Subsonic API **不使用** Native API 的 `Authenticator` 和 `JWTRefresher`，它有自己独立的认证中间件。

#### 阶段 1: authenticate（Subsonic 专属，`server/subsonic/middlewares.go:100-156`）

Subsonic 的 `authenticate` 中间件有**两个独立分支**，过期检查逻辑不同：

##### 分支 A：内部/反向代理认证（`server/subsonic/middlewares.go:108-120`）

```go
username, isInternalAuth := fromInternalOrProxyAuth(r)
if username != "" {
    usr, err = ds.User(ctx).FindByUsername(username)
    // 只检查用户是否存在，不检查 JWT 过期
}
```

**过期检查**：❌ 无过期检查
- 内部认证（插件调用）和反向代理认证直接信任用户名
- 不涉及 JWT 令牌，因此不检查过期

##### 分支 B：Subsonic 标准认证（`server/subsonic/middlewares.go:121-145`）

```go
p := req.Params(r)
username, _ := p.String("u")
pass, _ := p.String("p")
token, _ := p.String("t")
salt, _ := p.String("s")
jwt, _ := p.String("jwt")

usr, err = ds.User(ctx).FindByUsernameWithPassword(username)
if err == nil {
    err = validateCredentials(usr, pass, token, salt, jwt)  // 验证凭证
}
```

#### 阶段 2: validateCredentials（JWT 过期检查点，`server/subsonic/middlewares.go:158-181`）

```go
func validateCredentials(user *model.User, pass, token, salt, jwt string) error {
    valid := false
    switch {
    case jwt != "":
        // Subsonic JWT 认证的过期检查点
        claims, err := auth.Validate(jwt)  // ✅ 这里检查过期！
        valid = err == nil && claims.Subject == user.UserName
    case pass != "":
        // 密码认证，无过期检查
        valid = pass == user.Password
    case token != "":
        // MD5 令牌认证，无过期检查
        t := fmt.Sprintf("%x", md5.Sum([]byte(user.Password+salt)))
        valid = t == token
    }
    if !valid {
        return model.ErrInvalidAuth
    }
    return nil
}
```

**过期检查点**：`auth.Validate(jwt)` → `jwtauth.VerifyToken()`

```go
// core/auth/auth.go:86-92
func Validate(tokenStr string) (Claims, error) {
    token, err := jwtauth.VerifyToken(TokenAuth, tokenStr)  // 验证签名+过期
    if err != nil {
        return Claims{}, err
    }
    return ClaimsFromToken(token), nil
}
```

**关键边界**：
- 这是 **Subsonic API 的 JWT 过期检查点**
- `auth.Validate()` 会完整验证 JWT 的签名和过期时间
- 如令牌过期，返回错误 → `validateCredentials` 失败 → 返回 Subsonic 认证错误（代码 40）
- 密码认证（`p` 参数）和 MD5 令牌认证（`t`/`s` 参数）**不检查过期**，只要密码正确即可通过

---

### 3.3 两条认证路径对比

| 特性 | Native API | Subsonic API |
|------|-----------|-------------|
| 认证中间件 | `Authenticator`（`server/auth.go`） | `authenticate`（`server/subsonic/middlewares.go`） |
| JWT 过期检查点 | `UsernameFromToken()` → `jwtauth.FromContext()` | `validateCredentials()` → `auth.Validate(jwt)` |
| 密码认证过期检查 | ❌ 不支持密码认证 | ❌ 密码认证不检查过期 |
| MD5 令牌认证 | ❌ 不支持 | ❌ 不检查过期 |
| 滑动会话刷新 | ✅ `JWTRefresher` 自动刷新 | ❌ 无令牌刷新机制 |
| 过期后 HTTP 状态 | 401 Unauthorized | 200 OK（Subsonic 错误码 40） |
| 全局 `JWTVerifier` 作用 | 预解析令牌，存入上下文 | 预解析令牌，但 Subsonic 不使用上下文的令牌，而是重新从 `jwt` 参数解析 |

> **重要结论**：过期检查**不是集中在 `Authenticator` 一处完成**。Native API 和 Subsonic API 各自有独立的过期检查实现，检查位置和方式完全不同。

---

## 4. 公开访问路径分析

### 4.1 公开路由配置

**核心文件**: `server/public/public.go:38-60`

```go
func (pub *Router) routes() http.Handler {
    r := chi.NewRouter()
    r.Group(func(r chi.Router) {
        r.Use(server.URLParamsMiddleware)
        
        // 图片接口不受 EnableSharing 控制，始终可用
        r.Group(func(r chi.Router) {
            r.Use(server.ThrottleBacklog(...))
            r.HandleFunc("/img/{id}", pub.handleImages)
        })
        
        // 以下接口受 EnableSharing 控制
        if conf.Server.EnableSharing {
            r.HandleFunc("/s/{id}", pub.handleStream)
            // 下载接口同时受 EnableSharing 和 EnableDownloads 控制
            if conf.Server.EnableDownloads {
                r.HandleFunc("/d/{id}", pub.handleDownloads)
            }
            r.HandleFunc("/{id}/m3u", pub.handleM3U)
            r.HandleFunc("/{id}", pub.handleShares)
            r.HandleFunc("/", pub.handleShares)
            r.Handle("/*", pub.assetsHandler)
        }
    })
    return r
}
```

> **重要**: 公开路由 **不经过** `Authenticator`、`JWTRefresher` 和 `UpdateLastAccessMiddleware`，但会经过全局中间件链。

### 4.2 全局中间件应用分析

**路由挂载机制**: `server/server.go:51-57`

```go
func (s *Server) MountRouter(description, urlPath string, subRouter http.Handler) {
    urlPath = path.Join(conf.Server.BasePath, urlPath)
    log.Info(fmt.Sprintf("Mounting %s routes", description), "path", urlPath)
    s.router.Group(func(r chi.Router) {
        r.Mount(urlPath, subRouter)
    })
}
```

**全局中间件链** (`server/server.go:172-185`):
```go
defaultMiddlewares := chi.Middlewares{
    secureMiddleware(),           // 安全头
    corsHandler(),                // CORS 配置
    middleware.RequestID,         // 请求ID
    realIPMiddleware,             // 真实IP解析
    middleware.Recoverer,         // Panic 恢复
    middleware.Heartbeat("/ping"),// 健康检查
    robotsTXT(ui.BuildAssets()),  // robots.txt
    serverAddressMiddleware,      // 服务器地址重写
    clientUniqueIDMiddleware,     // 客户端唯一标识
    compressMiddleware(),         // 响应压缩
    loggerInjector,               // 日志注入
    JWTVerifier,                  // JWT 令牌验证
}
```

**关键结论**:
- ✅ **所有公开路由都经过全局中间件链**，包括 `JWTVerifier`
- ✅ 公开请求会被分配 RequestID、记录日志、解析真实 IP
- ✅ 公开请求支持 CORS，响应会被压缩
- ✅ `clientUniqueIDMiddleware` 会为公开请求设置 `ClientUniqueId`（从 `X-ND-Client-Unique-Id` 头或 Cookie）
- ❌ 但公开路由**不会**在 `Authenticator` 中验证用户身份
- ❌ `JWTVerifier` 是可选验证，不强制要求令牌存在（无令牌的请求继续执行）
- ❌ 公开路由不会经过 `JWTRefresher` 和 `UpdateLastAccessMiddleware`

### 4.3 公开下载路径配置分析

**路由注册**: `server/public/public.go:50-52`
```go
if conf.Server.EnableSharing {
    r.HandleFunc("/s/{id}", pub.handleStream)
    if conf.Server.EnableDownloads {
        r.HandleFunc("/d/{id}", pub.handleDownloads)  // 下载路由
    }
    r.HandleFunc("/{id}/m3u", pub.handleM3U)
    r.HandleFunc("/{id}", pub.handleShares)
}
```

**下载生效条件矩阵**:

| `EnableSharing` | `EnableDownloads` | `share.Downloadable` | 下载路径状态 |
|-----------------|-------------------|----------------------|-------------|
| `false` | 任意 | 任意 | ❌ 路由未注册 |
| `true` | `false` | 任意 | ❌ 路由未注册 |
| `true` | `true` | `false` | ✅ 路由存在，但请求返回 403 Forbidden |
| `true` | `true` | `true` | ✅ 完全可用 |

**下载权限检查**: `core/archiver.go:94-104`
```go
func (a *archiver) ZipShare(ctx context.Context, id string, out io.Writer) error {
    s, err := a.shares.Load(ctx, id)  // 检查分享是否存在且未过期
    if err != nil {
        return err
    }
    if !s.Downloadable {  // 检查分享的下载标记
        return model.ErrNotAuthorized
    }
    // ... 执行打包下载
}
```

**错误处理**: `server/public/handle_shares.go:66-81`
```go
func checkShareError(ctx context.Context, w http.ResponseWriter, err error, id string) {
    switch {
    case errors.Is(err, model.ErrNotAuthorized):
        log.Error(ctx, "Share is not downloadable", "id", id, err)
        http.Error(w, "This share is not downloadable", http.StatusForbidden)
    // ... 其他错误处理
    }
}
```

### 4.4 公开图片访问 (`/share/img/{id}`)

**核心文件**: `server/public/handle_images.go:17-88`

#### 4.4.1 令牌解析的回退逻辑

```go
func decodeArtworkID(tokenString string) (model.ArtworkID, error) {
    token, err := auth.TokenAuth.Decode(tokenString)
    if err != nil {
        return model.ArtworkID{}, err
    }
    if token == nil {
        return model.ArtworkID{}, errors.New("unauthorized")
    }
    c := auth.ClaimsFromToken(token)
    if c.ID == "" {
        return model.ArtworkID{}, errors.New("required claim \"id\" not found")
    }
    
    // 第一次尝试：直接解析为 ArtworkID（如 "al-123"、"ar-456"）
    artID, err := model.ParseArtworkID(c.ID)
    if err == nil {
        return artID, nil
    }
    
    // 🔴 回退逻辑：如果解析失败，尝试作为媒体文件ID处理
    // Try to default to mediafile artworkId (if used with a mediafileShare token)
    return model.ParseArtworkID("mf-" + c.ID)
}
```

**回退逻辑说明**：
- **第一次尝试**：将 `c.ID` 直接解析为 ArtworkID（格式如 `al-{专辑ID}`、`ar-{艺术家ID}`、`mf-{媒体文件ID}`）
- **回退尝试**：如果解析失败，自动添加 `mf-` 前缀，尝试作为媒体文件的封面ID解析
- **设计目的**：兼容分享流令牌（mediafileShare token），使分享流中的媒体文件ID可以直接用于访问其封面

> **重要边界**：回退逻辑意味着**任何包含有效 `ID` 声明的 JWT 令牌都可能被用于访问图片**，即使该令牌原本是为其他目的（如分享流）签发的。

---

#### 4.4.2 图片令牌的生成方式

**核心文件**: `core/publicurl/publicurl.go:18-28`

```go
func ImageURL(req *http.Request, artID model.ArtworkID, size int) string {
    token, _ := auth.CreatePublicToken(auth.Claims{ID: artID.String()})
    uri := path.Join(consts.URLPathPublicImages, token)
    // ... 构建完整 URL
}
```

**生成方式**：使用 `auth.CreatePublicToken()` 生成

```go
// core/auth/auth.go:48-52
func CreatePublicToken(claims Claims) (string, error) {
    claims.Issuer = consts.JWTIssuer
    _, token, err := TokenAuth.Encode(claims.ToMap())
    return token, err  // ❌ 不设置 ExpiresAt！
}
```

**令牌特征**：
- ✅ 设置 `Issuer`（签发者）
- ✅ 包含 `ID` 声明（资源标识）
- ❌ **不设置 `ExpiresAt`（过期时间）**
- ❌ 不设置 `IssuedAt`（签发时间）

---

#### 4.4.3 过期检查方式

**验证方法**：使用 `auth.TokenAuth.Decode()` 而非 `auth.Validate()`

```go
// 图片路径：仅解码，不验证过期
token, err := auth.TokenAuth.Decode(tokenString)

// 对比：分享流路径：完整验证（包括过期）
c, err := auth.Validate(tokenString)
```

**关键区别**：

| 方法 | 验证签名 | 检查过期 | 适用场景 |
|------|---------|---------|---------|
| `TokenAuth.Decode()` | ✅ | ❌ | 公开图片 |
| `auth.Validate()` | ✅ | ✅ | 分享流、下载、API认证 |

> **设计意图**：图片资源设置了超长缓存（`Cache-Control: public, max-age=315360000`，约10年），因此不检查过期，避免缓存的图片URL失效。

---

#### 4.4.4 访问控制边界

**权限矩阵**：

| 条件 | 结果 | 说明 |
|------|------|------|
| 签名无效 | ❌ 400 Bad Request | 令牌被篡改 |
| `ID` 声明缺失 | ❌ 400 Bad Request | 令牌格式错误 |
| 资源不存在 | ❌ 404 Not Found | 艺术品ID无效 |
| 令牌已过期（如有exp） | ✅ 正常访问 | Decode不检查过期 |
| 签名有效 + 资源存在 | ✅ 正常访问 | 永久有效 |

**访问控制边界总结**：
1. **无过期限制**：只要签名有效，永久可访问
2. **无分享关联**：图片令牌独立于分享记录，分享过期后图片仍可访问
3. **无用户关联**：不需要登录，不验证用户身份
4. **回退风险**：分享流令牌可被重用于访问对应媒体文件的封面

---

#### 4.4.5 与其他公开路由的异同对照

| 特性 | 公开图片 (`/img/{id}`) | 分享流 (`/s/{id}`) | 分享页面 (`/{id}`) | 公开下载 (`/d/{id}`) |
|------|----------------------|-------------------|-------------------|-------------------|
| 路由注册条件 | 始终可用（不受 `EnableSharing` 影响） | `EnableSharing=true` | `EnableSharing=true` | `EnableSharing && EnableDownloads` |
| 令牌验证方法 | `TokenAuth.Decode()` | `auth.Validate()` | 无JWT（分享ID） | 无JWT（分享ID） |
| 过期检查 | ❌ 不检查 | ✅ 检查（JWT过期+分享过期） | ✅ 检查（分享记录过期） | ✅ 检查（分享记录过期） |
| 分享记录检查 | ❌ 不检查 | ✅ 检查分享是否存在/过期 | ✅ 检查分享是否存在/过期 | ✅ 检查分享是否存在/过期 |
| 令牌回退逻辑 | ✅ 有（`mf-` 前缀回退） | ❌ 无 | ❌ 无 | ❌ 无 |
| 缓存策略 | 超长缓存（10年） | 不缓存 | 不缓存 | 不缓存 |
| 可被其他令牌复用 | ✅ 分享流令牌可访问封面 | ❌ 仅流令牌可用 | ❌ 仅分享ID可用 | ❌ 仅分享ID可用 |

---

### 4.5 公开流媒体访问 (`/share/s/{id}`)

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

### 4.6 分享页面访问 (`/share/{id}`)

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
| `ClientUniqueId` | ✅ 可选 | ✅ 可选 | `clientUniqueIDMiddleware`（全局中间件） |
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

### 7.1 过期机制（按访问路径分类）

#### Native API 路径

| 令牌类型 | 过期检查点 | 过期后行为 |
|---------|-----------|-----------|
| 会话令牌（JWT） | `Authenticator` → `UsernameFromToken` → `jwtauth.FromContext()` | 返回 401 Unauthorized |

#### Subsonic API 路径

| 认证方式 | 过期检查点 | 过期后行为 |
|---------|-----------|-----------|
| JWT 认证（`jwt` 参数） | `validateCredentials()` → `auth.Validate(jwt)` | 返回 200 OK + Subsonic 错误码 40 |
| 密码认证（`p` 参数） | ❌ 无过期检查 | 只要密码正确就通过 |
| MD5 令牌认证（`t`/`s` 参数） | ❌ 无过期检查 | 只要哈希匹配就通过 |
| 内部/反向代理认证 | ❌ 无过期检查 | 直接信任用户名 |

#### 公开路由路径

| 令牌类型 | 过期检查点 | 过期后行为 |
|---------|-----------|-----------|
| 分享流令牌 | `auth.Validate()` + 分享记录检查 | 返回 400 Bad Request 或 410 Gone |
| 分享页面 | `share.Load()` 中检查 `ExpiresAt` | 返回 410 Gone |
| 转码令牌 | `parseTranscodeParams()` | 返回错误，需要重新获取 |
| 公开图片 | ❌ 不检查过期 | 永久可访问（只要签名有效） |

> **重要澄清**:
> 1. 会话令牌的过期检查**不发生在 `JWTVerifier`**。`JWTVerifier` 仅将过期错误存入上下文但不拒绝请求
> 2. 过期检查**不是集中在 `Authenticator` 一处完成**：
>    - Native API: `Authenticator` → `UsernameFromToken`
>    - Subsonic API: `authenticate` → `validateCredentials` → `auth.Validate(jwt)`
>    - 公开路由: 各处理器自行调用 `auth.Validate()` 或检查分享记录

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

---

## 10. 统一结论与口径澄清

本节总结所有关键权限边界，确保理解一致，无口径冲突。

### 10.1 公开下载路由生效条件（三层控制）

**代码依据**: `server/public/public.go:48-57`

| 检查层级 | 检查位置 | 生效条件 | 不满足时行为 |
|---------|---------|---------|-------------|
| 1 | 路由注册 | `EnableSharing = true` | ❌ 路由不存在，返回 404 |
| 2 | 路由注册 | `EnableDownloads = true` | ❌ 路由不存在，返回 404 |
| 3 | 业务逻辑 | `share.Downloadable = true` | ✅ 路由存在，但返回 403 Forbidden |

> **统一结论**: `/share/d/{id}` 路由的**挂载前提**是 `EnableSharing && EnableDownloads` 同时为 `true`。即使路由挂载，下载请求仍需通过 `share.Downloadable` 的业务层检查。

### 10.2 三条访问路径的过期检查边界

#### 10.2.1 全局中间件 JWTVerifier 的职责

**代码依据**: `server/auth.go:174-176`

| 职责 | 是否执行 | 说明 |
|------|---------|------|
| 提取令牌 | ✅ | 从请求头、Cookie、查询参数提取 |
| 验证签名 | ✅ | 验证 JWT 签名有效性 |
| 检查过期 | ⚠️ | jwtauth 库内部检测，但仅将错误存入上下文，**不拒绝请求** |
| 无令牌处理 | ✅ | 直接通过，不做任何处理 |

> **结论**: `JWTVerifier` 是**全局可选验证中间件**，不做访问控制决策，仅为后续中间件预解析令牌。

---

#### 10.2.2 Native API 路径

**代码依据**: `server/auth.go:260-272`, `server/auth.go:187-198`

```
JWTVerifier（预解析）→ Authenticator（过期检查）→ JWTRefresher（刷新令牌）→ 业务处理器
```

| 中间件 | 职责 | 过期检查方式 | 拒绝条件 |
|-------|------|-------------|---------|
| `Authenticator` | 强制认证 | `UsernameFromToken()` → `jwtauth.FromContext()` | 令牌过期或无效 → 返回 401 |
| `JWTRefresher` | 滑动会话 | 仅对有效令牌刷新过期时间 | 令牌过期则跳过刷新 |

> **结论**: Native API 的过期检查发生在 `Authenticator` 中间件，通过 `jwtauth.FromContext()` 间接检测过期。

---

#### 10.2.3 Subsonic API 路径

**代码依据**: `server/subsonic/middlewares.go:100-181`

```
JWTVerifier（预解析但不使用）→ checkRequiredParameters → authenticate（分支处理）→ 业务处理器
```

Subsonic API **不使用** `Authenticator`，有自己独立的认证逻辑，分两个分支：

##### 分支 A：内部/反向代理认证
- 无 JWT 令牌，直接信任用户名
- ❌ **无过期检查**

##### 分支 B：Subsonic 标准认证
- 进一步分为三种凭证验证方式：

| 认证方式 | 过期检查点 | 拒绝条件 |
|---------|-----------|---------|
| JWT 认证（`jwt` 参数） | `validateCredentials()` → `auth.Validate(jwt)` | 令牌过期或用户名不匹配 → Subsonic 错误码 40 |
| 密码认证（`p` 参数） | ❌ 无过期检查 | 密码错误 → Subsonic 错误码 40 |
| MD5 令牌认证（`t`/`s`） | ❌ 无过期检查 | 哈希不匹配 → Subsonic 错误码 40 |

> **重要**: Subsonic API 会**重新从 `jwt` 查询参数解析令牌**，不使用全局 `JWTVerifier` 存入上下文的令牌。

---

#### 10.2.4 公开路由路径

**代码依据**: `server/public/handle_streams.go:84`, `server/public/handle_shares.go`, `server/public/handle_images.go:70-88`

```
JWTVerifier（预解析但不使用）→ URLParamsMiddleware → 各处理器自行验证
```

公开路由没有统一的认证中间件，各处理器自行验证：

| 处理器 | 过期检查方式 | 拒绝条件 |
|-------|-------------|---------|
| 分享流（`/s/{id}`） | `auth.Validate(tokenString)` | 令牌过期 → 400 Bad Request |
| 分享页面（`/{id}`） | `share.Load()` 检查 `ExpiresAt` | 分享过期 → 410 Gone |
| 转码流 | `parseTranscodeParams()` | 令牌过期 → 返回错误 |
| 公开图片（`/img/{id}`） | ❌ 不检查过期 | 只要签名有效就永久可访问 |

> **特别注意**：公开图片路径的 `decodeArtworkID` 函数存在**回退逻辑**：
> - 先尝试直接解析 `c.ID` 为 ArtworkID
> - 如失败，自动添加 `mf-` 前缀重试
> - 这意味着分享流令牌（包含媒体文件ID）可被重用于访问对应媒体文件的封面

---

### 10.3 过期检查点汇总（全路径）

| 访问路径 | 认证方式 | 过期检查点 | 过期后 HTTP 状态 |
|---------|---------|-----------|-----------------|
| Native API | JWT | `Authenticator` → `UsernameFromToken` | 401 Unauthorized |
| Subsonic API | JWT | `authenticate` → `validateCredentials` → `auth.Validate` | 200 OK（错误码 40） |
| Subsonic API | 密码/MD5 | ❌ 无过期检查 | N/A |
| 公开分享流 | JWT | `handleStream` → `auth.Validate` | 400 Bad Request |
| 公开分享页面 | 分享记录 | `handleShares` → `share.Load` | 410 Gone |
| 公开图片 | 仅签名 | ❌ 不检查过期 | N/A |

> **核心结论**: 过期检查**不是集中在 `Authenticator` 一处完成**。Navidrome 至少有 5 个独立的过期检查点，分布在不同的代码路径中。

### 10.4 公开路由与全局中间件关系

**代码依据**: `server/server.go:148-185`, `server/public/public.go:38-60`

| 中间件类型 | 是否应用于公开路由 | 具体说明 |
|-----------|-------------------|---------|
| 全局中间件 | ✅ 全部应用 | `JWTVerifier`、`realIPMiddleware`、`clientUniqueIDMiddleware`、日志、压缩、CORS 等 |
| 认证中间件 | ❌ 不应用 | `Authenticator`、`JWTRefresher`、`UpdateLastAccessMiddleware` |

> **统一结论**: 公开路由经过完整的全局中间件链，因此公开请求上下文中包含 `RequestID`、`RealIP`、`ClientUniqueId` 等字段。但公开路由不经过认证中间件，因此不强制用户登录。

### 10.5 公开图片路径的权限边界总结

**代码依据**: `server/public/handle_images.go:70-88`, `core/publicurl/publicurl.go:18-28`

#### 10.5.1 令牌解析的回退逻辑

公开图片路径是唯一存在**资源标识回退逻辑**的公开路由：

```go
// 第一次尝试：直接解析为 ArtworkID
artID, err := model.ParseArtworkID(c.ID)
if err == nil {
    return artID, nil
}

// 回退尝试：添加 mf- 前缀后重试
return model.ParseArtworkID("mf-" + c.ID)
```

**回退逻辑的影响：
- ✅ 设计目的：使分享流令牌（mediafileShare token）可直接用于访问媒体文件的封面
- ⚠️ 安全边界：任何包含有效 `ID` 声明的 JWT 令牌（无论签发目的）都可能被用于访问图片
- ❗ 注意：这是有意的设计选择，而非漏洞

---

#### 10.5.2 生成方式与过期检查的组合边界

| 环节 | 行为 | 访问控制影响 |
|------|------|-------------|
| 令牌生成 | `CreatePublicToken()` **不设置过期时间 | 令牌本身无过期限制 |
| 令牌验证 | `TokenAuth.Decode()` **不检查过期 | 即使令牌包含 `exp` 声明也不会被拒绝 |
| 缓存策略 | `Cache-Control: public, max-age=315360000`（10年） | 图片URL可被浏览器和CDN长期缓存 |

**组合后的访问控制边界**：
> 🔴 **图片令牌一经签发，**永久有效**，无法通过过期机制撤销。只能通过删除资源本身或重新生成JWT密钥来撤销访问。

---

#### 10.5.3 与其他公开路由的核心差异

| 维度 | 公开图片 | 分享流/下载 | 分享页面 |
|------|---------|------------|---------|
| 路由依赖 | 无（始终可用） | `EnableSharing` | `EnableSharing` |
| 令牌类型 | 专用图片令牌 | 专用分享流令牌 | 无JWT（分享ID） |
| 过期机制 | ❌ 无 | ✅ JWT过期 + 分享过期 | ✅ 分享记录过期 |
| 分享关联 | ❌ 独立于分享 | ✅ 依赖分享记录 | ✅ 依赖分享记录 |
| 撤销方式 | 只能删除资源/换密钥 | 删除/过期分享 | 删除/过期分享 |
| 令牌复用 | ✅ 分享流令牌可访问封面 | ❌ 仅专用令牌 | ❌ 仅分享ID |

### 10.6 公开请求中 ClientUniqueId 的真实来源

**代码依据**: `server/middlewares.go:92-118`

| 优先级 | 来源 | 说明 |
|-------|------|------|
| 1 | `X-ND-Client-Unique-Id` 请求头 | 客户端可主动设置 |
| 2 | 同名 Cookie | 如无请求头则尝试从 Cookie 读取 |
| 3 | 空字符串 | 两者都没有则为空 |

> **统一结论**: `ClientUniqueId` 是**客户端自声明的标识**，由全局中间件 `clientUniqueIDMiddleware` 设置，不依赖认证状态，公开请求中同样可用。

