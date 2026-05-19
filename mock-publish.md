# Mock 服务与 API 文档发布的咬合关系分析

## 一、核心设计思想：单一数据源，多场景复用

Hoppscotch 的 Mock 服务和 API 文档发布共享同一个核心数据模型 —— **Collection（API 集合）**。这种设计实现了"一次定义，多处使用"的目标。

```
                    ┌─────────────────────┐
                    │   Collection (API   │
                    │   请求定义集合)     │
                    │  - 请求方法/路径    │
                    │  - 请求头/参数      │
                    │  - 响应示例         │
                    └─────────┬───────────┘
                              │
           ┌──────────────────┼──────────────────┐
           │                  │                  │
┌──────────▼─────────┐  ┌─────▼──────────┐  ┌──▼──────────────┐
│  Mock Server       │  │  Published Docs│  │  API Testing    │
│  (运行时模拟)      │  │  (静态文档)    │  │  (接口测试)     │
└────────────────────┘  └────────────────┘  └─────────────────┘
```

---

## 二、接口定义复用：Collection 作为唯一真相源

### 2.1 Collection 数据结构

Collection 是所有 API 相关功能的基础，包含：
- 请求定义（方法、路径、头信息、参数）
- 认证配置
- 响应示例（`mockExamples`，这是 Mock 服务的核心数据源）

数据库层面的关联关系（Prisma Schema）：

```prisma
// Collection 存储模型（以 Team 为例）
model TeamCollection {
  id         String           @id @default(cuid())
  title      String           // 集合名称
  data       Json?            // 集合元数据
  requests   TeamRequest[]    // 关联的请求列表
}

model TeamRequest {
  id           String         @id @default(cuid())
  collectionID String         // 关联的 Collection ID
  request      Json           // 请求定义（方法、路径等）
  mockExamples Json?          // 响应示例 —— Mock 和文档共享
}
```

### 2.2 Mock Server 与 Published Docs 如何关联 Collection

两者都通过 `collectionID` 字段关联到同一个 Collection：

```prisma
// Mock Server 模型
model MockServer {
  id            String        @id @default(cuid())
  name          String
  subdomain     String        @unique  // Mock 服务访问域名
  collectionID  String        // 关联 Collection
  // ... 其他字段
}

// Published Docs 模型
model PublishedDocs {
  id            String        @id @default(cuid())
  slug          String        // 文档访问路径
  collectionID  String        // 关联 Collection
  autoSync      Boolean       // 是否自动同步 Collection 变更
  documentTree  Json?         // 快照数据（autoSync=false 时使用）
  // ... 其他字段
}
```

**关键复用点**：
- `TeamRequest.mockExamples` / `UserRequest.mockExamples` 字段同时服务于 Mock 服务和文档展示
- Mock 服务用它来生成响应，文档用它来展示示例响应
- 两者共享完全相同的数据结构，无需重复定义

---

## 三、Mock 行为生成：从 Collection 到 HTTP 响应

### 3.1 Mock 请求处理流程

Mock Server 的核心逻辑在 `packages/hoppscotch-backend/src/mock-server/mock-server.service.ts:698-800`

```
HTTP 请求到达
    │
    ▼
1. 解析 Mock Server ID（从子域名或路径）
    │
    ▼
2. 获取关联的 Collection 及其所有子集合 ID
    │  (getCollectionIds 方法，递归获取所有子集合)
    │
    ▼
3. 从数据库获取这些集合中所有带有 mockExamples 的请求
    │  (fetchRequestsWithExamples 方法，只查询 mockExamples not null 的记录)
    │
    ▼
4. 匹配请求：
    ├─ 快速路径：检查自定义请求头
    │   ├─ x-mock-response-id：按 ID 精确匹配
    │   ├─ x-mock-response-name：按名称匹配
    │   └─ x-mock-response-code：按状态码筛选
    │
    └─ 常规路径：智能匹配
        ├─ 快速过滤：couldPathMatch 预检查（分段数相同或含变量）
        ├─ 过滤 HTTP 方法不匹配的
        ├─ 路径匹配（支持 <<变量>> 语法）
        ├─ 查询参数匹配
        ├─ 评分排序（Postman 算法：基础分 100）
        │   ├─ 路径精确匹配 = 100 分
        │   ├─ 路径含变量 = 95 分
        │   └─ 查询参数匹配度按比例折算
        └─ 返回最高分的响应示例（同分优先 200 状态码）
    │
    ▼
5. 应用延迟（delayInMs），返回响应
```

### 3.2 核心匹配算法代码解析

在 `mock-server.service.ts:1076-1160` 的 `calculateMatchScore` 方法：

```typescript
private calculateMatchScore(
  example: any,
  requestPath: string,
  requestQueryParams: Record<string, string>,
): number {
  let score = 100;  // 基础分

  // 路径匹配
  if (example.path !== requestPath) {
    const examplePathParts = example.path.split('/').filter(Boolean);
    const requestPathParts = requestPath.split('/').filter(Boolean);

    // 路径分段数不同直接返回 0 分
    if (examplePathParts.length !== requestPathParts.length) {
      return 0;
    }

    // 逐段检查
    let pathMatches = true;
    for (let i = 0; i < examplePathParts.length; i++) {
      const examplePart = examplePathParts[i];
      const requestPart = requestPathParts[i];

      // 变量匹配规则：完全相等 或 以<<开头 或 包含<<
      if (
        examplePart === requestPart ||
        examplePart.startsWith('<<') ||
        examplePart.includes('<<')
      ) {
        continue; // 匹配
      } else {
        pathMatches = false;
        break;
      }
    }

    if (!pathMatches) {
      return 0; // 路径不匹配返回 0 分
    }

    // 路径含变量，扣 5 分
    score -= 5;
  }

  // 查询参数匹配：按匹配比例折算分数
  // 例如：2 个参数匹配了 1 个 → 分数 × 50%
  const totalParams = paramMatches + partialMatches + missingParams;
  if (totalParams > 0) {
    const matchPercentage = (paramMatches / totalParams) * 100;
    score = score * (matchPercentage / 100);
  }

  return score;
}
```

**预过滤优化**：在 `couldPathMatch` 方法（mock-server.service.ts:929-948）中先做初步过滤，减少需要评分的示例数量：
- 路径完全相同 → 匹配
- 路径分段数不同 → 不匹配
- 路径含 `<<` → 可能匹配，进入完整评分
- 其他情况 → 不匹配

### 3.3 Mock 响应的安全处理

在 `mock-server.controller.ts:19-183`，系统做了严格的安全防护：

```typescript
// 安全头黑名单 —— 不允许 Mock 响应覆盖这些安全相关头
const SECURITY_HEADER_BLOCKLIST = new Set([
  'content-security-policy',
  'x-content-type-options',
  'x-frame-options',
  'content-disposition',
  'set-cookie',
]);

// 同域访问时（路径模式），自动降级危险的 MIME 类型为 text/plain
// 防止 XSS 攻击
const ACTIVE_CONTENT_TYPES = new Set([
  'application/javascript',
  'text/html',
  'image/svg+xml',
  // ...
]);
```

---

## 四、文档站点的拼装方式：从 Collection 到可浏览文档

### 4.1 两种发布模式

Published Docs 支持两种模式，由 `autoSync` 字段控制：

| 模式 | autoSync | 数据存储 | 适用场景 |
|------|----------|----------|----------|
| 实时同步（Live 版本） | `true` | 不保存快照，访问时动态从 Collection 拉取 | 文档需要与 API 开发保持同步 |
| 版本快照（Frozen 版本） | `false` | 发布时将 Collection 数据快照存入 `documentTree` | 冻结特定版本的文档 |

**特别说明**：Live 版本（`autoSync=true`）是一个特殊的版本，通常使用 `CURRENT` 作为版本标识。

### 4.2 文档发布流程

前端组件：`packages/hoppscotch-common/src/components/collections/documentation/index.vue`

```
用户点击"发布文档"按钮
    │
    ▼
1. 打开 PublishDocModal 弹窗
    │
    ▼
2. 用户填写：
    ├─ 标题（title）
    ├─ 版本号（version）
    ├─ 是否自动同步（autoSync）
    └─ 关联环境（可选）
    │
    ▼
3. 调用 createPublishedDoc mutation
    │
    ▼
4. 后端处理（published-docs.service.ts:529-641）：
    ├─ 验证用户对 Collection 的访问权限
    ├─ 生成或复用 slug（同一 Collection 的不同版本共享 slug）
    │   └─ 规则：查询该 collectionID 最早创建的 publishedDoc，复用其 slug
    ├─ autoSync=false 时：导出 Collection 快照到 documentTree
    ├─ autoSync=true 时：documentTree 留空，访问时动态拉取
    └─ 保存到 PublishedDocs 表
    │
    ▼
5. 返回访问 URL：`/view/{slug}/{version}`
```

### 4.3 文档访问路径与版本优先规则

**后端控制器**：`packages/hoppscotch-backend/src/published-docs/published-docs.controller.ts`

支持两种访问方式：

| 路径 | 说明 |
|------|------|
| `GET /api/v1/published-docs/{slug}` | 无版本路径，自动选择默认版本 |
| `GET /api/v1/published-docs/{slug}/{version}` | 指定版本路径 |

**前端路由**：`packages/hoppscotch-common/src/pages/view/_id/_version.vue`

| 前端路径 | 说明 |
|----------|------|
| `/view/{slug}` | 无版本路径，由后端选择默认版本 |
| `/view/{slug}/{version}` | 显示指定版本 |

**版本优先规则（核心修正）**：

在 `published-docs.service.ts:295-303` 中，版本排序规则为：
```typescript
orderBy: [{ autoSync: 'desc' }, { createdOn: 'desc' }]
```

这意味着：
1. **第一优先级**：`autoSync=true` 的 Live 版本排在最前面
2. **第二优先级**：按创建时间降序（最新创建的排在前面）

当无版本访问时（`version=null`），后端自动选择排序后的第一个版本：
```typescript
version: version ? version : allVersions.right[0].version
```

**结论**：无版本路径时，**优先返回 Live 版本**；如果没有 Live 版本，才返回最新创建的版本。

### 4.4 文档访问时的数据流

访问已发布文档时的完整流程（`published-docs.service.ts:330-413`）：

```
GET /view/{slug} 或 /view/{slug}/{version}
    │
    ▼
1. 前端路由到文档页面（view/_id/_version.vue）
    │
    ▼
2. 调用 API：GET /api/v1/published-docs/{slug}/{version?}
    │
    ▼
3. 后端 getPublishedDocBySlugPublic 方法：
    ├─ 根据 slug 查询该 slug 下的所有版本（按 autoSync desc, createdOn desc 排序）
    ├─ 如果 version=null，使用排序后的第一个版本（Live 优先）
    ├─ 根据 slug + version 查找 PublishedDocs 记录
    │
    ├─ 如果 autoSync = true：
    │   └─ 实时从 Collection 导出最新数据（通过 collectionService）
    │       └─ 同时重新拉取关联环境的最新变量
    │
    ├─ 如果 autoSync = false：
    │   └─ 直接返回 documentTree 中保存的快照
    │
    └─ 附带环境变量信息（如果关联了环境）
    │
    ▼
4. 前端渲染文档页面：
    ├─ 解析 documentTree 为 HoppCollection 结构
    ├─ 左侧导航：Collection 的文件夹/请求树（flattenCollection 递归展开）
    ├─ 右侧内容：请求详情、参数说明、响应示例
    └─ "在 Hoppscotch 中打开" 按钮
```

---

## 五、Mock 与 Published Docs 的数据来源边界

### 5.1 数据来源对比

| 维度 | Mock Server | Published Docs |
|------|-------------|----------------|
| **直接数据源** | `UserRequest.mockExamples`<br>`TeamRequest.mockExamples` | autoSync=true: Collection 导出<br>autoSync=false: `documentTree` 快照 |
| **读取方式** | 直接查询 Request 表，过滤 `mockExamples not null` | autoSync=true: 调用 collectionService.exportXXX<br>autoSync=false: 读取 PublishedDocs 表 |
| **数据范围** | 只读取响应示例（mockExamples） | 读取整个 Collection 结构（文件夹、请求、认证、变量等） |
| **时效性** | 总是最新 | autoSync=true: 总是最新<br>autoSync=false: 发布时的快照 |

### 5.2 数据流转路径

```
┌─────────────────────────────────────────────────────────────┐
│                    Collection 数据库表                       │
│  ┌────────────────┐    ┌─────────────────────────────────┐  │
│  │ TeamCollection │    │ TeamRequest                     │  │
│  │  - id           │    │  - id                           │  │
│  │  - title        │    │  - collectionID                 │  │
│  │  - ...          │    │  - request (JSON: 方法/路径等)  │  │
│  └────────┬───────┘    │  - mockExamples (JSON)         │  │
│           │            └───────────────┬─────────────────┘  │
│           │                            │                    │
└───────────┼────────────────────────────┼────────────────────┘
            │                            │
            │                            │
            ▼                            ▼
┌──────────────────────────┐   ┌───────────────────────────┐
│    Mock Server           │   │   Published Docs          │
│                          │   │                           │
│  ┌─ fetchRequestsWith    │   │  autoSync=true:           │
│  │  Examples()           │   │    ┌─ exportCollectionTo  │
│  │  只查询 mockExamples  │   │    │  JSONObject()        │
│  │  不为空的记录         │   │    └─ 导出完整 Collection │
│  │                        │   │                           │
│  └─ calculateMatchScore()│   │  autoSync=false:          │
│     匹配并返回响应       │   │    └─ 读取 documentTree    │
└──────────────────────────┘   └───────────────────────────┘
```

**关键边界说明**：
- 虽然两者最终都依赖同一个 Collection，但它们从数据库读取的表和字段不同
- Mock Server 直接访问 Request 表的 `mockExamples` 字段
- Published Docs 通过 Collection Service 导出整个 Collection 结构
- 两者没有直接的代码调用关系，完全通过数据库中的 Collection 数据解耦

### 5.3 共同依赖

| 依赖项 | Mock Server | Published Docs | 说明 |
|--------|-------------|----------------|------|
| Collection ID | ✅ | ✅ | 都通过 collectionID 关联 |
| 请求定义（方法、路径） | ✅ | ✅ | 都需要知道 API 的基本信息 |
| 响应示例（mockExamples） | ✅ | ✅ | 核心共享数据，用于生成 Mock 响应和文档示例 |
| 工作空间权限 | ✅ | ✅ | 都遵循 USER/TEAM 权限模型 |
| 环境变量 | ❌ | ✅ | 文档可关联环境变量，Mock 直接用示例数据 |

---

## 六、两者的核心差异

| 维度 | Mock Server | Published Docs |
|------|-------------|----------------|
| 用途 | 运行时 API 模拟，返回 HTTP 响应 | 静态文档展示，供开发者阅读 |
| 访问方式 | HTTP 请求到 `/mock/{subdomain}/path` | 浏览器访问 `/view/{slug}/{version}` |
| 数据时效性 | 总是使用 Collection 最新数据 | 可选择实时同步（Live）或版本快照（Frozen） |
| 安全要求 | 严格的 XSS 防护、头信息过滤、MIME 类型降级 | 主要是访问控制 |
| 性能要求 | 低延迟、高并发 | 静态内容、可缓存 |
| 版本概念 | 无版本概念，始终最新 | 多版本管理，支持 Live 和 Frozen 版本 |

---

## 七、典型使用场景

一个完整的 API 开发工作流：

1. **开发阶段**：在 Collection 中定义 API 请求和响应示例
2. **前端对接**：为 Collection 创建 Mock Server，前端直接调用 Mock API
   - Mock Server 自动使用最新的响应示例
3. **文档交付**：为 Collection 发布文档，分享给团队成员
   - 选择 `autoSync=true`（Live 版本）：文档自动反映 API 变更
   - 选择 `autoSync=false`（快照版本）：冻结特定版本的文档
4. **版本迭代**：
   - Mock Server 始终使用最新的响应示例
   - Live 版本文档自动更新
   - 快照版本保持不变，可通过 `/view/{slug}/v1.0` 永久访问

---

## 八、代码溯源

| 功能 | 文件位置 | 关键方法/组件 |
|------|----------|--------------|
| Mock 服务核心逻辑 | `packages/hoppscotch-backend/src/mock-server/mock-server.service.ts` | `handleMockRequest`, `calculateMatchScore`, `couldPathMatch` |
| Mock 控制器 | `packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts` | `handleMockRequest` |
| 文档发布服务 | `packages/hoppscotch-backend/src/published-docs/published-docs.service.ts` | `createPublishedDoc`, `getPublishedDocBySlugPublic`, `getPublishedDocsVersions` |
| 文档控制器 | `packages/hoppscotch-backend/src/published-docs/published-docs.controller.ts` | `getPublishedDocsBySlugLatest`, `getPublishedDocsBySlug` |
| 文档发布前端 | `packages/hoppscotch-common/src/components/collections/documentation/index.vue` | `handlePublish`, `handleUpdate` |
| 文档浏览页面 | `packages/hoppscotch-common/src/pages/view/_id/_version.vue` | `fetchDocs`, `flattenCollection` |
| 文档服务（前端） | `packages/hoppscotch-common/src/services/documentation.service.ts` | `DocumentationService`, `CURRENT_VERSION_TAG`, `isLiveVersion` |
| 数据模型 | `packages/hoppscotch-backend/prisma/schema.prisma` | `MockServer`, `PublishedDocs`, `TeamRequest`, `UserRequest` |

---

## 九、设计亮点

1. **单一数据源**：API 定义只写一次，Mock 和文档自动复用，避免不一致
2. **灵活的版本控制**：文档支持 Live（实时同步）和 Frozen（版本快照）两种模式，满足不同场景
3. **智能版本路由**：无版本路径时优先返回 Live 版本，符合直觉
4. **安全优先**：Mock 服务有多层安全防护（头黑名单、MIME 降级、CSP），防止恶意利用
5. **性能优化**：Mock 服务通过 `couldPathMatch` 预过滤和数据库级 `mockExamples not null` 过滤减少计算量
6. **智能匹配算法**：Mock 服务的路径匹配支持 `<<变量>>` 语法和评分机制，接近真实 API 行为
7. **权限复用**：两者都复用现有的工作空间权限模型，无需单独设计权限系统
8. **解耦设计**：Mock 和 Published Docs 没有直接代码依赖，通过数据库中的 Collection 数据解耦
