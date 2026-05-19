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

---

## 9. 容易误判的细节分析

### 9.1 同键变量高优先级为空时的回退规则

**核心问题**：当高优先级作用域中存在同名变量但其值为空时，是否会"穿透"到低优先级作用域取值？

**答案**：**是，但条件非常严格**。

#### 9.1.1 回退触发条件

**filterNonEmptyEnvironmentVariables** (`packages/hoppscotch-common/src/helpers/RequestRunner.ts:337-361`) 是关键：

```typescript
const getTransformedEnvs = (
  env: Environment["variables"][number]
): Environment["variables"][number] => {
  return {
    ...env,
    currentValue: env.currentValue || env.initialValue,  // 第一步：自身回退
  }
}

export const filterNonEmptyEnvironmentVariables = (
  envs: Environment["variables"]
): Environment["variables"] => {
  const envsMap = new Map<string, Environment["variables"][number]>()
  envs.forEach((env) => {
    const transformedEnv = getTransformedEnvs(env)

    if (envsMap.has(transformedEnv.key)) {
      const existingEnv = envsMap.get(transformedEnv.key)

      // 🔴 只有当已存在（高优先级）变量 currentValue 为空，
      // 且当前（低优先级）变量 currentValue 非空时，才会回退
      if (
        existingEnv &&
        "currentValue" in existingEnv &&
        existingEnv.currentValue === "" &&
        transformedEnv.currentValue !== ""
      ) {
        envsMap.set(transformedEnv.key, transformedEnv)
      }
    } else {
      envsMap.set(transformedEnv.key, transformedEnv)
    }
  })

  return Array.from(envsMap.values())
}
```

#### 9.1.2 回退决策树

```
高优先级变量 currentValue 为空?
    ├─ 否 → 使用高优先级值（即使是空字符串也不会回退）
    └─ 是 → 尝试用高优先级变量的 initialValue 填充
              ├─ 填充后非空 → 使用填充后的值（不回退）
              └─ 填充后仍为空 → 检查低优先级同键变量
                        ├─ 低优先级变量经过自身回退后非空 → 使用低优先级值 ✅
                        └─ 低优先级变量也为空 → 保留空字符串
```

#### 9.1.3 常见误判场景

| 场景 | 预期行为 | 实际行为 | 原因 |
|------|----------|----------|------|
| 选中环境 key=""，全局 key="val" | 回退到全局 | ✅ 回退到全局 | 高优先级为空，低优先级非空 |
| 选中环境 key=" "（空格），全局 key="val" | 回退到全局 | ❌ 使用空格 | `||` 运算符将空格视为 truthy |
| 选中环境 currentValue=""，initialValue="x" | 回退到全局 | ❌ 使用 "x" | getTransformedEnvs 先做了自身回退 |
| 请求变量 key=""，环境变量 key="val" | 回退到环境 | ❌ 保留空 | combineEnvVariables 合并后，filterNonEmpty 才会处理 |

**关键代码位置**：`RequestRunner.ts:322-361`

---

### 9.2 请求开始与结束时环境快照防串写机制

**核心问题**：并行发送多个请求时，如何避免脚本修改的环境变量互相串写？

**答案**：**深拷贝快照 + 作用域隔离 + 增量更新**

#### 9.2.1 快照捕获时机

**captureInitialEnvironmentState** (`packages/hoppscotch-common/src/helpers/RequestRunner.ts:118-156`)：

```typescript
export const captureInitialEnvironmentState = (): InitialEnvironmentState => {
  // 🔴 每次调用都会创建深拷贝，避免引用共享
  const globalEnvs = cloneDeep(getGlobalVariables())
  const selectedEnv = getCurrentEnvironment()

  // 捕获选中环境的索引快照（用于后续确定更新目标）
  const initialEnvironmentIndex = cloneDeep(
    environmentsStore.value.selectedEnvironmentIndex
  )

  // 用于后续对比的完整变量快照
  const initialEnvs = getCombinedEnvVariables()
  const initialEnvsForComparison: TestResult["envs"] = {
    global: initialEnvs.global,
    selected: initialEnvs.selected,
  }

  return {
    globalEnvs,
    selectedEnv,
    initialEnvironmentIndex,
    initialSelectedEnvID: selectedEnv?.id,
    initialGlobalEnvID: globalEnvStore.value.id,
    initialEnvs,
    initialEnvsForComparison,
  }
}
```

#### 9.2.2 隔离机制详解

```
请求 A 开始                     请求 B 开始
    ↓                              ↓
captureInitialEnvironmentState  captureInitialEnvironmentState
    ↓ (cloneDeep)                   ↓ (cloneDeep)
创建独立副本 A_env              创建独立副本 B_env
    ↓                              ↓
前置脚本修改 A_env             前置脚本修改 B_env
    ↓                              ↓
用 A_env 发送请求               用 B_env 发送请求
    ↓                              ↓
对比 A_env 与初始快照          对比 B_env 与初始快照
    ↓ (增量更新)                   ↓ (增量更新)
仅更新 A 中变化的变量          仅更新 B 中变化的变量
```

#### 9.2.3 实际入参与更新路径

**真实函数签名** (`RequestRunner.ts:698-743`)：

```typescript
function updateEnvsAfterTestScript(
  runResult: E.Right<SandboxTestResult>,      // 🔴 实际第1参：脚本运行结果（含 envs）
  initialEnvironmentIndex: SelectedEnvironmentIndex,  // 🔴 第2参：选中环境索引快照
  initialEnvName: string,                     // 🔴 第3参：环境名称（用于团队环境）
  initialEnvID?: string                       // 🔴 第4参：环境ID
) {
  // 1. 更新全局环境
  const globalEnvVariables = updateEnvironments(
    runResult.right.envs.global,  // 从运行结果中提取 envs.global
    "global"
  )
  setGlobalEnvVariables({
    v: 2,
    variables: globalEnvVariables,
  })

  // 2. 更新选中环境
  const selectedEnvVariables = updateEnvironments(
    cloneDeep(runResult.right.envs.selected),  // 深拷贝避免引用问题
    "selected",
    initialEnvID
  )

  // 3. 根据环境类型分发更新
  if (initialEnvironmentIndex.type === "MY_ENV") {
    updateEnvironment(initialEnvironmentIndex.index, {
      name: env.name,
      v: 2,
      id: "id" in env ? env.id : "",
      variables: selectedEnvVariables,
    })
  } else if (initialEnvironmentIndex.type === "TEAM_ENV") {
    const envName = initialEnvName ?? getCurrentEnvironment().name
    updateTeamEnvironment(
      JSON.stringify(selectedEnvVariables),
      initialEnvironmentIndex.teamEnvID,
      envName
    )()
  }
}
```

**调用位置 1 - 单请求执行** (`RequestRunner.ts:636-641`)：
```typescript
if (hasEnvironmentChanges(initialEnvsForComparison, postRequestScriptResult.right.envs)) {
  updateEnvsAfterTestScript(
    combinedResult,           // 脚本运行结果（含 envs）
    initialEnvironmentIndex,  // 请求开始时捕获的索引
    initialEnvName,           // 请求开始时的环境名
    initialEnvID              // 请求开始时的环境ID
  )
}
```

**调用位置 2 - 测试运行器** (`RequestRunner.ts:939-944`)：
```typescript
if (hasEnvironmentChanges(initialEnvsForComparison, postRequestScriptResult.right.envs)) {
  updateEnvsAfterTestScript(
    postRequestScriptResult,  // 脚本运行结果
    initialEnvironmentIndex,
    initialEnvName,
    initialEnvID
  )
}
```

**关键证据**：
- 函数内部直接从 `runResult.right.envs` 提取环境变量，而非单独传入
- 更新路径：全局环境走 `setGlobalEnvVariables`，个人环境走 `updateEnvironment`，团队环境走 `updateTeamEnvironment`
- `initialEnvID` 只用于选中环境，用于 `updateEnvironments` 中定位 secret/current value 服务的存储键

#### 9.2.4 边界情况

| 场景 | 行为 | 风险 |
|------|------|------|
| 请求执行中用户切换了环境 | 脚本修改仍作用于**捕获时**的环境，不会写到新环境 | 可能更新到非预期环境 |
| 并行请求修改同一个变量 | 后完成的请求覆盖先完成的 | 竞态条件 |
| 前置脚本失败 | 环境变量不会被更新 | 符合预期 |
| 后置脚本失败 | 环境变量已更新 | 部分副作用已产生 |

**关键代码位置**：`RequestRunner.ts:118-156`, `748-837`

---

### 9.3 集合继承变量冲突决议

**核心问题**：多层嵌套集合中存在同名变量时，哪一层生效？

**答案**：**深度优先，后遍历优先（子集合优先于父集合）**

#### 9.3.1 冲突决议算法

**transformInheritedCollectionVariablesToAggregateEnv** (`packages/hoppscotch-common/src/helpers/utils/inheritedCollectionVarTransformer.ts:38-64`)：

```typescript
export const transformInheritedCollectionVariablesToAggregateEnv = (
  variables: HoppInheritedProperty["variables"],
  showSecret: boolean = true
): AggregateEnvironment[] => {
  // 1. 扁平化：按遍历顺序将所有继承变量展开为一维数组
  const flattened = variables.flatMap(({ parentID, inheritedVariables }) =>
    inheritedVariables.map(
      ({ currentValue, initialValue, key, secret }, index) => ({
        key,
        currentValue: getCurrentValue(secret, index, parentID, showSecret) ?? currentValue,
        initialValue,
        sourceEnv: "CollectionVariable",
        sourceEnvID: parentID,
        secret,
      })
    )
  )

  // 2. 去重：后遇到的值覆盖先遇到的值
  // 🔴 Map.set() 会直接覆盖已存在的 key
  const mapByKey = new Map<string, AggregateEnvironment>()
  flattened.forEach((variable) => {
    mapByKey.set(variable.key, variable)
  })

  return Array.from(mapByKey.values())
}
```

#### 9.3.2 遍历顺序与优先级

假设集合结构：
```
Collection A (var: key=A)
    ├─ Collection B (var: key=B)
    │     └─ Request X
    └─ Collection C (var: key=C)
          └─ Request Y
```

对于 Request X，继承变量遍历顺序为：`[A, B]` → 扁平化后为 `[A.key, B.key]`
→ Map 处理后 `key = B`（B 覆盖 A）

对于 Request Y，继承变量遍历顺序为：`[A, C]` → 扁平化后为 `[A.key, C.key]`
→ Map 处理后 `key = C`（C 覆盖 A）

**结论**：**越靠近请求的集合，变量优先级越高**。

#### 9.3.3 与其他作用域的交互

集合变量的优先级位于：**请求变量 < 集合变量 < 临时变量 < 环境变量**

```
优先级从高到低：
  1. 预定义变量
  2. 请求变量 (Request Variables)
  3. 集合变量 (Collection Variables) ← 继承冲突在此层内部解决
  4. 临时变量 (Temp Variables)
  5. 选中环境变量
  6. 全局环境变量
```

#### 9.3.4 CLI 中的特殊处理

**getResolvedVariables** (`packages/hoppscotch-cli/src/utils/getters.ts:276-335`)：

```typescript
export const getResolvedVariables = (
  requestVariables: HoppRESTRequestVariables,
  environmentVariables: EnvironmentVariable[],
  collectionVariables: HoppCollectionVariable[] = []
): EnvironmentVariable[] => {
  const activeRequestVariables = requestVariables
    .filter(({ active, value }) => active && value)  // 🔴 只保留 active 且 value 非空的
    .map(...)

  const requestVariableKeys = activeRequestVariables.map(({ key }) => key)

  // 过滤掉与请求变量同名的集合变量
  const filteredCollectionVariables = collectionVariables.filter(
    ({ key }) => !requestVariableKeys.includes(key)
  )

  const collectionVariableKeys = filteredCollectionVariables.map(({ key }) => key)

  // 过滤掉与请求/集合变量同名的环境变量
  const filteredEnvironmentVariables = environmentVariables.filter(
    ({ key }) => ![...requestVariableKeys, ...collectionVariableKeys].includes(key)
  )

  return [
    ...activeRequestVariables,    // 优先级 1
    ...processedCollectionVariables,  // 优先级 2
    ...processedEnvironmentVariables, // 优先级 3
  ]
}
```

#### 9.3.4 CLI getResolvedVariables 与前端 filterNonEmptyEnvironmentVariables 差异

**CLI 逻辑** (`getters.ts:276-335`)：
```typescript
export const getResolvedVariables = (
  requestVariables: HoppRESTRequestVariables,
  environmentVariables: EnvironmentVariable[],
  collectionVariables: HoppCollectionVariable[] = []
): EnvironmentVariable[] => {
  // 🔴 只保留 active=true 且 value 非空的请求变量
  const activeRequestVariables = requestVariables
    .filter(({ active, value }) => active && value)
    .map(...)

  const requestVariableKeys = activeRequestVariables.map(({ key }) => key)

  // 🔴 直接过滤掉与请求变量同名的集合变量（不管值是否为空）
  const filteredCollectionVariables = collectionVariables.filter(
    ({ key }) => !requestVariableKeys.includes(key)
  )

  const collectionVariableKeys = filteredCollectionVariables.map(({ key }) => key)

  // 🔴 直接过滤掉与请求/集合变量同名的环境变量（不管值是否为空）
  const filteredEnvironmentVariables = environmentVariables.filter(
    ({ key }) => ![...requestVariableKeys, ...collectionVariableKeys].includes(key)
  )

  return [
    ...activeRequestVariables,
    ...processedCollectionVariables,
    ...processedEnvironmentVariables,
  ]
}
```

**前端逻辑** (`RequestRunner.ts:337-361`)：
```typescript
export const filterNonEmptyEnvironmentVariables = (
  envs: Environment["variables"]
): Environment["variables"] => {
  const envsMap = new Map<string, Environment["variables"][number]>()
  envs.forEach((env) => {
    const transformedEnv = getTransformedEnvs(env)  // currentValue || initialValue

    if (envsMap.has(transformedEnv.key)) {
      const existingEnv = envsMap.get(transformedEnv.key)
      // 🔴 只有当高优先级变量 currentValue 为空，且低优先级非空时，才覆盖
      if (
        existingEnv &&
        "currentValue" in existingEnv &&
        existingEnv.currentValue === "" &&
        transformedEnv.currentValue !== ""
      ) {
        envsMap.set(transformedEnv.key, transformedEnv)
      }
    } else {
      envsMap.set(transformedEnv.key, transformedEnv)
    }
  })
  return Array.from(envsMap.values())
}
```

**同键空值回退差异对比表**：

| 场景 | 前端 (filterNonEmpty) | CLI (getResolvedVariables) |
|------|-----------------------|----------------------------|
| 请求变量 key=""，环境变量 key="val" | ✅ 回退到环境变量 | ❌ 请求变量 active=true 但 value="" 会被过滤，环境变量保留 |
| 请求变量 key=" "（空格），环境变量 key="val" | ❌ 使用空格（`||` 视为 truthy） | ❌ 请求变量 active=true 且 value=" " 保留，环境变量被过滤 |
| 请求变量 key="val1"，环境变量 key="val2" | ❌ 使用 "val1"（高优先级） | ❌ 使用 "val1"（高优先级，环境变量被过滤） |
| 集合变量 key=""，环境变量 key="val" | ✅ 回退到环境变量 | ✅ 集合变量保留 key=""，环境变量被过滤（行为不同！） |
| 选中环境 key=""，全局 key="val" | ✅ 回退到全局 | ✅ 选中环境 key="" 保留，全局被过滤（行为不同！） |

**关键差异本质**：
1. **CLI 是"存在即覆盖"**：只要高优先级作用域中存在该 key（无论值是否为空），低优先级同 key 变量直接被过滤
2. **前端是"空值回退"**：高优先级变量为空（`currentValue === ""` 且 `initialValue` 也为空）时，才会尝试用低优先级非空值覆盖
3. **请求变量过滤**：CLI 在第一步就过滤掉 `active=false` 或 `value` 为空的请求变量，前端则保留所有请求变量交由后续回退逻辑处理

**关键代码位置**：`inheritedCollectionVarTransformer.ts:38-64`, `getters.ts:276-335`, `RequestRunner.ts:322-361`

---

### 9.4 URL、Header、Params 与 Body 的模板解析差异

**核心问题**：不同位置的模板解析行为是否一致？

**答案**：**不一致，存在多处细微差异**

#### 9.4.1 前端请求构造阶段解析函数调用表

**getEffectiveRESTRequest** (`packages/hoppscotch-common/src/helpers/utils/EffectiveURL.ts:362-446`)：

| 字段 | 解析函数 | maskValue | showKeyIfSecret | showKeyIfNotFound | 错误回退 |
|------|----------|-----------|-----------------|-------------------|----------|
| URL | `parseTemplateString` | `false` | 函数入参 `showKeyIfSecret` | 函数入参 `showKeyIfNotFound` | `E.getOrElse(() => str)` 返回原始值 |
| Header Key | `parseTemplateString` | `false` | 函数入参 `showKeyIfSecret` | `undefined`（默认 false） | 返回原始值 |
| Header Value | `parseTemplateString` | `false` | 函数入参 `showKeyIfSecret` | `undefined`（默认 false） | 返回原始值 |
| Param Key | `parseTemplateString` | `false` | 函数入参 `showKeyIfSecret` | `undefined`（默认 false） | 返回原始值 |
| Param Value | `parseTemplateString` | `false` | 函数入参 `showKeyIfSecret` | `undefined`（默认 false） | 返回原始值 |
| Body | `getFinalBodyFromRequest` 内部调度 | - | 函数入参 `showKeyIfSecret` | - | 因类型而异 |

**代码证据**：
```typescript
// URL 解析（EffectiveURL.ts:434-440）
effectiveFinalURL: parseTemplateString(
  request.endpoint,
  environment.variables,
  false,
  showKeyIfSecret,
  showKeyIfNotFound   // 🔴 唯一传入 showKeyIfNotFound 的位置
)

// Header 解析（EffectiveURL.ts:376-387）
key: parseTemplateString(x.key, environment.variables, false, showKeyIfSecret),
value: parseTemplateString(x.value, environment.variables, false, showKeyIfSecret)
// 🔴 未传 showKeyIfNotFound，使用默认值 false → 未找到时返回空字符串
```

#### 9.4.2 前端 Body 解析调度（getFinalBodyFromRequest）

| Content-Type | 实际调用函数 | 错误回退 |
|--------------|--------------|----------|
| `application/json` | `parseBodyEnvVariables` | 解析失败返回原始 body |
| `application/x-www-form-urlencoded` | 每个 key/value 调 `parseTemplateStringE` | 解析失败的项被过滤，返回剩余项的 query string |
| `multipart/form-data` | 文本字段调 `parseTemplateString` | 解析失败返回空字符串 |
| `application/octet-stream` | 不解析 | 直接返回 File/Blob |
| 其他（text/plain 等） | `parseBodyEnvVariables` | 解析失败返回原始 body |

**parseBodyEnvVariables 的错误回退** (`environment/index.ts:94-101`)：
```typescript
export const parseBodyEnvVariables = (body: string, env: Environment["variables"]) =>
  pipe(
    parseBodyEnvVariablesE(body, env),
    E.getOrElse(() => body)  // 🔴 解析失败返回原始 body
  )
```

#### 9.4.3 CLI 请求构造阶段解析函数调用表

**CLI getEffectiveRESTRequest** (`packages/hoppscotch-cli/src/utils/pre-request.ts:144-500`)：

| 字段 | 解析函数 | 错误回退 | PARSING_ERROR 触发条件 |
|------|----------|----------|------------------------|
| URL | `parseTemplateStringE` | 返回 `Left(PARSING_ERROR)` | 只有递归达到上限（ENV_EXPAND_LOOP）|
| Header Key/Value | `getEffectiveFinalMetaData` 内调 `parseTemplateStringE` | 返回 `Left(PARSING_ERROR)` | 只要有任何一项解析返回 Left |
| Param Key/Value | `getEffectiveFinalMetaData` 内调 `parseTemplateStringE` | 返回 `Left(PARSING_ERROR)` | 只要有任何一项解析返回 Left |
| Body JSON | `parseBodyEnvVariablesE` | 返回 `Left(PARSING_ERROR)` | 只有递归达到上限（ENV_EXPAND_LOOP）|
| Body form-urlencoded | 每个 key/value 调 `parseTemplateStringE` | 过滤掉解析失败的项 | 只有 `parseRawKeyValueEntriesE` 失败时，而非单个项解析失败 |
| Body multipart | 文本字段调 `parseTemplateString` | 解析失败返回空字符串 | 永远不会报错 |

**代码证据 1 - URL 解析** (`pre-request.ts:454-465`)：
```typescript
const _effectiveFinalURL = parseTemplateStringE(endpoint, resolvedVariables);
if (E.isLeft(_effectiveFinalURL)) {
  return E.left(
    error({
      code: "PARSING_ERROR",
      data: `${request.endpoint} (${_effectiveFinalURL.left})`,
    })
  );
}
// 🔴 注意：parseTemplateStringE 只有 ENV_EXPAND_LOOP 会返回 Left
// 变量未命中只会返回空字符串，不会触发 PARSING_ERROR
```

**代码证据 2 - Header/Param 解析** (`getters.ts:84-91`)：
```typescript
E.fromPredicate(
  A.every(({ key, value }) => E.isRight(key) && E.isRight(value)),
  (reason) => error({ code: "PARSING_ERROR", data: reason })
)
// 🔴 A.every 检查所有项，只要有一个 Left 就整体返回 Left
// 但 parseTemplateStringE 只有 ENV_EXPAND_LOOP 会返回 Left
// 所以实际上只有递归上限才会触发 PARSING_ERROR
```

**代码证据 3 - form-urlencoded 解析** (`pre-request.ts:529-541`)：
```typescript
A.map(({ key, value }) => [
  parseTemplateStringE(key, resolvedVariables),
  parseTemplateStringE(value, resolvedVariables),
]),
A.filterMap(([key, value]) =>
  E.isRight(key) && E.isRight(value)
    ? O.some([key.right, value.right] as [string, string])
    : O.none
)
// 🔴 单个项解析失败只是被过滤掉，不会返回错误
// 只有 parseRawKeyValueEntriesE 解析原始格式失败时才会报错
```

**重要修正**：之前的分析"变量不存在 CLI 返回错误"是绝对化的错误判断。实际上：
1. **变量未命中不会返回 Left**，只会返回空字符串或保留占位符
2. **只有 ENV_EXPAND_LOOP（递归达到上限）会返回 Left**
3. **form-urlencoded 中单个项解析失败被过滤**，不会导致整体 PARSING_ERROR
4. **Header/Param 的 PARSING_ERROR 只有在递归上限时才会触发**，因为 `A.every` 检查所有项的 Either 状态

#### 9.4.4 parseTemplateStringE 变量未命中时的默认返回路径

**完整的分支逻辑** (`environment/index.ts:135-163`)：

```typescript
const variable = variables.find((x) => x && x.key === p1)

// 🔴 分支1：变量存在且有 currentValue
if (variable && "currentValue" in variable) {
  if (variable.secret && showKeyIfSecret) {
    isSecret = true
    return `<<${p1}>>`  // 返回原始占位符，标记为 secret
  }
  if (variable.secret && maskValue) {
    return "*".repeat(variable.currentValue.length)  // 返回掩码
  }
  return variable.currentValue  // 返回实际值
}

// 🔴 分支2：变量未找到
if (showKeyIfNotFound) {
  return `<<${p1}>>`  // 保留原始占位符
}

// 🔴 分支3：默认路径 - 变量未找到且不保留占位符
return ""  // 返回空字符串
```

**变量查找的边缘情况**：
- `variable && "currentValue" in variable`：要求变量不仅存在，还必须有 `currentValue` 属性
- `variables.find((x) => x && x.key === p1)`：`x &&` 跳过 falsy 元素（null/undefined）
- 如果变量存在但没有 `currentValue`，会落入"未找到"分支，返回空字符串或保留占位符

**反例 - 变量存在但无 currentValue**：
```typescript
// 变量列表中有这个 key，但结构不符合预期
const variables = [{ key: "foo", initialValue: "bar" }] // 没有 currentValue!
parseTemplateStringE("<<foo>>", variables)
// 🔴 返回空字符串 ""，因为 "currentValue" in variable 为 false
```

#### 9.4.6 前端 parseTemplateString 与 CLI parseTemplateStringE 错误暴露方式的真实差异

**前端 parseTemplateString 行为**：
```typescript
// environment/index.ts:192-208
export const parseTemplateString = (str, variables, maskValue, showKeyIfSecret, showKeyIfNotFound) =>
  pipe(
    parseTemplateStringE(str, variables, maskValue, showKeyIfSecret, showKeyIfNotFound),
    E.getOrElse(() => str)  // 🔴 唯一的错误处理：返回原始字符串
  )
```

**CLI parseTemplateStringE 行为**：
```typescript
// pre-request.ts:454-465
const _effectiveFinalURL = parseTemplateStringE(endpoint, resolvedVariables);
if (E.isLeft(_effectiveFinalURL)) {
  return E.left(error({ code: "PARSING_ERROR", data: `${request.endpoint} (${_effectiveFinalURL.left})` }));
}
```

**真实差异对比表**：

| 场景 | 前端 parseTemplateString | CLI parseTemplateStringE |
|------|--------------------------|---------------------------|
| 变量不存在 | 返回空字符串（或 `<<key>>` 仅 URL） | 返回空字符串 |
| 变量存在但无 currentValue | 返回空字符串 | 返回空字符串 |
| 自引用 A→A | 返回 `<<A>>`（第2次 break） | 返回 `<<A>>`（第2次 break） |
| 循环引用 A→B→A | 返回原始字符串（被 getOrElse 捕获） | 返回 `Left(ENV_EXPAND_LOOP)` → 终止请求 |
| 12 层嵌套引用 | 返回原始字符串（被 getOrElse 捕获） | 返回 `Left(ENV_EXPAND_LOOP)` → 终止请求 |
| str 为 null/undefined | 返回原值（函数入口判断） | 返回原值（函数入口判断） |
| variables 为 null/undefined | 返回原值（函数入口判断） | 返回原值（函数入口判断） |

**重要修正**：
1. ❌ 之前的绝对化判断："CLI 变量不存在返回错误" → ✅ 变量不存在不会返回错误，只有递归上限会
2. ❌ 之前的绝对化判断："前端循环引用静默返回原始字符串" → ✅ 自引用 A→A 不会触发错误，只有两两循环 A→B→A 会
3. ❌ 之前的绝对化判断："前端和 CLI 行为完全不同" → ✅ 除了递归上限场景，其他场景行为一致

#### 9.4.7 常见坑点

1. **URL 预览 vs 实际发送**：前端预览时 `showKeyIfNotFound=true` 显示 `<<key>>`，实际发送时 `showKeyIfNotFound=false` 替换为空字符串
2. **Header 空值**：变量未找到时 header 值为空字符串，该 header 仍会被发送
3. **CLI 严格性**：CLI 中只有递归上限场景才会导致请求终止，变量不存在只是替换为空字符串，前端静默降级
4. **变量结构检查**：变量存在但没有 `currentValue` 属性时，会落入"未找到"分支，返回空字符串
5. **Form Data 文件**：文件内容不解析，文件名会被解析

**关键代码位置**：`EffectiveURL.ts:362-446`, `pre-request.ts:144-500`, `environment/index.ts:55-208`

---

### 9.5 递归上限边界行为

**核心问题**：递归上限 10 层到底是怎么计算的？循环检测在什么情况下生效？

**答案**：**上限是 11 次替换尝试（0~10），两种解析函数行为不同**

#### 9.5.1 深度计数规则

```typescript
const ENV_MAX_EXPAND_LIMIT = 10

let depth = 0
while (
  result.match(REGEX_ENV_VAR) != null &&
  depth <= ENV_MAX_EXPAND_LIMIT  // 🔴 <= 意味着 depth 可以是 0,1,...,10（共11次）
) {
  // 执行替换
  depth++
}

return depth > ENV_MAX_EXPAND_LIMIT  // 🔴 > 意味着 depth=11 时才触发错误
  ? E.left(ENV_EXPAND_LOOP)
  : E.right(result)
```

**执行次数分析**：
- depth 初始为 0
- 第 1 次替换后 depth=1
- ...
- 第 11 次替换后 depth=11
- 循环条件 `11 <= 10` 不成立，退出
- 判断 `11 > 10` 为 true，返回错误

**结论**：最多执行 **11 次替换**。

#### 9.5.2 两种解析函数的边界行为差异

| 场景 | `parseTemplateStringE` | `parseBodyEnvVariablesE` |
|------|------------------------|--------------------------|
| 10 层以内解析完成 | ✅ 返回 `Right` | ✅ 返回 `Right` |
| 11 次替换后仍有占位符 | ❌ 返回 `Left(ENV_EXPAND_LOOP)` | ❌ 返回 `Left(ENV_EXPAND_LOOP)` |
| 中间某次无替换（如变量未找到） | ✅ 立即 break，返回 `Right` | ❌ 继续循环直到 depth=11 |
| 循环引用（A→B→A） | ❌ 执行 11 次后返回 `Left` | ❌ 执行 11 次后返回 `Left` |
| 自引用（A→A） | ✅ 第 2 次后无变化，break，返回 `Right` | ❌ 执行 11 次后返回 `Left` |

#### 9.5.3 循环引用真实停止条件详解

**测试用例** (`packages/hoppscotch-js-sandbox/src/__tests__/pw-namespace/env/resolve.spec.ts:119-154`)：

```typescript
test("if infinite loop in resolution, abandons resolutions altogether", () => {
  return expect(
    runTest(
      `const data = pw.env.resolve("<<hello>>")
       pw.expect(data).toBe("<<hello>>")`,
      {
        selected: [
          { key: "hello", currentValue: "<<there>>", ... },
          { key: "there", currentValue: "<<hello>>", ... },
        ],
      }
    )()
  ).resolves.toEqualRight(...)
})
```

**parseTemplateStringE 真实执行过程** (`environment/index.ts:103-178`)：

```
初始值: result = "<<hello>>", depth = 0

第 1 轮循环 (depth=0, 条件 0 <= 10: true):
  currentResult = "<<hello>>".replace(...) → "<<there>>"
  currentResult !== result → 继续
  result = "<<there>>"
  depth = 1

第 2 轮循环 (depth=1, 条件 1 <= 10: true):
  currentResult = "<<there>>".replace(...) → "<<hello>>"
  currentResult !== result → 继续
  result = "<<hello>>"
  depth = 2

第 3 轮循环 (depth=2, 条件 2 <= 10: true):
  currentResult = "<<hello>>".replace(...) → "<<there>>"
  currentResult !== result → 继续
  result = "<<there>>"
  depth = 3

第 4 轮循环 (depth=3, 条件 3 <= 10: true):
  currentResult = "<<there>>".replace(...) → "<<hello>>"
  currentResult !== result → 继续
  result = "<<hello>>"
  depth = 4

... 持续交替 ...

第 11 轮循环 (depth=10, 条件 10 <= 10: true):
  currentResult = 替换 → 另一个占位符
  currentResult !== result → 继续
  result = 另一个占位符
  depth = 11

循环终止: depth=11, 条件 11 <= 10: false

返回: depth > 10 → E.left(ENV_EXPAND_LOOP)

🔴 然后 parseTemplateString 包装器捕获错误并返回原始字符串 "<<hello>>"
```

**关键证据**：`parseTemplateStringE` 中循环条件是 `depth <= ENV_MAX_EXPAND_LIMIT`（`index.ts:118-122`）：
```typescript
while (
  result.match(REGEX_ENV_VAR) != null &&
  depth <= ENV_MAX_EXPAND_LIMIT &&  // 🔴 <= 意味着 depth=10 时仍会执行
  !isSecret
) {
  // 执行替换
  // ...
  if (currentResult === result) break  // 🔴 只有当字符串完全无变化时才 break
  result = currentResult
  depth++  // 🔴 depth 在替换后递增
}

return depth > ENV_MAX_EXPAND_LIMIT
  ? E.left(ENV_EXPAND_LOOP)  // 🔴 depth=11 时返回错误
  : E.right(result)
```

**为什么不会被 early break 终止？**

对于 `A = <<B>>, B = <<A>>`：
- 每次替换后字符串都会变化（`<<hello>>` ↔ `<<there>>`）
- `currentResult === result` 永远为 false
- 所以不会触发 early break
- 会一直执行到 `depth = 11` 才退出

**真实行为修正**：
| 场景 | parseTemplateStringE | parseBodyEnvVariablesE |
|------|------------------------|--------------------------|
| `A = <<B>>, B = <<A>>` | 执行 11 次替换 → 返回 `Left(ENV_EXPAND_LOOP)` | 执行 11 次替换 → 返回 `Left(ENV_EXPAND_LOOP)` |
| `A = <<A>>`（自引用） | 第 2 次替换后无变化 → break → 返回 `Right("<<A>>")` | 执行 11 次替换 → 返回 `Left(ENV_EXPAND_LOOP)` |

**parseBodyEnvVariablesE 的差异** (`environment/index.ts:55-89`)：
```typescript
while (result.match(REGEX_ENV_VAR) != null && depth <= ENV_MAX_EXPAND_LIMIT) {
  result = result.replace(REGEX_ENV_VAR, (key) => {
    const variableName = key.replace(/[<>]/g, "")
    const foundEnv = env.find((envVar) => envVar.key === variableName)
    if (foundEnv && "currentValue" in foundEnv) {
      return foundEnv.currentValue
    }
    return key  // 🔴 未找到时返回原始 <<key>>，而不是空字符串
  })
  depth++  // 🔴 无条件递增，没有 early break 检查
}
```

**关键差异**：
1. **变量未找到时**：`parseBodyEnvVariablesE` 返回原始 `<<key>>`，`parseTemplateStringE` 返回空字符串（或保留）
2. **Early break**：`parseBodyEnvVariablesE` 没有 `currentResult === result` 检查
3. **自引用处理**：`A = <<A>>` 在 `parseBodyEnvVariablesE` 中会执行 11 次替换后报错，而在 `parseTemplateStringE` 中第 2 次就 break

**测试用例为什么能通过？**

因为测试用例调用的是 `pw.env.resolve()`，它使用 `parseTemplateStringE` + `E.getOrElse(() => valueToUse)`（`shared.ts:298-302`）：
```typescript
return pipe(
  parseTemplateStringE(valueToUse, envVars),
  E.getOrElse(() => valueToUse)  // 🔴 捕获 ENV_EXPAND_LOOP 错误，返回原始值
)
```

**结论修正**：
- 两两循环引用（A→B→A）**不会被 early break 检测到**，因为每次替换后字符串都在变化
- 会执行满 11 次替换后返回 `ENV_EXPAND_LOOP` 错误
- 但在前端 UI 中，由于 `parseTemplateString` 的 `E.getOrElse(() => str)` 包装，用户会看到原始字符串
- 只有自引用（A→A）或替换后字符串完全相同时才会触发 early break

#### 9.5.4 边界场景测试

| 场景 | 结果 | 原因 |
|------|------|------|
| `A = <<B>>, B = <<C>>, C = value` | ✅ 解析为 value | 3 次替换，在限制内 |
| 嵌套 12 层变量引用 | ❌ ENV_EXPAND_LOOP | 超过 11 次限制 |
| `A = <<A>>`（自引用）parseTemplateStringE | ✅ 保留 `<<A>>` | 第 2 次无变化，break |
| `A = <<A>>`（自引用）parseBodyEnvVariablesE | ❌ ENV_EXPAND_LOOP | 无 early break，执行 11 次 |
| JSON Body 中 `A = <<B>>, B = <<A>>` | ❌ ENV_EXPAND_LOOP | 每次都有变化，执行 11 次 |
| URL 中 `A = <<B>>, B = <<A>>` | 前端显示 `<<A>>` | parseTemplateString 返回错误被 getOrElse 捕获 |
| CLI URL 中 `A = <<B>>, B = <<A>>` | ❌ 请求失败 PARSING_ERROR | CLI 直接返回 Left 错误 |

#### 9.5.5 错误处理差异

- **前端 UI**：`parseTemplateString`（非 E 版本）会捕获错误并返回原始字符串
- **请求执行**：`parseTemplateStringE` 返回 `Either`，调用方决定如何处理
- **Body JSON**：`parseBodyEnvVariablesE` 返回 `Left` 时，调用方通常返回 null
- **CLI**：错误会导致请求失败并在报告中显示

**关键代码位置**：`environment/index.ts:45-178`

---

## 10. 总结：容易踩坑的要点清单

### 10.1 已纠正的绝对化判断

| 之前的错误判断 | 修正后的正确结论 | 代码证据 |
|----------------|------------------|----------|
| CLI 变量不存在返回错误 | 变量不存在返回空字符串，只有递归上限返回错误 | `environment/index.ts:135-163` |
| 前端所有循环引用静默返回原始值 | 自引用 A→A 第 2 次 break，正常返回；只有两两循环 A→B→A 会触发上限 | `environment/index.ts:167-169` |
| parseTemplateStringE 有多种错误 | 只有 `ENV_EXPAND_LOOP` 一种错误类型 | `environment/index.ts:175-177` |
| CLI form-urlencoded 单个项解析失败报错 | 单个项失败被过滤掉，只有原始格式解析失败才报错 | `pre-request.ts:529-541` |
| 前端和 CLI 行为完全不同 | 除递归上限场景外，其他场景行为一致 | `environment/index.ts:192-208` |
| 变量存在就会命中 | 变量必须有 `currentValue` 属性才会命中，否则返回空字符串 | `environment/index.ts:137` |

### 10.2 updateEnvsAfterTestScript 关键点
- **真实入参**：第1参是 `E.Right<SandboxTestResult>`（含 envs），不是单独的 finalEnvs
- **更新路径**：全局走 `setGlobalEnvVariables`，个人环境走 `updateEnvironment`，团队环境走 `updateTeamEnvironment`
- **深拷贝**：选中环境更新前会 `cloneDeep`，避免引用共享

### 10.3 URL/Header/Params/Body 解析差异
- **URL 独有**：唯一传入 `showKeyIfNotFound` 的位置，预览时显示 `<<key>>`
- **前端容错**：`parseTemplateString` 用 `E.getOrElse(() => str)` 吞掉错误，返回原始字符串
- **CLI 严格**：只有递归上限时 `parseTemplateStringE` 返回 `Left` 错误，直接终止请求
- **Body 多策略**：JSON 用 `parseBodyEnvVariablesE`，form 逐个解析，文件不解析

### 10.4 parseTemplateStringE 变量未命中分支
```typescript
// 分支1：变量存在且有 currentValue → 返回 currentValue（或掩码/占位符）
if (variable && "currentValue" in variable) { ... }

// 分支2：变量未找到但 showKeyIfNotFound=true → 返回 "<<p1>>"
if (showKeyIfNotFound) { return `<<${p1}>>` }

// 分支3：默认路径 → 返回空字符串 ""
return ""
```

### 10.5 CLI PARSING_ERROR 触发场景

| 字段 | 触发 PARSING_ERROR | 过滤失败项 |
|------|---------------------|------------|
| URL | 递归达到上限 | ❌ |
| Header | 任何一项递归达到上限 | ❌ |
| Param | 任何一项递归达到上限 | ❌ |
| Body JSON | 递归达到上限 | ❌ |
| Body form-urlencoded | 原始格式解析失败 | ✅ 单个项失败被过滤 |
| Body multipart | 永远不会 | ✅ 失败返回空字符串 |

### 10.6 循环引用真实行为
- **两两循环（A→B→A）**：不会被 early break 检测到，执行满 11 次替换后返回 `ENV_EXPAND_LOOP`
- **自引用（A→A）**：`parseTemplateStringE` 第 2 次无变化 break，`parseBodyEnvVariablesE` 执行满 11 次报错
- **前端表面正常**：`parseTemplateString` 包装器捕获错误返回原始值，用户看不到错误
- **CLI 暴露错误**：CLI 递归上限时返回 `PARSING_ERROR`，请求终止

### 10.7 CLI 与前端同键空值回退差异
- **前端**：高优先级变量 `currentValue === ""` 且 `initialValue` 也为空时，回退到低优先级非空值
- **CLI**：高优先级只要存在 key（无论值是否为空），低优先级同 key 直接被过滤
- **请求变量过滤**：CLI 先过滤掉 `active=false` 或 `value` 为空的请求变量，前端保留所有

### 10.8 其他容易误判的细节
1. **空格非空**：`||` 运算符将空格 `" "` 视为 truthy，不会触发回退
2. **并发安全**：每个请求都有独立的环境快照，脚本修改不会互相串写
3. **增量更新**：只有与初始快照不同的变量才会被写回
4. **递归上限**：名义 10 层，实际最多 11 次替换（`depth <= 10` 循环 + `depth > 10` 判断）
5. **集合继承**：子集合变量优先于父集合，扁平化后后遍历覆盖先遍历
