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
      resolve()              // 检查点 1
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

### 4.2 放行条件的精确语义

**关键纠正**：`val === true` 时 resolve 并不意味着"认证已确认"。
`true` 表示初始化仍在进行中，`currentUser$` 必然为 null。

| `isGettingInitialUser` 值 | 含义 | `currentUser$` 可能值 | resolve 后效果 |
|--------------------------|------|---------------------|--------------|
| `null` | 初始化尚未开始 | null | 不触发 watch |
| `true` | 初始化进行中 | **必定 null** | resolve 但 `willAuthError()` 返回 true |
| `false` | 初始化已完成 | HoppUser 或 null | resolve，状态已确定 |

**`true` 放行的实际后果**：

当 `authExchange` 初始化时调用 `waitProbableLoginToConfirm()`，
如果在 `isGettingInitialUser = true` 阶段 resolve，`authExchange` 立即获得 `AuthConfig`，
其中 `willAuthError()` 检查 `currentUser$` → 仍为 null → 返回 true →
`authExchange` 在第一个请求前就会调用 `refreshAuth()`。

这解释了初始化路径中 `token_refresh` 后的**冗余刷新**现象：
不是因为"竞态"，而是因为 `waitProbableLoginToConfirm` 在 `true` 阶段就放行了。

### 4.3 `null` → `true` 的触发时机问题

`watch(isGettingInitialUser)` 只在**值变化**时触发。
`isGettingInitialUser` 从 `null` → `true` 是第一次变化，
如果 `watch` 是在 `null` 阶段注册的，这次变化会触发 resolve。

但如果 `watch` 注册时 `isGettingInitialUser` 已经是 `true`（例如在递归调用期间），
则 `true` → `true` **不会触发**，watch 会一直等到 `false` 变化。

### 4.4 检查点 1 和检查点 3 的重叠

如果 `waitProbableLoginToConfirm` 被调用时 `isGettingInitialUser = false`（初始化已完成）：
- 检查点 1：`currentUser$` 有值 → resolve（已登录场景）
- 检查点 1：`currentUser$` 为 null → 不 resolve
- 检查点 3：`isGettingInitialUser` 已经是 `false`，但 watch 只监听**变化**，
  不会因为当前值就是 `false` 而立即触发

**这意味着**：如果初始化已完成且用户未登录（`currentUser$ = null`，`isGettingInitialUser = false`），
`waitProbableLoginToConfirm` 会**永远挂起**，因为：
- 检查点 1 不满足（`currentUser$` 为 null）
- 检查点 2 不满足（`probableUser$` 可能有旧值）
- 检查点 3 的 watch 永远不会触发（`isGettingInitialUser` 不会从 `false` 再变化）

> **实际影响**：这个死挂不会发生在正常流程中，因为 `authExchange` 只在
> `probableUser !== null` 时才调用 `waitProbableLoginToConfirm`。
> 如果 `probableUser` 为 null（用户从未登录），`authExchange` 跳过等待直接返回 `AuthConfig`。
> 但在 `setUser(null)` 将 `probableUser$` 清空之后、初始化完成之前注册的 watch 可能遇到此问题。

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
但 `performAuthInit()` 中 `await setInitialUser()` 只等待第一次调用，
递归调用是"fire-and-forget"。

```
performAuthInit()
  └─ await setInitialUser()  [第一次调用]
       │
       ├─ [行93] isGettingInitialUser = true       (null → true)
       ├─ [行94] await getInitialUserDetails() → "Unauthorized"
       ├─ [行113] await refreshToken()
       │    ├─ GET /auth/refresh → 200 OK
       │    ├─ [行164] authEvents$.next("token_refresh")  ← 事件发射
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
2. 第二次调用是异步执行的（`await getInitialUserDetails()` 会在微任务中继续）
3. `performAuthInit()` 的 `await` 只等到了第一次调用的 return，
   此时 `isGettingInitialUser` 仍为 `true`，`currentUser$` 仍为 null
4. 但递归调用的 `await getInitialUserDetails()` 已经在事件循环中排队

**这意味着**：`performAuthInit()` 返回时，初始化**可能尚未完成**。

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

此处 `performAuthInit()` 返回时初始化已完成。

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

此处 `performAuthInit()` 返回时初始化已完成。

---

## 六、setInitialUser 递归调用与 token_refresh、login 事件的精确时序

### 6.1 刷新成功路径的完整时序（含调用栈分析）

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
    //     → onBackendGQLClientShouldReconnect 回调同步执行
    //       → createHoppClient() 同步执行
    //         → authExchange(async () => { ... }) 同步执行到第一个 await
    //           → waitProbableLoginToConfirm() 同步执行
    //             → getCurrentUser() → null
    //             → probableUser$ 有值 → 不 reject
    //             → watch(isGettingInitialUser) 注册
    //                → isGettingInitialUser 当前为 true，无变化，不触发
    //           → await watch 等待... (Promise 挂起)
    //     → 业务 syncers.startListening() 同步执行
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

**时序要点**：

- [5] `token_refresh` 事件的副作用在 `refreshToken()` 内同步执行，
  此时递归调用 [8] 尚未发生
- [8] 递归调用**不带 await**，所以 [9][10] 同步执行到 `await` 后让出
- [11] 第一次调用的 `return` 在递归调用让出后执行
- [10] 递归调用的网络请求在后续微任务中完成

### 6.2 事件与状态更新的先后关系

| 顺序 | 操作 | currentUser$ | isGettingInitialUser | 调用帧 |
|-----|------|-------------|---------------------|-------|
| 1 | setInitialUser(1) 开始 | null | true | #1 |
| 2 | refreshToken 网络请求完成 | null | true | #1 |
| 3 | authEvents$.next("token_refresh") | null | true | #1 |
| 4 | GQL 客户端重建 + watch 注册 | null | true | #1 |
| 5 | setInitialUser(2) 开始（无 await） | null | true | #2 |
| 6 | setInitialUser(1) return | null | true | #1 结束 |
| 7 | getInitialUserDetails(2) 网络请求完成 | null | true | #2 |
| 8 | setUser(hoppUser) | **HoppUser** | true | #2 |
| 9 | isGettingInitialUser = false | HoppUser | **false** | #2 |
| 10 | watch 触发 resolve | HoppUser | false | #2 |
| 11 | authEvents$.next("login") | HoppUser | false | #2 |

**关键结论**：

1. `token_refresh` 事件（步骤 3）在递归调用开始（步骤 5）**之前**发射
2. `token_refresh` 事件发射时，`currentUser$` 必然为 null（步骤 3 时递归还没开始）
3. `login` 事件（步骤 11）在 `currentUser$` 更新（步骤 8）**之后**发射
4. `isGettingInitialUser = false`（步骤 9）在 `login` 事件**之前**
5. 步骤 4 注册的 watch 在步骤 9 时被触发（`true` → `false`）

---

## 七、竞态分析（统一结论）

### 7.1 唯一真实的竞态窗口

初始化路径刷新成功时，**只存在一个竞态窗口**，不存在"场景 A / 场景 B"两个独立场景。

因为 JavaScript 单线程模型决定了 `authEvents$.next("token_refresh")` 的订阅者回调
一定是同步执行的，递归调用 `setInitialUser()` 一定在订阅者回调完成之后才开始。

**确定的执行时序**：

```
authEvents$.next("token_refresh")          ← 同步
  ├─ onBackendGQLClientShouldReconnect       ← 同步回调
  │    └─ createHoppClient()                 ← 同步
  │         └─ authExchange(async ...)       ← 同步执行到第一个 await
  │              └─ waitProbableLoginToConfirm()
  │                   ├─ getCurrentUser() → null
  │                   ├─ probableUser$ 有值
  │                   └─ watch 注册          ← Promise 挂起
  ├─ syncer1.startListening()                ← 同步
  ├─ syncer2.startListening()                ← 同步
  └─ ...所有订阅者回调完成

return true                                 ← refreshToken 返回

setInitialUser()  [递归，无 await]           ← 此时才开始递归
  ├─ isGettingInitialUser = true (无变化)
  └─ await getInitialUserDetails()           ← 让出执行，后续在微任务中
```

**结论**：不存在"递归调用先于副作用执行"的可能性。
`Subject.next()` 的订阅者回调一定在 `next()` 返回之前全部同步完成。

### 7.2 watch 等待的确切放行时机

| watch 注册时机 | 当前 isGettingInitialUser | 何时 resolve |
|--------------|-------------------------|------------|
| token_refresh 副作用期间（isGettingInitialUser = true） | true | 递归调用中 `isGettingInitialUser = false` 时 |
| login 副作用期间（isGettingInitialUser = false） | false | **不触发**，靠检查点 1 `getCurrentUser()` 有值直接 resolve |

### 7.3 冗余刷新的根因分析

冗余刷新不是竞态导致的，而是 `waitProbableLoginToConfirm` 的设计语义导致的：

```
token_refresh 事件
  → createHoppClient()
    → authExchange 初始化
      → waitProbableLoginToConfirm()
        → isGettingInitialUser = true 时 resolve  ← 问题根源
      → willAuthError() 检查 currentUser$ → null → true
      → 第一个请求前触发 refreshAuth()
        → refreshToken()  ← 冗余的第二次 /auth/refresh
```

**为什么 `true` 时就 resolve？**

这是有意为之的设计。`waitProbableLoginToConfirm` 的目的是防止 GQL 客户端
在初始化完成之前发出请求。当 `isGettingInitialUser = true` 时，
说明初始化流程已经在进行中，GQL 客户端可以开始工作，
由 `authExchange` 的 `willAuthError()` / `refreshAuth()` 机制来处理
尚未获取到用户信息的情况。

**冗余刷新的实际影响**：

- Cookie 已被第一次刷新更新，第二次 `/auth/refresh` 会成功但产生无意义的 Token 轮换
- `authRetryGuard` 记录一次成功（failCount 重置为 0），无害
- 多消耗一次 HTTP 请求

### 7.4 双重客户端重建

初始化路径刷新成功时会触发两次 `createHoppClient()`：

1. **第一次**：`token_refresh` 事件 → `onBackendGQLClientShouldReconnect` 回调
   - `currentUser$ = null`，WebSocket 不会被创建
   - 新 GQL 客户端的 `authExchange` 会因 `willAuthError() = true` 触发冗余刷新

2. **第二次**：`login` 事件 → `onBackendGQLClientShouldReconnect` 回调
   - `currentUser$ = HoppUser`，WebSocket 会被创建
   - 新 GQL 客户端正常工作

**第一次重建的 GQL 客户端是短暂的**：它只在 `token_refresh` 和 `login` 事件之间存在，
被第二次重建覆盖。这期间它可能发出冗余刷新请求，但不会造成数据问题。

---

## 八、两条刷新路径的完整调用顺序与状态变化

### 8.1 路径 A：应用初始化刷新（setInitialUser）

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
  │    │    → GQL 客户端重建（第一次，currentUser$ = null）
  │    │    → syncers.startListening()
  │    └─ return true
  ├─ setInitialUser() [第二次调用，无 await！]   ← 递归
  │    ├─ isGettingInitialUser = true (无变化)
  │    ├─ await getInitialUserDetails() → 成功
  │    ├─ await setUser(hoppUser)
  │    │    ├─ currentUser$.next(hoppUser)
  │    │    ├─ probableUser$.next(hoppUser)
  │    │    └─ persistence.setLocalConfig(...)
  │    ├─ isGettingInitialUser = false           ← 触发 watch
  │    └─ authEvents$.next("login", user)        ← 事件 2
  │         → GQL 客户端重建（第二次，currentUser$ = HoppUser）
  │         → WebSocket 创建
  │         → authRetryGuard.reset()
  └─ return                                      ← 第一次调用结束
```

**注意**：`performAuthInit` 的 `await` 在"第一次调用结束"时结束，
此时递归调用的 `getInitialUserDetails` 可能还在网络请求中。
`performAuthInit` 返回不等于初始化完成。

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

### 8.2 路径 B：GQL 运行时刷新（authExchange + retryGuard）

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
            │    │    → GQL 客户端重建（currentUser$ 有值，正常工作）
            │    │    → WebSocket 关闭并重建
            │    └─ return true
            ├─ failCount = 0
            └─ return true → authExchange 重试原始请求
```

> 运行时刷新成功后 `currentUser$` 保持旧值（不为 null），
> `willAuthError()` 返回 false，后续请求直接使用新 Cookie。
> 不需要重新获取用户信息。

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

## 九、两条路径的关键差异总结

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
| **`waitProbableLoginToConfirm`** | 在 `true` 阶段 resolve | 在检查点 1 直接 resolve（`currentUser$` 有值） |
| **GQL 客户端重建次数** | 2 次（token_refresh + login） | 1 次（token_refresh） |

---

## 十、后端 Token 刷新的完整验证链路

### 10.1 刷新端点

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

### 10.2 双重验证机制

1. **第一层**：Passport-JWT 验证签名和过期时间
2. **第二层**：`argon2.verify()` 验证 Token 哈希与数据库匹配

第二层的作用：即使 JWT 有效，但如果用户已在新设备登录（新 RT 覆盖旧哈希），
旧 RT 签名正确但哈希不匹配，会被拒绝。

### 10.3 Refresh Token Rotation

每次刷新成功后：
- 后端生成全新 Access Token + Refresh Token 对
- 新 RT 的 argon2 哈希覆盖数据库中旧值
- 旧 RT 即刻失效

---

## 十一、token_refresh 事件在各业务模块的副作用

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

## 十二、GQL 客户端重建的决策逻辑

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

**token_refresh 时（初始化路径）**：`currentUser$ = null`，WebSocket 不存在，
创建新 GQL 客户端但不创建 WebSocket。此客户端短暂存在后被 `login` 事件触发第二次重建覆盖。

**token_refresh 时（运行时路径）**：`currentUser$` 有值，WebSocket 关闭并重建，
新 GQL 客户端正常工作。

### authExchange 初始化逻辑

```typescript
authExchange(async (): Promise<AuthConfig> => {
  const probableUser = platform.auth.getProbableUser()
  if (probableUser !== null)
    await platform.auth.waitProbableLoginToConfirm()
  // ...
})
```

| 场景 | probableUser | 等待行为 | resolve 时机 |
|------|-------------|---------|------------|
| 初始化路径 token_refresh | 旧用户（非 null） | 进入等待 | `isGettingInitialUser = false` 时 |
| 初始化路径 login | HoppUser | 检查点 1 直接 resolve | `getCurrentUser()` 有值 |
| 运行时 token_refresh | HoppUser | 检查点 1 直接 resolve | `getCurrentUser()` 有值 |
| 运行时 logout | null | 跳过等待 | `probableUser` 为 null |
| 从未登录 | null | 跳过等待 | `probableUser` 为 null |

---

## 十三、完整状态变化矩阵

### 13.1 初始化路径（路径 A）

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
| T7 login 事件 | HoppUser | HoppUser | false | 是 | `login` |
| T2 刷新失败 | null | null | false | 是 | **无** |

### 13.2 GQL 运行时路径（路径 B）

| 时刻 | currentUser$ | probableUser$ | authRetryGuard | 发射的事件 |
|-----|-------------|--------------|---------------|----------|
| 刷新成功 | 不变(HoppUser) | 不变 | failCount=0 | `token_refresh` |
| 刷新失败(1-2次) | 不变(HoppUser) | 不变 | failCount++ | **无** |
| 刷新失败(3次) | null | null | isExhausted=true | `logout` |
| 耗尽后请求 | null | null | 直接返回 false | **无** |

---

## 十四、时序图：初始化路径刷新成功

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
    │    ├─ GQL 客户端重建 #1 (currentUser=null) │
    │    ├─ watch 注册 (等待 isGettingInitialUser)
    │    └─ syncers.startListening()             │
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
    │  authEvents$.next("login")                 │
    │    ├─ GQL 客户端重建 #2 (currentUser=用户)  │
    │    ├─ WebSocket 创建                       │
    │    └─ authRetryGuard.reset()               │
    │                                            │
```

---

## 十五、时序图：GQL 运行时刷新耗尽

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

## 十六、配置项说明

| 配置项 | 默认值 | 说明 |
|-------|-------|------|
| `INFRA.JWT_SECRET` | - | JWT 签名密钥 |
| `INFRA.ACCESS_TOKEN_VALIDITY` | `86400000` | Access Token 有效期 (ms，1天) |
| `INFRA.REFRESH_TOKEN_VALIDITY` | `604800000` | Refresh Token 有效期 (ms，7天) |
| `INFRA.ALLOW_SECURE_COOKIES` | `false` | 是否启用 Secure Cookie |

---

## 十七、安全设计要点

1. **HttpOnly Cookie**：前端 JS 无法读取 Token，防止 XSS 窃取
2. **Refresh Token Rotation**：每次刷新生成全新 Token 对，旧 Token 即刻失效
3. **argon2 双重验证**：数据库不存明文，即使 JWT 有效也需哈希匹配
4. **重试次数限制**（运行时路径）：3 次失败后强制登出
5. **SameSite=Lax Cookie**：防止 CSRF 攻击
6. **Secure Cookie（可选）**：HTTPS 传输加密

---

## 十八、代码溯源索引

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
| authRetryGuard 实现 | `hoppscotch-common/src/helpers/retryAuthGuard.ts` | 1-78 |
| Admin AuthEvent 类型 | `hoppscotch-sh-admin/src/helpers/auth.ts` | 37-40 |
| 后端刷新端点 | `hoppscotch-backend/src/auth/auth.controller.ts` | 87-100 |
| 后端刷新服务 | `hoppscotch-backend/src/auth/auth.service.ts` | 103-127, 335-363 |
| 后端 RT JWT 策略 | `hoppscotch-backend/src/auth/strategies/rt-jwt.strategy.ts` | 20-49 |
| 后端 AT JWT 策略 | `hoppscotch-backend/src/auth/strategies/jwt.strategy.ts` | 64-109 |
| 后端 Cookie 处理 | `hoppscotch-backend/src/auth/helper.ts` | 38-82 |
| GqlAuthGuard | `hoppscotch-backend/src/guards/gql-auth.guard.ts` | 6-11 |
