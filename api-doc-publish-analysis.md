# Hoppscotch 集合请求自动生成并发布 API 文档代码解读

本文从 Hoppscotch 代码中梳理出 “请求集合 → API 文档 → 对外发布” 的完整链路，重点围绕 **抽取规则**、**生成模板**、**发布路径** 三块内容。

## 一、整体流程一览

```
┌──────────────────────────────┐         ┌──────────────────────────────────┐
│ 用户集合 / 团队集合（数据） │──── 抽取 ──▶│ 文档项（CollectionDoc + RequestDoc） │
└──────────────────────────────┘         └──────────────────────────────────┘
                                                   │
                                                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  渲染模板（DocumentationContent / RequestPreview / Curl / Auth / Headers …）│
└──────────────────────────────────────────────────────────────────────────┘
                                                   │
                                                   ▼
┌──────────────────────────────────┐       ┌────────────────────────────────┐
│  createPublishedDoc / update      │──────▶│  GET /published-docs/:slug/:v  │
│  GraphQL Mutation → Prisma 落库   │       │  对外无鉴权公开访问              │
└──────────────────────────────────┘       └────────────────────────────────┘
```

核心角色：

| 角色 | 位置 | 职责 |
| --- | --- | --- |
| 前端集合读取与编辑 | `hoppscotch-common/src/components/collections/documentation/*` | 读取集合/请求数据，渲染预览与编辑界面 |
| 前端发布组件 | `PublishDocModal / PublishDocForm / PublishDocSnapshotPreview` | 填表单、预览快照、触发 GraphQL 发布 |
| 前端服务层 | `services/documentation.service.ts`、`composables/useDocumentationWorker.ts` | 暂存未保存描述、Web Worker 扁平化集合、管理已发布文档缓存 |
| 前端 GraphQL/REST 客户端 | `helpers/backend/queries/PublishedDocs.ts`、`mutations/PublishedDocs.ts` | 与后端通信 |
| 后端 GraphQL Resolver | `hoppscotch-backend/src/published-docs/published-docs.resolver.ts` | 暴露 `publishedDoc / createPublishedDoc / updatePublishedDoc / deletePublishedDoc` |
| 后端服务 | `published-docs.service.ts` | slug 生成、文档快照、鉴权、环境变量绑定、持久化 |
| 后端 REST 控制器 | `published-docs.controller.ts` | 对外发布路由 `/published-docs/:slug[/:version]` |
| 对外展示页 | `hoppscotch-common/src/pages/view/_id/_version.vue` + `DocumentationContent / Header.vue` | 用户访问发布链接时的展示 |

---

## 二、抽取规则：从集合到“文档项”

### 1. 文档项的基本类型

`services/documentation.service.ts:29-64` 定义了两类文档项：

- **集合级** `CollectionDocumentationItem`：绑定集合路径或 ID，携带 `collectionData: HoppCollection`。
- **请求级** `RequestDocumentationItem`：绑定父集合 ID + 文件夹路径，并根据 `requestID`（团队）或 `requestIndex`（个人）定位请求，携带 `requestData: HoppRESTRequest`。

两者共享 `BaseDocumentationItem`：

```ts
{
  id,
  documentation,            // Markdown 描述文本
  isTeamItem, teamID?,
  // 其余为定位信息
}
```

### 2. 抽取入口：`useDocumentationWorker`

`composables/useDocumentationWorker.ts:124-162` 暴露 `processDocumentation(collection, pathOrID, isTeamCollection)`。它不直接处理，而是把请求推入队列，通过 **Web Worker** 异步扁平化：

```ts
worker.postMessage({
  type: "GATHER_DOCUMENTATION",
  collection: JSON.stringify(nextItem.collection),
  pathOrID: nextItem.pathOrID,
  isTeamCollection: nextItem.isTeamCollection,
})
```

Worker 实现位于 `helpers/workers/documentation.worker.ts`。

### 3. Web Worker 的遍历算法（`gatherAllItems`）

- **两阶段遍历**：
  1. 第一遍 `countItems` 统计集合里所有 `requests` + 递归 `folders` 的数量，作为 `totalCount`。
  2. 第二遍 `processFoldersAsync` + 根级 requests 遍历递归扁平化。
- **路径生成**（`documentation.worker.ts:111-122`）：
  - 团队集合：`pathSegment = folderId`；
  - 个人集合：`pathSegment = folderIndex.toString()`；
  - 最终拼接为 `baseCollectionPath/currentFolderPath/pathSegment` 作为 `pathOrID`。
- **批处理**：按 `BATCH_SIZE = 20` 批量，每 10% 或 `folderIndex % 5 === 0` 才发一次 progress，避免主线程消息洪泛。
- **每一项的产物**（`DocumentationItem`，见 `composables/useDocumentationWorker.ts:4-13`）：
  ```ts
  {
    type: "folder" | "request",
    item: HoppCollection | HoppRESTRequest,
    parentPath, id,
    pathOrID?, folderPath?, requestIndex?, requestID?,
  }
  ```

### 4. 描述文本的“暂存 → 保存”流程

- **暂存**：用户编辑描述后，`RequestPreview.vue:300-343` 中 `handleBlur` 把内容写入 `DocumentationService.setRequestDocumentation(...)`；集合描述由 `index.vue:138-155` 中的 `setCollectionDocumentation` 写入。键分别为 `request_${id}` / `collection_${id}`。
- **批量保存**：`index.vue:665-725` 的 `saveDocumentation` 读取 `documentationService.getChangedItems()`，逐个区分集合/请求：
  - 团队集合：调用 `updateTeamCollection(collectionId, {auth, headers, variables, description, preRequestScript, testScript})`（`index.vue:730-773`）。
  - 个人集合：`editRESTCollection(parseInt(pathOrID), updatedCollection)` 或 `editRESTFolder`。
  - 团队请求：`updateTeamRequest(requestID, { request: JSON.stringify(updatedRequest), title })`。
  - 个人请求：`editRESTRequest(folderPath, requestIndex, updatedRequest)`。
- **脏状态保护**：关闭弹窗时，若 `documentationService.hasChanges` 为真，弹出 toast 提示 `unsaved_changes`，二次点击才真正关闭（`index.vue:955-986`）。

### 5. 继承属性（Inherited Properties）

`pages/view/_id/_version.vue:117-199` 的 `flattenCollection` 实现 **auth / headers / variables / scripts** 的递归级联：

- 若 `collection.auth.authType === "inherit"`，沿树向上沿用 `inheritedProperties.auth`，否则以当前集合的 auth 覆盖并标记 `parentID/parentName`。
- headers 与 variables 追加到数组，scripts 只有 `hasActualScript` 非空时才追加。
- 每一个 folder/request 都会被 push 进 `items`，携带当时的 `inheritedProperties`。

这使得单个请求的文档页能展示“最终生效”的 auth/headers/variables，而不仅是它自身定义。

---

## 三、生成模板：文档预览与发布时的内容组织

### 1. 文档项拆分

`components/collections/documentation/index.vue` 在弹窗里同时承载两种模式：

- `CollectionsDocumentationPreview`（集合级） — 当 `currentCollection` 存在。
- `CollectionsDocumentationRequestPreview`（请求级） — 当 `request` 存在。

### 2. 请求文档模板（`RequestPreview.vue`）

按固定顺序渲染 8 个区块，每个区块一个专门的 Section 组件：

1. `CollectionsDocumentationSectionsCurlView` — cURL 命令（CodeMirror 高亮）。
2. `CollectionsDocumentationSectionsAuth` — 鉴权，叠加继承 auth。
3. `CollectionsDocumentationSectionsHeaders` — 叠加继承 headers。
4. `CollectionsDocumentationSectionsParameters` — URL / Query 参数。
5. `CollectionsDocumentationSectionsVariables` — 请求级变量。
6. `CollectionsDocumentationSectionsRequestBody` — 请求体。
7. `CollectionsDocumentationSectionsResponse` — 示例响应（从 `request.responses` 中取出 code/headers/body）。
8. 顶部区域显示 **方法 + 名称 + 最终 URL**（通过 `getEffectiveRESTRequest` 解析环境与变量占位符，见 `RequestPreview.vue:207-260`）。

方法名按颜色分类（`getMethodClass`）：GET 绿、POST 蓝、PUT 橙、DELETE 红、PATCH 青。

### 3. 描述编辑器

`CollectionsDocumentationMarkdownEditor` 负责编辑 Markdown，仅在 `blur` 时才回写，避免高频触发 GraphQL 写。

### 4. 快照预览（`PublishDocSnapshotPreview.vue`）

- 从 `existingData.url` 中用正则 `/\/view\/([^/]+)\/([^/]+)/` 提取 `slug` 和 `version`（`PublishDocSnapshotPreview.vue:306-326`）。
- 调用 `getPublishedDocBySlugREST(slug, version)` 拉取 `documentTree`，再通过 `collectionFolderToHoppCollection` 转回 `HoppCollection`，并复用 `flattenCollection` 与 `DocumentationContent` 渲染，实现“所见即所得”。
- 环境变量单独从 `environmentVariables` 字段解析，`translateToNewEnvironmentVariables` 规范化后作为 props 传入。

### 5. 发布表单（`PublishDocForm.vue`）

表单字段：

| 字段 | 校验 / 默认 |
| --- | --- |
| `title` | 非空；未设置时回落到 `collectionTitle` |
| `version` | 正则 `/^[a-zA-Z0-9]+([.-][a-zA-Z0-9]+)*$/`；首次发布默认 `CURRENT` |
| `autoSync` | 布尔，默认首次发布为 `true`（快照/冻结版本为 `false`） |
| `environmentID` | 从 `CollectionsDocumentationEnvironmentPicker` 选择，可选 |
| `metadata` | 字符串化 JSON（保留扩展信息） |

变更检测 `hasChanges`（`PublishDocModal.vue:220-229`）比较 title/version/autoSync/environmentID 与 `existingData`，只有变化时才允许更新。

### 6. 版本切换与“冻结/活跃”概念

- `services/documentation.service.ts:99-107`：
  ```ts
  export const CURRENT_VERSION_TAG = "CURRENT"
  export const isLiveVersion = (doc: { autoSync: boolean }) => doc.autoSync
  ```
- `index.vue:394-396` 的 `findCurrentVersion` 默认选中 `autoSync` 版本，否则兜底到列表最后一个。
- 从 live 切换到 snapshot：`autoSync=true → false` 时后端会把集合导出为 `documentTree` 快照（见下文 `updatePublishedDoc`）。
- 从 snapshot 提升到 live：UI 弹出黄色提示 `snapshot_promote_warning`。

---

## 四、发布路径：从“点击发布”到“对外 URL”

### 1. 前端提交

- **新建**：`index.vue:988-1049 handlePublish` 构造 `CreatePublishedDocsArgs`，若用户附带环境变量则写入 `metadata.environmentVariables`，再调用 `platform.backend.createPublishedDoc(doc)`。成功后把返回的 `id/title/version/autoSync/url/createdOn/updatedOn` 写入 `DocumentationService.setPublishedDocStatus`，并按是否 live 决定是否选中新版本。
- **更新**：`index.vue:1051-1103 handleUpdate` 仅发送 `UpdatePublishedDocsArgs`，通过 `publishedDocId` 更新单个版本；成功后刷新 `existingPublishedData`。
- **删除**：`index.vue:1105-1133 handleDelete` 走 `deletePublishedDoc(id)`，成功后从 `publishedDocsMap` 中移除。

GraphQL 请求体见 `helpers/backend/gql/mutations/CreatePublishedDoc.graphql` 与 `UpdatePublishedDoc.graphql`。

### 2. 后端 Resolver（`published-docs.resolver.ts`）

- `@Query publishedDoc(id)` — 需 `GqlAuthGuard`，已登录用户按 ID 拉。
- `@Query userPublishedDocsList / teamPublishedDocsList` — 分页列出当前用户或团队的所有已发布文档（团队列表还要求 `RequiresTeamRole(VIEWER/EDITOR/OWNER)`）。
- `@Mutation createPublishedDoc(args)` / `updatePublishedDoc(id, args)` / `deletePublishedDoc(id)` — 全部要求 `GqlAuthGuard`。

Resolver 把 Either left 的错误统一 `throwErr(err)`。

### 3. 核心服务（`published-docs.service.ts`）

#### 3.1 创建 `createPublishedDoc(args, user)`

1. `validateWorkspace`：团队场景必须是 OWNER 或 EDITOR。
2. `validateCollection`：确认 `collectionID` 在该 workspace 下存在。
3. `stringToJson(args.metadata)`：失败直接返回错误。
4. `getOrGenerateSlug(collectionID, workspaceType, workspaceID)`：
   - 同集合若已有已发布文档，复用其 `slug`；
   - 否则 `crypto.randomUUID()` 生成新 slug。
5. **是否生成快照**：
   - `autoSync=true`：`documentTree = null`（展示时动态从集合实时导出）。
   - `autoSync=false`：调用 `userCollectionService.exportUserCollectionToJSONObject` 或 `teamCollectionService.exportCollectionToJSONObject` 把整棵集合序列化成 `CollectionFolder` JSON 存进 `documentTree`。
6. **环境绑定**：若 `environmentID` 存在，`fetchEnvironment` 校验归属并取 `name + variables`。
7. `prisma.publishedDocs.create(...)` 落库；唯一键冲突（`[slug, version]`）最多重试 2 次。

#### 3.2 更新 `updatePublishedDoc(id, args, user)`

- 权限校验：`checkPublishedDocsAccess(..., [OWNER, EDITOR])`。
- `documentTree` 行为分叉：
  - `args.autoSync === true`：`documentTree = null`（动态生成）。
  - 从 live 切到 snapshot（`publishedDocs.autoSync=true` 且 `args.autoSync=false`）：重新导出整棵集合做快照。
- 环境：`args.environmentID === null` 表示解绑；有值则重新 `fetchEnvironment`。
- `prisma.publishedDocs.update`，仅对 `!== undefined` 的字段做更新。

#### 3.3 对外访问 `getPublishedDocBySlugPublic(slug, version)`

- 先 `getPublishedDocsVersions(slug)` 拿到所有版本（按 `autoSync desc, createdOn desc` 排序）。
- 若 `version` 为空，取 `allVersions[0].version`。
- 按 `slug_version` 唯一键查 Prisma。
- **关键逻辑**：若 `publishedDocs.autoSync === true`，
  1. 调用 `userCollectionService.exportUserCollectionToJSONObject` 或 `teamCollectionService.exportCollectionToJSONObject` 实时导出；
  2. 若集合已不存在（返回 `USER_COLL_NOT_FOUND` / `TEAM_INVALID_COLL_ID`），**自动删除该 publishedDoc**（清理孤儿）；
  3. 环境变量同样实时按 `environmentID` 再取一遍。
  这意味着“live 版本”始终展示最新集合状态，不会过期。
- 最后 `plainToInstance(PublishedDocs, this.cast(docToReturn, versions))` 产出响应。

#### 3.4 URL 生成

`cast(doc)` 中：
```ts
url: `${this.configService.get("VITE_BASE_URL")}/view/${doc.slug}/${doc.version}`
```
该 URL 通过响应返回给前端，前端再把它展示给用户复制 / 打开。

### 4. 对外 REST 路由（`published-docs.controller.ts`）

```
GET /published-docs/:slug           → 最新版本
GET /published-docs/:slug/:version  → 指定版本
```

- 全程无鉴权（仅 `ThrottlerBehindProxyGuard` 限流），任何人可访问。
- 成功返回 `PublishedDocs`（含 `documentTree` 字符串化 JSON），失败 404。

### 5. 对外展示页（`pages/view/_id/_version.vue`）

- 读取 `route.params.id` 与 `route.params.version`。
- 调用 `getPublishedDocBySlugREST(id, version)` 拿到数据。
- `collectionFolderToHoppCollection(publishedData)` 把 `CollectionFolder` 还原成 `HoppCollection`。
- 再次 `flattenCollection` 扁平化，计算继承属性。
- `DocumentationContent` 渲染左侧导航 + 右侧 `RequestPreview`，`DocumentationHeader` 显示标题、live 指示灯（绿色脉冲小圆点）、版本下拉、环境开关。
- `usePageHead` 注入 OG/Twitter 卡片元信息，支持社交分享。

---

## 五、数据持久化

后端 Prisma 模型（可从 `published-docs.service.ts` 反推）字段：

- `id`、`slug`、`version`（唯一键 `slug_version`）。
- `title`、`metadata`（JsonValue）。
- `collectionID`、`creatorUid`。
- `workspaceType`、`workspaceID`。
- `autoSync: boolean`。
- `documentTree: JsonValue | null`（snapshot 版本才有，live 版本为 null）。
- `environmentID`、`environmentName`、`environmentVariables: JsonValue | null`。
- `createdOn`、`updatedOn`。

---

## 六、关键细节与注意点

1. **孤儿文档清理**：`cleanupOrphanedPublishedDocs` 在列表查询时会剔除并删除那些 `collectionID` 指向已不存在集合的记录，避免用户看到 404 的发布链接。
2. **并发安全**：创建时对 `[slug, version]` 唯一冲突做了最多 2 次重试；前端 `fetchRequestId` 计数器用于取消旧的拉取请求，避免竞态覆盖。
3. **环境安全**：UI 明确提示 `sensitive_data_warning`，但实际仍把 `environmentVariables` 作为 JSON 存入 Prisma 并通过公开 REST 返回，使用时须自行评估敏感数据暴露风险。
4. **权限层级**：创建/更新/删除要求 OWNER 或 EDITOR；团队列表 VIEWER 即可；对外 REST 无鉴权但限流。
5. **版本语义**：
   - `CURRENT` 只是一个标识，真正是否 live 由 `autoSync` 决定；
   - 同一 slug 可以有多个 version，一个集合下只能有一个 slug；
   - UI 对非 live 版本默认以 snapshot 视图模式打开，不允许编辑。

---

## 七、涉及的核心文件清单

| 区域 | 文件 |
| --- | --- |
| 前端文档服务 | `packages/hoppscotch-common/src/services/documentation.service.ts` |
| 前端 Worker | `packages/hoppscotch-common/src/composables/useDocumentationWorker.ts`、`src/helpers/workers/documentation.worker.ts` |
| 前端 UI | `packages/hoppscotch-common/src/components/collections/documentation/index.vue`、`PublishDocModal.vue`、`PublishDocForm.vue`、`PublishDocSnapshotPreview.vue`、`RequestPreview.vue` |
| 前端 Header/Content | `packages/hoppscotch-common/src/components/documentation/Header.vue`（对外页） |
| 对外页 | `packages/hoppscotch-common/src/pages/view/_id/_version.vue` |
| 前端 GraphQL | `packages/hoppscotch-common/src/helpers/backend/queries/PublishedDocs.ts`、`mutations/PublishedDocs.ts`、`gql/mutations/CreatePublishedDoc.graphql`、`UpdatePublishedDoc.graphql`、`DeletePublishedDoc.graphql` |
| 后端 Resolver | `packages/hoppscotch-backend/src/published-docs/published-docs.resolver.ts` |
| 后端 Service | `packages/hoppscotch-backend/src/published-docs/published-docs.service.ts` |
| 后端 REST 发布 | `packages/hoppscotch-backend/src/published-docs/published-docs.controller.ts` |
| 后端类型 | `published-docs.model.ts`、`input-type.args.ts`、`published-docs.module.ts` |

以上即为 Hoppscotch 团队集合 “自动抽 API 文档并发布” 的端到端实现：**前端用 Web Worker 扁平化集合 + 暂存编辑状态 + 分块渲染；后端用 Prisma 存储快照/动态导出两种模式；REST 路由 `/published-docs/:slug[/:version]` 对外无鉴权公开；前端 `/view/:id/:version` 负责最终展示。**
