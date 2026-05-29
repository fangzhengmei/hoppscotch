# 历史记录隐私模式与本地持久化实现分析

## 一、核心架构概述

Hoppscotch 的历史记录系统采用**分层架构**，分为：
1. 内存存储层（Store）
2. 本地持久化层（Persistence）
3. 云端同步层（Sync）
4. UI 展示层（Components）

隐私模式通过 `isHistoryStoreEnabled` 开关控制**云端同步**行为，但不影响**本地持久化**。

---

## 二、历史记录写入逻辑

### 2.1 REST 请求历史记录写入

**核心文件**：`packages/hoppscotch-common/src/newstore/history.ts:355-372`

```typescript
// 监听执行完成的响应流
executedResponses$.subscribe((res) => {
  const { _ref_id, id, ...request } = res.req

  addRESTHistoryEntry(
    makeRESTHistoryEntry({
      request,
      responseMeta: {
        duration: res.meta.responseDuration,
        statusCode: res.statusCode,
      },
      star: false,
      updatedOn: new Date(),
    })
  )
})
```

**写入流程**：
1. `RequestRunner.ts` 执行请求完成后，通过 `executedResponses$.next(res)` 发出事件
2. `history.ts` 中订阅该事件，自动创建历史记录条目
3. 通过 `addRESTHistoryEntry` 写入内存 Store，最多保留 `HISTORY_LIMIT = 50` 条

### 2.2 REST 请求成功与失败的历史记录写入规则

**响应类型定义**（`packages/hoppscotch-common/src/helpers/types/HoppRESTResponse.ts`）：
```typescript
type HoppRESTResponse =
  | HoppRESTLoadingResponse      // type: "loading"
  | HoppRESTSuccessResponse      // type: "success"
  | HoppRESTFailureResponse      // type: "failure" (注意：类型定义存在但实际不使用)
  | HoppRESTFailureNetwork       // type: "network_fail"
  | HoppRESTFailureScript        // type: "script_fail"
  | HoppRESTFailureExtension     // type: "extension_error"
  | HoppRESTFailureInterceptor   // type: "interceptor_error"
```

**历史记录过滤条件**（`packages/hoppscotch-common/src/helpers/RequestRunner.ts:588`）：
```typescript
const subscription = stream
  .pipe(filter((res) => res.type === "success" || res.type === "fail"))
  .subscribe(async (res) => {
    if (res.type === "success" || res.type === "fail") {
      executedResponses$.next(res)
      // ...
    }
  })
```

**实际写入分析**：

| 响应类型 | type 值 | 是否进入历史记录 | 说明 |
|---------|--------|----------------|------|
| HTTP 成功 (2xx) | "success" | ✅ 是 | 正常写入历史记录 |
| HTTP 失败 (4xx/5xx) | "success" | ✅ 是 | 只要响应体格式正确，即使状态码是 404/500 等，也会被标记为 type: "success" |
| 响应体转换错误 | "fail" | ❌ 否 | 在 network.ts:51-55 中被转换为 type: "network_fail"，不会通过过滤器 |
| 网络连接失败 | "network_fail" | ❌ 否 | 无法连接服务器时产生，不进入历史记录 |
| 脚本执行失败 | "script_fail" | ❌ 否 | Pre-request 或 Test 脚本出错，不进入历史记录 |
| 拦截器错误 | "interceptor_error" | ❌ 否 | 代理/浏览器/扩展拦截器出错，不进入历史记录 |
| 扩展错误 | "extension_error" | ❌ 否 | 浏览器扩展相关错误，不进入历史记录 |

**重要发现**：
1. **HTTP 错误响应 (4xx/5xx) 会进入历史记录**：只要内核能收到响应且响应体格式正确，无论 HTTP 状态码是什么，都会被标记为 `type: "success"` 并写入历史记录
2. **`type: "fail"` 实际上不会进入历史记录**：虽然过滤器中写了 `res.type === "fail"`，但在 network.ts:51-55 中，`type: "fail"` 的响应会被转换为 `type: "network_fail"`，而 `network_fail` 不会通过过滤器
3. **类型定义 bug**：`executedResponses$` 的类型定义中 `"fail "` 多了一个空格（`RequestRunner.ts:209`），这是一个拼写错误

### 2.3 GraphQL 请求历史记录写入

**核心文件**：`packages/hoppscotch-common/src/helpers/graphql/connection.ts:650-665`

```typescript
const addQueryToHistory = (options: RunQueryOptions, response: string) => {
  const { name, url, request, query, variables } = options
  addGraphqlHistoryEntry(
    makeGQLHistoryEntry({
      request: makeGQLRequest({
        name: name ?? "Untitled Request",
        url,
        query,
        headers: request.headers,
        variables,
        auth: request.auth as HoppGQLAuth,
      }),
      response,
      star: false,
    })
  )
}
```

**写入触发点**：
- 查询/突变操作成功后调用：`connection.ts:493`
- 订阅操作开始时调用：`connection.ts:641`

### 2.4 GraphQL responseDuration 计算方法及数据偏差

**严重 Bug 发现**（`packages/hoppscotch-common/src/helpers/graphql/connection.ts:477-488`）：

```typescript
// 请求执行完成后
const relayResponse = result.right
const parsedResponse = await GQLResponse.toResponse(relayResponse, options)

if (parsedResponse.type === "error") {
  throw new Error(parsedResponse.error.message)
}

// ⚠️ Bug 位置：这两行连续执行
const timeStart = Date.now()
const timeEnd = Date.now()

gqlMessageEvent.value = {
  ...parsedResponse,
  document: {
    type: "success",
    statusCode: relayResponse.status,
    statusText: relayResponse.statusText,
    meta: {
      responseSize: relayResponse.body.body.byteLength,
      responseDuration: timeEnd - timeStart,  // ⚠️ 结果几乎总是 0 或 1ms
    },
  },
}
```

**问题分析**：
1. `timeStart` 和 `timeEnd` 在请求完成后才被设置，且是连续的两行代码
2. 两行代码执行的时间差几乎为 0，导致 `responseDuration` 几乎总是 0 或 1 毫秒
3. 这完全不能反映真实的网络请求时间

**对比 REST 的正确实现**（`packages/hoppscotch-common/src/helpers/kernel/rest/response.ts:16-19`）：
```typescript
const extractTiming = (response: RelayResponse): number =>
  response.meta?.timing
    ? response.meta.timing.end - response.meta.timing.start  // 使用内核记录的真实时间
    : 0
```

REST 请求在内核层（`hoppscotch-kernel`）就记录了真实的 `startTime` 和 `endTime`，然后在前端计算差值。

**数据偏差影响**：
- GraphQL 历史记录中的响应时间完全没有参考价值
- 用户无法通过历史记录了解真实的 API 响应性能
- 如果基于响应时间做统计或监控，数据将完全失真

### 2.5 数据结构定义

**REST 历史记录条目**（`history.ts:13-28`）：
```typescript
type RESTHistoryEntry = {
  v: number                    // 版本号，当前为 1
  request: HoppRESTRequest     // 请求内容
  responseMeta: {              // 响应元数据
    duration: number | null    // 响应时长（REST 正确，GQL 有 bug）
    statusCode: number | null  // 状态码
  }
  star: boolean                // 是否收藏
  id?: string                  // 云端 Firestore ID
  updatedOn?: Date             // 更新时间
}
```

**GraphQL 历史记录条目**（`history.ts:30-41`）：
```typescript
type GQLHistoryEntry = {
  v: number                    // 版本号，当前为 1
  request: HoppGQLRequest      // 请求内容
  response: string             // 响应数据（JSON 字符串）
  star: boolean                // 是否收藏
  id?: string                  // 云端 Firestore ID
  updatedOn?: Date             // 更新时间
}
```

---

## 三、本地持久化实现

**核心文件**：`packages/hoppscotch-common/src/services/persistence/index.ts`

### 3.1 持久化键定义

```typescript
export const STORE_KEYS = {
  REST_HISTORY: "restHistory",   // REST 历史记录键
  GQL_HISTORY: "gqlHistory",     // GraphQL 历史记录键
  // ... 其他键
} as const
```

### 3.2 REST 历史记录持久化

**核心代码**：`persistence/index.ts:529-559`

```typescript
private async setupRESTHistoryPersistence() {
  // 1. 从存储加载
  const restLoadResult = await Store.get<any>(
    STORE_NAMESPACE,
    STORE_KEYS.REST_HISTORY
  )

  if (E.isRight(restLoadResult)) {
    const data = restLoadResult.right ?? []
    const result = z.array(REST_HISTORY_ENTRY_SCHEMA).safeParse(data)
    
    if (result.success) {
      const translatedData = result.data.map(translateToNewRESTHistory)
      setRESTHistoryEntries(translatedData)
    }
  }

  // 2. 订阅变化自动保存
  restHistoryStore.subject$.subscribe(async ({ state }) => {
    await Store.set(STORE_NAMESPACE, STORE_KEYS.REST_HISTORY, state)
  })
}
```

### 3.3 GraphQL 历史记录持久化

**核心代码**：`persistence/index.ts:561-591`

与 REST 类似，使用 `STORE_KEYS.GQL_HISTORY` 作为存储键。

### 3.4 持久化特点

1. **自动双向同步**：启动时从本地存储加载，运行时订阅变化自动保存
2. **数据版本迁移**：支持从旧版本数据格式迁移（`persistence/index.ts:141-248` 中的 migrations）
3. **Schema 校验**：使用 Zod 进行数据验证，失败时创建备份（`-backup` 后缀）
4. **独立于隐私开关**：无论 `isHistoryStoreEnabled` 状态如何，本地持久化始终生效

### 3.5 Web 端与桌面端本地持久化介质差异

| 对比项 | Web 端（Browser） | 桌面端（Tauri） |
|-------|------------------|----------------|
| **存储引擎** | localStorage | Tauri Store（@tauri-apps/plugin-store） |
| **存储位置** | 浏览器沙箱 | 应用数据目录（二进制文件） |
| **序列化方式** | superjson.stringify() | 内置 JSON 序列化 |
| **数据结构** | 扁平化键值对，每个 key 单独存储 | 结构化对象，所有数据在一个文件中 |
| **键命名** | `namespace:key` 格式 | 两级嵌套：namespace → key |
| **安全特性** | 无加密能力 | 支持加密（capabilities: "secure"） |
| **数据持久化** | 受浏览器隐私设置限制，可能被清除 | 持久化存储，除非用户卸载应用 |
| **存储上限** | 受 localStorage 限制（通常 5MB） | 受磁盘空间限制 |
| **写入性能** | 同步写入，小数据快 | 异步写入 + 显式 save() 调用 |
| **实现文件** | `packages/hoppscotch-kernel/src/store/impl/web/v/1.ts` | `packages/hoppscotch-kernel/src/store/impl/desktop/v/1.ts` |

**Web 端实现要点**（`web/v/1.ts`）：
```typescript
// 使用 localStorage 存储
async set(namespace: string, key: string, value: StoredData): Promise<void> {
  const validated = StoredDataSchema.parse(value)
  localStorage.setItem(
    this.getFullKey(namespace, key),  // key 格式："namespace:key"
    superjson.stringify(validated)
  )
  this.notifyListeners(namespace, key, validated.data)
}
```

**桌面端实现要点**（`desktop/v/1.ts`）：
```typescript
// 使用 Tauri Store 存储
async set(namespace: string, key: string, value: StoredData): Promise<void> {
  if (!this.store) throw new Error("Store not initialized")

  const validated = StoredDataSchema.parse(value)
  this.data[namespace] = this.data[namespace] || {}
  this.data[namespace][key] = validated
  await this.store.set("data", this.data)  // 所有数据存在一个 key 下
  await this.store.save()  // 需要显式调用 save() 持久化
}
```

---

## 四、隐私开关（isHistoryStoreEnabled）管理

### 4.1 开关定义

**核心文件**：`packages/hoppscotch-common/src/platform/history.ts:3-9`

```typescript
export type HistoryPlatformDef = {
  initHistorySync: () => void
  requestHistoryStore?: {
    isHistoryStoreEnabled: Ref<boolean>           // 历史记录存储是否启用
    isFetchingHistoryStoreStatus: Ref<boolean>    // 是否正在获取状态
    hasErrorFetchingHistoryStoreStatus: Ref<boolean>  // 获取状态是否出错
  }
}
```

### 4.2 开关值设置逻辑

**核心文件**：`packages/hoppscotch-selfhost-web/src/platform/history/web/index.ts`

```typescript
export const isHistoryStoreEnabled = ref(false)  // 默认值：false

async function getUserHistoryStatus() {
  const currentUser = platformAuth.getCurrentUser()

  // 未登录用户：默认启用
  if (!currentUser) {
    isHistoryStoreEnabled.value = true
    return
  }

  // 已登录用户：从后端获取状态
  isFetchingHistoryStoreStatus.value = true
  const res = await getUserHistoryStore()

  if (E.isLeft(res)) {
    hasErrorFetchingHistoryStoreStatus.value = true
    isFetchingHistoryStoreStatus.value = false
    return
  }

  // 根据后端返回的 ServiceStatus 设置
  isHistoryStoreEnabled.value =
    res.right.isUserHistoryEnabled.value === ServiceStatus.Enable

  isFetchingHistoryStoreStatus.value = false
}
```

**设置规则**：
- 默认值：`false`（`index.ts:318`）
- 未登录用户：`true`（`index.ts:133`）
- 已登录用户：从后端 `getUserHistoryStore` 查询（`index.ts:139-148`）

### 4.3 开关实时更新

通过 GraphQL 订阅实时监听开关状态变化：

**核心代码**：`index.ts:286-300`

```typescript
function setupUserHistoryStoreStatusChangedSubscription() {
  const [userHistoryStoreStatusChanged$, userHistoryStoreStatusChangedSub] =
    runUserHistoryStoreStatusChangedSubscription()

  userHistoryStoreStatusChanged$.subscribe((res) => {
    if (E.isRight(res)) {
      const status =
        res.right.infraConfigUpdate == ServiceStatus.Enable ? true : false

      isHistoryStoreEnabled.value = status
    }
  })

  return userHistoryStoreStatusChangedSub
}
```

### 4.4 开关作用范围

**核心文件**：`packages/hoppscotch-selfhost-web/src/platform/history/web/sync.ts`

### 4.5 syncHistory 与 isHistoryStoreEnabled 生效范围详解

**两个开关的区别**：
1. **`syncHistory`**（用户设置）：总开关，控制是否启用历史记录同步功能
2. **`isHistoryStoreEnabled`**（隐私开关）：细粒度控制，只影响新增记录

**同步机制核心**（`packages/hoppscotch-selfhost-web/src/lib/sync/index.ts:54-72`）：
```typescript
function startStoreSync() {
  store.dispatches$.subscribe((actionParams) => {
    if ((storeSyncDefinition as any)[actionParams.dispatcher]) {
      const dispatcher = actionParams.dispatcher
      const payload = actionParams.payload
      const operationMapperFunction = (storeSyncDefinition as any)[dispatcher]

      if (
        operationMapperFunction &&
        _isRunningDispatchWithoutSyncing &&
        shouldSyncValue()  // 检查 syncHistory 设置
      ) {
        operationMapperFunction(payload)  // 执行具体操作
      }
    }
  })
}
```

**各操作的生效范围**：

| 操作 | syncHistory = false | syncHistory = true<br>isHistoryStoreEnabled = false | syncHistory = true<br>isHistoryStoreEnabled = true |
|-----|---------------------|---------------------------------------------------|--------------------------------------------------|
| **addEntry**（新增记录） | ❌ 不同步 | ❌ 不同步 | ✅ 同步 |
| **deleteEntry**（删除记录） | ❌ 不同步 | ⚠️ 如果有 id 则同步删除 | ✅ 同步 |
| **toggleStar**（收藏切换） | ❌ 不同步 | ⚠️ 如果有 id 则同步 | ✅ 同步 |
| **clearHistory**（清空历史） | ❌ 不同步 | ✅ 同步清空 | ✅ 同步清空 |
| **订阅监听** | ❌ 停止监听 | ✅ 正常监听 | ✅ 正常监听 |

**详细分析**：

1. **addEntry（新增记录）**
   - 受两个开关双重控制：`syncHistory` 必须为 true 且 `isHistoryStoreEnabled` 必须为 true
   - 代码位置：`sync.ts:29-32`（REST）、`sync.ts:65-68`（GraphQL）
   ```typescript
   async addEntry({ entry }) {
     if (!isHistoryStoreEnabled.value) {
       return  // 开关关闭时，不同步到后端
     }
     // ... 同步到后端
   }
   ```

2. **deleteEntry（删除记录）**
   - 只受 `syncHistory` 控制
   - 只要记录有 `id`（说明曾经同步过），就会同步删除
   - 代码位置：`sync.ts:47-51`（REST）、`sync.ts:83-87`（GraphQL）
   ```typescript
   deleteEntry({ entry }) {
     if (entry.id) {  // 只检查是否有 id，不检查 isHistoryStoreEnabled
       removeRequestFromHistory(entry.id)
     }
   }
   ```

3. **toggleStar（收藏切换）**
   - 只受 `syncHistory` 控制
   - 只要记录有 `id`，就会同步收藏状态
   - 代码位置：`sync.ts:52-56`（REST）、`sync.ts:88-92`（GraphQL）
   ```typescript
   toggleStar({ entry }) {
     if (entry.id) {  // 只检查是否有 id，不检查 isHistoryStoreEnabled
       toggleHistoryStarStatus(entry.id)
     }
   }
   ```

4. **clearHistory（清空历史）**
   - 只受 `syncHistory` 控制
   - 总是同步到后端
   - 代码位置：`sync.ts:57-59`（REST）、`sync.ts:93-95`（GraphQL）
   ```typescript
   clearHistory() {
     deleteAllUserHistory(ReqType.Rest)  // 不检查 isHistoryStoreEnabled
   }
   ```

**重要结论**：
- `isHistoryStoreEnabled = false` 只**阻止新记录同步**，不影响已有记录的删除、收藏、清空操作
- 这意味着即使关闭了隐私开关，之前同步过的记录仍然可以被删除和收藏
- 清空操作也会同步到后端，删除云端所有历史记录

### 4.6 UI 层对开关的响应

**历史记录列表**（`Personal.vue:66, 119`）：
- 开关开启时：显示历史记录列表
- 开关关闭时：显示占位图，提示 "History Disabled"

**清空按钮**（`Personal.vue:53-57`）：
- 开关关闭时：清空按钮被禁用

---

## 五、历史记录清理触发逻辑

### 5.1 用户手动清理

**触发位置**：`components/history/Personal.vue:312-316`

```typescript
const clearHistory = () => {
  if (props.page === "rest") clearRESTHistory()
  else clearGraphqlHistory()
  toast.success(`${t("state.history_deleted")}`)
}
```

**触发方式**：
- 点击历史记录页面的垃圾桶图标
- 通过 Spotlight 搜索 "Clear History"
- 键盘快捷键触发 `history.clear` action

### 5.2 后端批量删除订阅

**核心代码**：`index.ts:269-284`

```typescript
function setupUserHistoryDeletedManySubscription() {
  const [userHistoryDeletedMany$, userHistoryDeletedManySub] =
    runUserHistoryDeletedManySubscription()

  userHistoryDeletedMany$.subscribe((res) => {
    if (E.isRight(res)) {
      const { reqType } = res.right.userHistoryDeletedMany

      runDispatchWithOutSyncing(() => {
        reqType == ReqType.Rest ? clearRESTHistory() : clearGraphqlHistory()
      })
    }
  })

  return userHistoryDeletedManySub
}
```

**触发场景**：后端发送某类型（REST 或 GraphQL）历史记录全部删除事件

### 5.3 后端全部删除订阅

**核心代码**：`index.ts:302-316`

```typescript
function setupUserHistoryAllDeletedSubscription() {
  const [userHistoryAllDeleted$, userHistoryAllDeletedSub] =
    runUserHistoryAllDeletedSubscription()

  userHistoryAllDeleted$.subscribe((res) => {
    if (E.isRight(res)) {
      runDispatchWithOutSyncing(() => {
        clearRESTHistory()
        clearGraphqlHistory()
      })
    }
  })

  return userHistoryAllDeletedSub
}
```

**触发场景**：后端发送所有历史记录删除事件

### 5.4 单条记录删除

**触发方式**：
- 点击历史记录卡片的删除按钮：`Personal.vue:349-353`
- 按时间分组批量删除：`Personal.vue:336-347`

### 5.5 Store 清空实现

**核心代码**：`history.ts:157-161, 219-224`

```typescript
// REST
clearHistory(_, {}) {
  return {
    state: [],
  }
}

// GraphQL
clearHistory(_, {}) {
  return {
    state: [],
  }
}
```

### 5.6 登出行为

**核心代码**：`index.ts:61-69`

```typescript
authEvents$.subscribe((event) => {
  if (event.event == "login" || event.event == "token_refresh") {
    restHistorySyncer.startListeningToSubscriptions()
  }

  if (event.event == "logout") {
    restHistorySyncer.stopListeningToSubscriptions()
    // 注意：不清除本地历史记录
  }
})
```

**关键点**：登出时仅停止云端订阅监听，**不会清除本地历史记录**。

---

## 六、同步控制与数据流

### 6.1 同步配置

**核心文件**：`sync.ts:98-110`

```typescript
export const restHistorySyncer = getSyncInitFunction(
  restHistoryStore,
  restHistoryStoreSyncDefinition,
  () => settingsStore.value.syncHistory,    // 同步开关设置
  getSettingSubject("syncHistory")
)
```

同步受两个开关控制：
1. `syncHistory`（用户设置）：是否启用历史记录同步
2. `isHistoryStoreEnabled`（后端/隐私开关）：是否允许存储到后端

### 6.2 完整数据流

```
请求执行
    ↓
executedResponses$ 事件 (REST) / addQueryToHistory (GQL)
    ↓
写入内存 Store (history.ts)
    ├─→ 本地持久化 (persistence/index.ts) → Store.set()
    └─→ 云端同步 (sync.ts)
          ├─ 检查 syncHistory 设置
          ├─ 检查 isHistoryStoreEnabled 开关（仅 addEntry）
          └─ 调用 createUserHistory API
```

### 6.3 避免重复插入

**核心代码**：`sync.ts:40-45, 76-81`

```typescript
if (E.isRight(res)) {
  entry.id = res.right.createUserHistory.id
  // 防止 storeSync 和 Subscription 重复插入
  removeDuplicateRestHistoryEntry(entry.id)
}
```

本地写入后，云端同步会返回 `id`，此时需要去重，因为后端订阅也会推送新记录。

---

## 七、关键文件索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| 历史记录 Store | `packages/hoppscotch-common/src/newstore/history.ts` | 13-372 |
| 平台定义 | `packages/hoppscotch-common/src/platform/history.ts` | 3-9 |
| 本地持久化 | `packages/hoppscotch-common/src/services/persistence/index.ts` | 529-591 |
| REST 自动写入 | `packages/hoppscotch-common/src/newstore/history.ts` | 355-372 |
| REST 响应过滤 | `packages/hoppscotch-common/src/helpers/RequestRunner.ts` | 588-591 |
| GraphQL 写入 | `packages/hoppscotch-common/src/helpers/graphql/connection.ts` | 650-665 |
| GraphQL 响应时间 Bug | `packages/hoppscotch-common/src/helpers/graphql/connection.ts` | 477-488 |
| 隐私开关实现 | `packages/hoppscotch-selfhost-web/src/platform/history/web/index.ts` | 129-151, 318 |
| 开关作用范围 | `packages/hoppscotch-selfhost-web/src/platform/history/web/sync.ts` | 29-95 |
| 同步机制 | `packages/hoppscotch-selfhost-web/src/lib/sync/index.ts` | 54-72 |
| 实时更新订阅 | `packages/hoppscotch-selfhost-web/src/platform/history/web/index.ts` | 286-300 |
| 手动清理 | `packages/hoppscotch-common/src/components/history/Personal.vue` | 312-316 |
| 后端批量删除 | `packages/hoppscotch-selfhost-web/src/platform/history/web/index.ts` | 269-284 |
| 后端全部删除 | `packages/hoppscotch-selfhost-web/src/platform/history/web/index.ts` | 302-316 |
| 登出行为 | `packages/hoppscotch-selfhost-web/src/platform/history/web/index.ts` | 61-69 |
| UI 响应 | `packages/hoppscotch-common/src/components/history/Personal.vue` | 53-57, 66, 119 |
| Web 端存储 | `packages/hoppscotch-kernel/src/store/impl/web/v/1.ts` | 39-88 |
| 桌面端存储 | `packages/hoppscotch-kernel/src/store/impl/desktop/v/1.ts` | 87-95, 216-226 |

---

## 八、架构设计特点

1. **分层清晰**：内存 Store、本地持久化、云端同步三层分离，职责明确
2. **隐私分级**：`isHistoryStoreEnabled` 只控制云端同步，本地持久化始终可用
3. **响应式设计**：通过 RxJS 流和 Vue 响应式系统实现数据自动同步
4. **容错机制**：Schema 验证失败时自动备份，数据迁移支持多版本
5. **实时同步**：通过 GraphQL 订阅实现后端状态实时更新到前端
6. **跨平台兼容**：抽象的 Store 接口，Web 端用 localStorage，桌面端用 Tauri Store

---

## 九、已知问题与注意事项

### 9.1 已知 Bug

1. **GraphQL responseDuration 计算错误**（严重）
   - 位置：`packages/hoppscotch-common/src/helpers/graphql/connection.ts:477-488`
   - 影响：响应时间几乎总是 0 或 1ms，完全不能反映真实网络请求时间
   - 修复建议：使用 kernel 层 `relayResponse.meta.timing` 中的真实时间

2. **类型定义拼写错误**
   - 位置：`packages/hoppscotch-common/src/helpers/RequestRunner.ts:209`
   - 问题：`"fail "` 多了一个空格
   - 影响：实际不影响功能（因为 `type: "fail"` 会被转换为 `network_fail`），但类型定义不准确

### 9.2 功能设计注意事项

1. **隐私开关不影响本地存储**：即使 `isHistoryStoreEnabled = false`，历史记录仍会保存在浏览器/本地存储中
2. **登出不清除本地数据**：用户登出后，本地历史记录仍然保留，需要手动清除
3. **历史记录上限**：内存中最多保留 50 条记录（`HISTORY_LIMIT = 50`）
4. **数据迁移**：本地存储支持数据格式迁移，从旧版本自动升级到新版本
5. **4xx/5xx 响应进入历史记录**：HTTP 错误响应只要响应体格式正确就会进入历史记录
6. **网络失败不进入历史记录**：`network_fail`、`script_fail` 等类型不会进入历史记录
7. **隐私开关关闭时已有记录仍可操作**：关闭 `isHistoryStoreEnabled` 后，之前同步的记录仍然可以删除、收藏、清空

### 9.3 跨平台差异

1. **Web 端**：使用 localStorage，受浏览器隐私设置限制，可能被清除
2. **桌面端**：使用 Tauri Store，数据持久化更可靠，支持加密，存储上限更高
3. **两者 API 一致**：上层代码无需关心底层存储介质差异
