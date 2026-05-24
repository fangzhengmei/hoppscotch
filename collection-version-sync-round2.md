# 集合版本同步与冲突提示 - 错误传播链路深度分析（Round 2）

## 一、整体错误传播链路概览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            后端错误传播链路                               │
├─────────────┬─────────────┬─────────────┬───────────────────────────────┤
│ Service 层  │ Resolver 层 │ GraphQL 层  │  网络传输                     │
│ (重试逻辑)  │ (错误转换)  │ (格式封装)  │                               │
├─────────────┼─────────────┼─────────────┼───────────────────────────────┤
│ Prisma 错误 │ throwErr()  │ NestJS      │  HTTP 200 + GraphQL errors    │
│ → Either左值│ → Error     │ Apollo      │                               │
└─────────────┴─────────────┴─────────────┴───────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            前端错误传播链路                               │
├─────────────┬─────────────┬─────────────┬───────────────────────────────┤
│ GQLClient   │ API 调用层  │ 同步模块    │  UI 交互层                    │
│ (错误解析)  │ (透传)      │ (storeSync) │ (用户操作)                    │
├─────────────┼─────────────┼─────────────┼───────────────────────────────┤
│ parseGQLEr- │ runMutation │ **静默失败**│ TE.match + toast.error        │
│ rorString   │ → Either    │ 无错误处理  │ getErrorMessage 映射          │
└─────────────┴─────────────┴─────────────┴───────────────────────────────┘
```

---

## 二、后端：冲突重试失败后的错误返回

### 2.1 Service 层重试机制与错误码

**文件**：`team-collection.service.ts:60`

```typescript
MAX_RETRIES = 5; // 最大重试次数

// 以 updateOrderIndex 为例（team-collection.service.ts:560-613）
private async deleteCollectionAndUpdateSiblingsOrderIndex(...) {
  let retryCount = 0;
  while (retryCount < this.MAX_RETRIES) {
    try {
      await this.prisma.$transaction(async (tx) => {
        // 1. 行级锁：SELECT ... FOR UPDATE
        await this.prisma.lockTeamCollectionByTeamAndParent(...);
        // 2. 执行数据库操作
        // ...
      });
      break; // 成功，跳出循环
    } catch (error) {
      console.error('Error from TeamCollectionService.updateOrderIndex', error);
      retryCount++;

      // 重试耗尽或非预期错误 → 返回业务错误码
      if (
        retryCount >= this.MAX_RETRIES ||
        (error.code !== PrismaError.UNIQUE_CONSTRAINT_VIOLATION &&   // P2002
         error.code !== PrismaError.TRANSACTION_DEADLOCK &&          // P2034
         error.code !== PrismaError.TRANSACTION_TIMEOUT)             // P2028
      ) {
        return E.left(TEAM_COL_REORDERING_FAILED);  // 关键点：返回 Either 左值
      }

      await delay(retryCount * 100); // 线性退避
      console.debug(`Retrying updateOrderIndex... (${retryCount})`);
    }
  }
  return E.right(true);
}
```

**关键错误码**（`errors.ts`）：
- `TEAM_COL_REORDERING_FAILED = 'team_coll/reordering_failed'` - 团队集合重排序失败
- `USER_COLL_REORDERING_FAILED = 'user_coll/reordering_failed'` - 用户集合重排序失败
- `TEAM_REQ_REORDERING_FAILED = 'team_req/reordering_failed'` - 团队请求重排序失败
- `USER_REQUEST_REORDERING_FAILED = 'user_request/reordering_failed'` - 用户请求重排序失败

### 2.2 Resolver 层错误转换

**文件**：`team-collection.resolver.ts`

Resolver 层使用统一模式处理 Service 返回的 Either：

```typescript
// 以 updateTeamCollectionOrder 为例
@Mutation(...)
@UseGuards(...)
async updateTeamCollectionOrder(
  @Args() args: UpdateTeamCollectionOrderArgs,
  @GqlUser() user: AuthUser,
) {
  const res = await this.teamCollectionService.updateTeamCollectionOrder(
    args.teamID,
    args.collectionID,
    args.nextCollectionID,
    user.uid,
  );

  // 关键点：Either 左值 → throwErr() 抛出 Error
  if (E.isLeft(res)) {
    throwErr(res.left);  // res.left 是错误码字符串，如 'team_coll/reordering_failed'
  }
  return res.right;
}
```

**`throwErr` 实现**（`utils.ts:44-46`）：
```typescript
export function throwErr(errMessage: string): never {
  throw new Error(errMessage);  // 将错误码字符串包装成 Error 对象
}
```

### 2.3 GraphQL 层错误封装

NestJS + Apollo Server 自动将未捕获的 Error 转换为 GraphQL 错误响应：

```json
// HTTP 200 OK 响应体
{
  "data": null,
  "errors": [
    {
      "message": "[GraphQL] team_coll/reordering_failed",  // 自动添加前缀
      "locations": [...],
      "path": ["updateTeamCollectionOrder"],
      "extensions": {
        "code": "INTERNAL_SERVER_ERROR"
      }
    }
  ]
}
```

---

## 三、前端：错误接收与处理

### 3.1 GQLClient 层错误解析

**文件**：`GQLClient.ts:361-419`

`runMutation` 函数统一处理 GraphQL 响应：

```typescript
export const runMutation = <DocType, DocVariables, DocErrors extends string>(
  mutation: TypedDocumentNode<DocType, DocVariables>,
  variables: DocVariables,
): TE.TaskEither<GQLError<DocErrors>, DocType> =>
  pipe(
    TE.tryCatch(
      () => client.value!.mutation(mutation, variables).toPromise(),
      () => constVoid() as never
    ),
    TE.chainEitherK((result) =>
      pipe(
        result.data,
        E.fromNullable(
          pipe(
            result.error?.networkError,
            E.fromNullable(result.error?.message),
            E.match(
              // GraphQL 错误（无 networkError）
              (gqlErr) => {
                // 发送错误事件到错误流
                if (result.error) {
                  gqlClientError$.next({
                    type: "GQL_CLIENT_REPORTED_ERROR",
                    opType: "mutation",
                    opResult: result,
                  });
                }

                return <GQLError<DocErrors>>{
                  type: "gql_error",
                  error: parseGQLErrorString(gqlErr ?? ""),  // 关键点：解析错误码
                };
              },
              // 网络错误
              (networkErr) => <GQLError<DocErrors>>{
                type: "network_error",
                error: networkErr,
              }
            )
          )
        )
      )
    )
  );
```

**错误码解析**（`GQLClient.ts:361-362`）：
```typescript
export const parseGQLErrorString = (s: string) =>
  s.startsWith("[GraphQL] ") ? s.split("[GraphQL] ")[1] : s;
```

解析结果：
- 输入：`"[GraphQL] team_coll/reordering_failed"`
- 输出：`"team_coll/reordering_failed"`

### 3.2 API 调用层透传

**文件**：`platform/collections/web/api.ts`

API 层直接调用 `runMutation`，无额外错误处理：

```typescript
export const updateUserCollectionOrder = (
  collectionID: string,
  nextCollectionID?: string
) =>
  runMutation<
    UpdateUserCollectionOrderMutation,
    UpdateUserCollectionOrderMutationVariables,
    ""  // 注意：这里错误类型定义为空字符串，丢失具体错误码类型信息
  >(UpdateUserCollectionOrderDocument, {
    collectionID,
    nextCollectionID,
  })();
```

> ⚠️ **设计缺陷**：`DocErrors` 泛型参数被设为 `""`，导致 TypeScript 无法追踪具体错误码类型。

---

## 四、关键分歧点：用户主动操作 vs 后台自动同步

### 4.1 用户主动操作（UI 层）- 有错误提示

**文件**：`components/collections/index.vue:934-947`

用户点击按钮触发的操作会显式处理错误：

```typescript
// 创建集合 - 用户主动点击"新建"按钮
const handleCreateCollection = (name: string) => {
  modalLoadingState.value = true;

  pipe(
    createNewRootCollection(name, selectedTeam.teamID),  // 调用 API
    TE.match(
      // 错误分支：显示 Toast 提示
      (err: GQLError<string>) => {
        toast.error(`${getErrorMessage(err)}`);  // 错误 → Toast
        modalLoadingState.value = false;
      },
      // 成功分支：显示成功 Toast
      () => {
        modalLoadingState.value = false;
        toast.success(t("collection.created"));  // 成功 → Toast
        displayModalAdd(false);
      }
    )
  )();
};
```

**错误消息映射**（`helpers/runner/collection-tree.ts:4-41`）：

```typescript
export const getErrorMessage = (err: GQLError<string>, t: ComposerTranslation) => {
  console.error(err);
  if (err.type === "network_error") {
    return t("error.network_error");
  }
  switch (err.error) {
    case "team_coll/short_title":
      return t("collection.name_length_insufficient");
    case "team/invalid_coll_id":
    case "bug/team_coll/no_coll_id":
      return t("team.invalid_coll_id");
    case "team/not_required_role":
    case "team_req/not_required_role":
      return t("profile.no_permission");
    case "team_req/not_found":
      return t("team.no_request_found");
    case "team/collection_is_parent_coll":
      return t("team.parent_coll_move");
    case "team/target_and_destination_collection_are_same":
      return t("team.same_target_destination");
    case "team/target_collection_is_already_root_collection":
      return t("collection.invalid_root_move");
    // ... 其他业务错误码
    default:
      return t("error.something_went_wrong");  // 冲突错误码会落到这里
  }
};
```

> ⚠️ **重要发现**：`getErrorMessage` 中**没有**处理冲突相关的错误码：
> - `team_coll/reordering_failed`
> - `user_coll/reordering_failed`
> - `team_req/reordering_failed`
> - `user_request/reordering_failed`
>
> 这些冲突错误会落到 `default` 分支，显示通用的"Something went wrong"。

### 4.2 后台自动同步（storeSyncDefinition）- **静默失败**

**文件**：`platform/collections/web/sync.ts`

**这是本次分析的核心发现**：`storeSyncDefinition` 中的同步操作**完全没有错误处理**，错误被静默吞掉。

```typescript
export const storeSyncDefinition: StoreSyncDefinitionOf<typeof restCollectionStore> = {
  // ✅ 有降级处理，但降级后仍无错误提示
  async appendCollections({ entries }) {
    const result = await importUserCollectionsFromJSON(...);
    if (E.isLeft(result)) {
      // 批量导入失败 → 降级为逐个调用
      // 但逐个调用 recursivelySyncCollections 仍然没有错误处理
      entries.forEach((collection) => {
        recursivelySyncCollections(collection, `${indexStart}`);
        indexStart++;
      });
    }
  },

  // ❌ 完全无错误处理 - 编辑集合
  editCollection({ partialCollection: collection, collectionIndex }) {
    const collectionID = navigateToFolderWithIndexPath(...)?.id;
    const data = { ... };
    if (collectionID) {
      // fire-and-forget：调用但不 await，也不处理返回值
      updateUserCollection(collectionID, collection.name, JSON.stringify(data));
    }
  },

  // ❌ 完全无错误处理 - 删除集合
  async removeCollection({ collectionID }) {
    if (collectionID) {
      await deleteUserCollection(collectionID);  // await 但不检查返回值
    }
  },

  // ❌ 完全无错误处理 - 移动文件夹
  async moveFolder({ destinationPath, path }) {
    const sourceCollectionID = navigateToFolderWithIndexPath(...)?.id;
    const destinationCollectionID = ...;
    if (sourceCollectionID) {
      await moveUserCollection(sourceCollectionID, destinationCollectionID);  // 无错误处理
    }
  },

  // ❌ 完全无错误处理 - 编辑请求
  editRequest({ path, requestIndex, requestNew }) {
    const requestBackendID = navigateToFolderWithIndexPath(...)?.requests[requestIndex]?.id;
    if (requestBackendID) {
      // fire-and-forget
      editUserRequest(requestBackendID, requestNew.name, JSON.stringify(requestNew));
    }
  },

  // ❌ 完全无错误处理 - 更新集合排序
  async updateCollectionOrder({ collectionIndex, destinationCollectionIndex }) {
    const sourceCollectionID = navigateToFolderWithIndexPath(...)?.id;
    const nextCollectionID = ...;
    if (sourceCollectionID) {
      await updateUserCollectionOrder(sourceCollectionID, nextCollectionID);  // 无错误处理
    }
  },

  // ... 其他操作模式相同
};
```

**`recursivelySyncCollections` 也没有错误处理**（`sync.ts:70-210`）：

```typescript
const recursivelySyncCollections = async (
  collection: HoppCollection,
  collectionPath: string,
  parentUserCollectionID?: string
) => {
  // ...
  const res = await createRESTRootUserCollection(collection.name, data);
  if (E.isRight(res)) {
    // 成功：回填 ID
    collection.id = parentCollectionID;
    // ...
  }
  // ❌ else 分支：失败时什么也不做，无日志，无提示
  // 错误被静默忽略！

  // 子文件夹也一样
  collection.folders.forEach(async (folder, index) => {
    recursivelySyncCollections(folder, `${collectionPath}/${index}`, parentCollectionID);
  });
};
```

---

## 五、同步模块触发机制

**文件**：`lib/sync/index.ts:54-72`

同步模块通过订阅 Store 的 dispatch 流来触发后端同步：

```typescript
function startStoreSync() {
  store.dispatches$.subscribe((actionParams) => {
    // 检查是否有对应的同步处理器
    if ((storeSyncDefinition as any)[actionParams.dispatcher]) {
      const dispatcher = actionParams.dispatcher;
      const payload = actionParams.payload;
      const operationMapperFunction = (storeSyncDefinition as any)[dispatcher];

      if (
        operationMapperFunction &&
        _isRunningDispatchWithoutSyncing &&  // 防止循环
        shouldSyncValue()                     // 同步开关已开启
      ) {
        // 🔥 调用同步处理器，但完全不等待，也不处理错误
        operationMapperFunction(payload);
      }
    }
  });
}
```

**关键点**：
1. `operationMapperFunction(payload)` 是"发射后不管"（fire-and-forget）
2. 没有 `await`，没有返回值检查
3. 即使 Promise reject，也不会有任何捕获
4. 错误只会出现在浏览器控制台（如果有 `console.error`）

---

## 六、冲突错误完整传播路径示例

以"并发移动集合导致重排序冲突"为例：

```
1. 用户A在前端拖拽集合A到新位置
   ↓ Store.dispatch('updateCollectionOrder')
   ↓ startStoreSync 捕获 dispatch
   ↓ operationMapperFunction(payload) → updateCollectionOrder()
   ↓ 调用 API：updateUserCollectionOrder(collectionID, nextID)

2. 用户B同时也在移动集合，后端发生冲突
   ↓ 后端 Service 层捕获 Prisma P2002 错误
   ↓ 重试 5 次，每次延迟 100ms/200ms/300ms/400ms/500ms
   ↓ 5 次重试全部失败
   ↓ return E.left('user_coll/reordering_failed')

3. Resolver 层接收到左值
   ↓ throwErr('user_coll/reordering_failed')
   ↓ throw new Error('user_coll/reordering_failed')

4. NestJS 封装为 GraphQL 错误响应
   ↓ HTTP 200 + { errors: [{ message: "[GraphQL] user_coll/reordering_failed" }] }

5. 前端 GQLClient 解析
   ↓ parseGQLErrorString() → 'user_coll/reordering_failed'
   ↓ 返回 E.left({ type: 'gql_error', error: 'user_coll/reordering_failed' })

6. storeSyncDefinition 中的 updateCollectionOrder
   ↓ await 但不检查 E.isLeft/E.isRight
   ↓ ❌ 错误被静默忽略！
   ↓ 用户无任何感知
   ↓ 本地状态已更新（乐观更新），但后端状态不一致
```

---

## 七、当前提示策略的两层分化

| 操作类型 | 触发方式 | 错误处理 | 用户提示 | 数据一致性风险 |
|---------|---------|---------|---------|--------------|
| 创建集合/文件夹 | 用户点击按钮 | `TE.match` 显式处理 | ✅ Toast 错误提示 | 低 |
| 重命名集合 | 用户点击按钮 | `TE.match` 显式处理 | ✅ Toast 错误提示 | 低 |
| 删除集合 | 用户点击按钮 | `TE.match` 显式处理 | ✅ Toast 错误提示 | 低 |
| 创建请求 | 用户点击保存 | `TE.match` 显式处理 | ✅ Toast 错误提示 | 低 |
| 编辑集合内容 | Store dispatch（自动触发） | 无错误处理 | ❌ 无提示 | 高 |
| 移动集合/文件夹 | Store dispatch（自动触发） | 无错误处理 | ❌ 无提示 | 高 |
| 更新集合排序 | Store dispatch（自动触发） | 无错误处理 | ❌ 无提示 | 高 |
| 编辑请求内容 | Store dispatch（自动触发） | 无错误处理 | ❌ 无提示 | 高 |
| 移动请求 | Store dispatch（自动触发） | 无错误处理 | ❌ 无提示 | 高 |
| 删除请求 | Store dispatch（自动触发） | 无错误处理 | ❌ 无提示 | 高 |

---

## 八、Toast 提示系统

**文件**：`modules/toast.ts` + `composables/toast.ts`

```typescript
// 配置（toast.ts:13-17）
app.use(Toasted, <ToastOptions>{
  position: "bottom-center",  // 底部居中
  duration: 3000,             // 3秒自动消失
  keepOnHover: true,          // 鼠标悬停时保持
})

// 使用方式（组件内）
const toast = useToast();
toast.success("操作成功");    // 绿色提示
toast.error("操作失败");      // 红色提示
```

---

## 九、关键代码位置索引

| 功能模块 | 文件路径 | 行号 |
|---------|---------|------|
| 后端重试逻辑 | `team-collection.service.ts` | 560-613 |
| 后端错误码定义 | `errors.ts` | 248, 329, 671 |
| Resolver 错误抛出 | `team-collection.resolver.ts` | 各处 |
| throwErr 实现 | `utils.ts` | 44-46 |
| GQLClient runMutation | `GQLClient.ts` | 364-419 |
| 错误码解析 | `GQLClient.ts` | 361-362 |
| 同步框架核心 | `lib/sync/index.ts` | 54-72 |
| storeSyncDefinition | `platform/collections/web/sync.ts` | 233-600+ |
| 错误消息映射 | `helpers/runner/collection-tree.ts` | 4-41 |
| UI 层错误处理 | `components/collections/index.vue` | 934-947, 1017-1024 |
| Toast 配置 | `modules/toast.ts` | 11-18 |

---

## 十、问题诊断与改进建议

### 10.1 现存问题

| 问题 | 严重程度 | 影响 |
|-----|---------|------|
| 后台同步无错误处理 | 🔴 高 | 用户对冲突无感知，前后端数据不一致 |
| 冲突错误码无明确映射 | 🟡 中 | 用户只能看到"Something went wrong" |
| API 层丢失错误类型信息 | 🟡 中 | TypeScript 无法提供类型安全 |
| 无同步状态指示 | 🟡 中 | 用户不知道操作是否真正同步成功 |
| 无冲突后自动恢复机制 | 🔴 高 | 数据不一致持续存在，直到用户刷新 |

### 10.2 改进建议

#### 建议 1：为 storeSyncDefinition 添加统一错误处理

```typescript
// 包装同步操作，添加统一错误处理
const withErrorHandling = <T>(
  op: () => Promise<E.Either<GQLError<string>, T>>,
  operationName: string
) => async () => {
  try {
    const result = await op();
    if (E.isLeft(result)) {
      console.error(`[Sync Error] ${operationName}:`, result.left);
      // 可配置是否显示用户提示
      if (shouldShowUserPrompt(result.left)) {
        toast.error(getSyncErrorMessage(result.left));
      }
      // 触发状态重同步
      triggerFullReconciliation();
    }
    return result;
  } catch (e) {
    console.error(`[Sync Exception] ${operationName}:`, e);
    throw e;
  }
};

// 使用方式
editCollection: withErrorHandling(async ({ partialCollection, collectionIndex }) => {
  // ... 原有逻辑
}, 'editCollection');
```

#### 建议 2：添加冲突错误码到消息映射

```typescript
// 在 getErrorMessage 中添加
case "team_coll/reordering_failed":
case "user_coll/reordering_failed":
  return t("collection.reorder_conflict");  // "集合排序冲突，请刷新后重试"
case "team_req/reordering_failed":
case "user_request/reordering_failed":
  return t("request.reorder_conflict");     // "请求排序冲突，请刷新后重试"
```

#### 建议 3：添加同步状态指示器

在 UI 上显示同步状态：
- ✅ 已同步
- 🔄 同步中
- ❌ 同步失败（点击可查看详情/重试）

#### 建议 4：错误发生后触发自动对账

当检测到同步失败时，自动拉取最新状态与本地对账，避免数据不一致持续存在。

---

## 十一、总结

当前系统的错误传播链路在**用户主动操作**场景下工作良好，有完整的错误提示。但在**后台自动同步**场景下存在严重设计缺陷：

1. **冲突重试失败后**：后端正确返回错误码 → 前端正确解析 → **同步模块静默丢弃**
2. **用户无感知**：本地已乐观更新，但后端实际失败，导致数据不一致
3. **错误信息不明确**：即使有提示，冲突错误也只会显示通用的"Something went wrong"

这是多端协同系统中典型的"乐观更新 vs 实际一致性"矛盾。建议优先为 `storeSyncDefinition` 添加统一的错误处理和用户提示机制，确保用户对同步状态有清晰的认知。
