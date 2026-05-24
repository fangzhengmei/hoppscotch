# 多端协同中集合版本同步及冲突提示机制分析

## 1. 整体架构概览

Hoppscotch 采用 **本地优先 + 实时订阅** 的混合架构实现多端协同：

```
┌──────────────────────────────────────────────────────────────────┐
│                         前端 (客户端)                            │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────────┐    │
│  │ 本地 Store  │────▶│  同步模块   │────▶│  GraphQL API    │    │
│  │ (状态管理)  │     │ (sync.ts)   │     │   (变更写入)    │    │
│  └─────────────┘     └─────────────┘     └─────────────────┘    │
│         ▲                  ▲                                      │
│         │                  │                                      │
│         │                  │                                      │
│  ┌─────────────┐     ┌─────────────┐                              │
│  │  订阅处理器  │────▶│  对账模块   │                              │
│  │ (实时接收)  │     │ (ID映射)    │                              │
│  └─────────────┘     └─────────────┘                              │
│         ▲                                                         │
└─────────┼─────────────────────────────────────────────────────────┘
          │ GraphQL Subscriptions (Pub/Sub)
          ▼
┌──────────────────────────────────────────────────────────────────┐
│                         后端 (服务端)                            │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────────┐    │
│  │  Prisma ORM │────▶│  事务引擎   │────▶│  PostgreSQL DB  │    │
│  │  (数据层)   │     │ (行级锁)    │     │  (持久化存储)   │    │
│  └─────────────┘     └─────────────┘     └─────────────────┘    │
│         ▲                  ▲                                      │
│         │                  │                                      │
│  ┌─────────────┐     ┌─────────────┐                              │
│  │  Pub/Sub    │────▶│  冲突处理   │                              │
│  │ (事件广播)  │     │ (重试机制)  │                              │
│  └─────────────┘     └─────────────┘                              │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. 版本标识系统

### 2.1 双重ID机制

系统采用 **前端引用ID + 后端数据库ID** 的双重标识策略：

| 标识 | 生成位置 | 用途 | 代码位置 |
|------|---------|------|---------|
| `_ref_id` | 前端 | 本地临时唯一引用，用于前端状态追踪 | `@hoppscotch/data` 中 `generateUniqueRefId("coll")` |
| `id` | 后端 | 数据库主键，全局唯一标识 | PostgreSQL `cuid()` 生成 |
| `orderIndex` | 后端 | 排序顺序索引，保证同层级唯一 | 数据库事务中维护 |
| `updatedOn` | 数据库 | 最后更新时间戳，自动维护 | `@updatedAt @db.Timestamptz(3)` |
| `createdOn` | 数据库 | 创建时间戳 | `@default(now()) @db.Timestamptz(3)` |

### 2.2 数据模型设计

**TeamCollection 模型** (`schema.prisma:42-57`):
```prisma
model TeamCollection {
  id         String           @id @default(cuid())
  parentID   String?
  teamID     String
  title      String
  orderIndex Int
  createdOn  DateTime         @default(now()) @db.Timestamptz(3)
  updatedOn  DateTime         @updatedAt @db.Timestamptz(3)
  data       Json?

  @@unique([teamID, parentID, orderIndex])  // 唯一约束，防止顺序冲突
}
```

**关键设计点：**
- `@@unique([teamID, parentID, orderIndex])` 确保同一父级下的子集合不会有重复的排序位置
- `updatedOn` 由数据库自动维护，可用于隐式版本比较

---

## 3. 本地修改与远端版本对账机制

### 3.1 变更同步流程（本地 → 远端）

**核心流程代码** (`sync/index.ts:54-72`):
```typescript
function startStoreSync() {
  store.dispatches$.subscribe((actionParams) => {
    if ((storeSyncDefinition as any)[actionParams.dispatcher]) {
      const dispatcher = actionParams.dispatcher
      const payload = actionParams.payload
      const operationMapperFunction = (storeSyncDefinition as any)[dispatcher]

      if (
        operationMapperFunction &&
        _isRunningDispatchWithoutSyncing &&  // 避免订阅触发的变更反向同步
        shouldSyncValue()
      ) {
        operationMapperFunction(payload)  // 调用后端API
      }
    }
  })
}
```

**同步流程详解：**

1. **本地优先更新**：用户操作首先更新本地 Store，保证UI响应性
   - 例如：`addRESTCollection()` 先更新前端状态，再触发同步

2. **同步拦截**：`storeSyncDefinition` 定义了每个 dispatcher 对应的同步操作
   - 代码位置：`collections/web/sync.ts:233-559`

3. **API调用与ID回填**：后端创建成功后，将返回的 `id` 赋值给本地对象
   - 代码位置：`sync.ts:95-124` 的 `recursivelySyncCollections` 函数

4. **去重处理**：通过 `removeDuplicateRESTCollectionOrFolder` 防止本地和订阅重复创建
   - 代码位置：`collections.ts:1935-1987`

### 3.2 订阅同步流程（远端 → 本地）

**核心代码** (`collections/web/index.ts:299-338`):
```typescript
function setupSubscriptions() {
  const subs = [
    setupUserCollectionCreatedSubscription(),
    setupUserCollectionUpdatedSubscription(),
    setupUserCollectionRemovedSubscription(),
    setupUserCollectionMovedSubscription(),
    setupUserCollectionOrderUpdatedSubscription(),
    // ... 更多订阅
  ]
  return () => subs.forEach(sub => sub.unsubscribe())
}
```

**关键对账逻辑 - 集合创建订阅** (`index.ts:340-452`):
```typescript
function setupUserCollectionCreatedSubscription() {
  userCollectionCreated$.subscribe((res) => {
    // Step 1: 通过后端ID查找本地是否已存在
    const userCollectionLocalID = getCollectionPathFromCollectionID(
      userCollectionBackendID,
      collectionStore.value.state
    )

    // Step 2: 已存在则跳过（说明是本端创建的）
    if (userCollectionLocalID) return

    // Step 3: 不存在则通过 runDispatchWithOutSyncing 添加到本地
    runDispatchWithOutSyncing(() => {
      addRESTCollection({ ... })
      // 回填后端ID
      addedCollection.id = userCollectionBackendID
    })
  })
}
```

### 3.3 对账核心函数

**`getCollectionPathFromCollectionID`** (`index.ts:1062-1084`):
- 功能：通过后端 `id` 反向查找本地集合的路径（如 "0/1/2"）
- 用途：订阅事件到达时，判断该变更是否已在本地存在
- 递归遍历整个集合树进行匹配

**`runDispatchWithOutSyncing`** (`sync/index.ts:23-30`):
- 功能：执行 dispatch 但不触发反向同步
- 实现：通过全局标志 `_isRunningDispatchWithoutSyncing` 控制
- 用途：防止订阅接收的变更又被同步回后端，形成循环

---

## 4. 冲突检测机制

### 4.1 数据库层面的冲突防护

#### 4.1.1 行级锁（悲观并发控制）

**锁机制实现** (`prisma.service.ts:150-185`):
```typescript
async lockTeamCollectionByTeamAndParent(
  tx: Prisma.TransactionClient,
  teamId: string,
  parentID: string | null,
) {
  const lockQuery = parentID
    ? Prisma.sql`SELECT "orderIndex" FROM "TeamCollection" WHERE "teamID" = ${teamId} AND "parentID" = ${parentID} FOR UPDATE`
    : Prisma.sql`SELECT "orderIndex" FROM "TeamCollection" WHERE "teamID" = ${teamId} AND "parentID" IS NULL FOR UPDATE`;
  return tx.$executeRaw(lockQuery);
}
```

**锁应用场景**：
- 创建集合前锁定同级所有记录，保证 `orderIndex` 连续唯一
- 移动/重排序时锁定源和目标父级的所有子集合
- 按 `parentID` 排序获取锁，防止死锁

#### 4.1.2 死锁预防 - 按顺序获取锁

**代码示例** (`team-collection.service.ts:811-845`):
```typescript
// Acquire locks in deterministic order (sorted by parentID) to prevent deadlocks
const srcParentID = collection.right.parentID ?? '';
const destParentID = destCollection.right.parentID ?? '';

if (srcParentID === destParentID) {
  await this.prisma.lockTeamCollectionByTeamAndParent(tx, teamID, collection.right.parentID);
} else if (srcParentID < destParentID) {
  await this.prisma.lockTeamCollectionByTeamAndParent(tx, teamID, collection.right.parentID);
  await this.prisma.lockTeamCollectionByTeamAndParent(tx, teamID, destCollection.right.parentID);
} else {
  await this.prisma.lockTeamCollectionByTeamAndParent(tx, teamID, destCollection.right.parentID);
  await this.prisma.lockTeamCollectionByTeamAndParent(tx, teamID, collection.right.parentID);
}
```

### 4.2 错误码识别与冲突处理

**Prisma 错误码定义** (`prisma-error-codes.ts:1-8`):
```typescript
export enum PrismaError {
  DATABASE_UNREACHABLE = 'P1001',
  TABLE_DOES_NOT_EXIST = 'P2021',
  UNIQUE_CONSTRAINT_VIOLATION = 'P2002',  // 唯一约束冲突
  RECORD_NOT_FOUND = 'P2025',              // 记录已被删除
  TRANSACTION_TIMEOUT = 'P2028',           // 事务超时
  TRANSACTION_DEADLOCK = 'P2034',          // 死锁或写冲突
}
```

### 4.3 冲突重试机制

**重试逻辑实现** (`user-collection.service.ts:511-564`):
```typescript
private async removeCollectionAndUpdateSiblingsOrderIndex(...) {
  let retryCount = 0;
  while (retryCount < this.MAX_RETRIES) {  // MAX_RETRIES = 5
    try {
      await this.prisma.$transaction(async (tx) => {
        // 1. 锁行
        await this.prisma.lockUserCollectionByParent(tx, userID, collection.parentID);
        // 2. 处理删除（容忍已被删除的情况）
        try {
          await tx.userCollection.delete({ where: { id: collection.id } });
        } catch (deleteError) {
          // P2025: 记录已被其他并发事务删除，静默处理
          if (deleteError?.code === PrismaError.RECORD_NOT_FOUND) return;
          throw deleteError;
        }
        // 3. 更新兄弟节点 orderIndex
        await tx.userCollection.updateMany({ ... });
      });
      break;
    } catch (error) {
      retryCount++;
      if (retryCount >= this.MAX_RETRIES ||
          (error.code !== PrismaError.UNIQUE_CONSTRAINT_VIOLATION &&
           error.code !== PrismaError.TRANSACTION_DEADLOCK &&
           error.code !== PrismaError.TRANSACTION_TIMEOUT))
        return E.left(USER_COLL_REORDERING_FAILED);

      await delay(retryCount * 100);  // 指数退避（线性增长）
    }
  }
}
```

**重试策略：**
- 最多重试 **5次**
- 仅对以下冲突类型重试：
  - `P2002` 唯一约束冲突（并发创建导致 orderIndex 重复）
  - `P2034` 死锁或写冲突
  - `P2028` 事务超时
- 重试间隔：`retryCount * 100ms`（100ms, 200ms, 300ms, 400ms, 500ms）

### 4.4 乐观删除处理

**代码示例** (`user-collection.service.ts:523-531`):
```typescript
try {
  await tx.userCollection.delete({ where: { id: collection.id } });
} catch (deleteError) {
  // P2025: Record not found — already deleted by a concurrent transaction
  if (deleteError?.code === PrismaError.RECORD_NOT_FOUND) return;
  throw deleteError;
}
```

**设计意图：**
- 当删除操作发现记录已不存在时（P2025），不报错而是静默成功
- 这是一种"最终一致性"策略：目标都是让记录消失

---

## 5. 冲突提示策略

### 5.1 当前策略分析

当前系统的冲突处理以 **"静默自动解决"** 为主，**无显式用户冲突提示**：

| 冲突场景 | 处理策略 | 用户感知 |
|---------|---------|---------|
| 并发创建导致 orderIndex 重复 | 自动重试最多5次 | 无感知（延迟增加） |
| 并发删除同一集合 | 静默处理（P2025 错误忽略） | 无感知 |
| 并发移动同一集合到不同位置 | 后执行的成功，先执行的被覆盖 | 无感知（看到最后结果） |
| 并发编辑同一集合属性 | 后写入的覆盖先写入的 | 无感知（看到最后修改） |
| 死锁/事务超时 | 自动重试 | 无感知（延迟增加） |
| 订阅事件与本地操作重复 | 通过ID匹配跳过重复应用 | 无感知 |

### 5.2 "最后写入者胜"策略

系统采用 **Last Write Wins (LWW)** 策略：
- 数据库层面：`updatedOn` 自动更新，但未用于显式比较
- 实际效果：后到达的写操作覆盖先到达的
- 订阅广播：所有客户端最终看到的是最后一次成功写入的状态

### 5.3 幂等性保证

**订阅端去重** (`index.ts:353-361`):
```typescript
const userCollectionLocalID = getCollectionPathFromCollectionID(
  userCollectionBackendID,
  collectionStore.value.state
);

// collection already exists in store ( this instance created it )
if (userCollectionLocalID) {
  return;  // 跳过重复应用
}
```

**同步端去重** (`sync.ts:121-124`):
```typescript
removeDuplicateRESTCollectionOrFolder(
  parentCollectionID,
  `${collectionPath}`
);
```

---

## 6. 全量对账与初始化

### 6.1 登录/初始化全量同步

**代码** (`index.ts:262-297`):
```typescript
async function loadUserCollections(collectionType: "REST" | "GQL") {
  const res = await exportUserCollectionsToJSON(
    undefined,
    collectionType == "REST" ? ReqType.Rest : ReqType.Gql
  );
  if (E.isRight(res)) {
    // 全量替换本地状态
    runDispatchWithOutSyncing(() => {
      setRESTCollections(exportedCollections.map(...));
    });
  }
}
```

**触发时机：**
1. 应用启动时 (`initCollectionsSync` 中调用)
2. 用户登录后 (`currentUser$` 订阅触发)
3. Token 刷新后

### 6.2 实时增量同步

**订阅事件类型：**

| 事件类型 | 触发时机 | 处理函数 |
|---------|---------|---------|
| `coll_added` | 集合创建 | `setupUserCollectionCreatedSubscription` |
| `coll_updated` | 集合属性更新 | `setupUserCollectionUpdatedSubscription` |
| `coll_removed` | 集合删除 | `setupUserCollectionRemovedSubscription` |
| `coll_moved` | 集合移动（父级变更） | `setupUserCollectionMovedSubscription` |
| `coll_order_updated` | 集合重排序 | `setupUserCollectionOrderUpdatedSubscription` |
| `coll_duplicated` | 集合复制 | `setupUserCollectionDuplicatedSubscription` |
| （请求类同） |  |  |

---

## 7. 关键代码位置索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| 同步核心框架 | `packages/hoppscotch-selfhost-web/src/lib/sync/index.ts` | 32-103 |
| REST 集合同步定义 | `packages/hoppscotch-selfhost-web/src/platform/collections/web/sync.ts` | 233-559 |
| 订阅事件处理 | `packages/hoppscotch-selfhost-web/src/platform/collections/web/index.ts` | 299-1054 |
| 本地 Store 定义 | `packages/hoppscotch-common/src/newstore/collections.ts` | 284-1988 |
| 行级锁实现 | `packages/hoppscotch-backend/src/prisma/prisma.service.ts` | 150-197 |
| 用户集合服务（冲突重试） | `packages/hoppscotch-backend/src/user-collection/user-collection.service.ts` | 505-567 |
| 团队集合服务（死锁预防） | `packages/hoppscotch-backend/src/team-collection/team-collection.service.ts` | 811-845 |
| 数据模型 | `packages/hoppscotch-backend/prisma/schema.prisma` | 42-73 |
| 错误码定义 | `packages/hoppscotch-backend/src/prisma/prisma-error-codes.ts` | 1-8 |

---

## 8. 架构评价与改进建议

### 8.1 现有优点

1. **本地优先**：保证了UI响应速度，用户操作无等待
2. **细粒度锁**：按 `parentID` 加锁，锁范围小，并发性能好
3. **死锁预防**：按固定顺序获取锁，从根源避免死锁
4. **自动重试**：对可恢复冲突自动处理，用户无感知
5. **幂等设计**：通过ID匹配避免重复应用，保证最终一致性

### 8.2 潜在问题

1. **无显式版本号**：依赖 `updatedOn` 但未做显式版本比较，可能丢失更新
2. **无冲突提示**：所有冲突静默处理，用户可能不知道自己的修改被覆盖
3. **重试策略简单**：线性退避而非指数退避，高并发下可能仍失败
4. **无变更历史**：无法追溯历史版本，冲突发生后无法回滚
5. **全量替代风险**：初始化时全量替换可能丢失未同步的本地变更

### 8.3 改进建议

**1. 引入显式版本号：**
```prisma
model TeamCollection {
  // ... 现有字段
  version    Int    @default(1)  // 新增：乐观锁版本号
}
```

**2. 增加冲突检测的用户提示：**
- 对重要属性的并发修改，弹窗提示用户选择保留哪个版本
- 或采用合并策略（如请求URL保留A，headers合并A+B）

**3. 改进重试策略：**
- 使用指数退避：`delay(2 ** retryCount * 50)` 替代线性退避
- 增加随机抖动，避免惊群效应

**4. 本地未同步变更保护：**
- 全量同步前检查本地是否有未同步的"脏"数据
- 如有，先执行增量同步，再应用全量更新

---

## 9. 总结

Hoppscotch 的集合版本同步机制采用 **"本地优先 + 悲观锁 + 自动重试 + 最后写入者胜"** 的组合策略，在保证并发性能的同时实现了基本的多端协同。当前设计以**无感知自动解决**为核心原则，牺牲了一定的可追溯性和用户选择权，换取了良好的用户体验。对于API测试工具这类协作强度中等的场景，这是一个合理的权衡。
