# Hoppscotch 协同编辑请求的实时同步通道分析

Hoppscotch 中"多端协同编辑请求"指的是团队（Team）场景下多个成员共同维护 TeamCollection / TeamRequest 的场景。它**不是**典型的 CRDT/OT 富文本协同编辑器，而是一个"以数据库为准、以 GraphQL Subscription 广播增量"的协作模型：
- 本地的每次改动都先通过 GraphQL Mutation 写入后端数据库；
- 后端把数据库写入成功作为权威事实源；
- 然后向同 Team 的其他在线客户端广播一条"实体 X 发生了 Y 变化"的通知；
- 其它客户端据此把自己的内存树（集合/请求树）增量更新。

下面按"连接建立 → 操作合并 → 离线恢复"三段展开，并把每段对应的源码位置列清楚。

---

## 1. 连接建立

连接建立分成"HTTP 鉴权查询通道"和"WebSocket 订阅通道"两条，它们共用同一套鉴权信息。

### 1.1 后端：把 Subscription 挂到 GraphQL 上

后端在 `packages/hoppscotch-backend/src/app.module.ts` 的 `GraphQLModule.forRootAsync` 里开启订阅：

```ts
installSubscriptionHandlers: true,
subscriptions: {
  'subscriptions-transport-ws': {
    path: '/graphql',
    onConnect: (connectionParams, websocket) => { ... }
  },
},
```

关键点：

- 使用 `subscriptions-transport-ws`（apollo 旧协议，前端用同名包），路径是 `/graphql`。
- `onConnect` 会尝试从 `connectionParams.Authorization` 或 cookie 里取 access token，拼出 `{ headers: { authorization, cookie } }` 作为每个订阅的 GQL Context，供 `GqlAuthGuard` / `GqlTeamMemberGuard` 使用。
- 广播的"总线"由 `PubSubService` 提供，实现在 `packages/hoppscotch-backend/src/pubsub/pubsub.service.ts`：

```ts
export class PubSubService implements OnModuleInit {
  private pubsub: LocalPubSub; // 来自 graphql-subscriptions
  onModuleInit() { this.pubsub = new LocalPubSub(); }
  asyncIterator<T>(topic) { return this.pubsub.asyncIterableIterator(topic); }
  async publish<T>(topic, payload) { await this.pubsub.publish(topic, payload); }
}
```

- `LocalPubSub` 是进程内的事件总线。源码注释写的是"dev 用 local、prod 用 Redis"，但当前仓库只实现了本地模式，意味着多实例部署下订阅不会跨进程广播（这是一个潜在的部署坑）。

### 1.2 主题（Topic）的统一约定

所有协作相关的主题都写在 `packages/hoppscotch-backend/src/pubsub/topicsDefs.ts` 的 `TopicDef` 类型里，形式是 `` `${entity}/${id}/${event}` ``，例如：

```ts
team_coll/${teamID}/coll_added | coll_updated | coll_removed | coll_moved | coll_order_updated
team_req/${teamID}/req_created | req_updated | req_moved | req_deleted | req_order_updated
team/${teamID}/member_added | member_updated | member_removed
team_environment/${id}/created | updated | deleted
...
```

`PubSubService.publish<T>` 的泛型 `T extends keyof TopicDef` 把 topic 和 payload 类型绑死，发布时类型错误会在编译期暴露。

### 1.3 前端：建立 GQL 客户端与订阅 WebSocket

前端入口是 `packages/hoppscotch-common/src/helpers/backend/GQLClient.ts`，它基于 urql + subscriptions-transport-ws：

```ts
const BACKEND_WS_URL = import.meta.env.VITE_BACKEND_WS_URL ?? "wss://api.hoppscotch.io/graphql";

const createSubscriptionClient = () =>
  new SubscriptionClient(BACKEND_WS_URL, {
    reconnect: true,                                // 断线自动重连
    connectionParams: () => platform.auth.getBackendHeaders(), // 每次重连都带最新 token
    connectionCallback(error) { /* 上报错误 */ },
  });

const createHoppClient = () => {
  const exchanges = [authExchange(...), fetchExchange, errorExchange(...)];
  if (subscriptionClient) {
    exchanges.push(subscriptionExchange({
      forwardSubscription: (op) => subscriptionClient!.request(op),
    }));
  }
  return createClient({ url: BACKEND_GQL_URL, exchanges, ... });
};
```

鉴权由 `authExchange` 处理 token 失效 → `platform.auth.refreshAuthToken` → 成功后重试；token 刷新失败则登出。这一步决定了"连接建立时的身份"。

`initBackendGQLClient` 还挂了一个钩子 `platform.auth.onBackendGQLClientShouldReconnect`，在登录/登出/token 刷新后：
- 有用户且没订阅客户端 → 创建；
- 有用户且已有客户端 → 先 `close()` 再重建；
- 没用户 → 销毁。

### 1.4 前端：把订阅注册到协作服务

业务层把订阅集中注册在 `TeamCollectionsService`（`packages/hoppscotch-common/src/services/team-collection.service.ts`）的 `registerSubscriptions`。当用户切到某个 team：

```ts
changeTeamID(newTeamID) {
  this.teamID = newTeamID;
  this.collections.value = [];
  this.entityIDs.clear();
  this.unsubscribeSubscriptions();
  if (this.teamID) this.initialize();
}
```

`initialize` 先 `loadRootCollections`（普通 GQL Query，走 fetchExchange）拉全量，再 `registerSubscriptions` 用 `runGQLSubscription` 把 `TeamCollectionAdded / TeamRequestUpdated / TeamRequestOrderUpdated ...` 这些流都挂进去。每个订阅都把 `teamID` 作为变量传后端，后端用 `GqlTeamMemberGuard` 校验成员身份再放行。

订阅的 .graphql 文件都放在 `packages/hoppscotch-common/src/helpers/backend/gql/subscriptions/`，例如 `TeamRequestUpdated.graphql`：

```graphql
subscription TeamRequestUpdated($teamID: ID!) {
  teamRequestUpdated(teamID: $teamID) { id collectionID request title }
}
```

`runGQLSubscription`（在 `GQLClient.ts`）返回一个 `[rxjs Subject, wonka Subscription]`，前者用来 `subscribe` 结果，后者用来 `unsubscribe()` 清理。

### 1.5 后端：Resolver 把 PubSub 暴露成 GraphQL Subscription

以 TeamRequest 为例（`team-request.resolver.ts`）：

```ts
@Subscription(() => TeamRequest, { ... })
@SkipThrottle()
@UseGuards(GqlAuthGuard, GqlTeamMemberGuard)
@RequiresTeamRole(VIEWER, EDITOR, OWNER)
teamRequestUpdated(@Args({ name: 'teamID', type: () => ID }) teamID: string) {
  return this.pubsub.asyncIterator(`team_req/${teamID}/req_updated`);
}
```

关键点：
- 订阅本身是幂等的"读"操作，所以 `@SkipThrottle()` 不受限流。
- 鉴权靠 Guard，不是 topic 名字本身——topic 只是个字符串通道。
- `resolve: (value) => value` 用 PubSub 发出的原始 payload 做 GQL 返回值。

---

## 2. 操作合并（Operation Merge）

这里的"合并"不是 OT/CRDT 里的 transform/merge，而是"**远端变更怎么安全地落到本地内存树，避免把我自己刚发的操作再应用一遍**"。

### 2.1 服务端：先写数据库，再广播

写路径一律是 Mutation → Service → Prisma。核心约束是"数据库是权威源"：

1. Mutation 里调用 Service 做修改；
2. Service 用 Prisma 事务写库，必要时用 `lockTeamRequestByCollections` / `lockTeamCollectionByTeamAndParent` 显式拿行级锁，防止并发改 orderIndex 时乱序；
3. 写成功后才 `this.pubsub.publish(topic, payload)`。

`team-request.service.ts` 的 `createTeamRequest` 片段：

```ts
dbTeamRequest = await this.prisma.$transaction(async (tx) => {
  await this.prisma.lockTeamRequestByCollections(tx, teamID, [collectionID]);
  const lastTeamRequest = await tx.teamRequest.findFirst({ where: { collectionID }, orderBy: { orderIndex: 'desc' } });
  return tx.teamRequest.create({
    data: { request: jsonReq, title, orderIndex: lastTeamRequest ? lastTeamRequest.orderIndex + 1 : 1, team: { connect: { id: team.right.id } }, collection: { connect: { id: collectionID } } },
  });
});
this.pubsub.publish(`team_req/${teamRequest.teamID}/req_created`, teamRequest);
```

`moveCollection` / `moveRequest` 用 `MAX_RETRIES = 5` 做死锁重试（`PrismaError.TRANSACTION_DEADLOCK / UNIQUE_CONSTRAINT_VIOLATION / TRANSACTION_TIMEOUT`），每次退避 `retryCount * 100ms`。这是"写时合并"的关键：多端同时拖同一个集合时，数据库的锁 + 重试保证最终有一个顺序，客户端用订阅到的结果对齐。

### 2.2 客户端：幂等合并靠 entityIDs

`TeamCollectionsService` 维护了一个 `entityIDs: Set<string>`，键形如 `collection-${id}` / `request-${id}`。每次收到订阅事件时先检查：

```ts
private addCollection(collection, parentID) {
  if (this.entityIDs.has(`collection-${collection.id}`)) return;
  // ... 真正加到树里
  this.entityIDs.add(`collection-${id}`);
}

private addRequest(request) {
  if (this.entityIDs.has(`request-${request.id}`)) return;
  // ...
  this.entityIDs.add(`request-${request.id}`);
}
```

这带来两个效果：

- **自己触发的变更不会重复应用**：Mutation 成功后本地通常已经通过 mutation 返回值或乐观更新把树改了，订阅再推一条相同的 `req_created` 时 `entityIDs` 会把它挡掉。
- **订阅乱序 / 重放**：因为 PubSub 本身不保证 exactly-once，所以靠这个 Set 做幂等。

### 2.3 客户端：远端改动落到本地树的策略

针对不同事件类型，合并策略不一样（都在 `team-collection.service.ts`）：

| 事件 | 处理方法 | 说明 |
| --- | --- | --- |
| `teamCollectionAdded` | `addCollection(coll, parentID)` | 找不到 parent 时静默放弃（表示那个分支还没展开） |
| `teamCollectionUpdated` | `updateCollection({id, title, data})` | `Object.assign` 到已存在节点上 |
| `teamCollectionRemoved` | `removeCollection(id)` + `entityIDs.delete` | 递归移除子树 |
| `teamRequestAdded` | `addRequest(req)` | 若所在 collection 未展开则跳过 |
| `teamRequestUpdated` | `updateRequest({id, collectionID, request, title})` | 只改已在树里的节点 |
| `teamRequestDeleted` | `removeRequest(id)` | |
| `teamRequestMoved` | `moveRequest(req)` → 先 `removeRequest` 再 `addRequest` 到新 collection | |
| `teamRequestOrderUpdated` | `updateRequestOrder(src, dst, coll)` → 用 `reorderItems` 做本地 array splice | 只动客户端数组，不重新拉数据 |
| `teamCollectionMoved` | `moveCollection(id, parentID, title, data)` → 先 remove 再按新 parent add | |
| `teamCollectionOrderUpdated` | `updateCollectionOrder` | 同上，处理 root / child 两种情况 |
| `teamRootCollectionsSorted` | `loadRootCollections(true)` | 整段重拉，replace 模式 |
| `teamChildCollectionsSorted` | `expandCollection(id, true)` | 强制 reFetch 子节点 |

注意：如果目标节点还没进 `entityIDs`（比如某个 collection 从未被展开过），对应的 `update*` / `move*` 基本都是直接 return，等用户真正展开那个节点时再用 Query 拉最新状态。这是"按需加载 + 增量通知"折中的结果。

### 2.4 写冲突怎么处理

Hoppscotch 选择了"最后写入者胜（last-write-wins）+ 不回滚本地"：
- 服务端靠 Prisma 事务 + 行锁保证 orderIndex / 唯一性约束不会出错；
- 客户端靠订阅通知把远端结果最终对齐；
- `teamRequestUpdated` 推过来时本地若也在改同一条请求，UI 层不会阻止用户编辑，最终以订阅推到的"数据库里的最新值"覆盖本地。

换句话说：**没有本地 pending queue、没有版本向量、没有基于 vector clock 的冲突检测**。它依赖"协作是粗粒度的"——请求/集合是整体替换，不是逐字符编辑——所以冲突代价低，LWW 足够。

---

## 3. 离线恢复（Reconnection）

### 3.1 传输层重连：SubscriptionClient 自动做

前端用的 `subscriptions-transport-ws` 在构造时传了 `reconnect: true`，所以只要 WebSocket 断开它就会指数退避重连，重连时 `connectionParams` 会重新调用 `platform.auth.getBackendHeaders()` 拿最新 token。

`GQLClient.ts` 里 `onBackendGQLClientShouldReconnect` 在 token 刷新时主动 `subscriptionClient.client.close()`，触发一次干净的重连（因为旧的 ws 可能带着过期 token）。

### 3.2 业务层重连：订阅会重新跑一次，但没有"补期间事件"

`runGQLSubscription` 返回的 `result$` 是 RxJS Subject，订阅在网络断开时不会收到任何事件。重连成功后：

- Apollo Server 会把 `AsyncIterator` 重新连上 `this.pubsub.asyncIterator(topic)`；
- 但**离线期间发生的那些事件已经被丢掉了**——`LocalPubSub` 是即时的内存发布，没有持久化，也没有"自上次收到的游标"。

所以业务层的恢复方式只能是"**重新拉全量**"。目前仓库没有看到主动的"断线后自动重新 `loadRootCollections`"逻辑，用户需要手动切走再切回 team，或者刷新页面。如果要补强，可以在：

- `TeamCollectionsService.registerSubscriptions` 里用 urql 的 subscription `error` 事件监听断连；
- 或订阅 `subscriptionClient.onReconnected` 后跑一次 `loadRootCollections(true)`。

### 3.3 写操作离线：直接抛错

如果在离线期间用户触发 `createTeamRequest` / `moveCollection` 等 Mutation，因为走的是 `fetchExchange`（HTTP POST），请求会直接失败：

- `runGQLQuery` / `runMutation` 返回 `E.Left(GQLError)`；
- `team-collection.service.ts` 里所有 Mutation 都没有"入本地队列等待上线后 replay"的代码；
- UI 层通常直接弹 `toast.error` 让用户重试。

也就是说：**离线只读体验尚可（靠本地缓存的树继续浏览），但离线写是禁止的**。这是"以数据库为准"模型的自然结论——没有 DB 就不能写。

### 3.4 鉴权失效：token 刷新与登出

`authExchange` 里 `didAuthError` 检测 `auth/fail`、`jwt expired`、`UNAUTHENTICATED`，命中就 `platform.auth.refreshAuthToken`。刷新成功后 urql 会重放失败的 operation；刷新失败则 `platform.auth.signOutUser()`，此时 `onBackendGQLClientShouldReconnect` 会走到 `!currentUser && subscriptionClient` 分支，把订阅客户端关掉，避免拿着失效 token 无限重连。

---

## 4. 一张小图总结

```
 浏览器 A ──Mutation──► /graphql (HTTP)  ─┐
                                          │
                                     NestJS Resolver
                                          │
                                     Prisma (行锁 + 事务 + 死锁重试)  ← 权威源
                                          │
                                     PubSubService.publish(`team_req/${id}/req_updated`, req)
                                          │
                ┌─────────────────────────┴──────────────────────────┐
                │                                                    │
          AsyncIterator A                                      AsyncIterator B
                │                                                    │
          SubscriptionClient A (reconnect:true)               SubscriptionClient B (reconnect:true)
                │                                                    │
          TeamCollectionsService A ──addRequest/updateRequest──  entityIDs 幂等挡掉重复
```

## 5. 源码定位速查表

| 关注点 | 文件 |
| --- | --- |
| GQL 订阅通道配置、`onConnect` 鉴权 | `packages/hoppscotch-backend/src/app.module.ts` |
| PubSub 实现（进程内 `graphql-subscriptions`） | `packages/hoppscotch-backend/src/pubsub/pubsub.service.ts` |
| 所有协作相关 Topic 约定 | `packages/hoppscotch-backend/src/pubsub/topicsDefs.ts` |
| TeamRequest 订阅 Resolver | `packages/hoppscotch-backend/src/team-request/team-request.resolver.ts` |
| TeamRequest 写操作 + 行锁 + publish | `packages/hoppscotch-backend/src/team-request/team-request.service.ts` |
| TeamCollection 订阅 Resolver | `packages/hoppscotch-backend/src/team-collection/team-collection.resolver.ts` |
| TeamCollection 写操作 + 死锁重试 | `packages/hoppscotch-backend/src/team-collection/team-collection.service.ts` |
| 前端 GQL 客户端、SubscriptionClient、reconnect、authExchange | `packages/hoppscotch-common/src/helpers/backend/GQLClient.ts` |
| 业务层订阅注册、entityIDs 幂等、合并策略 | `packages/hoppscotch-common/src/services/team-collection.service.ts` |
| 订阅的 .graphql 文档 | `packages/hoppscotch-common/src/helpers/backend/gql/subscriptions/` |
| `moveCollection` 失败路径、锁顺序、死锁重试（`MAX_RETRIES`） | `hoppscotch-backend/src/team-collection/team-collection.service.ts:560-616, 747-867` |
| `moveRequest` 失败路径、事务内"已被并发删除"静默空操作 | `hoppscotch-backend/src/team-request/team-request.service.ts:312-353, 423-516` |
| 22 个订阅解绑、`changeTeamID` / `clearCollections` | `hoppscotch-common/src/services/team-collection.service.ts:202-255` |
| `expandCollection` 错误兜底、`loadingCollections` 软锁 | `hoppscotch-common/src/services/team-collection.service.ts:1030-1071` |
| `waitForCollectionLoading` 忙等待、`collectionLoadingWatcher` 延迟补偿 | `hoppscotch-common/src/services/team-collection.service.ts:179-196, 1247-1251` |

## 6. `moveCollection` 与 `moveRequest` 失败路径的差异

两者都是"写库 → 广播"，但失败时的处理粒度、重试策略、锁粒度并不对称。

### 6.1 `TeamCollectionService.moveCollection`（集合移动）

`packages/hoppscotch-backend/src/team-collection/team-collection.service.ts:747-867`：

```ts
async moveCollection(collectionID, destCollectionID) {
  try {
    return await this.prisma.$transaction(async (tx) => {
      // (1) 查 collection；已在 root → TEAM_COL_ALREADY_ROOT
      // (2) destCollectionID == null：只锁旧 parent，调 changeParentAndUpdateOrderIndex(tx, coll, null)
      // (3) destCollectionID != null：
      //     - ID 相同 → TEAM_COLL_DEST_SAME
      //     - 目标 collection 不存在 → TEAM_COLL_NOT_FOUND
      //     - 跨 team → TEAM_COLL_NOT_SAME_TEAM
      //     - 是自己子孙 → TEAM_COLL_IS_PARENT_COLL（isParent 返回 O.none 表示冲突）
      //     - srcParentID / destParentID 按字典序上锁，防双向 move 死锁
      // (4) changeParentAndUpdateOrderIndex(tx, coll, destColl.id)
      // (5) pubsub.publish(`team_coll/${teamID}/coll_moved`, updatedColl)
    });
  } catch (error) {
    console.error('Error from TeamCollectionService.moveCollection', error);
    return E.left(TEAM_COL_REORDERING_FAILED);   // ← 任何未分类错误都"折叠"成这一条
  }
}
```

注意点：
- **整段 try/catch 只包一层**，事务里 Prisma 抛出的任何异常（死锁、唯一冲突、超时）都一律转成 `TEAM_COL_REORDERING_FAILED`。**没有单独的死锁重试**。
- `changeParentAndUpdateOrderIndex`（`:651-688`）内部的 `updateMany(..., { orderIndex: { decrement: 1 } })` + `update({ orderIndex: last+1 })` 是在**同一事务里做的**，失败会整个回滚；但它不自己 catch，抛错走外层的 `TEAM_COL_REORDERING_FAILED`。
- 相比之下，`deleteCollectionAndUpdateSiblingsOrderIndex`（`:555-616`）和 `updateOrderIndex` 才有 `MAX_RETRIES = 5` + `delay(retryCount * 100)` 的死锁重试循环，这些被 `updateCollectionOrder` / `deleteCollection` 复用，但**不被 `moveCollection` 复用**——即移动集合失败不会自动重试，由调用方决定。
- `Resolver.moveCollection`（`team-collection.resolver.ts:302`）对 `E.Left` 直接 `throwErr(left)`，所以客户端看到的是一个 GraphQL 错误，错误消息是 `"TEAM_COL_REORDERING_FAILED"`。

### 6.2 `TeamRequestService.moveRequest`（请求移动）

`packages/hoppscotch-backend/src/team-request/team-request.service.ts:312-353`：

```ts
async moveRequest(srcCollID, requestID, destCollID, nextRequestID, callerFunction) {
  // step 1: 校验 + 查出 request / nextRequest
  const twoRequests = await this.findRequestAndNextRequest(...);
  if (E.isLeft(twoRequests)) return E.left(twoRequests.left);   // ← 这里是"早返回"，不是 throw

  // step 2: 事务
  const updatedRequest = await this.reorderRequests(request, srcCollID, nextRequest, destCollID);
  if (E.isLeft(updatedRequest)) return E.left(updatedRequest.left);

  // step 3: 按 callerFunction 分 topic 广播
  if (callerFunction === 'moveRequest') {
    this.pubsub.publish(`team_req/${teamID}/req_moved`, teamReq);
  } else if (callerFunction === 'updateLookUpRequestOrder') {
    this.pubsub.publish(`team_req/${teamID}/req_order_updated`, {
      request, nextRequest,
    });
  }
  return E.right(teamReq);
}
```

`reorderRequests`（`:423-516`）：

```ts
private async reorderRequests(...) {
  try {
    return await this.prisma.$transaction(async (tx) => {
      await this.prisma.lockTeamRequestByCollections(tx, teamID, [srcCollID, destCollID]);   // ← 锁粒度是集合，不是 parent
      request = await tx.teamRequest.findUnique(...);
      nextRequest = nextRequest ? await tx.teamRequest.findUnique(...) : null;
      if (!request) return;                 // ← 已被并发删除则"静默成功"，不抛错
      // same collection：一次 updateMany 挪 orderIndex
      // diff collection：源 collection 后面的 orderIndex 全部 -1，目标 collection 自 nextRequest.orderIndex 起全部 +1
      // 最后 update({ collectionID: destCollID, orderIndex: newOrderIndex })
    });
  } catch (err) {
    return E.left(TEAM_REQ_REORDERING_FAILED);   // ← 折叠
  }
}
```

对比 `moveCollection` 的关键差异：

| 维度 | `moveCollection` | `moveRequest` |
| --- | --- | --- |
| 锁粒度 | `lockTeamCollectionByTeamAndParent(tx, teamID, parentID)`，按 **parentID** 字典序加两把锁防双向死锁 | `lockTeamRequestByCollections(tx, teamID, [srcCollID, destCollID])`，按 **collectionID 列表** 加锁，不排序 |
| 业务校验（非数据库） | `ALREADY_ROOT / DEST_SAME / NOT_SAME_TEAM / IS_PARENT_COLL` 四种显式返回 | `TEAM_REQ_NOT_FOUND / TEAM_REQ_INVALID_TARGET_COLL_ID` 两种 |
| 事务内"已被删除"分支 | 没处理；如果 collection 已被并发删除，`getCollection` 返回 `E.Left(NOT_FOUND)` 早返回 | `if (!request) return;` —— **静默空操作**，不视为失败 |
| 死锁 / 唯一冲突重试 | **没有**，直接折成 `TEAM_COL_REORDERING_FAILED` | **没有**，直接折成 `TEAM_REQ_REORDERING_FAILED` |
| 发布 Topic | 固定 `team_coll/${id}/coll_moved` | 由 `callerFunction` 决定：`req_moved` 或 `req_order_updated` |
| Resolver 返回 | `TeamCollection` 对象 | `TeamRequest` 对象（`moveRequest`）或 `Boolean`（`updateLookUpRequestOrder`） |

最重要的一点：**两者失败都不会自动重试**——只有 `updateCollectionOrder` / `updateRequestOrder` 那条路径（走 `updateOrderIndex` / `reorderRequests` 之外的专门实现）才会在 `deleteCollectionAndUpdateSiblingsOrderIndex` 里做 `MAX_RETRIES=5`。所以拖动（move）冲突时客户端看到的就是一个失败 toast，需要用户再拖一次。

### 6.3 事务内"已被并发删除"的不同处理

请求移动有一个很实用的细节（`team-request.service.ts:439-451`）：
- 上锁后，再 `findUnique` 一次拿最新的 request/nextRequest；
- 如果 request 已经不存在（比如被另一个人删了），**整个事务静默成功，不抛错，不发布任何订阅**。

这避免了"我刚拖到一半被别人删了 → 事务抛错 → 客户端收到失败"这种噪声。`moveCollection` 没有同样的处理——如果集合被并发删除，`getCollection` 会返回 `TEAM_COLL_NOT_FOUND` 给上层，用户会看到失败。

---

## 7. 订阅解绑与状态清理

### 7.1 客户端持有的资源

`TeamCollectionsService`（`packages/hoppscotch-common/src/services/team-collection.service.ts:131-170`）里每个订阅都有两份引用：

- **RxJS 订阅**：`teamCollectionAdded$ / Updated$ / Removed$ / Moved$ / OrderUpdated$` 等 11 个（加上 `teamRequest*` 共 20 个 `Subscription | null`），这些是对 urql 返回的流 `.subscribe(...)` 的产物。
- **Wonka 订阅**：`teamCollectionAddedSub / UpdatedSub / ...` 共 11 个（`WSubscription | null`），来自 `runGQLSubscription` 的第二个返回值，`unsubscribe()` 会真正把 GQL Subscription 从 `SubscriptionClient` 上卸载。

两种都要成对释放，否则：只关 RxJS 订阅会让 GQL Subscription 还在后台跑（服务器仍推），只关 Wonka 订阅会让 RxJS 侧出现永远收不到事件的悬挂流。

### 7.2 触发清理的入口

`changeTeamID(newTeamID)`（`:202-212`）：

```ts
public changeTeamID(newTeamID: string | null) {
  this.teamID = newTeamID;
  this.collections.value = [];         // 清空 vue ref
  this.entityIDs.clear();              // 幂等 Set 清空
  this.loadingCollections.value = [];  // 清空 loading 列表
  this.unsubscribeSubscriptions();     // 关 22 个订阅
  if (this.teamID) this.initialize();
}
```

`clearCollections()`（`:217-223`）是登出 / 完全退出 team 时用的，比 `changeTeamID(null)` 多了一行 `this.teamID = null`（`changeTeamID` 也做了但写在参数里，`clearCollections` 显式强调）。两者都走同一个 `unsubscribeSubscriptions`。

### 7.3 `unsubscribeSubscriptions` 的实现（`:229-255`）

```ts
unsubscribeSubscriptions() {
  this.teamCollectionAdded$?.unsubscribe();
  this.teamCollectionUpdated$?.unsubscribe();
  this.teamCollectionRemoved$?.unsubscribe();
  this.teamRequestAdded$?.unsubscribe();
  this.teamRequestDeleted$?.unsubscribe();
  this.teamRequestUpdated$?.unsubscribe();
  this.teamRequestMoved$?.unsubscribe();
  this.teamCollectionMoved$?.unsubscribe();
  this.teamRequestOrderUpdated$?.unsubscribe();
  this.teamCollectionOrderUpdated$?.unsubscribe();
  this.teamRootCollectionSorted$?.unsubscribe();
  this.teamChildCollectionSorted$?.unsubscribe();

  this.teamCollectionAddedSub?.unsubscribe();
  this.teamCollectionUpdatedSub?.unsubscribe();
  this.teamCollectionRemovedSub?.unsubscribe();
  this.teamRequestAddedSub?.unsubscribe();
  this.teamRequestDeletedSub?.unsubscribe();
  this.teamRequestUpdatedSub?.unsubscribe();
  this.teamRequestMovedSub?.unsubscribe();
  this.teamCollectionMovedSub?.unsubscribe();
  this.teamRequestOrderUpdatedSub?.unsubscribe();
  this.teamCollectionOrderUpdatedSub?.unsubscribe();
  this.teamRootCollectionSortedSub?.unsubscribe();
  this.teamChildCollectionSortedSub?.unsubscribe();
}
```

注意：
- 只负责"停掉订阅"，**不负责置回 null**。`changeTeamID` 之后立刻调 `initialize → registerSubscriptions` 会把这些字段覆盖赋值；如果没有调 `initialize`，字段还挂着旧的 `Subscription` 对象（已经 `unsubscribe` 过，变成 closed 状态），但不会影响后续新订阅。
- 没有 `finalize`/`error` 回调。每个 `subscribe` 里都写了 `if (E.isLeft(result)) throw new Error(...)`——也就是说订阅收流到 `E.Left` 时会直接 throw，RxJS 会把这个错误上报到全局（`onError`），但**不会停止该订阅对应的 WebSocket 订阅**（Wonka 那边依然活着）。要处理这类错误必须 `unsubscribeSubscriptions` 手动关。
- 清理后 `entityIDs` 也被 `.clear()`，保证下一次 `initialize` 后 `loadRootCollections` 可以把同一条 collection 重新加回来——如果忘了清，重新拉出来的集合会被 `addCollection` 的幂等判断直接跳过。

### 7.4 客户端销毁链路

整体链路是：
1. 用户切换 team → `TeamCollectionsService.changeTeamID`；或用户登出 → `TeamCollectionsService.clearCollections`。
2. `GQLClient.ts` 里 `platform.auth.onBackendGQLClientShouldReconnect` 收到登出事件 → `subscriptionClient.client.close()` → `initBackendGQLClient` 把 `subscriptionClient` 置 `null`。
3. 但要注意：**这两个清理是并行的，没有同步关系**。`TeamCollectionsService.unsubscribeSubscriptions` 在 Wonka 侧调用 `unsubscribe()` 时，如果 `subscriptionClient` 已经被 close 了，Wonka 会得到一个 "Client is not connected" 错误被吞掉（看源码 Wonka 的 `SubscriptionClient` source 在 closed 状态下 `unsubscribe` 只是静默）。所以先后顺序无影响，但代码里没有 `try/catch` 保护，理论上极端时序下可能有 warning。

---

## 8. 离线恢复补偿的真实边界

"离线恢复补偿"分三段：WebSocket 重连、期间事件补放、本地 pending 操作 replay。Hoppscotch 的"补偿"只做了第一段，后面两段都没有。下面把代码里能体现边界的细节列出来。

### 8.1 `expandCollection` 的错误兜底（离线时仍能展开 UI 壳）

`team-collection.service.ts:1030-1071`：

```ts
async expandCollection(collectionID, reFetch = false) {
  if (this.loadingCollections.value.includes(collectionID)) return;   // ← 节流：正在展开就不重复

  const tree = this.collections.value;
  const collection = findCollInTree(tree, collectionID);
  if (!collection) return;
  if (collection.children !== null && !reFetch) return;              // ← 已展开过且不强制刷新就不再拉

  this.loadingCollections.value.push(collectionID);
  try {
    const [collections, requests] = await Promise.all([
      this.getCollectionChildren(collection),
      this.getCollectionRequests(collection),
    ]);
    collection.children = collections;
    collection.requests = requests;
    collections.forEach(c => this.entityIDs.add(`collection-${c.id}`));
    requests.forEach(r => this.entityIDs.add(`request-${r.id}`));
    this.collections.value = [...tree];
  } catch (error) {
    console.error(`Error expanding collection ${collectionID}:`, error);
    // 关键兜底：把 children / requests 设成 [] 而不是 null，
    // 防止下次再点展开时陷入"永远失败→每次都重试"的无限循环
    collection.children = [];
    collection.requests = [];
    this.collections.value = [...tree];
  } finally {
    this.loadingCollections.value = this.loadingCollections.value.filter(x => x !== collectionID);
  }
}
```

这段体现的补偿策略是：
- **一旦某次展开失败，就把该 collection 标成"已展开但空"**，后续再调 `expandCollection` 时 `collection.children !== null` 直接 return，避免无限爆错。
- 代价是用户需要切 team 回来或刷新页面才能重新展开——没有重试机制，也没有"重新加载"按钮。

### 8.2 `loadingCollections` 作为同步标记

`loadingCollections` 同时承担三个职责：
1. UI 层显示 loading spinner（外部订阅 `loadingCollections$`）。
2. 作为"我正在展开 collection X"的软锁——防止并发展开同个 collection（`:1031`）。
3. 被 `waitForCollectionLoading(collectionID)`（`:1247-1251`）当作忙等待信号：

```ts
private async waitForCollectionLoading(collectionID: string) {
  while (this.loadingCollections.value.includes(collectionID)) {
    await new Promise((resolve) => setTimeout(resolve, 50));
  }
}
```

这用在 `cascadeParentCollectionForPropertiesAsync` 里，级联查 path 上每个 collection 的继承属性时，如果某个 collection 还没展开就先等它展开完。**缺点是纯轮询**，超时没有上限，理论上网络挂死会一直等。

### 8.3 `collectionLoadingWatcher`：loading 归零时刷继承属性

`team-collection.service.ts:179-196`：

```ts
private collectionLoadingWatcher() {
  watch(
    () => this.loadingCollections.value.length,
    (loadingCount) => {
      if (loadingCount === 0 &&
          this.pendingTeamCollectionPath.value &&
          this.collections.value.length > 0) {
        updateInheritedPropertiesForAffectedRequests(
          this.pendingTeamCollectionPath.value,
          'rest',
        );
        this.pendingTeamCollectionPath.value = null;
      }
    },
  );
}
```

这是"补偿"的另一种形式：当某个 collection 正在展开期间有人想算它的继承属性，不直接失败，而是把 path 存到 `pendingTeamCollectionPath`，等 `loadingCollections` 清空后自动重算。

### 8.4 `loadRootCollections(replace=true)` 是唯一的全量补偿入口

`team-collection.service.ts:302-372` 里 `replace=true` 时：
```ts
if (replace) {
  this.collections.value = [];
  this.entityIDs.clear();
  totalCollections.push(...);
} else {
  totalCollections.push(...);
}
// ...
this.collections.value.push(...totalCollections);
```

注意 **`replace=true` 只清了 root collections，不会清掉已展开的子 collection 的 `children/requests`**——但因为子 collection 是从父 collection 的 `children` 数组里引用的，父 collection 被整棵替换掉后子树自然就没了。`entityIDs.clear()` 保证下一次重新展开不会被幂等挡掉。

### 8.5 订阅重连时不自动触发 `loadRootCollections(replace=true)`

这是最明显的"没做补偿"的地方。`runGQLSubscription` 返回的流不会在 WebSocket 重连后补发任何东西；`registerSubscriptions` 里也没有订阅 `subscriptionClient.onReconnected` 然后重新 `loadRootCollections(true)`。实际效果是：
- 断线期间别人做的改动在重连后**不会自动反映**；
- 用户如果知道有这个问题，可以手动切换 team 或刷新；
- 如果只靠 `subscriptionClient.reconnect:true`，UI 会表现成"数据停留在断网前最后一帧，直到自己再触发一条 mutation 才把那边的最新值推回来"。

### 8.6 离线写：没有 replay，没有乐观更新队列

所有写操作（`createTeamRequest` / `updateTeamRequest` / `moveRequest` / `moveCollection` / `updateRequestOrder` 等）都是直接 `runGQLQuery` 调 Mutation，失败就返回 `E.Left`。没有：
- 本地 pending queue
- `offline-first` 乐观更新
- 上线后自动 replay
- 基于 vector clock / version 的冲突合并

离线读体验相对可接受——`collections.value` 已经是 vue ref 的完整内存树，断网后仍能翻树、选中请求、看详情；但断网期间任何写都会 toast 报错。

### 8.7 边界清单

| 场景 | 实际行为 | 代码位置 |
| --- | --- | --- |
| 拖动集合到一个它自己的子树里 | 返回 `TEAM_COLL_IS_PARENT_COLL` | `team-collection.service.ts:802-809` |
| 拖动请求时源/目标集合已被别人删 | 事务内再查一次，静默空操作 | `team-request.service.ts:439-451` |
| 拖动集合时集合已被别人删 | `getCollection` 返回 `TEAM_COLL_NOT_FOUND` 早返回 | `team-collection.service.ts:751-752` |
| 同 team 内双向同时 move collection | 按 parentID 字典序上锁，防止死锁 | `team-collection.service.ts:813-845` |
| 同 team 内双向同时 move request | 按 collectionID 列表上锁，不排序，可能死锁 | `team-request.service.ts:434-437` |
| 死锁 / 唯一冲突 / 事务超时 | `move*` 直接失败，`updateOrderIndex*` 重试 5 次 | `team-collection.service.ts:560-616` |
| 展开 collection 失败 | 标成空数组防无限重试，下次刷新才能再开 | `team-collection.service.ts:1057-1065` |
| 级联继承属性计算时遇到未展开 collection | `waitForCollectionLoading` 轮询，无超时 | `team-collection.service.ts:1247-1251` |
| WebSocket 重连后数据对齐 | 不自动拉全量，需手动切 team 或刷新 | `GQLClient.ts` + `TeamCollectionsService` |
| 离线写 | 直接失败，无队列 | 所有 Mutation 调用处 |

## 9. 现状的边界（重要）

阅读代码时容易误判的几点：

1. **没有操作队列，也没有 CRDT/OT**。任何"断网期间编辑，上线后自动合并"的想象都不成立。
2. **PubSub 是进程内的**。多实例部署时订阅事件不会跨节点广播，需替换成 RedisPubSub 或外部 broker 才能真正多活。
3. **"合并"只是幂等应用远端增量**，不处理本地未提交修改与远端修改的细粒度冲突；冲突时以后到者为准。
4. **没有持久化的重放游标**，重连后需要主动拉全量，目前没有自动触发逻辑。
5. `packages/hoppscotch-common/src/newstore/*Session.ts` 和 `helpers/realtime/*` 那几个文件是"前端作为 WebSocket/SSE/Socket.IO/MQTT 客户端去调试外部服务"的功能，**与团队协作无关**，别把它们当成同步通道。
