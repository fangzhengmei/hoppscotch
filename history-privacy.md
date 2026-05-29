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

### 2.2 GraphQL 请求历史记录写入

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

### 2.3 数据结构定义

**REST 历史记录条目**（`history.ts:13-28`）：
```typescript
type RESTHistoryEntry = {
  v: number                    // 版本号，当前为 1
  request: HoppRESTRequest     // 请求内容
  responseMeta: {              // 响应元数据
    duration: number | null    // 响应时长
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

**核心文件**：`packages/hoppscotch-selfhost-web/src/platform/history/web/sync.ts:29-32, 65-68`

```typescript
// REST 历史记录同步
async addEntry({ entry }) {
  if (!isHistoryStoreEnabled.value) {
    return  // 开关关闭时，不同步到后端
  }
  // ... 同步到后端
}

// GraphQL 历史记录同步
async addEntry({ entry }) {
  if (!isHistoryStoreEnabled.value) {
    return  // 开关关闭时，不同步到后端
  }
  // ... 同步到后端
}
```

**重要**：`isHistoryStoreEnabled` 只控制**云端同步**，不控制**本地持久化**。本地历史记录始终会被保存。

### 4.5 UI 层对开关的响应

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
          ├─ 检查 isHistoryStoreEnabled 开关
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
| GraphQL 写入 | `packages/hoppscotch-common/src/helpers/graphql/connection.ts` | 650-665 |
| 隐私开关实现 | `packages/hoppscotch-selfhost-web/src/platform/history/web/index.ts` | 129-151, 318 |
| 开关作用范围 | `packages/hoppscotch-selfhost-web/src/platform/history/web/sync.ts` | 29-32, 65-68 |
| 实时更新订阅 | `packages/hoppscotch-selfhost-web/src/platform/history/web/index.ts` | 286-300 |
| 手动清理 | `packages/hoppscotch-common/src/components/history/Personal.vue` | 312-316 |
| 后端批量删除 | `packages/hoppscotch-selfhost-web/src/platform/history/web/index.ts` | 269-284 |
| 后端全部删除 | `packages/hoppscotch-selfhost-web/src/platform/history/web/index.ts` | 302-316 |
| 登出行为 | `packages/hoppscotch-selfhost-web/src/platform/history/web/index.ts` | 61-69 |
| UI 响应 | `packages/hoppscotch-common/src/components/history/Personal.vue` | 53-57, 66, 119 |

---

## 八、架构设计特点

1. **分层清晰**：内存 Store、本地持久化、云端同步三层分离，职责明确
2. **隐私分级**：`isHistoryStoreEnabled` 只控制云端同步，本地持久化始终可用
3. **响应式设计**：通过 RxJS 流和 Vue 响应式系统实现数据自动同步
4. **容错机制**：Schema 验证失败时自动备份，数据迁移支持多版本
5. **实时同步**：通过 GraphQL 订阅实现后端状态实时更新到前端

---

## 九、注意事项

1. **隐私开关不影响本地存储**：即使 `isHistoryStoreEnabled = false`，历史记录仍会保存在浏览器本地
2. **登出不清除本地数据**：用户登出后，本地历史记录仍然保留，需要手动清除
3. **历史记录上限**：内存中最多保留 50 条记录（`HISTORY_LIMIT = 50`）
4. **数据迁移**：本地存储支持数据格式迁移，从旧版本自动升级到新版本
