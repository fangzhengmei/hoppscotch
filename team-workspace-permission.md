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

| 守卫类                     | 资源级别 | 提取参数   | 定位方式                     | 文件                                                                 |
|--------------------------|----------|------------|------------------------------|----------------------------------------------------------------------|
| GqlTeamMemberGuard       | 团队级   | teamID     | 直接使用 teamID              | [src/team/guards/gql-team-member.guard.ts](packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts) |
| GqlCollectionTeamMemberGuard | 集合级 | collectionID | 通过 collection → teamID     | [src/team-collection/guards/gql-collection-team-member.guard.ts](packages/hoppscotch-backend/src/team-collection/guards/gql-collection-team-member.guard.ts) |
| GqlRequestTeamMemberGuard | 请求级   | requestID  | 通过 request → teamID        | [src/team-request/guards/gql-request-team-member.guard.ts](packages/hoppscotch-backend/src/team-request/guards/gql-request-team-member.guard.ts) |
| GqlTeamEnvTeamGuard      | 环境级   | id         | 通过 environment → teamID    | [src/team-environments/gql-team-env-team.guard.ts](packages/hoppscotch-backend/src/team-environments/gql-team-env-team.guard.ts) |

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
- 创建/编辑环境变量
- 删除环境
- 复制环境

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

**团队环境 Resolver** [src/team-environments/team-environments.resolver.ts](packages/hoppscotch-backend/src/team-environments/team-environments.resolver.ts)

| 操作           | 所需角色                  | 守卫类型                     |
|----------------|---------------------------|------------------------------|
| 创建环境       | OWNER, EDITOR             | GqlTeamMemberGuard           |
| 更新环境       | OWNER, EDITOR             | GqlTeamEnvTeamGuard          |
| 删除环境       | OWNER, EDITOR             | GqlTeamEnvTeamGuard          |

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

### 5.2 工作区切换与权限联动

[src/services/workspace.service.ts:123-162](packages/hoppscotch-common/src/services/workspace.service.ts#L123-L162)

当切换到团队工作区时：
1. 同步团队集合服务的 teamID
2. 拉取该团队的集合和请求数据
3. 前端根据 `role` 字段控制 UI 元素的显示/隐藏

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

```
用户登录 → 选择团队工作区 → 访问资源（团队/集合/请求/环境）
    ↓
提取资源ID → 定位所属团队 → 查询用户在该团队的角色
    ↓
对比装饰器声明的角色要求 → 允许/拒绝访问
```

**关键洞察**：
- 所有团队资源的权限都**派生自用户在团队中的角色**，而非资源本身的独立权限
- 不同层级的守卫通过"资源 → 团队"的关联关系，实现了权限的隐式向下传播
- 这种设计简化了权限管理，但也意味着无法对单个集合或请求设置独立权限
