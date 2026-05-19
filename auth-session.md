# Hoppscotch 认证与会话机制深度分析

## 一、整体架构概览

Hoppscotch 采用 **双令牌认证机制**（Access Token + Refresh Token），但在 **Web 端** 和 **Desktop 端** 采用了完全不同的会话承载方式：

```
┌─────────────────────────────────────────────────────────────────┐
│                        认证架构双层设计                          │
├─────────────────────────────────┬───────────────────────────────┤
│         Web 浏览器端            │       Desktop 桌面端           │
│  HttpOnly Cookie 存储令牌       │  本地持久化存储 + Auth Header  │
│  浏览器自动携带凭证             │  代码手动注入凭证              │
└─────────────────────────────────┴───────────────────────────────┘
```

```
┌─────────────────┐     1. 登录请求     ┌─────────────────┐
│   前端应用      │ ──────────────────> │   后端服务      │
│  (Web/Desktop) │                     │                 │
└─────────────────┘                     └─────────────────┘
          ^                                      │
          │ 2. 下发令牌                          │
          │    - Web: Set-Cookie                 │
          │    - Desktop: JSON Body              │
          │                                      │
          │                                      ▼
          │                              ┌─────────────────┐
          │                              │  生成双令牌     │
          │                              │  存储RT哈希     │
          │                              └─────────────────┘
          │                                      │
          │                                      │
          │ 3. 后续请求携带凭证                  │
          │    - Web: Cookie (自动)              │
          │    - Desktop: Authorization Header   │
          └──────────────────────────────────────┘
```

---

## 二、登录流程与会话存储

### 2.1 登录入口

支持四种登录方式：
- **邮箱魔术链接** (`/auth/signin` → `/auth/verify`)
- **Google SSO** (`/auth/google` → `/auth/google/callback`)
- **GitHub SSO** (`/auth/github` → `/auth/github/callback`)
- **Microsoft SSO** (`/auth/microsoft` → `/auth/microsoft/callback`)

**核心代码位置**：`packages/hoppscotch-backend/src/auth/auth.controller.ts`

### 2.2 令牌生成

登录验证通过后，调用 `AuthService.generateAuthTokens()` 生成令牌对：

**代码位置**：`packages/hoppscotch-backend/src/auth/auth.service.ts:135-151`

```typescript
async generateAuthTokens(userUid: string) {
  const accessTokenPayload = {
    iss: this.configService.get('VITE_BASE_URL'),
    sub: userUid,  // 用户唯一标识
    aud: [this.configService.get('VITE_BASE_URL')],
  };

  const refreshToken = await this.generateRefreshToken(userUid);
  
  return {
    access_token: await this.jwtService.sign(accessTokenPayload, {
      expiresIn: this.configService.get('INFRA.ACCESS_TOKEN_VALIDITY'), // 默认 1 天
    }),
    refresh_token: refreshToken.right, // 默认 7 天
  };
}
```

### 2.3 Refresh Token 的特殊处理

Refresh Token 生成后，**不会直接存入数据库**，而是先进行哈希处理：

**代码位置**：`packages/hoppscotch-backend/src/auth/auth.service.ts:103-127`

```typescript
private async generateRefreshToken(userUid: string) {
  const refreshTokenPayload = { /* ... */ };
  const refreshToken = await this.jwtService.sign(refreshTokenPayload, {
    expiresIn: this.configService.get('INFRA.REFRESH_TOKEN_VALIDITY'), // 7 天
  });

  // 关键：使用 argon2 哈希后存储
  const refreshTokenHash = await argon2.hash(refreshToken);
  
  await this.usersService.updateUserRefreshToken(refreshTokenHash, userUid);
  
  return E.right(refreshToken);
}
```

**存储位置**：数据库 `User` 表的 `refreshToken` 字段（存储的是哈希值）

### 2.4 Web 端：Cookie 存储（HttpOnly）

Web 端令牌通过 `authCookieHandler` 设置到响应的 HttpOnly Cookie 中：

**代码位置**：`packages/hoppscotch-backend/src/auth/helper.ts:38-82`

```typescript
export const authCookieHandler = (res, authTokens, redirect, redirectUrl, configService) => {
  res.cookie('access_token', authTokens.access_token, {
    httpOnly: true,        // 禁止 JS 访问，防止 XSS
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

**关键安全特性**：
- `httpOnly: true` - 令牌对前端 JavaScript 不可见，防范 XSS 攻击
- `sameSite: 'lax'` - 防范 CSRF 攻击
- `secure` - HTTPS 环境下才会发送 Cookie

### 2.5 Desktop 端：本地持久化存储 + Authorization Header

Desktop 端 **不使用 Cookie**，而是通过特殊的回调接口获取令牌，由前端代码管理：

**后端 Desktop 回调**：`packages/hoppscotch-backend/src/auth/auth.controller.ts:201-219`

```typescript
@Get('desktop')
@UseGuards(JwtAuthGuard)  // JwtAuthGuard 支持从 Authorization Header 提取
@UseInterceptors(UserLastLoginInterceptor)
async desktopAuthCallback(
  @GqlUser() user: AuthUser,
  @Query('redirect_uri') redirectUri: string,
) {
  // ... 验证 redirectUri
  const tokens = await this.authService.generateAuthTokens(user.uid);
  return tokens.right;  // 直接返回 JSON，不设置 Cookie
}
```

**前端 Desktop 存储**：`packages/hoppscotch-selfhost-web/src/platform/auth/desktop/index.ts:289-302`

```typescript
async function setAuthCookies(headers: Headers) {
  const cookieHeader = headers.get("set-cookie");
  // 从 Set-Cookie 头部解析出令牌
  const accessTMatch = cookieHeader.match(/access_token=([^;,\s]+)/);
  const refreshTMatch = cookieHeader.match(/refresh_token=([^;,\s]+)/);

  if (accessTMatch) {
    await persistenceService.setLocalConfig("access_token", accessTMatch[1]);
  }
  if (refreshTMatch) {
    await persistenceService.setLocalConfig("refresh_token", refreshTMatch[1]);
  }
}
```

### 2.6 Web vs Desktop 会话承载对比表

| 特性 | Web 浏览器端 | Desktop 桌面端 |
|------|-------------|----------------|
| 存储位置 | HttpOnly Cookie | 本地持久化 (Tauri Store) |
| JS 可见性 | 不可见 (httpOnly) | 完全可见 |
| 请求携带方式 | 浏览器自动附加 Cookie | 代码手动注入 Authorization Header |
| 令牌获取方式 | Set-Cookie 响应头 | JSON 响应体 / 解析 Set-Cookie |
| XSS 风险 | 低 (httpOnly 保护) | 高 (令牌可被 JS 读取) |
| CSRF 风险 | 中 (依赖 SameSite) | 无 (自定义 Header) |

---

## 三、认证守卫拦截机制

### 3.1 守卫层级结构

```
AuthGuard ('jwt') ──> JwtAuthGuard ──> GqlAuthGuard (GraphQL专用)
         │
         └─ 支持 Cookie + Authorization Header 双提取

AuthGuard ('jwt-refresh') ──> RTJwtAuthGuard (刷新专用)
         │
         └─ 仅支持 Cookie 提取 ⚠️
```

### 3.2 JwtAuthGuard - 主认证守卫（Web + Desktop 通用）

**代码位置**：`packages/hoppscotch-backend/src/auth/guards/jwt-auth.guard.ts`

```typescript
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

该守卫基于 Passport 的 `jwt` 策略，具体验证逻辑在 `JwtStrategy` 中：

**代码位置**：`packages/hoppscotch-backend/src/auth/strategies/jwt.strategy.ts`

**令牌提取优先级**：
1. 从 Cookie 中提取 `access_token`
2. 若 Cookie 不存在，从 `Authorization: Bearer <token>` 头部提取

```typescript
const extractToken = (request: Request): E.Either<Error, string> =>
  pipe(
    extractFromCookie(request),           // 先尝试 Cookie
    O.alt(() => extractFromAuthHeaders(request)),  // 回退到 Authorization Header
    E.fromOption(() => new ForbiddenException(COOKIES_NOT_FOUND)),
  );
```

**验证流程**：
```typescript
async validate(payload: AccessTokenPayload) {
  if (!payload) throw new ForbiddenException(INVALID_ACCESS_TOKEN);
  
  const user = await this.usersService.findUserById(payload.sub);
  if (O.isNone(user)) {
    throw new UnauthorizedException(USER_NOT_FOUND);
  }
  
  return user.value;  // 用户对象附加到 request.user
}
```

### 3.3 GqlAuthGuard - GraphQL 认证守卫

**代码位置**：`packages/hoppscotch-backend/src/guards/gql-auth.guard.ts`

```typescript
@Injectable()
export class GqlAuthGuard extends AuthGuard('jwt') {
  getRequest(context: ExecutionContext) {
    const ctx = GqlExecutionContext.create(context);
    const { req, headers } = ctx.getContext();
    return headers ? headers : req;
  }
}
```

**作用**：适配 GraphQL 执行上下文，从正确的位置提取请求对象和 headers。

### 3.4 RTJwtAuthGuard - Refresh Token 守卫（⚠️ Web 专用）

**代码位置**：`packages/hoppscotch-backend/src/auth/guards/rt-jwt-auth.guard.ts`

```typescript
@Injectable()
export class RTJwtAuthGuard extends AuthGuard('jwt-refresh') {}
```

专用策略 `RTJwtStrategy`：

**代码位置**：`packages/hoppscotch-backend/src/auth/strategies/rt-jwt.strategy.ts`

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
          // ⚠️ 重要：仅从 Cookie 提取 refresh_token
          const RTCookie = request.cookies?.['refresh_token'];
          if (!RTCookie) {
            console.error('`refresh_token` not found');
            throw new ForbiddenException(COOKIES_NOT_FOUND);
          }
          return RTCookie;
        },
      ]),
      secretOrKey: configService.get('INFRA.JWT_SECRET'),
    });
  }

  async validate(payload: RefreshTokenPayload) {
    if (!payload) throw new ForbiddenException(INVALID_REFRESH_TOKEN);
    const user = await this.usersService.findUserById(payload.sub);
    if (O.isNone(user)) throw new UnauthorizedException(USER_NOT_FOUND);
    return user.value;
  }
}
```

**⚠️ 关键限制**：`RTJwtStrategy` **只从 Cookie 提取 refresh_token**，不支持 `Authorization Header` 方式。这意味着 Desktop 端的 refresh 流程有兼容问题，详见第四章分析。

---

## 四、过期处理与接力机制（含 Web/Desktop 差异）

### 4.1 问题场景

1. **Access Token 过期**（默认 1 天）：正常请求失败，返回 401
2. **Refresh Token 过期**（默认 7 天）：刷新失败，需要重新登录
3. **令牌轮换安全**：每次刷新都会生成新的 refresh_token，防止被盗用

### 4.2 请求携带凭证的实际实现

#### Web 端凭证携带

Web 端通过 `withCredentials: true` (Axios) 或 `credentials: "include"` (Fetch) 让浏览器自动携带 Cookie：

**代码位置**：`packages/hoppscotch-selfhost-web/src/platform/auth/web/index.ts:209-222`

```typescript
getGQLClientOptions() {
  return {
    fetchOptions: {
      credentials: "include",  // Fetch API: 自动携带 Cookie
    },
  };
},

axiosPlatformConfig() {
  return {
    withCredentials: true,  // Axios: 自动携带 Cookie
  };
},

getBackendHeaders() {
  return {};  // Web 端不需要手动设置 Header，Cookie 自动携带
},
```

#### Desktop 端凭证携带

Desktop 端需要手动将 token 注入到 Authorization Header：

**代码位置**：`packages/hoppscotch-selfhost-web/src/platform/auth/desktop/index.ts:316-352`

```typescript
getBackendHeaders() {
  const accessToken = currentUser$.value?.accessToken;
  return accessToken
    ? {
        Authorization: `Bearer ${accessToken}`,  // 手动注入
      }
    : ({} as Record<string, string>);
},

getGQLClientOptions() {
  const accessToken = currentUser$.value?.accessToken;
  return {
    connectionParams: accessToken
      ? {
          Authorization: `Bearer ${accessToken}`,  // WebSocket 订阅
        }
      : undefined,
    fetchOptions: {
      headers: accessToken
        ? { Authorization: `Bearer ${accessToken}` }  // HTTP 请求
        : undefined,
    },
  };
},

axiosPlatformConfig() {
  const accessToken = currentUser$.value?.accessToken;
  return {
    headers: accessToken
      ? {
          Authorization: `Bearer ${accessToken}`,  // Axios 请求
        }
      : {},
  };
},
```

### 4.3 前端自动刷新流程（authExchange）

前端通过 URQL 的 `authExchange` 实现无缝的 token 刷新：

**代码位置**：`packages/hoppscotch-common/src/helpers/backend/GQLClient.ts:74-117`

```typescript
authExchange(async (): Promise<AuthConfig> => {
  return {
    addAuthToOperation(operation) {
      // 通过 platform.auth.getBackendHeaders() 获取认证头
      // Web 端返回空对象，依赖 credentials: "include" 自动携带 Cookie
      // Desktop 端返回 { Authorization: "Bearer <token>" }
      const authHeaders = platform.auth.getBackendHeaders();
      
      return makeOperation(operation.kind, operation, {
        ...operation.context,
        fetchOptions: {
          ...fetchOptions,
          // Web 端通过 getGQLClientOptions() 设置 credentials: "include"
          headers: { ...fetchOptions.headers, ...authHeaders },
        },
      });
    },

    willAuthError() {
      return platform.auth.willBackendHaveAuthError();
    },

    didAuthError(error) {
      return error.graphQLErrors.some(
        (e) =>
          e.message.includes("auth/fail") ||
          e.message.includes("jwt expired") ||
          e.extensions?.code === "UNAUTHENTICATED"
      );
    },

    async refreshAuth() {
      const refresh = platform.auth.refreshAuthToken;
      if (!refresh) return;
      await authRetryGuard.execute(() => refresh.call(platform.auth));
    },
  };
});
```

### 4.4 Refresh 在不同端的传递与守卫提取差异

#### Web 端 Refresh 流程（✅ 正常工作）

**代码位置**：`packages/hoppscotch-selfhost-web/src/platform/auth/web/index.ts:152-173`

```typescript
async function refreshToken() {
  const res = await axios.get(
    `${import.meta.env.VITE_BACKEND_API_URL}/auth/refresh`,
    {
      withCredentials: true,  // 浏览器自动携带 refresh_token Cookie
    }
  );
  return res.status === 200;
}
```

后端接口：`packages/hoppscotch-backend/src/auth/auth.controller.ts:87-100`

```typescript
@Get('refresh')
@UseGuards(RTJwtAuthGuard)  // ✅ RTJwtAuthGuard 从 Cookie 提取 refresh_token
async refresh(
  @GqlUser() user: AuthUser,
  @RTCookie() refresh_token: string,  // 从 Cookie 提取
  @Res() res,
) {
  const newTokenPair = await this.authService.refreshAuthTokens(
    refresh_token,
    user,
  );
  authCookieHandler(res, newTokenPair.right, false, null, this.configService);
}
```

**Web 端 Refresh 时序**：
```
浏览器                          后端
   │                              │
   │ GET /auth/refresh            │
   │ Cookie: refresh_token=xxx    │
   │─────────────────────────────>│
   │                              │ RTJwtAuthGuard 从 Cookie 提取
   │                              │ 验证通过，生成新令牌对
   │                              │ Set-Cookie: access_token=yyy
   │                              │ Set-Cookie: refresh_token=zzz
   │<─────────────────────────────│
   │                              │
```

#### Desktop 端 Refresh 流程（⚠️ 兼容边界问题）

**代码位置**：`packages/hoppscotch-selfhost-web/src/platform/auth/desktop/index.ts:221-260`

```typescript
async function refreshToken() {
  const refreshToken = await persistenceService.getLocalConfig("refresh_token");
  if (!refreshToken) return false;

  const { response } = interceptorService.execute({
    id: Date.now(),
    url: `${import.meta.env.VITE_BACKEND_API_URL}/auth/refresh`,
    method: "GET",
    version: "HTTP/1.1",
    headers: {
      // ⚠️ Desktop 端通过 Authorization Header 发送 refresh_token
      Authorization: `Bearer ${refreshToken}`,
    },
  });

  const res = await response;
  if (E.isLeft(res)) return false;

  await setAuthCookies(res.right.headers);  // 从 Set-Cookie 解析新令牌
  return res.right.status === 200;
}
```

**⚠️ 兼容边界问题**：

`RTJwtStrategy` 只从 `request.cookies?.['refresh_token']` 提取，**不支持 Authorization Header 方式**。这意味着：

1. Desktop 端调用 `/auth/refresh` 时，将 refresh_token 放在 `Authorization: Bearer <token>` 中
2. `RTJwtAuthGuard` 调用 `RTJwtStrategy`，但策略只查找 Cookie 中的 refresh_token
3. Cookie 不存在，抛出 `COOKIES_NOT_FOUND` 错误
4. Desktop 端 refresh 流程失败

**临时兼容方案**：后端必须配置中间件，将 `Authorization: Bearer <refresh_token>` 中的 token 写入到 `request.cookies.refresh_token` 中，使 RTJwtStrategy 可以正常提取。

### 4.5 后端刷新核心验证逻辑

**代码位置**：`packages/hoppscotch-backend/src/auth/auth.service.ts:335-363`

```typescript
async refreshAuthTokens(hashedRefreshToken: string, user: AuthUser) {
  // 验证 refresh_token 哈希匹配（argon2 验证）
  const isTokenMatched = await argon2.verify(
    user.refreshToken,      // 数据库中存储的哈希
    hashedRefreshToken,     // 客户端传来的 refresh_token
  );
  
  if (!isTokenMatched) {
    return E.left({ message: INVALID_REFRESH_TOKEN, statusCode: 404 });
  }
  
  // 生成新的令牌对（同时生成新的 refresh_token - 令牌轮换）
  return this.generateAuthTokens(user.uid);
}
```

### 4.6 重试保护机制

**代码位置**：`packages/hoppscotch-common/src/helpers/retryAuthGuard.ts`

为防止无限刷新循环，实现了重试次数限制：

```typescript
const MAX_RETRIES = 3;

export function createAuthRetryGuard(onExhausted: () => void) {
  let failCount = 0;
  let isExhausted = false;

  return {
    async execute(refreshFn: () => Promise<boolean>): Promise<boolean> {
      if (isExhausted || failCount >= MAX_RETRIES) return false;

      const success = await refreshFn();
      
      if (success) {
        failCount = 0;  // 成功则重置计数
        return true;
      }

      failCount++;
      if (failCount >= MAX_RETRIES && !isExhausted) {
        isExhausted = true;
        onExhausted();  // 连续失败 3 次，触发登出
      }
      return false;
    },
  };
}
```

---

## 五、Logout 会话失效处理现状与风险

### 5.1 Logout 实现分析

#### 后端 Logout 接口

**代码位置**：`packages/hoppscotch-backend/src/auth/auth.controller.ts:186-191`

```typescript
@Get('logout')
async logout(@Res() res: Response) {
  res.clearCookie('access_token');
  res.clearCookie('refresh_token');
  return res.status(200).send();
}
```

**⚠️ 关键问题**：后端 logout **只清除 Cookie**，**不会失效数据库中的 refresh_token 哈希**。

#### Web 端 Logout

**代码位置**：`packages/hoppscotch-selfhost-web/src/platform/auth/web/index.ts:22-26, 341-352`

```typescript
async function logout() {
  await axios.get(`${import.meta.env.VITE_BACKEND_API_URL}/auth/logout`, {
    withCredentials: true,
  });
}

async signOutUser() {
  await logout();  // 请求后端清除 Cookie
  
  probableUser$.next(null);
  currentUser$.next(null);
  await persistenceService.removeLocalConfig("login_state");  // 清除本地状态
  
  authEvents$.next({ event: "logout" });
}
```

#### Desktop 端 Logout

**代码位置**：`packages/hoppscotch-selfhost-web/src/platform/auth/desktop/index.ts:50-61, 512-522`

```typescript
async function logout() {
  const { response } = interceptorService.execute({
    id: Date.now(),
    url: `${import.meta.env.VITE_BACKEND_API_URL}/auth/logout`,
    version: "HTTP/1.1",
    method: "GET",
  });

  await response;
  // 清除本地存储的令牌
  await persistenceService.removeLocalConfig("refresh_token");
  await persistenceService.removeLocalConfig("access_token");
}

async signOutUser() {
  await logout();
  
  probableUser$.next(null);
  currentUser$.next(null);
  await persistenceService.removeLocalConfig("login_state");
  
  authEvents$.next({ event: "logout" });
}
```

### 5.2 Logout 安全风险分析

| 风险点 | 描述 | 影响范围 |
|--------|------|----------|
| **Refresh Token 未失效** | 数据库中存储的 refresh_token 哈希仍然有效。如果 refresh_token 之前被泄露，攻击者仍可在有效期内（7天）使用它获取新的访问令牌 | Web + Desktop |
| **无令牌黑名单机制** | 没有实现 token 黑名单或 token 版本号机制，无法主动使已颁发的令牌失效 | Web + Desktop |
| **Desktop 本地存储风险** | Desktop 端令牌存储在本地文件系统中，如果设备被盗取，攻击者可以直接读取 token 文件 | Desktop 专用 |
| **后端 Logout 无验证** | `/auth/logout` 接口不需要认证，任何人都可以调用（但只能清除自己的 Cookie） | 低风险 |

### 5.3 风险场景示例

**场景 1：Refresh Token 泄露后 Logout 无效**

```
攻击者获取用户的 refresh_token (有效期 7 天)
↓
用户执行 Logout 操作
↓
后端清除 Cookie，但数据库中的 refresh_token 哈希仍有效
↓
攻击者使用泄露的 refresh_token 调用 /auth/refresh
↓
获取新的 access_token + refresh_token
↓
攻击者可以继续访问用户资源，直到新 refresh_token 过期
```

**场景 2：Desktop 设备被盗**

```
用户 Desktop 设备被盗
↓
攻击者访问本地持久化存储
↓
读取 access_token 和 refresh_token
↓
即使远程 logout（如果有），本地 token 仍可使用
↓
直到 refresh_token 自然过期（7天）
```

---

## 六、完整请求生命周期时序图

### 6.1 Web 端完整流程

```
浏览器                          后端
   │                              │
   │ 1. 请求 GraphQL API          │
   │    (Cookie 自动携带)         │
   │─────────────────────────────>│
   │                              │
   │                              │ 2. GqlAuthGuard 拦截
   │                              │    ├─ 提取 access_token (Cookie)
   │                              │    ├─ JWT 签名验证
   │                              │    └─ 查询用户信息
   │                              │
   │                              │ 3. access_token 过期 ✗
   │                              │    返回 401 UNAUTHENTICATED
   │<─────────────────────────────│
   │                              │
   │ 4. authExchange 捕获错误     │
   │    触发 refreshAuth()        │
   │                              │
   │ 5. GET /auth/refresh         │
   │    Cookie: refresh_token=xxx │
   │─────────────────────────────>│
   │                              │
   │                              │ 6. RTJwtAuthGuard 拦截
   │                              │    ├─ 从 Cookie 提取 refresh_token
   │                              │    └─ JWT 签名验证
   │                              │
   │                              │ 7. refreshAuthTokens 验证
   │                              │    ├─ argon2.verify 哈希匹配
   │                              │    ├─ 生成新 access_token
   │                              │    └─ 生成新 refresh_token
   │                              │
   │                              │ 8. Set-Cookie 更新令牌
   │<─────────────────────────────│
   │                              │
   │ 9. 自动重试原始 GraphQL 请求 │
   │    (携带新 Cookie)           │
   │─────────────────────────────>│
   │                              │
   │                              │ 10. 验证通过，执行业务逻辑 ✓
   │<─────────────────────────────│
```

### 6.2 Desktop 端完整流程（含兼容边界）

```
Desktop 应用                    后端
   │                              │
   │ 1. 请求 GraphQL API          │
   │    Authorization: Bearer AT  │
   │─────────────────────────────>│
   │                              │
   │                              │ 2. GqlAuthGuard 拦截
   │                              │    ├─ Cookie 不存在，回退到 Header
   │                              │    ├─ 提取 Authorization Header
   │                              │    └─ JWT 签名验证 + 查询用户
   │                              │
   │                              │ 3. access_token 过期 ✗
   │                              │    返回 401 UNAUTHENTICATED
   │<─────────────────────────────│
   │                              │
   │ 4. authExchange 捕获错误     │
   │    触发 refreshAuth()        │
   │                              │
   │ 5. GET /auth/refresh         │
   │    Authorization: Bearer RT  │
   │─────────────────────────────>│
   │                              │
   │                              │ 6. RTJwtAuthGuard 拦截 ⚠️
   │                              │    ├─ 尝试从 Cookie 提取 refresh_token
   │                              │    ├─ Cookie 不存在！
   │                              │    └─ 抛出 COOKIES_NOT_FOUND ✗
   │                              │
   │                              │ ⚠️ 兼容边界：需要中间件将
   │                              │    Authorization Header 写入 Cookie
   │                              │
   │ [兼容后正常流程]              │
   │                              │ 7. refreshAuthTokens 验证
   │                              │ 8. Set-Cookie: 新令牌
   │<─────────────────────────────│
   │                              │
   │ 9. 解析 Set-Cookie 头        │
   │    提取新令牌并本地存储       │
   │                              │
   │ 10. 自动重试原始请求         │
   │     携带新 Authorization     │
   │─────────────────────────────>│
```

---

## 七、关键设计要点总结

### 7.1 安全设计

| 特性 | 实现方式 | 代码位置 |
|------|----------|----------|
| XSS 防护（Web） | HttpOnly Cookie 存储令牌 | `helper.ts:57-68` |
| CSRF 防护（Web） | SameSite: 'lax' Cookie 属性 | `helper.ts:61,66` |
| 令牌泄露防护 | Refresh Token 轮换机制 | `auth.service.ts:355` |
| 数据库泄露防护 | Refresh Token 以 argon2 哈希存储 | `auth.service.ts:114` |
| ⚠️ 缺失设计 | Logout 时 refresh_token 未失效 | `auth.controller.ts:186-191` |
| ⚠️ 风险点（Desktop） | 本地持久化存储，令牌对 JS 可见 | `desktop/index.ts:289-302` |

### 7.2 用户体验设计

| 特性 | 实现方式 | 代码位置 |
|------|----------|----------|
| 无缝续期 | 前端 authExchange 自动刷新 | `GQLClient.ts:74-117` |
| 失败保护 | 重试 3 次失败后登出 | `retryAuthGuard.ts:17-78` |
| 多端适配 | Cookie + Authorization Header 双支持 | `jwt.strategy.ts:64-73` |
| 跨平台抽象 | AuthPlatformDef 统一接口 | `platform/auth.ts` |

### 7.3 可扩展性设计

| 特性 | 实现方式 | 代码位置 |
|------|----------|----------|
| 多认证提供者 | 策略模式（Google/GitHub/Microsoft/Email） | `auth.controller.ts` |
| 平台无关性 | AuthPlatformDef 抽象接口 | `platform/auth.ts` |
| GraphQL/REST 双支持 | GqlAuthGuard + JwtAuthGuard 分离 | `gql-auth.guard.ts` |

### 7.4 Web vs Desktop 架构差异总结

| 维度 | Web 浏览器端 | Desktop 桌面端 |
|------|-------------|----------------|
| 会话载体 | HttpOnly Cookie | 本地持久化 + Auth Header |
| Refresh 传递 | Cookie | Authorization Header |
| 守卫兼容性 | ✅ 原生支持 | ⚠️ 需要中间件兼容 |
| JS 令牌可见性 | ❌ 不可见 | ✅ 可见 |
| 自动凭证携带 | ✅ 浏览器处理 | ❌ 代码手动注入 |
| Logout 后端行为 | 清除 Cookie | 清除 Cookie（无实际作用） |
| Logout 前端行为 | 清除本地状态 | 清除本地 token + 状态 |

---

## 八、核心文件索引

| 文件路径 | 主要职责 |
|----------|----------|
| `packages/hoppscotch-backend/src/auth/auth.controller.ts` | 认证接口定义（登录/刷新/登出/Desktop回调） |
| `packages/hoppscotch-backend/src/auth/auth.service.ts` | 令牌生成、刷新核心逻辑 |
| `packages/hoppscotch-backend/src/auth/helper.ts` | Cookie 处理工具函数 |
| `packages/hoppscotch-backend/src/auth/strategies/jwt.strategy.ts` | Access Token 验证策略（Cookie+Header双支持） |
| `packages/hoppscotch-backend/src/auth/strategies/rt-jwt.strategy.ts` | Refresh Token 验证策略（仅Cookie） |
| `packages/hoppscotch-backend/src/guards/gql-auth.guard.ts` | GraphQL 认证守卫 |
| `packages/hoppscotch-common/src/helpers/backend/GQLClient.ts` | 前端 GQL 客户端 + authExchange 自动刷新 |
| `packages/hoppscotch-common/src/helpers/retryAuthGuard.ts` | 刷新重试保护 |
| `packages/hoppscotch-selfhost-web/src/platform/auth/web/index.ts` | Web 版前端认证实现（Cookie 模式） |
| `packages/hoppscotch-selfhost-web/src/platform/auth/desktop/index.ts` | Desktop 版前端认证实现（Header 模式） |
| `packages/hoppscotch-sh-admin/src/helpers/auth.ts` | 自托管 Admin 面板认证实现 |
