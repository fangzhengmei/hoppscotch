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
  if (connection.state !== "CONNECTED") {
    throw new Error("No connections are running to be disconnected")
  }

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

**注意**：`disconnect()` 有前置条件检查——只有当 `connection.state === "CONNECTED"` 时才能正常调用，否则抛出错误。这一细节对理解失败恢复至关重要。

### 2.5 失败恢复机制

#### 2.5.1 首次连接失败的完整状态流转（精确时序）

当用户点击 Connect 但首次 introspection 请求失败时，**精确的代码执行时序**如下：

```
用户点击 Connect
    │
    ▼
connect(options, isRunGQLOperation = false)
    │
    ├─ ① connection.state = "CONNECTING"    ← 初始状态
    │
    ▼
poll() 立即执行（await poll()）
    │
    ├─ try {
    │    ├─ await getSchema(options)
    │    │     │
    │    │     ├─ try {
    │    │     │    ├─ 发送 POST introspection 请求 → 失败（网络错误/401/404/500）
    │    │     │    │
    │    │     │    ├─ ② E.isLeft(res) → true
    │    │     │    │   ├─ ③ connection.state = "ERROR"
    │    │     │    │   ├─ ④ connection.error = { type, message, component }
    │    │     │    │   └─ ⑤ throw new Error(error.message)   ← Error A
    │    │     │    │
    │    │     │    └─（成功路径不会执行）
    │    │     │       connection.schema = schemaData
    │    │     │       connection.error = null
    │    │     │
    │    │     └─ catch (e: any)   ← ⑥ 捕获 Error A
    │    │          ├─ console.error(e)
    │    │          └─ ⑦ disconnect()
    │    │                │
    │    │                ├─ ⑧ 检查：connection.state !== "CONNECTED"
    │    │                │      （当前是 "ERROR"，条件成立）
    │    │                └─ ⑨ throw new Error("No connections are running to be disconnected")   ← Error B
    │    │
    │    └─（Error B 从 getSchema 向上冒泡，未被 getSchema catch 捕获）
    │
    └─ catch (error)   ← ⑩ poll() 的 catch 捕获 Error B
         ├─ ⑪ connection.state = "ERROR"   （已是 ERROR，重复赋值）
         ├─ ⑫ !isRunGQLOperation → toast.error("连接失败")
         └─ ⑬ console.error(error)

         ← ⑭ poll() 执行完毕，不会执行到 setTimeout(poll, 7000)
         ← ⑮ 轮询终止！
```

**各变量的最终状态**：

| 变量 | 初始值 | 变化点 | 最终值 | 说明 |
|------|--------|--------|--------|------|
| `connection.state` | `"DISCONNECTED"` | ① → `"CONNECTING"`；③ → `"ERROR"`；⑪ → `"ERROR"` | `"ERROR"` | 在 E.isLeft 分支已设置为 ERROR，poll catch 中重复设置 |
| `connection.schema` | `null` | **无任何变化** | `null`（首次连接）或旧值（重连失败） | 只有成功路径才会赋值 `connection.schema = schemaData`；失败路径中 `disconnect()` 抛错，`schema = null` 不会执行 |
| `connection.error` | `null` | ④ → `{ type, message, component }` | `{ type, message, component }` | 保存了错误详情供 UI 展示 |
| `timeoutSubscription` | `undefined` | **无任何变化** | `undefined` | 从未设置 setTimeout |
| 轮询 | 未开始 | ⑮ 终止 | 已终止 | 不会有自动重试 |

**核心要点（修正后）**：

1. **二次错误抛出路径**：`getSchema()` 内部 catch 捕获 `E.isLeft` throw 的 Error A → 调用 `disconnect()` → `disconnect()` 因 `state !== "CONNECTED"` 抛出 Error B → Error B 从 `getSchema()` 函数**向上冒泡** → 被 `poll()` 的 catch 捕获。
2. **`connection.schema` 永不修改**：失败路径中，`disconnect()` 抛出异常，因此 `disconnect()` 内部的 `connection.schema = null` **不会被执行**。如果是首次连接，schema 保持 `null`；如果是重连失败，schema 保留之前成功连接时的缓存值。
3. **轮询停止的充要条件**：只要 `poll()` 的 catch 块被执行（无论什么原因），就不会执行 `setTimeout(poll, 7000)`，轮询终止。没有例外。
4. **`disconnect()` 抛错的充要条件**：`disconnect()` 第一行检查 `if (connection.state !== "CONNECTED")`，只要不是 `CONNECTED` 状态就抛错。失败路径中 `state` 已被设为 `"ERROR"`，所以必然抛错。

#### 2.5.2 已连接后某次轮询失败的状态流转

当连接已建立（`state = "CONNECTED"`，有有效 schema），后续某次轮询的 introspection 请求失败：

```
state = "CONNECTED"，schema = <有效对象>
    │
    ▼
setTimeout 触发 poll()
    │
    ▼
getSchema(options) → 请求失败
    │
    ├─ E.isLeft(res) → true
    │   ├─ ① connection.state = "ERROR"    ← 先设为 ERROR！
    │   └─ ② throw new Error(...)           ← Error A
    │
    └─ catch (e)
         ├─ console.error(e)
         └─ ③ disconnect()
              │
              ├─ ④ 检查：connection.state !== "CONNECTED"（当前是 "ERROR"！）
              └─ ⑤ throw new Error("No connections...")   ← Error B

    ↓（冒泡到 poll() catch）
    ↓ 轮询终止
```

**注意**：无论首次失败还是轮询中失败，只要进入 `E.isLeft` 分支，**总是先设置 `state = "ERROR"` 再 throw**。因此后续 `disconnect()` 检查时 `state` 已不是 `"CONNECTED"`，必然抛错。

**各变量的最终状态（轮询失败）**：

| 变量 | 失败前值 | 最终值 | 说明 |
|------|----------|--------|------|
| `connection.state` | `"CONNECTED"` | `"ERROR"` | 被 E.isLeft 分支修改 |
| `connection.schema` | `<有效GraphQLSchema>` | `<有效GraphQLSchema>` | **完全不变**！既不会更新为新值，也不会被置 null |
| `connection.error` | `null` | `{ type, message, component }` | 保存了本次失败的详情 |
| `timeoutSubscription` | `<有效timer ID>` | `<有效timer ID>` | 旧的 timer 已触发，新的 timer 不会设置 |
| 轮询 | 运行中 | 已终止 | 不会再有下一次轮询 |

**重要结论**：已连接后某次轮询失败，**旧的 schema 不会被清除**，仍然可以正常浏览 schema 和生成查询，只是不会再收到 schema 更新。

#### 2.5.3 恢复过程——手动重试

失败后没有自动重试，恢复依赖**用户手动再次点击 Connect 按钮**：

1. 用户看到按钮显示 "Connect"（`state === "ERROR"` 时 UI 按钮文字切换回 Connect）
2. 再次点击 → 调用 `gqlConnect()` → `connect()`
3. `connect()` 第一行检查 `connection.state === "CONNECTED"` → 否（当前是 `"ERROR"`），允许继续
4. 重新开始完整的连接流程（`state = "CONNECTING"` → 立即 `poll()` → ...）
5. 如果这次成功：`state = "CONNECTED"`，`schema` 更新为新值，`error = null`，轮询恢复

#### 2.5.4 `isRunGQLOperation` 标志的精确行为

当 `runGQLOperation()` 自动调用 `connect()` 时，传入 `isRunGQLOperation = true`：

```ts
// connection.ts:337-348
if (connection.state !== "CONNECTED") {
  await connect({ url, request, inheritedHeaders, inheritedAuth }, true)
}
// 无论 connect 成功失败，都会继续执行后续代码！
```

**行为差异**：

| | `isRunGQLOperation = false`（用户点击 Connect） | `isRunGQLOperation = true`（runGQLOperation 自动调用） |
|---|---|---|
| 失败时是否弹 toast | 是（`poll()` catch 中 `toast.error()`） | 否（`if (!isRunGQLOperation)` 跳过） |
| `connect()` 是否抛异常给调用者 | **否**（错误在 `poll()` catch 中被消化） | **否**（同上，错误在 `poll()` catch 中被消化） |
| 后续查询是否执行 | 不适用（用户只是点击 Connect） | **是**！`await connect()` 返回后继续执行请求发送 |

**关键澄清**：`connect()` 本身**永远不会向外抛出异常**，因为 `poll()` 内部有 try-catch 消化所有错误。无论成功失败，`await connect()` 都会正常返回。

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

#### 4.2.5 Variables 组装细节与边界条件

Variables 在 3 个不同阶段被处理，最终以解析后的 JSON 对象形式出现在请求体中：

**阶段 1：编辑器输入 → 字符串存储**

在 Variables 标签页中，用户输入的是 JSON 字符串，直接存储在 `request.variables` 中：

```ts
// 默认值来自 default.ts:25-27
export const getDefaultGQLRequest = (): HoppGQLRequest => ({
  ...
  variables: `{
  "id": "1"
}`,
  ...
})
```

**阶段 2：RequestOptions → runGQLOperation — 透传字符串**

**`RequestOptions.vue:132-158`**

```ts
const runQuery = async (definition: gql.OperationDefinitionNode | null = null) => {
  const runURL = clone(url.value)
  const runQuery = clone(request.value.query)
  const runVariables = clone(request.value.variables)   // ← 仍然是字符串

  await runGQLOperation({
    ...
    variables: runVariables,   // ← 字符串透传
    ...
  })
}
```

**阶段 3：GQLRequest.toRequest — JSON 解析**

**`helpers/kernel/gql/request.ts:12-19, 33, 42-44`**

```ts
const parseVariables = async (variables: string | null): Promise<unknown> => {
  if (!variables) return undefined
  try {
    return JSON.parse(variables)      // ← 字符串 → 对象
  } catch {
    throw new Error("Invalid JSON")   // ← 解析失败抛错
  }
}

// 在 toRequest 中调用：
const variables = await parseVariables(request.variables)

return {
  ...
  content: content.json(
    { query: request.query, variables },   // ← variables 是解析后的对象
    MediaType.APPLICATION_JSON
  ),
}
```

**阶段 4：runGQLOperation — 解析异常处理**

**`connection.ts:496-506`**

```ts
} catch (error: any) {
  gqlMessageEvent.value = {
    type: "error",
    error: {
      type: "network_error",
      message: error.message || "An unknown error occurred",
    },
  }

  throw error   // ← 继续抛出，被 RequestOptions.runQuery catch
}
```

**Variables 处理的完整链路（含异常路径）**：

```
用户在 Variables 编辑器输入 JSON 字符串
    │
    ├─ 存储为 request.variables (string)
    │
    ▼
RequestOptions.runQuery()
    │
    ├─ clone(request.value.variables) → string
    │
    ▼
runGQLOperation({ variables: string })
    │
    ├─ 存入 gqlRequest.variables (string)
    │
    ▼
GQLRequest.toRequest()
    │
    ├─ parseVariables(variables)
    │   ├─ 分支 1：!variables → return undefined
    │   ├─ 分支 2：JSON.parse 成功 → return object
    │   └─ 分支 3：JSON.parse 失败 → throw Error("Invalid JSON")
    │
    ├─ 正常路径：content.json({ query, variables: object|undefined })
    └─ 异常路径：throw Error("Invalid JSON") → 冒泡到 runGQLOperation catch
            ↓
            gqlMessageEvent.value = { type: "error", ... }
            throw error → RequestOptions.runQuery catch 捕获
            ↓
            completePageProgress()，console.error(e)
```

**Variables 所有边界条件的精确行为**：

| 输入值 | `parseVariables` 行为 | 返回值 / 抛出 | 最终请求体 | 对用户的可见影响 |
|--------|----------------------|---------------|------------|-----------------|
| `null` | `if (!variables)` 命中 | `return undefined` | `{ "query": "...", "variables": undefined }`（JSON 中无 variables 键） | 无提示，请求正常发送（不带 variables） |
| 空字符串 `""` | `!""` 为 true？否 → `JSON.parse("")` | 抛 `SyntaxError` → 捕获后抛 `Error("Invalid JSON")` | 不发送请求 | 进度条完成，控制台报错，响应区显示 network_error |
| 非空但无效 JSON（如 `{id:1}`） | `JSON.parse()` 失败 | 抛 `SyntaxError` → 捕获后抛 `Error("Invalid JSON")` | 不发送请求 | 同上 |
| 有效 JSON 对象 `{"id": "1"}` | `JSON.parse()` 成功 | `return { id: "1" }` | `{ "query": "...", "variables": { "id": "1" } }` | 请求正常发送 |
| 有效 JSON 数组 `[1, 2, 3]` | `JSON.parse()` 成功 | `return [1, 2, 3]` | `{ "query": "...", "variables": [1, 2, 3] }` | 请求正常发送（但 server 通常期望 object） |
| 有效 JSON 标量 `"hello"` | `JSON.parse()` 成功 | `return "hello"` | `{ "query": "...", "variables": "hello" }` | 请求正常发送（但 server 通常期望 object） |

**注意**：`parseVariables` 只检查是否能 `JSON.parse` 成功，**不检查解析结果是否为对象**。数组、字符串、数字等都会被正常放入请求体。

#### 4.2.6 OperationName 组装细节与多 Operation 选择

OperationName 的处理分为**解析**和**注入**两个独立阶段，且**不在 GQLRequest.toRequest 内部处理**。

**阶段 1：Query 编辑器 → AST 解析 → 确定 selectedOperation**

**`Query.vue:153-197`**

```ts
const debouncedOnUpdateQueryState = debounce((update: ViewUpdate) => {
  const selectedPos = update.state.selection.main.head  // ← 光标位置
  const queryString = update.state.doc.toJSON().join(update.state.lineBreak)

  try {
    const ast = parse(queryString)

    // 提取所有 OperationDefinition
    operationDefinitions.value = ast.definitions.filter(
      (def) => def.kind === "OperationDefinition"
    ) as OperationDefinitionNode[]

    // 单个 operation → 直接选中
    if (ast.definitions.length === 1) {
      selectedOperation.value = ast.definitions[0] as OperationDefinitionNode
      return
    }

    // 多个 operation → 根据光标位置选中
    selectedOperation.value =
      (ast.definitions.find((def) => {
        if (def.kind !== "OperationDefinition") return false
        const { start, end } = def.loc!
        return selectedPos >= start && selectedPos <= end
      }) as OperationDefinitionNode) ?? null
  } catch (_error) {
    if (queryString.trim() === "") {
      operationDefinitions.value = []
    }
  }
}, 100)

// 初始化时同样解析
onMounted(() => {
  try {
    const ast = parse(gqlQueryString.value)
    operationDefinitions.value = ast.definitions.filter(...)
    if (ast.definitions.length) {
      selectedOperation.value = ast.definitions[0] as OperationDefinitionNode
    }
  } catch (_error) {}
})
```

**选中逻辑精确规则**：
1. 编辑器中只有 1 个 operation → 自动选中它，**无论光标在哪里**
2. 编辑器中有多个 operation → 根据当前光标位置落在哪个 operation 的 `loc.start` 和 `loc.end` 之间来选中
3. 光标不在任何 operation 范围内 → `selectedOperation = null`
4. 解析失败（语法错误）→ `selectedOperation` 保持上一次的值或 null
5. 空字符串 → `operationDefinitions = []`，`selectedOperation` 保持 null

**阶段 2：Run 按钮 → 传入 operationName**

**`Query.vue:23-36`** — Run 按钮渲染：

```html
<HoppButtonSecondary
  v-if="selectedOperation && subscriptionState !== 'SUBSCRIBED'"
  :label="`${selectedOperation.name?.value ?? t('request.run')}`"
  :disabled="!selectedOperation"
  @click="runQuery(selectedOperation)"   // ← 传入 selectedOperation
/>
```

**`RequestOptions.vue:132-158`** — runQuery 接收并传入：

```ts
const runQuery = async (definition: gql.OperationDefinitionNode | null = null) => {
  await runGQLOperation({
    ...
    operationName: definition?.name?.value,    // ← 从 AST 节点 name 中获取
    operationType: definition?.operation ?? "query",
  })
}
```

**阶段 3：runGQLOperation → 后置注入 operationName**

**`connection.ts:426-435`**

```ts
const kernelRequest = await GQLRequest.toRequest(gqlRequest)

// operationName 是在 toRequest 之后额外注入的！
if (operationName) {
  if (kernelRequest.content?.kind === "json") {
    const content = kernelRequest.content.content as any
    content.operationName = operationName
    kernelRequest.content.content = content
  }
}
```

**关键设计**：`operationName` 不是在 `GQLRequest.toRequest()` 内部处理的，而是在 `toRequest()` 返回后，直接修改 `kernelRequest.content.content` 对象的 `operationName` 属性。这是一个**后置注入**。

**OperationName 的完整处理链路**：

```
用户在 Query 编辑器输入查询字符串
    │
    ▼
Query.vue: debouncedOnUpdateQueryState()
    │
    ├─ parse(queryString) → AST
    ├─ 过滤出 OperationDefinitionNode[]
    └─ 根据规则选中一个 → selectedOperation
    │
    ▼
selectedOperation 决定 Run 按钮状态：
    ├─ selectedOperation = null → 按钮隐藏（v-if 为 false），无法点击
    └─ selectedOperation 存在 → 按钮显示，label 为 operation name
    │
    ▼
用户点击 Run 按钮
    │
    ├─ definition = selectedOperation
    ├─ operationName = definition?.name?.value   // 如 "GetUser", undefined（匿名查询）
    └─ operationType = definition?.operation ?? "query"
    │
    ▼
runGQLOperation({ operationName, operationType, ... })
    │
    ├─ GQLRequest.toRequest(gqlRequest) → kernelRequest（只有 query 和 variables）
    │
    └─ 后置注入：如果 operationName 存在（truthy）
           kernelRequest.content.content.operationName = operationName
    │
    ▼
最终请求体
```

**operationName 缺失的两种场景**：

| 场景 | `selectedOperation` | Run 按钮状态 | `operationName` 参数 | 最终请求体 |
|------|---------------------|-------------|---------------------|------------|
| **匿名查询**（`query { method }`） | 存在（1 个 operation） | 显示，label 为 "Run"（因为 `name?.value` 为 undefined） | `undefined` | `{ "query": "query { method }", "variables": {...} }` （无 operationName 字段） |
| **多 operation，光标不在任何范围内** | `null` | **隐藏**（`v-if="selectedOperation"` 为 false） | 无法点击 | 不发送请求 |

**关键发现**：查询没有命名（匿名 operation）时，`selectedOperation` 仍然存在（因为 `ast.definitions.length === 1`），Run 按钮显示但 label 回退为 "Run"。此时 `operationName = undefined`，**不会注入 operationName 字段**。

**Subscription 中的 operationName**（`connection.ts:573-600`）：

```ts
export const runSubscription = (options: RunQueryOptions, headers?) => {
  const { url, query, operationName } = options
  const wsUrl = url.replace(/^http/, "ws")
  // ...
  connection.socket?.send(JSON.stringify({
    type: GQL.START,
    id: "1",
    payload: { query, operationName },   // ← 直接透传，可能为 undefined
  }))
}
```

WebSocket 场景下 `operationName` 直接放入 payload，如果是 undefined 则 JSON 序列化后该字段被省略。

#### 4.2.7 不同执行路径下的请求体差异对比（完整）

| 场景 | `selectedOperation` | `operationName` | `variables` | 最终请求体 |
|------|---------------------|----------------|-------------|------------|
| **单命名查询**<br>`query GetUser { user { id } }` | 存在 | `"GetUser"` | `{ "id": "1" }` | `{ "query": "query GetUser { user { id } }", "variables": { "id": "1" }, "operationName": "GetUser" }` |
| **单匿名查询**<br>`query { user { id } }` | 存在 | `undefined`（不注入） | `{ "id": "1" }` | `{ "query": "query { user { id } }", "variables": { "id": "1" } }` |
| **多 operation，光标在 GetUser 内**<br>`query GetUser {...} query GetPosts {...}` | `GetUser` 节点 | `"GetUser"` | `{}` | `{ "query": "...", "variables": {}, "operationName": "GetUser" }` |
| **多 operation，光标在 GetPosts 内**<br>（同上） | `GetPosts` 节点 | `"GetPosts"` | `{}` | `{ "query": "...", "variables": {}, "operationName": "GetPosts" }` |
| **多 operation，光标在两者之间** | `null` | - | - | **Run 按钮隐藏，无法发送** |
| **variables 为空字符串** | 存在（如果语法正确） | 同上 | `parseVariables("")` 抛错 | **不发送**，catch 中设置 `gqlMessageEvent = error` |
| **variables 为 `null`** | 存在 | 同上 | `undefined` | `{ "query": "...", "variables": undefined }` → JSON 中无 variables 键 |
| **variables 为无效 JSON** | 存在 | 同上 | 抛 `Error("Invalid JSON")` | **不发送**，同上 |
| **Introspection 请求** | - | 无 operationName 字段 | 无 variables 字段 | `{ "query": "query IntrospectionQuery { __schema { ... } }" }` |
| **Subscription（命名）** | 存在 | `"SubscribeUser"` | `{}` | WebSocket payload: `{ "query": "...", "operationName": "SubscribeUser" }` |
| **Subscription（匿名）** | 存在 | `undefined` | `{}` | WebSocket payload: `{ "query": "subscription { ... }" }`（无 operationName 字段） |

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

7. **失败无自动重试**：无论是首次连接失败还是轮询过程中失败，`getSchema()` 中的异常都会导致 `disconnect()` 因状态检查不通过而二次抛出，最终进入 `poll()` 的 catch 块，**不会设置下一次 `setTimeout`**，因此没有自动重试。恢复依赖用户手动再次点击 Connect 按钮。

8. **二次错误抛出陷阱**：`getSchema()` catch 中调用 `disconnect()` 时，由于 `connection.state` 已被设置为 `"ERROR"`，`disconnect()` 的前置检查 `state === "CONNECTED"` 不通过，会再次抛出错误。这个二次错误被 `poll()` 的 catch 捕获。

9. **失败分支中 schema 永不修改**：`getSchema()` 失败路径中，`disconnect()` 抛出异常，因此 `disconnect()` 内部的 `connection.schema = null` 不会被执行。如果是首次连接，schema 保持 `null`；如果是轮询失败，schema 保留之前的缓存值且仍可正常浏览。

10. **轮询停止的充要条件**：只要 `poll()` 的 catch 块被执行（无论什么原因），就不会执行 `setTimeout(poll, 7000)`，轮询终止。没有例外。

11. **`connect()` 永不向外抛异常**：因为 `poll()` 内部有 try-catch 消化所有错误，无论成功失败，`await connect()` 都会正常返回。`isRunGQLOperation = true` 时只额外跳过 toast。

12. **Variables 延迟解析**：用户在编辑器中输入的 variables 始终以字符串形式存储和传递，直到 `GQLRequest.toRequest()` 的最后一步才通过 `JSON.parse()` 解析。空字符串会抛错，`null` 会返回 `undefined`。

13. **Variables 只验 JSON 格式不验结构**：`parseVariables` 只检查是否能 `JSON.parse` 成功，**不检查解析结果是否为对象**。数组、字符串、数字等都会被正常放入请求体。

14. **OperationName 后置注入**：`operationName` 不在 `GQLRequest.toRequest()` 内部处理，而是在 `toRequest()` 返回后，由 `runGQLOperation()` 直接修改 `kernelRequest.content.content.operationName`。

15. **多 Operation 选中规则**：单 operation 时自动选中（无论光标位置）；多 operation 时根据光标位置落在哪个 operation 的 `loc.start/end` 范围内来选中；光标不在范围内时 `selectedOperation = null`，Run 按钮隐藏。

16. **匿名查询的特殊处理**：查询没有命名（`query { ... }`）时，`selectedOperation` 仍然存在，Run 按钮显示但 label 回退为 "Run"。此时 `operationName = undefined`，不会注入 operationName 字段。
