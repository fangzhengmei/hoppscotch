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
  | { event: "probable_login"; user: HoppUser }
  | { event: "login"; user: HoppUser }
  | { event: "logout" }
  | { event: "token_refresh"; user: HoppUser }
```

通用层定义 `token_refresh` **携带 `user` 字段**。这是为 Firebase 等能在刷新时立即拿到用户信息的平台设计的。

### 2.2 自托管 Web 实际发射（hoppscotch-selfhost-web）

**文件**: `packages/hoppscotch-selfhost-web/src/platform/auth/web/index.ts:16`

```typescript
export const authEvents$ = new Subject<AuthEvent | { event: "token_refresh" }>()
```

实际发射时（`web/index.ts:164-166`）：

```typescript
authEvents$.next({ event: "token_refresh" })  // 无 user 字段
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
  | { event: 'token_refresh' }
```

Admin 后台干脆在本地类型中移除了 `probable_login` 和 `token_refresh` 的 `user` 字段。

### 2.4 事件差异对照表

| 事件 | 通用层 AuthEvent | 自托管 Web 实际发射 | Admin 实际发射 |
|------|-----------------|-------------------|---------------|
| `probable_login` | `{ event, user }` | **从未发射** | **不存在** |
| `login` | `{ event, user }` | `{ event, user }` | `{ event, user }` |
| `logout` | `{ event }` | `{ event }` | `{ event }` |
| `token_refresh` | `{ event, user }` | `{ event }` 无 user | `{ event }` 无 user |

> **结论**：通用层的 `AuthEvent.token_refresh.user` 在自托管场景下永远不会被填充。
> `probable_login` 事件在自托管 Web 中也从未被发射。

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
      e.message.includes("auth/fail") ||
      e.message.includes("jwt expired") ||
      e.extensions?.code === "UNAUTHENTICATED"
  )
}
```

> **注意**：`setInitialUser()` 和 `didAuthError()` 使用了**不同的错误识别策略**。
> 初始化时通过 GQL `errors[0].message` 精确匹配，运行时通过字符串包含判断。

---

## 四、waitProbableLoginToConfirm 精确放行条件

### 4.1 实现代码

**文件**: `packages/hoppscotch-selfhost-web/src/platform/auth/web/index.ts:258-273`

```typescript
waitProbableLoginToConfirm() {
  return new Promise<void>((resolve, reject) => {
    if (this.getCurrentUser()) {
      resolve()              // 检查点 1：已登录直接放行
    }

    if (!probableUser$.value) reject(new Error("no_probable_user"))  // 检查点 2

    const unwatch = watch(isGettingInitialUser, (val) => {           // 检查点 3
      if (val === true || val === false) {
        resolve()
        unwatch()
      }
    })
  })
},
```

### 4.2 Vue watch 的惰性行为（关键纠正）

Vue 3 的 `watch` 默认是**惰性**的（lazy），即：
- **只在源发生变化时触发回调**
- **注册时不会立即执行**

这意味着 watch 的行为完全取决于**注册时 `isGettingInitialUser` 的当前值**。

### 4.3 两种注册场景的行为差异

这是之前文档最关键的遗漏：**watch 注册时机不同，行为完全不同**。

| 注册时机 | 注册时 `isGettingInitialUser` 的值 | 何时 resolve |
|---------|---------------------------------|------------|
| **页面首次加载**（initBackendGQLClient） | `null` | `null` → `true` 变化时触发 |
| **token_refresh 事件**（GQL 客户端重建） | `true` | `true` → `false` 变化时触发 |

### 4.4 场景 A：页面首次加载时注册 watch

```
时刻 0: isGettingInitialUser = null
时刻 1: watch 注册（不立即执行，惰性）
时刻 2: setInitialUser() 开始 → isGettingInitialUser = true
        → null → true，值变化了 → 触发 watch 回调 → resolve()
```

**此时 `currentUser$` 仍为 null**，`authExchange` 获得 `AuthConfig` 后，
`willAuthError()` 检查 `currentUser$` → null → 返回 true → 触发 `refreshAuth()`。

但注意：首次加载时 `setInitialUser()` 会自己判断是否需要刷新，
如果 `willAuthError()` 也触发刷新，可能会导致 **两次并行刷新**。

### 4.5 场景 B：token_refresh 事件时注册 watch

```
时刻 0: isGettingInitialUser = true（setInitialUser 正在进行中）
时刻 1: token_refresh 事件 → GQL 客户端重建
时刻 2: watch 注册（不立即执行，惰性）
        → 当前值是 true，没有变化 → 不触发
时刻 3: 等待...
时刻 N: isGettingInitialUser = false
        → true → false，值变化了 → 触发 watch 回调 → resolve()
```

**此时 `currentUser$` 已有值**（setUser 已执行），`willAuthError()` 返回 false，
**不会触发冗余刷新**。

> **关键结论**：之前文档中描述的"冗余刷新"只可能发生在**页面首次加载**场景，
> **token_refresh 事件触发的 GQL 重建不会导致冗余刷新**，因为 watch 会等到
> `isGettingInitialUser = false` 时才 resolve，此时 `currentUser$` 已有值。

### 4.6 checkPoint 1 的作用

`if (this.getCurrentUser())` 检查是一个快速路径：
- 如果 `currentUser$` 已有值（如运行时 token_refresh 后，用户已登录但需要重建客户端），
  直接 resolve，不需要注册 watch。

---

## 五、isGettingInitialUser 三态变化时序

### 5.1 状态变量声明

**文件**: `packages/hoppscotch-selfhost-web/src/platform/auth/web/index.ts:83`

```typescript
const isGettingInitialUser: Ref<null | boolean> = ref(null)
```

### 5.2 场景一：Token 有效，无需刷新

```
performAuthInit()
  └─ await setInitialUser()
       ├─ [行93] isGettingInitialUser = true    (null → true)
       ├─ [行94] await getInitialUserDetails() → 成功
       ├─ [行139] await setUser(hoppUser)
       ├─ [行141] isGettingInitialUser = false   (true → false)
       └─ [行143] authEvents$.next("login")
```

### 5.3 场景二：Token 过期，刷新成功（递归调用）

**核心纠正**：行 116 的递归调用是 `setInitialUser()` **不带 await**。
这意味着第一次调用不会等待递归调用完成就 return 了。

```
performAuthInit()
  └─ await setInitialUser()  [第一次调用]
       │
       ├─ [行93] isGettingInitialUser = true       (null → true)
       ├─ [行94] await getInitialUserDetails() → "Unauthorized"
       ├─ [行113] await refreshToken()
       │    ├─ GET /auth/refresh → 200 OK
       │    ├─ [行164] authEvents$.next("token_refresh")
       │    └─ return true
       │
       ├─ [行116] setInitialUser()  [第二次调用，不带 await！]
       │    │
       │    ├─ [行93] isGettingInitialUser = true   (true → true，无变化)
       │    ├─ [行94] await getInitialUserDetails() → 成功
       │    ├─ [行139] await setUser(hoppUser)
       │    ├─ [行141] isGettingInitialUser = false (true → false)
       │    └─ [行143] authEvents$.next("login")
       │
       └─ [行123] return  ← 第一次调用结束（不等待递归调用）
```

**关键发现**：

1. 第一次调用在行 123 `return` 后结束
2. 第二次调用是异步执行的
3. `performAuthInit()` 的 `await` 只等到了第一次调用的 return，
   此时 `isGettingInitialUser` 仍为 `true`，`currentUser$` 仍为 null
4. 但递归调用的 `await getInitialUserDetails()` 已经在事件循环中排队

| 时刻 | 调用栈 | isGettingInitialUser | currentUser$ |
|-----|-------|---------------------|-------------|
| T0 | performAuthInit 开始 | null | null |
| T1 | setInitialUser(1) 行93 | true | null |
| T2 | getInitialUserDetails 返回 "Unauthorized" | true | null |
| T3 | refreshToken 返回 true | true | null |
| T4 | authEvents$.next("token_refresh") | true | null |
| T5 | setInitialUser(2) 行93（无 await） | true（无变化） | null |
| T6 | setInitialUser(1) 行123 return | true | null |
| T7 | performAuthInit 的 await 结束 | true | null |
| T8 | setInitialUser(2) getInitialUserDetails 返回成功 | true | null |
| T9 | setUser(hoppUser) | true | HoppUser |
| T10 | isGettingInitialUser = false | false | HoppUser |
| T11 | authEvents$.next("login") | false | HoppUser |

### 5.4 场景三：Token 过期，刷新失败

```
performAuthInit()
  └─ await setInitialUser()
       ├─ [行93] isGettingInitialUser = true
       ├─ [行94] await getInitialUserDetails() → "Unauthorized"
       ├─ [行113] await refreshToken() → return false
       ├─ [行118] await setUser(null)
       ├─ [行119] isGettingInitialUser = false
       ├─ [行120] await logout()
       └─ [行123] return
```

### 5.5 场景四：无 Cookie

```
performAuthInit()
  └─ await setInitialUser()
       ├─ [行93] isGettingInitialUser = true
       ├─ [行94] await getInitialUserDetails() → "cookies_not_found"
       ├─ [行100] await setUser(null)
       ├─ [行101] isGettingInitialUser = false
       └─ [行102] return
```

---

## 六、token_refresh 事件的完整副作用链

### 6.1 事件订阅者的执行顺序

`authEvents$.next("token_refresh")` 会**同步**通知所有订阅者，顺序如下：

```
authEvents$.next("token_refresh")
  │
  ├─ 订阅者 1：onBackendGQLClientShouldReconnect 回调
  │    ├─ currentUser = platform.auth.getCurrentUser() → null（关键！）
  │    │
  │    ├─ currentUser && subscriptionClient → false，不关闭 WebSocket
  │    ├─ currentUser && !subscriptionClient → false，不创建 WebSocket
  │    ├─ !currentUser && subscriptionClient → 可能 true，关闭 WebSocket
  │    │
  │    └─ client.value = createHoppClient()  ← 重建 GQL 客户端
  │         └─ authExchange(async ...) 同步执行到第一个 await
  │              ├─ probableUser = platform.auth.getProbableUser() → 旧用户
  │              ├─ probableUser !== null → 进入等待
  │              └─ waitProbableLoginToConfirm()
  │                   ├─ getCurrentUser() → null
  │                   ├─ probableUser$ 有值 → 不 reject
  │                   └─ watch(isGettingInitialUser) 注册
  │                        → 当前值是 true，无变化 → 不触发
  │                        → 等待 true → false 变化
  │
  ├─ 订阅者 2：settings syncer
  │    └─ settingsSyncer.startListeningToSubscriptions()
  │         └─ startSubscriptions?.()
  │              └─ 调用 setupSubscriptions
  │                   └─ 调用 runGQLSubscription
  │                        └─ client.value!.executeSubscription(...)
  │                            → 但 authExchange 的初始化 Promise 还未 resolve
  │                            → 请求被 urql 缓冲，等待初始化完成
  │
  ├─ 订阅者 3：collections syncer
  │    └─ collectionsSyncer.startListeningToSubscriptions()
  │         └─ 同上，Subscription 请求被缓冲
  │
  ├─ 订阅者 4：environments syncer
  │    └─ 同上
  │
  └─ 订阅者 5：history syncer
       └─ 同上
```

### 6.2 urql 异步 exchange 的缓冲机制

`authExchange` 是一个异步 exchange，urql 会：
1. 调用初始化函数，获得 Promise
2. **缓冲**所有进入的请求，直到 Promise resolve
3. Promise resolve 后，才开始处理请求（调用 `addAuthToOperation` / `willAuthError` 等）

所以 `syncer.startListeningToSubscriptions()` 发起的 Subscription 请求不会立即发送，
它们会被缓冲直到 `waitProbableLoginToConfirm()` resolve。

### 6.3 请求实际发送的时机

```
时刻 0: token_refresh 事件发射
时刻 1: GQL 客户端重建，authExchange 初始化开始
时刻 2: watch 注册（isGettingInitialUser = true）
时刻 3: syncer.startListeningToSubscriptions() 调用，请求被缓冲
时刻 4: setInitialUser(2) 继续执行...
时刻 5: setUser(hoppUser) → currentUser$ = HoppUser
时刻 6: isGettingInitialUser = false  →  true → false，值变化了！
时刻 7: watch 触发 → resolve()
时刻 8: authExchange 初始化完成，开始处理缓冲的请求
时刻 9: willAuthError() 检查 currentUser$ → HoppUser ≠ null → 返回 false
时刻 10: 请求正常发送，不会触发 refreshAuth()
```

**统一结论**：**token_refresh 事件不会导致冗余刷新**。
因为：
1. watch 注册在 `isGettingInitialUser = true` 时，不会立即触发
2. watch 等到 `isGettingInitialUser = false` 时才 resolve
3. 此时 `currentUser$` 已有值，`willAuthError()` 返回 false

> 之前文档中"冗余刷新"的描述只适用于**页面首次加载**场景，
> 不适用于 token_refresh 事件场景。

---

## 七、setInitialUser 递归调用与 token_refresh、login 事件的精确时序

### 7.1 刷新成功路径的完整时序（含调用栈分析）

```typescript
// 第一次调用（由 performAuthInit 触发）
async function setInitialUser() {                   // 调用帧 #1
  isGettingInitialUser.value = true                 // [1] null → true
  const res = await getInitialUserDetails()          // [2] 发起网络请求，让出执行

  // ... 网络请求返回，微任务恢复 ...

  if (error && error.message === "Unauthorized") {   // [3]
    const isRefreshSuccess = await refreshToken()     // [4] 发起网络请求，让出执行

    // ... 网络请求返回，微任务恢复 ...

    // refreshToken 内部：
    //   authEvents$.next({ event: "token_refresh" })  // [5] 同步通知所有订阅者
    //     → 订阅者 1: GQL 客户端重建
    //          → authExchange 初始化到 waitProbableLoginToConfirm
    //              → watch 注册（isGettingInitialUser = true，等待 false）
    //     → 订阅者 2-5: syncer.startListeningToSubscriptions()
    //          → Subscription 请求被 urql 缓冲
    //   return true                                    // [6]

    if (isRefreshSuccess) {                          // [7] true
      setInitialUser()                               // [8] 递归调用，无 await！
    }                                                // [8a] 递归调用同步执行到第一个 await

    // 第二次调用（递归）
    // async function setInitialUser() {              // 调用帧 #2
    //   isGettingInitialUser.value = true            // [9] true → true，无变化
    //   const res = await getInitialUserDetails()     // [10] 发起网络请求，让出执行
    // }

    return                                           // [11] 第一次调用结束
  }
}
```

### 7.2 事件与状态更新的先后关系

| 顺序 | 操作 | currentUser$ | isGettingInitialUser | watch 状态 |
|-----|------|-------------|---------------------|-----------|
| 1 | setInitialUser(1) 开始 | null | true | - |
| 2 | refreshToken 网络请求完成 | null | true | - |
| 3 | authEvents$.next("token_refresh") | null | true | 注册，等待 |
| 4 | Subscription 请求被缓冲 | null | true | 等待 |
| 5 | setInitialUser(2) 开始（无 await） | null | true | 等待 |
| 6 | setInitialUser(1) return | null | true | 等待 |
| 7 | getInitialUserDetails(2) 网络请求完成 | null | true | 等待 |
| 8 | setUser(hoppUser) | **HoppUser** | true | 等待 |
| 9 | isGettingInitialUser = false | HoppUser | **false** | 触发 resolve |
| 10 | authExchange 初始化完成 | HoppUser | false | 完成 |
| 11 | 缓冲的 Subscription 开始发送 | HoppUser | false | - |
| 12 | authEvents$.next("login") | HoppUser | false | - |

**关键结论**：

1. `token_refresh` 事件在递归调用开始**之前**发射
2. `token_refresh` 事件发射时，`currentUser$` 必然为 null
3. `token_refresh` 事件触发的 watch 会等到 `isGettingInitialUser = false` 才 resolve
4. `login` 事件在 `currentUser$` 更新和 `isGettingInitialUser = false` **之后**发射
5. **watch resolve 时 `currentUser$` 已有值**，不会触发冗余刷新

---

## 八、竞态分析（统一结论）

### 8.1 确定的执行顺序，不存在多种场景

由于 JavaScript 单线程模型和 RxJS `Subject.next()` 的同步通知特性，
`token_refresh` 事件的执行顺序是**完全确定**的，不存在"场景 A / 场景 B"两种可能。

**唯一确定的时序**：

```
authEvents$.next("token_refresh")          ← 同步开始
  ├─ 订阅者 1：GQL 客户端重建               ← 同步
  │    └─ authExchange 初始化到 watch 注册   ← 同步执行到第一个 await
  ├─ 订阅者 2：settings syncer              ← 同步
  ├─ 订阅者 3：collections syncer           ← 同步
  ├─ 订阅者 4：environments syncer          ← 同步
  └─ 订阅者 5：history syncer               ← 同步
return true                                 ← refreshToken 返回
setInitialUser() [递归]                     ← 此时才开始递归
```

订阅者回调全部执行完之前，`refreshToken` 不会 return，递归也不会开始。
这是一个严格的单线程顺序，不存在任何不确定性。

### 8.2 唯一可能的竞态：页面首次加载

**唯一可能发生冗余刷新的场景**是**页面首次加载**：

```
页面加载 → initBackendGQLClient()
  ├─ client.value = createHoppClient()
  │    └─ authExchange(async ...)
  │         ├─ probableUser ≠ null（localStorage 有旧值）
  │         └─ waitProbableLoginToConfirm()
  │              ├─ getCurrentUser() → null
  │              └─ watch 注册（isGettingInitialUser = null）
  └─ performAuthInit()
       └─ setInitialUser() 开始
            ├─ isGettingInitialUser = true  → null → true，值变化！
            └─ watch 触发 → resolve()
                  → authExchange 初始化完成
                  → 第一个请求时 willAuthError() → currentUser$ = null → true
                  → 触发 refreshAuth()
```

但 `setInitialUser()` 自己也会检查是否需要刷新，可能导致：
- `setInitialUser()` 发起一次刷新（通过 `refreshToken()`）
- `authExchange` 的 `willAuthError()` 也发起一次刷新（通过 `refreshAuth()` → `refreshToken()`）

这种情况下可能出现**两次并行的 `/auth/refresh` 请求**，但功能上无害。

### 8.3 token_refresh 场景：无冗余刷新

如前所述，`token_refresh` 事件触发的 GQL 重建不会导致冗余刷新，
因为 watch 会等到 `isGettingInitialUser = false` 才 resolve，此时 `currentUser$` 已有值。

### 8.4 双重客户端重建

初始化路径刷新成功时会触发两次 `createHoppClient()`：

1. **第一次**：`token_refresh` 事件
   - `currentUser$ = null` → WebSocket 不创建
   - authExchange 初始化时 watch 注册，等待 `false`

2. **第二次**：`login` 事件
   - `currentUser$ = HoppUser` → WebSocket 创建
   - checkPoint 1 直接 resolve，不需要 watch

第一次重建的 GQL 客户端是短暂的，它的 authExchange 可能还在等待 watch resolve，
第二次重建就覆盖了它。这不是"竞态问题"，而是"重复工作"问题。

---

## 九、两条刷新路径的完整调用顺序与状态变化

### 9.1 路径 A：应用初始化刷新（setInitialUser）

**触发时机**：页面加载/刷新时 `performAuthInit()` 调用

#### 成功路径（Token 有效）

```
performAuthInit()
  ├─ 从 localStorage 读取 login_state → probableUser$.next(旧用户)
  └─ await setInitialUser()
       ├─ isGettingInitialUser = true
       ├─ await getInitialUserDetails() → GQL /me 查询成功
       ├─ await setUser(hoppUser)
       │    ├─ currentUser$.next(hoppUser)
       │    ├─ probableUser$.next(hoppUser)
       │    └─ persistence.setLocalConfig(...)
       ├─ isGettingInitialUser = false
       └─ authEvents$.next({ event: "login", user: hoppUser })
```

此时 `performAuthInit` 的 `await` 结束时，初始化已完成。

#### 失败路径 A1：无 Cookie（cookies_not_found）

```
setInitialUser()
  ├─ await getInitialUserDetails() → "auth/cookies_not_found"
  ├─ await setUser(null)
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
setInitialUser() [第一次调用，有 await]
  ├─ isGettingInitialUser = true
  ├─ await getInitialUserDetails() → "Unauthorized"
  ├─ await refreshToken()
  │    ├─ GET /auth/refresh → 200 OK
  │    ├─ authEvents$.next("token_refresh")     ← 事件 1
  │    │    → GQL 客户端重建（第一次）
  │    │        → authExchange 初始化，watch 注册（等待 false）
  │    │    → syncers.startListeningToSubscriptions()
  │    │        → Subscription 请求被缓冲
  │    └─ return true
  ├─ setInitialUser() [第二次调用，无 await！]   ← 递归
  │    ├─ isGettingInitialUser = true (无变化)
  │    ├─ await getInitialUserDetails() → 成功
  │    ├─ await setUser(hoppUser)
  │    │    ├─ currentUser$.next(hoppUser)
  │    │    ├─ probableUser$.next(hoppUser)
  │    │    └─ persistence.setLocalConfig(...)
  │    ├─ isGettingInitialUser = false           ← 触发 watch resolve
  │    │    → authExchange 初始化完成
  │    │    → 缓冲的 Subscription 请求开始发送
  │    │    → willAuthError() 返回 false
  │    └─ authEvents$.next("login", user)        ← 事件 2
  │         → GQL 客户端重建（第二次）
  │         → WebSocket 创建
  │         → authRetryGuard.reset()
  └─ return                                      ← 第一次调用结束
```

**状态变化时序**：

| 时间点 | currentUser$ | probableUser$ | isGettingInitialUser | performAuthInit 已返回？ |
|-------|-------------|--------------|---------------------|----------------------|
| performAuthInit 开始 | null | 旧用户 | null | 否 |
| setInitialUser(1) 开始 | null | 旧用户 | true | 否 |
| refreshToken 成功 | null | 旧用户 | true | 否 |
| token_refresh 事件 | null | 旧用户 | true | 否 |
| setInitialUser(2) 开始 | null | 旧用户 | true | 否 |
| **setInitialUser(1) return** | **null** | **旧用户** | **true** | **是** |
| setUser(hoppUser) | HoppUser | HoppUser | true | 是 |
| isGettingInitialUser = false | HoppUser | HoppUser | false | 是 |
| watch resolve | HoppUser | HoppUser | false | 是 |
| login 事件 | HoppUser | HoppUser | false | 是 |

#### 失败路径 A4：Access Token 过期（Unauthorized）→ 刷新失败

```
setInitialUser()
  ├─ await getInitialUserDetails() → "Unauthorized"
  ├─ await refreshToken() → return false
  ├─ await setUser(null)
  │    ├─ currentUser$.next(null)
  │    ├─ probableUser$.next(null)
  │    └─ persistence.setLocalConfig("login_state", "null")
  ├─ isGettingInitialUser = false
  ├─ await logout()
  └─ 【不发射 logout 事件】
```

> 初始化路径刷新失败时，**不发射 `logout` 事件**。
> GQL 客户端不会被重建，可能导致旧连接残留。

### 9.2 路径 B：GQL 运行时刷新（authExchange + retryGuard）

**触发时机**：已登录状态下 GQL 请求遇到认证错误

#### 触发条件

1. `willAuthError()`：`currentUser$` 为 null → 请求前刷新
2. `didAuthError()`：响应包含 `"auth/fail"` / `"jwt expired"` / `"UNAUTHENTICATED"` → 请求后刷新

#### 成功路径

```
GQL 请求 → didAuthError() = true
  └─ refreshAuth()
       └─ authRetryGuard.execute(() => refreshAuthToken())
            ├─ refreshToken()
            │    ├─ GET /auth/refresh → 200 OK
            │    ├─ authEvents$.next("token_refresh")
            │    │    → GQL 客户端重建（checkPoint 1 直接 resolve，无 watch）
            │    │    → WebSocket 关闭并重建
            │    └─ return true
            ├─ failCount = 0
            └─ return true → authExchange 重试原始请求
```

> 运行时刷新时 `currentUser$` 保持旧值（不为 null），
> `waitProbableLoginToConfirm()` 的 checkPoint 1 直接 resolve，
> 不需要注册 watch，也不会等待。

#### 失败路径：第 N 次（N < 3）失败

```
authRetryGuard.execute(...)
  ├─ refreshToken() → false
  ├─ failCount++
  └─ return false → authExchange 放弃，请求失败
```

#### 失败路径：第 3 次失败（耗尽重试）

```
authRetryGuard.execute(...)
  ├─ refreshToken() → false
  ├─ failCount = 3
  ├─ isExhausted = true
  ├─ onExhausted() → signOutUser()
  │    ├─ await logout()   → GET /auth/logout
  │    ├─ probableUser$.next(null)
  │    ├─ currentUser$.next(null)
  │    ├─ persistence.removeLocalConfig("login_state")
  │    └─ authEvents$.next("logout")
  │         → GQL 客户端重建（无用户）
  │         → WebSocket 关闭
  │         → syncers.stopListening()
  └─ return false
```

#### 耗尽后的后续请求

```
authRetryGuard.execute(...)
  ├─ isExhausted = true → 直接 return false
  └─ 不再调用 refreshToken
```

只有 `login` 事件触发 `authRetryGuard.reset()` 后才恢复。

---

## 十、两条路径的关键差异总结

| 维度 | 路径 A：初始化刷新 | 路径 B：GQL 运行时刷新 |
|------|------------------|---------------------|
| **触发入口** | `performAuthInit()` → `setInitialUser()` | urql `authExchange` |
| **错误识别** | `errors[0].message` 精确匹配 | `didAuthError()` 字符串包含 |
| **刷新调用** | 直接调用 `refreshToken()` | `authRetryGuard.execute()` 间接调用 |
| **重试上限** | 无限制（递归） | 最多 3 次 |
| **刷新成功后** | 递归 `setInitialUser()` 重新获取用户 | 仅重建 GQL 客户端 |
| **刷新失败后** | `setUser(null)` + `logout()`（不发事件） | `failCount++`，3 次后 `signOutUser()`（发 `logout` 事件） |
| **递归调用** | **无 await**（fire-and-forget） | 不涉及 |
| **performAuthInit 返回时** | 可能初始化尚未完成 | 不涉及 |
| **`currentUser$` 刷新成功后** | 仍为 null（等待递归完成） | 保持旧值 |
| **`waitProbableLoginToConfirm`** | watch 注册在 true，等待 false | checkPoint 1 直接 resolve |
| **GQL 客户端重建次数** | 2 次（token_refresh + login） | 1 次（token_refresh） |
| **冗余刷新风险** | 页面首次加载时可能 | 无 |

---

## 十一、后端 Token 刷新的完整验证链路

### 11.1 刷新端点

```
GET /api/v1/auth/refresh
  │
  ├─ RTJwtAuthGuard (passport jwt-refresh 策略)
  │    ├─ 从 Cookie 提取 refresh_token
  │    ├─ JWT 签名验证
  │    ├─ JWT 过期检查
  │    └─ validate(payload) → 查询用户
  │
  ├─ authService.refreshAuthTokens(refreshToken, user)
  │    ├─ 验证用户存在
  │    ├─ argon2.verify(dbHashedToken, refreshToken)  ← 二次验证
  │    └─ generateAuthTokens(userUid)
  │
  └─ authCookieHandler(res, newTokens)
       ├─ Set-Cookie: access_token=xxx
       └─ Set-Cookie: refresh_token=xxx
```

### 11.2 双重验证机制

1. **第一层**：Passport-JWT 验证签名和过期时间
2. **第二层**：`argon2.verify()` 验证 Token 哈希与数据库匹配

第二层的作用：即使 JWT 有效，但如果用户已在新设备登录（新 RT 覆盖旧哈希），
旧 RT 签名正确但哈希不匹配，会被拒绝。

### 11.3 Refresh Token Rotation

每次刷新成功后：
- 后端生成全新 Access Token + Refresh Token 对
- 新 RT 的 argon2 哈希覆盖数据库中旧值
- 旧 RT 即刻失效

---

## 十二、token_refresh 事件在各业务模块的副作用

**文件**: `packages/hoppscotch-selfhost-web/src/platform/*/web/index.ts`

所有业务模块对 `token_refresh` 的处理一致：

```typescript
authEvents$.subscribe((event) => {
  if (event.event == "login" || event.event == "token_refresh") {
    syncer.startListeningToSubscriptions()
  }
  if (event.event == "logout") {
    syncer.stopListeningToSubscriptions()
  }
})
```

| 业务模块 | 文件 | 订阅行为 |
|---------|------|---------|
| Settings | `platform/settings/web/index.ts:34` | `settingsSyncer.startListeningToSubscriptions()` |
| Collections | `platform/collections/web/index.ts:92` | `collectionsSyncer.startListeningToSubscriptions()` |
| Environments | `platform/environments/web/index.ts:49` | 同上模式 |
| History | `platform/history/web/index.ts:62` | 同上模式 |

---

## 十三、GQL 客户端重建的决策逻辑

**文件**: `packages/hoppscotch-common/src/helpers/backend/GQLClient.ts:155-190`

`initBackendGQLClient()` 注册 `onBackendGQLClientShouldReconnect` 回调，
在 `login`/`logout`/`token_refresh` 事件时触发：

```
回调触发
  ├─ currentUser 有值 && subscriptionClient 存在
  │    → 关闭旧 WebSocket
  ├─ currentUser 有值 && subscriptionClient 不存在
  │    → 创建新 WebSocket
  ├─ currentUser 为空 && subscriptionClient 存在
  │    → 关闭 WebSocket，置 null
  └─ client.value = createHoppClient()  ← 重建 GQL 客户端
```

### authExchange 初始化逻辑

```typescript
authExchange(async (): Promise<AuthConfig> => {
  const probableUser = platform.auth.getProbableUser()
  if (probableUser !== null)
    await platform.auth.waitProbableLoginToConfirm()
  // ...
})
```

| 场景 | probableUser | waitProbableLoginToConfirm 行为 | resolve 时机 |
|------|-------------|-------------------------------|------------|
| 页面首次加载 | 旧用户（非 null） | watch 注册（null → true 触发） | isGettingInitialUser = true 时 |
| token_refresh（初始化路径） | 旧用户（非 null） | watch 注册（true → false 触发） | isGettingInitialUser = false 时 |
| token_refresh（运行时路径） | HoppUser | checkPoint 1 直接 resolve | 立即 |
| logout 后重建 | null | 跳过等待 | 立即 |
| 从未登录 | null | 跳过等待 | 立即 |

---

## 十四、完整状态变化矩阵

### 14.1 初始化路径（路径 A）

| 时刻 | currentUser$ | probableUser$ | isGettingInitialUser | performAuthInit 已返回 | 发射的事件 |
|-----|-------------|--------------|---------------------|----------------------|----------|
| T0 performAuthInit 开始 | null | 旧用户 | null | 否 | - |
| T1 setInitialUser(1) 开始 | null | 旧用户 | true | 否 | - |
| T2 Token 有效成功 | HoppUser | HoppUser | false | 是 | `login` |
| T2 无 Cookie 失败 | null | null | false | 是 | **无** |
| T2 用户不存在失败 | null | null | false | 是 | **无** |
| T3 refreshToken 成功 | null | 旧用户 | true | 否 | `token_refresh` |
| T4 setInitialUser(1) return | **null** | **旧用户** | **true** | **是** | - |
| T5 setUser(hoppUser) | HoppUser | HoppUser | true | 是 | - |
| T6 isGettingInitialUser = false | HoppUser | HoppUser | false | 是 | - |
| T7 watch resolve | HoppUser | HoppUser | false | 是 | - |
| T8 login 事件 | HoppUser | HoppUser | false | 是 | `login` |
| T2 刷新失败 | null | null | false | 是 | **无** |

### 14.2 GQL 运行时路径（路径 B）

| 时刻 | currentUser$ | probableUser$ | authRetryGuard | 发射的事件 |
|-----|-------------|--------------|---------------|----------|
| 刷新成功 | 不变(HoppUser) | 不变 | failCount=0 | `token_refresh` |
| 刷新失败(1-2次) | 不变(HoppUser) | 不变 | failCount++ | **无** |
| 刷新失败(3次) | null | null | isExhausted=true | `logout` |
| 耗尽后请求 | null | null | 直接返回 false | **无** |

---

## 十五、时序图：初始化路径刷新成功

```
Frontend                                      Backend
    │                                            │
    │  performAuthInit()                         │
    │  probableUser$ ← localStorage              │
    │                                            │
    │  await setInitialUser() [调用#1]           │
    │  ├─ isGettingInitialUser = true            │
    │  └─── GQL /me ──────────────────────────▶ │
    │  ◀─── errors: "Unauthorized" ───────────── │
    │                                            │
    │  await refreshToken()                      │
    │  ──── GET /auth/refresh ─────────────────▶ │
    │                                            │  验证 RT ✅
    │                                            │  生成新 Token
    │  ◀─── 200 + Set-Cookie ────────────────── │
    │                                            │
    │  authEvents$.next("token_refresh")         │
    │    ├─ GQL 客户端重建                       │
    │    │  └─ authExchange 初始化               │
    │    │     └─ waitProbableLoginToConfirm()   │
    │    │        └─ watch 注册（等待 false）    │
    │    └─ syncers.startListeningToSubscriptions()
    │         └─ Subscription 请求被缓冲         │
    │                                            │
    │  setInitialUser() [调用#2，无 await]       │
    │  ├─ isGettingInitialUser = true (无变化)   │
    │  └─── GQL /me ──────────────────────────▶ │
    │                                            │
    │  setInitialUser(1) return                  │  ← performAuthInit 的 await 结束
    │  (performAuthInit 返回，但初始化未完成！)    │
    │                                            │
    │  ◀─── { data: { me: {...} } } ─────────── │
    │                                            │
    │  setUser(hoppUser)                         │
    │  currentUser$ ← hoppUser                   │
    │  isGettingInitialUser = false              │  ← watch 触发 resolve
    │                                            │  ← 缓冲的请求开始发送
    │                                            │  ← willAuthError() = false
    │  authEvents$.next("login")                 │
    │    ├─ GQL 客户端重建（第二次）              │
    │    ├─ WebSocket 创建                       │
    │    └─ authRetryGuard.reset()               │
    │                                            │
```

---

## 十六、时序图：GQL 运行时刷新耗尽

```
Frontend                                      Backend
    │                                            │
    │  GQL 请求 (已登录)                         │
    │  ──── GQL query ────────────────────────▶ │
    │  ◀─── errors: "jwt expired" ───────────── │
    │                                            │
    │  didAuthError() → true                     │
    │  authRetryGuard.execute() → refreshToken() │
    │  ──── GET /auth/refresh ────────────────▶ │
    │  ◀─── 401 ─────────────────────────────── │
    │  failCount = 1                             │
    │                                            │
    │  [下一次 GQL 请求]                         │
    │  ──── GQL query ────────────────────────▶ │
    │  ◀─── errors: "auth/fail" ─────────────── │
    │  failCount = 2                             │
    │                                            │
    │  [第三次 GQL 请求]                         │
    │  ──── GQL query ────────────────────────▶ │
    │  ◀─── errors: "UNAUTHENTICATED" ───────── │
    │  failCount = 3                             │
    │  isExhausted = true                        │
    │  signOutUser()                             │
    │  ──── GET /auth/logout ─────────────────▶ │
    │  currentUser$ = null                       │
    │  authEvents$.next("logout")                │
    │    ├─ GQL 客户端重建 (无用户)              │
    │    ├─ WebSocket 关闭                       │
    │    └─ syncers.stopListening()              │
    │                                            │
    │  [后续请求]                                │
    │  authRetryGuard → false (不再刷新)         │
    │                                            │
```

---

## 十七、配置项说明

| 配置项 | 默认值 | 说明 |
|-------|-------|------|
| `INFRA.JWT_SECRET` | - | JWT 签名密钥 |
| `INFRA.ACCESS_TOKEN_VALIDITY` | `86400000` | Access Token 有效期 (ms，1天) |
| `INFRA.REFRESH_TOKEN_VALIDITY` | `604800000` | Refresh Token 有效期 (ms，7天) |
| `INFRA.ALLOW_SECURE_COOKIES` | `false` | 是否启用 Secure Cookie |

---

## 十八、安全设计要点

1. **HttpOnly Cookie**：前端 JS 无法读取 Token，防止 XSS 窃取
2. **Refresh Token Rotation**：每次刷新生成全新 Token 对，旧 Token 即刻失效
3. **argon2 双重验证**：数据库不存明文，即使 JWT 有效也需哈希匹配
4. **重试次数限制**（运行时路径）：3 次失败后强制登出
5. **SameSite=Lax Cookie**：防止 CSRF 攻击
6. **Secure Cookie（可选）**：HTTPS 传输加密

---

## 十九、代码溯源索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| AuthEvent 类型定义 | `hoppscotch-common/src/platform/auth.ts` | 37-41 |
| authEvents$ 声明 | `hoppscotch-selfhost-web/src/platform/auth/web/index.ts` | 16 |
| isGettingInitialUser | 同上 | 83 |
| setUser() | 同上 | 85-90 |
| setInitialUser() | 同上 | 92-150 |
| refreshToken() | 同上 | 152-173 |
| willBackendHaveAuthError() | 同上 | 228-230 |
| onBackendGQLClientShouldReconnect() | 同上 | 232-242 |
| performAuthInit() | 同上 | 250-256 |
| waitProbableLoginToConfirm() | 同上 | 258-273 |
| signOutUser() | 同上 | 341-352 |
| refreshAuthToken() | 同上 | 354-356 |
| GQL authExchange 配置 | `hoppscotch-common/src/helpers/backend/GQLClient.ts` | 74-118 |
| initBackendGQLClient() | 同上 | 155-190 |
| authRetryGuard 创建 | 同上 | 69 |
| runGQLSubscription() | 同上 | 274-333 |
| authRetryGuard 实现 | `hoppscotch-common/src/helpers/retryAuthGuard.ts` | 1-78 |
| Admin AuthEvent 类型 | `hoppscotch-sh-admin/src/helpers/auth.ts` | 37-40 |
| 后端刷新端点 | `hoppscotch-backend/src/auth/auth.controller.ts` | 87-100 |
| 后端刷新服务 | `hoppscotch-backend/src/auth/auth.service.ts` | 103-127, 335-363 |
| 后端 RT JWT 策略 | `hoppscotch-backend/src/auth/strategies/rt-jwt.strategy.ts` | 20-49 |
| 后端 AT JWT 策略 | `hoppscotch-backend/src/auth/strategies/jwt.strategy.ts` | 64-109 |
| 后端 Cookie 处理 | `hoppscotch-backend/src/auth/helper.ts` | 38-82 |
| GqlAuthGuard | `hoppscotch-backend/src/guards/gql-auth.guard.ts` | 6-11 |
