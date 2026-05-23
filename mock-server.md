# Mock 服务端请求匹配与响应合成实现分析

## 一、整体架构概述

Mock 服务端采用 NestJS + Prisma 架构，核心流程如下：

```
用户请求 → MockRequestGuard（提取 mock server ID，挂载 mockServer 对象）
        → MockServerLoggingInterceptor（记录开始时间）
        → MockServerController.handleMockRequest（处理请求）
          │
          ├─ 调用 mockServerService.handleMockRequest()
          │   ├─ 匹配示例
          │   └─ 返回 MockServerResponse（含 statusCode/body/headers/delay）
          │
          ├─ 设置响应头
          ├─ 执行延迟（读取 mockServer.delayInMs）
          └─ 发送响应
```

**核心文件**：
- `packages/hoppscotch-backend/src/mock-server/mock-server.service.ts` - 核心匹配逻辑
- `packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts` - 请求入口
- `packages/hoppscotch-backend/src/mock-server/mock-request.guard.ts` - 路由解析与对象挂载
- `packages/hoppscotch-backend/src/mock-server/mock-server-logging.interceptor.ts` - 响应时间统计
- `packages/hoppscotch-backend/src/mock-server/mock-server.model.ts` - 数据模型

---

## 二、数据模型

### 2.1 MockServer 实体（数据库表）

**表定义**（`schema.prisma:246-265`）：

```prisma
model MockServer {
  id              String    @id @default(cuid())
  name            String
  subdomain       String    @unique      // 用于路由匹配
  collectionID    String                  // 关联的集合ID
  workspaceType   WorkspaceType           // USER / TEAM
  delayInMs       Int       @default(0)   // 全局延迟配置（唯一延迟来源）
  isPublic        Boolean   @default(true)
  isActive        Boolean   @default(true)
  hitCount        Int       @default(0)
  // ...
}
```

### 2.2 Mock Examples 存储结构

示例数据存储在 `UserRequest` / `TeamRequest` 表的 `mockExamples` Json 字段中（`schema.prisma:65,186`）。

**注意**：这与 `UserRequest.responses` 字段是完全不同的两个字段：
- `request` - 请求定义（method/endpoint/params 等）
- `responses` - 用户保存的响应历史/模板（传统 Hoppscotch 格式）
- `mockExamples` - Mock 服务专用的示例数据（`{ examples: [...] }` 格式）

`mockExamples` 的实际结构：

```json
{
  "examples": [
    {
      "key": "example1",           // 示例唯一标识（可选）
      "name": "Success Response",  // 示例名称
      "method": "GET",             // HTTP 方法
      "endpoint": "http://api.example.com/v2/pet/<<petId>>",  // 端点URL
      "statusCode": 200,           // 响应状态码
      "statusText": "OK",          // 状态文本
      "responseBody": "{\"id\": 1, \"name\": \"doggie\"}",  // 响应体
      "responseHeaders": [{"key": "content-type", "value": "application/json"}],
      "headers": []                // 请求头（匹配用）
      // ❗ 注意：mockExamples 中没有 delay 字段
    }
  ]
}
```

### 2.3 MockServerResponse 模型（返回值）

`mock-server.model.ts:208-232`：

```typescript
@ObjectType()
export class MockServerResponse {
  statusCode: number;
  body?: string;
  headers?: string;      // JSON 字符串格式的响应头
  delay: number;         // ⚠️ 此字段已赋值但当前未被实际使用
}
```

---

## 三、路由解析与 Mock Server 定位

### 3.1 双路由模式支持

MockRequestGuard 支持两种访问模式（`mock-request.guard.ts:97-189`）：

**模式1：子域名模式**
- URL: `https://{subdomain}.mock.hopp.io/product`
- 解析：从 Host 头提取，如 `abc123.mock.hopp.io` → subdomain = `abc123`

**模式2：路径模式**
- URL: `https://backend.hopp.io/mock/{subdomain}/product`
- 解析：正则匹配 `/^\/mock\/([^\/]+)/` 提取 subdomain

### 3.2 Guard 挂载对象

`MockRequestGuard.canActivate()` 验证通过后，会将以下对象挂载到 request 上供后续使用：

```typescript
(request as any).mockServer = mockServer;       // 完整的数据库对象，含 delayInMs
(request as any).mockServerId = mockServer.id;
(request as any).isSubdomainAccess = true;      // 或 false
```

### 3.3 路径清洗

`MockRequestGuard.getCleanPath()`（`mock-request.guard.ts:258-275`）移除路由前缀，获取纯净端点路径：

| 原始路径 | subdomain | 清洗后路径 |
|---------|-----------|-----------|
| `/mock/abc123/users` | `abc123` | `/users` |
| `/mock/product`（子域名模式，Caddy 重写后）| `abc123` | `/product` |

---

## 四、路径变量解析与匹配

### 4.1 路径变量语法

Hoppscotch 使用 `<<variableName>>` 语法定义路径变量，例如：
- `/v2/pet/<<petId>>` 可匹配 `/v2/pet/123`
- `/organizations/<<orgId>>/teams/<<teamId>>` 支持多变量

**⚠️ 重要说明**：路径变量仅作为通配符进行匹配，不会提取变量值注入到响应体中。也就是说，请求 `/v2/pet/123` 匹配到 `/v2/pet/<<petId>>` 时，响应体中仍然是用户定义的原始内容，不会自动把 `<<petId>>` 替换成 `123`。

### 4.2 端点解析流程

`parseExample()` 方法（`mock-server.service.ts:1016-1068`）负责解析 endpoint：

1. **变量前缀处理**：如果 endpoint 以 `<<` 开头，截取第一个 `>>` 后的内容（移除环境变量前缀如 `<<baseUrl>>`）
2. **域名移除**：正则 `/^([a-zA-Z0-9-]+\.)+[a-zA-Z]{2,}/` 移除域名
3. **URL 解析**：使用 `new URL(endpoint, 'http://dummy.com')` 解析 path 和 query
4. **路径解码**：`decodeURIComponent(url.pathname)` 保留 `<<variable>>` 语法

**解析后返回的内部格式**（无 delay 字段）：

```typescript
return {
  id: exampleData.key || `${requestId}-${exampleData.name}`,
  name: exampleData.name,
  method: exampleData.method || 'GET',
  endpoint: exampleData.endpoint,
  path,
  queryParams,
  statusCode: exampleData.statusCode || 200,
  statusText: exampleData.statusText || 'OK',
  responseBody: exampleData.responseBody || '',
  responseHeaders: exampleData.responseHeaders || [],
  requestHeaders: exampleData.headers || [],
  // ❗ 无 delay 字段 —— 确认不存在示例级延迟
};
```

### 4.3 快速路径预检查

`couldPathMatch()`（`mock-server.service.ts:929-948`）在评分前快速过滤不可能匹配的示例：

```typescript
private couldPathMatch(examplePath: string, requestPath: string): boolean {
  if (examplePath === requestPath) return true;           // 精确匹配
  
  const exampleParts = examplePath.split('/').filter(Boolean);
  const requestParts = requestPath.split('/').filter(Boolean);
  
  if (exampleParts.length !== requestParts.length) {
    return false;                                         // 路径段数不同
  }
  
  if (examplePath.includes('<<')) {
    return true;                                          // 含变量，需完整评分
  }
  
  return false;                                           // 无变量且不精确匹配
}
```

### 4.4 匹配评分算法

`calculateMatchScore()`（`mock-server.service.ts:1076-1160`）基于 Postman 算法：

**评分规则**（满分 100）：

| 匹配项 | 得分规则 |
|-------|---------|
| 路径精确匹配 | 100 分 |
| 路径含变量匹配 | 95 分（扣 5 分） |
| 路径段数不匹配 | 0 分 |
| 路径段内容不匹配 | 0 分 |
| 查询参数匹配 | 按匹配百分比加权 |

**查询参数评分公式**：
```
匹配百分比 = (精确匹配数) / (总参数数)
最终得分 = 路径得分 × (匹配百分比 / 100)
```

**变量匹配判断**（`mock-server.service.ts:1100-1104`）：
```typescript
if (examplePart === requestPart ||
    examplePart.startsWith('<<') ||
    examplePart.includes('<<')) {
  continue;  // 匹配成功（仅判断匹配，不提取变量值）
}
```

**Example 接口定义**（`mock-server.service.ts:884-896`）再次确认无 delay 字段：

```typescript
interface Example {
  id: string;
  name: string;
  method: string;
  endpoint: string;
  path: string;
  queryParams: Record<string, string>;
  statusCode: number;
  statusText: string;
  responseBody: string;
  responseHeaders: Array<{ key: string; value: string }>;
  requestHeaders?: Array<{ key: string; value: string }>;
  // ❗ 无 delay 字段
}
```

---

## 五、状态码切换与多响应示例

### 5.1 多响应示例存储

同一端点可定义多个状态码的响应示例。示例是 `mockExamples.examples` 数组中的独立元素，每个元素有自己的 `statusCode`。

```json
{
  "examples": [
    { "name": "success", "statusCode": 200, "responseBody": "{...}" },
    { "name": "bad request", "statusCode": 400, "responseBody": "{...}" },
    { "name": "not found", "statusCode": 404, "responseBody": "{...}" }
  ]
}
```

### 5.2 状态码选择优先级

`handleMockRequest()`（`mock-server.service.ts:698-800`）按以下优先级选择响应：

| 优先级 | 选择方式 | 触发条件 | 代码位置 |
|-------|---------|---------|---------|
| 1 | 精确匹配 ID | 请求头 `x-mock-response-id: example1` | 第726-738行 |
| 2 | 精确匹配名称 | 请求头 `x-mock-response-name: Success Response` | 第727-738行 |
| 3 | 指定状态码过滤 | 请求头 `x-mock-response-code: 404` | 第756-764行 |
| 4 | 最高评分 + 200优先 | 默认行为 | 第767-790行 |

**执行顺序说明**：
1. 先检查 `x-mock-response-id` / `x-mock-response-name`，匹配成功直接返回（跳过所有后续逻辑）
2. 未命中精确匹配时，先获取所有候选示例，再检查 `x-mock-response-code` 进行过滤
3. 如果 `x-mock-response-code` 过滤后无结果，**不会回退**，而是继续使用全部候选示例进行评分
4. 最终选择评分最高的示例，同分优先选 200 状态码

### 5.3 同分处理逻辑

当多个示例评分相同时（`mock-server.service.ts:783-790`）：

```typescript
const highestScore = scoredExamples[0].score;
const topExamples = scoredExamples.filter(s => s.score === highestScore);

// 优先选择 200 状态码，否则取第一个
const selectedExample =
  topExamples.find(s => s.example.statusCode === 200) || topExamples[0];
```

---

## 六、延迟模拟实现（重点修正）

### 6.1 延迟配置层级（实际情况）

**当前代码中仅存在全局延迟配置，不存在示例级别的延迟设置**：

| 层级 | 配置位置 | 范围 | 最大限制 | 实际生效 |
|-----|---------|------|---------|---------|
| 全局 | `MockServer.delayInMs`（数据库字段） | 整个 mock server | 60000ms（1分钟） | ✅ 生效 |
| 示例 | （不存在） | 单个响应 | - | ❌ 不存在 |
| 响应对象 | `MockServerResponse.delay`（模型字段） | 单个响应 | - | ⚠️ 已赋值但未使用 |

### 6.2 延迟完整读取与生效流程

```
┌─────────────────────────────────────────────────────────────┐
│  延迟读取与生效完整流程（按执行顺序）                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. MockRequestGuard                                        │
│     └─ 从数据库读取 MockServer 对象（含 delayInMs）          │
│        └─ 挂载到 request.mockServer                        │
│           │  mock-request.guard.ts:79-82                   │
│                                                             │
│  2. MockServerService.handleMockRequest                     │
│     ├─ 匹配到最佳示例后                                     │
│     └─ 调用 formatExampleResponse(example, mockServer.delayInMs)
│        ├─ 第737行（精确匹配路径）                          │
│        └─ 第794行（评分匹配路径）                          │
│           │                                                │
│           └─ formatExampleResponse() 赋值 delay 字段        │
│              mock-server.service.ts:1183                   │
│              return {                                      │
│                ...                                         │
│                delay: delayInMs || 0  // ← 仅赋值，未使用  │
│              }                                             │
│                                                             │
│  3. MockServerController.handleMockRequest                  │
│     ├─ 收到 MockServerResponse（含 delay 字段）              │
│     ├─ ❗ 不读取 mockResponse.delay                         │
│     └─ 直接读取 request.mockServer.delayInMs 执行延迟       │
│        mock-server.controller.ts:136-140                   │
│        if (mockServer.delayInMs && mockServer.delayInMs > 0) {
│          await new Promise(resolve =>                      │
│            setTimeout(resolve, mockServer.delayInMs)        │
│          );                                                │
│        }                                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6.3 关键代码分析

**延迟实际生效位置**（`mock-server.controller.ts:135-140`）：

```typescript
// Add delay if specified
// ⚠️ 注意：这里读取的是 mockServer.delayInMs（guard 挂载的对象）
//    而不是 mockResponse.delay（service 返回的字段）
if (mockServer.delayInMs && mockServer.delayInMs > 0) {
  await new Promise((resolve) =>
    setTimeout(resolve, mockServer.delayInMs),
  );
}
```

**延迟参数传递（未生效路径）**（`mock-server.service.ts:1165-1185`）：

```typescript
private formatExampleResponse(
  example: any,
  delayInMs: number,  // 传入全局延迟
): E.Either<string, MockServerResponse> {
  // ... 处理 headers ...
  
  return E.right({
    statusCode: example.statusCode || 200,
    body: example.responseBody || '',
    headers: JSON.stringify(headersObj),
    delay: delayInMs || 0,  // ⚠️ 赋值但 controller 不读取此字段
  });
}
```

**测试用例验证**（`mock-server.service.spec.ts:1322-1350`）：

```typescript
test('should include delay in response', async () => {
  const delayedMockServer = { ...dbMockServer, delayInMs: 500 };
  // ...
  const result = await mockServerService.handleMockRequest(
    delayedMockServer, '/users', 'GET'
  );
  
  // 测试仅验证 service 返回的 delay 字段值正确
  // 但未验证 controller 实际使用此字段
  expect((result.right as any).delay).toBe(500);
});
```

### 6.4 关于示例级延迟的结论

**结论：当前代码中不存在示例级别的延迟设置。**

证据：
1. `mockExamples` 数据结构中无 delay 字段（测试用例、数据库 schema 均无）
2. `parseExample()` 返回对象无 delay 字段（第1051-1063行）
3. `Example` 接口定义无 delay 字段（第884-896行）
4. controller 实际执行延迟时读取的是全局的 `mockServer.delayInMs`

**`MockServerResponse.delay` 字段现状**：该字段虽然在模型中定义、service 中也赋值了，但 controller 并未读取它来执行延迟。这是一个"已定义但未实际使用"的字段，可能是为未来扩展示例级延迟预留的接口。

### 6.5 响应时间统计（与延迟的关系）

`MockServerLoggingInterceptor`（`mock-server-logging.interceptor.ts:23,56`）统计的 `responseTime` 包含了延迟时间：

```typescript
const startTime = Date.now();  // interceptor 开始时记录
// ... 经过 guard、controller、延迟 ...
const responseTime = Date.now() - startTime;  // 包含 delayInMs
```

即：用户看到的请求响应时间 = 实际处理时间 + `mockServer.delayInMs`。

---

## 七、示例数据合成响应

### 7.1 响应头处理

**安全头黑名单**（`mock-server.controller.ts:19-25`）防止 XSS 攻击：
```typescript
const SECURITY_HEADER_BLOCKLIST = new Set([
  'content-security-policy',
  'x-content-type-options',
  'x-frame-options',
  'content-disposition',
  'set-cookie',
]);
```

**内容类型降级**（路径模式下，`mock-server.controller.ts:145-154`）：
```typescript
const ACTIVE_CONTENT_TYPES = new Set([
  'application/javascript', 'text/html', 'image/svg+xml', // ...
]);
if (!isSubdomainAccess && ACTIVE_CONTENT_TYPES.has(mimeType)) {
  res.setHeader('Content-Type', 'text/plain');  // 降级防止 XSS
}
```

### 7.2 响应格式自动检测

`MockServerController.handleMockRequest()`（`mock-server.controller.ts:157-176`）：

```typescript
if (!res.getHeader('Content-Type')) {
  try {
    JSON.parse(mockResponse.body);
    defaultContentType = 'application/json';  // 解析成功则为 JSON
  } catch {
    defaultContentType = 'text/plain';        // 否则为纯文本
  }
  res.setHeader('Content-Type', defaultContentType);
}
```

### 7.3 安全头注入

无论用户是否定义，都会自动注入安全头（`mock-server.controller.ts:177-182`）：
```typescript
res.setHeader('X-Content-Type-Options', 'nosniff');
if (!isSubdomainAccess) {
  res.setHeader('Content-Security-Policy', "default-src 'none'; sandbox");
  res.setHeader('X-Frame-Options', 'DENY');
}
```

### 7.4 关于自动创建示例的说明

`mockServerCollRequestExample()`（`constants/mock-server-coll-request-example.ts`）定义的是**传统 Hoppscotch 请求格式**，使用的是 `responses` 字段而非 `mockExamples` 字段。这个文件的作用是：

1. 当用户勾选"自动创建请求示例"时，导入这个集合作为起点模板
2. 用户需要在 UI 中手动将 `responses` 转换为 `mockExamples` 才能被 mock 服务使用
3. 导入的集合包含 6 个示例请求（见下表），但需要手动启用 mock

| 请求名称 | 方法 | 端点 | 状态码示例 |
|---------|------|------|-----------|
| addPet | POST | `/v2/pet` | 405 |
| updatePet | PUT | `/v2/pet` | 400, 404, 405 |
| findPetsByStatus | GET | `/v2/pet/findByStatus` | 200, 400 |
| getPetById | GET | `/v2/pet/<<petId>>` | 200, 400, 404 |
| updatePetWithForm | POST | `/v2/pet/<<petId>>` | 405 |
| deletePet | DELETE | `/v2/pet/<<petId>>` | 400, 404 |

---

## 八、完整匹配流程（修正版）

```
handleMockRequest(mockServer, path, method, query, headers)
│
├─ 步骤1：获取关联集合 ID（递归获取所有子集合）
│   getCollectionIds(mockServer)
│
├─ 步骤2：获取所有含 mockExamples 的请求（单库查询）
│   fetchRequestsWithExamples(mockServer, collectionIds)
│   条件：where: { mockExamples: { not: null } }
│
├─ 步骤3：检查精确匹配头（最快路径，跳过评分）
│   ├─ x-mock-response-id → findExampleByIdOrName()
│   ├─ x-mock-response-name → findExampleByIdOrName()
│   └─ 匹配成功 → formatExampleResponse(example, mockServer.delayInMs)
│            → 直接返回，不执行后续步骤
│
├─ 步骤4：获取候选示例（内存过滤）
│   fetchCandidateExamples()
│   ├─ 按 method 过滤（示例 method 必须与请求 method 相同）
│   ├─ parseExample() 解析端点
│   └─ 按 couldPathMatch() 快速过滤（段数不同直接排除）
│
├─ 步骤5：状态码头过滤（如果提供）
│   if (x-mock-response-code) {
│     过滤出 statusCode === 指定值的示例
│     （若过滤后为空，则使用全部候选示例继续）
│   }
│
├─ 步骤6：计算匹配分数
│   calculateMatchScore()
│   ├─ 路径匹配评分：精确100 / 含变量95 / 不匹配0
│   ├─ 查询参数加权：精确匹配数 / 总参数数 × 路径得分
│   └─ 过滤掉 score <= 0 的示例
│
├─ 步骤7：选择最优示例
│   ├─ 按分数降序排序
│   ├─ 提取最高分的所有示例
│   └─ 同分优先选择 statusCode === 200 的示例
│
└─ 步骤8：格式化响应（注入全局延迟）
    formatExampleResponse(selectedExample, mockServer.delayInMs)
    ├─ 转换 headers 数组为对象
    └─ 设置 delay = mockServer.delayInMs（⚠️ controller 不使用此字段）
```

---

## 九、关键优化点

1. **单次数据库查询**：`fetchRequestsWithExamples()` 一次性获取所有候选请求，避免 N+1 查询
2. **快速路径预检查**：`couldPathMatch()` 在评分前过滤 80% 不可能匹配的示例
3. **精确匹配快速路径**：通过请求头指定 ID/名称时直接返回，跳过所有评分流程
4. **数据库级过滤**：`where: { mockExamples: { not: null } }` 只获取有示例的请求
5. **集合 ID 复用**：`collectionIds` 在精确匹配和候选匹配中复用，避免重复查询
6. **method 前置过滤**：在解析端点前就过滤掉方法不匹配的示例，减少解析开销

---

## 十、代码引用速查

| 功能 | 文件位置 |
|-----|---------|
| 路由提取与对象挂载 | `mock-request.guard.ts:79-82, 97-189` |
| 端点解析（parseExample） | `mock-server.service.ts:1016-1068` |
| 路径匹配评分 | `mock-server.service.ts:1076-1160` |
| 快速路径预检查 | `mock-server.service.ts:929-948` |
| 状态码选择逻辑 | `mock-server.service.ts:726-790` |
| 延迟实际执行 | `mock-server.controller.ts:136-140` |
| 延迟字段赋值（未使用） | `mock-server.service.ts:1183` |
| 响应合成 | `mock-server.controller.ts:107-193` |
| 主处理流程 | `mock-server.service.ts:698-800` |
| 响应时间统计 | `mock-server-logging.interceptor.ts:23, 56` |
| Example 接口定义 | `mock-server.service.ts:884-896` |
