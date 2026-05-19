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

**Introspection 查询构造** (`connection.ts:285-289`):
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

**Schema 构建与存储** (`connection.ts:327-329`):
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

## 二、本地缓存策略

### 2.1 内存缓存层

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

### 2.2 Tab 状态持久化

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
    updateSchema(),     // Schema 更新时重建导航栈
    rebuildNavStack(),  // 校验并重建有效导航路径
  }
}
```

**导航重建机制** (`explorer.ts:133-191`):
- 当 Schema 更新时，自动校验当前导航栈中的每一项在新 Schema 中是否存在
- 遇到失效项立即截断，保留有效前缀

### 3.3 字段文档渲染

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

## 四、查询补全提示机制

### 4.1 三层架构

```
┌─────────────────────────────────────────────────┐
│  CodeMirror 编辑器                               │
│  ┌───────────────────────────────────────────┐  │
│  │  autocompletion extension                 │  │
│  └─────────────┬─────────────────────────────┘  │
│                │                                │
└────────────────┼────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│  gqlQuery Completer                             │
│  packages/hoppscotch-common/src/helpers/editor/ │
│  completion/gqlQuery.ts                         │
└────────────────┬────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│  graphql-language-service-interface             │
│  getAutocompleteSuggestions()                   │
└─────────────────────────────────────────────────┘
```

### 4.2 补全器实现

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

### 4.3 编辑器集成

**Query.vue** 中注入补全器 (`Query.vue:199-214`):
```typescript
const cmQueryEditor = useCodemirror(
  queryEditor,
  gqlQueryString,
  reactive({
    extendedEditorConfig: { mode: "graphql", ... },
    linter: createGQLQueryLinter(schema),
    completer: queryCompleter(schema),  // ← 注入补全器
    additionalExts: [markRaw(selectedGQLOpHighlight)],
    onUpdate: debouncedOnUpdateQueryState,
  })
)
```

**useCodemirror** 中注册补全扩展 (`codemirror.ts:106-141`):
```typescript
const hoppCompleterExt = (completer: Completer): Extension => {
  return autocompletion({
    override: [
      async (context) => {
        const text = context.state.doc.toJSON().join(context.state.lineBreak)
        const line = context.state.doc.lineAt(context.pos)
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

### 4.4 GraphQL 语法支持

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
  // ...
})
```

---

## 五、三者协作数据流

### 5.1 完整数据流图

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
│  Field...       │   │  autocompletion │   │                 │
└─────────────────┘   └─────────────────┘   └─────────────────┘
```

### 5.2 实时联动

1. **Schema 变更触发更新**：
   - `connection.schema` 是响应式引用
   - 所有依赖它的 `computed()` 自动重算
   - 文档浏览器、补全器、Schema 预览同步更新

2. **查询补全触发时机**：
   - 用户输入时，CodeMirror 自动调用补全扩展
   - 补全扩展将当前文档文本和光标位置传给 `queryCompleter`
   - `queryCompleter` 调用 `getAutocompleteSuggestions(schema, text, pos)`
   - 基于当前 Schema 上下文生成建议

3. **文档导航联动**：
   - 点击字段链接时，`push()` 到导航栈
   - `currentNavItem` 变化触发对应文档组件渲染
   - Schema 更新时自动重建导航栈，失效项被截断

---

## 六、关键设计决策

### 6.1 轮询 vs WebSocket
- **选择轮询**：简化实现，兼容所有 GraphQL 端点
- **间隔 7 秒**：平衡时效性与网络开销
- **适用场景**：Schema 变更不频繁的开发环境

### 6.2 响应式状态共享
- **全局单例**：`connection` 对象是模块级响应式状态
- **跨组件共享**：文档、补全、预览共用同一份 Schema 引用
- **自动更新**：Vue 响应式系统确保所有消费者同步

### 6.3 补全能力分层
- **语法层**：codemirror-lang-graphql 负责语法解析
- **语义层**：graphql-language-service-interface 负责基于 Schema 的补全
- **集成层**：Hoppscotch 自定义 Completer 桥接两者

---

## 七、代码索引

| 功能 | 文件路径 | 关键行号 |
|------|----------|----------|
| Schema 加载入口 | `helpers/graphql/connection.ts` | 186-220 |
| Introspection 请求 | `helpers/graphql/connection.ts` | 239-335 |
| 连接状态管理 | `helpers/graphql/connection.ts` | 118-124 |
| 文档导航管理 | `helpers/graphql/explorer.ts` | 57-203 |
| 查询补全器 | `helpers/editor/completion/gqlQuery.ts` | 1-27 |
| 编辑器补全扩展 | `composables/codemirror.ts` | 106-141 |
| GraphQL 语法定义 | `codemirror-lang-graphql/src/syntax.grammar` | 1-434 |
| Tab 持久化 | `services/tab/graphql.ts` | 33-53 |
