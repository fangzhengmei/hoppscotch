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
| `/registered-handshake` | GET | 已注册客户端验证连接有效性（前端未使用） |
| `/registration` | GET | 获取当前注册信息（加密） |
| `/registrations/:auth_key` | DELETE | 删除某个注册（前端未调用） |
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

### 拦截器切换的持久化与恢复

**文件**: `packages/hoppscotch-common/src/modules/kernel-interceptors.ts`

拦截器选择通过 Vue 响应式 + 持久化设置同步机制实现双向同步：

1. **保存（内存 → 存储）：
```typescript
function syncServiceToSettings(service: KernelInterceptorService): void {
  watch(
    () => service.current.value?.id,
    (id) => {
      applySetting("CURRENT_KERNEL_INTERCEPTOR_ID",
        id ?? platform.kernelInterceptors.default
      )
    }
  )
}
```

2. **恢复（存储 → 内存）：
```typescript
function syncSettingsToService(service: KernelInterceptorService): void {
  const [setting] = useSettingStatic("CURRENT_KERNEL_INTERCEPTOR_ID")
  watch(
    setting,
    () => {
      const fallback = setting.value ?? platform.kernelInterceptors.default
      service.setActive(fallback)
    },
    { immediate: true }  // 启动时立即恢复
  )
}
```

3. **有效性校验**：如果存储的拦截器不可用（例如切换平台后），自动回退：
```typescript
private setupInterceptorValidation(): void {
  watchEffect(() => {
    if (!this.state.currentId) return
    const currentInterceptor = this.state.interceptors.get(this.state.currentId)
    if (!this.validateCurrentInterceptor(currentInterceptor)) {
      this.resetToSelectableInterceptor()  // 回退到第一个可用的
    }
  })
}
```

**完整恢复流程**：
1. 应用启动 → 读取 `CURRENT_KERNEL_INTERCEPTOR_ID` 设置项
2. 调用 `service.setActive(fallback)` 设置拦截器
3. `watchEffect` 立即校验当前拦截器是否可用（`selectable.type === "selectable"`）
4. 若不可用，自动回退到第一个可选拦截器

---

## 三、注册流程（OTP 双因素验证

### ⚠️ 关键纠正：OTP 是 Agent 生成的

**之前的错误理解**：浏览器生成 OTP 发送给 Agent。

**正确理解**：**浏览器生成的 OTP 完全被忽略**！实际 OTP 是 **Agent 端生成的！

**文件**: `packages/hoppscotch-agent/src-tauri/src/controller.rs:57`

```rust
pub async fn receive_registration(
    State((state, app_handle)): State<(Arc<AppState>, AppHandle)>,
) -> AgentResult<Json<serde_json::Value>> {
    let otp = generate_otp();  // ← Agent 自己生成！
    *active_registration_code = Some(otp.clone());  // 存自己生成的
    app_handle.emit("registration-received", otp)  // 发给 Agent 前端显示
}
```

浏览器端 `store.ts:215-228` 虽然也生成了一个 OTP，但这个 OTP 被 Agent 完全忽略，Agent 根本不读取请求体！

---

### 完整注册时序图

```
┌──────────┐                         ┌──────────┐
│  浏览器   │                         │  Agent   │
└─────┬────┘                         └─────┬────┘
      │  1. POST /receive-registration    │
      │  (附带一个随机 OTP，但被忽略)        │
      │ ──────────────────────────────►   │
      │                                   │
      │                                   │  generate_otp() → 生成真实 OTP
      │                                   │  存入 active_registration_code
      │     { message: "Registration received" }
      │ ◄──────────────────────────────   │
      │                                   │
      │                                   │  emit("registration-received", otp)
      │                                   │  Agent 窗口弹出，显示真实 OTP
      │                                   │
      │  2. 用户从 Agent 窗口复制 OTP  │
      │     粘贴到浏览器输入框            │
      │                                   │
      │  3. POST /verify-registration     │
      │ ──────────────────────────────►   │
      │     { registration: "123456",           │
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

### 注册流程详解

#### 1. 发起注册

**文件**: `packages/hoppscotch-common/src/components/settings/AgentSubtitle.vue:90-103`

```typescript
const handleAgentCheck = async () => {
  try {
    await store.checkAgentStatus()  // GET /handshake → 确认 Agent 在线
    store.hasCheckedAgent.value = true
    if (!store.isAgentRunning.value) {
      await initiateRegistration()
    }
  } catch { ... }
}
```

`initiateRegistration()` 调用后端生成一个 OTP 但它**，但这个 OTP 没用**，Agent 会自己生成新的。

#### 2. Agent 显示 OTP

**文件**: `packages/hoppscotch-agent/src/App.vue:169-176`

```typescript
const handleRegistrationReceived = (payload: string) => {
  appState.value = {
    ...state(),
    view: "otp",
    otp: O.some(payload),  // payload 是 Agent 生成的 OTP
  }
  getCurrentWindow().setFocus()
}
```

Agent 窗口自动弹出并置顶，显示 6 位 OTP，用户需要把这个 OTP 复制粘贴回 Web 前端。

#### 3. 验证注册

**文件**: `packages/hoppscotch-common/src/components/settings/AgentSubtitle.vue:121-133`

```typescript
const register = async () => {
  if (!store.registrationOTP.value) return  // 用户输入的 Agent 显示的 OTP
  store.isRegistering.value = true
  try {
    await store.verifyRegistration(store.registrationOTP.value)  // 发送 OTP + 公钥
    await updateMaskedAuthKey()
    toast.success(t("settings.agent_registration_successful"))
  } finally {
    store.isRegistering.value = false
  }
}
```

**文件**: `packages/hoppscotch-common/src/platform/std/kernel-interceptors/agent/store.ts:230-258`

```typescript
public async verifyRegistration(otp: string): Promise<void> {
  // 浏览器生成 X25519 密钥对
  const myPrivateKey = crypto.getRandomValues(new Uint8Array(32))
  const myPublicKey = x25519.getPublicKey(myPrivateKey))
  const myPublicKeyB16 = base16.encode(myPublicKey)).toLowerCase()

  const response = await axios.post(
    "http://localhost:9119/verify-registration",
    {
      registration: otp,  // 用户输入的 Agent 生成的 OTP
      client_public_key_b16: myPublicKeyB16,
    }
  )

  // 计算共享密钥
  const agentPublicKey = new Uint8Array(
    base16.decode(agentPublicKeyB16.toUpperCase())
  )
  const sharedSecret = x25519.getSharedSecret(myPrivateKey, agentPublicKey))
  const sharedSecretB16 = base16.encode(sharedSecret)).toLowerCase()

  this.authKey.value = newAuthKey
  this.sharedSecretB16.value = sharedSecretB16
  await this.persistStore()
}
```

#### 4. Agent 端验证

**文件**: `packages/hoppscotch-agent/src-tauri/src/controller.rs:116-182`

```rust
pub async fn verify_registration(...) -> AgentResult<Json<AuthKeyResponse>> {
    // 验证 OTP 是否匹配
    if !state.validate_registration(&confirmed_registration.registration).await {
        return Err(AgentError::InvalidRegistration);
    }

    // 生成 auth_key（UUID）
    let auth_key = Uuid::new_v4().to_string();
    
    // Agent 也生成 X25519 密钥对
    let secret_key = EphemeralSecret::random();
    let public_key = PublicKey::from(&secret_key);

    // 计算共享密钥
    let their_public_key = PublicKey::from(...);
    let shared_secret = secret_key.diffie_hellman(&their_public_key);

    // 保存注册信息
    state.update_registrations(...);

    Ok(Json(AuthKeyResponse {
        auth_key,
        created_at,
        agent_public_key_b16: base16::encode_lower(public_key.as_bytes()),
    })
}
```

#### 5. 持久化

注册凭证保存在浏览器的 Kernel Store 中（`interceptors.agent.v1`），下次访问无需重新注册。

---

## 四、注册失效探测与注销

### 注册失效探测

**文件**: `packages/hoppscotch-agent/src-tauri/src/controller.rs:264-284`

`/registered-handshake` 端点**存在**，用于已注册客户端验证连接有效性：

```rust
pub async fn registered_handshake(...) -> AgentResult<EncryptedJson<...>> {
    // 如果 auth_key 有效，返回加密的 true
    // 如果无效，返回 401 Unauthorized
}
```

⚠️ **但是**，**前端代码中**没有调用这个端点**！

### 实际失效探测发生在：

1. **`fetchRegistrationInfo()` 调用 `/registration` 时**：
   ```typescript
   public async fetchRegistrationInfo(): Promise<...>> {
     try {
       const response = await axios.get("http://localhost:9119/registration", {
         headers: { Authorization: `Bearer ${this.authKey.value} },
         responseType: "arraybuffer",
       })
       // 解密返回
     } catch (error) {
       if (axios.isAxiosError(error)) {
         if (error.response?.status === 401) {
           this.authKey.value = null  // 401 时自动清除
           await this.persistStore()
         }
       }
       throw error
     }
   }
   ```

2. **每次执行请求时**：`/execute` 401 也会触发错误，但前端没有自动处理，只会报错。

### 注销路径

#### 前端单方面注销

**文件**: `packages/hoppscotch-common/src/components/settings/AgentSubtitle.vue:135-141`

```typescript
const resetRegistration = async () => {
  await store.resetAuthKey()  // 只清前端本地的！
  store.maskedAuthKey.value = ""
  store.registrationOTP.value = ""
  store.hasInitiatedRegistration.value = false
  store.hasCheckedAgent.value = false
}
```

⚠️ **只清除了前端的 authKey 和 sharedSecret，**没有调用后端删除接口**！

#### 后端删除接口（前端未调用）

**文件**: `packages/hoppscotch-agent/src-tauri/src/controller.rs:185-202`

```rust
pub async fn delete_registration(...) -> AgentResult<Json<...>> {
    if !state.validate_access(auth_header.token()) {
        return Err(AgentError::Unauthorized);
    }
    state.update_registrations(app_handle.clone(), |regs| {
        regs.remove(&auth_key);  // 从后端删除
    })?;
}
```

**当前状态**：
- 前端"注销"是**单向的**：前端清除自己的凭证，但后端还保留注册记录
- Agent 端的注册列表页面也**没有删除按钮**，只能查看不能删除
-  `/registered-handshake` 端点存在但前端没使用

---

## 五、请求转发实现

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
  const nonceB16 = base16.encode(nonce)).toLowerCase()

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

## 六、Desktop 平台 vs Web 平台的差异

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

## 七、关键错误总结（修正点）

### 1. **OTP 生成位置

| 之前错误 | 正确理解 |
|-----------|----------|
| 浏览器生成 OTP 发给 Agent | **Agent 自己生成 OTP！浏览器生成的 OTP 被完全忽略 |
| 浏览器知道 OTP 值 | 浏览器不知道 OTP，必须由用户从 Agent 窗口复制粘贴 |

### 2. 注册失效探测

| 之前错误 | 正确理解 |
|-----------|----------|
| （未提及） | `/registered-handshake` 存在但前端**未使用** |
| （未提及） | 实际靠 `/registration` 返回 401 时自动清除前端 authKey |

### 3. 注销路径

| 之前错误 | 正确理解 |
|-----------|----------|
| （未提及后端删除） | 前端"注销"是**单向**的，只清前端凭证，后端还保留注册 |
| （未提及） | 后端有 DELETE 接口但前端**从未调用 |
| （未提及） | Agent 注册列表页**没有删除按钮 |

### 4. 拦截器持久化恢复

| 之前错误 | 正确理解 |
|-----------|----------|
| （描述较简单） | 有双向同步（内存 ↔ 存储） |
| （未提及） | 有有效性校验，无效时自动回退 |

---

## 八、关键文件索引

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
| `hoppscotch-agent/src/App.vue` | Agent 前端主页面（OTP 显示、注册列表） |

### 前端（TypeScript/Vue）

| 文件 | 职责 |
|------|------|
| `hoppscotch-common/src/platform/std/kernel-interceptors/agent/index.ts` | Agent 拦截器服务、请求执行主逻辑 |
| `hoppscotch-common/src/platform/std/kernel-interceptors/agent/store.ts` | Agent 状态管理、加解密、注册、状态检测 |
| `hoppscotch-common/src/services/kernel-interceptor.service.ts` | 拦截器注册与切换的核心服务 |
| `hoppscotch-common/src/modules/kernel-interceptors.ts` | 拦截器模块初始化、设置双向同步 |
| `hoppscotch-common/src/components/settings/Agent.vue` | Agent 设置 UI（域名设置、证书、代理） |
| `hoppscotch-common/src/components/settings/AgentSubtitle.vue` | Agent 注册/状态 UI |
| `hoppscotch-common/src/helpers/functional/process-request.ts` | 请求预处理（URL 参数编码、superjson 序列化） |
| `hoppscotch-selfhost-web/src/main.ts` | 平台配置（Web/Desktop 拦截器列表） |
| `hoppscotch-common/src/platform/kernel-interceptors.ts` | 拦截器平台定义类型 |
