# Hoppscotch 环境变量解析链路分析

## 1. 概述

Hoppscotch 的环境变量系统采用多层级作用域设计，支持预定义变量、请求变量、集合变量、选中环境变量和全局环境变量的优先级覆盖机制。在请求构造前，系统会通过一套完整的解析链路将模板中的 `<<variable>>` 占位符替换为实际值，并对敏感字段进行特殊保护。

## 2. 变量作用域与优先级

### 2.1 作用域层级（优先级从高到低）

| 优先级 | 变量类型 | 作用域 | 存储位置 | 说明 |
|--------|----------|--------|----------|------|
| 1 | 预定义变量 (Predefined) | 全局 | 代码硬编码 | 如 `<<$guid>>`, `<<$timestamp>>` 等动态生成值 |
| 2 | 请求变量 (Request) | 请求级 | 请求文档 | 单个请求内有效，在请求的 Request Variables 标签页定义 |
| 3 | 集合变量 (Collection) | 集合级 | 集合文档 | 整个集合及其子项共享，从集合继承 |
| 4 | 临时变量 (Temp) | 运行时 | 内存 | 测试脚本中通过 `pm.variables.set()` 设置，不持久化 |
| 5 | 选中环境变量 (Selected Env) | 环境级 | environments store | 当前选中的个人/团队环境 |
| 6 | 全局环境变量 (Global) | 全局 | environments store | 所有环境共享，全局可用 |

### 2.2 优先级实现代码

**核心聚合逻辑** (`packages/hoppscotch-common/src/newstore/environments.ts:436-494`)：

```typescript
export const aggregateEnvs$: Observable<AggregateEnvironment[]> = combineLatest(
  [currentEnvironment$, globalEnv$]
).pipe(
  map(([selectedEnv, globalEnv]) => {
    const effectiveAggregateEnvs: AggregateEnvironment[] = []

    // 1. 预定义变量优先级最高
    HOPP_SUPPORTED_PREDEFINED_VARIABLES.forEach(({ key, getValue }) => {
      effectiveAggregateEnvs.push({
        key,
        currentValue: getValue(),
        initialValue: getValue(),
        secret: false,
        sourceEnv: selectedEnv?.name ?? "Global",
      })
    })

    // 2. 选中环境变量（优先级高于全局）
    selectedEnv?.variables.forEach((variable) => {
      if (!aggregateEnvKeys.includes(key)) {
        effectiveAggregateEnvs.push({...})
      }
    })

    // 3. 全局环境变量（优先级最低）
    globalEnv.variables.forEach((variable) => {
      if (!aggregateEnvKeys.includes(key)) {
        effectiveAggregateEnvs.push({...})
      }
    })

    return effectiveAggregateEnvs
  })
)
```

**请求运行时变量合并** (`packages/hoppscotch-common/src/helpers/RequestRunner.ts:192-206`)：

```typescript
export const combineEnvVariables = (variables: {
  environments: {
    selected: Environment["variables"]
    global: Environment["variables"]
    temp?: Environment["variables"]
  }
  requestVariables: Environment["variables"]
  collectionVariables: Environment["variables"]
}) => [
  ...variables.requestVariables,      // 优先级 1
  ...variables.collectionVariables,   // 优先级 2
  ...(variables.environments.temp ?? []),  // 优先级 3
  ...variables.environments.selected, // 优先级 4
  ...variables.environments.global,   // 优先级 5
]
```

## 3. 环境集合选中机制

### 3.1 选中状态管理

**类型定义** (`packages/hoppscotch-common/src/newstore/environments.ts:18-26`)：

```typescript
export type SelectedEnvironmentIndex =
  | { type: "NO_ENV_SELECTED" }
  | { type: "MY_ENV"; index: number }
  | {
      type: "TEAM_ENV"
      teamID: string
      teamEnvID: string
      environment: Environment
    }
```

### 3.2 选中环境获取流程

1. **用户选择环境** → 通过 `setSelectedEnvironmentIndex()` 更新 store
2. **获取当前环境** → `getCurrentEnvironment()` 根据选中类型返回对应环境
3. **无选中环境** → 返回空环境对象（variables: []）

**当前环境获取逻辑** (`packages/hoppscotch-common/src/newstore/environments.ts:702-720`)：

```typescript
export function getCurrentEnvironment(): Environment {
  if (environmentsStore.value.selectedEnvironmentIndex.type === "NO_ENV_SELECTED") {
    return {
      v: 2,
      id: "",
      name: "No environment",
      variables: [],
    }
  } else if (environmentsStore.value.selectedEnvironmentIndex.type === "MY_ENV") {
    return environmentsStore.value.environments[
      environmentsStore.value.selectedEnvironmentIndex.index
    ]
  }
  return environmentsStore.value.selectedEnvironmentIndex.environment
}
```

## 4. 模板占位符解析机制

### 4.1 占位符格式

使用双尖括号包裹变量名：`<<variable_name>>`

**正则表达式** (`packages/hoppscotch-data/src/environment/index.ts:40`)：

```typescript
const REGEX_ENV_VAR = /<<([^>]*)>>/g // "<<myVariable>>"
```

### 4.2 核心解析函数

**parseTemplateStringE** (`packages/hoppscotch-data/src/environment/index.ts:103-178`)：

```typescript
export function parseTemplateStringE(
  str: string,
  variables: Environment["variables"],
  maskValue = false,
  showKeyIfSecret = false,
  showKeyIfNotFound = false
) {
  let result = str
  let depth = 0
  const ENV_MAX_EXPAND_LIMIT = 10 // 最大递归深度

  while (
    result.match(REGEX_ENV_VAR) != null &&
    depth <= ENV_MAX_EXPAND_LIMIT
  ) {
    const currentResult = result.replace(REGEX_ENV_VAR, (_, p1) => {
      // 1. 优先匹配预定义变量
      const foundPredefinedVar = HOPP_SUPPORTED_PREDEFINED_VARIABLES.find(
        (preVar) => preVar.key === p1
      )
      if (foundPredefinedVar) return foundPredefinedVar.getValue()

      // 2. 匹配环境变量
      const variable = variables.find((x) => x && x.key === p1)
      if (variable && "currentValue" in variable) {
        if (variable.secret && maskValue) {
          return "*".repeat(variable.currentValue.length)
        }
        return variable.currentValue
      }

      return showKeyIfNotFound ? `<<${p1}>>` : ""
    })

    if (currentResult === result) break // 无替换则终止
    result = currentResult
    depth++
  }

  return depth > ENV_MAX_EXPAND_LIMIT
    ? E.left(ENV_EXPAND_LOOP)
    : E.right(result)
}
```

### 4.3 递归解析与循环检测

- **最大递归深度**：10 层（`ENV_MAX_EXPAND_LIMIT`）
- **循环检测**：如果一次替换后结果无变化，立即终止
- **循环错误**：超过最大深度返回 `ENV_EXPAND_LOOP` 错误

**示例**：
```
变量 A = <<B>>
变量 B = <<A>>
解析 <<A>> → 检测到循环 → 保留原始 <<A>> 或报错
```

## 5. 敏感字段隐藏机制

### 5.1 敏感变量存储架构

采用"双轨存储"策略：

| 服务 | 存储内容 | 持久化 | 同步服务器 |
|------|----------|--------|------------|
| `SecretEnvironmentService` | 敏感变量的实际值 | localStorage | ❌ 不同步 |
| `CurrentValueService` | 非敏感变量的当前值 | localStorage | ❌ 不同步 |
| environments store | 变量元数据（key, secret 标记, initialValue 占位） | 云端 | ✅ 同步 |

**SecretEnvironmentService** (`packages/hoppscotch-common/src/services/secret-environment.service.ts:10-190`)：

```typescript
export type SecretVariable = {
  key: string
  value: string           // 实际敏感值
  varIndex: number        // 对应环境中的索引
  initialValue?: string   // 初始敏感值
}

export class SecretEnvironmentService extends Service {
  // key: 环境ID, value: 敏感变量数组
  public secretEnvironments = reactive(new Map<string, SecretVariable[]>())
}
```

### 5.2 敏感变量解析流程

**unWrapEnvironments** (`packages/hoppscotch-common/src/helpers/utils/environments.ts:21-77`)：

```typescript
const unWrapEnvironments = (
  selected: Environment,
  global: Environment["variables"]
) => {
  const resolvedGlobalWithSecrets = global.map((globalVar, index) => {
    const secretVar = secretEnvironmentService.getSecretEnvironmentVariable(
      "Global",
      index
    )
    const currentVar = currentEnvironmentValueService.getEnvironmentVariable(
      "Global",
      index
    )

    if (secretVar) {
      return {
        ...globalVar,
        currentValue: secretVar.value,
        initialValue: secretVar.initialValue ?? "",
      }
    }
    return {
      ...globalVar,
      currentValue: currentVar?.currentValue || globalVar.currentValue || "",
    }
  })
  // ... 对选中环境做同样处理
}
```

### 5.3 UI 层掩码显示

**HoppEnvironment 编辑器插件** (`packages/hoppscotch-common/src/helpers/editor/extensions/HoppEnvironment.ts:168-183`)：

```typescript
// Display secret values as "******" when stored
if (isSecret) {
  if (hasSecretValueStored && hasSecretInitialValueStored) {
    envInitialValue = "******"
    envCurrentValue = "******"
  } else if (!hasSecretValueStored && hasSecretInitialValueStored) {
    envInitialValue = "******"
  } else if (hasSecretValueStored && !hasSecretInitialValueStored) {
    envCurrentValue = "******"
  } else {
    envInitialValue = "Empty"
    envCurrentValue = "Empty"
  }
}
```

### 5.4 导出/同步时的敏感字段保护

**updateEnvironments** (`packages/hoppscotch-common/src/helpers/RequestRunner.ts:220-278`)：

```typescript
const updateEnvironments = (
  envs: Environment["variables"],
  type: "global" | "selected",
  initialEnvID?: string
) => {
  const updatedSecretEnvironments: SecretVariable[] = []
  const nonSecretVariables: Variable[] = []

  const updatedEnv = pipe(
    envs,
    A.mapWithIndex((index, e) => {
      if (e.secret) {
        // 收集到 secret service，不同步到服务器
        updatedSecretEnvironments.push({
          key: e.key,
          value: e.currentValue ?? "",
          varIndex: index,
          initialValue: e.initialValue ?? "",
        })
        // 存储到 store 时清空 currentValue，避免同步到服务器
        return {
          key: e.key,
          secret: e.secret,
          initialValue: e.initialValue ?? "",
          currentValue: "", // 🔒 清空敏感值
        }
      }
      // ... 非敏感变量正常处理
    })
  )

  // 保存到本地服务
  secretEnvironmentService.addSecretEnvironment(envID, updatedSecretEnvironments)
  currentEnvironmentValueService.addEnvironment(envID, nonSecretVariables)

  return updatedEnv // 返回给 store 的版本不含敏感值
}
```

## 6. 完整解析链路

### 6.1 前端请求执行链路

```
用户点击"发送"请求
    ↓
runRESTRequest$()
    ↓
captureInitialEnvironmentState() 捕获初始环境状态
    ↓
delegatePreRequestScriptRunner() 执行前置脚本
    ├─ 脚本中可通过 pm.environment.set() 修改环境变量
    └─ 返回 updatedEnvs
    ↓
combineEnvVariables() 合并各层级变量
    ├─ 请求变量 → 集合变量 → 临时变量 → 选中环境 → 全局环境
    └─ 高优先级覆盖低优先级
    ↓
filterNonEmptyEnvironmentVariables() 过滤空值变量
    ↓
getEffectiveRESTRequest() 构造有效请求
    ├─ parseTemplateStringE(URL) 解析 URL 中的模板
    ├─ parseTemplateStringE(Headers) 解析请求头
    ├─ parseTemplateStringE(Params) 解析查询参数
    ├─ parseBodyEnvVariablesE(Body) 解析请求体
    └─ 各认证字段解析（Basic, Bearer, OAuth2 等）
    ↓
createRESTNetworkRequestStream() 发送网络请求
    ↓
请求返回 → 执行后置脚本 → 更新环境变量（如果有修改）
```

### 6.2 CLI 执行链路

```
hopp test 命令执行
    ↓
preRequestScriptRunner()
    ├─ runPreRequestScript() 执行前置脚本
    └─ getEffectiveRESTRequest() 构造有效请求
        ├─ getResolvedVariables() 合并变量（请求 → 集合 → 环境）
        ├─ getEffectiveFinalMetaData() 解析 Headers/Params
        ├─ getFinalBodyFromRequest() 解析 Body
        ├─ 认证字段解析
        └─ parseTemplateStringE(URL) 解析最终 URL
    ↓
发送请求 → 执行测试脚本 → 输出结果
```

### 6.3 关键节点代码位置

| 节点 | 文件 | 函数 |
|------|------|------|
| 环境聚合 | `newstore/environments.ts` | `aggregateEnvs$`, `getAggregateEnvsWithCurrentValue()` |
| 变量合并 | `helpers/RequestRunner.ts` | `combineEnvVariables()` |
| 模板解析 | `@hoppscotch/data/environment/index.ts` | `parseTemplateStringE()`, `parseBodyEnvVariablesE()` |
| 敏感值展开 | `helpers/utils/environments.ts` | `unWrapEnvironments()`, `getCombinedEnvVariables()` |
| 请求构造 | `helpers/RequestRunner.ts` | `runRESTRequest$()`, `getEffectiveRESTRequest()` |
| UI 高亮提示 | `helpers/editor/extensions/HoppEnvironment.ts` | `HoppEnvironmentPlugin` |
| 秘密存储 | `services/secret-environment.service.ts` | `SecretEnvironmentService` |

## 7. 关键协作关系

### 7.1 环境 Store 与 Service 协作

```
environmentsStore (云端同步)
    ├─ 存储变量元数据（key, secret 标记）
    ├─ 非敏感变量的 initialValue 和 currentValue
    └─ 敏感变量的 currentValue 为空（保护机制）
          ↑
          │ 同步时只存非敏感值
          ↓
SecretEnvironmentService (本地存储)
    └─ 存储敏感变量的实际 value 和 initialValue
          ↑
          │ 运行时合并
          ↓
unWrapEnvironments() 运行时展开
    └─ 为请求执行提供完整的变量值（含敏感值）
```

### 7.2 编辑器插件与环境系统协作

`HoppEnvironmentPlugin` 监听环境变化，实时更新编辑器中的变量高亮和提示：

1. 监听 `aggregateEnvsWithCurrentValue$` 流
2. 监听当前活动标签页的请求变量和集合变量
3. 合并所有变量源
4. 对 `<<variable>>` 进行语法高亮
5. 悬停时显示变量值（敏感变量掩码显示）

## 8. 设计特点与安全考虑

### 8.1 安全设计

1. **敏感值分离存储**：敏感值只存在于浏览器 localStorage，不同步到服务器
2. **UI 掩码**：敏感值在界面上显示为 `******`
3. **导出保护**：导出环境时敏感值被清空，只保留密钥名称
4. **运行时展开**：仅在请求执行前的内存中合并敏感值

### 8.2 灵活性设计

1. **多层级覆盖**：支持 6 层变量作用域，满足复杂场景
2. **递归解析**：支持变量嵌套引用（如 `<<host>>/api/<<version>>`）
3. **脚本修改**：前置/后置脚本可动态修改变量值
4. **预定义变量**：内置常用动态值（时间戳、GUID、随机数等）

### 8.3 健壮性设计

1. **循环检测**：最大 10 层递归，防止循环引用导致死循环
2. **优雅降级**：变量不存在时返回空字符串或保留原始占位符
3. **状态捕获**：请求执行前捕获环境快照，执行后对比更新
