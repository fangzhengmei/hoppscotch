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
KernelInterceptorService.execute (协议路由: 选择当前激活的拦截器)
        ↓
[当前拦截器].execute (具体拦截器执行 - 由平台配置和用户设置共同决定)
        ↓
[实际发送逻辑: Relay.execute / 代理服务器 / Agent服务 / 浏览器扩展]
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
- `packages/hoppscotch-common/src/modules/kernel-interceptors.ts`
- `packages/hoppscotch-common/src/services/initialization.service.ts`

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
  selectable: SelectableStatus     // 是否可选择
  settingsEntry?: {                // 设置面板配置
    title: (t) => string
    component: Component
  }
  execute: (request: RelayRequest) => ExecutionResult
}
```

#### 2.3.3 拦截器注册流程

拦截器注册发生在两个阶段：

**阶段1：Desktop 模式预初始化** (`initialization.service.ts:84-92`)
```typescript
private async initNativeKernelNetworking() {
  const interceptorService = getService(KernelInterceptorService)
  const nativeInterceptorService = getService(NativeKernelInterceptorService)
  interceptorService.register(nativeInterceptorService)
  interceptorService.setActive("native")  // 临时设置为 native
  
  this.initState.nativeKernelNetworking = true
  this.emit({ type: "NATIVE_KERNEL_NETWORKING_READY" })
}
```

> **注意**：这只在 `getKernelMode() === "desktop"` 时执行，目的是在完整初始化前让认证流程能使用网络。

**阶段2：模块完整初始化** (`modules/kernel-interceptors.ts:16-68`)
```typescript
function initKernelInterceptorService(): KernelInterceptorService {
  const service = getService(KernelInterceptorService)
  
  registerInterceptors(service)      // 注册所有平台定义的拦截器
  initializeDefaultInterceptor(service)  // 设置平台默认拦截器
  
  return service
}

function registerInterceptors(service: KernelInterceptorService): void {
  platform.kernelInterceptors.interceptors.forEach((interceptorDef) => {
    if (interceptorDef.type === "standalone") {
      service.register(interceptorDef.interceptor)
    } else {
      const interceptorService = getService(interceptorDef.service)
      service.register(interceptorService)
    }
  })
}

function initializeDefaultInterceptor(service: KernelInterceptorService): void {
  service.setActive(platform.kernelInterceptors.default)
}
```

**平台定义结构** (`platform/kernel-interceptors.ts`):
```typescript
export type KernelInterceptorsPlatformDef = {
  default: string                     // 默认拦截器 ID
  interceptors: KernelInterceptorDef[]  // 拦截器列表
}
```

#### 2.3.4 设置项同步回写与回放

拦截器选择与设置系统双向同步，确保用户选择持久化：

```typescript
function setupInterceptorSync(service: KernelInterceptorService): void {
  syncServiceToSettings(service)  // 服务状态 → 设置
  syncSettingsToService(service)  // 设置 → 服务状态
}

// 服务变化时写入设置
function syncServiceToSettings(service: KernelInterceptorService): void {
  watch(
    () => service.current.value?.id,
    (id) => {
      applySetting(
        "CURRENT_KERNEL_INTERCEPTOR_ID",
        id ?? platform.kernelInterceptors.default
      )
    }
  )
}

// 设置变化时回放到服务（包含立即回放）
function syncSettingsToService(service: KernelInterceptorService): void {
  const [setting] = useSettingStatic("CURRENT_KERNEL_INTERCEPTOR_ID")

  watch(
    setting,
    () => {
      const fallback = setting.value ?? platform.kernelInterceptors.default
      service.setActive(fallback)
    },
    { immediate: true }  // 立即执行一次，回放持久化的用户选择
  )
}
```

#### 2.3.5 当前拦截器的决定因素

**不是简单的 "native 默认执行"**，而是由以下因素共同决定：

| 决定因素 | 说明 | 优先级 |
|---------|------|--------|
| **平台默认值** | `platform.kernelInterceptors.default`，由具体运行环境（Web/Desktop）配置 | 低（兜底） |
| **持久化设置** | `CURRENT_KERNEL_INTERCEPTOR_ID` 设置项，保存用户上次选择 | 中（用户偏好） |
| **拦截器可选性** | `interceptor.selectable.type` 必须为 `"selectable"`，否则自动回退 | 高（可用性约束） |
| **初始化时序** | Desktop 模式下 native 会先注册，但会被后续流程覆盖 | 特殊情况 |

**决策流程**：
```
应用启动
    │
    ├─► Desktop 模式: initNativeKernelNetworking() 临时注册 native
    │
    └─► kernel-interceptors 模块初始化
         │
         ├─► registerInterceptors() 注册所有平台拦截器
         │
         ├─► initializeDefaultInterceptor() 设置平台默认值
         │
         └─► setupInterceptorSync()
              │
              ├─► syncSettingsToService({ immediate: true })
              │    读取 CURRENT_KERNEL_INTERCEPTOR_ID 并设置
              │    └─► 如果设置为空，使用 platform.kernelInterceptors.default
              │
              └─► syncServiceToSettings() 监听后续变化
```

#### 2.3.6 拦截器有效性验证

当拦截器变为不可选择时（如扩展未安装、Agent 未启动），系统会自动回退：

```typescript
private setupInterceptorValidation(): void {
  watchEffect(() => {
    if (!this.state.currentId) return

    const currentInterceptor = this.state.interceptors.get(this.state.currentId)

    if (!this.validateCurrentInterceptor(currentInterceptor)) {
      this.resetToSelectableInterceptor()  // 寻找第一个可选择的拦截器
    }
  })
}

private resetToSelectableInterceptor(): void {
  const selectableInterceptor = this.available.value.find(
    (int) => int.selectable.type === "selectable"
  )
  this.state.currentId = selectableInterceptor?.id ?? null
}
```

---

### 2.4 拦截器执行路径对比

系统支持 5 种拦截器，执行路径和能力各不相同：

#### 2.4.1 Native 拦截器（原生内核）

**文件**: `src/platform/std/kernel-interceptors/native/index.ts`

```typescript
public async executeRequest(request: RelayRequest): Promise<E.Either<any, RelayResponse>> {
  // 1. 预处理：补全请求配置（代理、证书、重定向等）
  const effectiveRequest = this.store.completeRequest(
    preProcessRelayRequest(request)
  )

  // 2. 注入 Cookie
  const relevantCookies = this.cookieJar.getCookiesForURL(new URL(effectiveRequest.url!))
  if (relevantCookies.length > 0) {
    effectiveRequest.headers!["Cookie"] = relevantCookies.join(";")
  }

  // 3. 添加 User-Agent
  const effectiveRequestWithUserAgent = {
    ...effectiveRequest,
    headers: { ...effectiveRequest.headers, "User-Agent": "HoppscotchKernel/0.2.0" },
  }

  // 4. 转换为内核原生格式
  const nativeRequest = await relayRequestToNativeAdapter(effectiveRequestWithUserAgent)
  const postProcessedRequest = postProcessRelayRequest(nativeRequest)
  
  // 5. 内核执行
  const relayExecution = Relay.execute(postProcessedRequest)
  return await relayExecution.response
}
```

**特点**：
- 能力最全：支持所有认证方式、代理、客户端证书、重定向控制等
- 直接调用内核 Relay，性能最优
- 需要内核支持（WASM 或原生模块）

---

#### 2.4.2 Browser 拦截器（浏览器 fetch）

**文件**: `src/platform/std/kernel-interceptors/browser/index.ts`

```typescript
public execute(request: RelayRequest): ExecutionResult {
  const processedRequest = preProcessRelayRequest(request)
  const relayExecution = Relay.execute(processedRequest)  // 内核使用浏览器 fetch

  return {
    cancel: relayExecution.cancel,
    response: pipe(relayExecution.response, ...)  // 错误转换
  }
}
```

**能力限制**：
- 仅支持基本认证方式：basic, bearer, apikey
- 不支持自定义代理、客户端证书
- 受浏览器 CORS 策略限制
- 不支持高级配置（重定向控制、Cookie 管理等）

---

#### 2.4.3 Proxy 拦截器（代理服务器）

**文件**: `src/platform/std/kernel-interceptors/proxy/index.ts`

```typescript
public execute(request: RelayRequest): ExecutionResult {
  const settings = this.store.getSettings()  // proxyUrl, accessToken
  const processedRequest = preProcessRelayRequest(request)

  // 构造发往代理服务器的请求
  const proxyRequest = this.constructProxyRequest(processedRequest, accessToken)

  // 将原始请求包装后 POST 到代理服务器
  const proxyRelayRequest: RelayRequest = {
    id: Date.now(),
    url: proxyUrl,
    method: "POST",
    headers: { "content-type": "application/json" },
    content: { kind: "json", content: proxyRequest, mediaType: MediaType.APPLICATION_JSON },
  }

  const relayExecution = Relay.execute(proxyRelayRequest)

  // 解析代理响应，转换为原始响应格式
  const response = pipe(relayExecution.response,
    E.map((res) => {
      const proxyResponse = parseBytesToJSON<ProxyResponse>(res.body.body)
      return E.right({
        ...res,
        status: proxyResponse.status,
        statusText: proxyResponse.statusText,
        headers: proxyResponse.headers,
        body: { body: decodeProxyData(proxyResponse.data), ... },
      })
    })
  )

  return { cancel: relayExecution.cancel, response }
}
```

**特点**：
- 请求经过代理服务器转发，可绕过 CORS
- 能力受限：仅支持 text 内容、basic 认证
- 需要用户配置代理服务器地址和访问令牌
- 额外的网络延迟（多一跳）

---

#### 2.4.4 Agent 拦截器（本地 Agent 服务）

**文件**: `src/platform/std/kernel-interceptors/agent/index.ts`

```typescript
public execute(request: RelayRequest): ExecutionResult {
  const reqID = Date.now()
  const cancelToken = axios.CancelToken.source()

  return {
    cancel: async () => {
      cancelToken.cancel()
      await this.store.cancelRequest(reqID)
    },
    response: pipe(
      this.executeRequest(request, reqID, cancelToken),  // 通过 axios 发送到 Agent
      ...
    )
  }
}
```

**特点**：
- 通过 HTTP 与本地运行的 Hoppscotch Agent 通信
- 能力接近 Native，支持完整功能
- Agent 需单独安装运行
- 适合需要本地网络访问的场景

---

#### 2.4.5 Extension 拦截器（浏览器扩展）

**文件**: `src/platform/std/kernel-interceptors/extension/index.ts`

```typescript
public execute(request: RelayRequest): ExecutionResult {
  const extensionHook = window.__POSTWOMAN_EXTENSION_HOOK__
  
  return {
    cancel: () => extensionHook?.cancelRequest(),
    response: new Promise((resolve) => {
      extensionHook.sendRequest(processedRequest, (response) => {
        resolve(E.right(transformExtensionResponse(response)))
      })
    })
  }
}
```

**特点**：
- 依赖浏览器扩展提供的 `window.__POSTWOMAN_EXTENSION_HOOK__`
- 能力由扩展实现决定
- 可绕过浏览器 CORS 限制
- 用户需安装浏览器扩展

---

#### 2.4.6 拦截器能力对比表

| 能力 | Native | Browser | Proxy | Agent | Extension |
|------|--------|---------|-------|-------|-----------|
| HTTP 方法 | 全部 | 全部 | 全部 | 全部 | 全部 |
| Header 支持 | 完整 | 基础 | 基础 | 完整 | 完整 |
| 内容类型 | 全部 | 常用 | Text | 全部 | 全部 |
| 认证方式 | 全部 | 3种 | Basic | 全部 | 3种 |
| 客户端证书 | ✅ | ❌ | ❌ | ✅ | ❌ |
| 自定义代理 | ✅ | ❌ | - | ✅ | ❌ |
| 重定向控制 | ✅ | ❌ | ❌ | ✅ | ❌ |
| Cookie 管理 | ✅ | ❌ | ❌ | ✅ | 取决于扩展 |
| 绕过 CORS | ❌ | ❌ | ✅ | ✅ | ✅ |
| 需要额外安装 | 内核 | 无 | 代理服务 | Agent | 浏览器扩展 |

---

### 2.5 协议适配：RESTRequest / RESTResponse

**核心文件**: 
- `packages/hoppscotch-common/src/helpers/kernel/rest/request.ts`
- `packages/hoppscotch-common/src/helpers/kernel/rest/response.ts`

#### 2.5.1 请求适配：HoppRESTRequest → RelayRequest

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

#### 2.5.2 响应适配：RelayResponse → HoppRESTResponse

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

### 2.6 内核执行：Relay

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

### 2.7 历史记录流程

#### 2.7.1 内存历史存储

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

#### 2.7.2 响应事件触发历史写入

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

### 2.8 历史持久化：PersistenceService

**核心文件**: 
- `packages/hoppscotch-common/src/services/persistence/index.ts`
- `packages/hoppscotch-common/src/kernel/store.ts`

#### 2.8.1 持久化初始化

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

#### 2.8.2 内核存储封装

Store 模块根据运行环境（Web/Desktop）选择不同的存储后端：

```typescript
// Web 模式: 使用 localStorage 路径
// Desktop 模式: 使用 Tauri 的文件存储
const HOST_SCOPED_STORE_PATH = orgParam
  ? `${orgParam.replace(/[^a-zA-Z0-9]/g, "_")}.hoppscotch.store`
  : `${window.location.host}.hoppscotch.store`

export const Store = createScopedStore(HOST_SCOPED_STORE_PATH)
```

#### 2.8.3 持久化流

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
- 每个拦截器声明自己的能力 (capabilities) 和可选性 (selectable)
- 支持动态切换拦截器（Native、Browser、Proxy、Agent、Extension）
- 拦截器失效时自动回退到可用的拦截器

### 3.4 适配器模式

- `RESTRequest.toRequest` / `RESTResponse.toResponse` 实现协议转换
- 将应用层的 `HoppRESTRequest` 转换为内核层的 `RelayRequest`
- 隔离内核变化对上层的影响

### 3.5 存储抽象

- `Store` 模块抽象了存储后端
- Web 环境使用 localStorage，Desktop 环境使用文件系统
- 上层代码无需关心具体存储实现

### 3.6 双向同步模式

- 拦截器选择与设置系统双向同步
- `immediate: true` 确保启动时回放入户选择
- 服务状态变化自动持久化到设置

---

## 四、完整数据流时序图

```
应用启动初始化
    │
    ├─► [Desktop] initNativeKernelNetworking()
    │    ├─► 注册 Native 拦截器
    │    └─► 临时设置为 active
    │
    └─► kernel-interceptors 模块初始化
         ├─► registerInterceptors() - 注册所有平台拦截器
         ├─► initializeDefaultInterceptor() - 设置平台默认
         └─► setupInterceptorSync()
              ├─► syncSettingsToService({ immediate: true })
              │    └─► 读取 CURRENT_KERNEL_INTERCEPTOR_ID 设置
              │         └─► 有值则使用，否则使用平台默认
              └─► syncServiceToSettings() - 监听后续变化

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
    │    │    └─► [当前激活拦截器].execute
    │    │         │
    │    │         ├─► Native: Relay.execute(处理后的请求)
    │    │         ├─► Browser: Relay.execute(原始请求)
    │    │         ├─► Proxy: Relay.execute(包装后的代理请求)
    │    │         ├─► Agent: axios → 本地Agent服务
    │    │         └─► Extension: window.__POSTWOMAN_EXTENSION_HOOK__
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
| 拦截器管理 | `src/services/kernel-interceptor.service.ts` | 拦截器注册、选择、执行、回退 |
| 拦截器初始化 | `src/modules/kernel-interceptors.ts` | 拦截器注册、默认值设置、双向同步 |
| 初始化服务 | `src/services/initialization.service.ts` | Desktop 模式预初始化 Native |
| Native拦截器 | `src/platform/std/kernel-interceptors/native/index.ts` | 原生内核请求执行 |
| Browser拦截器 | `src/platform/std/kernel-interceptors/browser/index.ts` | 浏览器 fetch 执行 |
| Proxy拦截器 | `src/platform/std/kernel-interceptors/proxy/index.ts` | 代理服务器转发 |
| Agent拦截器 | `src/platform/std/kernel-interceptors/agent/index.ts` | 本地 Agent 服务 |
| Extension拦截器 | `src/platform/std/kernel-interceptors/extension/index.ts` | 浏览器扩展 |
| 历史内存存储 | `src/newstore/history.ts` | 历史记录内存状态管理 |
| 持久化服务 | `src/services/persistence/index.ts` | 状态持久化到存储 |
| 内核存储 | `src/kernel/store.ts` | 存储后端抽象 |
| 内核Relay | `src/kernel/relay.ts` | 内核请求执行封装 |
| 平台定义 | `src/platform/kernel-interceptors.ts` | 拦截器平台配置类型 |

---

## 六、扩展点与可维护性考虑

### 6.1 添加新的请求类型 (如 GraphQL, WebSocket)
- 实现类似的 `runGQLRequest$` 编排函数
- 创建对应的协议适配器 (`GQLRequest.toRequest`, `GQLResponse.toResponse`)
- 添加对应的历史记录 store 和持久化配置

### 6.2 添加新的拦截器
- 实现 `KernelInterceptor` 接口
- 声明支持的 `capabilities` 和 `selectable` 状态
- 在平台定义中注册拦截器
- 如有需要，提供设置面板组件 (`settingsEntry`)

### 6.3 修改持久化策略
- 在 `PersistenceService` 中调整订阅逻辑
- 可以添加防抖、批量写入等优化

### 6.4 拦截器回退机制扩展
- 当前仅在 `selectable` 变化时触发回退
- 可扩展为在执行失败时自动尝试下一个可用拦截器
- 需要考虑用户体验（透明重试 vs 明确告知）
