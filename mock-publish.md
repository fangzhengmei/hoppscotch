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
- `TeamRequest.mockExamples` 字段同时服务于 Mock 服务和文档展示
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
    │  (fetchRequestsWithExamples 方法)
    │
    ▼
4. 匹配请求：
    ├─ 快速路径：检查自定义请求头
    │   ├─ x-mock-response-id：按 ID 精确匹配
    │   ├─ x-mock-response-name：按名称匹配
    │   └─ x-mock-response-code：按状态码筛选
    │
    └─ 常规路径：智能匹配
        ├─ 过滤 HTTP 方法不匹配的
        ├─ 路径匹配（支持 <<变量>> 语法）
        ├─ 查询参数匹配
        ├─ 评分排序（Postman 算法：基础分 100）
        │   ├─ 路径精确匹配 = 100 分
        │   ├─ 路径含变量 = 95 分
        │   └─ 查询参数匹配度按比例折算
        └─ 返回最高分的响应示例
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
    // 检查路径结构是否匹配（分段数相同）
    // 支持 Hoppscotch 变量语法：<<variable>>
    // 如果路径含变量，扣 5 分
    score -= 5;
  }

  // 查询参数匹配：按匹配比例折算分数
  // 例如：2 个参数匹配了 1 个 → 分数 × 50%
  const matchPercentage = (paramMatches / totalParams) * 100;
  score = score * (matchPercentage / 100);

  return score;
}
```

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

// 同域访问时，自动降级危险的 MIME 类型为 text/plain
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
| 实时同步 | `true` | 不保存快照，访问时动态从 Collection 拉取 | 文档需要与 API 开发保持同步 |
| 版本快照 | `false` | 发布时将 Collection 数据快照存入 `documentTree` | 冻结特定版本的文档 |

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
    ├─ autoSync=false 时：导出 Collection 快照到 documentTree
    ├─ autoSync=true 时：documentTree 留空，访问时动态拉取
    └─ 保存到 PublishedDocs 表
    │
    ▼
5. 返回访问 URL：`/view/{slug}/{version}`
```

### 4.3 文档访问流程

访问已发布文档时的数据流（`published-docs.service.ts:330-413`）：

```
GET /view/{slug}/{version}
    │
    ▼
1. 前端路由到文档页面
    │
    ▼
2. 调用 API：GET /api/v1/published-docs/{slug}/{version}
    │
    ▼
3. 后端 getPublishedDocBySlugPublic 方法：
    ├─ 根据 slug + version 查找 PublishedDocs 记录
    │
    ├─ 如果 autoSync = true：
    │   └─ 实时从 Collection 导出最新数据
    │
    ├─ 如果 autoSync = false：
    │   └─ 直接返回 documentTree 中保存的快照
    │
    └─ 附带环境变量信息（如果关联了环境）
    │
    ▼
4. 前端渲染文档页面
    ├─ 左侧导航：Collection 的文件夹/请求树
    ├─ 右侧内容：请求详情、参数说明、响应示例
    └─ "在 Hoppscotch 中打开" 按钮
```

---

## 五、两者的咬合点与差异

### 5.1 共同依赖

| 依赖项 | Mock Server | Published Docs | 说明 |
|--------|-------------|----------------|------|
| Collection ID | ✅ | ✅ | 都通过 collectionID 关联 |
| 请求定义 | ✅ | ✅ | 方法、路径、参数 |
| 响应示例（mockExamples） | ✅ | ✅ | 核心共享数据 |
| 工作空间权限 | ✅ | ✅ | 都遵循 USER/TEAM 权限模型 |

### 5.2 核心差异

| 维度 | Mock Server | Published Docs |
|------|-------------|----------------|
| 用途 | 运行时 API 模拟，返回 HTTP 响应 | 静态文档展示，供开发者阅读 |
| 访问方式 | HTTP 请求到 `/mock/{subdomain}/path` | 浏览器访问 `/view/{slug}/{version}` |
| 数据时效性 | 总是使用 Collection 最新数据 | 可选择实时同步或版本快照 |
| 安全要求 | 严格的 XSS 防护、头信息过滤 | 主要是访问控制 |
| 性能要求 | 低延迟、高并发 | 静态内容、可缓存 |

### 5.3 典型使用场景

一个完整的 API 开发工作流：

1. **开发阶段**：在 Collection 中定义 API 请求和响应示例
2. **前端对接**：为 Collection 创建 Mock Server，前端直接调用 Mock API
3. **文档交付**：为 Collection 发布文档，分享给团队成员
4. **版本迭代**：
   - Mock Server 自动使用最新的响应示例
   - 文档可以选择：
     - `autoSync=true`：自动反映 API 变更
     - `autoSync=false`：冻结特定版本，不随 Collection 变化

---

## 六、代码溯源

| 功能 | 文件位置 | 关键方法/组件 |
|------|----------|--------------|
| Mock 服务核心逻辑 | `packages/hoppscotch-backend/src/mock-server/mock-server.service.ts` | `handleMockRequest`, `calculateMatchScore` |
| Mock 控制器 | `packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts` | `handleMockRequest` |
| 文档发布服务 | `packages/hoppscotch-backend/src/published-docs/published-docs.service.ts` | `createPublishedDoc`, `getPublishedDocBySlugPublic` |
| 文档发布前端 | `packages/hoppscotch-common/src/components/collections/documentation/index.vue` | `handlePublish`, `handleUpdate` |
| 文档服务（前端） | `packages/hoppscotch-common/src/services/documentation.service.ts` | `DocumentationService` 类 |
| 数据模型 | `packages/hoppscotch-backend/prisma/schema.prisma` | `MockServer`, `PublishedDocs`, `TeamRequest`, `UserRequest` |

---

## 七、设计亮点

1. **单一数据源**：API 定义只写一次，Mock 和文档自动复用，避免不一致
2. **灵活的版本控制**：文档支持实时同步和快照两种模式，满足不同场景
3. **安全优先**：Mock 服务有多层安全防护，防止恶意利用
4. **智能匹配算法**：Mock 服务的路径匹配支持变量和评分，接近真实 API 行为
5. **权限复用**：两者都复用现有的工作空间权限模型，无需单独设计权限系统
