# Hoppscotch Desktop Agent 跨源代理协作机制

## 整体架构概览

Hoppscotch 的跨源请求方案由两层协作实现：

1. **本地 Agent**（`packages/hoppscotch-agent`）：基于 Tauri 的桌面应用，在本地 `127.0.0.1:9119` 启动 HTTP 服务，接收浏览器端的加密请求并转发至目标服务器。
2. **浏览器前端**（`packages/hoppscotch-common`）：运行在浏览器中的 Web 应用，通过"拦截器（Interceptor）"机制选择请求通道。Agent 拦截器将请求加密后发送到本地 Agent 服务。

两者之间的通信采用端到端加密（AES-256-GCM），密钥通过 X25519 Diffie-Hellman 密钥交换在注册阶段协商。

```
┌─────────────────────────────────────┐
│           浏览器 (Web App)           │
│                                     │
│  用户发起请求 → KernelInterceptor   │
│       │                             │
│       ├─ Agent 拦截器 (选中时)       │
│       │    │                        │
│       │    ├─ 加密请求 (AES-256-GCM)│
│       │    └─ POST → localhost:9119 │
│       │                             │
│       ├─ Proxy 拦截器               │
│       ├─ Extension 拦截器           │
│       └─ Browser 拦截器             │
└──────────────┬──────────────────────┘
               │ HTTP (加密载荷)
               ▼
┌─────────────────────────────────────┐
│     Hoppscotch Agent (Tauri 桌面)   │
│                                     │
│  Axum HTTP Server :9119             │
│       │                             │
│       ├─ 解密请求                    │
│       ├─ relay::execute() 转发      │
│       ├─ 加密响应                    │
│       └─ 返回加密响应给浏览器        │
└─────────────────────────────────────┘
```

---

## 一、Agent 本地服务启动

### 入口与生命周期

**文件**: `packages/hoppscotch-agent/src-tauri/src/lib.rs`

Agent 是 Tauri 应用，入口函数 `run()` 在 `lib.rs:69`。启动流程如下：

1. **WebView 初始化**（仅 Windows 便携版）：检查并安装 WebView2 运行时。
2. **创建 CancellationToken**：用于优雅关闭服务。
3. **构建 Tauri Builder**：
   - 注册 `tauri_plugin_single_instance`：确保只运行一个实例，重复启动时唤起已有窗口。
   - 注册 `tauri_plugin_store`：持久化存储（注册信息等）。
   - `.setup()` 回调中完成核心初始化：
     - 配置开机自启动（`tauri_plugin_autostart`）
     - 配置自动更新（`tauri_plugin_updater`）
     - 创建主窗口并立即隐藏（Agent 常驻系统托盘，不显示窗口）
     - 初始化 `AppState`（从持久化存储加载已注册的客户端）
     - **异步启动 HTTP 服务**：`tauri::async_runtime::spawn(async move { server::run_server(...) })`
     - 创建系统托盘图标
     - 监听 Tauri 事件（`maximize-window`、`registration-received`）

4. **窗口关闭行为**：关闭请求被拦截，窗口仅隐藏（`api.prevent_close()` + `window.hide()`），Agent 继续后台运行。

### HTTP 服务

**文件**: `packages/hoppscotch-agent/src-tauri/src/server.rs`

```rust
pub async fn run_server(state, cancellation_token, app_handle) {
    let cors = CorsLayer::permissive();
    let app = Router::new()
        .merge(route::route(state, app_handle))
        .layer(cors);
    let addr = std::net::SocketAddr::from(([127, 0, 0, 1], 9119));
    // 绑定监听 + 优雅关闭
    tokio::net::TcpListener::bind(&addr).await...
    axum::serve(listener, app.into_make_service())
        .with_graceful_shutdown(cancellation_token.cancelled().await)
        .await
}
```

关键点：
- 监听地址固定 `127.0.0.1:9119`，仅本机可访问。
- CORS 设置为 `permissive`（完全开放），允许浏览器跨源访问。
- 支持通过 `CancellationToken` 优雅关闭。

### 路由表

**文件**: `packages/hoppscotch-agent/src-tauri/src/route.rs`

| 路由 | 方法 | 用途 |
|------|------|------|
| `/handshake` | GET | 探测 Agent 是否在运行 |
| `/receive-registration` | POST | 发起注册流程 |
| `/verify-registration` | POST | 验证 OTP 并完成注册 |
| `/registered-handshake` | GET | 已注册客户端验证连接有效性 |
| `/registration` | GET | 获取当前注册信息（加密） |
| `/registrations/:auth_key` | DELETE | 删除某个注册 |
| `/execute` | POST | **执行代理请求**（核心） |
| `/cancel/:req_id` | POST | 取消正在执行的请求 |
| `/log-sink` | POST | 接收前端日志（加密） |

### 状态管理

**文件**: `packages/hoppscotch-agent/src-tauri/src/state.rs`

`AppState` 包含：
- `active_registration_code: RwLock<Option<String>>`：正在进行的注册 OTP。
- `cancellation_tokens: DashMap<usize, CancellationToken>`：运行中请求的取消令牌。
- `registrations: DashMap<String, Registration>`：已注册的客户端列表，key 是 auth_key。

`Registration` 结构（`model.rs`）：
```rust
pub struct Registration {
    pub registered_at: DateTime<Utc>,
    pub shared_secret_b16: String,  // X25519 协商出的共享密钥（base16）
}
```

注册信息通过 `tauri_plugin_store` 持久化到 `app_data.bin` 文件。

---

## 二、前端探测与切换机制

### 拦截器体系

Hoppscotch 前端有 5 种拦截器，每种对应一种请求通道：

| 拦截器 | ID | 运行位置 | 请求方式 |
|--------|----|----------|----------|
| Browser | `browser` | 浏览器 | 直接 fetch（受 CORS 限制） |
| Proxy | `proxy` | 浏览器 → 远程代理服务器 | 发到 Hoppscotch 代理 |
| Agent | `agent` | 浏览器 → 本地 Agent | 发到 localhost:9119 |
| Extension | `extension` | 浏览器扩展 | 通过 Chrome/Firefox 扩展 |
| Native | `native` | Desktop WebView 内 | 直接通过 Rust relay 层 |

**拦截器注册由平台配置决定**（`packages/hoppscotch-selfhost-web/src/main.ts`）：

```typescript
const PLATFORM_CONFIG = {
  web: {
    interceptors: [Browser, Proxy, Agent, Extension],
    defaultInterceptor: "browser",
  },
  desktop: {
    interceptors: [Native, Proxy],
    defaultInterceptor: "native",
  },
}
```

- **Web 平台**：注册 Browser、Proxy、Agent、Extension 四种拦截器，默认使用 Browser。
- **Desktop 平台**（Tauri 桌面应用）：仅注册 Native 和 Proxy，默认使用 Native（Native 拦截器直接通过 Rust relay 执行，无需本地 HTTP 服务中转）。

**Agent 拦截器主要面向 Web 平台用户**——在浏览器中使用 Hoppscotch Web 版时，选中 Agent 拦截器可以将请求代理到本地 Agent 执行，绕过 CORS 限制。

### KernelInterceptorService

**文件**: `packages/hoppscotch-common/src/services/kernel-interceptor.service.ts`

`KernelInterceptorService` 管理拦截器的注册与切换：

- `register(interceptor)`：注册拦截器到 Map。
- `setActive(id)`：设置当前激活的拦截器。
- `execute(req)`：使用当前激活的拦截器执行请求。
- `current`：computed，当前选中的拦截器。
- `available`：computed，所有已注册拦截器列表。

拦截器选择通过 Vue 响应式 + 持久化设置同步：
- 当用户切换拦截器时，`CURRENT_KERNEL_INTERCEPTOR_ID` 设置项自动保存。
- 应用启动时，从设置恢复上次选中的拦截器。

### Agent 探测流程

**文件**: `packages/hoppscotch-common/src/platform/std/kernel-interceptors/agent/store.ts`

`KernelInterceptorAgentStore` 管理 Agent 的状态和通信：

#### 1. Agent 运行状态检测

```typescript
public async checkAgentStatus(): Promise<void> {
  try {
    const handshakeResponse = await axios.get("http://localhost:9119/handshake")
    this.isAgentRunning.value =
      handshakeResponse.data.status === "success" &&
      handshakeResponse.data.__hoppscotch__agent__ === true
  } catch {
    this.isAgentRunning.value = false
  }
}
```

前端通过 GET `/handshake` 探测 Agent 是否在运行。Agent 返回：
```json
{ "status": "success", "__hoppscotch__agent__": true, "agent_version": "0.1.17" }
```

#### 2. 注册流程（OTP 双因素验证）

用户在 Web 端选中 Agent 拦截器后，需要先与本地 Agent 完成注册：

```
┌──────────┐                         ┌──────────┐
│  浏览器   │                         │  Agent   │
└─────┬────┘                         └─────┬────┘
      │  1. POST /receive-registration    │
      │ ──────────────────────────────►   │
      │     { message: "Registration received" }
      │ ◄──────────────────────────────   │
      │                                   │  Agent 弹出窗口显示 OTP
      │  2. 用户在浏览器输入 Agent 显示的 OTP │
      │                                   │
      │  3. POST /verify-registration     │
      │ ──────────────────────────────►   │
      │     { registration: "123456",     │
      │       client_public_key_b16: "…" }│
      │                                   │
      │     { auth_key: "uuid",           │
      │       agent_public_key_b16: "…" } │
      │ ◄──────────────────────────────   │
      │                                   │
      │  4. 双方计算 X25519 共享密钥       │
      │     保存 auth_key + shared_secret │
      └───────────────────────────────────┘
```

**步骤详解**：

1. **发起注册** (`initiateRegistration`)：
   - 浏览器生成一个 6 位 OTP。
   - POST `/receive-registration`，Agent 收到后弹出窗口显示 OTP 供用户确认。

2. **验证注册** (`verifyRegistration`)：
   - 用户在浏览器中输入 Agent 窗口显示的 OTP。
   - 浏览器生成 X25519 临时密钥对，将公钥和 OTP 一起发送。
   - Agent 验证 OTP 正确后，生成自己的 X25519 密钥对，计算共享密钥，返回 `auth_key` + Agent 公钥。
   - 浏览器计算相同的共享密钥。
   - 双方保存 `auth_key`（用作 Bearer Token）和 `shared_secret_b16`（用于 AES-256-GCM 加解密）。

3. **持久化**：注册凭证保存在浏览器的 Kernel Store 中（`interceptors.agent.v1`），下次访问无需重新注册。

#### 3. AgentSubtitle 组件交互

**文件**: `packages/hoppscotch-common/src/components/settings/AgentSubtitle.vue`

Agent 拦截器的副标题组件管理注册 UI：

- 未注册且未检测 Agent：显示"注册 Agent"按钮 → 点击触发 `checkAgentStatus()` → 若 Agent 运行则自动进入注册流程。
- Agent 未运行：显示错误提示"Agent not running"。
- 正在注册：显示 OTP 输入框和确认按钮。
- 已注册：显示 masked auth key hash 和"注销"按钮。

---

## 三、请求转发实现

### 完整请求链路

当用户选中 Agent 拦截器并发送请求时，执行流程如下：

```
用户请求
  │
  ▼
KernelInterceptorService.execute(req)
  │
  ▼
AgentKernelInterceptorService.execute(request)      ← agent/index.ts
  │
  ├── 1. checkAgentStatus()          → GET /handshake，确认 Agent 在线
  ├── 2. isAgentRunning && isAuthKeyPresent() → 检查注册凭证
  ├── 3. completeRequest()           → 合并域名级安全/代理/重定向设置
  ├── 4. cookieJar.getCookiesForURL() → 注入 Cookie
  ├── 5. relayRequestToNativeAdapter() → 转为原生请求格式
  ├── 6. postProcessRelayRequest()   → superjson 序列化
  ├── 7. encryptRequest()            → AES-256-GCM 加密
  ├── 8. axios.post("/execute", encryptedReq)  → 发送加密请求
  │      headers: Authorization: Bearer <auth_key>
  │               X-Hopp-Nonce: <nonce_b16>
  │               Content-Type: application/octet-stream
  │
  ▼
Agent 端 controller::execute()                   ← controller.rs
  │
  ├── 1. 验证 Bearer Token (auth_key)
  ├── 2. 从 X-Hopp-Nonce 头取 nonce
  ├── 3. validate_access_and_get_data() → 解密请求体
  ├── 4. relay::execute(request)        → 实际发送 HTTP 请求
  ├── 5. EncryptedJson { key_b16, data } → 加密响应
  │
  ▼
浏览器端接收响应
  │
  ├── 9. 从 X-Hopp-Nonce 头取 nonce
  ├── 10. decryptResponse(nonce, data) → AES-256-GCM 解密
  ├── 11. 处理 Set-Cookie 多值头部
  └── 12. 返回 RelayResponse
```

### 加密通信细节

#### 请求加密（浏览器端）

**文件**: `agent/store.ts` - `encryptRequest()`

```typescript
public async encryptRequest(request, reqID): Promise<[string, ArrayBuffer]> {
  const fullRequest = { ...request, id: reqID }
  const reqJSON = JSON.stringify(fullRequest)
  const reqJSONBytes = new TextEncoder().encode(reqJSON)
  const nonce = window.crypto.getRandomValues(new Uint8Array(12))
  const nonceB16 = base16.encode(nonce).toLowerCase()

  const sharedSecretKey = await window.crypto.subtle.importKey(
    "raw", sharedSecretKeyBytes, "AES-GCM", true, ["encrypt", "decrypt"]
  )
  const encryptedReq = await window.crypto.subtle.encrypt(
    { name: "AES-GCM", iv: nonce }, sharedSecretKey, reqJSONBytes
  )
  return [nonceB16, encryptedReq]
}
```

#### 请求解密（Agent 端）

**文件**: `state.rs` - `validate_access_and_get_data()`

```rust
let key: [u8; 32] = base16::decode(&registration.shared_secret_b16)?;
let nonce: [u8; 12] = base16::decode(nonce_header)?;
let cipher = Aes256Gcm::new(&key.into());
let plain_data = cipher.decrypt(&nonce.into(), data.as_slice())?;
let request: T = serde_json::from_reader(plain_data.as_slice())?;
```

#### 响应加密（Agent 端）

**文件**: `util.rs` - `EncryptedJson::into_response()`

```rust
fn into_response(self) -> Response {
  let serialized = serde_json::to_vec(&self.data)?;
  let key: [u8; 32] = base16::decode(&self.key_b16)?;
  let cipher = Aes256Gcm::new(&key.into());
  let nonce = Aes256Gcm::generate_nonce(&mut OsRng);
  let nonce_b16 = base16::encode_lower(&nonce);
  let encrypted = cipher.encrypt(&nonce, serialized.as_slice())?;
  // Content-Type: application/octet-stream
  // X-Hopp-Nonce: nonce_b16
}
```

#### 响应解密（浏览器端）

**文件**: `agent/store.ts` - `decryptResponse()`

```typescript
public async decryptResponse(nonceB16, encryptedResponse): Promise<PluginResponse> {
  const nonce = new Uint8Array(base16.decode(nonceB16.toUpperCase()))
  const sharedSecretKey = await window.crypto.subtle.importKey(
    "raw", sharedSecretKeyBytes, "AES-GCM", true, ["encrypt", "decrypt"]
  )
  const decryptedBytes = await window.crypto.subtle.decrypt(
    { name: "AES-GCM", iv: nonce }, sharedSecretKey, encryptedResponse
  )
  return JSON.parse(new TextDecoder().decode(decryptedBytes))
}
```

### 请求取消

浏览器端支持取消请求（`cancel`），有两种途径：
1. Axios 的 `CancelToken`：直接取消 HTTP 连接。
2. Agent 端的 `/cancel/:req_id`：通过 Agent 的 `CancellationToken` 取消正在执行的 relay 请求。

### Cookie 处理

Agent 拦截器在发送请求前：
1. 从 `CookieJarService` 获取目标 URL 相关的 Cookie。
2. 将 Cookie 拼接为 `Cookie` 头注入请求。
3. 响应返回后，解析 `Set-Cookie` 头（支持多个值，用 `\n` 分隔），转为 `multiHeaders` 格式。

### 域名级设置

Agent 拦截器支持按域名配置安全/代理/高级选项（类似 Native 拦截器）：

- **安全**：SSL 证书验证（verifyHost、verifyPeer）、CA 证书、客户端证书。
- **代理**：HTTP/HTTPS 代理 URL、代理认证。
- **高级**：跟随重定向。

设置合并策略：域名级设置覆盖全局默认设置（`*` 通配符），通过 `getMergedSettings()` 合并。

---

## 四、Desktop 平台 vs Web 平台的差异

| 特性 | Web 平台 | Desktop 平台 |
|------|----------|-------------|
| 默认拦截器 | Browser | Native |
| 可用拦截器 | Browser, Proxy, Agent, Extension | Native, Proxy |
| Agent 拦截器 | 可选（需本地安装 Agent 桌面应用） | 不提供（Native 已内建同等能力） |
| 请求执行路径 | 浏览器 → Agent(:9119) → 目标 | WebView → Rust relay → 目标 |
| 加密通信 | 需要（跨进程） | 不需要（同进程） |
| 注册流程 | 需要（OTP + X25519 密钥交换） | 不适用 |

**核心差异**：Desktop 平台内建了 Native 拦截器，直接通过 Rust `relay::execute()` 发送请求（绕过所有浏览器限制），功能与 Agent 等价但无需本地 HTTP 服务和加密。Agent 拦截器主要为 Web 平台设计——当用户在浏览器中使用 Hoppscotch 时，通过本地安装的 Agent 桌面应用作为代理，绕过 CORS 和浏览器安全限制。

---

## 五、关键文件索引

### Agent 端（Rust/Tauri）

| 文件 | 职责 |
|------|------|
| `hoppscotch-agent/src-tauri/src/lib.rs` | 应用入口、Tauri Builder 配置、服务启动 |
| `hoppscotch-agent/src-tauri/src/server.rs` | Axum HTTP 服务启动（:9119） |
| `hoppscotch-agent/src-tauri/src/route.rs` | 路由表定义 |
| `hoppscotch-agent/src-tauri/src/controller.rs` | 各路由处理函数（handshake、注册、execute 等） |
| `hoppscotch-agent/src-tauri/src/state.rs` | AppState 状态管理、加解密、认证 |
| `hoppscotch-agent/src-tauri/src/model.rs` | 数据模型（Registration、HandshakeResponse 等） |
| `hoppscotch-agent/src-tauri/src/util.rs` | EncryptedJson 响应加密、工具函数 |
| `hoppscotch-agent/src-tauri/src/global.rs` | 全局常量（NONCE 头名、存储 key） |
| `hoppscotch-agent/src-tauri/src/command.rs` | Tauri 命令（get_otp、list_registrations） |

### 前端（TypeScript/Vue）

| 文件 | 职责 |
|------|------|
| `hoppscotch-common/src/platform/std/kernel-interceptors/agent/index.ts` | Agent 拦截器服务、请求执行主逻辑 |
| `hoppscotch-common/src/platform/std/kernel-interceptors/agent/store.ts` | Agent 状态管理、加解密、注册、状态检测 |
| `hoppscotch-common/src/services/kernel-interceptor.service.ts` | 拦截器注册与切换的核心服务 |
| `hoppscotch-common/src/modules/kernel-interceptors.ts` | 拦截器模块初始化、设置同步 |
| `hoppscotch-common/src/components/settings/Agent.vue` | Agent 设置 UI（域名设置、证书、代理） |
| `hoppscotch-common/src/components/settings/AgentSubtitle.vue` | Agent 注册/状态 UI |
| `hoppscotch-common/src/helpers/functional/process-request.ts` | 请求预处理（URL 参数编码、superjson 序列化） |
| `hoppscotch-selfhost-web/src/main.ts` | 平台配置（Web/Desktop 拦截器列表） |
| `hoppscotch-common/src/platform/kernel-interceptors.ts` | 拦截器平台定义类型 |
