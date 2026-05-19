# Mock 服务与 API 文档发布的咬合关系分析

## 一、核心设计思想：同根同源，字段分流

Hoppscotch 的 Mock 服务和 API 文档发布**最终都关联到同一个 Collection**，但它们在数据库层面读取的是 **不同的字段**。这种设计既保证了数据来源的一致性（都属于同一个 API 集合），又允许两者独立演化。

```
                    ┌───────────────────────────────┐
                    │   TeamCollection /            │
                    │   UserCollection              │
                    │   (API 请求集合容器)          │
                    └───────────────┬───────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │   TeamRequest / UserRequest   │
                    │  ┌─────────────────────────┐  │
                    │  │ request (JSON)          │  │
                    │  │  - 方法、路径、参数     │  │
                    │  │  - responses (响应示例) │──┼───► Published Docs 读取
                    │  └─────────────────────────┘  │
                    │  ┌─────────────────────────┐  │
                    │  │ mockExamples (JSON?)    │  │
                    │  │  - Mock 专用响应示例    │──┼───► Mock Server 读取
                    │  └─────────────────────────┘  │
                    └───────────────────────────────┘
```

**关键修正**：Mock Server 和 Published Docs **并不共享同一个数据字段**。它们分别读取 `mockExamples` 和 `request.responses` 两个独立的字段。

---

## 二、数据模型与关联关系

### 2.1 数据库表结构

```prisma
// Collection 容器
model TeamCollection {
  id         String         @id @default(cuid())
  title      String
  requests   TeamRequest[]
}

// 请求表 —— 包含两个独立的响应示例字段
model TeamRequest {
  id           String         @id @default(cuid())
  collectionID String         // 关联 Collection
  title        String
  request      Json           // HoppRESTRequest 对象，内含 responses 字段
  mockExamples Json?          // Mock 专用响应示例，可为空
}

// Mock Server 配置
model MockServer {
  id            String        @id @default(cuid())
  subdomain     String        @unique
  collectionID  String        // 关联 Collection
  delayInMs     Int           // 响应延迟
}

// 已发布文档
model PublishedDocs {
  id            String        @id @default(cuid())
  slug          String        // 访问路径
  version       String        // 版本号
  collectionID  String        // 关联 Collection
  autoSync      Boolean       // 是否实时同步
  documentTree  Json?         // autoSync=false 时存储 Collection 快照
}
```

### 2.2 数据字段边界对比

| 维度 | Mock Server | Published Docs |
|------|-------------|----------------|
| **读取字段** | `TeamRequest.mockExamples` | `TeamRequest.request` (内含 `responses`) |
| **字段类型** | `Json?`（可为空） | `Json`（必有） |
| **读取方式** | 直接查询 `mockExamples not null` | 通过 Collection Service 导出完整 Collection |
| **数据范围** | 只包含响应示例 | 包含完整请求定义（方法、路径、参数、认证、响应等） |
| **前端展示** | 不展示，直接返回 HTTP 响应 | 通过 `request.responses` 展示示例 |

---

## 三、Mock 服务行为生成链路

### 3.1 Mock 请求处理全流程

Mock Server 核心逻辑：`packages/hoppscotch-backend/src/mock-server/mock-server.service.ts:698-800`

```
HTTP 请求到达 Mock 端点
    │
    ▼
1. 解析 Mock Server ID（从子域名或路径前缀）
    │
    ▼
2. 获取关联的 Collection 及其所有子集合 ID
    │  (getCollectionIds 递归查询)
    │
    ▼
3. 批量查询所有带 mockExamples 的请求
    │  fetchRequestsWithExamples():
    │  └─ WHERE mockExamples IS NOT NULL
    │     SELECT id, mockExamples
    │
    ▼
4. 匹配响应（两级路径）：
    ├─ 快速路径（自定义请求头）：
    │   ├─ x-mock-response-id → 按 ID 精确匹配
    │   ├─ x-mock-response-name → 按名称匹配
    │   └─ x-mock-response-code → 按状态码筛选
    │
    └─ 常规路径（智能匹配）：
        ├─ fetchCandidateExamples() 过滤候选
        │  ├─ 方法匹配检查
        │  └─ couldPathMatch() 预过滤（分段数检查）
        ├─ calculateMatchScore() 评分排序
        └─ 返回最高分（同分优先 200 状态码）
    │
    ▼
5. 应用延迟（delayInMs），格式化并返回响应
```

### 3.2 评分算法实现细节

**`calculateMatchScore` 方法**（mock-server.service.ts:1076-1160）

```typescript
private calculateMatchScore(
  example: any,
  requestPath: string,
  requestQueryParams: Record<string, string>,
): number {
  let score = 100;  // 基础分

  // 阶段一：路径匹配
  if (example.path !== requestPath) {
    const examplePathParts = example.path.split('/').filter(Boolean);
    const requestPathParts = requestPath.split('/').filter(Boolean);

    // 路径分段数不同 → 直接 0 分
    if (examplePathParts.length !== requestPathParts.length) {
      return 0;
    }

    // 逐段检查
    let pathMatches = true;
    for (let i = 0; i < examplePathParts.length; i++) {
      const examplePart = examplePathParts[i];
      const requestPart = requestPathParts[i];

      // 匹配条件：完全相等 或 以<<开头 或 包含<<
      if (
        examplePart === requestPart ||
        examplePart.startsWith('<<') ||
        examplePart.includes('<<')
      ) {
        continue;
      } else {
        pathMatches = false;
        break;
      }
    }

    if (!pathMatches) {
      return 0; // 路径不匹配 → 0 分
    }

    // 路径含变量 → 扣 5 分（95 分）
    score -= 5;
  }

  // 阶段二：查询参数匹配
  const exampleParams = example.queryParams || {};
  const exampleParamKeys = Object.keys(exampleParams);
  const requestParamKeys = Object.keys(requestQueryParams);

  if (exampleParamKeys.length > 0 || requestParamKeys.length > 0) {
    let paramMatches = 0;     // 完全匹配的参数
    let partialMatches = 0;   // 存在但值不同
    let missingParams = 0;    // 缺失的参数

    exampleParamKeys.forEach((key) => {
      if (requestQueryParams[key] !== undefined) {
        if (requestQueryParams[key] === exampleParams[key]) {
          paramMatches++;
        } else {
          partialMatches++;
        }
      } else {
        missingParams++;
      }
    });

    // 请求中额外的参数也算缺失
    requestParamKeys.forEach((key) => {
      if (exampleParams[key] === undefined) {
        missingParams++;
      }
    });

    // 按匹配比例折算分数
    const totalParams = paramMatches + partialMatches + missingParams;
    if (totalParams > 0) {
      const matchPercentage = (paramMatches / totalParams) * 100;
      score = score * (matchPercentage / 100);
    }
  }

  return score;
}
```

**预过滤优化**（`couldPathMatch` 方法，mock-server.service.ts:929-948）：
在进入完整评分前先做快速检查，减少计算量：
- 路径完全相同 → 匹配
- 路径分段数不同 → 不匹配
- 路径含 `<<` → 可能匹配，进入完整评分
- 其他情况 → 不匹配

### 3.3 Mock 响应安全防护

在 `mock-server.controller.ts:19-183` 实现了多层安全防护：

```typescript
// 安全头黑名单 —— 禁止 Mock 响应覆盖
const SECURITY_HEADER_BLOCKLIST = new Set([
  'content-security-policy',
  'x-content-type-options',
  'x-frame-options',
  'content-disposition',
  'set-cookie',
]);

// 同域访问时（路径模式），危险 MIME 类型自动降级为 text/plain
const ACTIVE_CONTENT_TYPES = new Set([
  'application/javascript',
  'text/html',
  'image/svg+xml',
  'application/xhtml+xml',
]);
```

---

## 四、已发布文档的拼装链路

### 4.1 两种发布模式

| 模式 | autoSync | 数据存储 | 适用场景 |
|------|----------|----------|----------|
| Live 版本 | `true` | `documentTree` 留空，访问时动态导出 Collection | 文档需与 API 开发实时同步 |
| Frozen 版本 | `false` | 发布时将 Collection 快照存入 `documentTree` | 冻结特定版本的文档 |

### 4.2 文档发布流程

前端组件：`packages/hoppscotch-common/src/components/collections/documentation/index.vue`

```
用户点击"发布文档"
    │
    ▼
1. 打开 PublishDocModal 弹窗
    ├─ 填写标题、版本号
    ├─ 选择是否自动同步（autoSync）
    └─ 选择关联环境（可选）
    │
    ▼
2. 调用 createPublishedDoc mutation
    │
    ▼
3. 后端处理（published-docs.service.ts:529-641）：
    ├─ 验证 Collection 访问权限
    ├─ getOrGenerateSlug()：同一 Collection 的版本共享 slug
    │  └─ 查询该 collectionID 最早的 publishedDoc，复用其 slug
    ├─ autoSync=false：
    │  └─ 调用 exportCollectionToJSONObject() 导出快照
    ├─ autoSync=true：
    │  └─ documentTree 留空
    └─ 保存到 PublishedDocs 表
    │
    ▼
4. 返回 URL：/view/{slug}/{version}
```

### 4.3 后端接口与前端路径的衔接

**后端 REST 接口**（`published-docs.controller.ts`）：

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/published-docs/:slug` | 无版本路径，自动选择默认版本 |
| GET | `/api/v1/published-docs/:slug/:version` | 指定版本路径 |

**前端路由**（`pages/view/_id/_version.vue`）：

| 前端路径 | 对应后端调用 |
|----------|-------------|
| `/view/{slug}` | `GET /api/v1/published-docs/{slug}`（无版本） |
| `/view/{slug}/{version}` | `GET /api/v1/published-docs/{slug}/{version}`（指定版本） |

**版本选择规则**（`published-docs.service.ts:295-303`）：

当无版本访问时（`version=null`），后端按以下规则排序并选择第一个：

```typescript
orderBy: [{ autoSync: 'desc' }, { createdOn: 'desc' }]
```

优先级：
1. **第一优先级**：`autoSync=true` 的 Live 版本
2. **第二优先级**：创建时间降序（最新创建的在前）

**结论**：无版本路径时，**优先返回 Live 版本**；无 Live 版本时返回最新创建的版本。

### 4.4 文档访问数据流

```
用户访问 /view/{slug} 或 /view/{slug}/{version}
    │
    ▼
1. 前端路由到 view/_id/_version.vue
    │
    ▼
2. 调用 getPublishedDocBySlugREST(slug, version)
    │
    ▼
3. 后端 getPublishedDocBySlugPublic()：
    ├─ 查询该 slug 下的所有版本（按 autoSync desc, createdOn desc 排序）
    ├─ version=null 时使用排序后的第一个版本
    ├─ 根据 slug + version 查找 PublishedDocs 记录
    │
    ├─ autoSync = true 时：
    │  └─ 实时调用 exportCollectionToJSONObject()
    │     └─ 重新拉取关联环境变量
    │
    ├─ autoSync = false 时：
    │  └─ 直接返回 documentTree 快照
    │
    └─ 返回 JSON 响应（含 documentTree 字符串）
    │
    ▼
4. 前端处理：
    ├─ JSON.parse(documentTree) → CollectionFolder
    ├─ collectionFolderToHoppCollection() → HoppCollection
    └─ flattenCollection() 递归展开为左侧导航树
    │
    ▼
5. 渲染页面：
    ├─ DocumentationHeader：标题、版本选择器
    ├─ CollectionStructure：左侧导航
    └─ RequestPreview：右侧请求详情（含 responses 展示）
```

---

## 五、两者的数据读取边界对比

### 5.1 完整读取链路对比

```
┌─────────────────────────────────────────────────────────────────┐
│                    TeamRequest 数据库记录                        │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ id: "req_123"                                             │  │
│  │ collectionID: "coll_456"                                  │  │
│  │ title: "Get User"                                         │  │
│  │                                                           │  │
│  │ request: {                   ◄────────── Published Docs  │  │
│  │   v: "17",                                            读  │  │
│  │   name: "Get User",                                     取  │  │
│  │   method: "GET",                                         │  │
│  │   endpoint: "/api/users/<<id>>",                         │  │
│  │   responses: {                  ◄─────────── 文档显示    │  │
│  │     "200 OK": { ... }             响应示例              │  │
│  │   }                                                      │  │
│  │ }                                                         │  │
│  │                                                           │  │
│  │ mockExamples: {                ◄────────── Mock Server   │  │
│  │   examples: [                                          读 │  │
│  │     {                     ◄─────────── Mock 服务         │  │
│  │       name: "200 OK",                  使用              │  │
│  │       method: "GET",                                      │  │
│  │       path: "/api/users/1",                               │  │
│  │       statusCode: 200,                                    │  │
│  │       responseBody: "{\"id\":1}"                         │  │
│  │     }                                                     │  │
│  │   ]                                                       │  │
│  │ }                                                         │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 边界总结表

| 维度 | Mock Server | Published Docs |
|------|-------------|----------------|
| **数据源字段** | `mockExamples` | `request.responses` |
| **查询方法** | `fetchRequestsWithExamples()` 直接查 Request 表 | `exportCollectionToJSONObject()` 通过 Collection Service 导出 |
| **数据过滤** | `WHERE mockExamples IS NOT NULL` | 无过滤，导出所有请求 |
| **数据格式** | 自定义格式（`{ examples: [...] }`） | HoppRESTRequest 标准格式 |
| **时效性** | 总是最新 | Live 版本总是最新，Frozen 版本是发布快照 |
| **是否为空** | 可为空（无 Mock 示例时不返回） | 必有（至少包含请求定义） |

### 5.3 共同依赖与差异

| 依赖项 | Mock Server | Published Docs |
|--------|-------------|----------------|
| Collection ID 关联 | ✅ | ✅ |
| 请求方法/路径信息 | ✅ | ✅ |
| 响应示例数据 | ✅（mockExamples） | ✅（request.responses） |
| 工作空间权限检查 | ✅ | ✅ |
| 环境变量支持 | ❌ | ✅ |
| 版本管理 | ❌（始终最新） | ✅（多版本、Live/Frozen） |

---

## 六、核心差异总结

| 维度 | Mock Server | Published Docs |
|------|-------------|----------------|
| **用途** | 运行时 API 模拟，返回 HTTP 响应 | 静态文档展示，供开发者阅读 |
| **访问方式** | `GET /mock/{subdomain}/{path}` | 浏览器访问 `/view/{slug}/{version}` |
| **数据字段** | 独立的 `mockExamples` 字段 | `request` 字段中的 `responses` |
| **版本概念** | 无版本，始终最新 | 多版本管理，支持 Live/Frozen |
| **安全要求** | XSS 防护、头过滤、MIME 降级 | 主要是访问控制 |
| **性能要求** | 低延迟、高并发 | 静态内容、可缓存 |
| **匹配逻辑** | 复杂的评分算法 | 按 ID 直接展示 |

---

## 七、代码溯源

| 功能 | 文件位置 | 关键方法/组件 |
|------|----------|--------------|
| Mock 服务核心 | `packages/hoppscotch-backend/src/mock-server/mock-server.service.ts` | `handleMockRequest`, `calculateMatchScore`, `couldPathMatch` |
| Mock 控制器 | `packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts` | `handleMockRequest` |
| 文档发布服务 | `packages/hoppscotch-backend/src/published-docs/published-docs.service.ts` | `createPublishedDoc`, `getPublishedDocBySlugPublic`, `getOrGenerateSlug` |
| 文档控制器 | `packages/hoppscotch-backend/src/published-docs/published-docs.controller.ts` | `getPublishedDocsBySlugLatest`, `getPublishedDocsBySlug` |
| Collection 导出 | `packages/hoppscotch-backend/src/team-collection/team-collection.service.ts` | `exportCollectionToJSONObject` |
| 文档发布前端 | `packages/hoppscotch-common/src/components/collections/documentation/index.vue` | `handlePublish` |
| 文档浏览页面 | `packages/hoppscotch-common/src/src/pages/view/_id/_version.vue` | `fetchDocs`, `flattenCollection` |
| 请求预览组件 | `packages/hoppscotch-common/src/components/collections/documentation/RequestPreview.vue` | `getResponseExamples` |
| 前端 API 调用 | `packages/hoppscotch-common/src/helpers/backend/queries/PublishedDocs.ts` | `getPublishedDocBySlugREST` |
| 数据模型 | `packages/hoppscotch-backend/prisma/schema.prisma` | `TeamRequest`, `UserRequest`, `MockServer`, `PublishedDocs` |

---

## 八、设计要点

1. **字段分流设计**：Mock 和文档分别使用独立字段，互不干扰，但最终都关联同一个 Collection
2. **灵活版本控制**：文档支持 Live（实时同步）和 Frozen（版本快照）两种模式
3. **智能版本路由**：无版本路径时优先返回 Live 版本，符合直觉
4. **安全分层防护**：Mock 服务有多层安全防护（头黑名单、MIME 降级）
5. **性能优化**：Mock 服务通过 `couldPathMatch` 预过滤和数据库级 `IS NOT NULL` 过滤减少计算量
6. **解耦架构**：Mock 和 Published Docs 没有直接代码依赖，通过数据库中的 Collection 数据解耦
