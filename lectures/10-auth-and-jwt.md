# L10 认证授权与 JWT

本篇定位：把「用户身份怎么证明、权限怎么判」讲清楚。覆盖密码存储、JWT 结构与安全校验、双令牌方案、传输安全、授权模型、中间件实现，并给出一个可运行的注册/登录/鉴权/刷新示例。
对应周次：W4。前置要求：熟悉 `net/http`、`context`、JSON 编解码与中间件写法。
本文只使用 Go 1.25 基线中确定存在的标准库 API；第三方库只给模块路径，版本以官方最新稳定版为准。

## 1. 认证与授权

| 概念 | 回答的问题 | 失败时的状态码 | 例子 |
| --- | --- | --- | --- |
| 认证 | 你是谁 | 401 Unauthorized | 校验密码、校验令牌签名 |
| 授权 | 你能做什么 | 403 Forbidden | 判断角色是否允许删除该资源 |

认证失败通常要客户端重新登录；授权失败重登也没用，是权限配置问题。

## 2. 会话与令牌对比

| 维度 | Cookie + 服务端 Session | JWT（无状态令牌） |
| --- | --- | --- |
| 状态存储 | 服务端或共享存储保存会话，客户端只持会话 ID | 状态全在令牌里，服务端不存 |
| 水平扩展 | 需要共享会话存储或粘性会话 | 任意实例都能独立校验 |
| 注销能力 | 删服务端会话即立刻失效 | 到期前默认一直有效，必须额外引入黑名单或版本号 |
| 跨域 | 受同源策略与 Cookie 作用域限制 | 放在请求头里，跨域更简单 |
| 移动端友好度 | 需要 Cookie 容器支持 | 天然适合移动端与第三方调用 |
| 被窃取后的处置 | 服务端可直接使会话失效 | 只能等过期，或维护吊销列表 |

需要「立即踢下线」的强控制场景，服务端会话更容易做对；需要跨域、跨端、无状态扩展的场景用 JWT，但要额外设计吊销机制。

## 3. 密码存储

原则：绝不存明文，也绝不用可逆加密；必须使用自适应哈希（代价可调），让离线暴力破解成本随硬件升级而上升；每个用户独立随机盐并与哈希一起存储，不使用全局固定盐；客户端预处理只是 TLS 之外的纵深防御，服务端必须再哈希后存储。

| 算法 | 模块路径 | 用途 |
| --- | --- | --- |
| bcrypt | `golang.org/x/crypto/bcrypt`，以官方最新稳定版为准 | 最常用的密码哈希；自带盐，输出含算法标识与参数 |
| argon2 | `golang.org/x/crypto/argon2`，以官方最新稳定版为准 | 内存硬哈希，抗 GPU/ASIC 更强；需自己管理盐与参数编码 |

构造函数签名、参数取值范围与错误语义以官方文档为准：<https://pkg.go.dev/golang.org/x/crypto/bcrypt>、<https://pkg.go.dev/golang.org/x/crypto/argon2>。

工作因子（cost）的选取：以生产机器为基准，把单次校验耗时调到「可接受但昂贵」的区间（通常几十到几百毫秒），并把该耗时计入登录接口的容量规划。升级路径是把参数写进存储的哈希字符串，登录校验成功后用当前参数重新计算并写回（rehash），因此不需要一次性重置全量密码。

比较哈希、签名、重置码等秘密值必须用 `crypto/subtle.ConstantTimeCompare`，不能直接用 `==`：普通比较的耗时随匹配前缀长度变化，会泄露可用于逐字节猜测的信息。该函数仅在两个切片长度相同时才返回相等，长度不同直接返回 0。

```go
// Hasher 抽象密码哈希；生产实现使用 golang.org/x/crypto/bcrypt 或 argon2。
type Hasher interface {
	Hash(plain string) (string, error)
	Verify(encoded, plain string) bool
}

// toyHasher 仅供示例运行，是单轮哈希而非自适应哈希，禁止用于生产。
type toyHasher struct{}

func saltAndHash(plain string, salt []byte) []byte {
	mac := hmac.New(sha256.New, salt)
	mac.Write([]byte(plain))
	return mac.Sum(nil)
}

func (toyHasher) Hash(plain string) (string, error) {
	salt := make([]byte, 16) // 每用户独立随机盐
	if _, err := rand.Read(salt); err != nil {
		return "", err
	}
	return hex.EncodeToString(salt) + "$" + hex.EncodeToString(saltAndHash(plain, salt)), nil
}

func (toyHasher) Verify(encoded, plain string) bool {
	p := strings.SplitN(encoded, "$", 2)
	if len(p) != 2 {
		return false
	}
	salt, err1 := hex.DecodeString(p[0])
	want, err2 := hex.DecodeString(p[1])
	if err1 != nil || err2 != nil {
		return false
	}
	return subtle.ConstantTimeCompare(saltAndHash(plain, salt), want) == 1
}
```

## 4. JWT 结构与安全要点

### 4.1 三段结构

JWT 是 `header.payload.signature` 三段用点号连接：header 与 payload 是 Base64URL 编码的 JSON，signature 是对前两段的签名。**payload 只是编码，不是加密**——任何拿到令牌的人都能读出内容，所以不要放手机号、身份证号、密钥等敏感信息。

### 4.2 alg 与算法混淆

| 要求 | 原因 |
| --- | --- |
| 固定允许的算法，按服务端配置选择密钥与校验方式 | 绝不信任 header 里的 `alg` 做密钥选择 |
| 显式拒绝 `none` | 否则无签名令牌会被接受 |
| `kid` 只在服务端白名单里查密钥 | 避免被用于拼接文件路径或 SQL |

算法混淆的典型场景：服务端原本用非对称算法（公钥验签），攻击者把 `alg` 改成对称算法，并用公开的**公钥**当 HMAC 密钥签名；若实现按 header 的 `alg` 选算法，验签就会通过。

### 4.3 标准声明语义

| 声明 | 含义 | 要求 |
| --- | --- | --- |
| `exp` | 过期时间 | 必须校验；没有 `exp` 等于永不过期 |
| `iat` | 签发时间 | 用于审计与「改密码后旧令牌失效」类逻辑 |
| `nbf` | 生效时间 | 校验，并配合时钟偏移容忍 |
| `iss` | 签发方 | 校验等于本服务标识，避免跨服务令牌混用 |
| `aud` | 接收方 | 校验是当前 API，防止其他系统的令牌被拿来用 |
| `jti` | 令牌唯一 ID | 吊销与刷新重放检测的关键字段 |

时钟偏移：多实例部署时机器时间不可能完全一致，校验时允许数十秒偏移（`nbf` 往后放宽、`exp` 往前收紧），不要放宽到几分钟以上。

### 4.4 key rotation 与 kid

目标是不中断服务地换密钥：`kid` 标识用哪把密钥签名；服务端维护 `kid -> key` 映射，校验时按 `kid` 查表，查不到直接拒绝；新令牌用新密钥签发，旧密钥保留到旧令牌全部过期后再删除。轮换期必须能同时校验两把密钥，否则会出现全量用户被强制登出的窗口。

### 4.5 库与教学实现

通用实现是 `github.com/golang-jwt/jwt/v5`，以官方最新稳定版为准（<https://pkg.go.dev/github.com/golang-jwt/jwt/v5>）。库只解决编码与验签，算法白名单、`exp`/`aud`/`iss` 校验与吊销策略仍要业务侧配置正确。下面的 HS256 实现用于看清三段结构与校验顺序，生产请替换为上面的库：

```go
var errInvalid = errors.New("invalid token")

type Claims struct {
	Subject   string `json:"sub"`
	Issuer    string `json:"iss"`
	Audience  string `json:"aud"`
	Role      string `json:"role"`
	TokenType string `json:"typ"`
	JTI       string `json:"jti"`
	NotBefore int64  `json:"nbf"`
	ExpiresAt int64  `json:"exp"`
}

type signer struct {
	keys     map[string][]byte // kid -> key，轮换期同时保留两把
	activeID string
	issuer   string
	audience string
}

func b64(b []byte) string { return base64.RawURLEncoding.EncodeToString(b) }

func (s *signer) mac(key []byte, body string) []byte {
	m := hmac.New(sha256.New, key)
	m.Write([]byte(body))
	return m.Sum(nil)
}

func (s *signer) Sign(c Claims) (string, error) {
	hb, _ := json.Marshal(map[string]string{"alg": "HS256", "typ": "JWT", "kid": s.activeID})
	pb, err := json.Marshal(c)
	if err != nil {
		return "", err
	}
	body := b64(hb) + "." + b64(pb)
	return body + "." + b64(s.mac(s.keys[s.activeID], body)), nil
}

func (s *signer) Parse(token string) (Claims, error) {
	var c Claims
	var hdr struct{ Alg, Kid string }
	parts := strings.Split(token, ".")
	if len(parts) != 3 {
		return c, errInvalid
	}
	hb, err := base64.RawURLEncoding.DecodeString(parts[0])
	if err != nil || json.Unmarshal(hb, &hdr) != nil || hdr.Alg != "HS256" {
		return c, errInvalid // 固定算法：拒绝 none 与算法混淆
	}
	key, ok := s.keys[hdr.Kid] // kid 只在服务端白名单中查
	sig, err := base64.RawURLEncoding.DecodeString(parts[2])
	if !ok || err != nil || subtle.ConstantTimeCompare(s.mac(key, parts[0]+"."+parts[1]), sig) != 1 {
		return c, errInvalid
	}
	pb, err := base64.RawURLEncoding.DecodeString(parts[1])
	if err != nil || json.Unmarshal(pb, &c) != nil {
		return c, errInvalid
	}
	now, skew := time.Now().Unix(), int64(30) // 时钟偏移容忍
	if c.ExpiresAt == 0 || now > c.ExpiresAt+skew || (c.NotBefore != 0 && now+skew < c.NotBefore) ||
		c.Issuer != s.issuer || c.Audience != s.audience {
		return c, errInvalid
	}
	return c, nil
}
```

## 5. Access Token + Refresh Token 双令牌

| 令牌 | 有效期 | 存放 | 用途 |
| --- | --- | --- | --- |
| Access Token | 短，分钟级（如 15 分钟） | 内存或 HttpOnly Cookie | 每次业务请求携带 |
| Refresh Token | 长，天级（如 7–30 天） | HttpOnly Cookie 或安全存储 | 只用于换新令牌 |

流程：access 过期 → 客户端带 refresh 调刷新接口 → 服务端校验 refresh（签名、`exp`、`jti` 是否在白名单、绑定信息是否一致）→ 签发新 access 与新 refresh → **旧 refresh 立即作废**（刷新轮换）。同一个 refresh 被用两次说明令牌已泄露（攻击者与用户各持一份），此时应作废该用户整条令牌链（把用户级令牌版本号加一）并强制重新登录。注销与踢下线有两种实现：维护 `jti` 黑名单（精确，需要存储与 TTL），或维护用户级版本号（简单，代价是该用户所有设备一起失效）。

## 6. 传输安全

| 维度 | `Authorization: Bearer <token>` | HttpOnly Cookie |
| --- | --- | --- |
| XSS 风险 | 令牌可被 JS 读取，有 XSS 即泄露 | JS 读不到 |
| CSRF 风险 | 不依赖 Cookie，天然免疫 | 需要 CSRF 防护（SameSite 或 token） |
| 跨域 | 简单直接 | 需要凭证模式与 CORS 配合 |
| 移动端 | 简单直接 | 需要 Cookie 容器 |
| 失效控制 | 依赖服务端黑名单 | 服务端可删除 Cookie |

选择 Bearer 就必须把 XSS 防护做扎实（输出转义、CSP、依赖审计）；选择 HttpOnly Cookie 就必须把 CSRF 防护做扎实。**Go 1.25 新增的 `net/http.CrossOriginProtection`** 基于 Fetch metadata 提供无需页面内嵌 token 的 CSRF 防护，用法见 <https://pkg.go.dev/net/http>。

密码传输：明文密码绝不允许经明文 HTTP 传输，**必须全站 HTTPS**，登录、注册、改密、刷新接口不能有例外，HTTP 只允许 301/308 跳转。在 HTTPS 之外，可让前端先做一次不可逆处理（如带服务端下发随机挑战值的哈希）再传输，作为纵深防御；但客户端哈希不能替代 HTTPS，请求仍可被重放，必须配合一次性挑战值或时间戳，且服务端仍要再哈希后存储。日志、链路追踪、错误上报都不得记录密码、令牌与 Cookie，请求体日志要做字段级脱敏。

## 7. 授权模型

| 模型 | 判定依据 | 适合 | 代价 |
| --- | --- | --- | --- |
| RBAC（角色-权限-资源） | 用户 → 角色 → 权限 → 资源 | 绝大多数后台与业务系统 | 角色数量增长后组合爆炸 |
| ABAC（属性） | 用户、资源、环境属性的策略表达式 | 细粒度动态规则 | 策略引擎与调试成本高 |
| ReBAC（关系） | 用户与资源的关系图 | 协作、分享、层级资源 | 需要关系存储与图查询 |

落地建议：从 RBAC 起步，把权限定义为动作（`article:read`、`article:delete`）而不是页面；出现「同角色但只能看自己数据」的需求时用资源级校验补足，而不是继续加角色。

鉴权必须两层：路由级（中间件校验令牌与所需权限）与资源级（业务逻辑校验这条数据是否属于当前身份）。IDOR 的典型漏洞是 `GET /api/orders/1001` 只校验登录、不校验归属，改 URL 里的 ID 就能读他人订单，属于水平越权；普通用户直接访问管理接口属于垂直越权。二者都要靠「按当前身份过滤查询」解决，而不是靠前端隐藏入口。令牌只放鉴权必需字段（用户 ID、角色、版本号、`jti`），默认拒绝、白名单放行。

## 8. 中间件实现

流程：从 `Authorization` 头按 `Bearer ` 前缀取出令牌 → 校验签名与声明 → 把身份写入 `context.Context` → 下游读取 → 未认证返回 401，已认证但无权限返回 403。关键是自定义 context key 类型，避免与字符串 key 冲突：

```go
type ctxKey struct{ name string }

var identityKey = ctxKey{"identity"} // 类型不同即不会与其他包的 key 冲突

type Identity struct {
	UserID string
	Role   string
	JTI    string
}

func identityFrom(ctx context.Context) (Identity, bool) {
	id, ok := ctx.Value(identityKey).(Identity)
	return id, ok
}
```

写入用 `context.WithValue(r.Context(), identityKey, Identity{...})`。令牌缺失、结构不合法、签名错误、已过期统一返回 401，且不要在响应里区分原因，避免给攻击者提供信息；授权失败返回 403，且不泄露资源是否存在等细节。

## 9. 常见实现漏洞清单

| 漏洞 | 现象 | 修复 |
| --- | --- | --- |
| 签名未校验 | 任意伪造令牌都能通过 | 必须完整验签 |
| 算法未固定 | 算法混淆攻击成功 | 固定算法白名单，显式拒绝 `none` |
| 无 `exp` 校验 | 令牌永久有效 | 校验 `exp` 并设合理有效期 |
| 不校验 `iss` / `aud` | 其他系统的令牌可用于本系统 | 逐项校验签发方与接收方 |
| 明文密码 | 库泄露即全量账号沦陷 | 自适应哈希 + 每用户独立盐 |
| 令牌写入日志 | 日志系统成为令牌仓库 | 全链路脱敏，禁止记录令牌与 Cookie |
| 宽泛 CORS | 任意站点可带凭证调用 | 只允许白名单来源，避免 `*` 与凭证同用 |
| 敏感信息放进 payload | Base64 解码即可读 | payload 只放标识与声明 |
| 只做路由级鉴权 | 换 ID 即可读他人数据（IDOR） | 增加资源级归属校验 |
| refresh 不轮换 | 泄露后长期可用 | 刷新即换新，旧令牌作废 |
| 注销只清客户端 | 服务端仍认旧令牌 | 黑名单或用户级版本号 |

## 10. 完整示例

下面是同一 `main` 包中的 HTTP 层，接在第 3 节的 `toyHasher` 与第 4.5 节的 `signer` 之后即可运行：注册、登录、受保护接口、刷新接口与 401/403 两级鉴权齐全。生产环境请把 `toyHasher` 换成 bcrypt/argon2、把 `dev-secret` 换成密钥管理、并放到 HTTPS 之后。

```go
package main

import (
	"context"
	"encoding/json"
	"log"
	"net/http"
	"strings"
	"sync"
	"time"
)

type user struct {
	Password string
	Role     string
}

type server struct {
	hasher     Hasher
	signer     *signer
	mu         sync.Mutex
	users      map[string]user
	refresh    map[string]string // refresh jti -> email（白名单，刷新即轮换）
	accessTTL  time.Duration
	refreshTTL time.Duration
}

func (s *server) issue(email, role, tokenType string, ttl time.Duration) (string, error) {
	now := time.Now()
	jti := make([]byte, 16)
	if _, err := rand.Read(jti); err != nil {
		return "", err
	}
	token, err := s.signer.Sign(Claims{
		Subject: email, Issuer: s.signer.issuer, Audience: s.signer.audience,
		Role: role, TokenType: tokenType, JTI: hex.EncodeToString(jti),
		NotBefore: now.Unix(), ExpiresAt: now.Add(ttl).Unix(),
	})
	if err != nil {
		return "", err
	}
	if tokenType == "refresh" { // 刷新令牌登记进白名单，供轮换与重放检测使用
		s.mu.Lock()
		s.refresh[hex.EncodeToString(jti)] = email
		s.mu.Unlock()
	}
	return token, nil
}

func decode(w http.ResponseWriter, r *http.Request, v any) bool {
	if err := json.NewDecoder(http.MaxBytesReader(w, r.Body, 1<<20)).Decode(v); err != nil {
		http.Error(w, "bad request", http.StatusBadRequest)
		return false
	}
	return true
}

func writeJSON(w http.ResponseWriter, code int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(code)
	_ = json.NewEncoder(w).Encode(v)
}

func bearer(h string) (string, bool) {
	const p = "Bearer "
	if len(h) <= len(p) || !strings.EqualFold(h[:len(p)], p) {
		return "", false
	}
	return strings.TrimSpace(h[len(p):]), true
}

type credentials struct {
	Email    string `json:"email"`
	Password string `json:"password"`
}

func (s *server) handleRegister(w http.ResponseWriter, r *http.Request) {
	var in credentials
	if !decode(w, r, &in) {
		return
	}
	if len(in.Email) < 3 || len(in.Password) < 12 {
		http.Error(w, "email or password too short", http.StatusBadRequest)
		return
	}
	hash, err := s.hasher.Hash(in.Password)
	if err != nil {
		http.Error(w, "hash failed", http.StatusInternalServerError)
		return
	}
	s.mu.Lock()
	defer s.mu.Unlock()
	if _, exists := s.users[in.Email]; exists {
		http.Error(w, "email already registered", http.StatusConflict) // 唯一约束冲突
		return
	}
	s.users[in.Email] = user{Password: hash, Role: "user"}
	writeJSON(w, http.StatusCreated, map[string]string{"email": in.Email})
}

func (s *server) handleLogin(w http.ResponseWriter, r *http.Request) {
	var in credentials
	if !decode(w, r, &in) {
		return
	}
	s.mu.Lock()
	u, ok := s.users[in.Email]
	s.mu.Unlock()
	if !ok || !s.hasher.Verify(u.Password, in.Password) {
		http.Error(w, "invalid credentials", http.StatusUnauthorized) // 不区分账号是否存在
		return
	}
	s.issuePair(w, in.Email, u.Role)
}

func (s *server) handleRefresh(w http.ResponseWriter, r *http.Request) {
	var body struct {
		RefreshToken string `json:"refresh_token"`
	}
	if !decode(w, r, &body) {
		return
	}
	claims, err := s.signer.Parse(body.RefreshToken)
	if err != nil || claims.TokenType != "refresh" {
		http.Error(w, "unauthorized", http.StatusUnauthorized)
		return
	}
	s.mu.Lock()
	_, live := s.refresh[claims.JTI]
	delete(s.refresh, claims.JTI) // 轮换：旧 refresh 立即作废
	s.mu.Unlock()
	if !live {
		http.Error(w, "refresh token reused", http.StatusUnauthorized) // 重放检测
		return
	}
	s.issuePair(w, claims.Subject, claims.Role)
}

// issuePair 签发 access 与 refresh；签发失败统一处理，避免在多个 handler 里重复。
func (s *server) issuePair(w http.ResponseWriter, email, role string) {
	access, err1 := s.issue(email, role, "access", s.accessTTL)
	refresh, err2 := s.issue(email, role, "refresh", s.refreshTTL)
	if err1 != nil || err2 != nil {
		http.Error(w, "sign failed", http.StatusInternalServerError)
		return
	}
	writeJSON(w, http.StatusOK, map[string]string{"access_token": access, "refresh_token": refresh})
}

// requireAuth 是路由级鉴权：未认证返回 401。
func (s *server) requireAuth(next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		raw, ok := bearer(r.Header.Get("Authorization"))
		if !ok {
			http.Error(w, "unauthorized", http.StatusUnauthorized)
			return
		}
		claims, err := s.signer.Parse(raw)
		if err != nil || claims.TokenType != "access" {
			http.Error(w, "unauthorized", http.StatusUnauthorized)
			return
		}
		ctx := context.WithValue(r.Context(), identityKey,
			Identity{UserID: claims.Subject, Role: claims.Role, JTI: claims.JTI})
		next(w, r.WithContext(ctx))
	}
}

// requireRole 是权限级鉴权：已认证但角色不符返回 403。
func (s *server) requireRole(role string, next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		if id, _ := identityFrom(r.Context()); id.Role != role {
			http.Error(w, "forbidden", http.StatusForbidden)
			return
		}
		next(w, r)
	}
}

func (s *server) handleMe(w http.ResponseWriter, r *http.Request) {
	id, _ := identityFrom(r.Context())
	writeJSON(w, http.StatusOK, map[string]string{"user_id": id.UserID, "role": id.Role})
}

func main() {
	s := &server{
		hasher: toyHasher{},
		signer: &signer{
			keys:     map[string][]byte{"k1": []byte("dev-secret")},
			activeID: "k1", issuer: "example-auth", audience: "example-api",
		},
		users:      map[string]user{},
		refresh:    map[string]string{},
		accessTTL:  15 * time.Minute,
		refreshTTL: 7 * 24 * time.Hour,
	}
	mux := http.NewServeMux()
	mux.HandleFunc("POST /register", s.handleRegister)
	mux.HandleFunc("POST /login", s.handleLogin)
	mux.HandleFunc("POST /refresh", s.handleRefresh)
	mux.HandleFunc("GET /me", s.requireAuth(s.handleMe))
	mux.HandleFunc("GET /admin/me", s.requireAuth(s.requireRole("admin", s.handleMe)))
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

自测：注册账号 → 登录拿两个令牌 → 带 `Authorization: Bearer <access_token>` 请求 `/me` 返回 200；缺少头部或误用 refresh 令牌返回 401；普通用户请求 `/admin/me` 返回 403；用旧 `refresh_token` 重放一次刷新，应返回 401。

## 11. 常见错误与反模式

| 错误写法 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| 密码明文或可逆加密存储 | 库泄露即全量沦陷 | 把可解密当安全 | bcrypt / argon2 自适应哈希 |
| 全局固定盐 | 彩虹表可批量命中 | 未使用每用户随机盐 | 每用户独立随机盐 |
| 用 `==` 比较哈希或签名 | 存在时序侧信道 | 比较耗时随前缀变化 | `crypto/subtle.ConstantTimeCompare` |
| 按 header 的 `alg` 选密钥 | 算法混淆攻击成功 | 信任了攻击者可控字段 | 服务端固定算法白名单 |
| 未拒绝 `none` | 无签名令牌被接受 | 未做显式拒绝 | 显式拒绝 `none` |
| 不校验 `exp` / `iss` / `aud` | 令牌永久有效或跨系统可用 | 只验了签名 | 逐项校验声明 |
| 在 payload 放敏感信息 | Base64 解码即可读 | 把编码当加密 | payload 只放标识与声明 |
| 令牌写入日志或追踪 | 日志成为凭据仓库 | 未做字段脱敏 | 全链路脱敏 |
| CORS 允许 `*` 且带凭证 | 任意站点可代发请求 | 配置过宽 | 来源白名单 + 凭证模式 |
| 只做路由级鉴权 | 改 URL 里的 ID 即可读他人数据 | 缺少资源级校验 | 查询按当前身份过滤 |
| refresh 不轮换、不作废 | 泄露后长期可用 | 无重放检测 | 刷新即换新并作废旧令牌 |
| 明文 HTTP 传密码 | 中间人直接拿到密码 | 未强制 HTTPS | 全站 HTTPS + 客户端预处理 |
| 登录失败区分「用户不存在」 | 可枚举账号 | 错误信息过细 | 统一返回无效凭据 |

## 12. 动手练习

1. 运行第 10 节的示例，分别用错误密码、过期 access、被复用一次的 refresh 调接口，确认返回 401；再用普通用户调 `/admin/me`，确认返回 403。
2. 把 `signer` 改成支持两把密钥（`k1`、`k2`），实现一次不中断服务的密钥轮换。
3. 把 `toyHasher` 换成 `golang.org/x/crypto/bcrypt`，标定 cost 参数并实现登录时按需 rehash。
4. 给示例加资源级校验：新增 `GET /users/{id}/profile`，要求只能读自己的资料，并写出对应的越权测试用例。
5. 用 `net/http.CrossOriginProtection` 给示例加上无 token 的 CSRF 防护，并说明它与 SameSite Cookie 的关系。

## 13. 自检清单

- [ ] 密码使用自适应哈希存储，每用户独立随机盐，绝无明文与可逆加密。
- [ ] 哈希参数按生产机器标定，并支持登录时按需 rehash。
- [ ] 秘密值比较使用 `crypto/subtle.ConstantTimeCompare`。
- [ ] JWT 校验固定算法、显式拒绝 `none`，并按 `kid` 在白名单中查密钥。
- [ ] `exp`、`nbf`、`iss`、`aud` 全部校验，且有时钟偏移容忍。
- [ ] access 短有效期，refresh 长有效期且轮换、可作废、可重放检测。
- [ ] 注销与踢下线有服务端吊销机制（黑名单或版本号）。
- [ ] 全站 HTTPS；密码与令牌不进入日志、追踪与错误上报。
- [ ] 选择 Bearer 时防护 XSS，选择 Cookie 时防护 CSRF（`CrossOriginProtection` 可用）。
- [ ] 鉴权分路由级与资源级两层，已覆盖 IDOR 与垂直越权用例。
- [ ] 未认证返回 401、已认证无权限返回 403，且响应不泄露额外信息。
- [ ] 令牌只携带鉴权必需字段，服务账号按最小权限授权。

## 14. 延伸阅读

- 密码哈希库：<https://pkg.go.dev/golang.org/x/crypto/bcrypt>、<https://pkg.go.dev/golang.org/x/crypto/argon2>
- 常量时间比较：<https://pkg.go.dev/crypto/subtle>
- JWT 库：<https://pkg.go.dev/github.com/golang-jwt/jwt/v5>
- Go 1.25 新增 `net/http.CrossOriginProtection`：<https://go.dev/doc/go1.25>、<https://pkg.go.dev/net/http>
- Go 1.22 起的 `ServeMux` 路由增强（示例中的 `POST /register` 写法）：<https://go.dev/doc/go1.22>
- 书籍：《数据密集型应用系统设计》，Martin Kleppmann，中国电力出版社
