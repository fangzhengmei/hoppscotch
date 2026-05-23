# Mock 服务端请求匹配与响应合成实现分析

## 一、整体架构概述

Mock 服务端采用 NestJS + Prisma 架构，核心流程如下：

```
用户请求 → MockRequestGuard（提取 mock server ID）
        → MockServerController（处理请求）
        → MockServerService.handleMockRequest（匹配示例+合成响应）
        → 延迟模拟 → 返回响应
```

**核心文件**：
- `packages/hoppscotch-backend/src/mock-server/mock-server.service.ts` - 核心匹配逻辑
- `packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts` - 请求入口
- `packages/hoppscotch-backend/src/mock-server/mock-request.guard.ts` - 路由解析
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
  delayInMs       Int       @default(0)   // 全局延迟
  isPublic        Boolean   @default(true)
  isActive        Boolean   @default(true)
  hitCount        Int       @default(0)
  // ...
}
```

### 2.2 Mock Examples 存储结构

示例数据存储在 `UserRequest` / `TeamRequest` 表的 `mockExamples` Json 字段中（`schema.prisma:65,186`）：

```json
{
  "examples": [
    {
      "key": "example1",           // 示例唯一标识
      "name": "Success Response",  // 示例名称
      "method": "GET",             // HTTP 方法
      "endpoint": "http://api.example.com/v2/pet/<<petId>>",  // 端点URL
      "statusCode": 200,           // 响应状态码
      "statusText": "OK",          // 状态文本
      "responseBody": "{\"id\": 1, \"name\": \"doggie\"}",  // 响应体
      "responseHeaders": [{"key": "content-type", "value": "application/json"}],
      "headers": []                // 请求头
    }
  ]
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

### 3.2 路径清洗

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

### 4.2 端点解析流程

`parseExample()` 方法（`mock-server.service.ts:1016-1068`）负责解析 endpoint：

1. **变量前缀处理**：如果 endpoint 以 `<<` 开头，截取第一个 `>>` 后的内容（移除环境变量前缀如 `<<baseUrl>>`）
2. **域名移除**：正则 `/^([a-zA-Z0-9-]+\.)+[a-zA-Z]{2,}/` 移除域名
3. **URL 解析**：使用 `new URL(endpoint, 'http://dummy.com')` 解析 path 和 query
4. **路径解码**：`decodeURIComponent(url.pathname)` 保留 `<<variable>>` 语法

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
  continue;  // 匹配成功
}
```

---

## 五、状态码切换与多响应示例

### 5.1 多响应示例存储

同一端点可定义多个状态码的响应示例（参考 `mock-server-coll-request-example.ts`）：

```json
{
  "getPetById": {
    "responses": {
      "successful operation": { "code": 200, "body": "..." },
      "Invalid ID supplied":   { "code": 400, "body": "..." },
      "Pet not found":         { "code": 404, "body": "..." }
    }
  }
}
```

### 5.2 状态码选择优先级

`handleMockRequest()`（`mock-server.service.ts:698-800`）按以下优先级选择响应：

| 优先级 | 选择方式 | 触发条件 |
|-------|---------|---------|
| 1 | 精确匹配 ID | 请求头 `x-mock-response-id: example1` |
| 2 | 精确匹配名称 | 请求头 `x-mock-response-name: Success Response` |
| 3 | 指定状态码 | 请求头 `x-mock-response-code: 404` |
| 4 | 最高评分 + 200优先 | 默认行为 |

### 5.3 同分处理逻辑

当多个示例评分相同时（`mock-server.service.ts:783-790`）：

```typescript
const highestScore = scoredExamples[0].score;
const topExamples = scoredExamples.filter(s => s.score === highestScore);

// 优先选择 200 状态码
const selectedExample =
  topExamples.find(s => s.example.statusCode === 200) || topExamples[0];
```

---

## 六、延迟模拟实现

### 6.1 延迟配置层级

延迟支持两个层级配置：

| 层级 | 配置位置 | 范围 | 最大限制 |
|-----|---------|------|---------|
| 全局 | `MockServer.delayInMs` | 整个 mock server | 60000ms（1分钟） |
| 示例 | `MockServerResponse.delay` | 单个响应 | 继承全局 |

### 6.2 延迟执行机制

`MockServerController.handleMockRequest()`（`mock-server.controller.ts:136-140`）：

```typescript
if (mockServer.delayInMs && mockServer.delayInMs > 0) {
  await new Promise((resolve) =>
    setTimeout(resolve, mockServer.delayInMs),
  );
}
```

### 6.3 延迟参数传递

`formatExampleResponse()`（`mock-server.service.ts:1165-1185`）将全局延迟传递给响应对象：

```typescript
return E.right({
  statusCode: example.statusCode || 200,
  body: example.responseBody || '',
  headers: JSON.stringify(headersObj),
  delay: delayInMs || 0,  // 来自 mockServer.delayInMs
});
```

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

---

## 八、完整匹配流程

```
handleMockRequest(mockServer, path, method, query, headers)
│
├─ 步骤1：获取关联集合 ID（递归获取所有子集合）
│   getCollectionIds(mockServer)
│
├─ 步骤2：获取所有含 mockExamples 的请求（单库查询）
│   fetchRequestsWithExamples(mockServer, collectionIds)
│
├─ 步骤3：检查精确匹配头（最快路径）
│   ├─ x-mock-response-id → findExampleByIdOrName()
│   ├─ x-mock-response-name → findExampleByIdOrName()
│   └─ 匹配成功 → 直接返回响应
│
├─ 步骤4：获取候选示例
│   fetchCandidateExamples()
│   ├─ 按 method 过滤
│   └─ 按 couldPathMatch() 快速过滤
│
├─ 步骤5：状态码头过滤
│   x-mock-response-code → 过滤出指定状态码的示例
│
├─ 步骤6：计算匹配分数
│   calculateMatchScore()
│   ├─ 路径匹配评分（100/95/0）
│   └─ 查询参数加权
│
├─ 步骤7：选择最优示例
│   ├─ 按分数降序排序
│   └─ 同分优先选择 200 状态码
│
└─ 步骤8：格式化响应
    formatExampleResponse()
    ├─ 转换 headers 数组为对象
    └─ 注入 delayInMs
```

---

## 九、关键优化点

1. **单次数据库查询**：`fetchRequestsWithExamples()` 一次性获取所有候选请求，避免 N+1 查询
2. **快速路径预检查**：`couldPathMatch()` 在评分前过滤 80% 不可能匹配的示例
3. **精确匹配快速路径**：通过请求头指定 ID/名称时直接返回，跳过评分流程
4. **数据库级过滤**：`where: { mockExamples: { not: null } }` 只获取有示例的请求
5. **集合 ID 复用**：`collectionIds` 在精确匹配和候选匹配中复用，避免重复查询

---

## 十、示例数据结构（自动创建）

`mockServerCollRequestExample()`（`constants/mock-server-coll-request-example.ts:5-781`）定义了默认示例集合，包含：

| 请求名称 | 方法 | 端点 | 状态码示例 |
|---------|------|------|-----------|
| addPet | POST | `/v2/pet` | 405 |
| updatePet | PUT | `/v2/pet` | 400, 404, 405 |
| findPetsByStatus | GET | `/v2/pet/findByStatus` | 200, 400 |
| getPetById | GET | `/v2/pet/<<petId>>` | 200, 400, 404 |
| updatePetWithForm | POST | `/v2/pet/<<petId>>` | 405 |
| deletePet | DELETE | `/v2/pet/<<petId>>` | 400, 404 |

---

## 十一、代码引用速查

| 功能 | 文件位置 |
|-----|---------|
| 路由提取 | `mock-request.guard.ts:97-189` |
| 端点解析 | `mock-server.service.ts:1016-1068` |
| 路径匹配 | `mock-server.service.ts:1076-1160` |
| 快速过滤 | `mock-server.service.ts:929-948` |
| 状态码选择 | `mock-server.service.ts:755-790` |
| 延迟执行 | `mock-server.controller.ts:136-140` |
| 响应合成 | `mock-server.controller.ts:107-193` |
| 示例格式化 | `mock-server.service.ts:1165-1185` |
| 主处理流程 | `mock-server.service.ts:698-800` |
