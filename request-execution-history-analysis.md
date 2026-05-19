# 请求执行与历史记录链路分析

## 概述

本文档分析 Hoppscotch 应用中一次 HTTP 请求从用户点击发送、协议适配器派发、收到响应到写入历史记录的完整链路。整个流程涉及界面层（UI）和服务层（Service）的多段协同，核心包括执行编排、协议路由和历史持久化三个关键环节。

---

## 一、整体架构概览

```
用户点击发送按钮 (UI层)
        ↓
RequestRunner.runRESTRequest$ (执行编排入口)
        ↓
createRESTNetworkRequestStream (网络请求流创建)
        ↓
RESTRequest.toRequest (协议适配: HoppRESTRequest → RelayRequest)
        ↓
KernelInterceptorService.execute (协议路由: 选择拦截器)
        ↓
NativeKernelInterceptorService.execute (具体拦截器执行)
        ↓
Relay.execute (内核实际发送请求)
        ↓
收到响应 → RESTResponse.toResponse (协议适配: RelayResponse → HoppRESTResponse)
        ↓
executedResponses$.next (响应事件广播)
        ↓
history.ts 订阅者 → addRESTHistoryEntry (写入内存历史)
        ↓
restHistoryStore.subject$ 变化 → PersistenceService 订阅 → Store.set (持久化到磁盘)
```

---

## 二、详细链路分析

### 2.1 界面层：用户点击发送按钮

**入口文件**: `packages/hoppscotch-common/src/components/http/Request.vue`

```vue
<HoppButtonPrimary
  id="send"
  :label="!isTabResponseLoading ? t('action.send') : t('action.cancel')"
  @click="!isTabResponseLoading ? newSendRequest() : cancelRequest()"
/>
```

**关键处理**:
- 按钮根据 `isTabResponseLoading` 状态切换为 "发送" 或 "取消"
- 点击事件触发 `newSendRequest()` 方法
- 该方法最终调用 `RequestRunner.runRESTRequest$` 启动请求执行流程

---

### 2.2 执行编排：RequestRunner

**核心文件**: `packages/hoppscotch-common/src/helpers/RequestRunner.ts`

#### 2.2.1 主执行函数 `runRESTRequest$`

这是整个请求执行的编排核心，负责协调整个请求生命周期：

```typescript
export function runRESTRequest$(
  tab: Ref<HoppTab<HoppRequestDocument>>
): [
  () => void,
  Promise<E.Left<"script_fail" | "cancellation"> | E.Right<Observable<HoppRESTResponse>>>,
] {
  // 1. 捕获初始环境状态
  const initialEnvState = captureInitialEnvironmentState()
  
  // 2. 执行前置脚本 (Pre-request Script)
  const preRequestResult = delegatePreRequestScriptRunner(...)
  
  // 3. 处理环境变量和请求变量
  const finalEnvs = combineEnvVariables(...)
  const effectiveRequest = await getEffectiveRESTRequest(finalRequest, ...)
  
  // 4. 创建网络请求流
  const [stream, cancelRun] = await createRESTNetworkRequestStream(effectiveRequest)
  
  // 5. 订阅响应流
  stream.pipe(filter(res => res.type === "success" || res.type === "fail"))
    .subscribe(async (res) => {
      // 5.1 广播响应事件 (用于历史记录)
      executedResponses$.next(res)
      
      // 5.2 执行后置脚本 (Test Script)
      const postRequestResult = await runPostRequestScript(...)
      
      // 5.3 更新环境变量
      updateEnvsAfterTestScript(...)
    })
  
  return [cancel, res]
}
```

#### 2.2.2 关键编排步骤

| 阶段 | 描述 | 核心函数 |
|------|------|----------|
| 环境捕获 | 保存执行前的环境状态，用于后续对比更新 | `captureInitialEnvironmentState()` |
| 前置脚本 | 执行用户定义的 pre-request 脚本，可修改请求和环境 | `delegatePreRequestScriptRunner()` |
| 变量解析 | 解析环境变量、集合变量、请求变量，生成最终请求 | `getEffectiveRESTRequest()` |
| 网络请求 | 调用网络层发送实际请求 | `createRESTNetworkRequestStream()` |
| 响应广播 | 将响应推送到 RxJS Subject，供历史记录等模块订阅 | `executedResponses$.next(res)` |
| 后置脚本 | 执行测试脚本，可进行断言和环境变量更新 | `runPostRequestScript()` |
| 环境更新 | 根据脚本执行结果更新环境变量 | `updateEnvsAfterTestScript()` |

---

### 2.3 协议路由：KernelInterceptorService

**核心文件**: 
- `packages/hoppscotch-common/src/services/kernel-interceptor.service.ts`
- `packages/hoppscotch-common/src/platform/std/kernel-interceptors/native/index.ts`

#### 2.3.1 拦截器服务架构

KernelInterceptorService 采用**策略模式**管理不同的请求执行拦截器：

```typescript
export class KernelInterceptorService extends Service {
  private readonly state = reactive({
    interceptors: new Map<string, KernelInterceptor>(),
    currentId: null as string | null,
  })

  public execute(req: RelayRequest): ExecutionResult {
    const interceptor = this.validateAndGetActiveInterceptor()
    return interceptor.execute(req)
  }
}
```

#### 2.3.2 拦截器接口定义

```typescript
export type KernelInterceptor = {
  id: string
  name: (t: ReturnType<typeof getI18n>) => string
  capabilities: RelayCapabilities  // 声明支持的能力
  execute: (request: RelayRequest) => ExecutionResult
}
```

#### 2.3.3 Native 拦截器实现

NativeKernelInterceptorService 是默认的拦截器，负责通过内核 Relay 发送请求：

```typescript
export class NativeKernelInterceptorService extends Service implements KernelInterceptor {
  public readonly id = "native"
  
  public execute(request: RelayRequest): ExecutionResult {
    return {
      cancel: async () => { /* 取消逻辑 */ },
      response: pipe(
        this.executeRequest(request, ...),
        // 错误处理和用户友好消息转换
      )
    }
  }

  private async executeRequest(
    request: RelayRequest,
    setRelayExecution: ...
  ): Promise<E.Either<any, RelayResponse>> {
    // 1. 预处理请求 (添加 cookies, user-agent 等)
    const effectiveRequest = this.store.completeRequest(preProcessRelayRequest(request))
    
    // 2. 转换为内核原生格式
    const nativeRequest = await relayRequestToNativeAdapter(effectiveRequestWithUserAgent)
    
    // 3. 调用内核 Relay 执行
    const relayExecution = Relay.execute(postProcessedRequest)
    
    return await relayExecution.response
  }
}
```

#### 2.3.4 能力声明 (Capabilities)

每个拦截器声明其支持的能力，用于 UI 层判断哪些功能可用：

```typescript
public readonly capabilities: RelayCapabilities = {
  method: new Set(["GET", "POST", "PUT", "DELETE", "PATCH", "HEAD", "OPTIONS"]),
  header: new Set(["stringvalue", "arrayvalue", "multivalue"]),
  content: new Set(["text", "json", "xml", "form", "binary", "multipart", "urlencoded"]),
  auth: new Set(["basic", "bearer", "apikey", "digest", "aws", "hawk"]),
  security: new Set(["clientcertificates", "cacertificates"]),
  proxy: new Set(["http", "https", "authentication"]),
  advanced: new Set(["redirects", "cookies", "localaccess"]),
}
```

---

### 2.4 协议适配：RESTRequest / RESTResponse

**核心文件**: 
- `packages/hoppscotch-common/src/helpers/kernel/rest/request.ts`
- `packages/hoppscotch-common/src/helpers/kernel/rest/response.ts`

#### 2.4.1 请求适配：HoppRESTRequest → RelayRequest

```typescript
export const RESTRequest = {
  async toRequest(request: EffectiveHoppRESTRequest): Promise<RelayRequest> {
    // 转换认证信息
    const auth = await pipe(transformAuth(request.auth), ...)
    
    // 转换请求体
    const content = await pipe(transformContent(request), ...)
    
    // 过滤活跃的 header 和 params
    const headers = filterActiveToRecord(request.effectiveFinalHeaders)
    const params = filterActiveParams(request.effectiveFinalParams)

    return {
      id: Date.now(),
      url: request.effectiveFinalURL,
      method: request.method.toUpperCase() as Method,
      version: "HTTP/1.1",
      headers,
      params,
      auth,
      content,
    }
  },
}
```

#### 2.4.2 响应适配：RelayResponse → HoppRESTResponse

```typescript
export const RESTResponse = {
  async toResponse(
    response: RelayResponse,
    originalRequest: HoppRESTRequest
  ): Promise<HoppRESTSuccessResponse | HoppRESTTransformError> {
    return {
      type: "success",
      headers: processHeaders(response.headers),  // 特殊处理 Set-Cookie
      body: response.body.body.buffer,
      statusCode: response.status,
      statusText: response.statusText ?? "",
      meta: {
        responseSize: extractSize(response),
        responseDuration: extractTiming(response),
      },
      req: originalRequest,
    }
  },
}
```

**特殊处理**: Set-Cookie header 可能包含多个值，需要特殊拆分处理。

---

### 2.5 内核执行：Relay

**核心文件**: `packages/hoppscotch-common/src/kernel/relay.ts`

Relay 是内核模块的封装，通过 `window.__KERNEL__` 注入：

```typescript
export const Relay = (() => {
  const module = () => getModule("relay")

  return {
    capabilities: () => module().capabilities,
    canHandle: (request: RelayRequest): E.Either<RelayError, true> =>
      module().canHandle(request),
    execute: (request: RelayRequest) => module().execute(request),
  }
})()
```

内核实际由 `@hoppscotch/kernel` 包提供，通常是 Rust 编译的 WASM 或原生模块。

---

### 2.6 历史记录流程

#### 2.6.1 内存历史存储

**核心文件**: `packages/hoppscotch-common/src/newstore/history.ts`

历史记录采用 RxJS + DispatchingStore 模式：

```typescript
// 历史记录条目定义
export type RESTHistoryEntry = {
  v: number
  request: HoppRESTRequest
  responseMeta: {
    duration: number | null
    statusCode: number | null
  }
  star: boolean
  updatedOn?: Date
}

// 定义 dispatcher
const RESTHistoryDispatchers = defineDispatchers({
  addEntry(currentVal, { entry }) {
    return {
      state: [entry, ...currentVal.state].slice(0, HISTORY_LIMIT),  // 最多 50 条
    }
  },
  // ... 其他 dispatcher: deleteEntry, toggleStar, etc.
})

// 创建 store
export const restHistoryStore = new DispatchingStore(
  defaultRESTHistoryState,
  RESTHistoryDispatchers
)
```

#### 2.6.2 响应事件触发历史写入

```typescript
// 监听完成的响应以添加到历史记录
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

**关键点**:
- 从响应中提取 `_ref_id` 和 `id`（集合引用），历史记录是快照不应携带这些引用
- 自动添加时间戳 `updatedOn: new Date()`
- 历史记录限制为最多 50 条 (`HISTORY_LIMIT = 50`)

---

### 2.7 历史持久化：PersistenceService

**核心文件**: 
- `packages/hoppscotch-common/src/services/persistence/index.ts`
- `packages/hoppscotch-common/src/kernel/store.ts`

#### 2.7.1 持久化初始化

```typescript
export class PersistenceService extends Service {
  private async setupRESTHistoryPersistence() {
    // 1. 从存储加载历史记录
    const restLoadResult = await Store.get<any>(
      STORE_NAMESPACE,
      STORE_KEYS.REST_HISTORY
    )

    // 2. 验证并应用到内存 store
    if (E.isRight(restLoadResult)) {
      const data = restLoadResult.right ?? []
      const result = z.array(REST_HISTORY_ENTRY_SCHEMA).safeParse(data)
      if (result.success) {
        const translatedData = result.data.map(translateToNewRESTHistory)
        setRESTHistoryEntries(translatedData)
      }
    }

    // 3. 订阅内存 store 变化，自动持久化
    restHistoryStore.subject$.subscribe(async ({ state }) => {
      await Store.set(STORE_NAMESPACE, STORE_KEYS.REST_HISTORY, state)
    })
  }
}
```

#### 2.7.2 内核存储封装

Store 模块根据运行环境（Web/Desktop）选择不同的存储后端：

```typescript
// Web 模式: 使用 localStorage 路径
// Desktop 模式: 使用 Tauri 的文件存储
const HOST_SCOPED_STORE_PATH = orgParam
  ? `${orgParam.replace(/[^a-zA-Z0-9]/g, "_")}.hoppscotch.store`
  : `${window.location.host}.hoppscotch.store`

export const Store = createScopedStore(HOST_SCOPED_STORE_PATH)
```

#### 2.7.3 持久化流

```
内存 history store 变化 (restHistoryStore.subject$)
        ↓
PersistenceService 订阅回调触发
        ↓
Store.set(STORE_NAMESPACE, "restHistory", state)
        ↓
内核 store 模块写入 (localStorage 或 文件系统)
```

---

## 三、关键设计模式与技术要点

### 3.1 响应式数据流 (RxJS)

- 使用 RxJS Subject (`executedResponses$`) 实现跨模块通信
- 历史记录模块通过订阅响应事件实现解耦
- Store 使用 BehaviorSubject 管理状态，支持订阅变化

### 3.2 依赖注入 (DIOC)

- 使用 `dioc` 框架进行服务依赖注入
- `KernelInterceptorService`, `PersistenceService` 等都是可注入的服务
- 通过 `getService(ServiceClass)` 获取服务实例

### 3.3 策略模式 (拦截器)

- `KernelInterceptorService` 管理多个拦截器策略
- 每个拦截器声明自己的能力 (capabilities)
- 支持动态切换拦截器（如 Native、Agent、Extension）

### 3.4 适配器模式

- `RESTRequest.toRequest` / `RESTResponse.toResponse` 实现协议转换
- 将应用层的 `HoppRESTRequest` 转换为内核层的 `RelayRequest`
- 隔离内核变化对上层的影响

### 3.5 存储抽象

- `Store` 模块抽象了存储后端
- Web 环境使用 localStorage，Desktop 环境使用文件系统
- 上层代码无需关心具体存储实现

---

## 四、数据流时序图

```
用户点击发送
    │
    ▼
RequestRunner.runRESTRequest$
    │
    ├─► 捕获初始环境状态
    │
    ├─► 执行 Pre-request 脚本
    │
    ├─► 解析变量生成 EffectiveRequest
    │
    ├─► createRESTNetworkRequestStream
    │    │
    │    ├─► RESTRequest.toRequest (HoppRESTRequest → RelayRequest)
    │    │
    │    ├─► KernelInterceptorService.execute
    │    │    │
    │    │    └─► NativeKernelInterceptor.execute
    │    │         │
    │    │         └─► Relay.execute (内核发送)
    │    │
    │    └─► 等待响应 → RESTResponse.toResponse
    │
    ├─► executedResponses$.next(response)  ◄─── 广播响应事件
    │    │
    │    └─► history.ts 订阅者 ──► addRESTHistoryEntry ──► restHistoryStore 更新
    │                                  │
    │                                  └─► PersistenceService 订阅 ──► Store.set ──► 持久化
    │
    └─► 执行 Post-request 脚本
         │
         └─► 更新环境变量
```

---

## 五、关键文件索引

| 模块 | 文件路径 | 主要职责 |
|------|----------|----------|
| UI入口 | `src/components/http/Request.vue` | 用户交互，发送按钮事件 |
| 执行编排 | `src/helpers/RequestRunner.ts` | 协调请求生命周期，脚本执行 |
| 网络流 | `src/helpers/network.ts` | 创建网络请求 Observable |
| 协议适配 | `src/helpers/kernel/rest/request.ts` | HoppRESTRequest → RelayRequest |
| 协议适配 | `src/helpers/kernel/rest/response.ts` | RelayResponse → HoppRESTResponse |
| 拦截器管理 | `src/services/kernel-interceptor.service.ts` | 拦截器注册、选择、执行 |
| Native拦截器 | `src/platform/std/kernel-interceptors/native/index.ts` | 原生内核请求执行 |
| 历史内存存储 | `src/newstore/history.ts` | 历史记录内存状态管理 |
| 持久化服务 | `src/services/persistence/index.ts` | 状态持久化到存储 |
| 内核存储 | `src/kernel/store.ts` | 存储后端抽象 |
| 内核Relay | `src/kernel/relay.ts` | 内核请求执行封装 |

---

## 六、扩展点与可维护性考虑

1. **添加新的请求类型** (如 GraphQL, WebSocket):
   - 实现类似的 `runGQLRequest$` 编排函数
   - 创建对应的协议适配器 (`GQLRequest.toRequest`, `GQLResponse.toResponse`)
   - 添加对应的历史记录 store 和持久化配置

2. **添加新的拦截器**:
   - 实现 `KernelInterceptor` 接口
   - 声明支持的 `capabilities`
   - 调用 `kernelInterceptorService.register()` 注册

3. **修改持久化策略**:
   - 在 `PersistenceService` 中调整订阅逻辑
   - 可以添加防抖、批量写入等优化
