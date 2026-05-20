# 桌面端工作区按需挂载插件技术分析

## 概述

Hoppscotch 桌面端采用**三层架构**实现工作区按需挂载插件：**插件清单解析层** → **生命周期注入层** → **运行时能力暴露层**。每一层职责明确，通过清晰的接口契约协作，实现了"按需加载、能力隔离、运行时可扩展"的插件系统。

---

## 一、插件清单解析层（Plugin Manifest Resolution）

### 1.1 核心组件：tauri-plugin-appload

位置：`packages/hoppscotch-desktop/plugin-workspace/tauri-plugin-appload/`

这是插件系统的 Rust 后端核心，负责插件包（Bundle）的全生命周期管理。

#### 关键数据结构

**BundleMetadata** (`src/models.rs:10-18`)
```rust
pub struct BundleMetadata {
    pub version: String,
    pub created_at: DateTime<Utc>,
    pub signature: Signature,        // 数字签名，用于完整性校验
    pub manifest: Manifest,          // 插件清单
    pub properties: HashMap<String, String>,
}
```

**Manifest** (`src/models.rs:81-85`)
```rust
pub struct Manifest {
    pub files: Vec<FileEntry>,       // 插件包含的所有文件
    pub version: Option<String>,
}
```

**FileEntry** (`src/models.rs:46-52`)
```rust
pub struct FileEntry {
    pub path: String,
    pub size: u64,
    pub hash: Hash,                  // blake3 哈希，用于文件校验
    pub mime_type: Option<String>,
}
```

#### 插件加载流程

`BundleLoader::load_bundle()` (`src/bundle/loader.rs:27-47`) 是核心入口：

```
1. 创建 API 客户端 → 连接插件服务器
2. 初始化密钥管理器 → 获取公钥用于签名验证
3. 拉取元数据 → 获取 BundleMetadata
4. 检查本地缓存 → 版本匹配则直接使用缓存
5. 下载插件包 → 从服务器获取完整 Bundle
6. 验证完整性 → 使用签名和哈希验证
7. 存储到本地 → 写入缓存和持久化存储
```

#### 缓存策略

- **内存缓存**：`CacheManager` 管理热数据
- **磁盘缓存**：按版本号存储，支持 TTL 过期
- **注册表**：`registry.json` 记录已安装的插件清单

---

## 二、生命周期注入层（Lifecycle Injection）

### 2.1 插件系统初始化时机

#### Rust 层初始化（`src-tauri/src/lib.rs:136-273`）

在 Tauri 应用构建时注册插件：

```rust
.plugin(tauri_plugin_appload::init(appload_config))  // 插件加载器
.plugin(tauri_plugin_relay::init())                  // 网络中继插件
```

`tauri-plugin-appload::init()` (`src/lib.rs:61-177`) 执行以下操作：

1. 初始化存储管理器（StorageManager）
2. 初始化缓存管理器（CacheManager）
3. 初始化 BundleLoader
4. 注册 URI 协议处理器（`app://` 方案）
5. 注册 IPC 命令处理器（download/load/close/remove/clear）
6. 初始化桌面端特定组件

#### Webview 初始化脚本注入

当调用 `load()` 命令创建 Webview 时（`src/commands.rs:108-242`）：

```rust
WebviewWindowBuilder::new(&app, &label, WebviewUrl::App(url.parse().unwrap()))
    .initialization_script(crate::KERNEL_JS)  // 注入内核脚本
    // ... 其他配置
```

**kernel.js** (`src/kernel.js:1-39`) 在 Webview 加载前执行：

```javascript
window.__KERNEL_MODE__ = "desktop"  // 设置运行模式
// 记录 Webview 身份到日志文件
```

### 2.2 前端启动流程

**主入口** (`packages/hoppscotch-desktop/src/main.ts:1-18`)
```typescript
import { initKernel } from "@hoppscotch/kernel"
initKernel("desktop")  // 初始化桌面内核
```

**应用初始化** (`src/composables/useAppInitialization.ts:37-409`)

`useAppInitialization()` 是工作区挂载的核心编排器：

```
1. 版本备份检查 → 防止升级数据丢失
2. 数据迁移 → 处理跨版本数据格式变化
3. 持久化存储初始化 → 连接本地 Store
4. 读取连接状态 → 判断上次连接的工作区
5. 根据实例类型选择加载策略：
   ├─ vendored → 直接加载内置包
   ├─ cloud-org → 使用内置包 + host 参数
   └─ self-hosted → download() + load() 两步走
6. 关闭主窗口 → 切换到工作区 Webview
```

#### 实例类型与加载策略

| 实例类型 | 加载方式 | 说明 |
|---------|---------|------|
| `vendored` | 直接 `load()` | 内置 Hoppscotch 包，无需下载 |
| `cloud` | 直接 `load()` | 官方云实例，使用内置包 |
| `cloud-org` | `load()` + `host` 参数 | 组织实例，共享内置包，通过 `?org=` 区分上下文 |
| `self-hosted` | `download()` → `load()` | 自建实例，从服务器下载包 |

---

## 三、运行时能力暴露层（Runtime Capability Exposure）

### 3.1 Kernel API 架构

位置：`packages/hoppscotch-kernel/src/index.ts`

`initKernel(mode)` 根据运行模式（web/desktop）注入不同的能力实现：

```typescript
export interface KernelAPI {
  info: KernelInfo          // 内核元数据
  io: IoV1                  // I/O 能力
  relay: RelayV1            // 网络请求能力
  store: StoreV1            // 存储能力
  log: LogV1                // 日志能力
}
```

### 3.2 Desktop 端能力实现

#### IO 能力 (`io/impl/desktop/v/1.ts`)

```typescript
api: {
  saveFileWithDialog()    // 保存文件对话框（调用 tauri-plugin-dialog）
  openExternalLink()      // 打开外部链接（调用 tauri-plugin-shell）
  listen()                // 监听自定义事件
  once()                  // 单次事件监听
  emit()                  // 发送自定义事件
}
```

#### Relay 能力 (`relay/impl/desktop/v/1.ts`)

桌面端使用原生网络栈，绕过浏览器 CORS 限制：

```typescript
capabilities: {
  method: Set(["GET", "POST", "PUT", "DELETE", "PATCH", "HEAD", "OPTIONS"])
  content: Set(["text", "json", "xml", "form", "binary", "multipart", "stream"])
  auth: Set(["basic", "bearer", "digest", "oauth2", "apikey"])
  security: Set(["clientcertificates", "cacertificates"])
  proxy: Set(["http", "https", "authentication"])
  advanced: Set(["retry", "redirects", "timeout", "cookies", "keepalive"])
}
```

实现方式：调用 `@hoppscotch/plugin-relay` 的 `execute()` 方法，该方法通过 IPC 调用 Rust 侧的 `tauri-plugin-relay` 插件执行原生网络请求。

#### Store 能力 (`store/impl/desktop/v/1.ts`)

使用 `tauri-plugin-store` 实现持久化存储：

```typescript
capabilities: Set(["permanent", "structured", "watch", "namespace", "secure"])
```

核心特性：
- **命名空间隔离**：按 `storePath:namespace:key` 三级隔离
- **元数据追踪**：记录 createdAt、updatedAt、加密压缩状态
- **变更监听**：支持 watch 模式监听数据变化
- **单例管理**：`TauriStoreManager` 确保同一文件只有一个实例

### 3.3 能力访问方式

应用层通过 `getModule()` 安全访问内核能力：

```typescript
// packages/hoppscotch-desktop/src/kernel/index.ts
export const getModule = <K extends keyof KernelAPI>(name: K) => {
  const kernel = window.__KERNEL__
  if (!kernel?.[name]) throw new Error(`Kernel ${String(name)} not initialized`)
  return kernel[name]
}
```

---

## 四、三层协作关系图

```
┌─────────────────────────────────────────────────────────────┐
│                  运行时能力暴露层                            │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
│  │   IO    │  │  Relay  │  │  Store  │  │   Log   │        │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘        │
│         ↑            ↑            ↑            ↑            │
│         └────────────┴────────────┴────────────┘            │
│                              │                              │
│                       Kernel API 契约                        │
└──────────────────────────────┬──────────────────────────────┘
                               │ 注入
┌──────────────────────────────▼──────────────────────────────┐
│                  生命周期注入层                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  kernel.js (Webview 初始化脚本)                       │   │
│  │  - 设置 __KERNEL_MODE__ = "desktop"                  │   │
│  │  - 记录 Webview 身份                                 │   │
│  └──────────────────────────────────────────────────────┘   │
│         ↓ 调用 initKernel("desktop")                        │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  useAppInitialization()                              │   │
│  │  - 读取连接状态                                      │   │
│  │  - 分发加载策略（vendored/cloud-org/self-hosted）    │   │
│  └──────────────────────────────────────────────────────┘   │
│         ↓ IPC 调用                                          │
└──────────────────────────────┬──────────────────────────────┘
                               │ 命令调用
┌──────────────────────────────▼──────────────────────────────┐
│                  插件清单解析层                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  BundleLoader                                        │   │
│  │  - download(): 下载 + 验证 + 缓存                    │   │
│  │  - load(): 创建 Webview + 注入 kernel.js             │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  URI Handler (app:// 协议)                           │   │
│  │  - 解析 app://{bundle}/ 路径                         │   │
│  │  - 从缓存/存储读取文件返回                           │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 五、关键设计决策

### 5.1 多实例共享 Bundle 机制

**问题**：云组织实例（cloud-org）使用相同的应用代码，只是组织上下文不同。

**解决方案** (`src/commands.rs:126-145`)：
- 所有 Webview 使用相同的 URL host：`app://{bundle_name}/`
- 组织上下文通过 query 参数传递：`app://hoppscotch/?org=acme.hoppscotch.io`
- JS 侧通过 `window.location.search` 读取 org 参数
- Kernel Store 使用 org 参数实现文件隔离

**优势**：
- 避免重复下载相同的 Bundle
- 保持 Tauri IPC 安全模型（origin 验证）
- 实现多组织数据隔离

### 5.2 双窗口切换机制

**问题**：主窗口负责实例选择，工作区窗口负责实际应用运行。

**解决方案**：
1. 应用启动显示主窗口（loading/error 状态）
2. 加载完成后调用 `close({ windowLabel: "main" })` 关闭主窗口
3. 工作区 Webview 成为唯一可见窗口

### 5.3 版本化能力契约

**设计**：每个 Kernel 模块都有版本化的 API 实现（`v1`, `v2` 等）。

**优势**：
- 向后兼容：旧版本插件可继续运行
- 渐进升级：能力实现可独立迭代
- 类型安全：TypeScript 接口确保契约一致性

---

## 六、代码溯源索引

| 功能模块 | 文件位置 | 关键行号 |
|---------|---------|---------|
| 插件初始化入口 | `tauri-plugin-appload/src/lib.rs` | 61-177 |
| Bundle 加载核心 | `tauri-plugin-appload/src/bundle/loader.rs` | 27-47 |
| 工作区加载编排 | `hoppscotch-desktop/src/composables/useAppInitialization.ts` | 37-409 |
| Webview 创建与注入 | `tauri-plugin-appload/src/commands.rs` | 108-242 |
| Kernel 初始化 | `hoppscotch-kernel/src/index.ts` | 46-78 |
| Desktop IO 实现 | `hoppscotch-kernel/src/io/impl/desktop/v/1.ts` | 17-60 |
| Desktop Relay 实现 | `hoppscotch-kernel/src/relay/impl/desktop/v/1.ts` | 21-202 |
| Desktop Store 实现 | `hoppscotch-kernel/src/store/impl/desktop/v/1.ts` | 216-388 |
| 插件 JS API | `tauri-plugin-appload/guest-js/index.ts` | 71-89 |
