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

## 四、注册生命周期

### 注册的两阶段状态

Agent 中的注册分为两个不同的阶段和状态，生命周期完全不同：

| 状态 | 存储位置 | 生命周期 | 持久化 |
|------|----------|----------|--------|
| `active_registration_code`（进行中的 OTP） | `AppState.active_registration_code: RwLock<Option<String>>` | **短暂**：从生成到验证成功或窗口关闭 | **不持久化**，仅内存 |
| `registrations`（已完成的注册） | `AppState.registrations: DashMap<String, Registration>` | **持久**：直到被显式删除 | **持久化**到 `app_data.bin` |

### 阶段一：active_registration_code 的生命周期

`active_registration_code` 是一个"正在进行的注册"标记，同一时刻只能有一个。

#### 生成时机

**文件**: `packages/hoppscotch-agent/src-tauri/src/controller.rs:53-83`

当浏览器 POST `/receive-registration` 时，Agent 调用 `generate_otp()` 生成 6 位 OTP 并存入 `active_registration_code`。同时通过 `app_handle.emit("registration-received", otp)` 通知 Agent 前端窗口显示 OTP。

如果已有一个 `active_registration_code` 存在（即另一个注册正在进行），Agent 拒绝新注册请求：

```rust
if !active_registration_code.is_none() {
    return Ok(Json(json!({ "message": "There is already an existing registration happening" })));
}
```

#### 清除时机（三处）

1. **验证成功时**（`controller.rs:174`）：

   `verify_registration()` 成功后显式调用：
   ```rust
   let _ = state.clear_active_registration().await;
   ```
   同时 emit `"authenticated"` 事件，Agent 窗口收到后自动隐藏。

2. **Agent 窗口关闭时**（`lib.rs:220-225`）：

   窗口关闭请求被拦截后，清除 `active_registration_code`：
   ```rust
   tauri::WindowEvent::CloseRequested { api, .. } => {
       api.prevent_close();
       window.hide();
       let mut current_code = app_state.active_registration_code.blocking_write();
       if current_code.is_some() {
           *current_code = None;  // 清除进行中的注册
       }
       window.emit("window-hidden", ());
   }
   ```
   这意味着：**如果用户在注册完成前关闭了 Agent 窗口，OTP 会被清除，正在进行的注册流程将无法完成。**

3. **Agent 重启时**：

   `active_registration_code` 只存于内存（`RwLock<Option<String>>`），不持久化。Agent 重启后自动为 `None`。这意味着：**重启 Agent 会使进行中的注册流程作废。**

#### 不清除的情况

- 浏览器端超时或用户放弃注册，只要 Agent 窗口未关闭且 Agent 未重启，`active_registration_code` 就会一直保留，阻止新的注册发起。

### 阶段二：registrations 的生命周期

`registrations` 是已完成的注册列表，key 是 `auth_key`（UUID），value 是 `Registration`（含 `registered_at` 和 `shared_secret_b16`）。

#### 创建时机

**文件**: `packages/hoppscotch-agent/src-tauri/src/controller.rs:151-162`

`verify_registration()` 成功后通过 `update_registrations()` 写入：
```rust
regs.insert(auth_key_copy, Registration {
    registered_at: created_at,
    shared_secret_b16: base16::encode_lower(shared_secret.as_bytes()),
});
```

#### 持久化

**文件**: `packages/hoppscotch-agent/src-tauri/src/state.rs:83-130`

`update_registrations()` 不仅更新内存中的 DashMap，还同步写入 `tauri_plugin_store`（文件 `app_data.bin`，key `"registrations"`）：

```rust
pub fn update_registrations(&self, app_handle, update_func) -> Result<(), AgentError> {
    update_func(&self.registrations);  // 更新内存

    let store = app_handle.store(AGENT_STORE)?;
    if store.has(REGISTRATIONS) {
        store.delete(REGISTRATIONS)?;  // 先清旧的
    }
    store.set(REGISTRATIONS, serde_json::to_value(self.registrations.clone())?);
    store.save()?;  // 写盘
}
```

**关键设计**：此方法绕过 `store.reload()`，以内存中的 `self.registrations` 为唯一真相来源，避免磁盘上的过期数据覆盖最新变更。

#### 加载时机

**文件**: `packages/hoppscotch-agent/src-tauri/src/state.rs:30-65`

Agent 启动时 `AppState::new()` 从持久化存储恢复：
```rust
let registrations = store
    .get(REGISTRATIONS)
    .and_then(|val| serde_json::from_value(val.clone()).ok())
    .unwrap_or_else(|| DashMap::new());
```

**结论：已完成的注册在 Agent 重启后依然有效。**

#### 删除时机（三处）

1. **DELETE `/registrations/:auth_key` API**（`controller.rs:185-202`）：

   通过 HTTP 接口删除单个注册（需要 Bearer Token 认证 + auth_key 路径参数）。前端**当前未调用**此接口。

2. **系统托盘 "Clear Registrations" 菜单项**（`tray.rs:83-89`）：

   ```rust
   "clear_registrations" => {
       let app_state = app.state::<Arc<AppState>>();
       app_state.clear_registrations(app.clone())
           .expect("Invariant violation: Failed to clear registrations");
   }
   ```
   调用 `state.rs:134` 的 `clear_registrations()`，清空**所有**注册：
   ```rust
   pub fn clear_registrations(&self, app_handle) -> Result<(), AgentError> {
       self.update_registrations(app_handle, |regs| regs.clear())?;
   }
   ```
   这是 Agent 端**唯一可用的注册删除入口**，但它会删除全部注册，不支持删除单个。

3. **卸载/删除 Agent 应用数据**：直接删除 `app_data.bin` 文件。

#### 不存在过期机制

`Registration` 结构只有 `registered_at` 和 `shared_secret_b16` 两个字段，**没有 TTL 或过期时间**。注册一旦创建，永远不会自动过期失效。

---

### 注册失效探测

#### 方式一：`/registered-handshake` 端点（存在但前端未使用）

**文件**: `packages/hoppscotch-agent/src-tauri/src/controller.rs:264-284`

```rust
pub async fn registered_handshake(
    TypedHeader(auth_header): TypedHeader<Authorization<Bearer>>,
) -> AgentResult<EncryptedJson<serde_json::Value>> {
    match state.get_registration(auth_header.token()) {
        Some(reg) => Ok(EncryptedJson { key_b16: reg.shared_secret_b16, data: json!(true) }),
        None => Err(AgentError::Unauthorized),  // → HTTP 401
    }
}
```

此端点用共享密钥加密响应 `true`，客户端需要同时拥有有效 auth_key 和 shared_secret 才能验证成功。**但前端代码从未调用此端点。**

#### 方式二：`fetchRegistrationInfo()` 调用 `/registration` 时（实际生效）

**文件**: `packages/hoppscotch-common/src/platform/std/kernel-interceptors/agent/store.ts:260-287`

```typescript
public async fetchRegistrationInfo(): Promise<{ registered_at: Date; auth_key_hash: string }> {
  try {
    const response = await axios.get("http://localhost:9119/registration", {
      headers: { Authorization: `Bearer ${this.authKey.value}` },
      responseType: "arraybuffer",
    })
    return await this.decryptResponse(nonceB16, response.data)
  } catch (error) {
    if (axios.isAxiosError(error) && error.response?.status === 401) {
      this.authKey.value = null       // 清除内存中的 authKey
      await this.persistStore()        // 持久化清除结果
    }
    throw error
  }
}
```

此方法在 `AgentSubtitle.vue` 的 `onMounted` 和注册成功后调用（`updateMaskedAuthKey`），是前端感知注册是否有效的唯一途径。当返回 401 时，前端自动清除本地凭证。

#### 方式三：`/execute` 请求时（被动感知）

每次执行代理请求时，如果 auth_key 在 Agent 端已不存在，`/execute` 返回 401。但前端 `executeRequest()` 没有像 `fetchRegistrationInfo()` 那样对 401 做特殊处理，只会抛出错误显示给用户。

---

### 注销路径（完整对比）

注销涉及两个独立的数据存储，清理路径不对称：

```
┌─────────────────────────────────────────────────────────────┐
│                    注销路径全景                               │
│                                                             │
│   前端 Kernel Store              Agent 后端 DashMap + disk  │
│   (interceptors.agent.v1)        (app_data.bin)             │
│                                                             │
│   ┌─────────────────┐            ┌──────────────────────┐  │
│   │ auth.key         │            │ registrations[ak_A]  │  │
│   │ auth.sharedSecret│            │ registrations[ak_B]  │  │
│   │ domains          │            │ registrations[ak_C]  │  │
│   └────────┬─────────┘            └──────────┬───────────┘  │
│            │                                  │              │
│   路径 A: 前端"注销"按钮           路径 C: 托盘               │
│   resetAuthKey()                   "Clear Registrations"    │
│   → auth.key = null               → regs.clear()           │
│   → auth.sharedSecret = null      → persist to disk         │
│   → persistStore()                                          │
│            │                                  │              │
│            │                        路径 D: DELETE API       │
│            │                        /registrations/:auth_key │
│            │                        (前端未调用)              │
│            │                        → regs.remove(&ak)       │
│            │                        → persist to disk         │
│            │                                  │              │
│            ▼                                  ▼              │
│   ✅ 前端凭证清除                    ✅ 后端注册清除           │
│   ❌ 后端注册保留                    ❌ 前端凭证保留           │
│   (下次发请求会 401)               (前端凭残留凭证            │
│                                    发请求仍会 401)           │
└─────────────────────────────────────────────────────────────┘
```

#### 路径 A：前端"注销"按钮（只清前端）

**文件**: `packages/hoppscotch-common/src/components/settings/AgentSubtitle.vue:135-141`

```typescript
const resetRegistration = async () => {
  await store.resetAuthKey()  // → authKey = null, sharedSecretB16 = null
  store.maskedAuthKey.value = ""
  store.registrationOTP.value = ""
  store.hasInitiatedRegistration.value = false
  store.hasCheckedAgent.value = false
}
```

`resetAuthKey()` 实现（`store.ts:129-133`）：
```typescript
public async resetAuthKey(): Promise<void> {
  this.authKey.value = null
  this.sharedSecretB16.value = null
  await this.persistStore()
}
```

`persistStore()` 将 `auth.key: null, auth.sharedSecret: null` 写入 Kernel Store，但**域名设置 `domains` 被保留**。

**效果**：前端丢失凭证，但 Agent 后端的注册记录仍然存在。前端无法再发请求（authKey 为 null），但 auth_key 在 Agent 端仍占一个注册槽位。

#### 路径 C：Agent 托盘 "Clear Registrations"（只清后端，且清全部）

**文件**: `packages/hoppscotch-agent/src-tauri/src/tray.rs:83-89`

```rust
"clear_registrations" => {
    let app_state = app.state::<Arc<AppState>>();
    app_state.clear_registrations(app.clone())
        .expect("Invariant violation: Failed to clear registrations");
}
```

**效果**：Agent 后端清空所有注册（内存 + 磁盘），但前端的 Kernel Store 仍保留 `auth.key` 和 `auth.sharedSecret`。前端下次尝试发请求时，Agent 返回 401 → 前端不会自动清除凭证，只显示错误。

#### 路径 D：DELETE `/registrations/:auth_key` API（只清后端，删单个，前端未调用）

**文件**: `packages/hoppscotch-agent/src-tauri/src/controller.rs:185-202`

此接口的访问控制存在一个重要细节，详见下方"删除接口访问控制边界"。

#### 综合结论：注销是不对称的

| 操作 | 前端 Kernel Store | Agent 后端注册 | 域名设置 |
|------|-------------------|---------------|----------|
| 前端"注销"按钮 | ✅ 清除 auth | ❌ 保留 | ✅ 保留 |
| 托盘 "Clear Registrations" | ❌ 保留 auth | ✅ 清除全部 | — |
| DELETE API（未调用） | ❌ 保留 auth | ✅ 删除指定 | — |

**没有一条路径能同时清理前端和后端**。要完全注销，需要两步操作：
1. 前端点击"注销"清除本地凭证
2. Agent 托盘点击 "Clear Registrations" 清除后端记录（但会清除所有注册）

---

### 注册成功链路的异常状态一致性

`verify_registration()` 函数的成功路径涉及多个独立的状态变更，这些变更并非原子性的，存在部分成功部分失败的风险。

#### 成功链路的 5 个独立步骤

**文件**: `packages/hoppscotch-agent/src-tauri/src/controller.rs:115-182`

```rust
pub async fn verify_registration(...) -> AgentResult<Json<AuthKeyResponse>> {
    // 步骤 1: 验证 OTP（读操作，无副作用）
    if !state.validate_registration(...).await {
        return Err(AgentError::InvalidRegistration);
    }

    // 步骤 2: 生成密钥（纯计算，无副作用）
    let auth_key = Uuid::new_v4().to_string();
    let secret_key = EphemeralSecret::random();
    let shared_secret = secret_key.diffie_hellman(&their_public_key);

    // 步骤 3: 更新注册（内存 + 磁盘）
    state.update_registrations(app_handle.clone(), |regs| {
        regs.insert(auth_key_copy, Registration { ... });
    })?;  // ← 如果这里失败，前面的都回滚了

    // 步骤 4: emit "authenticated" 事件（通知 Agent 前端 UI）
    app_handle.emit("authenticated", &auth_payload)?;  // ← 如果这里失败？

    // 步骤 5: 清除 active_registration_code（内存）
    let _ = state.clear_active_registration().await;  // ← 忽略错误！

    Ok(Json(AuthKeyResponse { ... }))
}
```

#### 异常场景 1：`update_registrations` 失败（步骤 3）

`update_registrations()` 内部有 3 个可能的失败点：

**文件**: `packages/hoppscotch-agent/src-tauri/src/state.rs:83-130`

```rust
pub fn update_registrations(&self, app_handle, update_func) -> Result<(), AgentError> {
    update_func(&self.registrations);  // 1. 更新内存 → 总能成功

    let store = app_handle.store(AGENT_STORE)?;  // 2. 访问 store → 可能失败

    if store.has(REGISTRATIONS) {
        store.delete(REGISTRATIONS)?;  // 3. 删除旧的 → 可能失败
    }

    store.set(REGISTRATIONS, serde_json::to_value(...)?)?;  // 4. 设置新值 → 可能失败
    store.save()?;  // 5. 持久化到磁盘 → 可能失败

    Ok(())
}
```

**不一致风险**：
- `update_func(&self.registrations)` **总是先执行**，内存已被修改
- 后续步骤（`store.delete` / `store.set` / `store.save`）**任何一个失败**，都会导致：
  - ✅ **内存**：已更新（新注册已加入）
  - ❌ **磁盘**：未更新（`app_data.bin` 仍为旧值）
- 函数返回 `Err` 给调用者，但内存状态已经改变
- **Agent 重启后，内存状态丢失，新注册消失**

#### 异常场景 2：`emit("authenticated")` 失败（步骤 4）

**文件**: `packages/hoppscotch-agent/src-tauri/src/controller.rs:169-172`

```rust
if let Err(e) = app_handle.emit("authenticated", &auth_payload) {
    tracing::error!("Failed to emit authenticated event: {:?}", e);
    return Err(AgentError::InternalServerError);  // ← 返回错误
}
```

**不一致风险**：
- 步骤 3（`update_registrations`）已成功：✅ 内存 + ✅ 磁盘 都有新注册
- 步骤 4 emit 失败：返回 `Err(InternalServerError)`
- 步骤 5（`clear_active_registration`）**不会执行**
- 结果：
  - ✅ 新注册已持久化
  - ❌ `active_registration_code` 仍保留（阻止新的注册发起）
  - ❌ Agent 前端窗口不会自动隐藏（因为没收到 `authenticated` 事件）
  - ❌ HTTP 返回 500 给前端，前端**不会保存 auth_key**
- **最终状态**：后端有注册记录，但前端没有凭证，这个注册永久"孤儿化"

#### 异常场景 3：`clear_active_registration` 失败（步骤 5）

```rust
let _ = state.clear_active_registration().await;  // 错误被忽略！
```

**不一致风险**：
- 即使清除失败，`_` 忽略了错误，函数仍返回 `Ok`
- 结果：
  - ✅ 注册已成功创建
  - ✅ 前端已收到 auth_key
  - ❌ `active_registration_code` 仍保留（阻止新的注册发起）
- 但这个风险较低：`clear_active_registration()` 只是获取 `RwLock` 写锁然后设为 `None`，几乎不会失败

#### 前端侧的异常处理

**文件**: `packages/hoppscotch-common/src/platform/std/kernel-interceptors/agent/store.ts:230-258`

```typescript
public async verifyRegistration(otp: string): Promise<void> {
  const response = await axios.post("/verify-registration", { ... })
  // ← 如果 HTTP 请求失败（网络错误、500 等），直接抛出异常

  const { auth_key: newAuthKey, agent_public_key_b16 } = response.data

  if (typeof newAuthKey !== "string")
    throw new Error("Invalid auth key received")  // ← 验证失败

  // 计算共享密钥...

  this.authKey.value = newAuthKey                // ← 只有全部成功才写入内存
  this.sharedSecretB16.value = sharedSecretB16
  await this.persistStore()                      // ← 写入 Kernel Store
}
```

前端处理是**原子性的**：要么全部成功，要么全部不写入。

#### AgentSubtitle 中的 catch-all

**文件**: `packages/hoppscotch-common/src/components/settings/AgentSubtitle.vue:121-133`

```typescript
const register = async () => {
  store.isRegistering.value = true
  try {
    await store.verifyRegistration(store.registrationOTP.value)
    await updateMaskedAuthKey()
    toast.success(...)
  } catch (_e) {
    // 静默失败，不做任何清理！
  } finally {
    store.isRegistering.value = false
  }
}
```

**catch (_e)** 吞掉了所有异常，包括后端状态不一致的情况。用户只会看到 spinner 消失，但不知道后端可能留下了孤儿注册。

---

### 删除注册接口的访问控制边界

**文件**: `packages/hoppscotch-agent/src-tauri/src/controller.rs:184-202`

```rust
pub async fn delete_registration(
    State((state, app_handle)): State<(Arc<AppState>, AppHandle)>,
    TypedHeader(auth_header): TypedHeader<Authorization<Bearer>>,  // 请求者的 token
    Path(auth_key): Path<String>,                                  // 要删除的 auth_key
) -> AgentResult<Json<serde_json::Value>> {
    if !state.validate_access(auth_header.token()) {  // 只验证请求者自己是否已注册
        return Err(AgentError::Unauthorized);
    }
    state.update_registrations(app_handle.clone(), |regs| {
        regs.remove(&auth_key);  // 删除路径参数指定的 auth_key
    })?;
}
```

#### 访问控制分析

认证逻辑**只检查请求者（`auth_header.token()`）是否是已注册的客户端**，不检查请求者是否有权删除目标 `auth_key`。这意味着：

**任何已注册的客户端都可以删除任意（包括其他客户端的）注册。**

```
客户端 A（auth_key = "aaa"）
客户端 B（auth_key = "bbb"）

DELETE /registrations/bbb
Authorization: Bearer aaa   ← 请求者是 A

→ validate_access("aaa") → true  ← A 是已注册的，通过验证
→ regs.remove("bbb")              ← B 的注册被 A 删除了！
```

#### 安全影响

由于 Agent 仅监听 `127.0.0.1:9119`，只有本机进程能访问。但以下场景仍有风险：

1. **同机多浏览器/多用户**：如果同一台机器上有多个浏览器实例分别注册了不同的 auth_key，任何一个都可以删除其他的。
2. **恶意本地程序**：任何能访问 `localhost:9119` 的本地程序，只要获得任一 auth_key，就能删除所有注册。
3. **XSS 攻击**：如果 Hoppscotch Web 前端存在 XSS 漏洞，攻击者可以从 `Kernel Store` 读取 auth_key，然后调用 DELETE API 删除其他客户端的注册。

#### 改进方向（代码现状未实现）

合理的访问控制应该是：**请求者只能删除自己的注册**，即 `auth_header.token() == auth_key`。当前代码未做此限制。

---

## 五、请求取消机制的并发风险分析

### 取消机制的两层架构

取消机制实际分为**两层独立的实现**，分别位于不同的代码库：

| 层级 | 存储位置 | 实现方式 | 用途 |
|------|----------|----------|------|
| Layer 1: AppState | `AppState.cancellation_tokens: DashMap<usize, CancellationToken>` | Agent 端内部 | **未实际使用** |
| Layer 2: relay crate | `ACTIVE_REQUESTS: DashMap<i64, Arc<AtomicBool>>` | relay 全局静态 | **实际生效** |

⚠️ **重要发现**：`AppState.cancellation_tokens` 虽然有 `add_cancellation_token()` 和 `remove_cancellation_token()` 方法，但**在整个代码库中从未被调用过**！实际生效的是 relay crate 中的 `ACTIVE_REQUESTS`。

### Layer 2: relay 中的取消实现

**文件**: `packages/hoppscotch-desktop/plugin-workspace/relay/src/relay.rs:22-175`

```rust
lazy_static::lazy_static! {
    static ref ACTIVE_REQUESTS: DashMap<i64, Arc<AtomicBool>> = DashMap::new();
}

pub async fn execute(request: Request) -> Result<Response> {
    let request_id = request.id;
    let cancelled = Arc::new(AtomicBool::new(false));

    ACTIVE_REQUESTS.insert(request_id, Arc::clone(&cancelled));  // ← 加入活跃请求表

    let cancel_token = CancellationToken::new();
    let cancel_token_clone = cancel_token.clone();
    let cancelled_clone = Arc::clone(&cancelled);

    // 派生一个 OS 线程执行阻塞的 curl 操作
    let handle = std::thread::spawn(move || {
        let result = execute_request(&request, &cancel_token);
        if cancel_token_clone.is_cancelled() {
            cancelled_clone.store(true, Ordering::SeqCst);
        }
        result
    });

    let result = match handle.join() { ... };  // ← 等待线程结束

    ACTIVE_REQUESTS.remove(&request_id);  // ← 从活跃请求表移除
    result
}

pub async fn cancel(request_id: i64) -> Result<()> {
    if let Some(cancelled) = ACTIVE_REQUESTS.get(&request_id) {
        cancelled.store(true, Ordering::SeqCst);  // ← 只设置标志位！
        Ok(())
    } else {
        Err(RelayError::Network { message: "Request not found".into(), cause: None })
    }
}
```

### 并发风险 1：取消机制的"软取消"问题

**关键发现**：`cancel()` 函数**只设置 `AtomicBool` 标志位，并不实际调用 `CancellationToken::cancel()`**！

```rust
// cancel() 只做这件事：
cancelled.store(true, Ordering::SeqCst);

// 但 CancellationToken 的取消需要显式调用：
// cancel_token.cancel()  ← 从未被调用！
```

**影响**：
- `execute_request()` 中的 `transfer_handler.handle_transfer(&mut handle, cancel_token)` 监听的是 `cancel_token`，而不是 `cancelled` 标志
- 取消请求实际上**不会中断正在进行的 HTTP 传输**
- `cancelled` 标志只在**线程结束后**检查，用于决定返回值是成功还是取消
- **用户体验问题**：点击"取消"按钮后，网络传输可能还在继续，只是最终结果被标记为取消

### 并发风险 2：request_id 生成的竞态条件

**文件**: `packages/hoppscotch-common/src/platform/std/kernel-interceptors/agent/index.ts:82`

```typescript
public execute(request: RelayRequest): ExecutionResult {
  const reqID = Date.now()  // ← 毫秒级时间戳作为 request_id
  const cancelToken = axios.CancelToken.source()
  // ...
}
```

**问题**：`Date.now()` 的精度只有 **1 毫秒**。

**竞态场景**：
1. 时间 T: 用户快速连续点击发送按钮两次
2. 两次请求都在同一毫秒内执行
3. 两个请求获得**相同的 reqID**（例如 `1716800000000`）
4. 第一次请求加入 `ACTIVE_REQUESTS`，key 为 `1716800000000`
5. 第二次请求加入 `ACTIVE_REQUESTS`，**覆盖**了第一次的条目
6. 用户尝试取消第一个请求 → 找到的是第二个请求的 `AtomicBool`
7. 第一个请求的取消令牌**永久丢失**，无法取消

**影响范围**：
- 短时间内快速发送多个请求（自动化测试、脚本、快速点击）
- 所有请求都使用相同的 `auth_key`（同一浏览器）
- 取消操作可能取消错误的请求，或者根本无法取消

### 并发风险 3：ACTIVE_REQUESTS 的内存泄漏

`ACTIVE_REQUESTS` 是一个全局 `DashMap`，但：
- **没有清理过期条目的机制**
- **没有最大容量限制**
- 如果请求线程崩溃或 `join()` 失败，条目可能永远留在 map 中

**代码证据**：`execute()` 中 `ACTIVE_REQUESTS.remove(&request_id)` 只在正常路径调用，如果 `handle.join()` 发生 panic 或 `Err(_)`，remove 可能不会执行。

### 并发风险 4：跨客户端取消

**文件**: `packages/hoppscotch-agent/src-tauri/src/controller.rs:286-304`

```rust
pub async fn cancel(
    State((state, _app_handle)): State<(Arc<AppState>, AppHandle)>,
    TypedHeader(auth_header): TypedHeader<Authorization<Bearer>>,  // 请求者的 auth_key
    Path(request_id): Path<usize>,                                // 要取消的 request_id
) -> AgentResult<Json<serde_json::Value>> {
    if !state.validate_access(auth_header.token()) {  // 只验证请求者是否已注册
        return Err(AgentError::Unauthorized);
    }

    if let Ok(()) = relay::cancel(request_id.try_into().unwrap()).await {
        Ok(Json(json!({"message": "Request cancelled successfully"})))
    } else {
        Err(AgentError::RequestNotFound)
    }
}
```

**访问控制分析**：
- 与 DELETE API 有同样的问题：**只验证请求者是否已注册，不验证请求者是否是 request_id 的所有者**
- `ACTIVE_REQUESTS` 是全局的，不区分哪个 auth_key 发起的请求
- **任何已注册客户端都可以取消任何（包括其他客户端的）请求**

**攻击场景**：
1. 客户端 A 发起一个大文件下载请求（req_id = 12345）
2. 客户端 B（已注册）恶意调用 `POST /cancel/12345`
3. 客户端 A 的下载被意外中断

### 前端取消的双重机制

**文件**: `packages/hoppscotch-common/src/platform/std/kernel-interceptors/agent/index.ts:86-88`

```typescript
cancel: async () => {
  cancelToken.cancel()              // ← 机制 1: 取消 Axios HTTP 连接（浏览器 → Agent）
  await this.store.cancelRequest(reqId)  // ← 机制 2: 通知 Agent 取消实际请求
},
```

前端有两层取消：
1. **Axios CancelToken**：取消浏览器到 Agent 的 HTTP 连接
2. **POST /cancel/:reqId**：通知 Agent 取消实际转发的请求

但机制 2 存在上述的竞态和跨客户端问题。

---

## 六、请求转发实现

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

### 请求取消（旧描述已废弃）

详见**第五节 "请求取消机制的并发风险分析"**。

⚠️ **已废弃的旧理解**："Agent 端的 `/cancel/:req_id`：通过 Agent 的 `CancellationToken` 取消正在执行的 relay 请求。"

**实际正确机制**：
- 取消分为两层独立实现，`AppState.cancellation_tokens` 实际上**未被使用**
- 实际生效的是 relay crate 中全局的 `ACTIVE_REQUESTS: DashMap<i64, Arc<AtomicBool>>`
- `cancel()` 只设置 `AtomicBool` 标志位，**不调用 `CancellationToken::cancel()`**，不会中断正在传输的 HTTP 请求
- 存在 request_id 竞态、内存泄漏、跨客户端取消等并发风险

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

### 1. OTP 生成位置

| 之前错误 | 正确理解 |
|-----------|----------|
| 浏览器生成 OTP 发给 Agent | **Agent 自己生成 OTP！浏览器生成的 OTP 被完全忽略** |
| 浏览器知道 OTP 值 | 浏览器不知道 OTP，必须由用户从 Agent 窗口复制粘贴 |

### 2. 注册生命周期的两阶段

| 之前错误 | 正确理解 |
|-----------|----------|
| （未区分两阶段） | `active_registration_code`（进行中 OTP）和 `registrations`（已完成注册）是两个独立状态，生命周期完全不同 |
| （未提及 OTP 清除时机） | OTP 有三处清除：验证成功、窗口关闭、Agent 重启。窗口关闭会中断进行中的注册 |
| （未提及注册持久化） | 已完成注册持久化到磁盘，Agent 重启后仍然有效 |
| （未提及过期机制） | 注册**没有 TTL**，永不过期，只能显式删除 |

### 3. 注册失效探测

| 之前错误 | 正确理解 |
|-----------|----------|
| （未提及） | `/registered-handshake` 存在但前端**未使用** |
| （未提及） | 实际靠 `/registration` 返回 401 时自动清除前端 authKey |
| （未提及） | `/execute` 返回 401 时前端**不会**自动清除凭证，只报错 |

### 4. 注销路径（不对称）

| 之前错误 | 正确理解 |
|-----------|----------|
| （未提及后端删除） | 前端"注销"是**单向的**，只清前端凭证，后端还保留注册 |
| （未提及托盘清注册） | Agent 系统托盘有 "Clear Registrations" 菜单项，清全部后端注册但不清前端 |
| （未提及 DELETE API） | 后端有 DELETE 接口但前端**从未调用** |
| （未提及域名设置保留） | 前端"注销"保留域名设置（`domains`），只清除 auth 凭证 |
| （未提及完全注销需要两步） | **没有一条路径能同时清理前端和后端**，完全注销需两步操作 |

### 5. 删除注册接口的访问控制边界

| 之前错误 | 正确理解 |
|-----------|----------|
| （未分析访问控制） | DELETE API 只验证请求者是否已注册，**不验证是否有权删除目标 auth_key** |
| （未提及） | **任何已注册客户端都能删除任意（包括其他客户端的）注册** |
| （未提及安全影响） | 存在跨客户端删除风险，特别是同机多浏览器场景 |

### 6. 请求取消机制

| 之前错误 | 正确理解 |
|-----------|----------|
| 取消通过 Agent 的 `CancellationToken` 实现 | `AppState.cancellation_tokens` 实际上**从未被调用**，是死代码 |
| （未提及取消机制分层） | 实际取消机制在 relay crate 中，用全局 `ACTIVE_REQUESTS` DashMap |
| （未提及软取消问题） | `cancel()` 只设置 `AtomicBool` 标志，**不调用 `CancellationToken::cancel()`**，不会中断正在传输的 HTTP 请求 |
| （未提及竞态条件） | `reqID = Date.now()` 只有毫秒精度，快速发送请求时会产生**相同 request_id**，导致取消令牌覆盖丢失 |
| （未提及访问控制） | 与 DELETE API 一样，**任何已注册客户端都能取消任意（包括其他客户端的）请求** |

### 7. 注册成功链路的异常状态一致性

| 之前错误 | 正确理解 |
|-----------|----------|
| （假设原子性） | 注册成功链路有 5 个独立步骤，**非原子性**，存在部分成功部分失败的风险 |
| （未提及持久化失败场景） | `update_registrations()` 先改内存再写磁盘，**磁盘写入失败时内存已修改**，重启后新注册丢失 |
| （未提及 emit 失败场景） | `emit("authenticated")` 失败时，**后端注册已持久化但前端无凭证**，产生"孤儿注册" |
| （未提及前端 catch-all） | `AgentSubtitle.vue` 中 `catch (_e)` **吞掉所有异常**，用户感知不到后端状态不一致 |

### 8. 拦截器持久化恢复

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
| `hoppscotch-agent/src-tauri/src/tray.rs` | 系统托盘菜单（Clear Registrations、Show Registrations 等） |
| `hoppscotch-agent/src-tauri/src/error.rs` | AgentError 枚举与 HTTP 状态码映射 |
| `hoppscotch-agent/src/App.vue` | Agent 前端主页面（OTP 显示、注册列表） |

### Relay 库（Rust）

| 文件 | 职责 |
|------|------|
| `hoppscotch-desktop/plugin-workspace/relay/src/relay.rs` | HTTP 请求执行与取消的核心实现（ACTIVE_REQUESTS、execute、cancel） |

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
