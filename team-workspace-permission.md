# 团队工作区角色权限传播机制分析

## 1. 核心角色定义

### 1.1 角色枚举定义

系统定义了三种团队角色，在数据库层和应用层保持一致：

**数据库层（Prisma Schema）**
[prisma/schema.prisma:331-335](packages/hoppscotch-backend/prisma/schema.prisma#L331-L335)
```prisma
enum TeamAccessRole {
  OWNER
  VIEWER
  EDITOR
}
```

**应用层（TypeScript 模型）**
[src/team/team.model.ts:31-35](packages/hoppscotch-backend/src/team/team.model.ts#L31-L35)
```typescript
export enum TeamAccessRole {
  OWNER = 'OWNER',
  VIEWER = 'VIEWER',
  EDITOR = 'EDITOR',
}
```

### 1.2 角色权限矩阵

| 角色   | 查看资源 | 编辑资源 | 管理成员 | 删除团队 |
|--------|----------|----------|----------|----------|
| OWNER  | ✅       | ✅       | ✅       | ✅       |
| EDITOR | ✅       | ✅       | ❌       | ❌       |
| VIEWER | ✅       | ❌       | ❌       | ❌       |

## 2. 成员归属模型

### 2.1 数据库关系模型

[prisma/schema.prisma:20-28](packages/hoppscotch-backend/prisma/schema.prisma#L20-L28)
```prisma
model TeamMember {
  id      String         @id @default(uuid())
  role    TeamAccessRole
  userUid String
  teamID  String
  team    Team           @relation(fields: [teamID], references: [id], onDelete: Cascade)

  @@unique([teamID, userUid])
}
```

**关键约束**：
- `@@unique([teamID, userUid])` 确保一个用户在一个团队中只有一个角色
- `onDelete: Cascade` 团队删除时级联删除成员关系

### 2.2 成员管理服务

[src/team/team.service.ts:78-103](packages/hoppscotch-backend/src/team/team.service.ts#L78-L103)

核心成员管理方法：
- `addMemberToTeam(teamID, uid, role)` - 添加成员
- `updateTeamAccessRole(teamID, userUid, newRole)` - 更新角色
- `leaveTeam(teamID, userUid)` - 离开/移除成员
- `getTeamMember(teamID, userUid)` - 获取成员信息
- `getRoleOfUserInTeam(teamID, userUid)` - 获取用户角色

**特殊约束**：团队必须至少有一个 OWNER [src/team/team.service.ts:157-180](packages/hoppscotch-backend/src/team/team.service.ts#L157-L180)

## 3. 权限检查机制

### 3.1 权限装饰器

[src/team/decorators/requires-team-role.decorator.ts:1-5](packages/hoppscotch-backend/src/team/decorators/requires-team-role.decorator.ts#L1-L5)
```typescript
export const RequiresTeamRole = (...roles: TeamAccessRole[]) =>
  SetMetadata('requiresTeamRole', roles);
```

**使用方式**：在 GraphQL resolver 方法上标记所需角色
```typescript
@RequiresTeamRole(TeamAccessRole.OWNER, TeamAccessRole.EDITOR)
```

### 3.2 守卫（Guard）体系

系统为不同资源层级实现了专门的守卫，核心逻辑一致但资源定位方式不同：

#### 通用权限检查流程

```
请求到达 → 提取用户身份 → 提取资源ID → 定位所属团队 → 获取用户角色 → 验证角色权限
```

---

#### 守卫类型对比

| 守卫类                     | 协议类型 | 资源级别 | 提取参数   | 定位方式                     | 文件                                                                 |
|--------------------------|----------|----------|------------|------------------------------|----------------------------------------------------------------------|
| GqlTeamMemberGuard       | GraphQL  | 团队级   | teamID     | 直接使用 teamID              | [src/team/guards/gql-team-member.guard.ts](packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts) |
| GqlCollectionTeamMemberGuard | GraphQL | 集合级 | collectionID | 通过 collection → teamID     | [src/team-collection/guards/gql-collection-team-member.guard.ts](packages/hoppscotch-backend/src/team-collection/guards/gql-collection-team-member.guard.ts) |
| GqlRequestTeamMemberGuard | GraphQL  | 请求级   | requestID  | 通过 request → teamID        | [src/team-request/guards/gql-request-team-member.guard.ts](packages/hoppscotch-backend/src/team-request/guards/gql-request-team-member.guard.ts) |
| GqlTeamEnvTeamGuard      | GraphQL  | 环境级   | id         | 通过 environment → teamID    | [src/team-environments/gql-team-env-team.guard.ts](packages/hoppscotch-backend/src/team-environments/gql-team-env-team.guard.ts) |
| RESTTeamMemberGuard      | REST     | 团队级   | teamID     | 从 URL params 提取 teamID    | [src/team/guards/rest-team-member.guard.ts](packages/hoppscotch-backend/src/team/guards/rest-team-member.guard.ts) |

---

### 3.3 团队级守卫实现（直接权限）

[src/team/guards/gql-team-member.guard.ts:21-43](packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts#L21-L43)

```typescript
async canActivate(context: ExecutionContext): Promise<boolean> {
  // 1. 获取装饰器标记的所需角色
  const requireRoles = this.reflector.get<TeamAccessRole[]>(
    'requiresTeamRole',
    context.getHandler(),
  );
  
  // 2. 获取当前用户
  const { user } = gqlExecCtx.getContext().req;
  
  // 3. 提取 teamID（直接从参数获取）
  const { teamID } = gqlExecCtx.getArgs<{ teamID: string }>();
  
  // 4. 查询用户在该团队的成员信息
  const teamMember = await this.teamService.getTeamMember(teamID, user.uid);
  
  // 5. 验证角色
  return requireRoles.includes(teamMember.role);
}
```

**典型应用场景**：
- 查看团队详情
- 重命名团队
- 删除团队
- 管理成员角色

---

### 3.4 集合级守卫实现（间接权限）

[src/team-collection/guards/gql-collection-team-member.guard.ts:24-50](packages/hoppscotch-backend/src/team-collection/guards/gql-collection-team-member.guard.ts#L24-L50)

```typescript
async canActivate(context: ExecutionContext): Promise<boolean> {
  // 1. 获取所需角色
  const requireRoles = this.reflector.get<TeamAccessRole[]>(...);
  
  // 2. 获取当前用户
  const { user } = gqlExecCtx.getContext().req;
  
  // 3. 提取 collectionID
  const { collectionID } = gqlExecCtx.getArgs<{ collectionID: string }>();
  
  // 4. 通过 collection 找到所属 team（权限派生关键步骤）
  const collection = await this.teamCollectionService.getCollection(collectionID);
  const teamID = collection.right.teamID;
  
  // 5. 查询用户在该团队的角色
  const member = await this.teamService.getTeamMember(teamID, user.uid);
  
  // 6. 验证角色
  return requireRoles.includes(member.role);
}
```

**权限派生链路**：
```
collectionID → TeamCollection.teamID → TeamMember.role → 权限验证
```

**典型应用场景**：
- 创建/删除集合
- 重命名集合
- 移动集合位置
- 导入/导出集合

---

### 3.5 请求级守卫实现（深度间接权限）

[src/team-request/guards/gql-request-team-member.guard.ts:25-53](packages/hoppscotch-backend/src/team-request/guards/gql-request-team-member.guard.ts#L25-L53)

**权限派生链路**：
```
requestID → TeamRequest.teamID → TeamMember.role → 权限验证
```

**典型应用场景**：
- 创建/编辑请求
- 删除请求
- 移动请求到其他集合

---

### 3.6 环境级守卫实现

[src/team-environments/gql-team-env-team.guard.ts:30-56](packages/hoppscotch-backend/src/team-environments/gql-team-env-team.guard.ts#L30-L56)

**权限派生链路**：
```
environmentID → TeamEnvironment.teamID → TeamMember.role → 权限验证
```

**典型应用场景**：
- 编辑环境变量
- 删除环境
- 复制环境

---

### 3.7 REST 接口守卫实现

[src/team/guards/rest-team-member.guard.ts:21-46](packages/hoppscotch-backend/src/team/guards/rest-team-member.guard.ts#L21-L46)

```typescript
async canActivate(context: ExecutionContext): Promise<boolean> {
  // 1. 获取所需角色
  const requireRoles = this.reflector.get<TeamAccessRole[]>(...);
  
  // 2. 获取当前用户（从 REST request 对象）
  const request = context.switchToHttp().getRequest();
  const { user } = request;
  
  // 3. 从 URL params 提取 teamID（如 /team-collection/search/:teamID）
  const teamID = request.params.teamID;
  
  // 4. 查询用户在该团队的角色
  const teamMember = await this.teamService.getTeamMember(teamID, user.uid);
  
  // 5. 验证角色
  return requireRoles.includes(teamMember.role);
}
```

**权限派生链路**：
```
URL params.teamID → TeamMember.role → 权限验证
```

**典型应用场景**：
- REST API 搜索团队集合
- 其他 REST 风格的团队资源访问

---

### 3.8 重要修正：创建操作的守卫选择

**创建操作的特殊逻辑**：由于资源创建时目标资源ID尚未生成，需要通过"父资源ID"或"直接teamID"进行权限验证。

#### 创建请求（createRequestInCollection）
- **使用的守卫**：`GqlCollectionTeamMemberGuard`（不是 `GqlRequestTeamMemberGuard`）
- **原因**：创建时 requestID 不存在，需要通过 `collectionID` 定位团队
- **权限链路**：`collectionID → TeamCollection.teamID → TeamMember.role`

[src/team-request/team-request.resolver.ts:126-154](packages/hoppscotch-backend/src/team-request/team-request.resolver.ts#L126-L154)
```typescript
@Mutation(() => TeamRequest)
@UseGuards(GqlAuthGuard, GqlCollectionTeamMemberGuard)  // 注意这里用的是集合级守卫
@RequiresTeamRole(TeamAccessRole.EDITOR, TeamAccessRole.OWNER)
async createRequestInCollection(
  @Args('collectionID') collectionID: string,
  @Args('data') data: CreateTeamRequestInput,
) { ... }
```

#### 创建环境（createTeamEnvironment）
- **使用的守卫**：`GqlTeamMemberGuard`（不是 `GqlTeamEnvTeamGuard`）
- **原因**：创建时 environmentID 不存在，需要通过 `teamID` 直接验证
- **权限链路**：`teamID → TeamMember.role`

[src/team-environments/team-environments.resolver.ts:30-47](packages/hoppscotch-backend/src/team-environments/team-environments.resolver.ts#L30-L47)
```typescript
@Mutation(() => TeamEnvironment)
@UseGuards(GqlAuthGuard, GqlTeamMemberGuard)  // 注意这里用的是团队级守卫
@RequiresTeamRole(TeamAccessRole.OWNER, TeamAccessRole.EDITOR)
async createTeamEnvironment(@Args() args: CreateTeamEnvironmentArgs) { ... }
```

## 4. 资源与权限映射

### 4.1 GraphQL Resolver 权限配置示例

**团队集合 Resolver** [src/team-collection/team-collection.resolver.ts](packages/hoppscotch-backend/src/team-collection/team-collection.resolver.ts)

| 操作           | 所需角色                  | 守卫类型                     |
|----------------|---------------------------|------------------------------|
| 查看集合       | OWNER, EDITOR, VIEWER     | GqlTeamMemberGuard           |
| 创建根集合     | OWNER, EDITOR             | GqlTeamMemberGuard           |
| 创建子集合     | OWNER, EDITOR             | GqlCollectionTeamMemberGuard |
| 重命名集合     | OWNER, EDITOR             | GqlCollectionTeamMemberGuard |
| 删除集合       | OWNER, EDITOR             | GqlCollectionTeamMemberGuard |
| 移动集合       | OWNER, EDITOR             | GqlCollectionTeamMemberGuard |

**团队 Resolver** [src/team/team.resolver.ts](packages/hoppscotch-backend/src/team/team.resolver.ts)

| 操作           | 所需角色                  | 守卫类型                     |
|----------------|---------------------------|------------------------------|
| 查看团队       | OWNER, EDITOR, VIEWER     | GqlTeamMemberGuard           |
| 创建团队       | 登录用户（自动成为OWNER） | 仅认证守卫                   |
| 重命名团队     | OWNER                     | GqlTeamMemberGuard           |
| 删除团队       | OWNER                     | GqlTeamMemberGuard           |
| 更新成员角色   | OWNER                     | GqlTeamMemberGuard           |
| 移除成员       | OWNER                     | GqlTeamMemberGuard           |

**团队请求 Resolver** [src/team-request/team-request.resolver.ts](packages/hoppscotch-backend/src/team-request/team-request.resolver.ts)

| 操作           | 所需角色                  | 守卫类型                     | 说明                                  |
|----------------|---------------------------|------------------------------|---------------------------------------|
| 搜索请求       | OWNER, EDITOR, VIEWER     | GqlTeamMemberGuard           | 通过 teamID 直接验证                  |
| 查看请求详情   | OWNER, EDITOR, VIEWER     | GqlRequestTeamMemberGuard    | 通过 requestID → teamID 验证          |
| 查看集合内请求 | OWNER, EDITOR, VIEWER     | GqlCollectionTeamMemberGuard | 通过 collectionID → teamID 验证       |
| **创建请求**   | OWNER, EDITOR             | **GqlCollectionTeamMemberGuard** | ⚠️ 创建时无 requestID，通过 collectionID 验证 |
| 更新请求       | OWNER, EDITOR             | GqlRequestTeamMemberGuard    | 通过 requestID → teamID 验证          |
| 删除请求       | OWNER, EDITOR             | GqlRequestTeamMemberGuard    | 通过 requestID → teamID 验证          |
| 移动请求       | OWNER, EDITOR             | GqlRequestTeamMemberGuard    | 通过 requestID → teamID 验证          |

**团队环境 Resolver** [src/team-environments/team-environments.resolver.ts](packages/hoppscotch-backend/src/team-environments/team-environments.resolver.ts)

| 操作           | 所需角色                  | 守卫类型                     | 说明                                  |
|----------------|---------------------------|------------------------------|---------------------------------------|
| **创建环境**   | OWNER, EDITOR             | **GqlTeamMemberGuard**       | ⚠️ 创建时无 environmentID，通过 teamID 直接验证 |
| 更新环境       | OWNER, EDITOR             | GqlTeamEnvTeamGuard          | 通过 environmentID → teamID 验证      |
| 删除环境       | OWNER, EDITOR             | GqlTeamEnvTeamGuard          | 通过 environmentID → teamID 验证      |
| 清空环境变量   | OWNER, EDITOR             | GqlTeamEnvTeamGuard          | 通过 environmentID → teamID 验证      |
| 复制环境       | OWNER, EDITOR             | GqlTeamEnvTeamGuard          | 通过 environmentID → teamID 验证      |

---

### 4.2 REST API 权限配置示例

**团队集合 Controller** [src/team-collection/team-collection.controller.ts](packages/hoppscotch-backend/src/team-collection/team-collection.controller.ts)

| 操作           | 所需角色                  | 守卫类型                     | 权限链路                                  |
|----------------|---------------------------|------------------------------|-------------------------------------------|
| 搜索团队集合   | OWNER, EDITOR, VIEWER     | RESTTeamMemberGuard          | `GET /team-collection/search/:teamID` → URL params.teamID → TeamMember.role |

```typescript
@Get('search/:teamID')
@RequiresTeamRole(TeamAccessRole.VIEWER, TeamAccessRole.EDITOR, TeamAccessRole.OWNER)
@UseGuards(JwtAuthGuard, RESTTeamMemberGuard)
async searchByTitle(
  @Param('teamID') teamID: string,
  @Query('searchQuery') searchQuery: string,
) { ... }
```

---

### 4.3 守卫选择决策树

```
请求到达
   ↓
是否有资源ID（已存在资源）？
   ├─ 是 → 选择对应资源级守卫
   │     ├─ teamID → GqlTeamMemberGuard
   │     ├─ collectionID → GqlCollectionTeamMemberGuard
   │     ├─ requestID → GqlRequestTeamMemberGuard
   │     └─ environmentID → GqlTeamEnvTeamGuard
   └─ 否（创建操作）→ 选择父资源级守卫
         ├─ 创建团队 → 仅需认证（自动成为OWNER）
         ├─ 创建根集合 → GqlTeamMemberGuard（通过teamID）
         ├─ 创建子集合 → GqlCollectionTeamMemberGuard（通过父collectionID）
         ├─ 创建请求 → GqlCollectionTeamMemberGuard（通过collectionID）
         └─ 创建环境 → GqlTeamMemberGuard（通过teamID）
```

## 5. 前端工作区权限感知

### 5.1 工作区模型

[src/services/workspace.service.ts:16-27](packages/hoppscotch-common/src/services/workspace.service.ts#L16-L27)
```typescript
export type PersonalWorkspace = {
  type: "personal"
}

export type TeamWorkspace = {
  type: "team"
  teamID: string
  teamName: string
  role: TeamAccessRole | null | undefined  // 用户在该团队的角色
}
```

### 5.2 后端 myRole 到前端 workspace role 的完整转换链路

#### 第1步：后端 myRole 字段解析

[src/team/team.resolver.ts:67-77](packages/hoppscotch-backend/src/team/team.resolver.ts#L67-L77)

```typescript
@ResolveField(() => TeamAccessRole, {
  description: 'The role of the current user in the team',
  nullable: true,
})
@UseGuards(GqlAuthGuard)
myRole(
  @Parent() team: Team,
  @GqlUser() user: AuthUser,
): Promise<TeamAccessRole | null> {
  return this.teamService.getRoleOfUserInTeam(team.id, user.uid);
}
```

**实现逻辑**：`myRole` 是 Team 类型的一个动态字段 resolver，根据当前登录用户和团队ID，调用 `getRoleOfUserInTeam` 查询用户在该团队的角色。

#### 第2步：GraphQL 查询包含 myRole

[src/helpers/backend/gql/queries/GetMyTeams.graphql:1-18](packages/hoppscotch-common/src/helpers/backend/gql/queries/GetMyTeams.graphql#L1-L18)

```graphql
query GetMyTeams($cursor: ID) {
  myTeams(cursor: $cursor) {
    id
    name
    myRole          # 关键：请求包含当前用户的角色
    ownersCount
    teamMembers {
      membershipID
      user { ... }
      role
    }
  }
}
```

#### 第3步：前端 TeamListAdapter 拉取团队列表

[src/helpers/teams/TeamListAdapter.ts:60-98](packages/hoppscotch-common/src/helpers/teams/TeamListAdapter.ts#L60-L98)

```typescript
async fetchList() {
  const results: GetMyTeamsQuery["myTeams"] = []
  
  while (true) {
    const cursor = results.length > 0 ? results[results.length - 1].id : undefined
    
    // 调用后端 GraphQL 接口，获取包含 myRole 的团队列表
    const result = await platform.backend.getUserTeams(cursor)
    
    if (E.isLeft(result)) { /* 错误处理 */ }
    
    results.push(...result.right.myTeams)
    
    if (result.right.myTeams.length !== BACKEND_PAGE_SIZE) break
  }
  
  // 通过 BehaviorSubject 广播团队列表（包含每个团队的 myRole）
  this.teamList$.next(results)
}
```

#### 第4步：工作区选择器消费团队列表并设置 role

[src/components/workspace/Selector.vue:169-177](packages/hoppscotch-common/src/components/workspace/Selector.vue#L169-L177)

```typescript
const switchToTeamWorkspace = (team: GetMyTeamsQuery["myTeams"][number]) => {
  REMEMBERED_TEAM_ID.value = team.id
  
  // 关键：将后端返回的 team.myRole 赋值给 workspace.role
  workspaceService.changeWorkspace({
    teamID: team.id,
    teamName: team.name,
    type: "team",
    role: team.myRole,  // myRole → workspace.role 的转换点
  })
}
```

#### 第5步：WorkspaceService 保存 role 并联动其他服务

[src/services/workspace.service.ts:123-162](packages/hoppscotch-common/src/services/workspace.service.ts#L123-L162)

```typescript
private setupWorkspaceSync() {
  watch(
    [this._currentWorkspace, this.currentUser],
    async ([newWorkspace, user], [oldWorkspace, oldUser]) => {
      if (newWorkspace?.type === "team" && newWorkspace.teamID) {
        // 切换到团队工作区时，同步 teamID 到集合服务
        this.teamCollectionService.changeTeamID(newWorkspace.teamID)
        
        // 拉取该团队的文档数据
        await this.documentationService.fetchTeamPublishedDocs(newWorkspace.teamID)
      }
    },
    { immediate: true }
  )
}
```

#### 完整转换流程图

```
后端 TeamResolver.myRole
    ↓ (GraphQL)
GetMyTeams 查询包含 myRole 字段
    ↓ (HTTP)
TeamListAdapter.fetchList() 获取团队列表
    ↓ (RxJS)
teamList$.next(results) 广播团队数据（含 myRole）
    ↓ (Vue watch)
workspace Selector 显示团队列表供用户选择
    ↓ (用户点击)
switchToTeamWorkspace(team) 被调用
    ↓
workspaceService.changeWorkspace({
  teamID: team.id,
  teamName: team.name,
  type: "team",
  role: team.myRole  // 转换完成
})
    ↓
setupWorkspaceSync() 触发联动
    → teamCollectionService.changeTeamID(teamID)
    → documentationService.fetchTeamPublishedDocs(teamID)
    → 前端各组件根据 workspace.role 控制 UI 权限
```

---

### 5.3 前端权限控制应用示例

前端通过 `workspace.role` 控制 UI 元素的显示与隐藏：

```typescript
// 示例：根据角色判断是否显示编辑按钮
const canEdit = computed(() => {
  const role = workspaceService.currentWorkspace.value.role
  return role === TeamAccessRole.OWNER || role === TeamAccessRole.EDITOR
})

// 示例：根据角色判断是否显示成员管理入口
const canManageMembers = computed(() => {
  const role = workspaceService.currentWorkspace.value.role
  return role === TeamAccessRole.OWNER
})
```

**重要提示**：前端 UI 控制仅为体验优化，**真实权限校验始终在服务端通过守卫执行**。

## 6. 架构特点与设计思考

### 6.1 优点

1. **统一权限模型**：所有资源的权限最终都派生自团队成员角色，避免了复杂的细粒度权限管理

2. **守卫分层设计**：不同资源级别使用专门的守卫，职责清晰，易于维护

3. **声明式权限**：通过装饰器 `@RequiresTeamRole` 声明权限要求，代码可读性高

4. **运行时验证**：所有权限检查在服务端执行，前端仅做 UI 控制，安全性有保障

5. **级联删除**：数据库层面配置 `onDelete: Cascade`，保证数据一致性

### 6.2 潜在优化点

1. **权限缓存**：频繁调用 `getTeamMember` 可能导致重复查询，可考虑在守卫层增加缓存

2. **角色继承**：当前实现需要显式列出所有允许的角色（如 `OWNER, EDITOR, VIEWER`），可考虑角色继承机制（OWNER 自动拥有 EDITOR 和 VIEWER 权限）

3. **操作日志**：权限敏感操作缺乏审计日志记录

4. **批量权限检查**：对于列表查询场景，目前是逐条检查，可优化为批量预检查

## 7. 总结：权限传播完整链路

### 7.1 后端权限校验全链路

```
用户登录 → 访问资源（GraphQL/REST）
    ↓
提取资源标识符 → 判定资源存在性 → 定位所属团队
    ↓
查询用户在该团队的角色（TeamMember.role）
    ↓
对比 @RequiresTeamRole 声明的角色要求 → 允许/拒绝访问
```

### 7.2 创建操作的特殊链路

```
创建请求 → 提取 collectionID（父资源）
    ↓
查询 TeamCollection.teamID → 定位团队
    ↓
查询用户角色 → 验证 EDITOR/OWNER 权限
    ↓
允许创建请求

创建环境 → 提取 teamID（直接参数）
    ↓
查询用户角色 → 验证 EDITOR/OWNER 权限
    ↓
允许创建环境
```

### 7.3 前端角色感知全链路

```
后端 myRole resolver → GraphQL GetMyTeams 查询
    ↓
TeamListAdapter 拉取团队列表（含 myRole）
    ↓
用户选择团队 → switchToTeamWorkspace(team)
    ↓
workspace.role = team.myRole → 保存到 WorkspaceService
    ↓
前端组件根据 workspace.role 控制 UI 显示
```

---

### 7.4 关键洞察（修正版）

1. **创建操作的守卫选择是关键**：资源创建时目标资源ID不存在，必须通过**父资源ID**或**直接teamID**进行权限验证：
   - 创建请求 → 使用 `GqlCollectionTeamMemberGuard`（通过 collectionID）
   - 创建环境 → 使用 `GqlTeamMemberGuard`（通过 teamID）

2. **所有团队资源的权限都派生自团队角色**：权限并非在资源层面独立设置，而是通过"资源 → teamID → TeamMember.role"的关联链隐式向下传播。

3. **协议层的守卫差异**：
   - GraphQL 接口使用 `Gql*TeamMemberGuard` 系列，从 `gqlExecCtx.getArgs()` 提取参数
   - REST 接口使用 `RESTTeamMemberGuard`，从 `request.params` 提取 URL 参数

4. **myRole 是动态计算字段**：前端获得的角色信息并非直接存储的字段，而是后端根据当前登录用户动态计算的 resolver 结果。

5. **前端权限控制仅为体验优化**：即使前端隐藏了某些按钮，后端守卫仍会对每个请求进行权限校验，确保安全性。

6. **这种设计的局限性**：由于权限完全派生自团队角色，系统**不支持对单个集合、请求或环境设置独立的访问权限**。所有同团队内的资源权限级别一致。
