# Secret 环境变量处理机制分析

## 1. 数据结构与版本演进

### 1.1 环境变量数据结构 (V2)

在 `packages/hoppscotch-data/src/environment/v/2.ts` 中定义：

```typescript
{
  key: string,
  initialValue: string,
  currentValue: string,
  secret: boolean
}
```

### 1.2 版本迁移策略 (V1 → V2)

从 V1 升级到 V2 时，**secret 变量的值会被清空**：
- `secret: true` 的变量：`initialValue = ""`, `currentValue = ""`
- `secret: false` 的变量：保留原值

**设计意图**：服务端数据库不存储 secret 变量的实际值，仅存储元数据。

---

## 2. 存储架构：三层分离设计

### 2.1 三层存储模型

| 层级 | 服务/Store | 存储内容 | 存储位置 | 同步到服务端 |
|------|-----------|---------|---------|------------|
| 元数据层 | `environmentsStore` | key, secret 标记, 非 secret 的值 | IndexedDB | ✅ 是 |
| 当前值层 | `CurrentValueService` | 所有变量的 currentValue | LocalStorage | ❌ 否 |
| Secret 层 | `SecretEnvironmentService` | secret 变量的真实值 | LocalStorage | ❌ 否 |

### 2.2 SecretEnvironmentService

**文件**: `packages/hoppscotch-common/src/services/secret-environment.service.ts`

核心功能：
- `secretEnvironments: Map<string, SecretVariable[]>` - 按环境 ID 分组存储
- `SecretVariable` 结构：
  ```typescript
  {
    key: string,
    value: string,           // 当前值（运行时）
    varIndex: number,        // 索引位置匹配
    initialValue?: string    // 初始值
  }
  ```

关键方法：
- `getSecretEnvironmentVariableValue(id, varIndex)` - 获取 secret 值
- `hasSecretValue(id, key)` - 检查是否有值
- `persistableSecretEnvironments` - 可持久化的数据

### 2.3 CurrentValueService

**文件**: `packages/hoppscotch-common/src/services/current-environment-value.service.ts`

存储所有环境变量（包括非 secret）的当前值，用于运行时临时修改。

---

## 3. 变量解析流程

### 3.1 运行时值合并 (unWrapEnvironments)

**文件**: `packages/hoppscotch-common/src/helpers/utils/environments.ts`

```typescript
function unWrapEnvironments(selected, global) {
  // 1. 对 Global 变量：从 SecretEnvironmentService 获取 secret 值
  // 2. 对 Selected 环境变量：从 SecretEnvironmentService 获取 secret 值
  // 3. 非 secret 变量：从 CurrentValueService 或环境元数据获取
}
```

### 3.2 聚合环境变量 (aggregateEnvsWithCurrentValue$)

**文件**: `packages/hoppscotch-common/src/newstore/environments.ts`

优先级顺序：
1. **预定义变量** (Pre-defined)
2. **请求变量** (Request Variables)
3. **选中环境变量** (Selected Environment)
4. **全局环境变量** (Global Environment)

Secret 变量处理：
```typescript
if (x.secret) {
  currentValue = secretEnvironmentService
    .getSecretEnvironmentVariableValue(envId, index)?.value ?? ""
}
```

### 3.3 模板字符串解析

**文件**: `packages/hoppscotch-data/src/environment/index.ts`

`parseTemplateStringE(str, variables, maskValue, showKeyIfSecret, showKeyIfNotFound)`

参数说明：
- `maskValue: boolean` - secret 值用星号遮蔽
- `showKeyIfSecret: boolean` - secret 变量保留 `<<key>>` 形式不解析
- `showKeyIfNotFound: boolean` - 未找到的变量保留 `<<key>>`

遮蔽策略：
```typescript
if (variable.secret && maskValue) {
  return "*".repeat(variable.currentValue.length)
}
```

---

## 4. 遮蔽渲染策略

### 4.1 编辑器悬停提示 (HoppEnvironment)

**文件**: `packages/hoppscotch-common/src/helpers/editor/extensions/HoppEnvironment.ts`

悬停时 secret 值显示规则：

| 状态 | Initial Value | Current Value |
|------|--------------|--------------|
| 都有值 | `******` | `******` |
| 仅 Initial 有值 | `******` | `Empty` |
| 仅 Current 有值 | `Empty` | `******` |
| 都无值 | `Empty` | `Empty` |

关键代码 (168-180行)：
```typescript
if (isSecret) {
  if (hasSecretValueStored && hasSecretInitialValueStored) {
    envInitialValue = "******"
    envCurrentValue = "******"
  }
  // ... 其他分支
}
```

### 4.2 环境变量高亮

根据来源区分高亮样式：
- `HOPP_GLOBAL_ENVIRONMENT_HIGHLIGHT` - 全局环境
- `HOPP_ENVIRONMENT_HIGHLIGHT` - 普通环境
- `HOPP_REQUEST_VARIABLE_HIGHLIGHT` - 请求变量
- `HOPP_COLLECTION_ENVIRONMENT_HIGHLIGHT` - 集合变量

---

## 5. 导出策略

### 5.1 单环境导出

**文件**: `packages/hoppscotch-common/src/helpers/import-export/export/environment.ts`

`transformEnvironmentVariables()` 函数处理：

```typescript
return {
  key,
  secret,
  initialValue,
  currentValue: variable.secret ? "" : (variable.currentValue ?? "")
}
```

**规则**：
- Secret 变量：`currentValue` 被清空为 `""`
- 非 Secret 变量：保留 `currentValue`
- `initialValue` 始终保留（但 Secret 的 initialValue 在服务端本身就是空的）

### 5.2 批量环境导出

**文件**: `packages/hoppscotch-common/src/helpers/import-export/export/environments.ts`

直接 `JSON.stringify()` 导出，**不做特殊处理**。

---

## 6. 脚本沙箱中的访问

### 6.1 共享环境方法

**文件**: `packages/hoppscotch-js-sandbox/src/utils/shared.ts`

脚本中可访问：

```typescript
// pw 命名空间
pw.env.get(key, options)
pw.env.getResolve(key, options)  // 递归解析模板
pw.env.set(key, value, options)
pw.env.unset(key, options)
pw.env.resolve(value)           // 解析字符串中的变量

// hopp 命名空间
hopp.env.set(key, value, options)
hopp.env.delete(key, options)
hopp.env.reset(key, options)    // currentValue 重置为 initialValue
hopp.env.getInitialRaw(key, options)
hopp.env.setInitial(key, value, options)
```

### 6.2 值解析逻辑

`envGetFn` 中的值获取优先级：
1. `currentValue`（非空时）
2. 否则 fallback 到 `initialValue`

---

## 7. 持久化与同步

### 7.1 本地持久化

**文件**: `packages/hoppscotch-common/src/services/persistence/index.ts`

持久化的 Key：
- `secretEnvironments` - Secret 变量值
- `currentEnvironmentValue` - 当前值
- `environments` - 环境元数据（不含 secret 值）

**重要**：secret 值仅存在于浏览器本地存储，**不会同步到服务端**。

### 7.2 服务端同步

- 环境元数据（key, secret 标记）同步到服务端
- secret 的 `initialValue` 和 `currentValue` 在服务端存储为空字符串
- 真实值仅在本地 SecretEnvironmentService 中维护

---

## 8. 持久化加载与运行时恢复链路

### 8.1 应用启动时的加载顺序

**文件**: `packages/hoppscotch-common/src/services/persistence/index.ts`

应用启动时 `PersistenceService` 按以下顺序初始化：

```
1. setupEnvironmentsPersistence()      → 加载环境元数据到 environmentsStore
2. setupSecretEnvironmentsPersistence() → 加载 secret 值到 SecretEnvironmentService
3. setupCurrentEnvironmentValuePersistence() → 加载当前值到 CurrentValueService
4. setupSelectedEnvPersistence()       → 恢复选中的环境索引
```

**Secret 环境加载流程** (`setupSecretEnvironmentsPersistence`, 727-771行):

```typescript
// 1. 从本地存储读取
const loadResult = await Store.get(STORE_NAMESPACE, STORE_KEYS.SECRET_ENVIRONMENTS)

// 2. Schema 验证
const result = SECRET_ENVIRONMENT_VARIABLE_SCHEMA.safeParse(loadResult.right)

// 3. 加载到服务
if (result.success) {
  this.secretEnvironmentService.loadSecretEnvironmentsFromPersistedState(result.data)
}
```

**加载方法** (`loadSecretEnvironmentsFromPersistedState`):

```typescript
public loadSecretEnvironmentsFromPersistedState(
  secretEnvironments: Record<string, SecretVariable[]>
) {
  if (secretEnvironments) {
    this.secretEnvironments.clear()  // 先清空
    Object.entries(secretEnvironments).forEach(([id, secretVars]) => {
      this.addSecretEnvironment(id, secretVars)
    })
  }
}
```

### 8.2 varIndex 对齐机制

**核心设计**：Secret 值通过 `varIndex`（数组索引）与环境元数据中的变量位置对齐。

**数据结构对应关系**：

```
environmentsStore.variables[0]  ←→  SecretEnvironmentService["envId"][0].varIndex = 0
environmentsStore.variables[1]  ←→  SecretEnvironmentService["envId"][1].varIndex = 1
environmentsStore.variables[2]  ←→  SecretEnvironmentService["envId"][2].varIndex = 2
```

**恢复时的索引查找** (`resolveEnvVars`, RequestRunner.ts:1044-1062):

```typescript
const resolveEnvVars = (envID: string, vars: Environment["variables"]) =>
  vars.map((v, index) => {
    const secretMeta = v.secret
      ? getSecretEnvironmentVariableValue(envID, index)  // 用 index 查找
      : null
    return {
      ...v,
      currentValue: v.secret ? secretMeta?.value : ...,
      initialValue: v.secret ? secretMeta?.initialValue : ...,
    }
  })
```

### 8.3 运行时值恢复的触发点

#### 触发点 1: 请求执行前 (captureInitialEnvironmentState)

**文件**: `packages/hoppscotch-common/src/helpers/RequestRunner.ts:118-153`

```typescript
export const captureInitialEnvironmentState = (): InitialEnvironmentState => {
  // 解析 Global 变量的 secret 值
  const initialGlobalEnvs = resolveEnvVars("Global", cloneDeep(getGlobalVariables()))
  
  // 解析选中环境的 secret 值
  const initialSelectedEnvs = resolveEnvVars(initialEnvID, initialEnvVariables)
  
  // 合并所有变量（包含 secret 真实值）
  const initialEnvs = getCombinedEnvVariables()
  // ...
}
```

#### 触发点 2: 环境切换时 (aggregateEnvsWithCurrentValue$)

**文件**: `packages/hoppscotch-common/src/newstore/environments.ts:619-700`

当 `currentEnvironment$` 或 `globalEnv$` 变化时，自动重新聚合：

```typescript
export const aggregateEnvsWithCurrentValue$ = combineLatest([currentEnvironment$, globalEnv$]).pipe(
  map(([selectedEnv, globalEnv]) => {
    // 遍历选中环境的变量，对 secret 变量从 SecretEnvironmentService 恢复真实值
    selectedEnv?.variables.map((x, index) => {
      if (x.secret) {
        currentValue = secretEnvironmentService
          .getSecretEnvironmentVariableValue(selectedEnv.id, index)?.value ?? ""
      }
      // ...
    })
  })
)
```

#### 触发点 3: 编辑器悬停提示 (HoppEnvironment Plugin)

**文件**: `packages/hoppscotch-common/src/helpers/editor/extensions/HoppEnvironment.ts:410-448`

```typescript
watch(() => restTabs.currentActiveTab.value, (currentTab) => {
  // 重新计算 requestAndCollVars
  // 触发 compartment.reconfigure，重新应用 cursorTooltipField
  // 悬停时实时查询 secret 值并遮蔽显示
}, { immediate: true, deep: true })
```

### 8.4 环境切换时的数据流转

```
用户切换环境
    ↓
environmentsStore.selectedEnvironmentIndex 变更
    ↓
currentEnvironment$ 发射新值
    ↓
aggregateEnvsWithCurrentValue$ 重新计算
    ↓
┌─────────────────────────────────────────────┐
│ 遍历新环境的 variables (按 index)            │
│   ↓                                         │
│ if (variable.secret)                        │
│   → SecretEnvironmentService.getSecret...   │
│     (envId, varIndex)                       │
│   → 恢复真实值                              │
│ else                                        │
│   → CurrentValueService.get...              │
│     (envId, varIndex)                       │
│   → 恢复当前值                              │
└─────────────────────────────────────────────┘
    ↓
编辑器插件 / UI 组件订阅更新
    ↓
显示遮蔽后的值 (******)
```

### 8.5 Setup 并发执行关系与竞态分析

**文件**: `packages/hoppscotch-common/src/services/persistence/index.ts:1169-1196`

#### 并发执行模型

```typescript
public async setupLater() {
  await Promise.all([
    this.setupLocalStatePersistence(),
    this.setupSettingsPersistence(),
    // ... 其他 setup
    
    this.setupEnvironmentsPersistence(),           // ① 加载环境元数据
    this.setupGlobalEnvsPersistence(),             // ② 加载全局环境
    this.setupSelectedEnvPersistence(),            // ③ 加载选中环境索引
    
    // ... 其他 setup
    
    this.setupSecretEnvironmentsPersistence(),     // ④ 加载 Secret 值
    this.setupCurrentEnvironmentValuePersistence(),// ⑤ 加载当前值
  ])
}
```

#### 并发执行的依赖关系

```
Promise.all (所有任务并发启动)
    │
    ├── ① setupEnvironmentsPersistence
    │   └── await Store.get(ENVIRONMENTS) → replaceEnvironments(data)
    │
    ├── ② setupGlobalEnvsPersistence
    │   └── await Store.get(GLOBAL_ENV) → setGlobalEnvVariables(data)
    │
    ├── ③ setupSelectedEnvPersistence ⚠️ 依赖 ①
    │   └── await Store.get(SELECTED_ENV) → setSelectedEnvironmentIndex(data)
    │       └── 内部检查: store.environments[selectedEnvironmentIndex.index] 是否存在
    │
    ├── ④ setupSecretEnvironmentsPersistence
    │   └── await Store.get(SECRET_ENVIRONMENTS) → loadSecretEnvironmentsFromPersistedState(data)
    │
    └── ⑤ setupCurrentEnvironmentValuePersistence
        └── await Store.get(CURRENT_ENVIRONMENT_VALUE) → loadEnvironmentsFromPersistedState(data)
```

#### 竞态条件分析

**问题场景**: ③ 在 ① 之前完成

```
时序:
  T0: ① 开始 await Store.get(ENVIRONMENTS)
  T1: ③ 开始 await Store.get(SELECTED_ENV)
  T2: ③ 完成，调用 setSelectedEnvironmentIndex({ type: "MY_ENV", index: 2 })
  T3: 此时 environmentsStore.environments 还是默认值（① 尚未完成）
  T4: setSelectedEnvironmentIndex 检查 store.environments[2]
  T5: 该索引不存在 → 回退为 { type: "NO_ENV_SELECTED" }
  T6: ① 完成，加载环境列表
  T7: 但选中环境索引已经被回退，丢失了用户的选择
```

**实际影响**:
- 由于 IndexedDB 读取通常很快，竞态发生概率较低
- 但理论上存在，尤其在首次加载或大数据量时
- 结果：用户上次选择的环境可能丢失，需要重新选择

### 8.6 selectedEnvironmentIndex 回退条件详解

**文件**: `packages/hoppscotch-common/src/newstore/environments.ts:54-76`

#### 回退逻辑

```typescript
setSelectedEnvironmentIndex(store, { selectedEnvironmentIndex }) {
  if (selectedEnvironmentIndex.type === "MY_ENV") {
    // 关键检查：目标索引是否在环境列表中存在
    if (store.environments[selectedEnvironmentIndex.index]) {
      return { selectedEnvironmentIndex }  // ✅ 正常设置
    }
    // ❌ 索引不存在，回退为 NO_ENV_SELECTED
    return {
      selectedEnvironmentIndex: { type: "NO_ENV_SELECTED" },
    }
  }
  return { selectedEnvironmentIndex }  // TEAM_ENV 或 NO_ENV_SELECTED 直接设置
}
```

#### 回退触发条件汇总

| 条件 | 是否触发回退 | 说明 |
|------|------------|------|
| `type === "MY_ENV"` 且 `environments[index]` 存在 | ❌ 不回退 | 正常设置 |
| `type === "MY_ENV"` 且 `environments[index]` 不存在 | ✅ 回退 | 索引越界 |
| `type === "MY_ENV"` 且 `environments` 为空数组 | ✅ 回退 | 无任何环境 |
| `type === "MY_ENV"` 且 `environments` 尚未加载 | ✅ 回退 | 竞态条件导致 |
| `type === "TEAM_ENV"` | ❌ 不回退 | 直接设置，不做检查 |
| `type === "NO_ENV_SELECTED"` | ❌ 不回退 | 直接设置 |

#### 环境删除时的索引调整

**文件**: `packages/hoppscotch-common/src/newstore/environments.ts:142-176`

```typescript
deleteEnvironment(store, { envIndex }) {
  let newCurrEnvIndex = selectedEnvironmentIndex

  // Scenario 1: 删除的是当前选中的环境 → 回退为 NO_ENV_SELECTED
  if (selectedEnvironmentIndex.type === "MY_ENV" &&
      envIndex === selectedEnvironmentIndex.index) {
    newCurrEnvIndex = { type: "NO_ENV_SELECTED" }
  }

  // Scenario 2: 删除的环境在当前选中环境之前 → 索引左移
  if (selectedEnvironmentIndex.type === "MY_ENV" &&
      envIndex < selectedEnvironmentIndex.index) {
    newCurrEnvIndex = {
      type: "MY_ENV",
      index: selectedEnvironmentIndex.index - 1,
    }
  }

  // Scenario 3: 删除的环境在当前选中环境之后 → 索引不变
  return {
    environments: environments.filter((_, index) => index !== envIndex),
    selectedEnvironmentIndex: newCurrEnvIndex,
  }
}
```

### 8.7 varIndex 重建规则

**文件**: `packages/hoppscotch-common/src/components/environments/my/Details.vue:524-582`

#### varIndex 的定义与用途

```typescript
// SecretEnvironmentService 中的数据结构
type SecretVariable = {
  key: string
  value: string
  varIndex: number        // 在环境变量数组中的索引位置
  initialValue?: string
}

// CurrentValueService 中的数据结构
type Variable = {
  key: string
  currentValue: string
  varIndex: number        // 在环境变量数组中的索引位置
  isSecret: boolean
}
```

**核心机制**: varIndex 不是固定 ID，而是**基于位置的动态索引**。

#### 保存时的 varIndex 重建流程

```typescript
// saveEnvironment() 中的关键代码
const filteredVariables = pipe(
  vars.value,
  A.filterMap(
    flow(
      O.fromPredicate((e) => e.env.key !== ""),
      O.map((e) => e.env)
    )
  )
)

// 为 secret 变量重新分配 varIndex
const secretVariables = pipe(
  filteredVariables,
  A.filterMapWithIndex((i, e) =>  // ← i 是新的位置索引
    e.secret
      ? O.some({
          key: e.key,
          value: e.currentValue,
          varIndex: i,              // ← 基于新位置重建
          initialValue: e.initialValue,
        })
      : O.none
  )
)

// 为非 secret 变量重新分配 varIndex
const nonSecretVariables = pipe(
  filteredVariables,
  A.filterMapWithIndex((i, e) =>  // ← i 是新的位置索引
    !e.secret
      ? O.some({
          key: e.key,
          currentValue: e.currentValue,
          varIndex: i,              // ← 基于新位置重建
          isSecret: e.secret ?? false,
        })
      : O.none
  )
)
```

#### varIndex 变化场景分析

**场景 1: 删除变量**

```
删除前: [var0, var1, var2, var3]  (varIndex: 0, 1, 2, 3)
删除 var1 后:
  vars.value = [var0, var2, var3]
  保存时重新计算:
    secretVariables: varIndex = 0 (var0)
    nonSecretVariables: varIndex = 1 (var2), varIndex = 2 (var3)
```

**场景 2: 重排变量**

```
重排前: [var0, var1, var2]  (varIndex: 0, 1, 2)
重排后: [var2, var0, var1]
  保存时重新计算:
    新的 varIndex: 0 (var2), 1 (var0), 2 (var1)
```

**场景 3: 混合 secret/非 secret**

```
vars.value = [nonSecret0, secret1, nonSecret2, secret3]
  secretVariables: [{ ..., varIndex: 1 }, { ..., varIndex: 3 }]
  nonSecretVariables: [{ ..., varIndex: 0 }, { ..., varIndex: 2 }]
```

**关键点**: secret 和非 secret 变量分开存储，但 varIndex 是基于**完整变量列表**的位置。

#### 存储时的覆盖语义

```typescript
// SecretEnvironmentService.addSecretEnvironment
public addSecretEnvironment(id: string, secretVars: SecretVariable[]) {
  this.secretEnvironments.set(id, secretVars)  // ← 完全覆盖，不是增量
}

// CurrentValueService.addEnvironment  
public addEnvironment(id: string, vars: Variable[]) {
  this.environments.set(id, vars)  // ← 完全覆盖，不是增量
}
```

**影响**: 每次保存都会完全替换该环境 ID 下的所有变量记录。

### 8.8 取消 Secret 后的残留影响评估

#### 残留产生机制

**文件**: `packages/hoppscotch-common/src/services/secret-environment.service.ts:174-190`

```typescript
protected watchSecretEnvironments() {
  watch(
    () => this.secretEnvironments,
    () => {
      nextTick(() => {
        this.secretEnvironments.forEach((secretVars, id) => {
          // 只清理 key 为空的变量
          const filteredVars = secretVars.filter((v) => v.key !== "")
          
          // 如果所有变量都被清理，删除整个环境条目
          if (filteredVars.length === 0) {
            this.secretEnvironments.delete(id)
          }
        })
      })
    },
    { deep: true }
  )
}
```

#### 场景分析

**场景 1: 将 secret 变量改为非 secret（切换开关）**

```
初始状态:
  vars.value = [{ key: "TOKEN", secret: true, currentValue: "abc123" }]
  SecretEnvironmentService["envId"] = [{ key: "TOKEN", value: "abc123", varIndex: 0 }]

用户操作: 将 TOKEN 的 secret 开关切换为 false
  vars.value = [{ key: "TOKEN", secret: false, currentValue: "abc123" }]

保存时:
  secretVariables = []  (空，因为没有 secret 变量)
  nonSecretVariables = [{ key: "TOKEN", currentValue: "abc123", varIndex: 0 }]

  // 调用
  secretEnvironmentService.addSecretEnvironment("envId", [])  // ← 设置为空数组
  currentEnvironmentValueService.addEnvironment("envId", [...])

watchSecretEnvironments 触发:
  filteredVars = [].filter(v => v.key !== "") = []
  → this.secretEnvironments.delete("envId")  // ✅ 清理成功
```

**场景 2: 删除所有 secret 变量**

```
初始状态:
  vars.value = [
    { key: "TOKEN", secret: true, ... },
    { key: "API_KEY", secret: true, ... }
  ]

用户操作: 删除所有变量
  vars.value = []

保存时:
  secretVariables = []
  nonSecretVariables = []

  secretEnvironmentService.addSecretEnvironment("envId", [])
  → watchSecretEnvironments 清理整个环境条目 ✅
```

**场景 3: 仅删除部分 secret 变量**

```
初始状态:
  vars.value = [secret0, secret1, secret2]
  SecretEnvironmentService["envId"] = [secret0, secret1, secret2]

用户操作: 删除 secret1
  vars.value = [secret0, secret2]

保存时:
  secretVariables = [{ ..., varIndex: 0 }, { ..., varIndex: 1 }]
  // varIndex 重新计算，旧的 secret1 记录被覆盖掉 ✅
```

#### 环境删除时的清理流程对比

**个人环境删除 (my/Environment.vue:224-234)** ✅ 完整清理

```typescript
const removeEnvironment = async () => {
  const isValidToken = await handleTokenValidation()
  if (!isValidToken) return
  if (props.environmentIndex === null) return
  if (!isGlobalEnvironment.value) {
    // 1. 删除元数据
    deleteEnvironment(props.environmentIndex as number, props.environment.id)
    // 2. 删除 Secret 记录
    secretEnvironmentService.deleteSecretEnvironment(props.environment.id)
    // 3. 删除当前值记录
    currentEnvironmentValueService.deleteEnvironment(props.environment.id)
  }
  toast.success(`${t("state.deleted")}`)
}
```

**团队环境删除 (teams/Environment.vue:208-222)** ✅ 完整清理

```typescript
const removeEnvironment = () => {
  pipe(
    deleteTeamEnvironment(props.environment.id),  // API 调用
    TE.match(
      (err: GQLError<string>) => { console.error(err) },
      () => {
        toast.success(`${t("team_environment.deleted")}`)
        // 成功后清理本地记录
        secretEnvironmentService.deleteSecretEnvironment(props.environment.id)
        currentEnvironmentValueService.deleteEnvironment(props.environment.id)
      }
    )
  )()
}
```

**删除选中环境 (index.vue:297-318)** ⚠️ 不完整清理

```typescript
const removeSelectedEnvironment = () => {
  const selectedEnvIndex = getSelectedEnvironmentIndex()
  if (selectedEnvIndex?.type === "NO_ENV_SELECTED") return

  if (selectedEnvIndex?.type === "MY_ENV") {
    // ❌ 只删除元数据，不清理 Secret 和 CurrentValue
    deleteEnvironment(selectedEnvIndex.index)
    toast.success(`${t("state.deleted")}`)
  }

  if (selectedEnvIndex?.type === "TEAM_ENV") {
    pipe(
      deleteTeamEnvironment(selectedEnvIndex.teamEnvID),
      TE.match(
        (err: GQLError<string>) => { console.error(err) },
        () => {
          toast.success(`${t("team_environment.deleted")}`)
          // ❌ 团队环境删除时也没有清理本地记录！
        }
      )
    )()
  }
}
```

#### 删除入口与清理行为矩阵

| 删除入口 | 个人环境清理 Secret | 个人环境清理 CurrentValue | 团队环境清理 Secret | 团队环境清理 CurrentValue |
|---------|-------------------|-------------------------|-------------------|-------------------------|
| my/Environment.vue 删除按钮 | ✅ 是 | ✅ 是 | - | - |
| teams/Environment.vue 删除按钮 | - | - | ✅ 是 | ✅ 是 |
| index.vue "删除选中环境" | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 |

#### 潜在残留场景（修正后）

**场景 4: 通过 index.vue 删除环境时 Secret 记录未清理**

```
用户操作: 选中环境后点击"删除选中环境"
  removeSelectedEnvironment()
    → deleteEnvironment(selectedEnvIndex.index)  // 仅删除元数据
    → ❌ 未调用 secretEnvironmentService.deleteSecretEnvironment
    → ❌ 未调用 currentEnvironmentValueService.deleteEnvironment

残留状态:
  SecretEnvironmentService["envId"] 仍然存在
  CurrentValueService["envId"] 仍然存在

影响评估:
  - 不会影响正常恢复：因为该环境 ID 不再在 environmentsStore 中
  - 占用存储空间：LocalStorage 中残留数据
  - 潜在风险：如果未来创建相同 ID 的环境，可能意外恢复旧值
```

**场景 5: 保存失败导致的不一致**

```
正常流程:
  1. secretEnvironmentService.addSecretEnvironment(id, secretVariables)
  2. currentEnvironmentValueService.addEnvironment(id, nonSecretVariables)
  3. environmentsStore.updateEnvironment(...)  // 更新元数据

如果步骤 3 失败:
  - SecretEnvironmentService 已更新
  - CurrentValueService 已更新
  - environmentsStore 未更新

结果:
  - Secret 值与元数据不一致
  - 下次加载时可能恢复到旧状态
```

**场景 6: 团队环境 API 删除成功但本地清理失败**

```
teams/Environment.vue 流程:
  1. deleteTeamEnvironment API 调用成功
  2. toast.success()
  3. secretEnvironmentService.deleteSecretEnvironment()
  4. currentEnvironmentValueService.deleteEnvironment()

如果步骤 3 或 4 因异常中断:
  - 服务端环境已删除
  - 本地 Secret 记录残留

发生概率: 低（都是同步内存操作）
```

#### 残留记录对恢复的影响矩阵（修正后）

| 残留场景 | 是否影响恢复 | 影响程度 | 说明 |
|---------|------------|---------|------|
| 取消 secret 后残留空数组 | ❌ 不影响 | 无 | watch 会清理 |
| 删除所有 secret 后残留空数组 | ❌ 不影响 | 无 | watch 会清理 |
| 删除部分 secret 后残留旧记录 | ❌ 不影响 | 无 | addSecretEnvironment 覆盖 |
| my/Environment.vue 删除环境 | ❌ 不影响 | 无 | 完整清理 |
| teams/Environment.vue 删除环境 | ❌ 不影响 | 无 | 完整清理 |
| index.vue 删除选中环境 | ⚠️ 轻微 | 低 | 残留但不影响功能 |
| 保存失败导致的不一致 | ⚠️ 轻微 | 中 | 下次加载可能恢复旧值 |
| 并发写入导致的冲突 | ⚠️ 轻微 | 中 | 后写入者生效 |

#### 边界说明

**残留记录何时会被清理？**
- 应用重启重新加载时：如果环境 ID 已不存在于 environmentsStore，残留记录仍会被加载但永远不会被使用
- 手动清理：无 UI 入口，只能通过清除浏览器数据或开发者工具删除
- 自动清理：无自动垃圾回收机制

**残留记录何时会造成实际问题？**
- 极端情况：用户删除环境 A（ID: "abc123"），之后由于某种原因（如同步恢复）创建了新环境且恰好复用相同 ID "abc123"
- 此时残留的 Secret 值会被"恢复"到新环境中，可能导致值错位

---

## 9. 导出策略深度分析

### 9.1 单环境导出的安全机制

**文件**: `packages/hoppscotch-common/src/helpers/import-export/export/environment.ts`

```typescript
export const transformEnvironmentVariables = ({ id, v, name, variables }: Environment) => {
  return {
    id, v, name,
    variables: variables.map((variable) => ({
      key: variable.key,
      secret: variable.secret,
      initialValue: variable.initialValue,
      currentValue: variable.secret ? "" : (variable.currentValue ?? "")  // Secret 清空
    })),
  }
}
```

**安全保证**：
1. Secret 变量的 `currentValue` 被强制清空为 `""`
2. `initialValue` 保留（但服务端同步的环境中，Secret 的 initialValue 本身就是空的）

### 9.2 批量导出的安全分析

**文件**: `packages/hoppscotch-common/src/helpers/import-export/export/environments.ts`

```typescript
export const environmentsExporter = (myEnvironments: Environment[]) => {
  return JSON.stringify(myEnvironments, null, 2)
}
```

**调用位置**: `packages/hoppscotch-common/src/components/environments/ImportExport.vue:75-83`

```typescript
const environmentJson = computed(() => {
  if (isTeamEnvironment.value && props.teamEnvironments) {
    return props.teamEnvironments.map(({ environment }) =>
      transformEnvironmentVariables(environment)  // ✅ 先转换
    )
  }
  return myEnvironments.value.map(transformEnvironmentVariables)  // ✅ 先转换
})
```

### 9.3 批量导出不泄露 Secret 的前提条件

**前提 1：调用链必须经过 `transformEnvironmentVariables`**

- ✅ ImportExport.vue 中使用 `computed` 先转换再导出
- ✅ Team 环境导出也经过转换

**前提 2：环境元数据本身不包含 Secret 真实值**

- `environmentsStore` 存储的环境元数据中，Secret 变量的 `currentValue` 和 `initialValue` 都是 `""`
- 真实值仅存在于 `SecretEnvironmentService`（本地内存，不参与导出）

**前提 3：服务端同步的环境不含 Secret 值**

- V1→V2 迁移时清空 Secret 值
- 服务端 API 返回的环境数据中 Secret 值为空

### 9.4 批量导出的失效范围（潜在泄露风险）

**风险场景 1：直接调用 `environmentsExporter` 而不经过 `transformEnvironmentVariables`**

```typescript
// ❌ 危险：如果 myEnvironments 包含本地修改过的 Secret 值
const json = environmentsExporter(myEnvironments)
```

**风险场景 2：脚本或插件直接访问 `environmentsStore` 数据**

```typescript
// ⚠️ 注意：environmentsStore 中的 Secret 变量 currentValue 通常为空
// 但如果有代码路径将真实值写入了 environmentsStore，就会泄露
const data = environmentsStore.value.environments
JSON.stringify(data)
```

**风险场景 3：运行时内存对象被序列化**

- `getCombinedEnvVariables()` 返回的对象包含 Secret 真实值
- 如果此对象被意外序列化并导出，将泄露 Secret

**风险场景 4：LocalStorage 直接读取**

- `secretEnvironments` key 存储在 LocalStorage
- 恶意脚本或浏览器扩展可直接读取

### 9.5 导出安全矩阵

| 导出方式 | Secret 值来源 | 是否经过转换 | 安全状态 |
|---------|-------------|------------|---------|
| UI 单环境导出 | environmentsStore | ✅ transformEnvironmentVariables | ✅ 安全 |
| UI 批量导出 | environmentsStore | ✅ transformEnvironmentVariables | ✅ 安全 |
| 直接 environmentsExporter | 取决于传入 | ❌ 无转换 | ⚠️ 取决于数据 |
| 直接 JSON.stringify(environmentsStore) | environmentsStore | ❌ 无转换 | ✅ 通常安全（值为空） |
| JSON.stringify(getCombinedEnvVariables()) | SecretEnvironmentService | ❌ 无转换 | ❌ 泄露风险 |

---

## 10. 关键设计决策总结

### 10.1 安全性设计
1. **服务端零知识**：Secret 值永不离开浏览器
2. **本地隔离存储**：与普通环境变量分开存储
3. **UI 遮蔽**：所有显示场景均对 secret 值做脱敏处理
4. **导出时清空**：单环境导出时清空 Secret 的 currentValue

### 10.2 运行时恢复机制
1. 通过 `varIndex` 索引匹配，确保值与变量正确对应
2. 请求执行前通过 `resolveEnvVars` 注入真实值
3. 环境切换时通过 `aggregateEnvsWithCurrentValue$` 自动恢复
4. 脚本沙箱可访问完整解析后的值

### 10.3 导出安全边界
- 单环境导出时清空 secret 的 currentValue
- 批量导出依赖 `transformEnvironmentVariables` 的前置转换
- 真实值仅存在于 `SecretEnvironmentService`，不参与标准导出流程
- 避免意外导出敏感数据

---

## 11. 相关文件索引

| 功能 | 文件路径 |
|------|---------|
| Secret 服务 | `packages/hoppscotch-common/src/services/secret-environment.service.ts` |
| 当前值服务 | `packages/hoppscotch-common/src/services/current-environment-value.service.ts` |
| 环境 Store | `packages/hoppscotch-common/src/newstore/environments.ts` |
| 值合并工具 | `packages/hoppscotch-common/src/helpers/utils/environments.ts` |
| 编辑器插件 | `packages/hoppscotch-common/src/helpers/editor/extensions/HoppEnvironment.ts` |
| 模板解析 | `packages/hoppscotch-data/src/environment/index.ts` |
| 环境导出 | `packages/hoppscotch-common/src/helpers/import-export/export/environment.ts` |
| 持久化服务 | `packages/hoppscotch-common/src/services/persistence/index.ts` |
| 沙箱方法 | `packages/hoppscotch-js-sandbox/src/utils/shared.ts` |
| 数据结构 V2 | `packages/hoppscotch-data/src/environment/v/2.ts` |
