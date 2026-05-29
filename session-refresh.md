# 登录会话刷新与自托管后台对接流程详解

## 一、整体架构概览

Hoppscotch 自托管版本采用 **双 Token 机制（Access Token + Refresh Token）实现会话保持，配合 Cookie 存储 Token，前端通过 urql 的 authExchange 实现自动 Token 刷新。

---

## 二、核心概念

### 2.1 Token 类型与有效期

| Token 类型 | 存储位置 | HttpOnly | 默认有效期 | 用途 |
|---------|---------|----------|-----------|------|
| **Access Token** | Cookie | ✅ 是 | 1 天 | API 请求认证 |
| **Refresh Token** | Cookie | ✅ 是 | 7 天 | 续签 Access Token |

> 两个 Token 均为 HttpOnly Cookie，前端 JavaScript 无法直接访问，保障安全性。

### 2.2 关键文件位置

**后端 (hoppscotch-backend)：
- `packages/hoppscotch-backend/src/auth/auth.service.ts
- `packages/hoppscotch-backend/src/auth/auth.controller.ts
- `packages/hoppscotch-backend/src/auth/strategies/rt-jwt.strategy.ts
- `packages/hoppscotch-backend/src/auth/helper.ts

**前端 (hoppscotch-selfhost-web)：
- `packages/hoppscotch-selfhost-web/src/platform/auth/web/index.ts
- `packages/hoppscotch-common/src/helpers/backend/GQLClient.ts
- `packages/hoppscotch-common/src/helpers/retryAuthGuard.ts

---

## 三、后端 Token 刷新流程

### 3.1 刷新端点

**文件**: `packages/hoppscotch-backend/src/auth/auth.controller.ts:87-100`

```typescript
@Get('refresh')
@UseGuards(RTJwtAuthGuard)
async refresh(
  @GqlUser() user: AuthUser,
  @RTCookie() refresh_token: string,
  @Res() res,
) {
  const newTokenPair = await this.authService.refreshAuthTokens(
    refresh_token,
    user,
  );
  if (E.isLeft(newTokenPair)) throwHTTPErr(newTokenPair.left);
  authCookieHandler(res, newTokenPair.right, false, null, this.configService);
}
```

**流程说明**：

1. **请求进入**：前端调用 `/api/v1/auth/refresh
2. **Guard 验证**：`RTJwtAuthGuard 通过 `rt-jwt.strategy.ts` 验证 Refresh Token
3. **服务处理**：`authService.refreshAuthTokens()` 验证并生成新 Token
4. **Cookie 更新**：`authCookieHandler()` 将新 Token 写入 Cookie

### 3.2 Refresh Token 验证策略

**文件**: `packages/hoppscotch-backend/src/auth/strategies/rt-jwt.strategy.ts

```typescript
@Injectable()
export class RTJwtStrategy extends PassportStrategy(Strategy, 'jwt-refresh') {
  constructor(
    private usersService: UserService,
    private configService: ConfigService,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromExtractors([
        (request: Request) => {
          const RTCookie = request.cookies?.['refresh_token'];
          if (!RTCookie) {
            throw new ForbiddenException(COOKIES_NOT_FOUND);
          }
          return RTCookie;
        },
      ]),
      secretOrKey: configService.get('INFRA.JWT_SECRET'),
    });
  }

  async validate(payload: RefreshTokenPayload) {
    // 1. 验证 JWT 签名有效性
    // 2. 从数据库查询用户信息
    const user = await this.usersService.findUserById(payload.sub);
    return user.value;
  }
}
```

**关键点**：
- 从 Cookie 中提取 `refresh_token`
- 验证 JWT 签名和过期时间
- 查询用户并附加到请求上下文

### 3.3 Token 刷新服务逻辑

**文件**: `packages/hoppscotch-backend/src/auth/auth.service.ts:335-363

```typescript
async refreshAuthTokens(hashedRefreshToken: string, user: AuthUser) {
  // 1. 验证用户存在
  if (!user) return E.left(USER_NOT_FOUND);

  // 2. 验证 Refresh Token 哈希匹配（数据库 vs 传入的
  const isTokenMatched = await argon2.verify(
    user.refreshToken,
    hashedRefreshToken,
  );
  if (!isTokenMatched)
    return E.left(INVALID_REFRESH_TOKEN);

  // 3. 生成新的 Access Token + Refresh Token
  const generatedAuthTokens = await this.generateAuthTokens(user.uid);

  return E.right(generatedAuthTokens.right);
}
```

**安全机制**：
- Refresh Token 采用 **Refresh Token Rotation 机制
- 每次刷新都会生成**新的 Refresh Token
- 数据库存储的是 argon2 哈希值进行比对

### 3.4 Cookie 处理助手

**文件**: `packages/hoppscotch-backend/src/auth/helper.ts:38-82

```typescript
export const authCookieHandler = (
  res: Response,
  authTokens: AuthTokens,
  redirect: boolean,
  redirectUrl: string | null,
  configService: ConfigService,
) => {
  // 设置 HttpOnly Cookie
  res.cookie('access_token', authTokens.access_token, {
    httpOnly: true,
    secure: configService.get('INFRA.ALLOW_SECURE_COOKIES') === 'true',
    sameSite: 'lax',
    maxAge: accessTokenValidityInMs,
  });

  res.cookie('refresh_token', authTokens.refresh_token, {
    httpOnly: true,
    secure: configService.get('INFRA.ALLOW_SECURE_COOKIES') === 'true',
    sameSite: 'lax',
    maxAge: refreshTokenValidityInMs,
  });
};
```

**Cookie 属性**：
- `httpOnly: true` - 防止 XSS 攻击
- `secure` - HTTPS 传输
- `sameSite: 'lax' - 防止 CSRF 攻击

---

## 四、前端 Token 刷新触发时机

### 4.1 应用初始化时的刷新

**文件**: `packages/hoppscotch-selfhost-web/src/platform/auth/web/index.ts:92-150

```typescript
async function setInitialUser() {
  const res = await getInitialUserDetails()

  const error = res.errors && res.errors[0]

  // ...错误处理...

  // Access Token 过期，需要刷新
  if (error && error.message === "Unauthorized") {
    const isRefreshSuccess = await refreshToken()

    if (isRefreshSuccess) {
      setInitialUser()  // 重试获取用户信息
    } else {
      await setUser(null)
      await logout()
    }
    return
  }
}
```

**触发场景**：
- 页面刷新/重新打开应用
- 检测到 Access Token 过期（"Unauthorized" 错误）

### 4.2 GQL API 请求时的自动刷新

**文件**: `packages/hoppscotch-common/src/helpers/backend/GQLClient.ts:74-118

```typescript
authExchange(async (): Promise<AuthConfig> => {
  return {
    // 1. 操作前检查是否需要认证
    willAuthError() {
      return platform.auth.willBackendHaveAuthError()
    },

    // 2. 检测认证错误
    didAuthError(error) {
      return error.graphQLErrors.some(
        (e) =>
          e.message.includes("auth/fail") ||
          e.message.includes("jwt expired") ||
          e.extensions?.code === "UNAUTHENTICATED"
      )
    },

    // 3. 执行刷新
    async refreshAuth() {
      const refresh = platform.auth.refreshAuthToken
      if (!refresh) return

      await authRetryGuard.execute(() => refresh.call(platform.auth))
    },
  }
})
```

**urql authExchange 工作流**：

```
API 请求
    ↓
willAuthError()?
    ├─ 是 → 直接发起请求
    └─ 否 → refreshAuth() → 重试请求
    ↓
请求返回
    ↓
didAuthError()?
    ├─ 是 → refreshAuth() → 重试请求
    └─ 否 → 返回结果
```

### 4.3 前端刷新实现

**文件**: `packages/hoppscotch-selfhost-web/src/platform/auth/web/index.ts:152-173

```typescript
async function refreshToken() {
  try {
    const res = await axios.get(
      `${import.meta.env.VITE_BACKEND_API_URL}/auth/refresh`,
      {
        withCredentials: true,  // 自动携带 Cookie
      }
    )

    const isSuccessful = res.status === 200

    if (isSuccessful) {
      authEvents$.next({
        event: "token_refresh",
      })
    }

    return isSuccessful
  } catch (_error) {
    return false
  }
}
```

**关键点**：
- `withCredentials: true` 自动携带 Cookie
- 刷新成功后触发 `token_refresh` 事件
- 失败返回 `token_refresh` 事件触发 GQL 客户端重连

---

## 五、重试机制与失败处理

### 5.1 认证重试守卫

**文件**: `packages/hoppscotch-common/src/helpers/retryAuthGuard.ts

```typescript
const MAX_RETRIES = 3

export function createAuthRetryGuard(onExhausted: () => void | Promise<void>) {
  let failCount = 0
  let isExhausted = false

  return {
    async execute(refreshFn: () => Promise<boolean>): Promise<boolean> {
      if (isExhausted || failCount >= MAX_RETRIES) {
        return false
      }

      let success: boolean
      try {
        success = await refreshFn()
      } catch (_) {
        success = false
      }

      if (success) {
        failCount = 0  // 成功重置计数器
        return true
      }

      failCount++
      if (failCount >= MAX_RETRIES && !isExhausted) {
        isExhausted = true
        onExhausted()  // 触发登出
      }

      return false
    },

    reset() {
      failCount = 0
      isExhausted = false
    },
  }
}
```

**设计目的**：
- 防止无限刷新循环（参考 Issue #5885
- 连续 3 次刷新失败后自动登出
- 登录成功后重置计数器

### 5.2 失败降级策略

| 失败次数 | 处理方式 |
|---------|---------|
| 1-2 次 | 静默失败，下次请求继续尝试 |
| 第 3 次 | 触发 `signOutUser() 登出 |
| 登录成功 | 重置计数器为 0 |

---

## 六、事件流与状态更新

### 6.1 认证事件流

**文件**: `packages/hoppscotch-common/src/platform/auth.ts:37-41

```typescript
export type AuthEvent =
  | { event: "probable_login"; user: HoppUser }
  | { event: "login"; user: HoppUser }
  | | | | | | | | | |
```

### 6.2 Token 刷新后的客户端重连机制

**文件**: `packages/hoppscotch-common/src/helpers/backend/GQLClient.ts:169-189

```typescript
platform.auth.onBackendGQLClientShouldReconnect(() => {
  const currentUser = platform.auth.getCurrentUser()

  // 有用户 && 有连接 → 关闭旧连接
  if (currentUser && subscriptionClient) {
    subscriptionClient?.client?.close()
  }

  // 有用户 && 无连接 → 创建新连接
  if (currentUser && !subscriptionClient) {
    subscriptionClient = createSubscriptionClient()
  }

  // 无用户 && 有连接 → 关闭连接
  if (!currentUser && subscriptionClient) {
    subscriptionClient.close()
    subscriptionClient = null
  }

  // 重建 GQL 客户端
  client.value = createHoppClient()
})
```

**触发重连的事件**：
- `login` - 用户登录
- `logout` - 用户登出
- `token_refresh` - Token 刷新

---

## 七、完整流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                     前端应用启动                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────┐
                │  performAuthInit()        │
                └─────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────┐
                │ setInitialUser()      │
                └─────────────────────────┘
                              │
          ┌─────────────────────────┐
          │  调用 /me 查询   │
          └─────────────────────────┘
                              │
          ┌─────────────────────────┐
          │  Token 有效?        │
          └─────────────────────────┘
                │         │
                │ 是        │ 否
                │            │
                ▼            ▼
┌───────────────────────┐  ┌──────────────────────┐
│  设置用户状态      │  │  refreshToken()     │
│  触发 login 事件   │  └──────────────────────┘
└───────────────────────┘            │
                │                  │
                │                  ▼
                │        ┌──────────────────────┐
                │        │  调用 /auth/refresh      │
                │        └──────────────────────┘
                │                  │
                │                  ▼
                │        ┌──────────────────────┐
                │        │ RTJwtStrategy 验证  │
                │        └──────────────────────┘
                │                  │
                │                  ▼
                │        ┌──────────────────────┐
                │        │ refreshAuthTokens() │
                │        └──────────────────────┘
                │                  │
                │                  ▼
                │        ┌──────────────────────┐
                │        │ 生成新 Token 对     │
                │        └──────────────────────┘
                │                  │
                │                  ▼
                │        ┌──────────────────────┐
                │        │ authCookieHandler  │
                │        │ 写入 HttpOnly Cookie │
                │        └──────────────────────┘
                │                  │
                │                  ▼
                │        ┌──────────────────────┐
                │        │ 触发 token_refresh │
                │        └──────────────────────┘
                │                  │
                │                  ▼
                │        ┌──────────────────────┐
                │        │ 重建 GQL 客户端    │
                │        │ 重建 WebSocket      │
                │        └──────────────────────┘
                │                  │
                │                  ▼
                │        ┌──────────────────────┐
                │        │ 重试 setInitialUser()│
                │        └──────────────────────┘
                │                  │
                └───────────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────┐
                │  正常 API 请求        │
                └─────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────┐
                │ urql authExchange     │
                └─────────────────────────┘
                              │
          ┌─────────────────────────┐
          │ willAuthError()?        │
          └─────────────────────────┘
                │         │
                │ 是        │ 否
                │            │
                ▼            ▼
┌───────────────────────┐  ┌──────────────────────┐
│  发起请求            │  │  refreshAuth()       │
└───────────────────────┘  └──────────────────────┘
                              │
                              ▼
                        ┌──────────────────────┐
                        │ authRetryGuard.execute() │
                        └──────────────────────┘
                              │
          ┌─────────────────────────┐
          │  请求返回              │
          └─────────────────────────┘
                              │
          ┌─────────────────────────┐
          │ didAuthError()?        │
          └─────────────────────────┘
                │         │
                │ 是        │ 否
                │            │
                ▼            ▼
┌───────────────────────┐  ┌──────────────────────┐
│  返回结果            │  │  refreshAuth()       │
└───────────────────────┘  └──────────────────────┘
                              │
                              ▼
                        ┌──────────────────────┐
                        │  连续失败 3 次?     │
                        └──────────────────────┘
                              │
                │         │
                │ 是        │ 否
                │            │
                ▼            ▼
┌───────────────────────┐  ┌──────────────────────┐
│  signOutUser()      │  │  继续重试          │
│  清除本地状态         │  └──────────────────────┘
└───────────────────────┘
```

---

## 八、关键交互时序图

```
前端 (Frontend)                          后端 (Backend)
     │                                         │
     │ 1. GET /api/v1/auth/refresh         │
     │──────────────────────────────────────────▶│
     │  (携带 Cookie: refresh_token            │
     │                                         │
     │                                         │
     │                                         │ 2. RTJwtStrategy 验证
     │                                         │    - 从 Cookie 提取 refresh_token
     │                                         │    - 验证 JWT 签名
     │                                         │    - 查询用户信息
     │                                         │
     │                                         │
     │                                         │ 3. refreshAuthTokens()
     │                                         │    - argon2.verify(DBHash, 传入 token)
     │                                         │    - 生成新 access_token + refresh_token
     │                                         │
     │                                         │
     │                                         │ 4. authCookieHandler()
     │                                         │    - Set-Cookie: access_token=xxx
     │                                         │    - Set-Cookie: refresh_token=xxx
     │                                         │
     │ 5. HTTP 200 OK                       │
     │◀──────────────────────────────────────────│
     │                                         │
     │ 6. 触发 token_refresh 事件              │
     │    - 重建 GQL 客户端                    │
     │    - 重建 WebSocket 连接                  │
     │                                         │
     │ 7. 重试失败?                               │
     │    - 失败计数++                            │
     │    - 失败 >= 3? → 登出                      │
     │                                         │
```

---

## 九、配置项说明

**环境变量配置** (`.env`)：

| 配置项 | 默认值 | 说明 |
|-------|-------|------|
| `INFRA.JWT_SECRET` | - | JWT 签名密钥 |
| `INFRA.ACCESS_TOKEN_VALIDITY` | `86400000 | Access Token 有效期 (ms) |
| `INFRA.REFRESH_TOKEN_VALIDITY` | `604800000` | Refresh Token 有效期 (ms) |
| `INFRA.ALLOW_SECURE_COOKIES` | `false` | 是否启用 Secure Cookie |
| `INFRA.TOKEN_SALT_COMPLEXITY` | - | bcrypt salt 复杂度 |

---

## 十、安全设计要点

1. **HttpOnly Cookie**：防止 XSS 窃取 Token
2. **Refresh Token Rotation**：每次刷新生成新 Token，降低泄露风险
3. **argon2 哈希存储**：数据库不存储明文 Refresh Token
4. **重试次数限制**：防止暴力破解
5. **SameSite Cookie**：防止 CSRF 攻击
6. **Secure Cookie**：HTTPS 传输加密

---

## 十一、常见问题排查

### Q1: Token 刷新失败怎么办？
- 检查 Refresh Token 是否过期（7 天有效期）
- 检查 Cookie 是否被正确发送
- 查看后端日志查看是否有错误信息

### Q2: 为什么会自动登出？
- 连续 3 次刷新失败
- Refresh Token 已过期
- 用户被管理员吊销

### Q3: 如何延长会话保持？
- 只要 7 天内有活动，会自动刷新
- 超过 7 天无活动，需要重新登录

### Q4: 多端登录会互相影响吗？
- 不会，每个设备有独立的 Refresh Token
- 数据库每个用户只有一个 Refresh Token（会覆盖）

---

## 十二、代码溯源索引

| 功能模块 | 文件路径 | 行号 |
|---------|---------|------|
| 刷新端点 | `auth.controller.ts` | 87-100 |
| 刷新服务 | `auth.service.ts` | 335-363 |
| RT策略 | `rt-jwt.strategy.ts` | 1-50 |
| Cookie处理 | `helper.ts` | 38-82 |
| 前端刷新函数 | `web/index.ts` | 152-173 |
| GQL自动刷新 | `GQLClient.ts` | 74-118 |
| 重试守卫 | `retryAuthGuard.ts` | 1-78 |
| 初始化刷新 | `web/index.ts` | 92-150 |
| 客户端重连 | `GQLClient.ts` | 169-189 |
