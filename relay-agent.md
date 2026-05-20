# 本地 Agent 注册、目标地址改写与响应回传的接力关系分析

> **端到端代码走查版本**：所有结论均经代码逐行核对

## 系统架构概览

Hoppscotch 通过**内核拦截器（Kernel Interceptor）**架构实现跨平台的请求转发。核心执行路径：

```
用户发起请求 → KernelInterceptorService → 当前激活拦截器 → [拦截器内部逻辑] → 实际请求执行 → 响应原路返回
```

支持 5 种拦截器类型：

| 拦截器类型 | 用途 | 执行位置 |
|-----------|------|---------|
| `browser` | 浏览器原生 fetch（受 CORS 限制） | 浏览器 axios |
| `proxy` | 远端代理中继转发（绕过 CORS） | `https://proxy.hoppscotch.io/` |
| `agent` | 本地桌面 Agent（绕过 CORS + 高级功能） | `http://localhost:9119` |
| `native` | 桌面端原生（Tauri 插件） | Tauri Rust 后端 |
| `extension` | 浏览器扩展 | 浏览器扩展程序 |

---

## 一、注册流程端到端走查：口令的真实来源与校验关系

### 1.1 OTP 接力时序（逐行代码核对版）

> **关键修正**：前端生成的 OTP 被 Agent 完全忽略，Agent 重新生成 OTP 并通过弹窗显示给用户，用户输入的是 Agent 生成的 OTP。

```
【步骤 1】前端检查 Agent 状态
   │
   ├─ 代码位置：AgentSubtitle.vue:90-103 handleAgentCheck()
   └─ 调用 store.checkAgentStatus()
          ↓
          GET http://localhost:9119/handshake  [明文，无认证]
          ↓
          Agent 返回明文响应 {"status":"success","__hoppscotch__agent__":true}

=====================================================================

【步骤 2】前端发起注册（前端生成 OTP_A，但被忽略）
   │
   ├─ 代码位置：store.ts:215-227 initiateRegistration()
   │
   ├─ 🔴 前端生成 OTP_A = Math.floor(100000 + Math.random() * 900000).toString()
   │   （store.ts:216）
   │
   ├─ POST /receive-registration, 明文请求体 {"registration": OTP_A}
   │   （store.ts:218-221）
   │
   └─ 🔴 Agent 侧 receive_registration() [controller.rs:54-83]
          │
          ├─ 函数签名：State((state, app_handle))
          │   🔴 注意：没有 Json 提取器！完全不读取请求体！
          │
          ├─ 🔴 Agent 忽略 OTP_A，自己生成 OTP_B = generate_otp()
          │   （controller.rs:57）
          │
          ├─ 存储 OTP_B 到 state.active_registration_code
          │   （controller.rs:69）
          │
          ├─ Tauri 事件 "registration-received"，payload = OTP_B
          │   （controller.rs:71）
          │   ↓
          │   Agent UI App.vue:169-176 handleRegistrationReceived(payload)
          │   弹窗显示 OTP_B 给用户
          │
          └─ 返回明文响应 {"message":"Registration received and stored"}

=====================================================================

【步骤 3】用户手动输入 OTP_B
   │
   ├─ 用户看到 Agent 弹窗显示的 OTP_B
   ├─ 在前端输入框中输入 OTP_B
   ├─ 输入框绑定：v-model="store.registrationOTP.value"
   │   （AgentSubtitle.vue:15）
   │   🔴 注意：registrationOTP 初始值是 ""（store.ts:60），
   │           前端生成的 OTP_A 从未赋值给这个变量！
   │
   └─ 用户点击"确认"按钮
          ↓
          AgentSubtitle.vue:121-133 register()
          ↓
          调用 store.verifyRegistration(store.registrationOTP.value)
          🔴 参数就是用户输入的 OTP_B

=====================================================================

【步骤 4】校验 OTP + 密钥交换（明文！）
   │
   ├─ 代码位置：store.ts:230-258 verifyRegistration(otp)
   │   参数 otp = OTP_B
   │
   ├─ 生成客户端 ECDH 密钥对：x25519.getPublicKey()
   │
   ├─ POST /verify-registration，明文请求体：
   │   {
   │     "registration": OTP_B,                    ← 用户输入的
   │     "client_public_key_b16": "<client_pub>"
   │   }
   │
   └─ Agent 侧 verify_registration() [controller.rs:115-182]
          │
          ├─ 从请求体提取 confirmed_registration.registration = OTP_B
          │
          ├─ state.validate_registration(OTP_B)  [state.rs:152]
          │   比较：active_registration_code == OTP_B
          │   即：OTP_B == OTP_B  ✅ 校验通过
          │
          ├─ 生成 auth_key = Uuid::new_v4().to_string()
          │
          ├─ 生成服务端 ECDH 密钥对
          │
          ├─ 计算共享密钥 = secret_key.diffie_hellman(their_public_key)
          │
          ├─ 存储映射：auth_key → Registration { shared_secret_b16, ... }
          │
          ├─ 🔴 返回明文 JSON 响应！
          │   Ok(Json(AuthKeyResponse {
          │     auth_key,
          │     created_at,
          │     agent_public_key_b16: "<server_pub>"
          │   }))
          │   🔴 注意：是 Json(...) 不是 EncryptedJson(...)！
          │
          └─ 前端接收明文响应，计算共享密钥，保存 auth_key 和 sharedSecret
```

### 1.2 OTP 接力关系总结表

| OTP 版本 | 生成位置 | 生成方 | 用途 | 是否被实际使用 |
|---------|---------|-------|------|---------------|
| OTP_A | `store.ts:216` | 前端 Web App | 作为 `/receive-registration` 请求体发送 | ❌ 被 Agent 完全忽略 |
| OTP_B | `controller.rs:57` | 本地 Agent | 弹窗显示给用户，用户输入后校验 | ✅ 实际用于校验 |

**代码铁证**：
- `controller.rs:54` 函数签名 `State((state, app_handle))` **没有 `Json` 提取器**，不读取请求体
- `controller.rs:57` `let otp = generate_otp();` Agent 自己重新生成
- `store.ts:60` `registrationOTP = ref(this.authKey.value ? null : "")` 初始为空字符串
- `store.ts:227` `return otp` 返回了 OTP_A，但调用方 `AgentSubtitle.vue:107` **没有使用返回值**

---

## 二、明文交互与加密通信的边界划分

### 2.1 逐接口加密矩阵（代码逐行核对）

| 方法 | 路径 | Authorization 头 | 请求体 | 响应体 | 代码位置 |
|------|------|-----------------|--------|--------|---------|
| GET | `/handshake` | 无 | 空 | 🔓 明文 `Json` | `controller.rs:32-51` |
| POST | `/receive-registration` | 无 | 🔓 明文 JSON（但被忽略） | 🔓 明文 `Json` | `controller.rs:54-83` |
| POST | `/verify-registration` | 无 | 🔓 明文 JSON（OTP + 客户端公钥） | 🔓 明文 `Json`（auth_key + 服务端公钥） | `controller.rs:115-182` |
| GET | `/registered-handshake` | 🔓 明文 `Bearer <auth_key>` | 空 | 🔒 加密 `EncryptedJson` | `controller.rs:86-107` |
| GET | `/registration` | 🔓 明文 `Bearer <auth_key>` | 空 | 🔒 加密 `EncryptedJson` | `controller.rs:184-202` |
| DELETE | `/registrations/:auth_key` | 🔓 明文 `Bearer <auth_key>` | 空 | 🔓 明文 `Json` | `controller.rs:305-326` |
| POST | `/execute` | 🔓 明文 `Bearer <auth_key>` | 🔒 AES-GCM 加密二进制 | 🔒 加密 `EncryptedJson` | `controller.rs:204-255` |
| POST | `/cancel/:req_id` | 🔓 明文 `Bearer <auth_key>` | 🔓 明文 JSON（空对象 `{}`） | 🔓 明文 `Json` | `controller.rs:257-274` |
| POST | `/log-sink` | 🔓 明文 `Bearer <auth_key>` | 🔒 AES-GCM 加密二进制 | 🔓 明文 `Json` | `controller.rs:276-294` |

### 2.2 边界划分核心原则

> **永不加密的部分**：
> 1. HTTP 方法、路径、查询参数
> 2. HTTP 请求头（包括 `Authorization: Bearer <auth_key>`）
> 3. HTTP 响应头（包括 `X-Hopp-Nonce`、`Content-Type`）
>
> **可能加密的部分**：
> 只有 HTTP 请求体和响应体

> **加密标识**：
> - 请求加密：`Content-Type: application/octet-stream` + `X-Hopp-Nonce: <nonce_b16>`
> - 响应加密：`Content-Type: application/octet-stream` + `X-Hopp-Nonce: <nonce_b16>`

### 2.3 加密通信协议细节（核对版）

**请求加密流程** (`store.ts:289-317`):
```typescript
async encryptRequest(request, reqID): Promise<[nonceB16, encrypted]> {
  1. JSON.stringify(request) → 序列化
  2. TextEncoder.encode(...) → Uint8Array
  3. crypto.getRandomValues(new Uint8Array(12)) → 生成 12 字节 nonce
  4. crypto.subtle.importKey("raw", sharedSecret, {name:"AES-GCM"}, false, ["encrypt"])
  5. crypto.subtle.encrypt({ name: "AES-GCM", iv: nonce }, key, data)
  6. base16.encode(nonce) → nonceB16
  7. 返回 [nonceB16, encryptedBuffer]
}
```

**响应加密流程** (`util.rs:64-93`):
```rust
impl<T> IntoResponse for EncryptedJson<T> {
  fn into_response(self) -> Response {
    1. serde_json::to_vec(&self.data) → Vec<u8>
    2. base16::decode(&self.key_b16) → 共享密钥字节
    3. Aes256Gcm::generate_nonce(&mut OsRng) → 12 字节 nonce
    4. cipher.encrypt(&nonce, plaintext) → 密文
    5. 响应头：
       - Content-Type: application/octet-stream
       - X-Hopp-Nonce: base16::encode(&nonce)
    6. Body: 密文
  }
}
```

---

## 三、请求回传顺序完整梳理

### 3.1 执行层关键差异：哪些经过 Relay.execute

| 拦截器 | 是否经过 `Relay.execute` | 实际执行方式 |
|-------|-------------------------|-------------|
| browser | ✅ 是 | Web 内核 axios |
| proxy | ✅ 是（拦截器内部嵌套调用） | Web 内核 axios → 远端代理 |
| agent | ❌ 否（直接 axios） | 直接连接 localhost:9119 |
| native | ✅ 是 | Desktop 内核 Rust 插件 |
| extension | ❌ 否 | 浏览器扩展通信 |

**Relay.execute 分派逻辑** (`hoppscotch-kernel/src/index.ts:17-30`):
```typescript
window.__KERNEL__ = {
  relay: {
    execute: (req, meta) => window.__KERNEL__.relay[req.version].api.execute(req, meta),
    v1: { api: { execute: WEB_RELAY_IMPLS.v1.api.execute } }  // Web 环境
    // Desktop 环境: DESKTOP_RELAY_IMPLS.v1.api.execute
  }
}
```

### 3.2 Proxy 拦截器完整回传链（远端中继）

> **关键嵌套关系**：Proxy interceptor 改写目标地址后，**内部嵌套调用 `Relay.execute()`** 来实际发送请求到代理服务器。

```
┌──────────────────────────────────────────────────────────────────────────┐
│ 【层级 1】用户界面                                                      │
│  用户在 UI 点击"发送"按钮                                                │
└──────────────────────────────────────┬───────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────┐
│ 【层级 2】拦截器调度                                                    │
│  KernelInterceptorService.execute(originalRequest)                       │
│  选择当前激活拦截器 = "proxy"                                            │
└──────────────────────────────────────┬───────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────┐
│ 【层级 3】Proxy 拦截器执行                                              │
│  ProxyKernelInterceptorService.execute(originalRequest)                  │
│                                                                          │
│  ├─ 🔄 目标地址改写 (proxy/index.ts:205-273)                             │
│  │   1. preProcessRelayRequest(originalRequest)                         │
│  │   2. constructProxyRequest() → 构造 ProxyRequest:                    │
│  │      {                                                               │
│  │        url: originalRequest.url,        ← 原始目标地址               │
│  │        method: originalRequest.method,                               │
│  │        headers: originalRequest.headers,                             │
│  │        data: originalRequest.body,                                   │
│  │        wantsBinary: true,                                            │
│  │        accessToken: "..."                                            │
│  │      }                                                               │
│  │   3. 构造新的 RelayRequest (目标改写为代理服务器):                    │
│  │      {                                                               │
│  │        id: Date.now(),                                               │
│  │        url: proxyUrl,                   ← 🔴 https://proxy.hoppscotch.io │
│  │        method: "POST",                  ← 🔴 强制 POST                │
│  │        headers: { "content-type": "application/json" },              │
│  │        content: {                     ← 🔴 原始请求作为 Body 携带     │
│  │          kind: "json",                                               │
│  │          content: ProxyRequest,                                      │
│  │          mediaType: "application/json"                               │
│  │        }                                                             │
│  │      }                                                               │
│  │                                                                      │
│  └─ 🔴 嵌套调用！Relay.execute(proxyRelayRequest)                        │
│     (proxy/index.ts:275)                                                │
└──────────────────────────────────────┬───────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────┐
│ 【层级 4】内核执行层                                                    │
│  window.__KERNEL__.relay.execute(proxyRelayRequest)                      │
│  → WEB_RELAY_IMPLS.v1.api.execute()                                     │
│    (hoppscotch-kernel/src/relay/impl/web/v/1.ts:113-288)                │
│                                                                          │
│  ├─ axios config:                                                       │
│  │   {                                                                  │
│  │     url: proxyUrl,              ← 目标是代理服务器                   │
│  │     method: "POST",                                                 │
│  │     headers: { "content-type": "application/json" },                │
│  │     data: ProxyRequest,          ← 封装的原始请求                   │
│  │     responseType: "arraybuffer",                                     │
│  │     validateStatus: null                                             │
│  │   }                                                                  │
│  │                                                                      │
│  └─ axios.request(config) → 发送请求                                    │
└──────────────────────────────────────┬───────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────┐
│ 【层级 5】网络传输（HTTP）                                              │
│  POST https://proxy.hoppscotch.io/                                      │
│  实际传输：ProxyRequest JSON 作为请求体                                 │
└──────────────────────────────────────┬───────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────┐
│ 【层级 6】远端代理服务器                                                │
│  proxy.hoppscotch.io 接收请求                                           │
│                                                                          │
│  ├─ 1. 解析请求体 JSON → ProxyRequest                                   │
│  ├─ 2. 从 ProxyRequest 提取：                                           │
│  │   - 目标 URL: ProxyRequest.url                                       │
│  │   - 方法: ProxyRequest.method                                        │
│  │   - Headers: ProxyRequest.headers                                    │
│  │   - Body: ProxyRequest.data                                          │
│  │                                                                      │
│  ├─ 3. 代理服务器作为客户端发起真实请求：                                │
│  │   GET https://api.example.com/data                                   │
│  │   Headers: { Authorization: "Bearer xxx" }                           │
│  │   ↓                                                                  │
│  │   目标服务器返回响应 (status=200, headers, body)                     │
│  │                                                                      │
│  └─ 4. 包装为 ProxyResponse JSON:                                       │
│      {                                                                  │
│        success: true,                                                   │
│        status: 200,             ← 🔴 目标服务器真实状态码               │
│        statusText: "OK",                                                │
│        data: "<base64_encoded_body>",  ← binary 数据 base64 编码        │
│        isBinary: true,                                                  │
│        headers: { "content-type": "application/json" } ← 目标响应头     │
│      }                                                                  │
└──────────────────────────────────────┬───────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────┐
│ 【层级 7】网络传输（HTTP）                                              │
│  代理服务器返回 ProxyResponse JSON 给前端                               │
└──────────────────────────────────────┬───────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────┐
│ 【层级 8】内核响应处理                                                  │
│  axios 收到响应（arraybuffer 格式）                                     │
│  包装为 RelayResponse:                                                  │
│  {                                                                      │
│    status: 200,          ← 🔴 注意：这是到代理服务器的 HTTP 状态码！    │
│    statusText: "OK",                                                    │
│    headers: { ... },     ← 🔴 代理服务器的响应头，不是目标服务器的！    │
│    body: {                                                              │
│      body: <Uint8Array of ProxyResponse JSON>,                          │
│      mediaType: "application/json"                                      │
│    }                                                                    │
│  }                                                                      │
└──────────────────────────────────────┬───────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────┐
│ 【层级 9】Proxy 拦截器响应解析                                          │
│  回到 ProxyKernelInterceptorService (proxy/index.ts:346-434)            │
│                                                                          │
│  ├─ 1. parseBytesToJSON(res.body.body) → 解析 ProxyResponse             │
│  ├─ 2. 判断 ProxyResponse.success                                       │
│  ├─ 3. 处理数据:                                                        │
│  │   - isBinary = true:                                                 │
│  │     base64 解码 data → Uint8Array                                    │
│  │     尝试 JSON 解析，成功则编码回 UTF-8                                │
│  │     失败则作为二进制返回                                             │
│  │   - isBinary = false:                                                │
│  │     直接使用文本数据                                                 │
│  │                                                                      │
│  └─ 4. 🔴 重新包装为最终 RelayResponse（替换状态码和头）:                │
│     {                                                                   │
│       status: parsedProxyResponse.status,     ← 目标服务器真实状态码    │
│       statusText: parsedProxyResponse.statusText,                       │
│       headers: parsedProxyResponse.headers,   ← 目标服务器真实响应头    │
│       body: {                                                           │
│         body: <Uint8Array of actual response data>,                     │
│         mediaType: "application/json"                                   │
│       }                                                                 │
│     }                                                                   │
└──────────────────────────────────────┬───────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────┐
│ 【层级 10】响应适配层                                                   │
│  convertRelayResponseToSerializableResponse()                           │
│  (services/network/rest/rest.ts 或 hopp-fetch.ts)                        │
│                                                                          │
│  ├─ 1. 处理 headers:                                                    │
│  │   - 提取 Set-Cookie 到 getSetCookie()                                │
│  │   - 构造 headersObj（去重）                                          │
│  │                                                                      │
│  ├─ 2. 转换 body: Array → Uint8Array                                    │
│  │                                                                      │
│  └─ 3. 实现 Response 接口方法:                                          │
│     text(), json(), arrayBuffer(), blob(), formData()                   │
└──────────────────────────────────────┬───────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────┐
│ 【层级 11】应用层与 UI                                                  │
│  REST 服务 / Pre-request Script / Tests 处理响应                         │
│  ↓                                                                      │
│  UI 渲染：状态码、响应时间、Header 列表、Body 内容、Cookies 等           │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Agent 拦截器完整回传链（本地执行）

> **关键差异**：Agent interceptor **不经过 `Relay.execute`**，直接用 axios 与本地 Agent 通信。

```
┌──────────────────────────────────────────────────────────────────────────┐
│ 【层级 1】用户界面                                                      │
│  用户在 UI 点击"发送"按钮                                                │
└──────────────────────────────────────┬───────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────┐
│ 【层级 2】拦截器调度                                                    │
│  KernelInterceptorService.execute(originalRequest)                       │
│  选择当前激活拦截器 = "agent"                                            │
└──────────────────────────────────────┬───────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────┐
│ 【层级 3】Agent 拦截器执行                                              │
│  AgentKernelInterceptorService.execute(originalRequest)                  │
│  (agent/index.ts:81-177)                                                │
│                                                                          │
│  ├─ 🔄 预处理与改写 (agent/index.ts:81-164)                             │
│  │   1. preProcessRelayRequest(originalRequest)                         │
│  │   2. completeRequest() 补充 domain settings:                          │
│  │      - proxy 配置（HTTP/SOCKS 代理）                                  │
│  │      - security 配置（证书验证、CA 证书）                             │
│  │      - options 配置（重定向、超时、HTTP 版本）                        │
│  │   3. 添加 Cookie 头（从 CookieJarService）                            │
│  │   4. 添加 User-Agent 头                                              │
│  │   5. relayRequestToNativeAdapter() → 转换为原生格式                  │
│  │                                                                      │
│  ├─ 🔒 请求加密 (agent/index.ts:159-164)                                │
│  │   6. store.encryptRequest(postProcessedRequest, reqID):              │
│  │      - JSON 序列化 → AES-256-GCM 加密                                │
│  │      - 返回 [nonceB16, encryptedBuffer]                              │
│  │                                                                      │
│  └─ 🔴 直接 axios！不经过 Relay.execute                                  │
│     (agent/index.ts:165-177)                                            │
│     axios.post("http://localhost:9119/execute", encryptedReq, {         │
│       headers: {                                                        │
│         Authorization: `Bearer ${authKey}`,  ← 🔓 明文 auth_key        │
│         "X-Hopp-Nonce": nonceB16,           ← 🔓 明文 nonce            │
│         "Content-Type": "application/octet-stream" ← 标识加密          │
│       },                                                                │
│       responseType: "arraybuffer"                                        │
│     })                                                                  │
└──────────────────────────────────────┬───────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────┐
│ 【层级 4】网络传输（本地回环）                                          │
│  POST http://localhost:9119/execute                                     │
│  请求头：Authorization + X-Hopp-Nonce（均明文）                          │
│  请求体：AES-GCM 加密二进制数据                                         │
└──────────────────────────────────────┬───────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────┐
│ 【层级 5】Agent 服务端解密与执行                                        │
│  controller.rs:204-255 execute()                                        │
│                                                                          │
│  ├─ 1. extract_bearer_token() → 从 Authorization 头提取 auth_key        │
│  ├─ 2. state.validate_access_and_get_data():                            │
│  │   a. 用 auth_key 查找注册信息 → shared_secret_b16                    │
│  │   b. 从 X-Hopp-Nonce 头提取 nonce_b16                                │
│  │   c. AES-256-GCM 解密请求体                                          │
│  │   d. JSON 反序列化为 relay::Request                                  │
│  │                                                                      │
│  ├─ 3. relay::execute(request) → 调用 Rust relay crate                 │
│  │   (基于 libcurl，支持 HTTP/2、代理、证书、认证等)                     │
│  │   ↓                                                                  │
│  │   真实 HTTP 请求到目标服务器                                          │
│  │   ↓                                                                  │
│  │   目标服务器返回响应                                                  │
│  │                                                                      │
│  └─ 4. 🔒 加密响应 (controller.rs:249-255)                              │
│     EncryptedJson {                                                     │
│       key_b16: reg_info.shared_secret_b16,                              │
│       data: response                                                    │
│     }                                                                   │
│     → 自动 AES-256-GCM 加密                                             │
│     → 响应头：                                                          │
│        Content-Type: application/octet-stream                           │
│        X-Hopp-Nonce: <response_nonce_b16>                               │
└──────────────────────────────────────┬───────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────┐
│ 【层级 6】网络传输（本地回环）                                          │
│  Agent 返回加密响应给前端                                               │
│  响应头：Content-Type + X-Hopp-Nonce（均明文）                          │
│  响应体：AES-GCM 加密二进制数据                                         │
└──────────────────────────────────────┬───────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────┐
│ 【层级 7】Agent 拦截器响应解密与处理                                    │
│  回到 AgentKernelInterceptorService (agent/index.ts:179-222)            │
│                                                                          │
│  ├─ 🔓 响应解密 (agent/index.ts:179-183)                                │
│  │   1. 读取响应头 X-Hopp-Nonce → responseNonceB16                      │
│  │   2. store.decryptResponse(responseNonceB16, response.data):         │
│  │      - AES-256-GCM 解密 → JSON 解析                                  │
│  │                                                                      │
│  ├─ 🍪 Set-Cookie 特殊处理 (agent/index.ts:191-207)                     │
│  │   3. 遍历 decryptedResponse.headers:                                 │
│  │      - 如果 key.toLowerCase() === "set-cookie":                      │
│  │        按 \n 分割值 → 每个作为独立 Set-Cookie 条目                    │
│  │        存入 multiHeaders 数组                                         │
│  │      - 其他头:                                                       │
│  │        直接存入 multiHeaders                                         │
│  │                                                                      │
│  └─ 4. body.body() 包装 body 数据                                       │
│     最终 RelayResponse:                                                 │
│     {                                                                   │
│       status: 200,                        ← 目标服务器状态码            │
│       headers: { ... },                   ← 常规 headers（向后兼容）    │
│       multiHeaders: [{key, value}, ...], ← 🔴 保留 Set-Cookie 完整性   │
│       body: { body: Uint8Array, mediaType }                             │
│     }                                                                   │
└──────────────────────────────────────┬───────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────┐
│ 【层级 8】响应适配层                                                   │
│  convertRelayResponseToSerializableResponse()                           │
│  (services/network/rest/rest.ts:427-469)                                │
│                                                                          │
│  ├─ 🔴 优先使用 multiHeaders！(rest.ts:433-444)                         │
│  │   1. 如果有 multiHeaders:                                            │
│  │      - 遍历提取 Set-Cookie → setCookieHeaders 数组                   │
│  │      - 其他头存入 headersObj（去重）                                 │
│  │   2. 如果没有 multiHeaders，从 headers 提取                           │
│  │                                                                      │
│  ├─ 2. 转换 body: Array → Uint8Array                                    │
│  │                                                                      │
│  └─ 3. 实现 Response 接口方法:                                          │
│     text(), json(), arrayBuffer(), blob(), formData()                   │
└──────────────────────────────────────┬───────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────┐
│ 【层级 9】应用层与 UI                                                  │
│  REST 服务 / Pre-request Script / Tests 处理响应                         │
│  ↓                                                                      │
│  UI 渲染：状态码、响应时间、Header 列表、Body 内容、Cookies 等           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 四、路径对比与核心差异

### 4.1 Proxy vs Agent 执行路径对比

| 维度 | Proxy 拦截器（远端中继） | Agent 拦截器（本地执行） |
|-----|-------------------------|-------------------------|
| 是否经过 Relay.execute | ✅ 是（嵌套调用） | ❌ 否（直接 axios） |
| 目标地址 | 改写为 `proxyUrl` | 改写为 `http://localhost:9119/execute` |
| 请求方法 | 强制 POST | 强制 POST |
| 请求体包装 | JSON 明文（ProxyRequest） | AES-256-GCM 加密二进制 |
| 认证方式 | `accessToken` 在请求体内 | `Bearer <auth_key>` 在请求头（明文） |
| 响应解析 | 双层解析：先解析到代理的响应，再解析 ProxyResponse | 单层：解密后直接使用 |
| 状态码替换 | 需要从 ProxyResponse.status 替换 | 直接使用解密后的 status |
| Set-Cookie 处理 | 无特殊处理（逗号分割可能丢失） | 按 `\n` 分割存入 multiHeaders，保留完整性 |
| 实际执行位置 | 远端代理服务器 | 本地 Agent Rust relay crate |
| 延迟 | 较高（公网往返） | 极低（本地回环） |
| CORS 限制 | 无 | 无 |

### 4.2 关键代码位置索引

| 模块 | 文件路径 | 行号 |
|------|---------|------|
| 前端 OTP 生成（被忽略） | `agent/store.ts` | 216 |
| registrationOTP 初始化（空字符串） | `agent/store.ts` | 60 |
| Agent OTP 生成（实际使用） | `controller.rs` | 57 |
| OTP 校验比较 | `state.rs` | 152 |
| verify_registration 明文返回 | `controller.rs` | 177-181 |
| Proxy 拦截器地址改写 | `proxy/index.ts` | 259-273 |
| Proxy 嵌套调用 Relay.execute | `proxy/index.ts` | 275 |
| Proxy 响应解析与状态码替换 | `proxy/index.ts` | 346-434 |
| Agent 直接 axios 调用 | `agent/index.ts` | 165-177 |
| Agent 响应解密 | `agent/index.ts` | 179-183 |
| Agent multiHeaders 处理 | `agent/index.ts` | 191-207 |
| Agent 服务端解密执行 | `controller.rs` | 204-255 |
| Agent 服务端加密响应 | `util.rs` | 64-93 |
| Web 内核 axios 实现 | `hoppscotch-kernel/src/relay/impl/web/v/1.ts` | 113-288 |
| 响应适配（优先 multiHeaders） | `rest.ts` | 427-469 |

---

## 五、核心发现总结

### 5.1 OTP 接力的真相

1. **前端生成的 OTP 完全被忽略**：Agent `receive_registration()` 不读取请求体，自己重新生成 OTP
2. **用户输入的是 Agent 生成的 OTP**：Agent 通过 Tauri 事件弹窗显示自己生成的 OTP，用户手动输入
3. **OTP 是单因素校验**：只比较值是否相等，不校验谁生成的
4. **安全设计**：OTP 由 Agent 生成确保不可预测性，防止前端篡改

### 5.2 明文/加密边界的真相

1. **`auth_key` 永远明文传输**：在 `Authorization` 请求头中明文传递
2. **注册阶段完全明文**：包括 `auth_key` 的返回都是明文 JSON
3. **加密只针对 Body**：HTTP 方法、路径、头永远明文，只有请求体/响应体可能加密
4. **`nonce` 永远明文**：在 `X-Hopp-Nonce` 头中明文传递，用于 AES-GCM 解密
5. **管理接口大多明文**：`/cancel`、`/registrations` 删除、`/log-sink` 响应都是明文

### 5.3 回传路径的真相

1. **Proxy 是双层嵌套调用**：Proxy interceptor → Relay.execute → 代理服务器 → 目标
2. **Proxy 需要双层状态码解析**：外层是到代理服务器的状态码，内层 ProxyResponse 才是目标的真实状态码
3. **Agent 是单层直连**：Agent interceptor 直接 axios 到 localhost:9119，不经过 Relay
4. **Set-Cookie 完整性**：Agent 通过 `multiHeaders` 按 `\n` 分割保留多个 Set-Cookie，避免逗号分割导致的 Cookie 值损坏
5. **适配层优先级**：`convertRelayResponseToSerializableResponse()` 优先使用 `multiHeaders`，只有没有时才用 `headers`

### 5.4 加密通信协议的真相

1. **密钥交换**：x25519 椭圆曲线 Diffie-Hellman，明文交换公钥
2. **对称加密**：AES-256-GCM，12 字节随机 nonce，每次请求独立
3. **密钥存储**：前端存储 `sharedSecretB16`，Agent 存储 `shared_secret_b16`
4. **加密标识**：`Content-Type: application/octet-stream` + `X-Hopp-Nonce` 头
