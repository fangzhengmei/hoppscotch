# GraphQL Schema Introspection、缓存与请求体生成

本文从代码实现角度梳理 Hoppscotch 中 GraphQL schema introspection 的触发机制、schema 的缓存与派生计算、以及如何从 schema 生成可执行的请求体。

---

## 1. 整体架构总览

```
用户点击 "Connect" ──▶ Request.vue ──▶ connect()
                                            │
                                            ▼
                                      getSchema()  ◀── 轮询 (7s)
                                        │
                                        ▼
                              POST getIntrospectionQuery()
                                        │
                                        ▼
                              buildClientSchema(response.data)
                                        │
                                        ▼
                              connection.schema (reactive)
                                        │
                          ┌─────────────┼─────────────┐
                          ▼             ▼             ▼
                    queryFields   mutationFields   subscriptionFields
                    graphqlTypes  schemaString  ... (computed)
                          │
                          ▼
                    DocExplorer / SchemaDocumentation (UI 渲染)
                          │
                    用户点击字段/参数
                          ▼
                    useExplorer().push()  ──▶  navStack 更新
                          │
                          ▼
                    useQuery().handleAddField / handleAddArgument
                          │
                          ▼
                    processOperation()  ──▶  AST 构建/合并
                          │
                          ▼
                    print(document)  ──▶  更新编辑器 query 字符串
                          │
                    用户点击 "Run"
                          ▼
                    runGQLOperation()  ──▶  GQLRequest.toRequest()
                          │                    │
                          ▼                    ▼
                    KernelInterceptor    { query, variables } → JSON body
```

核心文件位置：

| 职责 | 文件路径 |
|------|---------|
| 连接管理 + introspection + 请求执行 | `packages/hoppscotch-common/src/helpers/graphql/connection.ts` |
| Explorer 导航栈 | `packages/hoppscotch-common/src/helpers/graphql/explorer.ts` |
| 查询 AST 操作（添加字段/参数） | `packages/hoppscotch-common/src/helpers/graphql/query.ts` |
| Kernel 层请求序列化 | `packages/hoppscotch-common/src/helpers/kernel/gql/request.ts` |
| Kernel 层响应反序列化 | `packages/hoppscotch-common/src/helpers/kernel/gql/response.ts` |
| 连接按钮 UI 入口 | `packages/hoppscotch-common/src/components/graphql/Request.vue` |
| 查询编辑器 + 运行按钮 | `packages/hoppscotch-common/src/components/graphql/Query.vue` |
| 请求选项面板（组装 runGQLOperation 参数） | `packages/hoppscotch-common/src/components/graphql/RequestOptions.vue` |
| Schema 文档浏览器 | `packages/hoppscotch-common/src/components/graphql/DocExplorer.vue` |

---

## 2. Introspection 触发机制

### 2.1 用户触发入口

**`Request.vue:93-98`** — URL 输入栏旁的 Connect/Disconnect 按钮：

```ts
const onConnectClick = () => {
  if (!connected.value) {
    gqlConnect()   // ← 发起连接
  } else {
    disconnect()   // ← 断开连接
  }
}
```

`gqlConnect()` 收集当前 tab 的 URL、请求头、认证信息后调用 `connect()`：

```ts
const gqlConnect = () => {
  connect({
    url: url.value,
    request: tabs.currentActiveTab.value.document.request,
    inheritedHeaders,
    inheritedAuth: ...,
  })
}
```

### 2.2 connect() — 连接与首次 introspection

**`connection.ts:186-220`**

```ts
export const connect = async (
  options: ConnectionRequestOptions,
  isRunGQLOperation = false
) => {
  if (connection.state === "CONNECTED") {
    throw new Error("A connection is already running...")
  }

  connection.state = "CONNECTING"

  const poll = async () => {
    try {
      await getSchema(options)                          // ← 执行 introspection
      if (connection.state !== "CONNECTED")
        connection.state = "CONNECTED"
      timeoutSubscription = setTimeout(() => {
        poll()                                          // ← 定时轮询
      }, GQL_SCHEMA_POLL_INTERVAL)                     // 7000ms
    } catch (error) {
      connection.state = "ERROR"
      // ...
    }
  }

  await poll()  // 立即执行一次
}
```

关键设计要点：

1. **首次立即执行**：`connect()` 内部直接 `await poll()`，首次进入时立即发起 introspection 请求。
2. **定时轮询**：成功后通过 `setTimeout` 递归调用 `poll()`，间隔 `GQL_SCHEMA_POLL_INTERVAL = 7000`（7 秒），持续同步远端 schema 变化。
3. **状态机**：`connection.state` 在 `DISCONNECTED → CONNECTING → CONNECTED` 或 `CONNECTING → ERROR` 之间转换，UI 据此显示按钮状态。
4. **递归而非 setInterval**：每次 poll 完成后才安排下一次，保证不会出现并发 introspection 请求。

### 2.3 getSchema() — 实际 introspection 请求

**`connection.ts:239-335`**

```ts
const getSchema = async (options: ConnectionRequestOptions) => {
  // 1. 合并 headers（请求级 + 继承级 + auth headers）
  const finalHeaders: Record<string, string> = {}
  const { authHeaders } = await generateAuthHeader(url, auth)
  runHeaders.forEach(header => {
    if (header.active && header.key !== "") {
      finalHeaders[header.key] = header.value
    }
  })
  Object.assign(finalHeaders, authHeaders)

  // 2. 构建 kernel 请求
  const kernelRequest: RelayRequest = {
    id: Date.now(),
    url: options.url,
    method: "POST",
    version: "HTTP/1.1",
    headers: {
      ...finalHeaders,
      "content-type": "application/json",
    },
    content: content.json(
      { query: getIntrospectionQuery() },   // ← graphql 库提供的标准 introspection 查询
      MediaType.APPLICATION_JSON
    ),
  }

  // 3. 通过 KernelInterceptor 发送
  const kernelInterceptorService = getService(KernelInterceptorService)
  const { response } = kernelInterceptorService.execute(kernelRequest)
  const res = await response

  // 4. 错误处理（fp-ts Either）
  if (E.isLeft(res)) { ... }

  // 5. 解析响应 → 构建 schema
  const data = res.right
  const decoder = new TextDecoder("utf-8")
  const responseText = decoder.decode(data.body.body)
  const introspectResponse = JSON.parse(responseText)
  const schemaData = buildClientSchema(introspectResponse.data)  // ← 核心

  // 6. 更新全局响应式 schema
  connection.schema = schemaData
  connection.error = null
}
```

关键实现细节：

- **`getIntrospectionQuery()`**：来自 `graphql` 库，生成完整的 GraphQL introspection 查询字符串（查询 `__schema` 的所有类型、字段、参数等）。
- **`buildClientSchema()`**：同样来自 `graphql` 库，将 introspection 响应的 JSON 数据转换为客户端可操作的 `GraphQLSchema` 对象。
- **认证透传**：introspection 请求会携带与普通 GraphQL 请求相同的 auth headers（Basic、Bearer、OAuth2、API Key、AWS Signature），确保需要认证的 endpoint 也能完成 introspection。
- **错误传播**：任何异常都会调用 `disconnect()` 重置连接状态。

### 2.4 disconnect() — 断开与清理

**`connection.ts:222-237`**

```ts
export const disconnect = () => {
  clearTimeout(timeoutSubscription)   // 停止轮询
  connection.state = "DISCONNECTED"
  connection.schema = null            // 清空 schema
}

export const reset = () => {
  if (connection.state === "CONNECTED") disconnect()
  connection.state = "DISCONNECTED"
  connection.schema = null
}
```

---

## 3. Schema 缓存与派生计算

### 3.1 全局单一 schema 存储

schema 存储在 `connection` 这个 `reactive` 对象中：

```ts
export const connection = reactive<Connection>({
  state: "DISCONNECTED",
  subscriptionState: new Map<string, SubscriptionState>(),
  socket: undefined,
  schema: null,      // ← GraphQLSchema | null
  error: null,
})
```

这是一个**全局单例**，所有组件共享同一个 `connection` 对象。schema 没有独立的缓存层——`connection.schema` 本身就是缓存，由轮询机制持续更新。

### 3.2 Computed 派生值

schema 变更会自动触发一系列 `computed` 重新计算，供 UI 消费：

**`connection.ts:126-182`**

```ts
// schema 本身的 computed 快捷引用
export const schema = computed(() => connection.schema)

// SDL 格式的 schema 字符串（用于导出/展示）
export const schemaString = computed(() => {
  if (!connection.schema) return ""
  return printSchema(connection.schema)
})

// Query 类型的所有字段
export const queryFields = computed(() => {
  const fields = connection.schema.getQueryType()?.getFields()
  return fields ? Object.values(fields) : []
})

// Mutation 类型的所有字段
export const mutationFields = computed(() => { ... })

// Subscription 类型的所有字段
export const subscriptionFields = computed(() => { ... })

// 所有用户定义的类型（过滤掉 __ 前缀和根类型）
export const graphqlTypes = computed(() => {
  const typeMap = connection.schema.getTypeMap()
  return Object.values(typeMap).filter(type => {
    return (
      !type.name.startsWith("__") &&
      ![queryTypeName, mutationTypeName, subscriptionTypeName].includes(type.name) &&
      (type instanceof GraphQLObjectType ||
       type instanceof GraphQLInputObjectType ||
       type instanceof GraphQLEnumType ||
       type instanceof GraphQLInterfaceType)
    )
  })
})
```

### 3.3 缓存更新策略

| 策略 | 实现 |
|------|------|
| 缓存粒度 | 全局单一 schema（per 连接） |
| 更新触发 | 7 秒轮询 `getSchema()` |
| 更新方式 | 直接赋值 `connection.schema = schemaData`，利用 Vue reactivity 自动传播 |
| 失效条件 | `disconnect()` 时置 `null`；网络错误时 `disconnect()` 并置 `null` |
| 多 tab 共享 | 所有 tab 共享同一 `connection` 对象，切换 tab 时 schema 不变 |

---

## 4. 请求体生成

请求体生成分为两个层面：
1. **从 Schema Explorer 点击 → 生成查询字符串**（AST 层面）
2. **从查询字符串 → 发送 HTTP 请求**（Kernel 层面）

### 4.1 Schema Explorer → 查询字符串

#### 4.1.1 导航栈（ExplorerNavStack）

**`explorer.ts`** 定义了导航栈数据结构：

```ts
type ExplorerNavStackItem = {
  readonly?: boolean
  name: string
  def?: GraphQLNamedType | ExplorerFieldDef   // 关联的 GraphQL 类型定义
}

type ExplorerNavStack = [ExplorerNavStackItem, ...ExplorerNavStackItem[]]
```

初始状态为 `[{ name: "Root" }]`。用户在 Doc Explorer 中浏览时，每一级点击都会 `push` 一个新项。

**导航栈的含义示例**：

```
[Root, Query, user, id]
  ↑     ↑     ↑    ↑
 根   操作类型  字段  子字段

[Root, Mutation, createUser, name]
  ↑      ↑          ↑          ↑
 根   操作类型     字段      参数
```

#### 4.1.2 点击字段 → push → handleAddField

**`Field.vue:59-67`**

```ts
const { push } = useExplorer()
const { handleAddField, isFieldInOperation } = useQuery()

const handleClick = () => {
  push({ name: props.field.name, def: props.field })  // 导航
}

const insertQuery = () => {
  handleAddField(props.field)  // 插入查询
}
```

**`Argument.vue:94-101`**

```ts
const { handleAddArgument, isArgumentInOperation } = useQuery()

const handleClick = () => {
  push({ name: props.arg.name, def: props.arg })
}

const insertQuery = debounce(() => {
  handleAddArgument(props.arg)
}, 50)
```

注意：点击字段名是**导航**（`push`），点击旁边的 `+` 按钮才是**插入查询**（`handleAddField` / `handleAddArgument`）。

#### 4.1.3 handleOperation — 核心查询生成逻辑

**`query.ts:317-375`**

```ts
const handleOperation = (item: ExplorerFieldDef, isArgument = false) => {
  const currentTab = tabs.currentActiveTab.value
  const currentQuery = currentTab.document.request.query || ""
  const selectedOperation = getOperation(cursorPosition)
  const navItems = [...navStack.value, { name: item.name, def: item }]

  const result = processOperation(navItems, selectedOperation, isArgument)

  const newQuery = result.document
    ? print(result.document.definitions[0])
    : "\n"

  // 不同操作类型 → 追加新 operation
  if (!selectedOperation ||
      selectedOperation.operation !== getOperationTypeNode(navItems[1].name) ||
      result.append) {
    updatedQuery.value = currentQuery.trim()
      ? `${currentQuery}\n\n${newQuery}`
      : newQuery
  } else {
    // 同操作类型 → 替换现有 operation
    updatedQuery.value = currentQuery.replace(
      currentQuery.substring(selectedOperation.loc!.start, selectedOperation.loc!.end),
      newQuery
    )
  }
}
```

#### 4.1.4 processOperation — AST 构建与合并

**`query.ts:131-308`**

这是最核心的函数，负责根据导航栈和已有查询 AST 生成新的 `DocumentNode`。

**新建操作（无已有 operation 或操作类型不同）**：

```ts
// 从底向上构建嵌套的 FieldNode 链
let currentSelection = createFieldNode(
  lastItem.name,
  isArgument ? [argumentItem.def] : lastItem.def?.args,
  lastItem.def?.fields?.length > 0
)

for (let i = queryPath.length - 2; i >= 0; i--) {
  const parentField = createFieldNode(item.name, item.def?.args, true)
  parentField.selectionSet!.selections = [currentSelection]
  currentSelection = parentField
}

return {
  document: {
    kind: Kind.DOCUMENT,
    definitions: [{
      kind: Kind.OPERATION_DEFINITION,
      operation: requestedOperationType,
      name: { kind: Kind.NAME, value: queryPath[0].name },
      selectionSet: { kind: Kind.SELECTION_SET, selections: [currentSelection] },
    }],
  },
}
```

**合并到已有操作**：

1. 沿 `queryPath` 逐层在已有 `selectionSet` 中查找同名字段
2. 找到 → 继续深入或删除（toggle 行为）
3. 未找到 → `createFieldNode` 并追加到当前 `selectionSet`
4. 参数操作 → 在字段的 `arguments` 中 toggle（存在则删除，不存在则添加）

**createFieldNode** — 构建 FieldNode AST 节点：

```ts
const createFieldNode = (
  name: string,
  args: readonly GraphQLArgument[] | undefined,
  hasNestedFields = false
): Mutable<FieldNode> => ({
  kind: Kind.FIELD,
  name: { kind: Kind.NAME, value: name },
  arguments: args?.map(arg => createArgumentNode(arg.name, arg.type)) || [],
  directives: [],
  selectionSet: hasNestedFields
    ? { kind: Kind.SELECTION_SET, selections: [] }
    : undefined,
})
```

**createArgumentNode** — 构建带默认值的参数节点：

```ts
const createArgumentNode = (argName: string, type: GraphQLType): ArgumentNode => ({
  kind: Kind.ARGUMENT,
  name: { kind: Kind.NAME, value: argName },
  value: { kind: Kind.STRING, value: getDefaultArgumentValue(type) },
})
```

**getDefaultArgumentValue** — 标量类型的默认值：

```ts
const getDefaultArgumentValue = (type: GraphQLType): string => {
  const namedType = getNamedType(type)
  const defaultValues: Record<string, string> = {
    String: "",
    Int: "0",
    Float: "0.0",
    Boolean: "false",
  }
  return defaultValues[namedType.name] || "null"
}
```

#### 4.1.5 编辑器更新

**`Query.vue:217-228`**

```ts
watch(updatedQuery, async (newQuery) => {
  if (newQuery) {
    gqlQueryString.value = newQuery      // 更新编辑器内容
    await nextTick()
    if (cursorPosition.value) {
      cmQueryEditor.cursor.value = cursorPosition.value  // 移动光标
    }
  }
})
```

### 4.2 查询字符串 → HTTP 请求

#### 4.2.1 用户点击 Run → runGQLOperation

**`RequestOptions.vue:132-174`**

```ts
const runQuery = async (definition: gql.OperationDefinitionNode | null = null) => {
  await runGQLOperation({
    name: request.value.name,
    url: runURL,
    request: request.value,
    inheritedHeaders,
    inheritedAuth: ...,
    query: runQuery,
    variables: runVariables,
    operationName: definition?.name?.value,
    operationType: definition?.operation ?? "query",
  })
}
```

#### 4.2.2 runGQLOperation — 请求执行

**`connection.ts:337-507`**

```ts
export const runGQLOperation = async (options: RunQueryOptions) => {
  // 如果未连接，先自动连接（isRunGQLOperation = true 表示不弹连接错误 toast）
  if (connection.state !== "CONNECTED") {
    await connect({ url, request, inheritedHeaders, inheritedAuth }, true)
  }

  // subscription → 走 WebSocket
  if (operationType === "subscription") {
    return runSubscription(options, finalHeaders)
  }

  // query/mutation → 走 HTTP
  const gqlRequest: HoppGQLRequest = {
    v: 9,
    name: options.name || "Untitled Request",
    url: finalUrl,
    headers: finalHoppHeaders,
    query,
    variables,
    auth: auth ?? request.auth,
  }

  const kernelRequest = await GQLRequest.toRequest(gqlRequest)

  // 注入 operationName
  if (operationName && kernelRequest.content?.kind === "json") {
    kernelRequest.content.content.operationName = operationName
  }

  const kernelInterceptorService = getService(KernelInterceptorService)
  const { response } = kernelInterceptorService.execute(kernelRequest)
  const result = await response
  // ... 解析响应
}
```

#### 4.2.3 GQLRequest.toRequest — 最终请求体序列化

**`helpers/kernel/gql/request.ts:21-47`**

```ts
export const GQLRequest = {
  async toRequest(request: HoppGQLRequest) {
    const headers = {
      ...filterActiveToRecord(request.headers),
      "content-type": "application/json",
    }

    const auth = await pipe(
      transformAuth(request.auth),
      TE.getOrElse(() => T.of<AuthType>(defaultAuth))
    )()

    const variables = await parseVariables(request.variables)

    return {
      id: Date.now(),
      url: request.url,
      method: "POST",
      version: "HTTP/1.1",
      headers,
      auth,
      content: content.json(
        { query: request.query, variables },   // ← 最终 JSON body
        MediaType.APPLICATION_JSON
      ),
    }
  },
}
```

最终发送的 HTTP 请求体结构：

```json
{
  "query": "query Request { method url headers { key value } }",
  "variables": { "id": "1" },
  "operationName": "Request"
}
```

#### 4.2.4 Introspection 请求 vs 普通请求的对比

| | Introspection 请求 | 普通 Query/Mutation 请求 |
|---|---|---|
| 触发方式 | `connect()` → 自动轮询 | 用户点击 Run / 快捷键 |
| 请求体 | `{ query: getIntrospectionQuery() }` | `{ query, variables, operationName }` |
| 请求路径 | 相同 URL | 相同 URL |
| Headers | 合并 auth + 请求头 | 合并 auth + 请求头 |
| 响应处理 | `buildClientSchema()` → 存入 `connection.schema` | `GQLResponse.toResponse()` → 存入 tab response |
| 传输方式 | HTTP POST（通过 KernelInterceptor） | HTTP POST / WebSocket（subscription） |

---

## 5. Subscription 的特殊处理

Subscription 不走 HTTP，而是使用 WebSocket（graphql-ws 协议）：

**`connection.ts:573-644`**

```ts
export const runSubscription = (options: RunQueryOptions, headers?) => {
  const wsUrl = url.replace(/^http/, "ws")
  connection.socket = new WebSocket(wsUrl, "graphql-ws")

  connection.socket.onopen = () => {
    connection.socket?.send(JSON.stringify({
      type: GQL.CONNECTION_INIT,
      payload: headers ?? {},
    }))
    connection.socket?.send(JSON.stringify({
      type: GQL.START,
      id: "1",
      payload: { query, operationName },
    }))
  }

  connection.socket.onmessage = (event) => {
    const data = JSON.parse(event.data)
    switch (data.type) {
      case GQL.CONNECTION_ACK:   // 订阅确认
      case GQL.DATA:             // 推送数据 → 更新 gqlMessageEvent
      case GQL.COMPLETE:         // 完成
    }
  }
}
```

---

## 6. 关键设计总结

1. **Introspection 与连接绑定**：schema 获取不是独立操作，而是"连接"概念的一部分。Connect = 首次 introspection + 持续轮询。

2. **全局单例 schema**：`connection.schema` 是唯一的 schema 存储点，所有组件通过 computed 派生消费，无需额外的缓存管理层。

3. **AST-first 的查询构建**：不是拼接字符串，而是构建 GraphQL AST（`DocumentNode` → `FieldNode` → `ArgumentNode`），最后用 `print()` 序列化。这保证了语法正确性。

4. **Toggle 语义**：再次点击已添加的字段会从查询中移除，再次点击已添加的参数也会移除——不是纯粹的增加操作。

5. **自动连接**：执行查询时如果未连接，`runGQLOperation` 会自动调用 `connect()`，实现"即点即发"的体验。

6. **认证一致性**：introspection 请求与普通请求使用相同的认证逻辑（`generateAuthHeader`），确保 introspection 能通过认证网关。
