# 前置脚本与测试脚本沙箱执行环境分析

## 一、沙箱架构概述

Hoppscotch 使用 **`faraday-cage`** 库（基于 QuickJS 引擎）实现脚本沙箱执行环境，支持两种执行模式：

| 模式 | 技术栈 | 适用场景 |
|------|--------|----------|
| **实验性沙箱（默认）** | faraday-cage + QuickJS | 新版功能，支持完整 API |
| **Legacy 沙箱** | Web Worker（浏览器）/ isolated-vm（Node.js） | 向后兼容 |

核心代码位于 `packages/hoppscotch-js-sandbox/` 目录。

---

## 二、运行时注入的对象

### 2.1 核心命名空间

沙箱通过 `bootstrap-code` 引导代码在 `globalThis` 上注入三个核心命名空间：

#### `hopp` 命名空间（Hoppscotch 原生 API）
```javascript
// 环境变量管理
hopp.env.get(key)           // 获取环境变量（已解析模板）
hopp.env.getRaw(key)        // 获取原始值（不解析模板）
hopp.env.set(key, value)    // 设置环境变量
hopp.env.delete(key)        // 删除环境变量
hopp.env.reset(key)         // 重置为初始值
hopp.env.getInitialRaw(key) // 获取初始值
hopp.env.setInitial(key, value)

// 分层环境管理
hopp.env.active.*   // 仅操作选中环境
hopp.env.global.*   // 仅操作全局环境

// 请求操作（前置脚本）
hopp.request.url                // 只读
hopp.request.method             // 只读
hopp.request.headers            // 只读
hopp.request.setUrl(url)
hopp.request.setMethod(method)
hopp.request.setHeader(name, value)
hopp.request.setHeaders(headers)
hopp.request.removeHeader(key)
hopp.request.setParam(name, value)
hopp.request.setBody(body)
hopp.request.setAuth(auth)

// 请求变量
hopp.request.variables.get(key)
hopp.request.variables.set(key, value)

// Cookie 管理（仅桌面端）
hopp.cookies.get(domain, name)
hopp.cookies.set(domain, cookie)
hopp.cookies.has(domain, name)
hopp.cookies.getAll(domain)
hopp.cookies.delete(domain, name)
hopp.cookies.clear(domain)

// 网络请求
hopp.fetch(url, options)        // 封装的 fetch API
```

**代码位置**：
- 引导代码：`src/bootstrap-code/pre-request.js:88-186`
- 命名空间实现：`src/cage-modules/namespaces/hopp-namespace.ts:10-76`

---

#### `pw` 命名空间（旧版兼容 API）
```javascript
pw.env.get(key)
pw.env.getResolve(key)    // 解析模板后的值
pw.env.set(key, value)
pw.env.unset(key)
pw.env.resolve(key)
```

**代码位置**：`src/bootstrap-code/pre-request.js:27-35`

---

#### `pm` 命名空间（Postman 兼容层）
```javascript
// 环境变量
pm.environment.get(key)
pm.environment.set(key, value)    // 支持任意类型（数组、对象等）
pm.environment.unset(key)
pm.environment.has(key)
pm.environment.clear()
pm.environment.toObject()

// 全局变量
pm.globals.get/set/unset/has/clear/toObject()

// 变量（优先级：选中 > 全局）
pm.variables.get/set/has/replaceIn()

// 请求信息
pm.request.id           // 只读
pm.request.name         // 只读
pm.request.url          // 可修改（Postman 兼容 URL 对象）
pm.request.method       // 可修改
pm.request.headers      // 可修改（PropertyList 接口）
pm.request.body         // 可修改
pm.request.auth         // 可修改

// 脚本上下文
pm.info.eventName       // "pre-request" 或 "test"
pm.info.requestName
pm.info.requestId

// 发送请求
pm.sendRequest(urlOrRequest, callback)

// 响应（测试脚本）
pm.response.code
pm.response.status
pm.response.headers
pm.response.body
pm.response.text()
pm.response.json()
pm.response.responseTime
pm.response.responseSize
pm.response.to.be.*     // Chai 断言链
pm.expect(value)        // Chai 风格断言
```

**代码位置**：
- 前置脚本：`src/bootstrap-code/pre-request.js:194-1431`
- 测试脚本：`src/bootstrap-code/post-request.js`
- 命名空间实现：`src/cage-modules/namespaces/pm-namespace.ts:10-24`

---

### 2.2 全局 API 注入

#### 标准 JavaScript API
| API | 说明 | 代码位置 |
|-----|------|----------|
| `console` | 沙箱化控制台，支持 log/error/warn/table/dir 等 | `src/cage-modules/default.ts:23-61` |
| `fetch` | 封装的网络请求，可通过 `hoppFetchHook` 拦截 | `src/cage-modules/fetch.ts:79-927` |
| `Headers` | 沙箱化 Headers 类 | `src/cage-modules/fetch.ts:936-1080` |
| `Request` | 沙箱化 Request 类 | `src/cage-modules/fetch.ts:1113-1272` |
| `Response` | 沙箱化 Response 类 | `src/cage-modules/fetch.ts:1297-1350+` |
| `crypto` | Web Crypto API 封装 | `src/cage-modules/crypto.ts:47-1030` |
| `crypto.subtle` | 支持 digest/encrypt/decrypt/sign/verify 等 | `src/cage-modules/crypto.ts:219-1019` |
| `Blob` | Polyfill | `src/cage-modules/default.ts:2` |
| `URL` | Polyfill | `src/cage-modules/default.ts:8` |
| `atob` / `btoa` | Base64 编解码 | `src/cage-modules/default.ts:72` |
| `setTimeout` / `setInterval` | 定时器 | `src/cage-modules/default.ts:73` |
| `TextEncoder` / `TextDecoder` | 编解码 | `src/cage-modules/default.ts:72` |

---

### 2.3 测试脚本专属注入（post-request）

```javascript
// Chai 风格断言
pm.expect(value).to.equal(expected)
pm.expect(value).to.eql(expected)
pm.expect(value).to.be.a('string')
pm.expect(value).to.have.lengthOf(5)
pm.expect(value).to.include('foo')
pm.expect(value).to.be.above(10)
pm.expect(value).to.be.below(100)
pm.expect(value).to.be.within(1, 10)
pm.expect(value).to.match(/regex/)
pm.expect(value).to.throw()
pm.expect(value).to.be.instanceOf(Constructor)
// ... 更多 Chai 断言方法

// 响应状态断言
pm.expect(pm.response.code).to.be.level2xx
pm.expect(pm.response.code).to.be.level3xx
pm.expect(pm.response.code).to.be.level4xx
pm.expect(pm.response.code).to.be.level5xx

// 测试块
pm.test("测试描述", () => {
  // 断言语句
})
```

**代码位置**：`src/bootstrap-code/post-request.js`

---

## 三、可变作用域分析

### 3.1 作用域隔离机制

沙箱使用 **QuickJS 虚拟机** 实现完全隔离的作用域：

1. **每次执行创建新上下文**：`faraday-cage` 的 `runCode()` 每次调用创建独立的 QuickJS 运行时
2. **宿主对象通过句柄传递**：所有宿主对象通过 `defineSandboxFn` / `defineSandboxObject` 包装后注入
3. **值序列化边界**：跨边界值通过 `ctx.vm.dump()` 和 `marshalValue` 序列化

#### VM 单例管理与错误重试

**生产环境单例模式**：
```typescript
// 代码位置：src/utils/cage.ts:34-54
let cagePromise: Promise<FaradayCage> | null = null

export const acquireCage = async (): Promise<FaradayCage> => {
  if (!cagePromise) {
    cagePromise = FaradayCage.create().catch((err) => {
      cagePromise = null
      throw err
    })
  }
  return cagePromise
}
```

**测试环境独立实例**：
- 测试环境每次调用 `acquireCage()` 创建新实例
- 避免测试间状态污染

**基础设施错误重试机制**：
```typescript
// 代码位置：src/utils/cage.ts:22-23
export const isInfraError = (err: unknown): boolean => err instanceof Error
```

错误类型区分：
| 错误类型 | 判断标准 | 处理方式 |
|---------|---------|---------|
| **用户脚本错误** | `result.err` 是普通对象（非 `instanceof Error`） | 直接返回错误信息 |
| **基础设施错误** | `err instanceof Error`（QuickJSUnwrapError、marshal 失败、WASM 初始化失败等） | 重置沙箱并重试一次 |

**重试流程**（`src/web/pre-request/index.ts:89-173`）：
1. 首次执行遇到基础设施错误 → `resetCage()` → 返回 "retry"
2. 调用方创建新的 FaradayCage 实例并重试
3. 连续两次失败则终止并上报错误

**代码位置**：
- 上下文管理：`src/utils/cage.ts:34-54`
- 沙箱函数包装：`src/cage-modules/scripting-modules.ts:213-280`
- 重试逻辑：`src/web/pre-request/index.ts:89-173`

---

### 3.2 可变状态分类

#### 环境变量（Environment Variables）
```typescript
// 内部表示（执行期间）
type SandboxEnvs = {
  global: SandboxEnvironmentVariable[]  // 全局环境
  selected: SandboxEnvironmentVariable[] // 选中环境
}

type SandboxEnvironmentVariable = {
  key: string
  currentValue: SandboxValue  // 执行期间可是任意类型
  initialValue: SandboxValue  // 初始值
  secret: boolean
}
```

**读写路径**：
1. 进入沙箱：`cloneDeep(envs)` → 深拷贝传入（`scripting-modules.ts:70,80`）
2. 执行期间：直接修改 `updatedEnvs` 引用（`shared.ts:211`）
3. 离开沙箱：`getUpdatedEnvs()` → 序列化为字符串（`base-inputs.ts:128-175`）

**类型转换特殊处理**：
- `undefined` → `__HOPPSCOTCH_UNDEFINED__` 标记
- `null` → `__HOPPSCOTCH_NULL__` 标记
- 对象/数组 → `JSON.stringify()` 序列化
- 其他非字符串 → `String()` 转换

#### 值序列化与跨边界传输机制

沙箱与宿主之间通过 `marshalValue` 进行值的序列化与反序列化，确保 QuickJS 虚拟机与宿主环境之间的安全数据传输：

```typescript
// 代码位置：src/cage-modules/utils/vm-marshal.ts:5-46
export const marshalValue = (ctx: CageModuleContext, value: any): any => {
  if (value === null) return ctx.vm.null
  if (value === undefined) return ctx.vm.undefined
  if (value === true) return ctx.vm.true
  if (value === false) return ctx.vm.false
  if (typeof value === "string") return ctx.scope.manage(ctx.vm.newString(value))
  if (typeof value === "number") return ctx.scope.manage(ctx.vm.newNumber(value))
  // ... 对象和数组递归处理
}
```

**特殊类型处理**：

| 类型 | 处理方式 | 原因 |
|------|---------|------|
| `Uint8Array` / `ArrayBuffer` | 转换为普通数组 + `byteLength` 属性 | QuickJS 不支持原生 TypedArrays/ArrayBuffer |
| 普通对象 | 递归转换为 QuickJS 对象 | 防止引用逃逸 |
| 数组 | 递归转换为 QuickJS 数组 | 确保元素也被正确序列化 |

**沙箱标记常量**（`src/constants/sandbox-markers.ts:10-11`）：
```typescript
export const UNDEFINED_MARKER = "__HOPPSCOTCH_UNDEFINED__" as const
export const NULL_MARKER = "__HOPPSCOTCH_NULL__" as const
```

**标记同步机制**：
- 引导代码 `bootstrap-code/pre-request.js:78-79` 中硬编码相同的标记值
- 注释明确要求必须与 `sandbox-markers.ts` 保持一致
- 无法直接导入，因为引导代码运行在 QuickJS 沙箱中

**序列化流转图**：
```
宿主值 → marshalValue → QuickJS VM 值 → 用户脚本执行
                                                        ↓
用户脚本修改 → ctx.vm.dump() → 宿主值 → 标记转换 → 最终结果
```

#### 环境变量类型转换深度分析

**执行期间类型保持**（`src/utils/shared.ts:190-220`）：
```typescript
// 外部 API 使用 string 类型
type TestResult["envs"] = {
  global: { key: string; currentValue: string; ... }[]
  selected: { key: string; currentValue: string; ... }[]
}

// 执行期间实际存储 SandboxValue（任意类型）
type SandboxEnvs = {
  global: { key: string; currentValue: SandboxValue; ... }[]
  selected: { key: string; currentValue: SandboxValue; ... }[]
}
```

**PM 命名空间特殊处理**：
```typescript
// 代码位置：src/cage-modules/utils/base-inputs.ts:111-117
// PM 专用 setter，支持任意类型（数组、对象等）
const pmEnvSetAny = defineSandboxFn(
  ctx,
  "pmEnvSetAny",
  function (key: SandboxValue, value: SandboxValue, options: SandboxValue) {
    return pmSetAny(key, value, options)
  }
)
```

**PM vs Hopp 命名空间差异**：

| 特性 | `hopp.env.set(key, value)` | `pm.environment.set(key, value)` |
|------|---------------------------|----------------------------------|
| 接受类型 | 仅字符串 | 任意类型（数组、对象、数字等） |
| 类型保持 | 不保持（转换为字符串） | 保持原始类型（执行期间） |
| 序列化时机 | 设置时转换 | 离开沙箱时转换 |

**离开沙箱时的序列化逻辑**（`src/cage-modules/utils/base-inputs.ts:128-175`）：
```typescript
getUpdatedEnvs: () => {
  const convertMarkersToStrings = (env: SandboxValue) => {
    const convertValue = (value: SandboxValue) => {
      // 1. 标记转换
      if (value === UNDEFINED_MARKER) return "undefined"
      if (value === NULL_MARKER) return "null"
      
      // 2. 对象/数组 → JSON 字符串
      if (typeof value === "object" && value !== null) {
        try {
          return JSON.stringify(value)
        } catch (_) {
          return String(value)
        }
      }
      
      // 3. 非字符串 → String() 转换
      // Vue UI 调用 .match() 方法，仅字符串支持
      if (typeof value !== "string") {
        return String(value)
      }
      
      // 4. 字符串原样返回
      return value
    }
    // ...
  }
}
```

**强制字符串转换原因**：
- Vue UI 对环境变量值调用 `.match()` 方法进行模板解析
- `.match()` 仅存在于 String 原型上
- 非字符串值会导致运行时错误
- 统一转换为字符串确保 UI 兼容性

**完整类型流转**：
```
外部传入（string）
    ↓ cloneDeep
沙箱执行（SandboxValue：string/number/object/array/null/undefined）
    ↓ pmEnvSetAny 保持类型
执行期间（任意类型操作）
    ↓ getUpdatedEnvs() 序列化
离开沙箱（全部转换为 string）
    ↓
宿主使用（string）
```

**代码位置**：
- 核心逻辑：`src/utils/shared.ts:136-497`
- 标记常量：`src/constants/sandbox-markers.ts:10-11`
- 序列化实现：`src/cage-modules/utils/vm-marshal.ts:5-46`
- PM 类型保持：`src/cage-modules/utils/base-inputs.ts:111-117`
- 输出序列化：`src/cage-modules/utils/base-inputs.ts:128-175`

---

#### 请求对象（Request）
仅前置脚本可修改：

```typescript
// 可修改属性
request.endpoint      // 通过 setRequestUrl
request.method        // 通过 setRequestMethod
request.headers       // 通过 setRequestHeader / setRequestHeaders
request.params        // 通过 setRequestParam / setRequestParams
request.body          // 通过 setRequestBody
request.auth          // 通过 setRequestAuth
request.requestVariables // 通过 setRequestVariable
```

**读写路径**：
1. 进入沙箱：`cloneDeep(request)` 传入
2. 执行期间：`createRequestSetterMethods` 维护内部 `updatedRequest` 引用
3. 离开沙箱：`getUpdatedRequest()` 返回修改后的对象

**代码位置**：
- Setter 实现：`src/cage-modules/utils/request-setters.ts:17-96`
- 结果捕获：`src/cage-modules/scripting-modules.ts:441-457`

---

#### Cookie（仅桌面端）
```typescript
type Cookie = {
  domain: string
  name: string
  value: string
  path?: string
  expires?: string
  httpOnly?: boolean
  secure?: boolean
}
```

**读写路径**：
1. 进入沙箱：`cloneDeep(cookies)` 传入
2. 执行期间：`getSharedCookieMethods` 维护 `updatedCookies` 数组
3. 离开沙箱：`getUpdatedCookies()` 返回

**代码位置**：`src/utils/shared.ts:499-593`

---

#### 测试运行栈（Test Run Stack）
仅测试脚本使用：

```typescript
type TestDescriptor = {
  descriptor: string           // 测试块名称
  expectResults: ExpectResult[] // 断言结果
  children: TestDescriptor[]    // 子测试块
}
```

**执行流程**：
1. 初始化根节点：`{ descriptor: "root", expectResults: [], children: [] }`
2. `pm.test()` 调用时压入栈，执行后弹出
3. 断言结果追加到当前测试块的 `expectResults`

**代码位置**：
- 栈管理：`src/cage-modules/scripting-modules.ts:187-274`
- 断言方法：`src/utils/shared.ts:652-918`

#### 测试结果深拷贝与 UI 反应性隔离

**问题场景**：
- 异步测试回调可能在结果捕获后仍在执行
- 测试结果对象直接传递给 Vue UI 层
- Vue 的响应式系统会监听对象变化
- 异步回调对 `testRunStack` 的后续修改会导致 UI 闪烁

**解决方案**（`src/cage-modules/scripting-modules.ts:462-474`）：
```typescript
captureHook.capture = () => {
  // Deep clone testRunStack to prevent UI reactivity to async mutations
  // Without this, async test callbacks that complete after capture will mutate
  // the same object being displayed in the UI, causing flickering test results

  postConfig.handleSandboxResults({
    envs: postInputs.getUpdatedEnvs() || { global: [], selected: [] },
    testRunStack: cloneDeep(postConfig.testRunStack),  // 关键：深拷贝
    cookies: postInputs.getUpdatedCookies() || null,
  })
}
```

**设计考量**：

| 方案 | 优点 | 缺点 |
|-----|------|------|
| **深拷贝** | 完全隔离，UI 稳定性高 | 有性能开销（测试结果通常较小） |
| **Object.freeze** | 性能好，防止修改 | 异步回调修改会抛出错误 |
| **不处理** | 无开销 | UI 闪烁，用户体验差 |

**选择深拷贝的原因**：
1. 测试结果数据量通常很小（几十到几百个测试用例）
2. 性能开销可以忽略不计
3. 确保异步回调的后续修改不会影响已展示的结果
4. 避免 Vue 响应式系统追踪不必要的变化
5. 保证测试结果的一致性（捕获时的快照）

**时序图**：
```
pm.test("async test", async () => { ... })
    ↓
同步执行完成 → captureHook.capture() 被调用
    ↓
cloneDeep(testRunStack) → 返回给宿主
    ↓
UI 渲染测试结果（稳定的快照）
    ↓
异步回调继续执行 → 修改原始 testRunStack
    ↓
（无影响）因为 UI 使用的是深拷贝后的副本
```

---

## 四、外部依赖访问限制

### 4.1 网络访问限制

#### Fetch API 封装
```typescript
// 可通过 hoppFetchHook 拦截所有网络请求
type HoppFetchHook = (
  input: RequestInfo | URL,
  init?: RequestInit
) => Promise<Response>
```

**拦截场景**：
- **Web 应用**：通过 `KernelInterceptorService` 路由，尊重代理/拦截器设置
- **CLI**：使用 axios 直接请求
- **测试**：可注入 mock 实现

**安全措施**：
1. 不暴露原生 `fetch`，所有请求经过封装层
2. `Headers` / `Request` / `Response` 全部重新实现，不直接暴露原生对象
3. 响应体预读为 `_bodyBytes`，防止流式访问逃逸

**代码位置**：`src/cage-modules/fetch.ts:27-1350`

---

### 4.2 宿主环境隔离

| 环境 | 禁止访问 | 原因 |
|------|----------|------|
| **浏览器** | `window`, `document`, `DOM API`, `localStorage` | 防止 XSS 和数据泄露 |
| **Node.js** | `fs`, `process`, `require`, `child_process` | 防止文件系统访问和命令执行 |
| **通用** | 宿主 `globalThis` 原型链 | 防止原型污染 |

**隔离实现**：
1. QuickJS 本身不提供这些 API
2. `faraday-cage` 仅显式注入白名单内的对象
3. 引导代码使用 `"use strict"` 防止意外的全局变量泄漏

---

### 4.3 加密 API 限制

```typescript
// getRandomValues 大小限制（Web Crypto 规范）
const MAX_GET_RANDOM_VALUES_SIZE = 65536
```

**安全措施**：
1. `CryptoKey` 不穿越沙箱边界，通过 `CryptoKeyRegistry` 在宿主侧存储
2. 沙箱内仅持有 `__keyId` 引用
3. 所有加密操作实际在宿主侧执行
4. 支持的算法：digest, encrypt, decrypt, sign, verify, generateKey, importKey, exportKey, deriveBits, deriveKey, wrapKey, unwrapKey

#### CryptoKey 注册表深度分析

**存储结构**（`src/cage-modules/utils/vm-marshal.ts:82-235`）：
```typescript
interface KeyEntry {
  ref: WeakRef<CryptoKey | CryptoKeyPair> | CryptoKey | CryptoKeyPair
  strongRef: CryptoKey | CryptoKeyPair
  expiresAt: number
}

const KEY_EXPIRY_MS = 5 * 60 * 1000 // 5 分钟过期

export class CryptoKeyRegistry {
  private keys = new Map<string, KeyEntry>()
  private finalizer?: FinalizationRegistry<string>
  // ...
}
```

**密钥生命周期管理**：

| 阶段 | 处理方式 | 安全机制 |
|-----|---------|---------|
| **存储** | `store(key, ttl)` 生成 UUID，存储在 `Map<string, KeyEntry>` | 支持 `WeakRef` 优化内存，同时保留 `strongRef` 防止意外回收 |
| **访问** | `get(id)` 检查过期时间，重置 TTL | 访问时自动续期 5 分钟 |
| **过期** | 定时清理线程每 5 分钟检查一次 | 过期密钥自动删除 |
| **垃圾回收** | 使用 `FinalizationRegistry` 监听密钥对象回收 | 密钥被 GC 时自动从注册表删除 |

**UUID 生成策略**：
```typescript
// 优先使用 crypto.randomUUID()
// 降级到 getRandomValues() 手动生成 RFC 4122 v4 UUID
// 最后降级到 Date.now() + Math.random()
```

**沙箱边界交互**：
```
沙箱内调用 crypto.subtle.generateKey()
        ↓
宿主侧执行真实操作生成 CryptoKey
        ↓
存储到 CryptoKeyRegistry，返回 keyId
        ↓
沙箱内得到 { __keyId: "uuid", type: "CryptoKey", ... }
        ↓
沙箱内使用 __keyId 引用进行后续操作
        ↓
宿主侧根据 __keyId 查找真实密钥执行操作
```

**内存安全特性**：
- 支持 `WeakRef` 的环境中优先使用弱引用
- 保留强引用作为 fallback，确保密钥在使用期间不被回收
- `FinalizationRegistry` 确保对象回收时清理注册表
- 定期清理过期密钥防止内存泄漏

**代码位置**：
- 密钥注册：`src/cage-modules/utils/vm-marshal.ts:82-235`
- 限制常量：`src/cage-modules/crypto.ts:7`（引用自 vm-marshal）
- 注册表实现：`src/cage-modules/utils/vm-marshal.ts:90-235`

---

### 4.4 错误报告安全

```javascript
// 引导代码中锁定错误报告函数
Object.defineProperty(globalThis, "__hoppReportScriptExecutionError", {
  value: (error) => { /* ... */ },
  enumerable: false,
  configurable: false,  // 不可删除
  writable: false,      // 不可覆盖
})
```

**防止**：
- 用户脚本删除或篡改错误报告函数
- 抑制执行错误的上报

**代码位置**：
- 前置脚本：`src/bootstrap-code/pre-request.js:12-25`
- 测试脚本：`src/bootstrap-code/post-request.js:12-25`

### 4.5 只读保护与对象冻结

**请求属性只读保护**（`src/bootstrap-code/pre-request.js:58-73`）：
```javascript
// 定义所有属性 with unified read-only protection
;["url", "method", "params", "headers", "body", "auth"].forEach((prop) => {
  Object.defineProperty(requestProps, prop, {
    enumerable: true,
    configurable: false,
    get() {
      const currentValues = inputs.getRequestProps()
      return currentValues[prop]
    },
    set(_value) {
      throw new TypeError(`hopp.request.${prop} is read-only`)
    },
  })
})

// 冻结整个 requestProps 对象
Object.freeze(requestProps)
```

**保护层级**：

| 层级 | 机制 | 防止的攻击 |
|-----|------|-----------|
| **属性描述符** | `configurable: false` | 防止用户脚本重新定义属性、删除属性 |
| **只读 getter** | 无 setter，访问时动态获取 | 防止直接赋值修改，确保读取最新值 |
| **显式抛出** | setter 抛出 `TypeError` | 提供清晰的错误提示，避免静默失败 |
| **对象冻结** | `Object.freeze(requestProps)` | 防止添加/删除属性，防止修改原型 |

**严格模式隔离**（`src/bootstrap-code/pre-request.js:4`）：
```javascript
;(inputs) => {
  // Keep strict mode scoped to this IIFE to avoid leaking strictness
  "use strict"
  // ...
}
```

**严格模式设计考量**：
- 仅限制在 IIFE 内部，不泄漏到用户脚本
- 防止意外的全局变量泄漏
- 避免静默的赋值失败（只读属性赋值会抛出错误）
- 提升代码安全性

**引导代码执行上下文**：
- 引导代码本身运行在严格模式下
- 用户脚本继承严格模式吗？不，因为 `"use strict"` 仅在 IIFE 内部有效
- 用户脚本可以选择自己的严格模式

---

## 五、跨请求状态保持

### 5.1 状态流转模型

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   请求 1 脚本    │     │   宿主环境      │     │   请求 2 脚本    │
│                 │     │                 │     │                 │
│  envs (修改) ───┼────►│  handleSandbox- │     │  envs (新值)    │
│  cookies (修改) ┼────►│  Results 回调   ├────►│  cookies (新值) │
│  request (修改) ┼────►│                 │     │                 │
│                 │     │  持久化存储 ◄───┼─────┤                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
         │                        │                        │
         └────────────────────────┼────────────────────────┘
                                  ▼
                        每次执行独立 VM 上下文
                        无隐式状态共享
```

---

### 5.2 状态传递机制

#### 前置脚本 → 请求
```typescript
// 捕获时机：cage.runCode() 完成后调用 captureHook.capture()
// 代码位置：src/web/pre-request/index.ts:120-122
if (captureHook.capture) {
  captureHook.capture()
}

// 返回结果
return E.right({
  updatedEnvs: finalEnvs,
  consoleEntries,
  updatedRequest: finalRequest,
  updatedCookies: finalCookies,
})
```

#### 测试脚本 → 后续请求
```typescript
// 代码位置：src/web/test-runner/index.ts:140-142
if (captureHook.capture) {
  captureHook.capture()
}

// 返回结果
return E.right({
  tests: safeTestResults,
  envs: safeEnvs,
  consoleEntries: safeConsoleEntries,
  updatedCookies: safeCookies,
})
```

---

### 5.3 关键特性

| 特性 | 说明 |
|------|------|
| **无隐式共享** | 每次脚本执行使用独立的 QuickJS 上下文，沙箱内全局变量不跨请求 |
| **显式传递** | 只有 `envs` 和 `cookies` 通过返回值显式传递 |
| **深拷贝隔离** | 进入沙箱的所有数据都经过 `cloneDeep`，避免引用共享 |
| **异步等待** | 通过 `keepAlivePromises` 等待异步操作完成后再捕获结果 |

**异步等待实现**：
```typescript
// 代码位置：src/cage-modules/scripting-modules.ts:396-409
if (type === "post") {
  testPromiseKeepAlive = new Promise<void>((resolve, reject) => {
    resolveKeepAlive = resolve
    rejectKeepAlive = reject
  })
  ctx.keepAlivePromises.push(testPromiseKeepAlive)
}
```

#### keepAlivePromises 完整工作机制

**测试脚本异步执行链**（`src/cage-modules/scripting-modules.ts:524-563`）：
```typescript
ctx.afterScriptExecutionHooks.push(async () => {
  try {
    // 1. 等待引导代码返回的测试执行链 Promise
    if (testExecutionChainPromise) {
      const resolvedPromise = ctx.vm.resolvePromise(testExecutionChainPromise)
      const awaitResult = await resolvedPromise
      // 处理执行错误
    }
    
    // 2. 等待所有旧风格测试 Promise（向后兼容）
    if (testPromises.length > 0) {
      await Promise.allSettled(testPromises)
    }
    
    resolveKeepAlive?.()
  } catch (error) {
    rejectKeepAlive?.(error)
  }
})
```

**Promise 追踪机制**：
```typescript
// 代码位置：src/cage-modules/scripting-modules.ts:411-419
const originalOnTestPromise = (config as PostRequestModuleConfig).onTestPromise
if (originalOnTestPromise) {
  ;(config as PostRequestModuleConfig).onTestPromise = (promise) => {
    testPromises.push(promise)  // 追踪所有测试 Promise
    originalOnTestPromise(promise)
  }
}
```

**关键时序保证**：
| 阶段 | 动作 | 目的 |
|-----|------|------|
| 模块初始化 | 创建 `testPromiseKeepAlive` 并加入 `ctx.keepAlivePromises` | 告知 faraday-cage 需要等待异步操作完成 |
| 脚本执行 | `onTestPromise` 追踪测试创建的 Promise | 收集所有需要等待的异步操作 |
| 脚本同步执行完成 | `afterScriptExecutionHooks` 触发 | 开始等待异步操作 |
| 所有 Promise 完成 | 调用 `resolveKeepAlive()` | 解除沙箱保持状态 |
| 结果捕获 | `captureHook.capture()` 被调用 | 确保包含异步回调中的状态修改 |

**设计意图**：
- 确保 `hopp.fetch().then()` 等异步回调中的环境变量修改被正确捕获
- 测试脚本中的 `pm.test()` 异步断言能完整执行
- 避免 QuickJS 上下文在异步回调完成前被销毁

**前置脚本 vs 测试脚本差异**：
- **前置脚本**：不使用 `keepAlivePromises`，同步执行完成即返回
- **测试脚本**：使用 `keepAlivePromises` 等待所有异步操作

---

### 5.4 状态持久化责任

**沙箱不负责持久化**，仅通过回调返回修改结果：

```typescript
// 宿主侧需要实现：
handleSandboxResults: ({ envs, request, cookies }) => {
  // 1. 更新环境变量存储
  // 2. 更新请求对象（前置脚本）
  // 3. 持久化 Cookie（桌面端）
  // 4. 传递给下一个请求
}
```

**代码位置**：
- 前置脚本回调：`src/cage-modules/scripting-modules.ts:73-77`
- 测试脚本回调：`src/cage-modules/scripting-modules.ts:85-89`

---

## 六、安全机制总结

| 层级 | 机制 | 代码位置 |
|------|------|----------|
| **VM 隔离** | QuickJS 虚拟机，独立上下文 | faraday-cage 库 |
| **对象封装** | `defineSandboxFn` / `defineSandboxObject` 包装所有宿主对象 | `src/cage-modules/scripting-modules.ts:499` |
| **白名单注入** | 仅显式注入的 API 可访问 | `src/cage-modules/default.ts:19-74` |
| **网络拦截** | `hoppFetchHook` 可审计/拦截所有请求 | `src/cage-modules/fetch.ts:29` |
| **值序列化** | 跨边界值经过序列化，防止引用逃逸 | `src/cage-modules/utils/vm-marshal.ts` |
| **错误锁定** | 错误报告函数不可删除/覆盖 | `src/bootstrap-code/pre-request.js:12-25` |
| **深拷贝** | 输入数据深拷贝，防止引用共享 | `src/web/pre-request/index.ts:70-72` |
| **密钥隔离** | CryptoKey 存储在宿主注册表，不穿越边界 | `src/cage-modules/crypto.ts:88-90` |
| **重试机制** | 基础设施错误自动重置沙箱并重试 | `src/web/pre-request/index.ts:89-91` |

---

## 七、执行流程示例（前置脚本）

```
1. acquireCage() → 获取或创建 FaradayCage 单例
2. cloneDeep(envs, request, cookies) → 深拷贝输入
3. cage.runCode(script, [
     defaultModules({ handleConsoleEntry, hoppFetchHook }),
     preRequestModule(config, captureHook)
   ])
   ├─ 执行引导代码（pre-request.js）→ 注入 hopp/pw/pm 命名空间
   ├─ 执行用户脚本 → 可调用注入 API 修改状态
   └─ 等待 keepAlivePromises（异步操作）
4. captureHook.capture() → 捕获修改后的 envs/request/cookies
5. 返回 SandboxPreRequestResult → 宿主更新状态
6. 下次请求使用更新后的 envs/cookies
```

**代码位置**：`src/web/pre-request/index.ts:132-179`

---

## 八、文件索引

| 功能 | 文件路径 |
|------|----------|
| 命名空间定义 | `src/cage-modules/namespaces/*.ts` |
| 沙箱模块配置 | `src/cage-modules/scripting-modules.ts` |
| 默认注入模块 | `src/cage-modules/default.ts` |
| Fetch 封装 | `src/cage-modules/fetch.ts` |
| Crypto 封装 | `src/cage-modules/crypto.ts` |
| 环境变量逻辑 | `src/utils/shared.ts` |
| 前置脚本引导 | `src/bootstrap-code/pre-request.js` |
| 测试脚本引导 | `src/bootstrap-code/post-request.js` |
| Web 执行入口 | `src/web/pre-request/index.ts` |
| Web 测试入口 | `src/web/test-runner/index.ts` |
| Node 执行入口 | `src/node/pre-request/index.ts` |
| Node 测试入口 | `src/node/test-runner/index.ts` |
| Cage 管理 | `src/utils/cage.ts` |
| 类型定义 | `src/types/index.ts` |
| 沙箱标记 | `src/constants/sandbox-markers.ts` |
