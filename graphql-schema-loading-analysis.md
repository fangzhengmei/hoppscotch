# GraphQL Schema 加载、文档展示与补全提示协作机制分析

## 概述

Hoppscotch 的 GraphQL 工作区通过 **远端 Introspection 调用**、**本地响应式缓存** 和 **CodeMirror 编辑器联想数据源** 三层架构，实现了 schema 自动加载、字段文档展示和查询补全提示三大核心功能。

---

## 一、Schema 自动加载机制

### 1.1 核心文件
- `packages/hoppscotch-common/src/helpers/graphql/connection.ts`

### 1.2 加载流程

```
用户点击 "Connect" 按钮
       ↓
Request.vue: onConnectClick()
       ↓
connection.ts: connect()
       ↓
connection.ts: poll() [首次调用 + 7秒轮询]
       ↓
connection.ts: getSchema()
       ├─ 构造 Introspection Query (graphql 库 getIntrospectionQuery())
       ├─ 通过 KernelInterceptorService 发送 POST 请求
       ├─ 解析响应 → buildClientSchema() 构建 GraphQLSchema 对象
       └─ 存储到 connection.schema (响应式状态)
```

### 1.3 关键实现细节

**Introspection 查询构造** (`connection.ts:276-289`):
```typescript
const kernelRequest: RelayRequest = {
  id: Date.now(),
  url: options.url,
  method: "POST" as Method,
  headers: { ...finalHeaders, "content-type": "application/json" },
  content: content.json(
    { query: getIntrospectionQuery() },
    MediaType.APPLICATION_JSON
  ),
}
```

**Schema 构建与存储** (`connection.ts:325-329`):
```typescript
const introspectResponse = JSON.parse(responseText)
const schemaData = buildClientSchema(introspectResponse.data)
connection.schema = schemaData
```

**轮询策略** (`connection.ts:35,205-207`):
```typescript
const GQL_SCHEMA_POLL_INTERVAL = 7000  // 7秒轮询一次
timeoutSubscription = setTimeout(() => poll(), GQL_SCHEMA_POLL_INTERVAL)
```

### 1.4 连接状态管理

`connection` 响应式对象维护全局状态 (`connection.ts:118-124`):
```typescript
export const connection = reactive<Connection>({
  state: "DISCONNECTED",    // CONNECTING | CONNECTED | DISCONNECTED | ERROR
  subscriptionState: new Map<string, SubscriptionState>(),
  socket: undefined,
  schema: null,             // GraphQLSchema 实例存储位置
  error: null,
})
```

---

## 二、断连与异常路径下的状态回退逻辑

### 2.1 核心代码结构总览

连接状态机由三个关键函数组成，它们的嵌套调用关系如下：

```
connect()
   ↓
poll()  [循环调度]
   ↓ （try 块内）
getSchema()
   ├─ 成功：设置 connection.schema
   └─ 失败：throw Error → poll() catch 捕获
```

### 2.2 正常断连流程

**用户主动点击 Disconnect** (`connection.ts:222-230`):
```typescript
export const disconnect = () => {
  if (connection.state !== "CONNECTED") {
    throw new Error("No connections are running to be disconnected")
  }

  clearTimeout(timeoutSubscription)   // 停止轮询
  connection.state = "DISCONNECTED"   // 标记为断开
  connection.schema = null            // 清空 Schema 缓存
}
```

**前置条件**：`connection.state` 必须是 `"CONNECTED"`，否则直接抛错不执行任何清理。

**状态变化**（成功时）：
- 轮询定时器被清除
- `connection.state` → `"DISCONNECTED"`
- `connection.schema` → `null`（Schema 缓存被清空）
- 所有依赖 `connection.schema` 的组件自动切换到空状态

### 2.3 poll() 函数的调度逻辑（关键！）

**poll() 完整逻辑** (`connection.ts:201-217`):
```typescript
const poll = async () => {
  try {
    await getSchema(options)
    if (connection.state !== "CONNECTED") connection.state = "CONNECTED"
    // ⚠️  只有 getSchema 成功返回后，才会设置下一次轮询
    timeoutSubscription = setTimeout(() => {
      poll()
    }, GQL_SCHEMA_POLL_INTERVAL)
  } catch (error) {
    // getSchema 抛错会走到这里
    connection.state = "ERROR"
    // ⚠️  失败路径不会设置 setTimeout，轮询停止！
    if (!isRunGQLOperation) {
      toast.error(t("graphql.connection_error_http"))
    }
    console.error(error)
  }
}
```

**关键结论 1**：`setTimeout` 只在 `getSchema` 成功返回后才会被调用。**任何失败都会导致轮询停止**，不会继续调度下一次。

### 2.4 getSchema() 内部异常的完整路径

**getSchema() 的异常处理** (`connection.ts:296-334`):
```typescript
if (E.isLeft(res)) {
  connection.state = "ERROR"       // ① 先标记 ERROR
  // 设置 connection.error...
  throw new Error(...)             // ② 抛出错误
}
// ... JSON.parse 和 buildClientSchema 可能抛错 ...

} catch (e: any) {                   // ③ 外层 catch 捕获所有异常
  console.error(e)
  disconnect()                       // ④ 尝试调用 disconnect()
}
```

**关键结论 2**：`getSchema()` 中任何异常都会先被自身的 catch 捕获，然后尝试调用 `disconnect()`。但 `disconnect()` 有前置条件检查，因此结果取决于调用时的 `connection.state`。

### 2.5 所有异常场景的最终状态

#### 场景 A：首次 connect，网络请求失败（E.isLeft(res)）
```
调用前 state = "CONNECTING"
  ↓
getSchema 中 E.isLeft(res) = true
  → line 297: state = "ERROR"
  → line 315: throw Error
  ↓
getSchema catch 调用 disconnect()
  → disconnect() 检查 state !== "CONNECTED" → true（当前是 ERROR）
  → disconnect() 抛错："No connections are running to be disconnected"
  ↓
错误上抛到 poll() catch
  → poll() catch: state = "ERROR"
  ↓
最终状态：state = "ERROR", schema = null, 轮询停止
```

#### 场景 B：已连接后，某次轮询网络请求失败（E.isLeft(res)）
```
调用前 state = "CONNECTED"
  ↓
getSchema 中 E.isLeft(res) = true
  → line 297: state = "ERROR"  ← 先被改成 ERROR 了！
  → line 315: throw Error
  ↓
getSchema catch 调用 disconnect()
  → disconnect() 检查 state !== "CONNECTED" → true（当前是 ERROR）
  → disconnect() 抛错
  ↓
错误上抛到 poll() catch
  → poll() catch: state = "ERROR"
  ↓
最终状态：state = "ERROR", schema = 旧值（保留上一次成功的 Schema）, 轮询停止
```

#### 场景 C：getSchema() 中 JSON.parse / buildClientSchema 失败（非网络错误）
```
子场景 C1：首次调用（调用前 state = "CONNECTING"）
  → getSchema catch 调用 disconnect()
  → disconnect() 检查 state !== "CONNECTED" → true
  → disconnect() 抛错
  → 最终状态：state = "ERROR", schema = null, 轮询停止

子场景 C2：后续轮询（调用前 state = "CONNECTED"）
  → getSchema catch 调用 disconnect()
  → disconnect() 检查 state === "CONNECTED" → true
  → disconnect() 成功执行：clearTimeout + state = "DISCONNECTED" + schema = null
  → 最终状态：state = "DISCONNECTED", schema = null, 轮询停止
```

#### 场景 D：getSchema() 成功
```
最终状态：state = "CONNECTED", schema = 新值, 轮询继续（setTimeout 已设置）
```

#### 场景 E：用户主动点击 Disconnect（state = "CONNECTED"）
```
最终状态：state = "DISCONNECTED", schema = null, 轮询停止
```

### 2.6 状态回退矩阵（最终准确版）

| 场景 | connection.state | connection.schema | 轮询是否继续 |
|------|------------------|-------------------|--------------|
| 正常断连（state=CONNECTED 时调用 disconnect） | DISCONNECTED | null | 否 |
| 首次 connect 网络失败 | ERROR | null | 否 |
| 已连接后网络请求失败 | ERROR | **保留旧值** | 否 |
| getSchema 解析失败（首次） | ERROR | null | 否 |
| getSchema 解析失败（已连接） | DISCONNECTED | null | 否 |
| 切换 Tab | 无变化 | 无变化 | 是（如果之前在运行） |
| 页面卸载（state=CONNECTED） | DISCONNECTED | null | 否 |

### 2.7 disconnect() 前置条件检查的影响

**disconnect() 的前置检查** 导致了一个重要的行为差异：
- 网络请求失败时，`E.isLeft(res)` 分支会**先把 state 改成 ERROR**，然后才调用 `disconnect()`
- 此时 `disconnect()` 因为前置条件不满足而抛错，**不会清空 schema**
- 因此，已连接后的网络失败会保留旧的 Schema 缓存

**设计意图**：网络波动时保留上次成功的 Schema，让用户可以继续浏览文档和编写查询。

### 2.8 页面卸载时的清理

**graphql.vue onBeforeUnmount** (`graphql.vue:186-190`):
```typescript
onBeforeUnmount(() => {
  if (connection.state === "CONNECTED") {
    disconnect()
  }
})
```

只有在 `CONNECTED` 状态下才会调用 `disconnect()`，避免因前置检查抛错。

---

## 三、字段文档展示机制

### 3.1 核心模块

| 模块 | 位置 | 职责 |
|------|------|------|
| useExplorer | `helpers/graphql/explorer.ts` | 文档导航栈管理 |
| DocExplorer | `components/graphql/DocExplorer.vue` | 文档浏览器主容器 |
| FieldDocumentation | `components/graphql/FieldDocumentation.vue` | 字段详情展示 |
| TypeDocumentation | `components/graphql/TypeDocumentation.vue` | 类型详情展示 |

### 3.2 导航栈管理

`useExplorer()` composable 维护文档浏览的历史栈 (`explorer.ts:48-203`):

```typescript
export function useExplorer(initialSchema?: GraphQLSchema) {
  const navStack = ref<ExplorerNavStack>([initialNavStackItem])
  const schema = ref<GraphQLSchema | null>()
  
  return {
    navStack,           // 导航栈，如 [Root, QueryType, userField, UserType]
    currentNavItem,     // 当前浏览项
    push(item),         // 进入下一级
    pop(),              // 返回上一级
    updateSchema(),     // Schema 更新时重建导航栈（定义但未调用）
    rebuildNavStack(),  // 校验并重建有效导航路径（定义但未调用）
  }
}
```

### 3.3 重要更正：Schema 更新时导航栈不会自动重建

**代码事实核查**：
- `updateSchema()` 和 `rebuildNavStack()` 函数在 `explorer.ts` 中定义
- 但在整个代码库中 **没有任何地方调用这两个函数**
- 导航栈的唯一自动重置时机是在切换 Tab 时调用 `reset()`

**Tab 切换时的重置** (`graphql.vue:128-131`):
```typescript
const changeTab = (tabID: string) => {
  reset()           // ← 重置导航栈到 [Root]
  tabs.setActiveTab(tabID)
}
```

**reset() 实现** (`explorer.ts:91-94`):
```typescript
function reset() {
  navStack.value =
    navStack.value.length === 1 ? navStack.value : [initialNavStackItem]
}
```

**实际行为**：
- Schema 轮询更新后，导航栈 **保持原样**
- 如果新 Schema 中不存在导航栈指向的字段，文档组件会因为找不到对应类型而静默失败
- 只有用户手动返回根目录或切换 Tab 时才会重置

### 3.4 字段文档渲染

**FieldDocumentation.vue** 展示字段详情 (`FieldDocumentation.vue:1-62`):
- 字段名称与类型签名
- 弃用警告（deprecationReason）
- Markdown 格式描述
- 参数列表（Arguments）
- 嵌套字段（Fields）
- 指令信息（Directives）

**类型解析** (`FieldDocumentation.vue:61`):
```typescript
const resolvedType = computed(() => getNamedType(props.field.type))
```

---

## 四、文档浏览改写查询与补全上下文联动

### 4.1 点击字段添加到查询的完整链路

```
用户在文档中点击 "+" 按钮
       ↓
FieldLink.vue: addField() [emit "add-field" 事件]
       ↓
Field.vue: insertQuery()
       ↓
query.ts: handleAddField(field) → handleOperation(field, false)
       ├─ 读取当前 Tab 的查询文本和光标位置
       ├─ 构造 navItems = [...navStack.value, { name: item.name, def: item }]
       ├─ 调用 processOperation(navItems, selectedOperation, false)
       ├─ 生成新的查询字符串 → updatedQuery.value = newQuery
       └─ 设置 cursorPosition.value
       ↓
Query.vue: watch(updatedQuery) 触发
       ├─ 更新 gqlQueryString.value = newQuery
       └─ 更新 CodeMirror 编辑器光标位置
```

### 4.2 processOperation 的核心逻辑

**基于导航栈构建/修改查询 AST** (`query.ts:131-308`):
```typescript
const processOperation = (
  navItems: ExplorerNavStackItem[],
  existingOperation?: OperationDefinitionNode,
  isArgument = false
): OperationResult => {
  const queryPath = navItems.slice(2, isArgument ? -1 : undefined)
  // navItems 结构: [Root, Query/Mutation/Subscription, field1, field2, ...]
  // queryPath 是字段路径: [field1, field2, ...]
  
  if (!existingOperation || 操作类型不同) {
    // 1. 新建操作：从最内层字段开始，向上逐层构建 SelectionSet
    let currentSelection = createFieldNode(lastItem.name, ...)
    for (let i = queryPath.length - 2; i >= 0; i--) {
      const parentField = createFieldNode(item.name, ..., true)
      parentField.selectionSet!.selections = [currentSelection]
      currentSelection = parentField
    }
    return { document: new DocumentNode }
  } else {
    // 2. 修改现有操作：沿着现有 SelectionSet 查找对应字段
    for (let i = 0; i < queryPath.length; i++) {
      const existingFieldIndex = currentSelectionSet.selections.findIndex(...)
      if (existingFieldIndex !== -1) {
        // 字段已存在：如果是最后一项则删除（toggle 行为）
        if (isLastItem) {
          currentSelectionSet.selections.splice(existingFieldIndex, 1)
        }
        // 否则继续向下遍历
      } else {
        // 字段不存在：创建新字段并加入
        const newField = createFieldNode(item.name, ...)
        currentSelectionSet.selections.push(newField)
      }
    }
  }
}
```

### 4.3 对补全上下文的影响

**重要发现**：文档浏览改写查询 **不改变 Schema 本身**，但通过以下方式影响补全体验：

1. **查询文本变化**：
   - `updatedQuery.value` 更新 → `gqlQueryString.value` 更新 → CodeMirror 文档变化
   - 补全触发时，`getAutocompleteSuggestions(schema, text, pos)` 接收的 `text` 参数是最新的查询文本
   - `graphql-language-service-interface` 会基于当前查询的 AST 上下文提供更精准的建议

2. **光标位置变化**：
   - 新字段被添加后，`cursorPosition.value` 被设置到新字段附近
   - 补全触发时，`pos` 参数反映最新的光标位置
   - 补全建议会基于光标所在的语法上下文（如在 SelectionSet 内、在参数列表内等）

3. **补全器的 Schema 引用不变**：
   - `queryCompleter(schema)` 在 Query.vue 初始化时绑定的是 `connection.ts` 导出的 `schema` computed ref
   - 这个 ref 指向的是 `connection.schema`，只有当远端 Introspection 返回新 Schema 时才会变化
   - 文档操作不会触发 Schema 变化

### 4.4 补全器的响应式绑定

**Query.vue 中的绑定** (`Query.vue:209`):
```typescript
completer: queryCompleter(schema),
// schema 是从 ~/helpers/graphql/connection 导入的 computed ref
// 指向 connection.schema
```

**补全器实现** (`gqlQuery.ts:6-25`):
```typescript
const completer: (schemaRef: Ref<GraphQLSchema | null>) => Completer =
  (schemaRef) => (text, completePos) => {
    if (!schemaRef.value) return Promise.resolve(null)
    // 每次补全触发时，读取最新的 schemaRef.value
    const completions = getAutocompleteSuggestions(
      schemaRef.value,
      text,
      { line: completePos.line, character: completePos.ch } as any
    )
    return Promise.resolve({ completions: ... })
  }
```

**关键点**：
- `schemaRef` 是一个响应式 ref，指向 `connection.schema`
- 每次补全触发时，都会读取最新的 `schemaRef.value`
- 因此，当轮询获取到新 Schema 时，下一次补全自动使用新 Schema
- 文档操作不影响 `schemaRef`，只影响 `text` 和 `completePos`

---

## 五、查询补全提示机制

### 5.1 三层架构

```
┌─────────────────────────────────────────────────┐
│  CodeMirror 编辑器                               │
│  ┌───────────────────────────────────────────┐  │
│  │  autocompletion extension                 │  │
│  └─────────────┬─────────────────────────────┘  │
│                │  text, pos                     │
└────────────────┼────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│  gqlQuery Completer                             │
│  packages/hoppscotch-common/src/helpers/editor/ │
│  completion/gqlQuery.ts                         │
│  schema ref + text + pos → 建议列表              │
└────────────────┬────────────────────────────────┘
                 │  schema + text + pos            │
┌────────────────▼────────────────────────────────┐
│  graphql-language-service-interface             │
│  getAutocompleteSuggestions()                   │
└─────────────────────────────────────────────────┘
```

### 5.2 补全器实现

**gqlQuery.ts** (`completion/gqlQuery.ts:1-27`):
```typescript
const completer: (schemaRef: Ref<GraphQLSchema | null>) => Completer =
  (schemaRef) => (text, completePos) => {
    if (!schemaRef.value) return Promise.resolve(null)
    
    const completions = getAutocompleteSuggestions(
      schemaRef.value,
      text,
      { line: completePos.line, character: completePos.ch } as any
    )
    
    return Promise.resolve({
      completions: completions.map((x, i) => ({
        text: x.label!,
        meta: x.detail!,
        score: completions.length - i,
      })),
    })
  }
```

### 5.3 编辑器集成

**Query.vue 中注入补全器** (`Query.vue:199-214`):
```typescript
const cmQueryEditor = useCodemirror(
  queryEditor,
  gqlQueryString,
  reactive({
    extendedEditorConfig: { mode: "graphql", ... },
    linter: createGQLQueryLinter(schema),
    completer: queryCompleter(schema),  // ← 注入补全器，绑定 schema ref
    additionalExts: [markRaw(selectedGQLOpHighlight)],
    onUpdate: debouncedOnUpdateQueryState,
  })
)
```

**useCodemirror 中注册补全扩展** (`codemirror.ts:106-141`):
```typescript
const hoppCompleterExt = (completer: Completer): Extension => {
  return autocompletion({
    override: [
      async (context) => {
        const text = context.state.doc.toJSON().join(context.state.lineBreak)
        const line = context.state.doc.lineAt(context.pos)
        const lineNo = line.number - 1
        const ch = context.pos - line.from
        const result = await completer(text, { line: lineNo, ch })
        return {
          from: context.state.wordAt(context.pos)?.from ?? context.pos,
          options: result?.completions.map(...) ?? [],
        }
      },
    ],
  })
}
```

### 5.4 GraphQL 语法支持

**codemirror-lang-graphql** 包提供语法解析：
- `src/syntax.grammar`: Lezer 语法定义，覆盖完整 GraphQL 规范
- `src/index.js`: 导出 `GQLLanguage` 和 `GQL()` 语言支持

**语法高亮配置** (`codemirror-lang-graphql/src/index.js:29-46`):
```typescript
styleTags({
  Comment: t.lineComment,
  Name: t.propertyName,
  StringValue: t.string,
  "OperationDefinition/Name": t.definition(t.function(t.variableName)),
  "OperationType TypeKeyword SchemaKeyword": t.keyword,
  "Type! NamedType": t.typeName,
})
```

---

## 六、本地缓存策略

### 6.1 内存缓存层

**Schema 缓存**：
- 存储位置：`connection.schema` 响应式引用
- 类型：`GraphQLSchema | null`
- 更新时机：每次轮询成功后替换整个对象
- 消费方式：通过 `computed()` 派生各种查询字段列表

**派生计算属性** (`connection.ts:133-182`):
```typescript
export const schemaString = computed(() => printSchema(connection.schema))
export const queryFields = computed(() => connection.schema?.getQueryType()?.getFields())
export const mutationFields = computed(() => connection.schema?.getMutationType()?.getFields())
export const subscriptionFields = computed(() => connection.schema?.getSubscriptionType()?.getFields())
export const graphqlTypes = computed(() => Object.values(connection.schema?.getTypeMap() || {}))
```

### 6.2 Tab 状态持久化

**GQLTabService** (`services/tab/graphql.ts`):
- 持久化键：`STORE_KEYS.GQL_TABS`
- 持久化内容：tab 列表、请求配置、游标位置（**不含响应数据**）
- 排除字段：`response: null` (`graphql.ts:41`)

**持久化逻辑** (`graphql.ts:33-45`):
```typescript
public override persistableTabState = computed(() => ({
  lastActiveTabID: this.currentTabID.value,
  orderedDocs: this.tabOrdering.value.map((tabID) => {
    const tab = this.tabMap.get(tabID)!
    return {
      tabID: tab.id,
      doc: { ...tab.document, response: null },  // 响应不持久化
    }
  }),
}))
```

---

## 七、三者协作数据流

### 7.1 完整数据流图

```
┌──────────────────────────────────────────────────────────────────┐
│  用户操作：点击 Connect 按钮                                      │
└───────────────────────────────┬──────────────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────────────┐
│  connection.ts: connect() → getSchema()                          │
│  • 发送 Introspection Query 到远端                                │
│  • buildClientSchema() 构建 GraphQLSchema                        │
│  • 写入 connection.schema (响应式)                                │
└───────────────────────────────┬──────────────────────────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          ▼                     ▼                     ▼
┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│  文档展示        │   │  查询补全        │   │  Schema 预览     │
│                 │   │                 │   │                 │
│  useExplorer    │   │  queryCompleter │   │  schemaString   │
│  DocExplorer    │   │  CodeMirror     │   │  Sidebar.vue    │
│  Field/Type...  │   │  autocompletion │   │                 │
└─────────────────┘   └─────────────────┘   └─────────────────┘
          │                     │
          ▼                     ▼
  点击"+"添加字段          用户输入触发补全
          │                     │
          └───────────┬─────────┘
                      ▼
              更新查询文本 (gqlQueryString)
                      │
                      ▼
              补全时使用最新文本和 Schema
```

### 7.2 实时联动机制

1. **Schema 变更触发更新**：
   - `connection.schema` 是响应式引用
   - 所有依赖它的 `computed()` 自动重算
   - 文档浏览器、补全器、Schema 预览同步更新

2. **查询补全触发时机**：
   - 用户输入时，CodeMirror 自动调用补全扩展
   - 补全扩展将当前文档文本和光标位置传给 `queryCompleter`
   - `queryCompleter` 读取最新的 `connection.schema`
   - 调用 `getAutocompleteSuggestions(schema, text, pos)` 生成建议

3. **文档导航联动**：
   - 点击字段链接时，`push()` 到导航栈
   - `currentNavItem` 变化触发对应文档组件渲染
   - 点击 "+" 按钮调用 `handleAddField()` 修改查询
   - 查询文本更新后，下一次补全使用新文本作为上下文

---

## 八、关键设计决策

### 8.1 轮询 vs WebSocket
- **选择轮询**：简化实现，兼容所有 GraphQL 端点
- **间隔 7 秒**：平衡时效性与网络开销
- **适用场景**：Schema 变更不频繁的开发环境

### 8.2 响应式状态共享
- **全局单例**：`connection` 对象是模块级响应式状态
- **跨组件共享**：文档、补全、预览共用同一份 Schema 引用
- **自动更新**：Vue 响应式系统确保所有消费者同步

### 8.3 补全能力分层
- **语法层**：codemirror-lang-graphql 负责语法解析
- **语义层**：graphql-language-service-interface 负责基于 Schema 的补全
- **集成层**：Hoppscotch 自定义 Completer 桥接两者

### 8.4 轮询失败的保守策略
- **保留旧 Schema**：网络请求失败时，`disconnect()` 因前置条件不满足而抛错，不会清空 Schema 缓存
- **仅标记错误状态**：UI 显示错误，但旧 Schema 仍可用，用户可继续浏览文档和编写查询
- **轮询停止**：任何失败都会导致轮询停止，需要用户手动重新连接

### 8.5 disconnect() 前置条件的设计权衡
- **优点**：防止在非连接状态下重复清理，避免逻辑混乱
- **副作用**：网络失败时无法正常执行清理流程，导致 Schema 意外保留
- **实际效果**：这种"bug"反而成为了一个有用的特性——网络波动时保留上次的 Schema

---

## 九、代码索引

| 功能 | 文件路径 | 关键行号 |
|------|----------|----------|
| Schema 加载入口 | `helpers/graphql/connection.ts` | 186-220 |
| poll 函数（轮询调度） | `helpers/graphql/connection.ts` | 201-217 |
| Introspection 请求 | `helpers/graphql/connection.ts` | 276-329 |
| 连接状态管理 | `helpers/graphql/connection.ts` | 118-124 |
| 正常断连逻辑 | `helpers/graphql/connection.ts` | 222-230 |
| getSchema 异常处理 | `helpers/graphql/connection.ts` | 296-334 |
| 文档导航管理 | `helpers/graphql/explorer.ts` | 48-203 |
| reset 函数 | `helpers/graphql/explorer.ts` | 91-94 |
| Tab 切换重置导航 | `pages/graphql.vue` | 128-131 |
| 查询修改逻辑 | `helpers/graphql/query.ts` | 131-375 |
| 字段添加链路 | `components/graphql/Field.vue` | 60-68 |
| 查询补全器 | `helpers/editor/completion/gqlQuery.ts` | 1-27 |
| 编辑器补全扩展 | `composables/codemirror.ts` | 106-141 |
| GraphQL 语法定义 | `codemirror-lang-graphql/src/syntax.grammar` | 1-434 |
| Tab 持久化 | `services/tab/graphql.ts` | 33-53 |

---

## 十、重要更正说明

### 第三版更正（当前版本）

本文档对第二版分析的以下内容进行了修正：

1. **轮询失败后的调度行为**：
   - 第二版错误地认为轮询失败后会继续调度
   - 实际：`setTimeout` 只在 `getSchema` 成功返回后才会被调用，**任何失败都会导致轮询停止**

2. **getSchema 异常调用 disconnect 的真实结果**：
   - 第二版未考虑 `disconnect()` 的前置条件检查
   - 实际：`disconnect()` 要求 `connection.state === "CONNECTED"`，而网络失败时 `state` 已被先改成 `"ERROR"`
   - 因此 `disconnect()` 会抛错，**不会执行 clearTimeout、state 修改和 schema 清空**

3. **网络失败与解析失败的状态差异**：
   - 网络失败（E.isLeft(res)）：state → ERROR，schema 保留旧值，轮询停止
   - 解析失败（JSON.parse/buildClientSchema）且已连接时：state → DISCONNECTED，schema → null，轮询停止

### 第二版更正

1. **导航栈自动重建**：初版认为 Schema 更新时自动重建导航栈，实际 `updateSchema()` 和 `rebuildNavStack()` 从未被调用。只有切换 Tab 时会调用 `reset()` 重置导航栈。

2. **轮询失败的状态回退**：初版未区分不同异常路径。实际上，单次轮询失败仅标记 ERROR 状态，不会清空 Schema 缓存（第二版此处错误地认为不会停止轮询——实际任何失败都会停止轮询）。

3. **文档操作对补全的影响**：初版未说明具体影响路径。实际上文档操作不改变 Schema，只通过更新查询文本和光标位置间接影响补全的上下文输入。
