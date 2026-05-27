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

## 8. 关键设计决策总结

### 8.1 安全性设计
1. **服务端零知识**：Secret 值永不离开浏览器
2. **本地隔离存储**：与普通环境变量分开存储
3. **UI 遮蔽**：所有显示场景均对 secret 值做脱敏处理

### 8.2 运行时恢复机制
1. 通过 `varIndex` 索引匹配，确保值与变量正确对应
2. 请求执行前通过 `unWrapEnvironments` 注入真实值
3. 脚本沙箱可访问完整解析后的值

### 8.3 导出安全边界
- 单环境导出时清空 secret 的 currentValue
- 避免意外导出敏感数据

---

## 9. 相关文件索引

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
