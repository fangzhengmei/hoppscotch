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

## 6. 现状的边界（重要）

阅读代码时容易误判的几点：

1. **没有操作队列，也没有 CRDT/OT**。任何"断网期间编辑，上线后自动合并"的想象都不成立。
2. **PubSub 是进程内的**。多实例部署时订阅事件不会跨节点广播，需替换成 RedisPubSub 或外部 broker 才能真正多活。
3. **"合并"只是幂等应用远端增量**，不处理本地未提交修改与远端修改的细粒度冲突；冲突时以后到者为准。
4. **没有持久化的重放游标**，重连后需要主动拉全量，目前没有自动触发逻辑。
5. `packages/hoppscotch-common/src/newstore/*Session.ts` 和 `helpers/realtime/*` 那几个文件是"前端作为 WebSocket/SSE/Socket.IO/MQTT 客户端去调试外部服务"的功能，**与团队协作无关**，别把它们当成同步通道。
