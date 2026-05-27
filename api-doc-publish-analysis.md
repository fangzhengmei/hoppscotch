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

## 五、documentTree 与页面展示区块的字段映射

### 1. `documentTree` 的数据形状（`CollectionFolder`）

后端 `documentTree` 存储的是一棵 `CollectionFolder` JSON 树（`helpers/backend/queries/PublishedDocs.ts:60-67`）：

```ts
type CollectionFolder = {
  id?: string
  name: string
  folders: CollectionFolder[]
  requests: any[]                         // 原始 HoppRESTRequest 序列化形态
  data?: string                           // 字符串化 JSON：auth/headers/variables/description/preRequestScript/testScript
}
```

其中 `data` 字段的典型内容为 `CollectionDataProps`：

```ts
{ auth, headers, variables, description, preRequestScript, testScript }
```

`data` 可能为 `null`/`""`，此时由 `parseCollectionDataFromString`（`queries/PublishedDocs.ts:95-124`）回落到“继承 auth + 空 headers/variables/空 scripts”的默认值。

### 2. `collectionFolderToHoppCollection` 的反序列化链路

`queries/PublishedDocs.ts:131-156` 展示了 `CollectionFolder → HoppCollection` 的映射：

| `CollectionFolder` 字段 | `HoppCollection` 字段 | 备注 |
| --- | --- | --- |
| `name` | `name` | 直接映射 |
| `id` | `id` | 可选，直接映射 |
| `folders[]` | `folders[]` | 递归调用 `collectionFolderToHoppCollection` |
| `requests[]` | `requests[]` | 每个元素经过 `translateToNewRequest` 规范化为 `HoppRESTRequest` |
| `data.auth` | `auth` | 默认 `{ authType: "inherit", authActive: true }` |
| `data.headers` | `headers` | 默认 `[]` |
| `data.variables` | `variables` | 默认 `[]` |
| `data.description` | `description` | 默认 `null` |
| `data.preRequestScript` | `preRequestScript` | 默认 `""` |
| `data.testScript` | `testScript` | 默认 `""` |

### 3. 页面展示区块与字段的映射关系

对外页（`pages/view/_id/_version.vue` + `components/documentation/Content.vue` + `RequestPreview.vue`）把 `collectionFolderToHoppCollection` 产出的 `HoppCollection` 经过 `flattenCollection` 扁平化为 `DocumentationItem[]`，再按固定顺序渲染。下表列出了展示区块到原始字段的溯源：

| 区块 | 组件 | 数据来源 |
| --- | --- | --- |
| 左侧集合导航 | `CollectionsDocumentationCollectionStructure` | `collectionData.name`、`folders[].name`、`requests[].name` |
| 集合描述 | `CollectionsDocumentationCollectionPreview` | `collectionData.description`（来自 `data.description`） |
| 请求标题 + 方法标签 | `RequestPreview.vue` | `request.name`、`request.method` |
| 最终 URL | `RequestPreview.vue` 中 `getFullEndpoint` | `request.url` + `params` + 环境/集合/请求变量（经 `getEffectiveRESTRequest` 替换） |
| 请求 Markdown 描述 | `CollectionsDocumentationMarkdownEditor` | `request.description` |
| cURL 视图 | `CollectionsDocumentationSectionsCurlView` | `request.method/url/headers/body/auth` + 继承属性 + `environmentVariables` |
| Auth 区块 | `CollectionsDocumentationSectionsAuth` | `request.auth` + `inheritedProperties.auth`（由 `flattenCollection` 级联自祖先集合的 `data.auth`） |
| Headers 区块 | `CollectionsDocumentationSectionsHeaders` | `request.headers` + `inheritedProperties.headers`（祖先集合 `data.headers` 合并） |
| Parameters 区块 | `CollectionsDocumentationSectionsParameters` | `request.params` |
| Variables 区块 | `CollectionsDocumentationSectionsVariables` | `request.requestVariables` |
| Request Body 区块 | `CollectionsDocumentationSectionsRequestBody` | `request.body` |
| Response 区块 | `CollectionsDocumentationSectionsResponse` | `request.responses`（示例 code/headers/body） |
| 顶部 Header 标题、live 指示 | `DocumentationHeader`（`components/documentation/Header.vue`） | `publishedDoc.title / version / autoSync / environmentName` |

注意：**Auth/Headers/Variables 区块展示的并非当前节点自身字段，而是 `flattenCollection` 递归级联后的“最终生效值”**，这在 `pages/view/_id/_version.vue:124-172` 中通过 `inheritedProperties` 累积实现——每访问一个 folder/request，都把自身字段 push 到数组里，使渲染层可以用“最近一次非空值”覆盖祖先。

---

## 六、请求顺序与锚点的稳定生成

### 1. 请求顺序的来源与稳定性

文档的渲染顺序由两处共同决定：

1. **`documentTree` 自身的数组顺序**：后端在 `exportUserCollectionToJSONObject` / `exportCollectionToJSONObject` 时，按用户维护的集合数据顺序序列化 `folders`、`requests` 数组。因此，发布快照本身就是顺序的“事实来源”。
2. **`flattenCollection` 的 DFS 顺序**：`pages/view/_id/_version.vue:174-199` 先遍历 `folders`（按数组索引），再遍历 `requests`。这是稳定的深度优先顺序——文件夹先于兄弟请求，子文件夹先于父文件夹的兄弟请求。

同一份快照在不同客户端访问时，顺序必然一致。**live 版本（`autoSync=true`）每次访问会重新导出**，因此顺序随集合内容变化；若需要稳定顺序，应使用 snapshot 版本（`autoSync=false`）。

### 2. 锚点的三级降级策略

锚点在三处生成/消费：

- **生成（`flattenCollection`）**：`pages/view/_id/_version.vue:177-194`
  ```ts
  id: folder.id || folder._ref_id || `folder-${folder.name}`
  id: request.id || request._ref_id || `request-${request.name}`
  ```
  优先级依次为 `id` → `_ref_id` → 基于 `name` 的字符串兜底。团队集合通常有稳定 UUID；个人集合 `_ref_id` 由前端在创建时生成，迁移/导入后保持；最终兜底是 `request-<name>` / `folder-<name>`，若重名会冲突（见下方约束）。

- **侧边栏键值（`CollectionStructure.vue`）**：`getFolderId/getRequestId` 使用 `generateFallbackId`（`CollectionStructure.vue:139-149`）：
  ```ts
  item.id || item._ref_id || `${prefix}-${name.replace(/\s+/g, "-").toLowerCase()}-${index}`
  ```
  兜底时把空白转 `-` 并小写化，再追加兄弟索引，保证同一层级内即使重名也不会冲突。

- **渲染容器的锚点 id**：
  - 文档弹窗（`Preview.vue:98-101`）：`id="doc-item-${item.id}"`
  - 对外页（`Content.vue:38-41`）：`id="doc-item-${item.id}"`，并带 `scroll-mt-14` 以便浏览器原生锚点和 `scrollIntoView` 都能避开 sticky header。

### 3. 锚点的消费路径

1. **侧边栏点击 → URL → 滚动**（`Content.vue:116-141`）：选择请求或文件夹时若 `updateUrlOnSelect=true`，调用 `router.replace({ query: { ...route.query, section: id } })` 写入 URL。
2. **页面加载 → 根据 `?section=` 滚动**（`Content.vue:218-225`）：`onMounted` 读取 `route.query.section` 后执行 `scrollToItem`。
3. **`scrollToItem` 的安全查找**（`Content.vue:148-173`）：
   - 100ms 延迟等待 Vue 渲染完成；
   - 手动 `escapeId` 转义 CSS 选择器中的特殊字符（`!"#$%&'()*+,./:;<=>?@[\]^{|}~` 前加反斜杠）；
   - 在 `mainContentRef` 容器内用 `querySelector` 查找，避免与页面其他 id 冲突；
   - 用 `getBoundingClientRect` 计算相对滚动偏移并减去 14px 以匹配 `scroll-mt-14`。
4. **兜底：按名称+类型查找**（`Content.vue:178-188` `scrollToItemByName`）：当 `id` 缺失时，在 `allItems` 中按 `name + type` 线性查找再滚动。

### 4. 稳定性风险与约束

| 场景 | 是否稳定 | 说明 |
| --- | --- | --- |
| 同一 snapshot 版本，重复访问 | ✅ | `documentTree` 固定，`id` 不变 |
| live 版本，集合内容不变 | ✅ | 重新导出得到相同顺序和 id |
| live 版本，集合重命名/移动 | ⚠️ | `name` 兜底锚点会变；有 `id/_ref_id` 时仍稳定 |
| 两个同级请求同名且都无 id | ❌ | `request-<name>` 兜底会冲突；此时依赖 `CollectionStructure.vue` 的 `-index` 后缀策略，但该策略只用于侧边栏 key，未用于 `flattenCollection`，渲染层可能出现重复 `doc-item-*` |
| 从不同入口（文档弹窗/对外页）打开 | ✅ | 都使用 `doc-item-${id}` |

建议：对外分享锚点时，优先确保集合中的请求/文件夹有稳定 `id`（团队集合自动满足，个人集合 `_ref_id` 也会在首次创建时生成）。

---

## 七、环境变量注入失败或缺失的回退与降级

公开页（`pages/view/_id/_version.vue` + `RequestPreview.vue` + `getEffectiveRESTRequest`）对环境变量做了多层降级处理，任一层失败都不会导致页面崩溃。

### 1. 后端写入阶段的回退

- **未绑定环境**（`environmentID` 为空或 `null`）：`published-docs.service.ts:591-604` 创建时 `environmentName = null`、`environmentVariables = null`，`cast` 时 `environmentName ?? null`（`published-docs.service.ts:89`）。
- **环境已被删除或跨 workspace**：`fetchEnvironment`（`published-docs.service.ts:101-142`）返回 `TEAM_ENVIRONMENT_NOT_FOUND / USER_ENVIRONMENT_NOT_FOUND / PUBLISHED_DOCS_FORBIDDEN_ENVIRONMENT_ACCESS`，并以 Either left 向上传递：
  - `createPublishedDoc`：直接返回 left，不写入记录；
  - `updatePublishedDoc`（切换环境时）：直接返回 left，不更新；
  - `getPublishedDocBySlugPublic`（live 版本重取）：直接返回 left，**对外返回错误而不是回退到快照中的旧变量**，因此公开页会进入 `fetchDocs` 的 error 分支，显示 `documentation.publish.not_found` 而非降级页面。

### 2. 前端解析阶段的回退（`pages/view/_id/_version.vue:244-275`）

```
rawEnvVars (从响应中取)
   │
   ├─ 为空 / 为 null → 跳过，parsedEnvironmentVariables = []
   │
   └─ 非空
        ├─ 为字符串 → JSON.parse → Array.isArray ?
        │                                 ├─ 是 → map + translateToNewEnvironmentVariables + currentValue 兜底
        │                                 └─ 否 → parsedEnvironmentVariables = []
        │
        └─ 已是对象 → 同上
```

- **JSON 解析失败**：`try/catch` 兜底到 `parsedEnvironmentVariables = []`，仅 `console.error`，不抛错；
- **非数组**：同样兜底到 `[]`；
- **条目规范化失败**：`translateToNewEnvironmentVariables` 返回的对象可能缺少 `currentValue`，代码显式兜底：`currentValue: normalized.currentValue || normalized.initialValue`。

### 3. 使用阶段的回退（`RequestPreview.vue:207-260`）

`getEffectiveRequest` 组装环境时按优先级拼接变量：

```ts
env.variables = [
  ...requestVariables,                 // 1. 请求自身 requestVariables（active 才加入）
  ...collectionVariables,              // 2. inheritedProperties.variables 级联（来自祖先集合 data.variables）
  ...(props.environmentVariables || []) // 3. 发布时绑定的环境变量（props 为 undefined 时为空数组）
]
```

- **`environmentVariables` 为空**：不添加任何条目，URL 中出现的 `<<variable>>` 占位符会原样保留（`getEffectiveRESTRequest` 对未命中的占位符不替换）；
- **集合级变量注入失败**：`inheritedProperties` 为 `undefined` 时 `collectionVariables = []`；
- **请求级变量激活过滤**：`requestVariable.active === false` 的条目替换为 `{}`（无 key），不参与查找；
- **`getCurrentValue` 失败**：对 `secret` 环境变量若 `currentValue` 为空，兜底到 `initialValue`（`RequestPreview.vue:239-244`）。

### 4. 顶部开关的语义

- `environmentEnabled` 初始值：`!!environmentName`（`pages/view/_id/_version.vue:271`）——若后端没写 `environmentName`，开关默认关闭；
- 用户关闭开关：`environmentVariables.value = []`，此时所有三级变量源里只有请求/集合变量生效；
- 开关切换不重新请求后端，只切换前端是否把已解析的变量传给 `RequestPreview`。

### 5. 最终行为总结

| 故障场景 | 公开页表现 | 可见的占位符 |
| --- | --- | --- |
| 未绑定环境 | 正常渲染 | `<<name>>` 原样显示 |
| 环境绑定但 JSON 解析失败 | 正常渲染 | `<<name>>` 原样显示 |
| 环境绑定但值缺失 | 正常渲染 | 未命中的 `<<name>>` 原样显示 |
| 环境被删除且版本是 snapshot | 正常渲染（snapshot 中的变量仍存在，但已过期） | 可能显示过期值 |
| 环境被删除且版本是 live | 显示 `documentation.publish.not_found` 错误页 | — |
| 环境跨 workspace（越权） | 同 live 版本环境被删除 | — |

这意味着“环境变量注入失败”时系统以 **功能降级而非可用性降级** 为主：页面仍可访问、集合结构与描述完整，仅动态 URL 中的占位符无法被替换。若需要公开页不泄露占位符文本，可在发布时用 snapshot 模式并确保环境当时已绑定。

---

## 八、异常路径与盲点分析

### 1. 版本切换时环境变量解析结果重置的逻辑与跨版本数据残留

#### 重置逻辑

`pages/view/_id/_version.vue:297-304` 监听路由参数变化触发 `fetchDocs`：

```ts
watch(
  () => [route.params.id, route.params.version],
  ([newId, newVersion], [oldId, oldVersion]) => {
    if (newId !== oldId || newVersion !== oldVersion) {
      fetchDocs(newId as string, newVersion as string)
    }
  }
)
```

`fetchDocs` 每次调用会：
1. `loading.value = true`、`error.value = null`（`pages/view/_id/_version.vue:208-209`）；
2. 重新调用 `getPublishedDocBySlugREST` 获取新版本数据；
3. 重新解析 `rawEnvVars` 并赋值给 `parsedEnvironmentVariables`（`pages/view/_id/_version.vue:244-263`）；
4. 重置 `environmentEnabled.value = !!environmentName.value`（`pages/view/_id/_version.vue:271`）；
5. 重置 `environmentVariables.value` 为新的解析结果。

这意味着：**环境变量解析结果每次版本切换都会完全重置**，旧版本的解析值不会保留。

#### 跨版本数据残留的触发条件

并非所有状态都会重置，存在两处残留风险：

| 状态变量 | 重置策略 | 残留风险 |
| --- | --- | --- |
| `availableVersions` | 仅在 `availableVersions.value.length === 0` 时赋值（`pages/view/_id/_version.vue:265-267`） | ✅ **残留：** 首次进入页面时填充后，后续版本切换不会更新。若不同版本的后端返回 `versions` 字段有差异（例如新创建的版本在旧版本的 `versions` 列表中不存在），版本下拉框会始终显示首次加载时的列表。 |
| `parsedEnvironmentVariables` | 每次 `fetchDocs` 都会重新赋值 | ❌ 无残留 |
| `environmentEnabled` | 每次 `fetchDocs` 都会重置为 `!!environmentName` | ⚠️ **间接残留：** 用户手动关闭环境开关后切换版本，开关会被重置为新版本的 `environmentName` 是否存在，而非保持用户的上一次选择。 |
| `collectionData` | 每次 `fetchDocs` 都会重新 `collectionFolderToHoppCollection` | ❌ 无残留 |

**典型残留场景**：用户先访问版本 A（有 3 个历史版本），再切换到版本 B（该 slug 下实际已有 5 个历史版本）。由于 `availableVersions` 仅在首次填充，版本下拉框仍只显示 3 个版本，用户无法切换到新增的另外 2 个版本，除非刷新页面。

---

### 2. 缺少稳定 ID 时 section 深链不会写入 URL 的原因与刷新影响

#### 深链写入的前置条件

`components/documentation/Content.vue:116-141` 中 `handleRequestSelect/handleFolderSelect` 的逻辑：

```ts
const requestId = request.id || (request as any)._ref_id
if (requestId) {
  scrollToItem(requestId)
  if (props.updateUrlOnSelect) {
    router.replace({ query: { ...route.query, section: requestId } })
  }
} else {
  scrollToItemByName(request.name, "request")  // 不写 URL
}
```

**只有当 `id` 或 `_ref_id` 任一存在时，才会将 `section` 写入 URL query**；否则仅通过 `scrollToItemByName` 做本地滚动，不修改 URL。

#### 不写入 URL 的设计原因

`scrollToItemByName` 使用 `name` 作为查找键（`Content.vue:178-188`）：

```ts
const item = props.allItems.find(
  (item) => item.item.name === name && item.type === type
)
```

`name` 在集合中**不保证唯一性**，同一层级下可能出现重名的 folder/request，无法作为稳定的 URL 锚点。如果强行把 `name` 写入 URL，刷新后可能定位到错误的同名条目。

#### 刷新后的定位变化

当请求/文件夹缺少稳定 ID 时：
1. **点击侧边栏**：页面会滚动到目标位置，但 URL 不变化（无 `?section=`）；
2. **刷新页面**：URL 中无 `section` 参数，`onMounted` 中的 `scrollToItem(route.query.section)` 不会执行（`Content.vue:218-225`），页面停留在顶部；
3. **用户体验**：刷新后需要重新在侧边栏中手动查找之前浏览的位置，深链失效。

此外，即使有 ID，但 ID 在不同版本间发生变化（例如重新导入集合导致 `_ref_id` 重新生成），旧版本的深链在新版本中也会失效，`scrollToItem` 找不到对应元素时仅 `console.error`，不做回退滚动。

---

### 3. 版本切换链接在 fallback 分支中丢失 version 的触发条件与影响

#### 正常分支与 fallback 分支

`components/documentation/Header.vue:219-233` 的 `navigateToVersion`：

```ts
const navigateToVersion = (ver: PublishedDocVersion) => {
  if (ver.version === props.publishedDoc?.version) return

  try {
    const url = new URL(ver.url, window.location.origin)
    router.push(url.pathname)
  } catch {
    // Fallback: use regex to extract the path
    const match = ver.url.match(/\/view\/([^/]+)/)
    if (match) {
      router.push(match[0])
    }
  }
}
```

- **正常分支**：`new URL(ver.url, window.location.origin)` 解析成功，取 `url.pathname`（包含完整的 `/view/<slug>/<version>`）。
- **Fallback 分支**：`new URL` 抛出异常时，用正则 `/\/view\/([^/]+)/` 匹配。

#### version 丢失的触发条件

正则 `/\/view\/([^/]+)/` 的问题在于：**它只匹配到 `/view/<slug>`，不捕获后续的 `/<version>`**。`[^/]+` 匹配到第一个 `/` 就停止。

触发 `new URL` 异常并进入 fallback 分支的可能场景：

1. **`ver.url` 格式异常**：后端返回的 `url` 字段不是合法 URL（例如缺少协议、包含未编码的特殊字符）；
2. **`ver.url` 为相对路径**：某些部署环境下 `VITE_BASE_URL` 配置缺失或为空，`cast` 生成的 `url` 形如 `/view/<slug>/<version>` 但没有协议前缀；
3. **`ver.url` 包含 Unicode 字符未编码**：如版本名中包含中文、emoji 等字符且未做 URI 编码。

#### 影响

1. **跳转目标错误**：fallback 分支会导航到 `/view/<slug>` 而不是 `/view/<slug>/<version>`；
2. **版本自动回退到最新**：后端 `getPublishedDocBySlugPublic` 在 `version` 为 `null` 时会返回 `allVersions[0].version`（按 `autoSync desc, createdOn desc` 排序，即最新版本），用户点击旧版本却看到最新版本的内容；
3. **状态不一致**：URL 中缺少 version 参数，导致浏览器历史记录、书签、分享链接都指向“当前最新版本”而非用户实际选择的版本；
4. **无错误提示**：整个过程静默失败，用户只会看到页面内容变化但不知道版本被替换。

**防护建议**：后端 `cast` 方法应确保 `url` 是合法的绝对 URL；前端 fallback 正则应补全 version 捕获（例如 `/\/view\/([^/]+)(?:\/([^/]+))?/`），在无法解析时给出提示而非静默降级。

---

## 九、数据持久化

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

## 九、关键细节与注意点

1. **孤儿文档清理**：`cleanupOrphanedPublishedDocs` 在列表查询时会剔除并删除那些 `collectionID` 指向已不存在集合的记录，避免用户看到 404 的发布链接。
2. **并发安全**：创建时对 `[slug, version]` 唯一冲突做了最多 2 次重试；前端 `fetchRequestId` 计数器用于取消旧的拉取请求，避免竞态覆盖。
3. **环境安全**：UI 明确提示 `sensitive_data_warning`，但实际仍把 `environmentVariables` 作为 JSON 存入 Prisma 并通过公开 REST 返回，使用时须自行评估敏感数据暴露风险。
4. **权限层级**：创建/更新/删除要求 OWNER 或 EDITOR；团队列表 VIEWER 即可；对外 REST 无鉴权但限流。
5. **版本语义**：
   - `CURRENT` 只是一个标识，真正是否 live 由 `autoSync` 决定；
   - 同一 slug 可以有多个 version，一个集合下只能有一个 slug；
   - UI 对非 live 版本默认以 snapshot 视图模式打开，不允许编辑。

---

## 十一、涉及的核心文件清单

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
