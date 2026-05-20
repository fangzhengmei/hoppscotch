# 桌面端工作区按需挂载插件技术分析

## 概述

Hoppscotch 桌面端采用**三层架构**实现工作区按需挂载插件：**插件清单解析层** → **生命周期注入层** → **运行时能力暴露层**。每一层职责明确，通过清晰的接口契约协作，实现了"按需加载、能力隔离、运行时可扩展"的插件系统。

---

## 一、四类实例类型与对应关系

### 1.1 实例类型定义

位置：`packages/hoppscotch-common/src/platform/instance.ts:6`

```typescript
export type InstanceKind = "on-prem" | "cloud" | "cloud-org" | "vendored"
```

### 1.2 四类实例准确对应关系

| 实例类型 | 术语含义 | 加载策略 | Bundle 来源 | 典型场景 |
|---------|---------|---------|------------|---------|
| `vendored` | 内置捆绑包 | 直接 `load()` | 应用内置，无需下载 | 桌面端默认 Hoppscotch 实例 |
| `cloud` | 官方云实例 | 直接 `load()` | 共享内置 Bundle | 连接 `hoppscotch.io` |
| `cloud-org` | 组织云实例 | `load()` + `host` 参数 | 共享内置 Bundle | 连接 `acme.hoppscotch.io` |
| `on-prem` | 自托管实例 | `download()` → `load()` | 从用户服务器下载 | 连接企业私有部署实例 |

> **重要纠正**：代码中不存在 `self-hosted` 类型，自托管实例的官方类型是 `on-prem`。`useAppInitialization.ts` 中的注释 "self-hosted" 是描述性文字，实际类型判断使用的是 `on-prem`。

---

## 二、完整链路：从实例类型判断到内核能力可用

### 2.1 链路总览

```
实例类型判断 → load 调用参数构建 → HostMapper 注册 → URI Handler 取包 → 内核脚本注入 → 内核能力可用
```

---

### 2.2 第一阶段：实例类型判断与加载策略分发

#### 入口：useAppInitialization.loadVendoredIfMatches()

位置：`packages/hoppscotch-desktop/src/composables/useAppInitialization.ts:110-238`

**核心判断逻辑**：

```typescript
if (instance.kind === "vendored" || instance.kind === "cloud") {
  // 策略1: 内置/官方云实例 - 直接加载
  await loadVendoredInstance()
} else if (instance.kind === "cloud-org") {
  // 策略2: 组织云实例 - 加载 + host 参数
  const loadResp = await load({
    bundleName: instance.bundleName!,
    host: instance.serverUrl,  // 传入组织域名
    window: { title: "Hoppscotch" },
  })
} else {
  // 策略3: 自托管实例 (on-prem) - 先下载再加载
  await download({ serverUrl: instance.serverUrl })
  const loadResp = await load({
    bundleName: instance.bundleName!,
    window: { title: "Hoppscotch" },
  })
}
```

**判断依据**：
- `vendored` / `cloud`：Bundle 已内置，跳过下载
- `cloud-org`：Bundle 已内置，但需通过 `host` 参数传递组织上下文
- `on-prem`：Bundle 不在本地，必须先从用户服务器下载

---

### 2.3 第二阶段：load() 调用与 HostMapper 注册

#### 前端 JS API：@hoppscotch/plugin-appload

位置：`packages/hoppscotch-desktop/plugin-workspace/tauri-plugin-appload/guest-js/index.ts:75-77`

```typescript
export async function load(options: LoadOptions): Promise<LoadResponse> {
  return await invoke<LoadResponse>('plugin:appload|load', { options })
}
```

#### Rust 后端处理：commands::load()

位置：`packages/hoppscotch-desktop/plugin-workspace/tauri-plugin-appload/src/commands.rs:108-242`

**关键步骤**：

```rust
pub async fn load<R: Runtime>(app: AppHandle<R>, options: LoadOptions) -> Result<LoadResponse> {
    // 步骤1: 构建 Webview URL
    let sanitized_bundle = sanitize_window_label(&options.bundle_name)?;
    
    let url = match &options.host {
        Some(host) => {
            // cloud-org 实例: app://{bundle}/?org={host}
            format!(
                "app://{}/?org={}",
                sanitized_bundle.to_lowercase(),
                host.to_lowercase()
            )
        }
        None => format!("app://{}/", sanitized_bundle.to_lowercase()),
    };

    // 步骤2: 注册 HostMapper 映射
    let host_mapper = app.state::<Arc<HostMapper>>();
    host_mapper.register(
        &sanitized_bundle.to_lowercase(),
        &options.bundle_name.to_lowercase(),
    );

    // 步骤3: 创建 Webview 并注入内核脚本
    let builder = WebviewWindowBuilder::new(&app, &label, WebviewUrl::App(url.parse().unwrap()))
        .initialization_script(crate::KERNEL_JS)  // 注入 kernel.js
        .title(sanitized_title)
        .inner_size(options.window.width, options.window.height);

    let window = builder.build()?;
    
    Ok(LoadResponse {
        success: window.is_visible().unwrap_or(false),
        window_label: label,
    })
}
```

---

### 2.4 第三阶段：HostMapper 工作原理

位置：`packages/hoppscotch-desktop/plugin-workspace/tauri-plugin-appload/src/mapping.rs:26-118`

**核心作用**：建立虚拟主机名到实际 Bundle 名的映射，实现多组织共享同一 Bundle。

**数据结构**：
```rust
pub struct HostMapper {
    mappings: Arc<DashMap<String, String>>,  // host -> bundle_name
}
```

**关键方法**：

| 方法 | 作用 | 示例 |
|------|------|------|
| `register(host, bundle)` | 注册映射 | `register("acme_hoppscotch_io", "hoppscotch")` |
| `resolve(host)` | 解析映射 | `resolve("acme_hoppscotch_io")` → `"hoppscotch"` |
| `unregister(host)` | 注销映射 | 清理无效映射 |

**设计巧妙之处**：
- 如果没有找到映射，`resolve()` 直接返回 host 本身（passthrough）
- 这意味着非 cloud-org 场景无需特殊处理，直接使用 bundle_name 作为 host 即可
- 同一 Bundle 可以被多个虚拟主机映射，实现代码复用

---

### 2.5 第四阶段：URI Handler 解析路径与取包文件

位置：`packages/hoppscotch-desktop/plugin-workspace/tauri-plugin-appload/src/uri/handler.rs:12-133`

**URI 协议**：`app://{host}/{path}`

**处理流程**：

```rust
pub async fn handle(&self, uri: &Uri) -> Result<Response<Vec<u8>>> {
    let host = uri.host().unwrap_or_default();
    let path = uri.path().trim_start_matches('/');

    // 步骤1: 通过 HostMapper 解析实际 Bundle 名
    let fetch_result = match self.fetch_content(host, path).await {
        Ok(content) => Ok((content, path)),
        Err(e) if !path.is_empty() && !path.contains('.') => {
            // SPA 路由回退: 无扩展名路径视为前端路由，返回 index.html
            self.fetch_content(host, "").await.map(|content| (content, ""))
        }
        Err(e) => Err(e),
    };

    match fetch_result {
        Ok((content, resolved_path)) => {
            let mime_type = Self::determine_mime(resolved_path);
            Response::builder()
                .status(200)
                .header("content-type", mime_type)
                .header("content-security-policy", csp)
                .body(content)
        }
        Err(e) => Response::builder().status(404).body(Vec::new()),
    }
}

async fn fetch_content(&self, host: &str, path: &str) -> Result<Vec<u8>> {
    let file_path = if path.is_empty() { "index.html" } else { path };
    // 关键: 通过 HostMapper 解析获取实际的 bundle_name
    let bundle_name = self.mapper.resolve(host);
    // 从缓存中读取文件
    Ok(self.cache.get_file(&bundle_name, file_path).await?)
}
```

**完整示例**：

| 场景 | 请求 URL | HostMapper 解析 | 实际文件路径 |
|------|---------|----------------|-------------|
| vendored | `app://hoppscotch/` | `resolve("hoppscotch")` → `"hoppscotch"` | `bundles/hoppscotch/index.html` |
| cloud-org | `app://hoppscotch/?org=acme.hoppscotch.io` | `resolve("hoppscotch")` → `"hoppscotch"` | `bundles/hoppscotch/index.html` |
| on-prem | `app://mycompany_com/` | `resolve("mycompany_com")` → `"mycompany_com"` | `bundles/mycompany_com/index.html` |

---

### 2.6 第五阶段：内核脚本注入

#### 注入时机

在 `commands::load()` 创建 Webview 时注入：

```rust
WebviewWindowBuilder::new(...)
    .initialization_script(crate::KERNEL_JS)  // 在页面加载前执行
```

#### kernel.js 内容

位置：`packages/hoppscotch-desktop/plugin-workspace/tauri-plugin-appload/src/kernel.js:1-39`

```javascript
;(() => {
  console.log("Setting desktop kernel mode")
  window.__KERNEL_MODE__ = "desktop"  // 设置内核模式标记
  
  // 记录 Webview 身份到磁盘日志（便于调试）
  Promise.resolve().then(function () {
    var params = new URLSearchParams(window.location.search)
    var orgParam = params.get("org")
    
    // 通过 Tauri IPC 写入日志
    if (window.__TAURI_INTERNALS__) {
      window.__TAURI_INTERNALS__.invoke("append_log", {
        filename: "appload.diag.log",
        content: logLine,
      })
    }
  })
})()
```

**关键作用**：
- 在任何业务代码执行前设置 `window.__KERNEL_MODE__ = "desktop"`
- 这是后续 `initKernel()` 判断运行环境的唯一依据

---

### 2.7 第六阶段：内核能力可用化

#### 前端入口：initKernel()

位置：`packages/hoppscotch-kernel/src/index.ts:46-78`

```typescript
export function initKernel(mode?: KernelMode): KernelAPI {
  const actualMode = mode || window.__KERNEL_MODE__ || "web"
  
  if (actualMode === "desktop") {
    const kernel: KernelAPI = {
      info: { name: "desktop-kernel", version: { major: 1, minor: 0, patch: 0 }, capabilities: ["basic-io"] },
      io: DESKTOP_IO_IMPLS.v1.api,      // 桌面端 IO 实现
      relay: DESKTOP_RELAY_IMPLS.v1.api, // 桌面端网络实现
      store: DESKTOP_STORE_IMPLS.v1.api, // 桌面端存储实现
      log: DESKTOP_LOG_IMPLS.v1.api,     // 桌面端日志实现
    }
    window.__KERNEL__ = kernel
    return kernel
  } else {
    // web 模式实现...
  }
}
```

#### 调用时机

**桌面端主窗口** (`packages/hoppscotch-desktop/src/main.ts:18`)：
```typescript
import { initKernel } from "@hoppscotch/kernel"
initKernel("desktop")  // 显式指定 desktop 模式
```

**工作区 Webview** (`packages/hoppscotch-common/src/index.ts:28`)：
```typescript
export async function createHoppApp(el: string | Element, platformDef: PlatformDef) {
  initKernel(getKernelMode())  // 读取 window.__KERNEL_MODE__
  // ...
}
```

#### 能力访问方式

应用层通过 `getModule()` 安全访问：

```typescript
// packages/hoppscotch-desktop/src/kernel/index.ts
export const getModule = <K extends keyof KernelAPI>(name: K) => {
  const kernel = window.__KERNEL__
  if (!kernel?.[name]) throw new Error(`Kernel ${String(name)} not initialized`)
  return kernel[name]
}
```

---

## 三、三层架构详细说明

### 3.1 第一层：插件清单解析层

**核心组件**：`tauri-plugin-appload`（Rust 后端）

**职责**：
- **Bundle 管理**：下载、验证、缓存、存储插件包
- **URI 协议**：处理 `app://` 自定义协议请求
- **IPC 命令**：提供 `download`/`load`/`close`/`remove`/`clear` 命令

**关键数据结构**：

**BundleMetadata** (`src/models.rs:10-18`)
```rust
pub struct BundleMetadata {
    pub version: String,
    pub created_at: DateTime<Utc>,
    pub signature: Signature,        // 数字签名，完整性校验
    pub manifest: Manifest,          // 插件清单
    pub properties: HashMap<String, String>,
}
```

**BundleLoader 加载流程** (`src/bundle/loader.rs:27-47`)：
```
1. 创建 API 客户端 → 连接插件服务器
2. 初始化密钥管理器 → 获取公钥用于签名验证
3. 拉取元数据 → 获取 BundleMetadata
4. 检查本地缓存 → 版本匹配则直接使用缓存
5. 下载插件包 → 从服务器获取完整 Bundle
6. 验证完整性 → 使用签名和哈希验证
7. 存储到本地 → 写入缓存和持久化存储
```

---

### 3.2 第二层：生命周期注入层

**核心组件**：
- `kernel.js`：Webview 初始化脚本
- `useAppInitialization()`：前端加载编排器
- `DesktopInstanceService`：实例管理服务

**DesktopInstanceService.connectToInstance 流程**：

位置：`packages/hoppscotch-selfhost-web/src/platform/instance/desktop/index.ts:1215-1237`

```typescript
private connectToInstanceTE(
  serverUrl: string,
  instanceKind: InstanceKind,
  displayName?: string,
  options?: Partial<LoadOptions>
): TE.TaskEither<string, OperationResult> {
  return pipe(
    this.normalizeUrlE(serverUrl),
    TE.fromEither,
    TE.chain((normalizedUrl) =>
      pipe(
        // 1. 创建或获取实例（可能触发 download）
        this.createOrGetInstanceTE(normalizedUrl, instanceKind, displayName),
        // 2. 加载实例（触发 load）
        TE.chain((instance) => this.loadInstanceTE(instance, options)),
        TE.map((): OperationResult => ({
          success: true,
          message: `Successfully connected to ${displayName || serverUrl}`,
        }))
      )
    )
  )
}
```

**loadInstanceTE 核心逻辑** (`index.ts:1011-1063`)：
```typescript
private loadInstanceTE(
  instance: Instance,
  options?: Partial<LoadOptions>
): TE.TaskEither<string, LoadResponse> {
  return pipe(
    // 非 vendored 实例确保 Bundle 可用
    instance.kind === "vendored"
      ? TE.of(undefined)
      : TE.tryCatch(async () => {
          await download({ serverUrl: instance.serverUrl })
          return undefined
        }, ...),
    TE.chain(() => this.getBundleNameTE(instance)),
    TE.map(() => this.buildLoadOptions(instance, options)),
    TE.chain((loadOptions) => this.performLoadTE(loadOptions)),  // 调用 load()
    TE.chain((response) => this.validateLoadResponseTE(response, instance.displayName)),
    TE.chainFirst(() => this.updateInstanceStateTE(instance)),
    TE.chainFirst(() => this.updateInstanceLastUsed(instance)),
    // 加载完成后关闭当前窗口
    TE.chainFirst((_loadResponse) => TE.fromTask(async () => {
      try { await this.performCloseTE()() } catch { /* 忽略关闭错误 */ }
      return undefined
    }))
  )
}
```

---

### 3.3 第三层：运行时能力暴露层

**核心组件**：`@hoppscotch/kernel` 包

**Kernel API 契约** (`src/index.ts:25-31`)：
```typescript
export interface KernelAPI {
  info: KernelInfo          // 内核元数据
  io: IoV1                  // I/O 能力（文件、链接、事件）
  relay: RelayV1            // 网络请求能力
  store: StoreV1            // 存储能力
  log: LogV1                // 日志能力
}
```

#### Desktop 端能力实现详解

**IO 能力** (`io/impl/desktop/v/1.ts:17-60`)
- `saveFileWithDialog()`：调用 `tauri-plugin-dialog` 保存文件
- `openExternalLink()`：调用 `tauri-plugin-shell` 打开外部链接
- `listen()`/`once()`/`emit()`：调用 Tauri 事件系统

**Relay 能力** (`relay/impl/desktop/v/1.ts:21-202`)
- 使用原生网络栈（`tauri-plugin-relay`），绕过浏览器 CORS
- 支持完整的 HTTP 方法、认证方式、代理配置
- 能力声明系统确保不支持的请求在执行前被拒绝

**Store 能力** (`store/impl/desktop/v/1.ts:216-388`)
- 使用 `tauri-plugin-store` 实现持久化存储
- 三级隔离：`storePath` → `namespace` → `key`
- 内置元数据追踪（createdAt、updatedAt、加密状态）
- 支持变更监听（watch 模式）

---

## 四、完整协作关系图

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                        运行时能力暴露层 (Kernel API)                            │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐                              │
│  │   IO    │  │  Relay  │  │  Store  │  │   Log   │  ← window.__KERNEL__         │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘                              │
│         ↑            ↑            ↑            ↑                                 │
│         └────────────┴────────────┴────────────┘                                 │
│                                     │                                             │
│                            initKernel("desktop")                                 │
│                                     │                                             │
└─────────────────────────────────────┬─────────────────────────────────────────────┘
                                      │ 注入
┌─────────────────────────────────────▼─────────────────────────────────────────────┐
│                        生命周期注入层 (Lifecycle Injection)                       │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │  kernel.js (Webview initialization_script)                                   │  │
│  │  - window.__KERNEL_MODE__ = "desktop"  (在所有 JS 前执行)                     │  │
│  │  - 记录 Webview 身份到日志                                                   │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│                                      ↑                                            │
│                                      │ 创建 Webview 时注入                        │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │  useAppInitialization.loadVendoredIfMatches()                                │  │
│  │  - 读取 connectionState.instance.kind                                         │  │
│  │  - 分发到对应策略: vendored/cloud/cloud-org/on-prem                          │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                            │
│                                      │ 调用 load() 或 download() + load()         │
└──────────────────────────────────────┬────────────────────────────────────────────┘
                                       │ IPC 调用
┌──────────────────────────────────────▼────────────────────────────────────────────┐
│                        插件清单解析层 (tauri-plugin-appload)                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │  commands::load()                                                            │  │
│  │  1. 构建 URL: app://{bundle}/ 或 app://{bundle}/?org={host}                  │  │
│  │  2. HostMapper.register(sanitized_bundle, bundle_name)                       │  │
│  │  3. WebviewWindowBuilder::new().initialization_script(KERNEL_JS)             │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                            │
│                                      │ 创建 Webview                                │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │  UriHandler::handle()                                                        │  │
│  │  1. 解析 host = uri.host()                                                   │  │
│  │  2. bundle_name = HostMapper.resolve(host)                                   │  │
│  │  3. 从 CacheManager.get_file(bundle_name, path) 读取文件                     │  │
│  │  4. 构建 HTTP 响应返回                                                       │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 五、关键设计决策

### 5.1 多实例共享 Bundle 机制

**问题**：cloud-org 实例使用相同的应用代码，只是组织上下文不同。

**解决方案**：
- 所有 Webview 使用相同的 Bundle：`app://hoppscotch/`
- 组织上下文通过 query 参数传递：`?org=acme.hoppscotch.io`
- JS 侧通过 `window.location.search` 读取 org 参数
- Kernel Store 使用 org 参数实现数据隔离

**优势**：
- 避免重复下载相同的 Bundle
- 保持 Tauri IPC 安全模型（origin 验证）
- 实现多组织数据隔离

### 5.2 HostMapper 的透传设计

**设计**：`resolve()` 方法在找不到映射时直接返回 host 本身。

**优势**：
- 非 cloud-org 场景无需特殊处理
- 向后兼容：旧代码无需修改即可正常工作
- 代码简洁：无需分支判断

### 5.3 SPA 路由回退机制

**设计**：UriHandler 对无扩展名的路径自动回退到 `index.html`。

```rust
Err(e) if !path.is_empty() && !path.contains('.') => {
    self.fetch_content(host, "").await.map(|content| (content, ""))
}
```

**优势**：支持 Vue Router 等前端路由框架的 history 模式。

### 5.4 版本化能力契约

**设计**：每个 Kernel 模块都有版本化的 API 实现（`v1`, `v2` 等）。

**优势**：
- 向后兼容：旧版本插件可继续运行
- 渐进升级：能力实现可独立迭代
- 类型安全：TypeScript 接口确保契约一致性

---

## 六、代码溯源索引

| 功能模块 | 文件位置 | 关键行号 |
|---------|---------|---------|
| 实例类型定义 | `hoppscotch-common/src/platform/instance.ts` | 6 |
| VENDORED_INSTANCE_CONFIG | `hoppscotch-common/src/platform/instance.ts` | 17-24 |
| 加载策略分发 | `hoppscotch-desktop/src/composables/useAppInitialization.ts` | 110-238 |
| 插件 JS API (load) | `tauri-plugin-appload/guest-js/index.ts` | 75-77 |
| Rust load 命令 | `tauri-plugin-appload/src/commands.rs` | 108-242 |
| HostMapper 实现 | `tauri-plugin-appload/src/mapping.rs` | 26-118 |
| URI Handler 实现 | `tauri-plugin-appload/src/uri/handler.rs` | 12-133 |
| kernel.js 脚本 | `tauri-plugin-appload/src/kernel.js` | 1-39 |
| Kernel 初始化 | `hoppscotch-kernel/src/index.ts` | 46-78 |
| DesktopInstanceService | `hoppscotch-selfhost-web/src/platform/instance/desktop/index.ts` | 52-1250 |
| connectToInstanceTE | `hoppscotch-selfhost-web/src/platform/instance/desktop/index.ts` | 1215-1237 |
| loadInstanceTE | `hoppscotch-selfhost-web/src/platform/instance/desktop/index.ts` | 1011-1063 |
| Desktop IO 实现 | `hoppscotch-kernel/src/io/impl/desktop/v/1.ts` | 17-60 |
| Desktop Relay 实现 | `hoppscotch-kernel/src/relay/impl/desktop/v/1.ts` | 21-202 |
| Desktop Store 实现 | `hoppscotch-kernel/src/store/impl/desktop/v/1.ts` | 216-388 |
| 插件初始化入口 | `tauri-plugin-appload/src/lib.rs` | 61-177 |
| Bundle 加载核心 | `tauri-plugin-appload/src/bundle/loader.rs` | 27-47 |
