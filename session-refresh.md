# 登录会话刷新与自托管后台对接流程详解

## 一、整体架构概览

Hoppscotch 自托管版本采用 **双 Token 机制**（Access Token + Refresh Token）实现会话保持，
配合 HttpOnly Cookie 存储 Token，前端通过 urql 的 `authExchange` 实现自动 Token 刷新。

系统存在 **两条独立的刷新路径**（初始化路径 vs GQL 运行时路径），它们共享同一个后端刷新接口，
但在前端调用方式、重试逻辑和状态更新方面存在重要差异。

---

## 二、AuthEvent 类型定义与实现差异（核心易混淆点）

### 2.1 通用层定义（hoppscotch-common）

**文件**: `packages/hoppscotch-common/src/platform/auth.ts:37-41`

```typescript
export type AuthEvent =
  | { event: "probable_login"; user: HoppUser }  // 有历史登录态，等待确认
  | { event: "login"; user: HoppUser }            // 已认证
  | { event: "logout" }                           // 未认证且无历史状态
  | { event: "token_refresh"; user: HoppUser }    // Token 已刷新
```

通用层定义 `token_refresh` **携带 `user` 字段**。这是为 Firebase 等能在刷新时立即拿到用户信息的平台设计的。

### 2.2 自托管 Web 实际发射（hoppscotch-selfhost-web）

**文件**: `packages/hoppscotch-selfhost-web/src/platform/auth/web/index.ts:16`

```typescript
// 注意类型 widening：AuthEvent | { event: "token_refresh" }
export const authEvents$ = new Subject<AuthEvent | { event: "token_refresh" }>()
```

实际发射时（`web/index.ts:164-166`）：

```typescript
authEvents$.next({ event: "token_refresh" })  // 无 user 字段！
```

**关键差异**：自托管 Web 平台发射的 `token_refresh` **不携带 `user` 字段**。
原因：`refreshToken()` 只调用 `/auth/refresh` 接口刷新 Cookie，此时并没有用户信息可提供。
用户信息需要等到后续 `setInitialUser()` 重新查询 `/me` 成功后，才通过 `login` 事件补充。

### 2.3 Admin 后台实现（hoppscotch-sh-admin）

**文件**: `packages/hoppscotch-sh-admin/src/helpers/auth.ts:37-40`

```typescript
export type AuthEvent =
  | { event: 'login'; user: HoppUser }
  | { event: 'logout' }
  | { event: 'token_refresh' }   // 也不带 user
```

Admin 后台干脆在本地类型中移除了 `probable_login` 和 `token_refresh` 的 `user` 字段。

### 2.4 事件差异对照表

| 事件 | 通用层 AuthEvent | 自托管 Web 实际发射 | Admin 实际发射 |
|------|-----------------|-------------------|---------------|
| `probable_login` | `{ event, user }` | **从未发射** | **不存在** |
| `login` | `{ event, user }` | `{ event, user }` ✅ | `{ event, user }` ✅ |
| `logout` | `{ event }` | `{ event }` ✅ | `{ event }` ✅ |
| `token_refresh` | `{ event, user }` | `{ event }` ❌无 user | `{ event }` ❌无 user |

> **结论**：通用层的 `AuthEvent.token_refresh.user` 在自托管场景下永远不会被填充。
> `probable_login` 事件在自托管 Web 中也从未被发射（初始化时直接设置 `probableUser$`，
> 而非通过事件通知）。

---

## 三、后端错误消息 → 前端分支映射

### 3.1 后端 GQL 请求认证链路

```
GQL 请求 → GqlAuthGuard(使用 jwt Strategy)
                │
                ├─ Cookie/Header 中无 Token
                │   → ForbiddenException("auth/cookies_not_found")
                │
                ├─ JWT 签名无效或已过期
                │   → NestJS 自动返回 "Unauthorized"
                │
                └─ JWT 有效但用户不存在
                    → UnauthorizedException("user/not_found")
```

**文件**: `packages/hoppscotch-backend/src/auth/strategies/jwt.strategy.ts:64-73`

```typescript
const extractToken = (request: Request): E.Either<Error, string> =>
  pipe(
    extractFromCookie(request),        // 先从 Cookie 取 access_token
    O.alt(() => extractFromAuthHeaders(request)),  // 再从 Authorization header 取
    E.fromOption(() => {
      return new ForbiddenException(COOKIES_NOT_FOUND);  // 两个都没有 → "auth/cookies_not_found"
    }),
  );
```

### 3.2 前端 setInitialUser() 中的分支映射

**文件**: `packages/hoppscotch-selfhost-web/src/platform/auth/web/index.ts:92-150`

| 后端返回的错误消息 | 前端含义 | 前端处理 |
|------------------|---------|---------|
| `"auth/cookies_not_found"` | 无任何 Cookie，用户未登录 | `setUser(null)`，不触发任何事件 |
| `"user/not_found"` | Cookie 有效但用户已被删除 | `setUser(null)`，不触发任何事件 |
| `"Unauthorized"` | Access Token 过期，Cookie 仍在 | 尝试 `refreshToken()` |
| 无错误 + `res.data.me` 存在 | Token 有效，用户已认证 | `setUser(user)` + 发射 `login` 事件 |

### 3.3 前端 GQL 运行时 didAuthError() 中的错误识别

**文件**: `packages/hoppscotch-common/src/helpers/backend/GQLClient.ts:102-109`

```typescript
didAuthError(error) {
  return error.graphQLErrors.some(
    (e) =>
      e.message.includes("auth/fail") ||        // 后端 AUTH_FAIL 常量
      e.message.includes("jwt expired") ||       // JWT 过期
      e.extensions?.code === "UNAUTHENTICATED"   // Apollo 错误码
  )
}
```

| 错误模式 | 来源场景 |
|---------|---------|
| `"auth/fail"` | 后端 `errors.ts:25` 中的 `AUTH_FAIL` 常量 |
| `"jwt expired"` | Passport-JWT 在 Token 过期时抛出 |
| `"UNAUTHENTICATED"` | GraphQL 扩展错误码 |

> **注意**：`setInitialUser()` 和 `didAuthError()` 使用了**不同的错误识别策略**。
> 初始化时通过 GQL `errors[0].message` 精确匹配，运行时通过字符串包含判断。
> 初始化时不识别 `"auth/fail"`，运行时不识别 `"auth/cookies_not_found"`。

---

## 四、两条刷新路径的完整调用顺序与状态变化

### 4.1 路径 A：应用初始化刷新（setInitialUser）

**触发时机**：页面加载/刷新时 `performAuthInit()` 调用

#### 成功路径（Token 有效）

```
performAuthInit()
  ├─ 从 localStorage 读取 login_state → probableUser$.next(旧用户)
  └─ setInitialUser()
       ├─ isGettingInitialUser = true
       ├─ getInitialUserDetails() → GQL /me 查询（withCredentials: true）
       └─ 无错误，res.data.me 存在
            ├─ setUser(hoppUser)
            │    ├─ currentUser$.next(hoppUser)      ← 状态更新
            │    ├─ probableUser$.next(hoppUser)     ← 状态更新
            │    └─ persistence.setLocalConfig(...)   ← 持久化
            ├─ isGettingInitialUser = false
            └─ authEvents$.next({ event: "login", user: hoppUser })  ← 发射事件
```

**此时状态**：
| 状态 | 值 |
|------|-----|
| `currentUser$` | `HoppUser` |
| `probableUser$` | `HoppUser` |
| `login_state` (localStorage) | 用户 JSON |
| `authRetryGuard` | 不涉及（初始化路径不走 retryGuard） |

#### 失败路径 A1：无 Cookie（cookies_not_found）

```
setInitialUser()
  ├─ getInitialUserDetails() → GQL 返回 errors[0].message = "auth/cookies_not_found"
  ├─ setUser(null)
  │    ├─ currentUser$.next(null)
  │    ├─ probableUser$.next(null)
  │    └─ persistence.setLocalConfig("login_state", "null")
  ├─ isGettingInitialUser = false
  └─ 【不发射任何 authEvent】
```

#### 失败路径 A2：用户不存在（user/not_found）

与 A1 完全相同，只是错误消息不同。**不发射任何事件**。

#### 失败路径 A3：Access Token 过期（Unauthorized）→ 刷新成功

```
setInitialUser()
  ├─ getInitialUserDetails() → GQL 返回 errors[0].message = "Unauthorized"
  ├─ refreshToken()
  │    ├─ GET /auth/refresh (withCredentials: true)
  │    ├─ 后端验证 RT → 生成新 Token → Set-Cookie 写入
  │    ├─ res.status === 200 → 成功
  │    ├─ authEvents$.next({ event: "token_refresh" })  ← 发射事件（无 user）
  │    └─ return true
  ├─ 【token_refresh 事件的副作用】
  │    ├─ onBackendGQLClientShouldReconnect 回调触发
  │    │    ├─ createHoppClient() → 重建 GQL 客户端
  │    │    └─ （此时 currentUser$ 仍为 null！）
  │    └─ 各业务模块（settings/collections/environments/history）
  │         └─ syncer.startListeningToSubscriptions()
  ├─ setInitialUser()  ← 递归重试
  │    ├─ getInitialUserDetails() → 用新 Cookie 查询 /me
  │    └─ 成功 → setUser(hoppUser)
  │         ├─ currentUser$.next(hoppUser)
  │         ├─ isGettingInitialUser = false
  │         └─ authEvents$.next({ event: "login", user: hoppUser })  ← 第二次事件
  └─ 【login 事件的副作用】
       ├─ onBackendGQLClientShouldReconnect 回调再次触发
       │    └─ createHoppClient() → 再次重建 GQL 客户端
       └─ authRetryGuard.reset()  ← 重置重试计数器
```

**状态变化时序**：

| 时间点 | currentUser$ | probableUser$ | login_state | authRetryGuard |
|-------|-------------|--------------|-------------|----------------|
| performAuthInit 开始 | null | 旧用户(从 localStorage) | 旧用户 JSON | 初始状态 |
| refreshToken() 成功后 | **null**（未变） | 旧用户（未变） | 旧用户 JSON（未变） | 初始状态 |
| setInitialUser() 重试成功后 | HoppUser | HoppUser | 新用户 JSON | 初始状态 |

> **关键点**：`token_refresh` 事件发射时，`currentUser$` 仍为 null。
> 如果此时有 GQL 请求发起，`willAuthError()` 会返回 `true`，
> 触发 `authExchange` 尝试再次刷新。但 Cookie 已刷新，所以后续 `/me` 查询会成功。

#### 失败路径 A4：Access Token 过期（Unauthorized）→ 刷新失败

```
setInitialUser()
  ├─ getInitialUserDetails() → "Unauthorized"
  ├─ refreshToken()
  │    ├─ GET /auth/refresh → 抛异常或非 200
  │    └─ return false
  ├─ setUser(null)
  │    ├─ currentUser$.next(null)
  │    ├─ probableUser$.next(null)
  │    └─ persistence.setLocalConfig("login_state", "null")
  ├─ isGettingInitialUser = false
  ├─ logout() → GET /auth/logout → 后端清除 Cookie
  └─ 【不发射 logout 事件】
       注意：这里调用的是内部 logout() 函数，不是 signOutUser()。
       内部 logout() 只发 HTTP 请求，不发射 authEvent。
```

> **关键点**：初始化路径刷新失败时，**不发射 `logout` 事件**。
> 这与 GQL 运行时路径（通过 retryGuard 触发 `signOutUser()`）行为不同。
> 此时 GQL 客户端不会被重建，可能导致旧连接残留。

### 4.2 路径 B：GQL 运行时刷新（authExchange + retryGuard）

**触发时机**：已登录状态下 GQL 请求遇到认证错误

#### 触发条件一：willAuthError() 前置检查

```typescript
willAuthError() {
  return !currentUser$.value  // currentUser$ 为 null 时返回 true
}
```

返回 `true` → authExchange 在发起请求**之前**先调用 `refreshAuth()`。

#### 触发条件二：didAuthError() 后置检查

请求已经发出并返回错误后，检查错误内容：

```typescript
didAuthError(error) {
  return error.graphQLErrors.some(
    (e) =>
      e.message.includes("auth/fail") ||
      e.message.includes("jwt expired") ||
      e.extensions?.code === "UNAUTHENTICATED"
  )
}
```

返回 `true` → authExchange 调用 `refreshAuth()` 并重试原始请求。

#### 成功路径

```
GQL 请求 → didAuthError() = true（或 willAuthError() = true）
  └─ refreshAuth()
       └─ authRetryGuard.execute(() => platform.auth.refreshAuthToken())
            │
            ├─ refreshToken()
            │    ├─ GET /auth/refresh → 200 OK
            │    ├─ authEvents$.next({ event: "token_refresh" })
            │    └─ return true
            │
            ├─ authRetryGuard: failCount = 0, return true
            └─ authExchange 用新 Cookie 重试原始 GQL 请求
```

**token_refresh 事件副作用**：

| 订阅者 | 响应行为 |
|-------|---------|
| `onBackendGQLClientShouldReconnect` | `createHoppClient()` 重建 GQL 客户端 |
| settings syncer | `startListeningToSubscriptions()` |
| collections syncer | `startListeningToSubscriptions()` |
| environments syncer | `startListeningToSubscriptions()` |
| history syncer | `startListeningToSubscriptions()` |

> **注意**：与初始化路径不同，运行时刷新成功后 **不会自动调用 setInitialUser()**。
> `currentUser$` 保持原有值（不为 null），所以 `willAuthError()` 返回 `false`，
> 后续请求直接使用新 Cookie 即可。这是两条路径最大的状态差异。

#### 失败路径：第 N 次（N < 3）失败

```
refreshAuth()
  └─ authRetryGuard.execute(...)
       ├─ refreshToken() → return false
       ├─ failCount++  (例如 failCount = 1)
       └─ return false
            └─ authExchange 得到 false → 原始 GQL 请求失败，返回错误给调用方
```

**状态**：
| 状态 | 值 |
|------|-----|
| `currentUser$` | **不变**（仍为旧 HoppUser） |
| `authRetryGuard.failCount` | +1 |
| `authRetryGuard.isExhausted` | false |

#### 失败路径：第 3 次失败（耗尽重试）

```
refreshAuth()
  └─ authRetryGuard.execute(...)
       ├─ refreshToken() → return false
       ├─ failCount = 3
       ├─ isExhausted = true
       ├─ onExhausted() → platform.auth.signOutUser()
       │    ├─ logout() → GET /auth/logout
       │    ├─ probableUser$.next(null)
       │    ├─ currentUser$.next(null)
       │    ├─ persistence.removeLocalConfig("login_state")
       │    └─ authEvents$.next({ event: "logout" })  ← 发射 logout 事件
       └─ return false
```

**logout 事件副作用**：

| 订阅者 | 响应行为 |
|-------|---------|
| `onBackendGQLClientShouldReconnect` | `createHoppClient()` 重建（无用户）+ 关闭 WebSocket |
| settings syncer | `stopListeningToSubscriptions()` |
| collections syncer | `stopListeningToSubscriptions()` |
| environments syncer | `stopListeningToSubscriptions()` |
| history syncer | `stopListeningToSubscriptions()` |

#### 耗尽后的后续请求

```
authRetryGuard.execute(...)
  ├─ isExhausted = true → 直接 return false（不再调用 refreshToken）
  └─ 所有后续 GQL 请求的 refreshAuth() 都直接返回 false
```

只有当用户重新登录（`login` 事件触发 `authRetryGuard.reset()`）后，守卫才会恢复。

---

## 五、两条路径的关键差异总结

| 维度 | 路径 A：初始化刷新 | 路径 B：GQL 运行时刷新 |
|------|------------------|---------------------|
| **触发入口** | `performAuthInit()` → `setInitialUser()` | urql `authExchange` |
| **错误识别** | GQL `errors[0].message` 精确匹配 | `didAuthError()` 字符串包含判断 |
| **刷新调用** | 直接调用 `refreshToken()` | 通过 `authRetryGuard.execute()` 间接调用 |
| **重试上限** | 无限制（递归调用 `setInitialUser()`） | 最多 3 次 |
| **刷新成功后** | 递归调用 `setInitialUser()` 重新获取用户 | 仅重建 GQL 客户端，不重新获取用户 |
| **刷新失败后** | `setUser(null)` + `logout()`（不发事件） | `failCount++`，第 3 次触发 `signOutUser()`（发 `logout` 事件） |
| **`token_refresh` 事件** | 发射（无 user 字段） | 发射（无 user 字段） |
| **`currentUser$` 在刷新成功后** | 仍为 null（等待 setInitialUser 重试） | 保持旧值（已登录用户） |
| **retryGuard 影响** | 不涉及 | 核心机制 |

---

## 六、后端 Token 刷新的完整验证链路

### 6.1 刷新端点

**文件**: `packages/hoppscotch-backend/src/auth/auth.controller.ts:87-100`

```
GET /api/v1/auth/refresh
  │
  ├─ RTJwtAuthGuard (passport jwt-refresh 策略)
  │    ├─ 从 Cookie 提取 refresh_token
  │    ├─ JWT 签名验证（secretOrKey = INFRA.JWT_SECRET）
  │    ├─ JWT 过期检查（expiresIn = INFRA.REFRESH_TOKEN_VALIDITY）
  │    └─ validate(payload) → 查询用户 → 附加到 @GqlUser()
  │
  ├─ authService.refreshAuthTokens(refreshToken, user)
  │    ├─ 验证用户存在
  │    ├─ argon2.verify(dbHashedToken, refreshToken)  ← 二次验证
  │    └─ generateAuthTokens(userUid)
  │         ├─ generateRefreshToken() → JWT签名 + argon2哈希存DB
  │         └─ 签名 accessToken
  │
  └─ authCookieHandler(res, newTokens)
       ├─ Set-Cookie: access_token=xxx (httpOnly, sameSite=lax)
       └─ Set-Cookie: refresh_token=xxx (httpOnly, sameSite=lax)
```

### 6.2 双重验证机制

1. **第一层**：Passport-JWT 验证 Refresh Token 的签名和过期时间（`RTJwtStrategy`）
2. **第二层**：`refreshAuthTokens()` 用 argon2 验证 Token 哈希与数据库存储匹配

第二层的作用：即使 JWT 本身有效，但如果用户已在新设备上登录（新 Refresh Token 覆盖了旧哈希），
旧设备上的 Refresh Token 虽然签名正确但哈希不匹配，会被拒绝。

### 6.3 Refresh Token Rotation

每次刷新成功后：
- 后端生成**全新的** Access Token + Refresh Token 对
- 新 Refresh Token 的 argon2 哈希覆盖数据库中旧值
- 旧 Refresh Token 即刻失效（数据库哈希已变）

---

## 七、token_refresh 事件在各业务模块的副作用

**文件**: `packages/hoppscotch-selfhost-web/src/platform/*/web/index.ts`

所有订阅 `authEvents$` 的业务模块对 `token_refresh` 事件的处理一致：

```typescript
authEvents$.subscribe((event) => {
  if (event.event == "login" || event.event == "token_refresh") {
    syncer.startListeningToSubscriptions()  // 重新开启 GQL Subscription 监听
  }
  if (event.event == "logout") {
    syncer.stopListeningToSubscriptions()   // 停止 GQL Subscription 监听
  }
})
```

| 业务模块 | 文件 | 订阅行为 |
|---------|------|---------|
| Settings | `platform/settings/web/index.ts:34` | `settingsSyncer.startListeningToSubscriptions()` |
| Collections | `platform/collections/web/index.ts:92` | `collectionsSyncer.startListeningToSubscriptions()` |
| Environments | `platform/environments/web/index.ts:49` | 同上模式 |
| History | `platform/history/web/index.ts:62` | 同上模式 |

`login` 和 `token_refresh` 被同等对待——都触发"重新开始监听"。
这是因为 `token_refresh` 后 Cookie 已更新，需要用新凭证重建 GQL Subscription。

---

## 八、GQL 客户端重建的决策逻辑

**文件**: `packages/hoppscotch-common/src/helpers/backend/GQLClient.ts:169-189`

`onBackendGQLClientShouldReconnect` 回调在 `login`/`logout`/`token_refresh` 三个事件时触发：

```
回调触发
  ├─ currentUser 有值 && subscriptionClient 存在
  │    → 关闭旧 WebSocket，新的会在 createHoppClient 中创建
  │
  ├─ currentUser 有值 && subscriptionClient 不存在
  │    → 创建新 WebSocket（首次登录场景）
  │
  ├─ currentUser 为空 && subscriptionClient 存在
  │    → 关闭 WebSocket，置 null（登出场景）
  │
  └─ 无论哪种情况 → createHoppClient()
       → 重建 urql 客户端（包含新的 authExchange 实例）
```

### authExchange 初始化逻辑

```typescript
authExchange(async (): Promise<AuthConfig> => {
  const probableUser = platform.auth.getProbableUser()
  if (probableUser !== null)
    await platform.auth.waitProbableLoginToConfirm()  // 等待初始化完成
  // ...
})
```

`waitProbableLoginToConfirm()` 通过 `watch(isGettingInitialUser)` 等待 `setInitialUser()` 完成。
这确保了新建的 GQL 客户端在认证状态确认后才开始工作。

---

## 九、完整状态变化矩阵

### 9.1 初始化路径（路径 A）的状态变化

| 时刻 | currentUser$ | probableUser$ | login_state | 发射的事件 | isGettingInitialUser |
|-----|-------------|--------------|-------------|----------|---------------------|
| performAuthInit 开始 | null | 旧用户(从 localStorage) | 旧值 | 无 | null |
| setInitialUser 开始 | null | 旧用户 | 旧值 | 无 | true |
| Token 有效成功 | HoppUser | HoppUser | 新值 | `login` | false |
| 无 Cookie 失败 | null | null | "null" | **无** | false |
| 用户不存在失败 | null | null | "null" | **无** | false |
| Token 过期→刷新成功 | null（直到 setInitialUser 重试成功） | 旧用户→HoppUser | 旧值→新值 | `token_refresh` → `login` | true→false |
| Token 过期→刷新失败 | null | null | "null" | **无** | false |

### 9.2 GQL 运行时路径（路径 B）的状态变化

| 时刻 | currentUser$ | probableUser$ | login_state | authRetryGuard | 发射的事件 |
|-----|-------------|--------------|-------------|---------------|----------|
| 刷新成功 | 不变(旧 HoppUser) | 不变 | 不变 | failCount=0 | `token_refresh` |
| 刷新失败(1-2次) | 不变(旧 HoppUser) | 不变 | 不变 | failCount++ | **无** |
| 刷新失败(3次) | null | null | 已删除 | isExhausted=true | `logout` |
| 耗尽后的请求 | null | null | 已删除 | 直接返回 false | **无** |

---

## 十、时序图：初始化路径刷新成功

```
Frontend                                Backend
    │                                      │
    │  performAuthInit()                   │
    │  probableUser$ ← localStorage        │
    │                                      │
    │  setInitialUser()                    │
    │  ──── GQL /me (with cookies) ───────▶│
    │  ◀─── errors: "Unauthorized" ────────│  (Access Token 过期)
    │                                      │
    │  refreshToken()                      │
    │  ──── GET /auth/refresh ────────────▶│
    │                                      │  RTJwtStrategy 验证
    │                                      │  refreshAuthTokens()
    │                                      │  argon2.verify() ✅
    │                                      │  generateAuthTokens()
    │  ◀─── 200 + Set-Cookie ─────────────│  (新 access_token + refresh_token)
    │                                      │
    │  authEvents$.next("token_refresh")   │
    │    → GQL 客户端重建                   │
    │    → syncers.startListening()         │
    │  (currentUser$ 仍为 null!)            │
    │                                      │
    │  setInitialUser() [递归重试]          │
    │  ──── GQL /me (with 新 cookies) ────▶│
    │  ◀─── { data: { me: {...} } } ───────│
    │                                      │
    │  setUser(hoppUser)                   │
    │  currentUser$ ← hoppUser             │
    │  authEvents$.next("login")           │
    │    → GQL 客户端再次重建               │
    │    → authRetryGuard.reset()           │
    │  isGettingInitialUser = false         │
    │                                      │
```

---

## 十一、时序图：GQL 运行时刷新耗尽

```
Frontend                                Backend
    │                                      │
    │  GQL 请求 (已登录状态)               │
    │  ──── GQL query ───────────────────▶│
    │  ◀─── errors: "jwt expired" ────────│
    │                                      │
    │  didAuthError() → true               │
    │  refreshAuth()                       │
    │  └─ authRetryGuard.execute()         │
    │     └─ refreshToken()                │
    │        ──── GET /auth/refresh ──────▶│
    │        ◀─── 401 (RT 也过期) ─────────│
    │        return false                  │
    │     failCount = 1                    │
    │     return false                     │
    │  authExchange 放弃，请求失败          │
    │                                      │
    │  [下一次 GQL 请求]                   │
    │  ──── GQL query ───────────────────▶│
    │  ◀─── errors: "auth/fail" ──────────│
    │                                      │
    │  didAuthError() → true               │
    │  refreshAuth()                       │
    │  └─ authRetryGuard.execute()         │
    │     └─ refreshToken() → false        │
    │     failCount = 2                    │
    │     return false                     │
    │                                      │
    │  [第三次 GQL 请求]                   │
    │  ──── GQL query ───────────────────▶│
    │  ◀─── errors: "UNAUTHENTICATED" ────│
    │                                      │
    │  refreshAuth()                       │
    │  └─ authRetryGuard.execute()         │
    │     └─ refreshToken() → false        │
    │     failCount = 3                    │
    │     isExhausted = true               │
    │     onExhausted() → signOutUser()    │
    │        ├─ logout() ── GET /auth/logout ─▶│
    │        ├─ currentUser$.next(null)    │
    │        ├─ probableUser$.next(null)   │
    │        ├─ removeLocalConfig()        │
    │        └─ authEvents$.next("logout") │
    │           → GQL 客户端重建(无用户)    │
    │           → WebSocket 关闭            │
    │           → syncers.stopListening()  │
    │     return false                     │
    │                                      │
    │  [后续所有请求]                       │
    │  authRetryGuard.execute() → false    │
    │  (isExhausted, 不再调用 refreshToken)│
```

---

## 十二、配置项说明

| 配置项 | 默认值 | 说明 |
|-------|-------|------|
| `INFRA.JWT_SECRET` | - | JWT 签名密钥 |
| `INFRA.ACCESS_TOKEN_VALIDITY` | `86400000` | Access Token 有效期 (ms，1天) |
| `INFRA.REFRESH_TOKEN_VALIDITY` | `604800000` | Refresh Token 有效期 (ms，7天) |
| `INFRA.ALLOW_SECURE_COOKIES` | `false` | 是否启用 Secure Cookie |
| `INFRA.TOKEN_SALT_COMPLEXITY` | - | bcrypt salt 复杂度 |

---

## 十三、安全设计要点

1. **HttpOnly Cookie**：前端 JS 无法读取 Token，防止 XSS 窃取
2. **Refresh Token Rotation**：每次刷新生成全新 Token 对，旧 Token 即刻失效
3. **argon2 双重验证**：数据库不存明文，即使 JWT 有效也需哈希匹配
4. **重试次数限制**（运行时路径）：3 次失败后强制登出，防止无限循环
5. **SameSite=Lax Cookie**：防止 CSRF 攻击
6. **Secure Cookie（可选）**：HTTPS 传输加密

---

## 十四、代码溯源索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| AuthEvent 类型定义 | `hoppscotch-common/src/platform/auth.ts` | 37-41 |
| 自托管 Web authEvents$ 声明 | `hoppscotch-selfhost-web/src/platform/auth/web/index.ts` | 16 |
| 自托管 Web refreshToken() | 同上 | 152-173 |
| 自托管 Web setInitialUser() | 同上 | 92-150 |
| 自托管 Web signOutUser() | 同上 | 341-352 |
| 自托管 Web willBackendHaveAuthError() | 同上 | 228-230 |
| 自托管 Web onBackendGQLClientShouldReconnect() | 同上 | 232-242 |
| Admin AuthEvent 类型 | `hoppscotch-sh-admin/src/helpers/auth.ts` | 37-40 |
| Admin setInitialUser() | 同上 | 92-136 |
| GQL authExchange 配置 | `hoppscotch-common/src/helpers/backend/GQLClient.ts` | 74-118 |
| GQL 客户端重建逻辑 | 同上 | 155-190 |
| authRetryGuard 创建 | 同上 | 69 |
| authRetryGuard 实现 | `hoppscotch-common/src/helpers/retryAuthGuard.ts` | 1-78 |
| 后端刷新端点 | `hoppscotch-backend/src/auth/auth.controller.ts` | 87-100 |
| 后端刷新服务 | `hoppscotch-backend/src/auth/auth.service.ts` | 103-127, 335-363 |
| 后端 RT JWT 策略 | `hoppscotch-backend/src/auth/strategies/rt-jwt.strategy.ts` | 20-49 |
| 后端 AT JWT 策略 | `hoppscotch-backend/src/auth/strategies/jwt.strategy.ts` | 64-109 |
| 后端 Cookie 处理 | `hoppscotch-backend/src/auth/helper.ts` | 38-82 |
| 后端错误常量 | `hoppscotch-backend/src/errors.ts` | 25, 69, 601 |
| GqlAuthGuard | `hoppscotch-backend/src/guards/gql-auth.guard.ts` | 6-11 |
| 业务模块 token_refresh 订阅(settings) | `hoppscotch-selfhost-web/src/platform/settings/web/index.ts` | 33-41 |
| 业务模块 token_refresh 订阅(collections) | `hoppscotch-selfhost-web/src/platform/collections/web/index.ts` | 91-99 |
