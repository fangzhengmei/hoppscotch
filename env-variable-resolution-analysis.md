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

#### 9.2.3 更新时的冲突检测

**updateEnvsAfterTestScript** (`RequestRunner.ts:748-837`)：

```typescript
const updateEnvsAfterTestScript = (
  finalEnvs: TestResult["envs"],
  initialEnvs: TestResult["envs"],
  envIndex: SelectedEnvironmentIndex,
  initialEnvID: { selected: string; global: string },
  updateGlobal: boolean,
  updateSelected: boolean
) => {
  // 🔴 只有与初始快照不同的变量才会被更新
  const globalVarsToUpdate = finalEnvs.global.filter(
    (env) => !A.elem(isEqual(env))(initialEnvs.global)
  )

  const selectedVarsToUpdate = finalEnvs.selected.filter(
    (env) => !A.elem(isEqual(env))(initialEnvs.selected)
  )

  // 执行增量更新
  globalVarsToUpdate.forEach((env) => updateEnvironments(...))
  selectedVarsToUpdate.forEach((env) => updateEnvironments(...))
}
```

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

**关键差异**：CLI 中使用 `filter` 直接移除低优先级同名变量，而前端使用 `filterNonEmptyEnvironmentVariables` 的回退逻辑。

**关键代码位置**：`inheritedCollectionVarTransformer.ts:38-64`, `getters.ts:276-335`

---

### 9.4 URL、Header、Params 与 Body 的模板解析差异

**核心问题**：不同位置的模板解析行为是否一致？

**答案**：**不一致，存在多处细微差异**

#### 9.4.1 解析行为对比表

| 特性 | URL | Headers | Params | Body |
|------|-----|---------|--------|------|
| 解析函数 | `parseTemplateStringE` | `parseTemplateString` | `parseTemplateString` | 因类型而异 |
| 支持 `showKeyIfNotFound` | ✅ 支持 | ❌ 不支持 | ❌ 不支持 | ❌ 不支持 |
| 未找到变量时 | 返回 `<<key>>` 或空 | 返回空字符串 | 返回空字符串 | 因类型而异 |
| 递归解析 | ✅ 支持 | ✅ 支持 | ✅ 支持 | ✅ 支持（JSON/form） |
| 掩码支持 | ✅ 支持 | ✅ 支持 | ✅ 支持 | ❌ 不支持 |
| 循环检测 | ✅ 有 | ✅ 有 | ✅ 有 | ✅ 有 |
| 循环时行为 | 返回 `Left` 错误 | 静默保留原样 | 静默保留原样 | 返回 `Left` 或静默 |

#### 9.4.2 URL 解析特殊处理

**getEffectiveRESTRequest** (`packages/hoppscotch-common/src/helpers/utils/EffectiveURL.ts:432-445`)：

```typescript
const effectiveFinalURL = parseTemplateString(
  request.endpoint,
  environment.variables,
  false,              // maskValue = false
  showKeyIfSecret,    // 可配置
  showKeyIfNotFound   // 🔴 独有参数：未找到时保留 <<key>>
)
```

**用途**：预览 URL 时，为了让用户看到哪些变量未解析，会保留 `<<key>>` 格式。

#### 9.4.3 Body 解析特殊处理

**getFinalBodyFromRequest** (`EffectiveURL.ts:256-350`) 根据 content-type 有不同行为：

| Content-Type | 解析方式 | 未找到变量 |
|--------------|----------|------------|
| `application/json` | `parseBodyEnvVariablesE` | 保留 `<<key>>` |
| `application/x-www-form-urlencoded` | 解析为 key-value 后逐个 `parseTemplateStringE` | 过滤掉解析失败的 |
| `multipart/form-data` | 文本字段用 `parseTemplateString`，文件跳过 | 文件内容不解析 |
| `application/octet-stream` | 完全不解析 | 不解析 |
| 其他（text/plain 等） | `parseBodyEnvVariablesE` | 保留 `<<key>>` |

**关键差异**：`parseBodyEnvVariablesE` 与 `parseTemplateStringE` 的循环检测行为不同：

```typescript
// parseBodyEnvVariablesE：无 early break，会完整执行到上限
while (result.match(REGEX_ENV_VAR) != null && depth <= ENV_MAX_EXPAND_LIMIT) {
  result = result.replace(REGEX_ENV_VAR, ...)
  depth++  // 🔴 不管有没有变化，depth 都会递增
}

// parseTemplateStringE：有 early break
const currentResult = result.replace(...)
if (currentResult === result) {
  break  // 🔴 无变化则立即终止
}
```

**影响**：Body 中的循环引用会更快达到递归上限（11 次替换 vs 可能更少的次数）。

#### 9.4.4 常见坑点

1. **URL 中变量未找到**：预览时显示 `<<key>>`，实际发送时为空字符串
2. **JSON Body 中的循环**：`{ "a": "<<b>>", "b": "<<a>>" }` 会触发 ENV_EXPAND_LOOP 错误
3. **Form Data 中的文件**：文件内容不会被解析，文件名会被解析
4. **Header 中的空值**：变量未找到时 header 值为空字符串，该 header 仍会被发送

**关键代码位置**：`EffectiveURL.ts:362-446`, `environment/index.ts:55-89`, `environment/index.ts:103-178`

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
| 循环引用（A→B→A） | ✅ 第 2 次后无变化，break，返回 `Right`（保留原样） | ❌ 执行 11 次后返回 `Left` |

#### 9.5.3 循环引用行为详解

**测试用例** (`packages/hoppscotch-js-sandbox/src/__tests__/pw-namespace/env/resolve.spec.ts:119-154`)：

```typescript
test("if infinite loop in resolution, abandons resolutions altogether", () => {
  return expect(
    runTest(
      `const data = pw.env.resolve("<<hello>>")
       pw.expect(data).toBe("<<hello>>")`,  // 🔴 期望保留原始值
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

**parseTemplateStringE 执行过程**：
```
初始: "<<hello>>"
depth=0: 替换为 "<<there>>"  → 有变化，depth=1
depth=1: 替换为 "<<hello>>"  → 有变化，depth=2
depth=2: 替换为 "<<there>>"  → 有变化，depth=3
... 会在两个值之间来回替换吗？

🔴 实际上：
depth=2: currentResult="<<hello>>", result 之前也是 "<<hello>>"
→ currentResult === result → break 循环
→ 返回 Right("<<hello>>")
```

**为什么 depth=2 就终止了？**

因为 `depth=0` 时 `result="<<hello>>"`，替换后 `currentResult="<<there>>"`，不同，继续。
`depth=1` 时 `result="<<there>>"`，替换后 `currentResult="<<hello>>"`，不同，继续。
`depth=2` 时 `result="<<hello>>"`，替换后 `currentResult="<<hello>>"`，**相同**，break。

**结论**：对于两两循环，最多执行 **2 次替换** 就会被 early break 检测到。

#### 9.5.4 边界场景测试

| 场景 | 结果 | 原因 |
|------|------|------|
| `A = <<B>>, B = <<C>>, C = value` | ✅ 解析为 value | 3 次替换，在限制内 |
| 嵌套 12 层变量引用 | ❌ ENV_EXPAND_LOOP | 超过 11 次限制 |
| `A = <<A>>`（自引用） | ✅ 保留 `<<A>>` | 第 2 次无变化，break |
| JSON Body 中 `A = <<B>>, B = <<A>>` | ❌ ENV_EXPAND_LOOP | parseBodyEnvVariablesE 无 early break |
| URL 中 `A = <<B>>, B = <<A>>` | ✅ 保留 `<<A>>` | parseTemplateStringE 有 early break |

#### 9.5.5 错误处理差异

- **前端 UI**：`parseTemplateString`（非 E 版本）会捕获错误并返回原始字符串
- **请求执行**：`parseTemplateStringE` 返回 `Either`，调用方决定如何处理
- **Body JSON**：`parseBodyEnvVariablesE` 返回 `Left` 时，调用方通常返回 null
- **CLI**：错误会导致请求失败并在报告中显示

**关键代码位置**：`environment/index.ts:45-178`

---

## 10. 总结：容易踩坑的要点清单

1. **空值回退**：高优先级变量为空时，会先尝试自身 initialValue，再考虑低优先级
2. **空格非空**：`||` 运算符将空格视为 truthy，不会触发回退
3. **并发安全**：每个请求都有独立的环境快照，脚本修改不会互相串写
4. **增量更新**：只有与初始快照不同的变量才会被写回
5. **集合继承**：子集合变量优先于父集合，扁平化后后遍历覆盖先遍历
6. **CLI 差异**：CLI 使用 filter 直接移除低优先级同名变量，无回退逻辑
7. **URL 预览**：URL 解析支持 `showKeyIfNotFound`，预览时显示 `<<key>>`
8. **Body 解析**：不同 content-type 解析策略不同，文件不解析
9. **循环检测**：`parseTemplateStringE` 有 early break，`parseBodyEnvVariablesE` 没有
10. **递归上限**：名义 10 层，实际最多 11 次替换，两两循环在 2 次后被检测到
