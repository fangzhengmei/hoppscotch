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

## 七、Mock Examples 数据来源与写入流程（修正版）

### 7.1 字段关系与架构确认（核心修正）

**重要结论：`request` 和 `mockExamples` 是数据库中两个独立的列，`responses` 是 `request` JSON 列内部的内嵌结构。触发器将 `request.responses` 自动同步到 `mockExamples` 独立列。**

#### 架构层级确认

```
数据库表（UserRequest / TeamRequest）
├─ 列1: `id`             (String)
├─ 列2: `collectionID`   (String)
├─ 列3: `request`        (Json)    ← 独立列，内部含 responses
│  └─ {
│       "method": "GET",
│       "endpoint": "/v2/pet/<<petId>>",
│       "responses": {              ← 内嵌结构，不是独立列
│         "successful operation": {
│           "code": 200,
│           "body": "...",
│           "originalRequest": {...}
│         }
│       }
│     }
├─ 列4: `mockExamples`   (Json?)   ← 独立列，由触发器同步生成
│  └─ {
│       "examples": [
│         {"statusCode": 200, "responseBody": "...", ...}
│       ]
│     }
└─ ...
```

#### 字段对比表

| 名称 | 层级 | 类型 | 同步关系 | 被 mock 服务读取 | 代码证据 |
|-----|------|------|---------|----------------|---------|
| `request` | **独立列** | `Json` | **源列** | ❌ | `schema.prisma:185`（UserRequest）<br>`schema.prisma:64`（TeamRequest） |
| `responses` | **request 内部内嵌字段** | - | 数据源（内嵌于 request） | ❌ | `migration.sql:111` `NEW.request->'responses'` |
| `mockExamples` | **独立列** | `Json?` | **目标列，触发器自动同步** | ✅ | `schema.prisma:186`（UserRequest）<br>`schema.prisma:65`（TeamRequest） |

#### 证据 1：数据库 schema 定义（`schema.prisma:180-195`）

```prisma
model UserRequest {
  id             String         @id @default(cuid())
  collectionID   String
  userUid        String
  title          String
  request        Json           // ← 独立列（第 185 行）
  mockExamples   Json?          // ← 独立列（第 186 行）
  type           ReqType
  orderIndex     Int
  // ...
}
```

**TeamRequest 同理**（`schema.prisma:59-73`）：
```prisma
model TeamRequest {
  id           String         @id @default(cuid())
  collectionID String
  teamID       String
  title        String
  request      Json           // ← 独立列（第 64 行）
  mockExamples Json?          // ← 独立列（第 65 行）
  // ...
}
```

#### 证据 2：migration 添加独立列（`migration.sql:82-88`）

```sql
-- Add mockExamples column to UserRequest
ALTER TABLE "UserRequest" 
ADD COLUMN "mockExamples" JSONB;   -- ← 作为独立列添加（第 84 行）

-- Add mockExamples column to TeamRequest
ALTER TABLE "TeamRequest" 
ADD COLUMN "mockExamples" JSONB;   -- ← 作为独立列添加（第 88 行）
```

#### 证据 3：responses 是内嵌结构（`migration.sql:111-112`）

触发器函数从 `NEW.request->'responses'` 读取数据，证明 `responses` 是 `request` JSON 内部的字段：

```sql
FROM jsonb_each(NEW.request->'responses') AS responses(key, value)
WHERE jsonb_typeof(NEW.request->'responses') = 'object'
```

使用 `->` 操作符访问 JSON 内部字段，这是 PostgreSQL 访问 JSON 内嵌属性的标准语法。

---

### 7.2 sync_mock_examples() 自动同步函数（PostgreSQL）

**源码位置**：`migrations/20251016080714_mock_server/migration.sql:91-120`

这是一个 PostgreSQL PL/pgSQL 触发器函数，在每次 INSERT 或 UPDATE `request` 独立列时自动执行，将 `request` 列内部内嵌的 `responses` 结构转换为 `mockExamples` 独立列的格式。

#### 同步函数完整代码

```sql
CREATE OR REPLACE FUNCTION sync_mock_examples()
RETURNS TRIGGER AS $$
BEGIN
  NEW."mockExamples" := jsonb_build_object(                      -- 第 94 行：写入 mockExamples 独立列
    'examples',
    COALESCE(
      (
        SELECT jsonb_agg(
          jsonb_build_object(                                    -- 第 98 行：构造每个示例
            'key', key,                                          -- 第 100 行
            'name', value->>'name',                              -- 第 101 行
            'endpoint', value->'originalRequest'->>'endpoint',   -- 第 102 行
            'method', value->'originalRequest'->>'method',       -- 第 103 行
            'headers', COALESCE(value->'originalRequest'->'headers', '[]'::jsonb),  -- 第 104 行
            'statusCode', (value->>'code')::int,                 -- 第 105 行
            'statusText', value->>'status',                      -- 第 106 行
            'responseBody', value->>'body',                      -- 第 107 行
            'responseHeaders', COALESCE(value->'headers', '[]'::jsonb)  -- 第 108 行
          )
        )
        FROM jsonb_each(NEW.request->'responses') AS responses(key, value)  -- 第 111 行：从 request 内嵌字段读取
        WHERE jsonb_typeof(NEW.request->'responses') = 'object'  -- 第 112 行
      ),
      '[]'::jsonb
    )
  );
  
  RETURN NEW;                                                     -- 第 118 行
END;
$$ LANGUAGE plpgsql;
```

#### 函数逻辑拆解（含代码证据位置）

| 步骤 | 操作 | 源码位置 | 说明 |
|-----|------|---------|------|
| 1 | 构造外层 `{ examples: [...] }` 对象 | `migration.sql:94-95` | `jsonb_build_object('examples', ...)` |
| 2 | **从 `request` 内嵌字段读取** | `migration.sql:111` | `NEW.request->'responses'` 使用 `->` 访问 JSON 内部字段 |
| 3 | 将 responses object 转为行集 | `migration.sql:111` | `jsonb_each(NEW.request->'responses')` |
| 4 | 对每个响应构造 mock 示例格式 | `migration.sql:98-109` | `jsonb_build_object()` 映射 9 个字段 |
| 5 | 聚合成数组 | `migration.sql:98` | `jsonb_agg()` |
| 6 | 处理空值 | `migration.sql:96-115` | `COALESCE(..., '[]'::jsonb)` 无 responses 时返回空数组 |
| 7 | **写入 mockExamples 独立列** | `migration.sql:94` | `NEW."mockExamples" := ...` 赋值给独立列 |
| 8 | 返回 NEW 继续操作 | `migration.sql:118` | `RETURN NEW;` |

#### 字段映射表（含代码证据位置）

| responses 路径（内嵌于 request） | mockExamples 字段（独立列） | 说明 | 源码位置 |
|---------------------------------|-------------------|------|---------|
| `key`（jsonb_each 输出） | `key` | 响应唯一标识 | `migration.sql:100` |
| `value->>'name'` | `name` | 响应名称 | `migration.sql:101` |
| `value->'originalRequest'->>'endpoint'` | `endpoint` | 端点 URL（含 `<<variable>>`） | `migration.sql:102` |
| `value->'originalRequest'->>'method'` | `method` | HTTP 方法 | `migration.sql:103` |
| `value->'originalRequest'->'headers'` | `headers` | **请求头**（用于匹配） | `migration.sql:104` |
| `(value->>'code')::int` | `statusCode` | 响应状态码 | `migration.sql:105` |
| `value->>'status'` | `statusText` | 状态文本 | `migration.sql:106` |
| `value->>'body'` | `responseBody` | 响应体 | `migration.sql:107` |
| `value->'headers'` | `responseHeaders` | **响应头**（用于返回） | `migration.sql:108` |

#### 关键架构证据

**证据 1：从内嵌结构读取**（`migration.sql:111`）
```sql
FROM jsonb_each(NEW.request->'responses') AS responses(key, value)
```
使用 `->` 操作符访问 JSON 内部字段，证明 `responses` 是 `request` JSON 列的内嵌属性，不是独立列。

**证据 2：写入独立列**（`migration.sql:94`）
```sql
NEW."mockExamples" := jsonb_build_object(...)
```
直接赋值给 `NEW."mockExamples"`，这是数据库的独立列名。

**证据 3：同步方向是单向的**
- 仅在 `BEFORE INSERT OR UPDATE OF request` 时触发（见 7.3 节）
- 没有反向触发器将 mockExamples 同步回 request.responses
- 同步方向：`request.responses`（内嵌） → `mockExamples`（独立列）

---

### 7.3 触发器定义与触发时机

**源码位置**：`migrations/20251016080714_mock_server/migration.sql:123-132`

#### UserRequest 触发器（`migration.sql:123-126`）
```sql
CREATE TRIGGER trigger_sync_mock_examples_user_request
BEFORE INSERT OR UPDATE OF request ON "UserRequest"
FOR EACH ROW
EXECUTE FUNCTION sync_mock_examples();
```

#### TeamRequest 触发器（`migration.sql:129-132`）
```sql
CREATE TRIGGER trigger_sync_mock_examples_team_request
BEFORE INSERT OR UPDATE OF request ON "TeamRequest"
FOR EACH ROW
EXECUTE FUNCTION sync_mock_examples();
```

**触发时机详解（含代码证据）**：

| 触发条件 | 说明 | 源码位置 |
|---------|------|---------|
| `BEFORE INSERT` | 插入新请求记录**之前**执行，mockExamples 随记录一起写入 | `migration.sql:124, 130` |
| `BEFORE UPDATE OF request` | 仅当 `request` **独立列**被更新时**之前**执行，其他字段更新不触发 | `migration.sql:124, 130` |
| `FOR EACH ROW` | 每行记录变更都独立执行一次 | `migration.sql:125, 131` |

**触发场景（含代码证据）**：

| 操作 | 是否触发 | 原因 | 代码证据 |
|-----|---------|------|---------|
| 创建新请求（`createRequest`） | ✅ | INSERT 操作触发 `BEFORE INSERT` | `user-request.service.ts:119-175` |
| 更新请求内容（`updateRequest`） | ✅ | 更新 `request` 独立列触发 `BEFORE UPDATE OF request` | `user-request.service.ts:185-220` |
| 导入集合（`importCollectionsFromJSON`） | ✅ | 批量 INSERT 触发 `BEFORE INSERT` | `user-collection.service.ts:1126-1231` |
| 仅更新 request 标题 | ❌ | 未修改 `request` 独立列 | - |
| 仅更新 request 的 orderIndex | ❌ | 未修改 `request` 独立列 | - |
| 直接修改 mockExamples 独立列 | ❌ | 未修改 `request` 独立列（不建议） | - |

**架构确认证据**：

`BEFORE UPDATE OF request` 中的 `request` 是列名，证明：
- `request` 是数据库的独立列（不是内嵌结构）
- 只有当这个独立列被更新时才触发同步
- `responses` 内嵌于 `request` JSON 列中，所以更新 `request.responses` 也会触发触发器

---

### 7.4 autoCreateRequestExample 导入数据映射流程（修正版）

当用户创建 mock server 并勾选 "自动创建请求示例" 时，完整流程如下：

```
用户勾选 autoCreateRequestExample: true
        ↓
mockServerCollRequestExample(input.name)
  └─ 生成传统 Hoppscotch 请求格式（含 responses 字段，嵌套 originalRequest）
     源码位置：constants/mock-server-coll-request-example.ts:5-781
        ↓
importCollectionsFromJSON(jsonString, user.uid, ...)
  源码位置：user-collection.service.ts:1126-1231
        ↓
generatePrismaQueryObj(folder, userID, ...)
  源码位置：user-collection.service.ts:1065-1114
  ├─ 遍历 folder.requests
  └─ 对每个请求 r，创建 Prisma create 数据：
     {
       title: r.name,
       request: r,        // ← 含 responses.originalRequest
       orderIndex: index + 1,
       // ❗ 无需设置 mockExamples，触发器自动处理
     }
        ↓
tx.userCollection.create(...)
  源码位置：user-collection.service.ts:1182-1186
        ↓
🔴 【数据库触发器自动执行】
   trigger_sync_mock_examples_user_request
   BEFORE INSERT ON "UserRequest"
        ↓
sync_mock_examples() 函数执行
   ├─ 读取 NEW.request->'responses'
   ├─ jsonb_each() 展开每个响应
   ├─ 字段映射转换
   ├─ 构造 { examples: [...] }
   └─ 赋值给 NEW.mockExamples
        ↓
记录写入数据库（request + mockExamples 同时写入）
```

**关键代码：请求记录创建**（`user-collection.service.ts:1093-1104`）：
```typescript
requests: {
  create: folder.requests.map((r, index) => ({
    title: r.name,
    user: { connect: { uid: userID } },
    type: reqType,
    request: r,           // ← 仅写入 request，触发器自动同步 mockExamples
    orderIndex: index + 1,
    // ✅ 无需手动设置 mockExamples
  })),
},
```

**导入后各字段状态（已修正）**：
- `request` 字段：包含完整的请求定义，内部有 `responses` 对象（每个响应含 `originalRequest`）
- `mockExamples` 字段：**由触发器自动同步，格式为 `{ examples: [...] }`**
- 结果：**mock 服务可以直接匹配到示例，无需额外转换步骤**

---

### 7.5 Backfill 历史数据处理

**源码位置**：`migrations/20251016080714_mock_server/migration.sql:134-138`

migration 在创建函数和触发器后，执行以下 SQL 回填历史数据：

```sql
-- Backfill existing data for UserRequest
UPDATE "UserRequest" SET request = request WHERE request IS NOT NULL;

-- Backfill existing data for TeamRequest
UPDATE "TeamRequest" SET request = request WHERE request IS NOT NULL;
```

**Backfill 原理**：
- `UPDATE ... SET request = request` 看似是"无操作"，但实际上会触发 `UPDATE OF request` 触发器
- 对所有 `request IS NOT NULL` 的历史记录执行一次"假更新"
- 触发器被触发，为所有历史记录生成 `mockExamples` 字段

**Backfill 影响范围**：

| 场景 | 处理方式 |
|-----|---------|
| migration 执行前已存在的请求 | ✅ 自动回填，生成 mockExamples |
| migration 执行后新增/更新的请求 | ✅ 触发器实时同步 |
| request 为 null 的请求 | ❌ 跳过，mockExamples 保持 null |

**GIN 索引**（`migration.sql:140-142`）：
```sql
CREATE INDEX "idx_mock_examples_user_requests_gin" ON "UserRequest" USING GIN ("mockExamples");
CREATE INDEX "idx_mock_examples_team_requests_gin" ON "TeamRequest" USING GIN ("mockExamples");
```
为 mockExamples 字段创建 GIN 索引，加速 JSONB 查询。

---

### 7.6 mockExamples 写入路径完整分析（修正版）

**结论：后端不需要专门的 mockExamples 写入接口，数据库触发器自动处理所有同步。**

#### 路径 1：后端 createRequest / updateRequest（✅ 自动支持）

**`createRequest`**（`user-request.service.ts:119-175`）：
```typescript
return tx.userRequest.create({
  data: {
    collectionID,
    title,
    request: jsonRequest.right,  // ← 写入 request 字段
    type: ReqType[type],
    orderIndex: lastUserRequest ? lastUserRequest.orderIndex + 1 : 1,
    userUid: user.uid,
    // ✅ 无需设置 mockExamples，触发器自动同步
  },
});
```

**`updateRequest`**（`user-request.service.ts:185-220`）：
```typescript
data: {
  title,
  request: jsonRequest,  // ← 更新 request 字段，触发同步
  // ✅ 无需设置 mockExamples，触发器自动同步
},
```

#### 路径 2：后端导入接口（✅ 自动支持）

**`importCollectionsFromJSON`**（`user-collection.service.ts:1126-1231`）：
- 调用 `generatePrismaQueryObj()` 创建请求记录
- INSERT 操作触发触发器，自动同步 mockExamples
- **无需任何额外代码**

#### 路径 3：数据库直接操作（⚠️ 注意）

- 直接 `UPDATE` 修改 `request` 字段：✅ 触发器自动同步
- 直接 `UPDATE` 修改 `mockExamples` 字段：❌ 不会反向同步到 request.responses
- **建议始终通过修改 request 字段来间接更新 mockExamples**

#### 路径 4：前端 UI（✅ 透明处理）

前端无需感知 mockExamples 字段：
- 前端仅发送 `request` 对象（含 `responses`）
- 后端接收后写入 `request` 字段
- 数据库触发器自动完成同步
- 前端无需修改任何代码

---

### 7.7 完整同步链路总结（架构确认版）

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│           request.responses（内嵌） → mockExamples（独立列）完整同步链路            │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  1. 应用层写入（仅操作 request 独立列）                                            │
│     ├─ createRequest()                 → 写入 request 独立列                     │
│     │  源码：`user-request.service.ts:119-175`                                   │
│     ├─ updateRequest()                 → 更新 request 独立列                     │
│     │  源码：`user-request.service.ts:185-220`                                   │
│     └─ importCollectionsFromJSON()     → 批量 INSERT request 独立列              │
│        源码：`user-collection.service.ts:1126-1231`                              │
│                                                                                 │
│  2. 数据库触发层（监听 request 独立列变更）                                        │
│     ├─ BEFORE INSERT ON UserRequest                                              │
│     │  └─ trigger_sync_mock_examples_user_request                                │
│     │     源码：`migration.sql:123-126`                                          │
│     ├─ BEFORE UPDATE OF request ON UserRequest                                   │
│     │  └─ trigger_sync_mock_examples_user_request                                │
│     │     源码：`migration.sql:123-126`                                          │
│     └─ （TeamRequest 同理）                                                      │
│        源码：`migration.sql:129-132`                                             │
│                                                                                 │
│  3. 同步函数执行（sync_mock_examples）                                            │
│     源码：`migration.sql:91-120`                                                 │
│     ├─ 从内嵌结构读取：NEW.request->'responses'                                   │
│     │  源码：`migration.sql:111`                                                 │
│     ├─ jsonb_each() 展开为 (key, value) 行集                                      │
│     ├─ 9 个字段映射转换（见 7.2 节字段映射表）                                    │
│     ├─ jsonb_agg() 聚合成数组                                                     │
│     ├─ 构造 { examples: [...] } 外层对象                                          │
│     └─ 写入 mockExamples 独立列：NEW."mockExamples" := ...                        │
│        源码：`migration.sql:94`                                                  │
│                                                                                 │
│  4. 数据持久化（两个独立列同时写入）                                               │
│     └─ INSERT/UPDATE 记录                                                        │
│        ├─ request 独立列：{ ..., "responses": {...} }                            │
│        └─ mockExamples 独立列：{ "examples": [...] }                             │
│                                                                                 │
│  5. mock 服务读取（仅读取 mockExamples 独立列）                                    │
│     ├─ fetchRequestsWithExamples()                                               │
│     │  ├─ WHERE mockExamples IS NOT NULL                                         │
│     │  │  源码：`mock-server.service.ts:814,824`                                 │
│     │  └─ SELECT id, mockExamples                                                │
│     │     源码：`mock-server.service.ts:817-818,827-828`                         │
│     ├─ findExampleByIdOrName()                                                   │
│     │  └─ 直接遍历 mockExamples.examples                                         │
│     │     源码：`mock-server.service.ts:846-847`                                 │
│     └─ fetchCandidateExamples()                                                  │
│         └─ 直接遍历 mockExamples.examples                                         │
│            源码：`mock-server.service.ts:903-904`                                │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 同步方向确认（含代码证据）

**单向同步**：`request.responses`（内嵌结构） → `mockExamples`（独立列）

| 操作 | 结果 | 代码证据 |
|-----|------|---------|
| 修改 `request.responses` | ✅ 自动更新 `mockExamples` | `migration.sql:124, 130` 触发器定义 |
| 修改 `mockExamples` 独立列 | ❌ 不会反向同步到 `request.responses` | 无反向触发器 |
| 建议操作方式 | 始终通过修改 `request` 独立列来间接更新 mockExamples | - |

#### 架构设计优势

1. **性能优化**：mock 服务查询时只需读取 `mockExamples` 独立列，无需解析 `request` JSON 的嵌套结构
2. **数据一致性**：数据库触发器保证 `request.responses` 和 `mockExamples` 始终同步
3. **架构解耦**：应用层和 mock 服务层完全解耦，应用层只需维护 `request` 格式，mock 服务层只需读取 `mockExamples` 格式

---

### 7.8 与 mock 服务读取路径的衔接

mock 服务完全不需要感知同步机制，直接读取 `mockExamples` 独立列即可：

**`fetchRequestsWithExamples`**（`mock-server.service.ts:809-831`）：
```typescript
return mockServer.workspaceType === WorkspaceType.USER
  ? await this.prisma.userRequest.findMany({
      where: {
        collectionID: { in: collectionIds },
        mockExamples: { not: null },  // ← 数据库级过滤独立列
      },
      select: {
        id: true,
        mockExamples: true,  // ← 直接读取独立列，不读取 request 列
      },
    })
  : // team 同理
```

**`findExampleByIdOrName`**（`mock-server.service.ts:837-874`）：
```typescript
for (const request of requests) {
  const mockExamples = request.mockExamples as any;  // ← 从独立列读取
  if (mockExamples?.examples && Array.isArray(mockExamples.examples)) {
    for (const exampleData of mockExamples.examples) {
      // 直接使用同步后的数据，无需访问 request.responses
    }
  }
}
```

**`fetchCandidateExamples`**（`mock-server.service.ts:879-925`）：
```typescript
for (const request of requests) {
  const mockExamples = request.mockExamples as any;  // ← 从独立列读取
  if (mockExamples?.examples && Array.isArray(mockExamples.examples)) {
    for (const exampleData of mockExamples.examples) {
      // 直接使用同步后的数据，无需访问 request.responses
    }
  }
}
```

**架构证据**：mock 服务的查询中 `select` 子句只选择 `mockExamples` 独立列，不选择 `request` 列，证明 `mockExamples` 是完全独立的列，mock 服务不需要读取 `request` 列的任何内容。

---

## 八、示例数据合成响应

### 8.1 响应头处理

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

### 8.2 响应格式自动检测

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

### 8.3 安全头注入

无论用户是否定义，都会自动注入安全头（`mock-server.controller.ts:177-182`）：
```typescript
res.setHeader('X-Content-Type-Options', 'nosniff');
if (!isSubdomainAccess) {
  res.setHeader('Content-Security-Policy', "default-src 'none'; sandbox");
  res.setHeader('X-Frame-Options', 'DENY');
}
```

### 8.4 关于自动创建示例的说明

`mockServerCollRequestExample()`（`constants/mock-server-coll-request-example.ts`）定义的是**传统 Hoppscotch 请求格式**，使用的是 `responses` 字段（嵌套 `originalRequest`）。这个文件的作用是：

1. 当用户勾选"自动创建请求示例"时，导入这个集合作为起点模板
2. **导入后触发器自动同步 mockExamples 字段**，mock 服务可直接使用
3. 导入的集合包含 6 个示例请求（见下表）

| 请求名称 | 方法 | 端点 | 状态码示例 |
|---------|------|------|-----------|
| addPet | POST | `/v2/pet` | 405 |
| updatePet | PUT | `/v2/pet` | 400, 404, 405 |
| findPetsByStatus | GET | `/v2/pet/findByStatus` | 200, 400 |
| getPetById | GET | `/v2/pet/<<petId>>` | 200, 400, 404 |
| updatePetWithForm | POST | `/v2/pet/<<petId>>` | 405 |
| deletePet | DELETE | `/v2/pet/<<petId>>` | 400, 404 |

---

## 九、完整匹配流程（修正版）

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

## 十、关键优化点

1. **单次数据库查询**：`fetchRequestsWithExamples()` 一次性获取所有候选请求，避免 N+1 查询
2. **快速路径预检查**：`couldPathMatch()` 在评分前过滤 80% 不可能匹配的示例
3. **精确匹配快速路径**：通过请求头指定 ID/名称时直接返回，跳过所有评分流程
4. **数据库级过滤**：`where: { mockExamples: { not: null } }` 只获取有示例的请求
5. **集合 ID 复用**：`collectionIds` 在精确匹配和候选匹配中复用，避免重复查询
6. **method 前置过滤**：在解析端点前就过滤掉方法不匹配的示例，减少解析开销

---

## 十一、代码引用速查

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
| autoCreateRequestExample 导入 | `mock-server.service.ts:328-334` |
| generatePrismaQueryObj 请求创建 | `user-collection.service.ts:1093-1104` |
| importCollectionsFromJSON 主流程 | `user-collection.service.ts:1126-1231` |
| createRequest（触发同步） | `user-request.service.ts:119-175` |
| updateRequest（触发同步） | `user-request.service.ts:185-220` |
| fetchRequestsWithExamples | `mock-server.service.ts:809-831` |
| findExampleByIdOrName | `mock-server.service.ts:837-874` |
| fetchCandidateExamples | `mock-server.service.ts:879-925` |
| mockExamples 数据库 schema | `schema.prisma:62-65, 183-186` |
| **sync_mock_examples 触发器函数 | `migrations/20251016080714_mock_server/migration.sql:91-120` |
| **UserRequest 触发器定义 | `migrations/20251016080714_mock_server/migration.sql:123-126` |
| **TeamRequest 触发器定义 | `migrations/20251016080714_mock_server/migration.sql:129-132` |
| **Backfill 历史数据 | `migrations/20251016080714_mock_server/migration.sql:134-138` |
| **GIN 索引创建 | `migrations/20251016080714_mock_server/migration.sql:140-142` |
