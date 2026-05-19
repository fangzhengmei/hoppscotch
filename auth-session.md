# Hoppscotch 认证与会话机制深度分析

## 一、整体架构概览

Hoppscotch 采用 **双令牌认证机制**（Access Token + Refresh Token）结合 **HttpOnly Cookie** 存储，实现了安全且用户友好的会话管理。

```
┌─────────────────┐     1. 登录请求     ┌─────────────────┐
│   前端浏览器    │ ──────────────────> │   后端服务      │
└─────────────────┘                     └─────────────────┘
          ^                                      │
          │ 2. 设置 HttpOnly Cookie              │
          │    (access_token, refresh_token)    │
          │                                      │
          │                                      ▼
          │                              ┌─────────────────┐
          │                              │  生成双令牌     │
          │                              │  存储RT哈希     │
          │                              └─────────────────┘
          │                                      │
          │                                      │
          │ 3. 后续请求携带 Cookie               │
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

### 2.4 Cookie 设置

令牌通过 `authCookieHandler` 设置到响应的 HttpOnly Cookie 中：

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

---

## 三、认证守卫拦截机制

### 3.1 守卫层级结构

```
AuthGuard ('jwt') ──> JwtAuthGuard ──> GqlAuthGuard (GraphQL专用)
AuthGuard ('jwt-refresh') ──> RTJwtAuthGuard (刷新专用)
```

### 3.2 JwtAuthGuard - 主认证守卫

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

**验证流程**：
```typescript
async validate(payload: AccessTokenPayload) {
  if (!payload) throw new ForbiddenException(INVALID_ACCESS_TOKEN);
  
  // 根据 token 中的 sub (userUid) 查询用户
  const user = await this.usersService.findUserById(payload.sub);
  if (O.isNone(user)) {
    throw new UnauthorizedException(USER_NOT_FOUND);
  }
  
  // 用户对象会被附加到 request.user 上，供后续业务逻辑使用
  return user.value;
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

### 3.4 RTJwtAuthGuard - Refresh Token 守卫

**代码位置**：`packages/hoppscotch-backend/src/auth/guards/rt-jwt-auth.guard.ts`

```typescript
@Injectable()
export class RTJwtAuthGuard extends AuthGuard('jwt-refresh') {}
```

专用策略 `RTJwtStrategy`：

**代码位置**：`packages/hoppscotch-backend/src/auth/strategies/rt-jwt.strategy.ts`

```typescript
async validate(payload: RefreshTokenPayload) {
  if (!payload) throw new ForbiddenException(INVALID_REFRESH_TOKEN);
  
  const user = await this.usersService.findUserById(payload.sub);
  if (O.isNone(user)) {
    throw new UnauthorizedException(USER_NOT_FOUND);
  }
  
  return user.value;
}
```

**注意**：此守卫只验证 refresh_token 的 JWT 签名和用户存在性，**不验证与数据库存储的哈希是否匹配**。哈希验证在后续的业务逻辑中进行。

---

## 四、过期处理与接力机制

### 4.1 问题场景

1. **Access Token 过期**（默认 1 天）：正常请求失败，返回 401
2. **Refresh Token 过期**（默认 7 天）：刷新失败，需要重新登录
3. **令牌轮换安全**：每次刷新都会生成新的 refresh_token，防止被盗用

### 4.2 前端自动刷新流程

前端通过 URQL 的 `authExchange` 实现无缝的 token 刷新：

**代码位置**：`packages/hoppscotch-common/src/helpers/backend/GQLClient.ts:74-117`

```typescript
authExchange(async (): Promise<AuthConfig> => {
  return {
    // 1. 给每个请求添加认证头
    addAuthToOperation(operation) {
      const authHeaders = platform.auth.getBackendHeaders();
      // 将 Cookie 自动携带（因为是 httpOnly，JS 无法直接访问，浏览器自动处理）
      return makeOperation(operation.kind, operation, {
        ...operation.context,
        fetchOptions: {
          ...fetchOptions,
          credentials: 'include', // 关键：确保 Cookie 被发送
          headers: { ...fetchOptions.headers, ...authHeaders },
        },
      });
    },

    // 2. 预判是否会出现认证错误
    willAuthError() {
      return platform.auth.willBackendHaveAuthError();
    },

    // 3. 检测是否发生认证错误
    didAuthError(error) {
      return error.graphQLErrors.some(
        (e) =>
          e.message.includes("auth/fail") ||
          e.message.includes("jwt expired") ||
          e.extensions?.code === "UNAUTHENTICATED"
      );
    },

    // 4. 认证错误时触发刷新
    async refreshAuth() {
      const refresh = platform.auth.refreshAuthToken;
      if (!refresh) return;
      
      // 带重试保护的刷新
      await authRetryGuard.execute(() => refresh.call(platform.auth));
    },
  };
});
```

### 4.3 后端刷新接口

**代码位置**：`packages/hoppscotch-backend/src/auth/auth.controller.ts:87-100`

```typescript
@Get('refresh')
@UseGuards(RTJwtAuthGuard)  // 先用 RTJwtAuthGuard 验证 refresh_token 的有效性
async refresh(
  @GqlUser() user: AuthUser,
  @RTCookie() refresh_token: string,
  @Res() res,
) {
  // 再验证 refresh_token 与数据库存储的哈希是否匹配
  const newTokenPair = await this.authService.refreshAuthTokens(
    refresh_token,
    user,
  );
  if (E.isLeft(newTokenPair)) throwHTTPErr(newTokenPair.left);
  
  // 生成新的令牌对并设置 Cookie
  authCookieHandler(res, newTokenPair.right, false, null, this.configService);
}
```

**核心验证逻辑**：`packages/hoppscotch-backend/src/auth/auth.service.ts:335-363`

```typescript
async refreshAuthTokens(hashedRefreshToken: string, user: AuthUser) {
  // 验证 refresh_token 哈希匹配
  const isTokenMatched = await argon2.verify(
    user.refreshToken,      // 数据库中存储的哈希
    hashedRefreshToken,     // 客户端传来的 refresh_token
  );
  
  if (!isTokenMatched) {
    return E.left({ message: INVALID_REFRESH_TOKEN, statusCode: 404 });
  }
  
  // 生成新的令牌对（同时生成新的 refresh_token）
  return this.generateAuthTokens(user.uid);
}
```

### 4.4 重试保护机制

**代码位置**：`packages/hoppscotch-common/src/helpers/retryAuthGuard.ts`

为防止无限刷新循环，实现了重试次数限制：

```typescript
const MAX_RETRIES = 3;

export function createAuthRetryGuard(onExhausted: () => void) {
  let failCount = 0;
  let isExhausted = false;

  return {
    async execute(refreshFn: () => Promise<boolean>): Promise<boolean> {
      if (isExhausted || failCount >= MAX_RETRIES) {
        return false;
      }

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

## 五、完整请求生命周期时序图

```
前端浏览器                          后端服务
    │                                 │
    │ 1. 请求 GraphQL API             │
    │    (自动携带 Cookie)            │
    │────────────────────────────────>│
    │                                 │
    │                                 │ 2. GqlAuthGuard 拦截
    │                                 │    ├─ 提取 access_token
    │                                 │    ├─ JWT 签名验证
    │                                 │    └─ 查询用户信息
    │                                 │
    │                                 │ 3. access_token 过期 ✗
    │                                 │    返回 401 UNAUTHENTICATED
    │<────────────────────────────────│
    │                                 │
    │ 4. authExchange 捕获错误        │
    │    触发 refreshAuth()           │
    │                                 │
    │ 5. 请求 /auth/refresh           │
    │    (携带 refresh_token)         │
    │────────────────────────────────>│
    │                                 │
    │                                 │ 6. RTJwtAuthGuard 拦截
    │                                 │    ├─ 提取 refresh_token
    │                                 │    └─ JWT 签名验证
    │                                 │
    │                                 │ 7. refreshAuthTokens 验证
    │                                 │    ├─ argon2.verify 哈希匹配
    │                                 │    ├─ 生成新 access_token
    │                                 │    └─ 生成新 refresh_token
    │                                 │
    │                                 │ 8. Set-Cookie 更新令牌
    │<────────────────────────────────│
    │                                 │
    │ 9. 自动重试原始 GraphQL 请求    │
    │    (携带新 Cookie)              │
    │────────────────────────────────>│
    │                                 │
    │                                 │ 10. 验证通过，执行业务逻辑 ✓
    │<────────────────────────────────│
```

---

## 六、关键设计要点总结

### 6.1 安全设计

| 特性 | 实现方式 | 代码位置 |
|------|----------|----------|
| XSS 防护 | HttpOnly Cookie 存储令牌 | `helper.ts:57-68` |
| CSRF 防护 | SameSite: 'lax' Cookie 属性 | `helper.ts:61,66` |
| 令牌泄露防护 | Refresh Token 轮换机制 | `auth.service.ts:355` |
| 数据库泄露防护 | Refresh Token 以 argon2 哈希存储 | `auth.service.ts:114` |

### 6.2 用户体验设计

| 特性 | 实现方式 | 代码位置 |
|------|----------|----------|
| 无缝续期 | 前端 authExchange 自动刷新 | `GQLClient.ts:74-117` |
| 失败保护 | 重试 3 次失败后登出 | `retryAuthGuard.ts:17-78` |
| 多端适配 | Cookie + Authorization Header 双支持 | `jwt.strategy.ts:64-73` |

### 6.3 可扩展性设计

| 特性 | 实现方式 | 代码位置 |
|------|----------|----------|
| 多认证提供者 | 策略模式（Google/GitHub/Microsoft/Email） | `auth.controller.ts` |
| 平台无关性 | AuthPlatformDef 抽象接口 | `platform/auth.ts` |
| GraphQL/REST 双支持 | GqlAuthGuard + JwtAuthGuard 分离 | `gql-auth.guard.ts` |

---

## 七、核心文件索引

| 文件路径 | 主要职责 |
|----------|----------|
| `packages/hoppscotch-backend/src/auth/auth.controller.ts` | 认证接口定义 |
| `packages/hoppscotch-backend/src/auth/auth.service.ts` | 令牌生成、刷新核心逻辑 |
| `packages/hoppscotch-backend/src/auth/helper.ts` | Cookie 处理工具函数 |
| `packages/hoppscotch-backend/src/auth/strategies/jwt.strategy.ts` | Access Token 验证策略 |
| `packages/hoppscotch-backend/src/auth/strategies/rt-jwt.strategy.ts` | Refresh Token 验证策略 |
| `packages/hoppscotch-backend/src/guards/gql-auth.guard.ts` | GraphQL 认证守卫 |
| `packages/hoppscotch-common/src/helpers/backend/GQLClient.ts` | 前端 GQL 客户端 + authExchange |
| `packages/hoppscotch-common/src/helpers/retryAuthGuard.ts` | 刷新重试保护 |
| `packages/hoppscotch-sh-admin/src/helpers/auth.ts` | 自托管版前端认证实现 |
