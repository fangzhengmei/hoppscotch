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

**真实使用示例** [src/team/team.resolver.ts:133-135](packages/hoppscotch-backend/src/team/team.resolver.ts#L133-L135)：
```typescript
@RequiresTeamRole(
  TeamAccessRole.VIEWER,
  TeamAccessRole.EDITOR,
  TeamAccessRole.OWNER,
)
```

**另一示例** [src/team/team.resolver.ts:219](packages/hoppscotch-backend/src/team/team.resolver.ts#L219)：
```typescript
@RequiresTeamRole(TeamAccessRole.OWNER)
async removeTeamMember(...)
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
  const requireRoles = this.reflector.get<TeamAccessRole[]>(
    'requiresTeamRole',
    context.getHandler(),
  );
  if (!requireRoles) throw new Error(BUG_TEAM_NO_REQUIRE_TEAM_ROLE);

  const gqlExecCtx = GqlExecutionContext.create(context);
  const { req, headers } = gqlExecCtx.getContext();
  const user = headers ? headers.user : req.user;

  if (user == undefined) throw new Error(BUG_AUTH_NO_USER_CTX);

  const { teamID } = gqlExecCtx.getArgs<{ teamID: string }>();
  if (!teamID) throw new Error(BUG_TEAM_NO_TEAM_ID);

  const teamMember = await this.teamService.getTeamMember(teamID, user.uid);
  if (!teamMember) throw new Error(TEAM_MEMBER_NOT_FOUND);

  if (requireRoles.includes(teamMember.role)) return true;

  throw new Error(TEAM_NOT_REQUIRED_ROLE);
}
```

**典型应用场景**：
- 查看团队详情
- 重命名团队
- 删除团队
- 管理成员角色

---

### 3.3.1 守卫授权失败路径的语义差异

**重要发现**：所有守卫**没有使用 `return false`**，全部采用 `throw` 方式拒绝访问。但不同守卫使用了三种不同的 throw 方式，语义和排错影响不同：

| 守卫类                     | 错误抛出方式               | 错误类型        | 语义差异                                  | 排错影响                                  |
|--------------------------|----------------------------|-----------------|-------------------------------------------|-------------------------------------------|
| GqlTeamMemberGuard       | `throw new Error(msg)`     | Error           | 直接抛出通用错误                          | 需要从 error.message 解析错误码           |
| GqlCollectionTeamMemberGuard | `throw new Error(msg)` | Error           | 直接抛出通用错误                          | 需要从 error.message 解析错误码           |
| GqlRequestTeamMemberGuard | `throwErr(msg)` (部分场景) | Error           | 本质也是 `throw new Error`，工具函数封装  | 同上，但部分路径使用 `throwErr`           |
| GqlTeamEnvTeamGuard      | `throwErr(msg)`            | Error           | 全部使用工具函数封装                      | 同上                                      |
| RESTTeamMemberGuard      | `throwHTTPErr({msg, code})`| HttpException   | 抛出带状态码的 HTTP 异常                  | 直接返回带状态码的 HTTP 响应，前端可直接识别 |

**工具函数实现** [src/utils.ts:44-55](packages/hoppscotch-backend/src/utils.ts#L44-L55)
```typescript
// 作为表达式使用的 throw 封装
export function throwErr(errMessage: string): never {
  throw new Error(errMessage);
}

// REST 专用：抛出带 HTTP 状态码的异常
export function throwHTTPErr(errorData: RESTError): never {
  const { message, statusCode } = errorData;
  throw new HttpException(message, statusCode);
}
```

**对排错和复核的影响**：

1. **GraphQL 守卫**（前四个）：
   - 错误被 NestJS GraphQL 模块捕获后，会包装成 GraphQL errors 数组中的 `message` 字段
   - 排错时需要从 `errors[0].message` 中提取错误码（如 `team/not_required_role`）
   - 由于是统一的 `Error` 类型，无法通过 `instanceof` 判断错误类别
   - 复核时需要对比错误码与 `src/errors.ts` 中的定义

2. **REST 守卫**（RESTTeamMemberGuard）：
   - 直接抛出 `HttpException`，NestJS 会转换为标准 HTTP 响应
   - 状态码语义明确：400（参数错误）、403（权限不足）、404（资源不存在）
   - 排错时直接看 HTTP 状态码和响应体即可
   - 复核时可直接通过状态码快速定位问题类别

3. **错误码分类**（来自 [src/errors.ts](packages/hoppscotch-backend/src/errors.ts)）：
   - `BUG_*` 前缀：表示代码 bug（如缺少装饰器、缺少参数）
   - `TEAM_*` 前缀：表示业务错误（如成员不存在、角色不足）

**示例错误码**：
- `BUG_TEAM_NO_REQUIRE_TEAM_ROLE` - 代码错误：缺少 @RequiresTeamRole 装饰器
- `TEAM_MEMBER_NOT_FOUND` - 业务错误：用户不是该团队成员
- `TEAM_NOT_REQUIRED_ROLE` - 业务错误：用户角色不满足操作要求

---

### 3.4 集合级守卫实现（间接权限）

[src/team-collection/guards/gql-collection-team-member.guard.ts:24-50](packages/hoppscotch-backend/src/team-collection/guards/gql-collection-team-member.guard.ts#L24-L50)

```typescript
async canActivate(context: ExecutionContext): Promise<boolean> {
  const requireRoles = this.reflector.get<TeamAccessRole[]>(
    'requiresTeamRole',
    context.getHandler(),
  );
  if (!requireRoles) throw new Error(BUG_TEAM_NO_REQUIRE_TEAM_ROLE);

  const gqlExecCtx = GqlExecutionContext.create(context);

  const { user } = gqlExecCtx.getContext().req;
  if (user == undefined) throw new Error(BUG_AUTH_NO_USER_CTX);

  const { collectionID } = gqlExecCtx.getArgs<{ collectionID: string }>();
  if (!collectionID) throw new Error(BUG_TEAM_COLL_NO_COLL_ID);

  const collection =
    await this.teamCollectionService.getCollection(collectionID);
  if (E.isLeft(collection)) throw new Error(TEAM_INVALID_COLL_ID);

  const member = await this.teamService.getTeamMember(
    collection.right.teamID,
    user.uid,
  );
  if (!member) throw new Error(TEAM_REQ_NOT_MEMBER);

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

```typescript
async canActivate(context: ExecutionContext): Promise<boolean> {
  const requireRoles = this.reflector.get<TeamAccessRole[]>(
    'requiresTeamRole',
    context.getHandler(),
  );

  const gqlExecCtx = GqlExecutionContext.create(context);

  const { user } = gqlExecCtx.getContext().req;
  if (!user) throw new Error(BUG_AUTH_NO_USER_CTX);

  const { requestID } = gqlExecCtx.getArgs<{ requestID: string }>();
  if (!requestID) throw new Error(BUG_TEAM_REQ_NO_REQ_ID);

  const team =
    await this.teamRequestService.getTeamOfRequestFromID(requestID);
  if (O.isNone(team)) throw new Error(TEAM_REQ_NOT_FOUND);

  const member = await this.teamService.getTeamMember(
    team.value.id,
    user.uid,
  );
  if (!member) throwErr(TEAM_REQ_NOT_MEMBER);

  if (!(requireRoles && requireRoles.includes(member.role)))
    throw new Error(TEAM_REQ_NOT_REQUIRED_ROLE);

  return true;
}
```

**权限派生链路**：
```
requestID → TeamRequest.teamID → TeamMember.role → 权限验证
```

**典型应用场景**：
- 编辑请求
- 删除请求
- 移动请求到其他集合

---

### 3.6 环境级守卫实现

[src/team-environments/gql-team-env-team.guard.ts:30-56](packages/hoppscotch-backend/src/team-environments/gql-team-env-team.guard.ts#L30-L56)

```typescript
async canActivate(context: ExecutionContext): Promise<boolean> {
  const requireRoles = this.reflector.get<TeamAccessRole[]>(
    'requiresTeamRole',
    context.getHandler(),
  );
  if (!requireRoles) throw new Error(BUG_TEAM_ENV_GUARD_NO_REQUIRE_ROLES);

  const gqlExecCtx = GqlExecutionContext.create(context);

  const { user } = gqlExecCtx.getContext().req;
  if (user == undefined) throw new Error(BUG_AUTH_NO_USER_CTX);

  const { id } = gqlExecCtx.getArgs<{ id: string }>();
  if (!id) throwErr(BUG_TEAM_ENV_GUARD_NO_ENV_ID);

  const teamEnvironment =
    await this.teamEnvironmentService.getTeamEnvironment(id);
  if (E.isLeft(teamEnvironment)) throwErr(TEAM_ENVIRONMENT_NOT_FOUND);

  const member = await this.teamService.getTeamMember(
    teamEnvironment.right.teamID,
    user.uid,
  );
  if (!member) throwErr(TEAM_ENVIRONMENT_NOT_TEAM_MEMBER);

  return requireRoles.includes(member.role);
}
```

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
  const requireRoles = this.reflector.get<TeamAccessRole[]>(
    'requiresTeamRole',
    context.getHandler(),
  );
  if (!requireRoles)
    throwHTTPErr({ message: BUG_TEAM_NO_REQUIRE_TEAM_ROLE, statusCode: 400 });

  const request = context.switchToHttp().getRequest();

  const { user } = request;
  if (user == undefined)
    throwHTTPErr({ message: BUG_AUTH_NO_USER_CTX, statusCode: 400 });

  const teamID = request.params.teamID;
  if (!teamID)
    throwHTTPErr({ message: BUG_TEAM_NO_TEAM_ID, statusCode: 400 });

  const teamMember = await this.teamService.getTeamMember(teamID, user.uid);
  if (!teamMember)
    throwHTTPErr({ message: TEAM_MEMBER_NOT_FOUND, statusCode: 404 });

  if (requireRoles.includes(teamMember.role)) return true;

  throwHTTPErr({ message: TEAM_NOT_REQUIRED_ROLE, statusCode: 403 });
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
@Mutation(() => TeamRequest, {
  description: 'Create a team request in the given collection.',
})
@UseGuards(GqlAuthGuard, GqlCollectionTeamMemberGuard)  // 注意这里用的是集合级守卫
@RequiresTeamRole(TeamAccessRole.EDITOR, TeamAccessRole.OWNER)
async createRequestInCollection(
  @Args({ name: 'collectionID', type: () => ID }) collectionID: string,
  @Args({ name: 'data', type: () => CreateTeamRequestInput }) data: CreateTeamRequestInput,
) {
  const teamRequest = await this.teamRequestService.createTeamRequest(
    collectionID, data.teamID, data.title, data.request,
  );
  if (E.isLeft(teamRequest)) throwErr(teamRequest.left);
  return teamRequest.right;
}
```

#### 创建环境（createTeamEnvironment）
- **使用的守卫**：`GqlTeamMemberGuard`（不是 `GqlTeamEnvTeamGuard`）
- **原因**：创建时 environmentID 不存在，需要通过 `teamID` 直接验证
- **权限链路**：`teamID → TeamMember.role`

[src/team-environments/team-environments.resolver.ts:30-47](packages/hoppscotch-backend/src/team-environments/team-environments.resolver.ts#L30-L47)
```typescript
@Mutation(() => TeamEnvironment, {
  description: 'Create a new Team Environment for given Team ID',
})
@UseGuards(GqlAuthGuard, GqlTeamMemberGuard)  // 注意这里用的是团队级守卫
@RequiresTeamRole(TeamAccessRole.OWNER, TeamAccessRole.EDITOR)
async createTeamEnvironment(
  @Args() args: CreateTeamEnvironmentArgs,
): Promise<TeamEnvironment> {
  const teamEnvironment =
    await this.teamEnvironmentsService.createTeamEnvironment(
      args.name, args.teamID, args.variables,
    );
  if (E.isLeft(teamEnvironment)) throwErr(teamEnvironment.left);
  return teamEnvironment.right;
}
```

## 4. 资源与权限映射

### 4.1 GraphQL Resolver 权限配置示例

**团队集合 Resolver** [src/team-collection/team-collection.resolver.ts](packages/hoppscotch-backend/src/team-collection/team-collection.resolver.ts)

| 操作           | 所需角色                  | 守卫类型                     | 位置 |
|----------------|---------------------------|------------------------------|------|
| 查看根集合列表 | OWNER, EDITOR, VIEWER     | GqlTeamMemberGuard           | [Line 136](packages/hoppscotch-backend/src/team-collection/team-collection.resolver.ts#L136) |
| 查看单个集合   | OWNER, EDITOR, VIEWER     | GqlCollectionTeamMemberGuard | [Line 154](packages/hoppscotch-backend/src/team-collection/team-collection.resolver.ts#L154) |
| 导出所有集合   | OWNER, EDITOR, VIEWER     | GqlTeamMemberGuard           | [Line 86](packages/hoppscotch-backend/src/team-collection/team-collection.resolver.ts#L86) |
| 导出单个集合   | OWNER, EDITOR, VIEWER     | GqlTeamMemberGuard           | [Line 107](packages/hoppscotch-backend/src/team-collection/team-collection.resolver.ts#L107) |
| **创建根集合** | OWNER, EDITOR             | GqlTeamMemberGuard           | [Line 187](packages/hoppscotch-backend/src/team-collection/team-collection.resolver.ts#L187) |
| **创建子集合** | OWNER, EDITOR             | GqlCollectionTeamMemberGuard | [Line 240](packages/hoppscotch-backend/src/team-collection/team-collection.resolver.ts#L240) |
| 导入集合       | OWNER, EDITOR             | GqlTeamMemberGuard           | [Line 204](packages/hoppscotch-backend/src/team-collection/team-collection.resolver.ts#L204) |
| 重命名集合     | OWNER, EDITOR             | GqlCollectionTeamMemberGuard | [Line 263](packages/hoppscotch-backend/src/team-collection/team-collection.resolver.ts#L263) |
| 删除集合       | OWNER, EDITOR             | GqlCollectionTeamMemberGuard | [Line 279](packages/hoppscotch-backend/src/team-collection/team-collection.resolver.ts#L279) |
| 移动集合       | OWNER, EDITOR             | GqlCollectionTeamMemberGuard | [Line 300](packages/hoppscotch-backend/src/team-collection/team-collection.resolver.ts#L300) |
| 更新集合排序   | OWNER, EDITOR             | GqlCollectionTeamMemberGuard | [Line 314](packages/hoppscotch-backend/src/team-collection/team-collection.resolver.ts#L314) |
| 更新集合详情   | OWNER, EDITOR             | GqlCollectionTeamMemberGuard | [Line 328](packages/hoppscotch-backend/src/team-collection/team-collection.resolver.ts#L328) |
| 复制集合       | OWNER, EDITOR             | GqlCollectionTeamMemberGuard | [Line 345](packages/hoppscotch-backend/src/team-collection/team-collection.resolver.ts#L345) |
| 监听集合新增   | OWNER, EDITOR, VIEWER     | GqlTeamMemberGuard           | [Line 375](packages/hoppscotch-backend/src/team-collection/team-collection.resolver.ts#L375) |
| 监听集合更新   | OWNER, EDITOR, VIEWER     | GqlTeamMemberGuard           | [Line 397](packages/hoppscotch-backend/src/team-collection/team-collection.resolver.ts#L397) |
| 监听集合删除   | OWNER, EDITOR, VIEWER     | GqlTeamMemberGuard           | [Line 419](packages/hoppscotch-backend/src/team-collection/team-collection.resolver.ts#L419) |

**团队 Resolver** [src/team/team.resolver.ts](packages/hoppscotch-backend/src/team/team.resolver.ts)

| 操作           | 所需角色                  | 守卫类型                     | 位置 |
|----------------|---------------------------|------------------------------|------|
| 查看团队详情   | OWNER, EDITOR, VIEWER     | GqlTeamMemberGuard           | [Line 132](packages/hoppscotch-backend/src/team/team.resolver.ts#L132) |
| 查看我的团队   | 登录用户                  | GqlAuthGuard 仅认证          | [Line 113](packages/hoppscotch-backend/src/team/team.resolver.ts#L113) |
| **创建团队**   | 登录用户（自动成为OWNER） | GqlAuthGuard 仅认证          | [Line 186](packages/hoppscotch-backend/src/team/team.resolver.ts#L186) |
| 离开团队       | 登录用户                  | GqlAuthGuard 仅认证          | [Line 200](packages/hoppscotch-backend/src/team/team.resolver.ts#L200) |
| 重命名团队     | OWNER                     | GqlTeamMemberGuard           | [Line 243](packages/hoppscotch-backend/src/team/team.resolver.ts#L243) |
| 删除团队       | OWNER                     | GqlTeamMemberGuard           | [Line 259](packages/hoppscotch-backend/src/team/team.resolver.ts#L259) |
| 更新成员角色   | OWNER                     | GqlTeamMemberGuard           | [Line 273](packages/hoppscotch-backend/src/team/team.resolver.ts#L273) |
| 移除成员       | OWNER                     | GqlTeamMemberGuard           | [Line 218](packages/hoppscotch-backend/src/team/team.resolver.ts#L218) |
| 监听成员新增   | OWNER, EDITOR, VIEWER     | GqlTeamMemberGuard           | [Line 316](packages/hoppscotch-backend/src/team/team.resolver.ts#L316) |
| 监听成员更新   | OWNER, EDITOR, VIEWER     | GqlTeamMemberGuard           | [Line 339](packages/hoppscotch-backend/src/team/team.resolver.ts#L339) |
| 监听成员移除   | OWNER, EDITOR, VIEWER     | GqlTeamMemberGuard           | [Line 362](packages/hoppscotch-backend/src/team/team.resolver.ts#L362) |

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

### 5.1 真实的前端角色消费链路（修正版）

**重要修正**：经过全库搜索验证，`workspace.role` 虽然在类型定义中存在并在切换工作区时被赋值，但**实际上没有任何前端组件直接消费 `workspace.role` 做权限判断**。所有权限 UI 控制都直接使用 `team.myRole` 或 `selectedTeam.myRole`。

#### 类型定义：role 字段存在但未被消费

[src/services/workspace.service.ts:16-27](packages/hoppscotch-common/src/services/workspace.service.ts#L16-L27)
```typescript
export type TeamWorkspace = {
  type: "team"
  teamID: string
  teamName: string
  role: TeamAccessRole | null | undefined  // 定义了但未被消费
}
```

#### 前端角色消费的两条真实路径

---

##### 路径一：团队列表页 → 直接使用 `team.myRole`

**使用场景**：`/profile/teams` 页面展示用户所属团队列表，每个团队卡片根据当前用户角色显示不同的操作按钮。

**文件**：[src/components/teams/Team.vue](packages/hoppscotch-common/src/components/teams/Team.vue)

```typescript
// 直接使用 props.team.myRole，不经过 workspace.role
const props = defineProps<{
  team: GetMyTeamsQuery["myTeams"][number]  // 包含 myRole 字段
  teamID: string
  compact: boolean
}>()
```

**真实消费示例**（来自模板）：
```vue
<!-- 仅 OWNER 显示编辑按钮 -->
<HoppButtonSecondary
  v-if="team.myRole === 'OWNER'"
  :icon="IconEdit"
  @click="$emit('edit-team')"
/>

<!-- 仅 OWNER 显示邀请按钮 -->
<HoppButtonSecondary
  v-if="team.myRole === 'OWNER'"
  :icon="IconUserPlus"
  @click="emit('invite-team')"
/>

<!-- 键盘快捷键也绑定 myRole 判断 -->
<div
  @keyup.e="team.myRole === 'OWNER' ? edit.$el.click() : null"
  @keyup.delete="team.myRole === 'OWNER' ? deleteAction.$el.click() : null"
>
```

**权限判断一览**（Team.vue 中共 11 处直接使用 `team.myRole`）：

| 判断逻辑 | 控制元素 | 位置 |
|---------|---------|------|
| `team.myRole === 'OWNER'` | 编辑按钮显示 | [Line 36](packages/hoppscotch-common/src/components/teams/Team.vue#L36) |
| `team.myRole === 'OWNER'` | 邀请按钮显示 | [Line 47](packages/hoppscotch-common/src/components/teams/Team.vue#L47) |
| `team.myRole === 'OWNER'` | 删除菜单显示 | [Line 88](packages/hoppscotch-common/src/components/teams/Team.vue#L88) |
| `team.myRole === 'OWNER'` | 光标样式 | [Line 26](packages/hoppscotch-common/src/components/teams/Team.vue#L26) |
| `team.myRole === 'OWNER'` | 点击邀请动作 | [Line 17](packages/hoppscotch-common/src/components/teams/Team.vue#L17) |
| `!(team.myRole === 'OWNER' && team.ownersCount == 1)` | 退出按钮显示 | [Line 101](packages/hoppscotch-common/src/components/teams/Team.vue#L101) |

---

##### 路径二：Header 组件 → 直接使用 `selectedTeam.myRole`

**使用场景**：顶部导航栏根据当前选中团队的角色显示不同操作入口。

**文件**：[src/components/app/Header.vue](packages/hoppscotch-common/src/components/app/Header.vue)

**selectedTeam 的来源**：
```typescript
// Line 508: 定义 selectedTeam
const selectedTeam = ref<GetMyTeamsQuery["myTeams"][number] | undefined>()

// Line 511-513: 从 TeamListAdapter 获取团队列表
const teamListAdapter = workspaceService.acquireTeamListAdapter(null)
const myTeams = useReadonlyStream(teamListAdapter.teamList$, null)

// Line 527-555: 通过 watch 从 myTeams 中匹配当前 workspace.teamID
watch(
  () => myTeams.value,
  (newTeams) => {
    const space = workspace.value
    if (newTeams && space.type === "team" && space.teamID) {
      // 根据 workspace.teamID 从团队列表中找到对应的 team 对象
      const team = newTeams.find((team) => team.id === space.teamID)
      if (team) {
        selectedTeam.value = team  // 包含 myRole
      }
    }
  }
)

watch(
  () => workspace.value,
  (newWorkspace) => {
    if (newWorkspace.type === "team") {
      const team = myTeams.value?.find((t) => t.id === newWorkspace.teamID)
      if (team) {
        selectedTeam.value = team
      }
    }
  }
)
```

**真实消费示例**：
```typescript
// 编辑团队入口 - 仅 OWNER
const handleTeamEdit = () => {
  if (
    workspace.value.type === "team" &&
    workspace.value.teamID &&
    selectedTeam.value?.myRole === "OWNER"  // 直接使用 selectedTeam.myRole
  ) {
    editingTeamID.value = workspace.value.teamID
    displayModalEdit(true)
  } else {
    noPermission()
  }
}

// 邀请成员入口 - OWNER 或 EDITOR
defineActionHandler("modals.team.invite", () => {
  if (
    selectedTeam.value?.myRole === "OWNER" ||
    selectedTeam.value?.myRole === "EDITOR"
  ) {
    inviteTeam({ name: selectedTeam.value.name }, selectedTeam.value.id)
  } else {
    noPermission()
  }
})

// 删除团队入口 - 仅 OWNER
defineActionHandler("modals.team.delete", ({ teamId }) => {
  if (selectedTeam.value?.myRole !== TeamAccessRole.Owner) return noPermission()
  teamID.value = teamId
  confirmRemove.value = true
})
```

**权限判断一览**（Header.vue 中共 7 处直接使用 `selectedTeam.myRole`）：

| 判断逻辑 | 控制元素 | 位置 |
|---------|---------|------|
| `selectedTeam?.myRole === 'OWNER'` | 顶部编辑按钮显示 | [Line 186](packages/hoppscotch-common/src/components/app/Header.vue#L186) |
| `selectedTeam.value?.myRole === "OWNER"` | 编辑团队动作 | [Line 599](packages/hoppscotch-common/src/components/app/Header.vue#L599) |
| `selectedTeam.value?.myRole === "OWNER"` | 邀请团队动作 | [Line 585](packages/hoppscotch-common/src/components/app/Header.vue#L585) |
| `selectedTeam.value?.myRole === "OWNER"` \|\| `=== "EDITOR"` | 邀请成员动作 | [Line 639-640](packages/hoppscotch-common/src/components/app/Header.vue#L639-L640) |
| `selectedTeam.value?.myRole !== TeamAccessRole.Owner` | 删除团队动作 | [Line 657](packages/hoppscotch-common/src/components/app/Header.vue#L657) |

---

### 5.2 后端 myRole 到前端 myRole 的完整数据流

#### 第1步：后端 myRole 动态字段解析

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

**实现逻辑**：`myRole` 是 Team 类型的动态字段 resolver，根据当前登录用户和团队ID，查询用户在该团队的角色。

#### 第2步：GraphQL 查询包含 myRole

[src/helpers/backend/gql/queries/GetMyTeams.graphql:1-18](packages/hoppscotch-common/src/helpers/backend/gql/queries/GetMyTeams.graphql#L1-L18)

```graphql
query GetMyTeams($cursor: ID) {
  myTeams(cursor: $cursor) {
    id
    name
    myRole          # 关键：包含当前用户的角色
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
    const result = await platform.backend.getUserTeams(cursor)
    
    if (E.isLeft(result)) { /* 错误处理 */ }
    
    results.push(...result.right.myTeams)  // 每个 team 包含 myRole
    
    if (result.right.myTeams.length !== BACKEND_PAGE_SIZE) break
  }
  
  this.teamList$.next(results)  // 广播团队列表（含 myRole）
}
```

#### 第4步：切换工作区时设置 workspace.role（但未被消费）

[src/components/workspace/Selector.vue:169-177](packages/hoppscotch-common/src/components/workspace/Selector.vue#L169-L177)

```typescript
const switchToTeamWorkspace = (team: GetMyTeamsQuery["myTeams"][number]) => {
  REMEMBERED_TEAM_ID.value = team.id
  
  // role 被设置，但后续没有组件使用 workspace.role
  workspaceService.changeWorkspace({
    teamID: team.id,
    teamName: team.name,
    type: "team",
    role: team.myRole,  // 设置了但未被消费
  })
}
```

#### 第5步：Header 组件通过 workspace.teamID 间接获取角色

```typescript
// Header.vue:527-555
watch(myTeams, (newTeams) => {
  const space = workspace.value
  if (newTeams && space.type === "team" && space.teamID) {
    // 通过 teamID 匹配，间接获得 myRole
    const team = newTeams.find((team) => team.id === space.teamID)
    if (team) selectedTeam.value = team  // selectedTeam 包含 myRole
  }
})
```

#### 完整真实数据流图

```
后端 TeamResolver.myRole (动态字段)
    ↓ (GraphQL)
GetMyTeams 查询返回含 myRole 的团队列表
    ↓ (HTTP)
TeamListAdapter.fetchList() → teamList$.next(results)
    ↓
    ├─ 路径一：Team.vue (团队列表页)
    │     └─ 直接使用 props.team.myRole 控制 UI
    │
    └─ 路径二：Header.vue (顶部导航)
          ├─ watch(myTeams) → 通过 teamID 匹配
          ├─ selectedTeam.value = matchedTeam
          └─ 使用 selectedTeam.myRole 控制 UI

注：workspace.role 被设置但未被任何组件消费，是"死字段"
```

---

### 5.3 其他场景的 myRole 消费

#### 用户删除账户时检查团队所有权

[src/components/profile/UserDelete.vue:163](packages/hoppscotch-common/src/components/profile/UserDelete.vue#L163)
```typescript
// 检查是否有团队是唯一 OWNER，如果有则不能删除账户
const isOnlyOwnerOfATeam = computed(() => {
  return teams.value?.some(
    (team) => team.ownersCount === 1 && team.myRole === "OWNER"
  )
})
```

#### 环境选择器切换时设置 role（同样未被消费）

[src/components/environments/Selector.vue:441](packages/hoppscotch-common/src/components/environments/Selector.vue#L441)
```typescript
// 同样设置了 role，但未被消费
workspaceService.changeWorkspace({
  type: "team",
  teamID: team.id,
  teamName: team.name,
  role: team.myRole,
})
```

---

### 5.4 关键发现总结

1. **`workspace.role` 是死字段**：虽然定义了并在切换时赋值，但全库搜索确认没有任何组件通过 `workspace.role` 或 `currentWorkspace.value.role` 做权限判断。

2. **两种真实消费模式**：
   - **列表模式**（Team.vue）：遍历团队列表时，直接使用每个 `team.myRole`
   - **选中模式**（Header.vue）：通过 `workspace.teamID` 从团队列表中找到 `selectedTeam`，然后使用 `selectedTeam.myRole`

3. **数据单一来源**：所有前端角色信息都来自 `GetMyTeams` 查询返回的 `team.myRole` 字段，这是后端动态计算的结果。

4. **前端权限控制仅为体验优化**：即使前端绕过 UI 限制，后端守卫仍会对每个请求进行权限校验，确保安全性。

## 6. 架构特点与设计思考

### 6.1 优点（均有代码证据支持）

1. **统一权限模型**：所有资源的权限最终都派生自团队成员角色，避免了复杂的细粒度权限管理。代码证据：所有守卫最终都调用 `teamService.getTeamMember(teamID, user.uid)` 获取角色。

2. **守卫分层设计**：不同资源级别使用专门的守卫，职责清晰，易于维护。代码证据：5 个守卫类分别处理团队级、集合级、请求级、环境级、REST 接口的权限验证。

3. **声明式权限**：通过装饰器 `@RequiresTeamRole` 声明权限要求，代码可读性高。代码证据：[src/team/decorators/requires-team-role.decorator.ts:1-5](packages/hoppscotch-backend/src/team/decorators/requires-team-role.decorator.ts#L1-L5)

4. **运行时验证**：所有权限检查在服务端执行，前端仅做 UI 控制，安全性有保障。代码证据：前端 18 处 `myRole` 使用仅用于 UI 显示/隐藏，后端守卫对每个请求独立校验。

5. **级联删除**：数据库层面配置 `onDelete: Cascade`，保证数据一致性。代码证据：[prisma/schema.prisma:48](packages/hoppscotch-backend/prisma/schema.prisma#L48)

### 6.2 基于代码观察的架构特点

1. **`getTeamMember` 高频调用**：`getTeamMember` 方法在 5 个守卫中被独立调用，每次请求至少触发一次数据库查询。代码证据：
   - GqlTeamMemberGuard [Line 37](packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts#L37)
   - GqlCollectionTeamMemberGuard [Line 43](packages/hoppscotch-backend/src/team-collection/guards/gql-collection-team-member.guard.ts#L43)
   - GqlRequestTeamMemberGuard [Line 43](packages/hoppscotch-backend/src/team-request/guards/gql-request-team-member.guard.ts#L43)
   - GqlTeamEnvTeamGuard [Line 49](packages/hoppscotch-backend/src/team-environments/gql-team-env-team.guard.ts#L49)
   - RESTTeamMemberGuard [Line 39](packages/hoppscotch-backend/src/team/guards/rest-team-member.guard.ts#L39)

2. **无角色继承机制**：每个 `@RequiresTeamRole` 装饰器必须显式列出所有允许的角色，没有实现 OWNER 自动包含 EDITOR 权限的继承机制。代码证据：所有 resolver 都显式列出 `TeamAccessRole.VIEWER, TeamAccessRole.EDITOR, TeamAccessRole.OWNER` 三种角色。

3. **无权限缓存机制**：每次请求都独立查询 `TeamMember` 表，没有请求级或应用级的角色缓存。

4. **错误码分类清晰**：错误码分为 `BUG_*`（代码错误）和 `TEAM_*`（业务错误）两类，便于排错。代码证据：[src/errors.ts:513-558](packages/hoppscotch-backend/src/errors.ts#L513-L558)

5. **死字段存在**：`workspace.role` 字段定义并赋值但未被消费，可能是历史遗留代码或未来预留功能。代码证据：[src/services/workspace.service.ts:24](packages/hoppscotch-common/src/services/workspace.service.ts#L24) 定义了 role 字段，但全库无消费代码。

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

### 7.3 前端角色感知全链路（修正版）

```
后端 myRole resolver (动态字段)
    ↓ (GraphQL)
GetMyTeams 查询返回含 myRole 的团队列表
    ↓ (HTTP)
TeamListAdapter.fetchList() → teamList$.next(results)
    ↓
    ├─ 路径一：Team.vue (团队列表页)
    │     └─ 直接使用 props.team.myRole 控制 UI
    │
    └─ 路径二：Header.vue (顶部导航)
          ├─ watch(myTeams) → 通过 workspace.teamID 匹配
          ├─ selectedTeam.value = matchedTeam
          └─ 使用 selectedTeam.myRole 控制 UI

注：workspace.role 被设置但未被消费，是死字段
```

---

### 7.4 关键洞察（最终修正版）

1. **创建操作的守卫选择是关键**：资源创建时目标资源ID不存在，必须通过**父资源ID**或**直接teamID**进行权限验证：
   - 创建请求 → 使用 `GqlCollectionTeamMemberGuard`（通过 collectionID）
   - 创建环境 → 使用 `GqlTeamMemberGuard`（通过 teamID）

2. **所有团队资源的权限都派生自团队角色**：权限并非在资源层面独立设置，而是通过"资源 → teamID → TeamMember.role"的关联链隐式向下传播。

3. **协议层的守卫差异**：
   - GraphQL 接口使用 `Gql*TeamMemberGuard` 系列，从 `gqlExecCtx.getArgs()` 提取参数
   - REST 接口使用 `RESTTeamMemberGuard`，从 `request.params` 提取 URL 参数

4. **所有守卫均使用 throw，无 return false**：
   - GraphQL 守卫：`throw new Error(ERROR_CODE)`，错误码在 `errors[0].message` 中
   - REST 守卫：`throwHTTPErr({ message, statusCode })`，直接返回带状态码的 HTTP 响应
   - 错误码前缀：`BUG_*` 表示代码错误，`TEAM_*` 表示业务错误

5. **myRole 是动态计算字段**：前端获得的角色信息并非直接存储的字段，而是后端根据当前登录用户动态计算的 resolver 结果。

6. **`workspace.role` 是死字段**：虽然类型定义中存在并在切换工作区时赋值，但全库搜索确认没有任何组件通过 `workspace.role` 做权限判断。

7. **前端角色消费的两条真实路径**：
   - **列表模式**（Team.vue）：遍历团队列表时，直接使用每个 `team.myRole`（11处使用）
   - **选中模式**（Header.vue）：通过 `workspace.teamID` 从团队列表中找到 `selectedTeam`，然后使用 `selectedTeam.myRole`（7处使用）

8. **前端权限控制仅为体验优化**：即使前端绕过 UI 限制，后端守卫仍会对每个请求进行权限校验，确保安全性。

9. **这种设计的局限性**：由于权限完全派生自团队角色，系统**不支持对单个集合、请求或环境设置独立的访问权限**。所有同团队内的资源权限级别一致。
