# 本地 Agent 注册、目标地址改写与响应回传的接力关系分析

## 系统架构概览

Hoppscotch 通过**内核拦截器（Kernel Interceptor）**架构实现跨平台的请求转发，支持 5 种拦截器类型：

| 拦截器类型 | 用途 | 执行位置 |
|-----------|------|---------|
| `browser` | 浏览器原生 fetch | 前端浏览器 |
| `proxy` | 远端代理中继转发 | `https://proxy.hoppscotch.io/` |
| `agent` | 本地桌面 Agent | `http://localhost:9119` |
| `native` | 桌面端原生 | Tauri 后端 |
| `extension` | 浏览器扩展 | 浏览器扩展程序 |

请求流向核心路径：
```
用户发起请求 → KernelInterceptorService → 当前激活拦截器 → 实际执行请求 → 响应原路返回
```

---

## 一、本地 Agent 注册流程

### 1.1 注册时序

```
前端 Web App                          本地 Agent (localhost:9119)
     |                                        |
     | 1. GET /handshake                      | 检查 Agent 是否运行
     | -------------------------------------> |
     |                                        |
     | 2. POST /receive-registration          | 生成 6 位 OTP
     |    { registration: otp }               |
     | -------------------------------------> |
     |                                        | 3. 弹窗显示 OTP 给用户
     |                                        |    (agent/src/pages/otp.vue)
     |                                        |
     | 4. 用户输入 OTP 后                     |
     |    POST /verify-registration           | 验证 OTP + ECDH 密钥交换
     |    {                                   |
     |      registration: otp,                |
     |      client_public_key_b16: ...        |
     |    }                                   |
     | -------------------------------------> |
     |                                        |
     | 5. 返回 auth_key + agent_public_key    |
     | <------------------------------------- |
     |                                        |
     | 6. 双方计算共享密钥 (x25519)           |
     |    保存 auth_key + sharedSecret        |
```

### 1.2 核心代码分析

**前端注册入口** (`agent/store.ts:215-258`):
- `initiateRegistration()`: 发起注册，发送随机 OTP 到 `/receive-registration`
- `verifyRegistration(otp)`: 
  - 生成客户端 ECDH 密钥对 (`x25519.getPublicKey`)
  - 发送公钥到 `/verify-registration`
  - 接收服务端公钥，计算共享密钥 (`x25519.getSharedSecret`)
  - 保存 `auth_key` 和 `sharedSecretB16` 到本地存储

**Agent 服务端注册处理** (`controller.rs:53-182`):
- `receive_registration()`: 生成 6 位 OTP，存储到 `active_registration_code`，通过 Tauri 事件弹窗显示
- `verify_registration()`:
  - 验证 OTP 有效性 (`state.validate_registration`)
  - 生成 UUID 作为 `auth_key`
  - 生成服务端 ECDH 密钥对
  - 用客户端公钥计算共享密钥 (`EphemeralSecret::diffie_hellman`)
  - 存储 `auth_key` → `Registration { registered_at, shared_secret_b16 }`

**加密密钥交换机制**:
- 使用 x25519 椭圆曲线 Diffie-Hellman 密钥交换
- 共享密钥用于后续 AES-256-GCM 加密通信
- 所有请求/响应体均加密，`X-Hopp-Nonce` 头携带加密随机数

---

## 二、目标地址改写逻辑

### 2.1 Proxy 拦截器地址改写

**代码位置**: `proxy/index.ts:205-273`

改写流程：
```
原始请求:
  GET https://api.example.com/data
  Headers: { Authorization: "Bearer xxx" }

↓ 经过 constructProxyRequest() 封装

ProxyRequest: {
  url: "https://api.example.com/data",    // 原始目标地址
  method: "GET",
  headers: { Authorization: "Bearer xxx" },
  data: "",
  accessToken: "...",
  wantsBinary: true
}

↓ 重新封装为发送给代理服务器的请求

改写后的实际请求:
  POST https://proxy.hoppscotch.io/       ← 目标地址改写为代理服务器
  Headers: { "content-type": "application/json" }
  Body: JSON.stringify(ProxyRequest)      ← 原始请求作为 Body 携带
```

**关键改写点** (`proxy/index.ts:259-273`):
```typescript
const proxyRelayRequest: RelayRequest = {
  id: Date.now(),
  url: proxyUrl,                          // 目标改写为代理服务器地址
  method: "POST" as Method,               // 方法强制改为 POST
  version: "HTTP/1.1" as Version,
  headers: {
    "content-type": content.mediaType,    // Body 类型标识
  },
  content,                                // 封装后的原始请求
}
```

### 2.2 Agent 拦截器地址改写

**代码位置**: `agent/index.ts:81-177`

改写流程：
```
原始请求:
  GET https://api.example.com/data
  Headers: { Authorization: "Bearer xxx" }

↓ 经过 preProcessRelayRequest + completeRequest
  添加 domain settings (proxy, security, options)
  添加 Cookie 头
  添加 User-Agent 头

↓ 经过 relayRequestToNativeAdapter 转换为原生格式
↓ AES-256-GCM 加密请求体

改写后的实际请求:
  POST http://localhost:9119/execute      ← 目标地址改写为本地 Agent
  Headers: {
    Authorization: "Bearer <auth_key>",   ← 注册时获得的 auth_key
    X-Hopp-Nonce: "<nonce_b16>",          ← 加密随机数
    Content-Type: "application/octet-stream"
  }
  Body: <encrypted_request>               ← 加密后的原始请求
```

**关键改写点** (`agent/index.ts:165-177`):
```typescript
const response = await axios.post(
  "http://localhost:9119/execute",        // 目标改写为本地 Agent
  encryptedReq,                           // 加密后的请求体
  {
    headers: {
      Authorization: `Bearer ${this.store.authKey.value}`,
      "X-Hopp-Nonce": nonceB16,
      "Content-Type": "application/octet-stream",
    },
    responseType: "arraybuffer",          // 响应也是加密二进制
  }
)
```

### 2.3 Agent 服务端请求解密与执行

**代码位置**: `controller.rs:204-255`

```
接收加密请求:
  POST /execute
  Authorization: Bearer <auth_key>
  X-Hopp-Nonce: <nonce_b16>
  Body: <encrypted_data>

↓ state.validate_access_and_get_data()
  1. 用 auth_key 查找注册信息获取共享密钥
  2. AES-256-GCM 解密 Body (nonce + key)
  3. JSON 反序列化为 relay::Request

↓ relay::execute(request)
  调用底层 Rust relay crate 实际执行 HTTP 请求
  (基于 libcurl，支持 HTTP/2、代理、证书、认证等)

↓ 加密响应
  EncryptedJson {
    key_b16: reg_info.shared_secret_b16,
    data: response
  }
  → 自动 AES-256-GCM 加密，添加 X-Hopp-Nonce 响应头
```

---

## 三、响应回传的接力关系

### 3.1 Proxy 拦截器响应回传链

```
远端目标服务器
      |
      | 原始响应: { status: 200, body: ... }
      ▼
Proxy 服务器 (proxy.hoppscotch.io)
      |
      | 包装为 ProxyResponse:
      | {
      |   success: true,
      |   status: 200,
      |   data: "<base64_or_string>",
      |   isBinary: true,
      |   headers: {...}
      | }
      |
      ▼
前端 proxy/index.ts:346-434
      |
      | 1. parseBytesToJSON 解析 ProxyResponse
      | 2. 判断 isBinary:
      |    - true: base64 解码 → Uint8Array
      |    - false: 直接使用文本数据
      | 3. 重新包装为 RelayResponse:
      |    {
      |      status: parsedProxyResponse.status,
      |      headers: parsedProxyResponse.headers,
      |      body: {
      |        body: <Uint8Array>,
      |        mediaType: "..."
      |      }
      |    }
      |
      ▼
调用方 (hopp-fetch.ts / REST 服务等)
      |
      | convertRelayResponseToSerializableResponse()
      | 转换为可序列化的 Response-like 对象
      ▼
用户代码 / UI 展示
```

**关键代码** (`proxy/index.ts:387-421`):
```typescript
if (parsedProxyResponse.isBinary) {
  const decodedData = new Uint8Array(
    decodeB64StringToArrayBuffer(parsedProxyResponse.data)
  )
  // 尝试解析为 JSON，否则作为二进制返回
  const jsonResult = parseBytesToJSON(decodedData)
  if (O.isSome(jsonResult)) {
    return E.right({
      ...res,
      body: {
        body: new TextEncoder().encode(JSON.stringify(jsonResult.value)),
        mediaType: "application/json",
      },
    })
  }
}
```

### 3.2 Agent 拦截器响应回传链

```
远端目标服务器
      |
      | 原始 HTTP 响应
      ▼
本地 Agent (Rust relay crate)
      |
      | relay::execute() 执行请求，返回 Response
      |
      ▼
Agent controller.rs:249-255
      |
      | EncryptedJson 包装:
      | - AES-256-GCM 加密响应数据
      | - X-Hopp-Nonce 头携带解密 nonce
      | - Content-Type: application/octet-stream
      |
      ▼
前端 agent/index.ts:179-215
      |
      | 1. 读取响应头 X-Hopp-Nonce
      | 2. AES-256-GCM 解密响应体 (sharedSecret + nonce)
      | 3. 处理 Set-Cookie 头到 multiHeaders
      |    (按 \n 分割，每个单独作为 Set-Cookie 条目)
      | 4. body.body() 包装 body 数据
      |
      ▼
调用方 (hopp-fetch.ts / REST 服务等)
      |
      | convertRelayResponseToSerializableResponse()
      | 优先使用 multiHeaders（保留 Set-Cookie 完整性）
      ▼
用户代码 / UI 展示
```

**关键解密代码** (`agent/index.ts:179-183`):
```typescript
const responseNonceB16 = response.headers["x-hopp-nonce"]
const decryptedResponse = await this.store.decryptResponse(
  responseNonceB16,
  response.data
)
```

**Set-Cookie 特殊处理** (`agent/index.ts:191-207`):
```typescript
if (key.toLowerCase() === "set-cookie") {
  const cookieStrings = value
    .split("\n")
    .map((s) => s.trim())
    .filter(Boolean)
  for (const cookieString of cookieStrings) {
    multiHeaders.push({ key: "Set-Cookie", value: cookieString })
  }
}
```

---

## 四、加密通信协议细节

### 4.1 请求加密流程 (`store.ts:289-317`)
```typescript
async encryptRequest(request, reqID): Promise<[nonceB16, encrypted]> {
  1. JSON.stringify 序列化请求
  2. TextEncoder 编码为 Uint8Array
  3. 生成 12 字节随机 nonce
  4. 导入共享密钥为 AES-GCM 密钥
  5. AES-GCM 加密: encrypt({ name: "AES-GCM", iv: nonce }, key, data)
  6. base16 编码 nonce
  7. 返回 [nonceB16, encryptedBuffer]
}
```

### 4.2 响应加密流程 (`util.rs:64-93`)
```rust
impl<T> IntoResponse for EncryptedJson<T> {
  fn into_response(self) -> Response {
    1. serde_json::to_vec 序列化响应
    2. base16 解码共享密钥
    3. Aes256Gcm::generate_nonce 生成 12 字节 nonce
    4. AES-256-GCM 加密
    5. 响应头:
       - Content-Type: application/octet-stream
       - X-Hopp-Nonce: <base16(nonce)>
    6. Body: 加密后的二进制数据
  }
}
```

---

## 五、关键接口汇总

### 5.1 Agent HTTP 接口 (`route.rs:10-33`)

| 方法 | 路径 | 用途 |
|------|------|------|
| GET | `/handshake` | 检查 Agent 是否运行 |
| POST | `/receive-registration` | 发起注册，生成 OTP |
| POST | `/verify-registration` | 验证 OTP + 密钥交换 |
| GET | `/registered-handshake` | 已注册客户端心跳检查 |
| GET | `/registration` | 获取注册信息（加密返回） |
| DELETE | `/registrations/:auth_key` | 删除注册 |
| POST | `/execute` | 执行请求（加密通信） |
| POST | `/cancel/:req_id` | 取消请求 |
| POST | `/log-sink` | 日志收集 |

### 5.2 核心类型定义

**RelayRequest** (`@hoppscotch/kernel`):
```typescript
{
  id: number
  url: string
  method: Method
  version: Version
  headers: Record<string, string>
  params?: Record<string, string>
  auth: AuthType
  content?: ContentType
  proxy?: ProxySettings
  security?: SecuritySettings
}
```

**ProxyRequest** (`proxy/index.ts:28-40`):
```typescript
{
  url: string
  method: string
  headers: Record<string, string>
  params: Record<string, string>
  data: string
  wantsBinary: boolean
  accessToken: string
  auth?: { username: string; password: string }
}
```

---

## 六、请求流向对比

| 拦截器 | 目标地址 | 请求方法 | 数据加密 | 执行位置 |
|-------|---------|---------|---------|---------|
| browser | 原始 URL | 原始方法 | 无 | 浏览器 |
| proxy | `proxyUrl` | 强制 POST | 无（HTTPS 传输加密） | 远端代理服务器 |
| agent | `http://localhost:9119/execute` | 强制 POST | AES-256-GCM 端到端加密 | 本地 Agent |
| native | 内核直接调用 | - | 无 | 桌面端原生 |
| extension | 扩展通信 | - | 无 | 浏览器扩展 |

---

## 七、代码路径索引

| 模块 | 文件路径 |
|------|---------|
| 内核拦截器服务 | `hoppscotch-common/src/services/kernel-interceptor.service.ts` |
| Proxy 拦截器 | `hoppscotch-common/src/platform/std/kernel-interceptors/proxy/index.ts` |
| Agent 拦截器 | `hoppscotch-common/src/platform/std/kernel-interceptors/agent/index.ts` |
| Agent Store | `hoppscotch-common/src/platform/std/kernel-interceptors/agent/store.ts` |
| Agent 服务端路由 | `hoppscotch-agent/src-tauri/src/route.rs` |
| Agent 控制器 | `hoppscotch-agent/src-tauri/src/controller.rs` |
| Agent 状态管理 | `hoppscotch-agent/src-tauri/src/state.rs` |
| Agent 加密工具 | `hoppscotch-agent/src-tauri/src/util.rs` |
| Agent 服务启动 | `hoppscotch-agent/src-tauri/src/server.rs` |
| hopp-fetch 适配 | `hoppscotch-common/src/helpers/hopp-fetch.ts` |
| Kernel Relay 封装 | `hoppscotch-common/src/kernel/relay.ts` |
