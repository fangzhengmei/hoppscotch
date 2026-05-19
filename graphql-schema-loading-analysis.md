# GraphQL Schema 加载、文档展示与补全提示协作机制分析

> **版本说明**：本文档为第五版，经过多轮逐行走通代码后，状态机结论已 100% 对齐代码行为。

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

### 2.1 核心代码结构与调用栈

连接状态机由三个关键函数组成，它们的嵌套调用关系如下：

```
调用者（Connect 按钮或 Run 按钮）
  ↓
connect(options, isRunGQLOperation)  [connection.ts:186-220]
  ├─ line 199: connection.state = "CONNECTING"
  ├─ 定义内部 poll() 函数
  └─ line 219: await poll()
       ↓
poll()  [connection.ts:201-217]
  ├─ try {
  │    await getSchema(options)
  │    connection.state = "CONNECTED"
  │    setTimeout(poll, 7000)  ← 仅成功时设置下一次轮询
  │  } catch {
  │    connection.state = "ERROR"  ← 任何异常都会到这里
  │    // 无 setTimeout，轮询停止！
  │  }
       ↓
getSchema(options)  [connection.ts:239-335]
  ├─ try {
  │    发送 Introspection 请求
  │    await response
  │    if (E.isLeft(res)) {
  │      connection.state = "ERROR"  ← 先改成 ERROR
  │      throw Error()               ← 然后抛错
  │    }
  │    JSON.parse(responseText)      ← 可能抛错
  │    buildClientSchema(...)        ← 可能抛错
  │    connection.schema = schemaData
  │    connection.error = null
  │  } catch (e) {
  │    console.error(e)
  │    disconnect()                  ← 调用 disconnect（可能抛错）
  │    // ⚠️  没有 rethrow！函数正常返回
  │  }
       ↓
disconnect()  [connection.ts:222-230]
  ├─ if (connection.state !== "CONNECTED") throw Error()
  ├─ clearTimeout(timeoutSubscription)
  ├─ connection.state = "DISCONNECTED"
  └─ connection.schema = null
```

### 2.2 关键代码行为确认

#### 确认 1：JavaScript 异常传播机制

在 JavaScript 中，**catch 块中抛出的错误会继续向外传播**：
```javascript
try {
  try {
    throw new Error("inner")
  } catch (e) {
    console.error(e)
    throw new Error("from catch")  // 这个错误会继续向外冒泡！
  }
} catch (e) {
  console.log("caught:", e.message)  // 会输出 "caught: from catch"
}
```

应用到 `getSchema()`：
```typescript
} catch (e: any) {
  console.error(e)
  disconnect()  // 如果 disconnect() 抛错，这个错误会继续向外冒泡！
  // 只有当 disconnect() 不抛错时，才会执行到这里，函数正常返回
}
```

**结论**：
- 如果 `disconnect()` 抛错 → 错误向外冒泡到 `poll()` 的 catch
- 如果 `disconnect()` 不抛错 → `getSchema()` 正常返回

#### 确认 2：poll() catch 不是死代码

`poll()` 的 catch 分支会在以下场景被执行：
- `getSchema()` try 块内有未被捕获的错误（理论上不会发生，因为 try 包裹了整个函数体）
- 更常见的：`getSchema()` catch 块中调用 `disconnect()` 时，`disconnect()` 抛错

#### 确认 3：setTimeout 只在 getSchema 正常返回后设置

**代码证据** (`connection.ts:203-207`):
```typescript
await getSchema(options)  // 如果这里抛错，后面的代码不会执行
if (connection.state !== "CONNECTED") connection.state = "CONNECTED"
timeoutSubscription = setTimeout(() => {
  poll()
}, GQL_SCHEMA_POLL_INTERVAL)
```

**结论**：
- 如果 `getSchema()` 抛错（因为 `disconnect()` 抛错冒泡）→ `setTimeout` 不会设置，轮询停止
- 如果 `getSchema()` 正常返回 → 设置 `state = "CONNECTED"` 并调度下一次轮询

### 2.3 所有场景逐条走通（最终准确版）

---

#### 场景 1：getSchema() 完全成功

**调用前状态**：任意（CONNECTING 或 CONNECTED）

```
getSchema 中：
  网络请求成功 → E.isLeft(res) = false
  JSON.parse 成功
  buildClientSchema 成功
  line 329: connection.schema = schemaData  ← Schema 更新
  line 330: connection.error = null
  正常返回（无错误）

回到 poll()：
  line 204: connection.state = "CONNECTED"
  line 205-207: setTimeout 已设置，7秒后下次轮询
```

**最终状态**：
- `connection.state` = `"CONNECTED"`
- `connection.schema` = **新值**
- **轮询继续**（setTimeout 已设置）

---

#### 场景 2：首次 connect，网络请求失败（E.isLeft(res)）

**调用前状态**：`CONNECTING`

```
getSchema 中：
  line 294: await response → E.isLeft(res) = true
  line 297: connection.state = "ERROR"
  line 315: throw new Error(...)
  ↓
  catch (e) {
    console.error(e)
    disconnect()  ← 调用 disconnect()
      disconnect() 检查：state !== "CONNECTED"（当前是 ERROR）
      disconnect() 抛错："No connections are running to be disconnected"
    ↓
    ⚠️  catch 块中没有 try-catch，错误继续向外冒泡！
  }
  getSchema() 抛错退出

错误传播到 poll()：
  poll() catch 捕获错误
  line 209: connection.state = "ERROR"
  // 没有设置 setTimeout
```

**最终状态**：
- `connection.state` = `"ERROR"`
- `connection.schema` = `null`（从未被设置过）
- **轮询停止**（setTimeout 未设置）
- `connection.error` 保留错误信息

---

#### 场景 3：已连接后，某次轮询网络请求失败（E.isLeft(res)）

**调用前状态**：`CONNECTED`，`schema` = 旧值

```
getSchema 中：
  line 294: await response → E.isLeft(res) = true
  line 297: connection.state = "ERROR"
  line 315: throw new Error(...)
  ↓
  catch (e) {
    console.error(e)
    disconnect()  ← 调用 disconnect()
      disconnect() 检查：state !== "CONNECTED"（当前是 ERROR）
      disconnect() 抛错
    ↓
    ⚠️  错误继续向外冒泡！
  }
  getSchema() 抛错退出

错误传播到 poll()：
  poll() catch 捕获错误
  line 209: connection.state = "ERROR"
  // 没有设置 setTimeout
```

**最终状态**：
- `connection.state` = `"ERROR"`
- `connection.schema` = **旧值**（从未被修改，因为 disconnect() 抛错没执行清理）
- **轮询停止**（setTimeout 未设置）
- `connection.error` 保留错误信息

---

#### 场景 4：getSchema() 中 JSON.parse / buildClientSchema 失败，首次调用

**调用前状态**：`CONNECTING`

```
getSchema 中：
  网络请求成功 → E.isLeft(res) = false
  JSON.parse 失败 → throw SyntaxError
  ↓
  catch (e) {
    console.error(e)
    disconnect()  ← 调用 disconnect()
      disconnect() 检查：state !== "CONNECTED"（当前仍是 CONNECTING）
      disconnect() 抛错
    ↓
    ⚠️  错误继续向外冒泡！
  }
  getSchema() 抛错退出

错误传播到 poll()：
  poll() catch 捕获错误
  line 209: connection.state = "ERROR"
  // 没有设置 setTimeout
```

**最终状态**：
- `connection.state` = `"ERROR"`
- `connection.schema` = `null`（从未被设置过）
- **轮询停止**（setTimeout 未设置）

---

#### 场景 5：getSchema() 中 JSON.parse / buildClientSchema 失败，已连接后

**调用前状态**：`CONNECTED`，`schema` = 旧值

```
getSchema 中：
  网络请求成功 → E.isLeft(res) = false
  JSON.parse 失败 → throw SyntaxError
  ↓
  catch (e) {
    console.error(e)
    disconnect()  ← 调用 disconnect()
      disconnect() 检查：state === "CONNECTED"（当前仍是 CONNECTED，E.isLeft 分支没执行）
      disconnect() 成功执行：
        clearTimeout(timeoutSubscription)  ← 清空调时器
        connection.state = "DISCONNECTED"  ← 改成 DISCONNECTED
        connection.schema = null           ← 清空 Schema
      disconnect() 正常返回，没有抛错
    ↓
    catch 块继续执行完毕，没有错误
  }
  getSchema() 正常返回 undefined

回到 poll()：
  line 204: if (state !== "CONNECTED") state = "CONNECTED"
    → 当前 state 是 DISCONNECTED，所以改成 CONNECTED！
  line 205-207: setTimeout 已设置（虽然 disconnect() 清掉了，但这里又设置了新的）
```

**最终状态**：
- `connection.state` = `"CONNECTED"`（被 poll() 覆盖了！）
- `connection.schema` = `null`（被 disconnect() 清空了）
- **轮询继续**（setTimeout 已设置新的）

---

#### 场景 6：用户主动点击 Disconnect（state = "CONNECTED"）

**调用前状态**：`CONNECTED`

```
disconnect() 被外部直接调用：
  line 223: if (state !== "CONNECTED") throw → 通过检查
  line 227: clearTimeout(timeoutSubscription)  ← 停止轮询
  line 228: connection.state = "DISCONNECTED"
  line 229: connection.schema = null
```

**最终状态**：
- `connection.state` = `"DISCONNECTED"`
- `connection.schema` = `null`
- **轮询停止**

---

### 2.4 状态回退矩阵（100% 代码对齐版）

| 场景 | 调用前 state | 最终 state | 最终 schema | 轮询是否继续 | disconnect() 结果 | 错误是否到达 poll() catch |
|------|-------------|-----------|-------------|--------------|-------------------|--------------------------|
| getSchema 完全成功 | 任意 | CONNECTED | 新值 | **是** | 未调用 | 否 |
| 首次 connect 网络失败 | CONNECTING | **ERROR** | null | **否** | 抛错 | **是** |
| 已连接后网络失败 | CONNECTED | **ERROR** | **旧值** | **否** | 抛错 | **是** |
| 解析失败（首次） | CONNECTING | **ERROR** | null | **否** | 抛错 | **是** |
| 解析失败（已连接） | CONNECTED | **CONNECTED** | null | **是** | 成功执行 | 否 |
| 用户主动 Disconnect | CONNECTED | DISCONNECTED | null | 否 | 成功执行 | - |
| 页面卸载（state=CONNECTED） | CONNECTED | DISCONNECTED | null | 否 | 成功执行 | - |

### 2.5 异常传播链总结

#### 分叉点：disconnect() 是否抛错

```
getSchema() 发生异常
   ↓
catch (e) {
  disconnect()
    ↳ disconnect() 前置检查通过？
        ├─ 是 → 成功执行，不抛错 → getSchema() 正常返回 → poll() 设置 CONNECTED + setTimeout
        └─ 否 → 抛错 → 错误冒泡到 poll() catch → poll() 设置 ERROR + 不设置 setTimeout
}
```

#### 网络失败 vs 解析失败的关键区别

| 失败类型 | E.isLeft(res) 分支是否执行 | disconnect() 调用时的 state | disconnect() 结果 |
|----------|---------------------------|----------------------------|-------------------|
| 网络失败 | **是**（先把 state 改成 ERROR） | ERROR | 抛错（因为 state !== CONNECTED） |
| 解析失败 | **否**（网络成功了，但解析失败） | CONNECTED | 成功执行 |

#### 最意外的行为：场景 5（解析失败+已连接）

```
getSchema 中 JSON.parse 失败 → catch 调用 disconnect()
  → disconnect() 检查 state === "CONNECTED" → 通过
  → disconnect() 执行：clearTimeout + state = "DISCONNECTED" + schema = null
  → disconnect() 正常返回，没有抛错
  → getSchema() 正常返回 undefined

回到 poll()：
  → line 204: if (state !== "CONNECTED") state = "CONNECTED"
     → 当前 state 是 DISCONNECTED，所以改成 CONNECTED！
  → line 205: setTimeout 已设置新的轮询

最终：state = "CONNECTED", schema = null, 轮询继续
```

`disconnect()` 成功执行了所有清理工作，但 `poll()` 第 204 行**无条件**把 state 覆盖回 `CONNECTED`，并设置了新的轮询。

### 2.6 页面卸载时的清理

**graphql.vue onBeforeUnmount** (`graphql.vue:186-190`):
```typescript
onBeforeUnmount(() => {
  if (connection.state === "CONNECTED") {
    disconnect()
  }
})
```

只有在 `CONNECTED` 状态下才会调用 `disconnect()`，避免因前置检查抛错。

### 2.7 reset() 函数的兜底清理

**reset() 实现** (`connection.ts:232-237`):
```typescript
export const reset = () => {
  if (connection.state === "CONNECTED") disconnect()

  connection.state = "DISCONNECTED"
  connection.schema = null
}
```

- 先尝试调用 `disconnect()`（如果已连接）
- 然后**无条件**设置 `state = "DISCONNECTED"` 和 `schema = null`
- 这是唯一能保证最终状态一致的清理函数

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

### 8.4 轮询失败的实际行为（与设计意图可能不符）
- **网络失败时轮询停止**：网络失败时 `disconnect()` 抛错冒泡到 `poll()` catch，`setTimeout` 不会设置
- **解析失败时轮询继续**：解析失败时 `disconnect()` 成功执行，`getSchema()` 正常返回，`poll()` 继续调度
- **Schema 缓存策略不一致**：
  - 网络失败：保留旧 Schema（因为 `disconnect()` 抛错没执行）
  - 解析失败：清空 Schema（因为 `disconnect()` 成功执行了）
- **错误状态可见性**：
  - 网络失败：最终 state = ERROR，外部组件可以观察到错误状态
  - 解析失败：最终 state = CONNECTED，外部组件观察不到错误

### 8.5 disconnect() 前置条件的关键影响

**前置检查** `if (connection.state !== "CONNECTED") throw` 是整个状态机的核心分叉点：

| 调用时的 state | disconnect() 结果 | 对最终状态的影响 |
|---------------|-------------------|-------------------|
| CONNECTED | 成功执行 | schema 被清空，state 被改成 DISCONNECTED，但会被 poll() 覆盖回 CONNECTED |
| ERROR / CONNECTING | 抛错 | 错误冒泡到 poll() catch，最终 state = ERROR，schema 保留原值 |

**实际效果**：
- 网络失败时，`E.isLeft(res)` 分支先把 state 改成 `ERROR`，导致 `disconnect()` 前置检查失败抛错
- 旧 Schema 因此被保留——网络波动时用户仍可继续浏览
- 但 `connection.state` 最终是 `ERROR`，UI 会显示错误状态

### 8.6 代码中的意外行为

**场景 5（解析失败+已连接）的特殊行为：
- `disconnect()` 成功执行了所有清理工作（clearTimeout, state=DISCONNECTED, schema=null)
- 但 `poll()` 第 204 行 `if (state !== "CONNECTED") state = "CONNECTED"` 无条件覆盖
- 同时第 205 行设置新的 setTimeout
- 最终：state = CONNECTED, schema = null, 轮询继续
- 这是代码中最不符合直觉的"自动重连"行为

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

### 第五版更正（当前最终版本）

本文档对第四版分析的以下内容进行了彻底修正：

1. **getSchema catch 中 disconnect 抛错会向外冒泡**：
   - 第四版错误地认为 `getSchema()` 的 catch 块会吞掉 `disconnect()` 的错误
   - 实际：JavaScript 中 catch 块内抛出的错误会继续向外传播
   - 因此当 `disconnect()` 抛错时，错误会冒泡到 `poll()` 的 catch 分支
   - **`poll()` catch 不是死代码**

2. **轮询是否继续取决于 disconnect() 是否抛错**：
   - 第四版错误地认为轮询永不停止
   - 实际：
     - `disconnect()` 抛错 → 错误冒泡到 `poll()` catch → 不设置 setTimeout → **轮询停止**
     - `disconnect()` 不抛错 → `getSchema()` 正常返回 → 设置 setTimeout → **轮询继续**
   - 网络失败时 `disconnect()` 抛错 → 轮询停止
   - 解析失败（已连接）时 `disconnect()` 成功 → 轮询继续

3. **ERROR 状态是可见的**：
   - 第四版错误地认为 ERROR 状态会被覆盖
   - 实际：网络失败时，错误冒泡到 `poll()` catch，最终 state = ERROR，外部组件可以观察到
   - 只有解析失败（已连接）时，state 才会被 `poll()` 覆盖成 CONNECTED

4. **状态矩阵完全重写**：
   - 网络失败：最终 state = ERROR，schema 保留旧值，轮询停止
   - 解析失败（已连接）：最终 state = CONNECTED，schema = null，轮询继续
   - 其他场景详见 2.4 节矩阵

### 第四版更正

1. **getSchema catch 未 rethrow 的影响**：
   - 第三版错误地认为 `getSchema()` 中的错误会上抛到 `poll()`
   - 实际：`getSchema()` 的 catch 块吞掉了所有错误（包括 `disconnect()` 抛的错），**`getSchema()` 永远不会抛错**（第四版此处结论错误，已在第五版修正）
   - 因此 `poll()` 的 catch 分支是**死代码**，永远不会被执行（第四版此处结论错误，已在第五版修正）

2. **轮询失败后一定会继续调度**：
   - 第三版错误地认为失败后轮询停止
   - 实际：只要 `getSchema()` 返回（无论内部是否出错），`poll()` 就会执行 `setTimeout`，**轮询永不停止**（第四版此处结论错误，已在第五版修正）

3. **错误状态被覆盖**：
   - `getSchema()` 内部设置的 `ERROR` 或 `DISCONNECTED` 状态，会被 `poll()` 第 204 行无条件覆盖成 `CONNECTED`
   - 外部组件永远观察不到 `ERROR` 状态（第四版此处结论部分错误，已在第五版修正）

4. **场景 5 的完整走通**：
   - 解析失败（已连接）时，`disconnect()` 成功执行：清空 schema + 清空调时器 + state = DISCONNECTED
   - 但回到 `poll()` 后，state 被覆盖成 CONNECTED，且设置了新的 setTimeout
   - 最终：state = CONNECTED，schema = null，轮询继续（这一点第四版结论正确）

### 第三版更正

1. **轮询失败后的调度行为**：
   - 第二版错误地认为轮询失败后会继续调度
   - 实际：`setTimeout` 只在 `getSchema` 成功返回后才会被调用，**任何失败都会导致轮询停止**（第三版此处结论错误，已在第四版修正，第五版再次修正）

2. **getSchema 异常调用 disconnect 的真实结果**：
   - 第二版未考虑 `disconnect()` 的前置条件检查
   - 实际：`disconnect()` 要求 `connection.state === "CONNECTED"`，而网络失败时 `state` 已被先改成 `"ERROR"`
   - 因此 `disconnect()` 会抛错，**不会执行 clearTimeout、state 修改和 schema 清空**（这一点第三版结论正确）

3. **网络失败与解析失败的状态差异**：
   - 网络失败（E.isLeft(res)）：state → ERROR，schema 保留旧值，轮询停止（这一点第五版确认正确）
   - 解析失败（JSON.parse/buildClientSchema）且已连接时：state → DISCONNECTED，schema → null，轮询停止（第三版此处结论错误，已在第五版修正为轮询继续）

### 第二版更正

1. **导航栈自动重建**：初版认为 Schema 更新时自动重建导航栈，实际 `updateSchema()` 和 `rebuildNavStack()` 从未被调用。只有切换 Tab 时会调用 `reset()` 重置导航栈。

2. **轮询失败的状态回退**：初版未区分不同异常路径。实际上，单次轮询失败仅标记 ERROR 状态，不会清空 Schema 缓存（第二版此处错误地认为不会停止轮询——第三版又修正为会停止轮询——第四版又修正为轮询永不停止——第五版最终修正为取决于 disconnect 是否抛错）。

3. **文档操作对补全的影响**：初版未说明具体影响路径。实际上文档操作不改变 Schema，只通过更新查询文本和光标位置间接影响补全的上下文输入。
