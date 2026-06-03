# RBAC 权限模块概述

## 概述

RBAC（基于角色的访问控制）模块提供完整的用户、角色、权限和菜单管理功能，实现细粒度的权限控制和灵活的访问策略管理。

**模块路径**：`module/rbac/`

## 功能说明

### 用户管理
- 用户账号的 CRUD 操作
- 用户与角色的关联管理
- 用户与岗位的关联管理
- 用户状态控制

### 角色管理
- 角色的 CRUD 操作
- 角色与菜单的关联管理
- 角色数据范围控制
- 角色状态管理

### 权限管理
- 基于菜单的权限定义
- 权限与角色的关联
- 数据权限控制

### 菜单管理
- 树形菜单结构
- 菜单类型（目录、菜单、按钮/组件）
- 动态路由生成
- 多菜单源支持（Ruoyi、Pure）

## 主要服务

### UserService
用户管理服务，继承自 `YiCrudAppService`。

**关键方法**：
- `GetListAsync(UserGetListInput input)` - 获取用户列表
- `CreateAsync(UserCreateInput input)` - 创建用户
- `UpdateAsync(UserUpdateInput input)` - 更新用户
- `DeleteAsync(Guid id)` - 删除用户

### UserManager
用户管理器，处理用户相关的业务逻辑。

**关键方法**：
- `GiveUserSetRoleAsync(List<Guid> userIds, List<Guid> roleIds)` - 给用户分配角色
- `GiveUserSetPostAsync(List<Guid> userIds, List<Guid> postIds)` - 给用户分配岗位
- `SetPasswordAsync(...)` - 设置用户密码
- `GetUserAllInfoAsync(Guid userId)` - 获取用户完整信息

### RoleService
角色管理服务。

**关键方法**：
- `GetListAsync(RoleGetListInput input)` - 获取角色列表
- `CreateAsync(RoleCreateInput input)` - 创建角色
- `UpdateDataScopeAsync(UpdateDataScopeInput input)` - 更新数据范围

### RoleManager
角色管理器。

**关键方法**：
- `GiveRoleSetMenuAsync(List<Guid> roleIds, List<Guid> menuIds)` - 给角色分配菜单

### MenuService
菜单管理服务。

**关键方法**：
- `GetListAsync(MenuGetListInput input)` - 获取菜单列表
- `GetListRoleIdAsync(Guid roleId)` - 获取角色的菜单列表
- `CreateAsync(MenuCreateInput input)` - 创建菜单
- `UpdateAsync(MenuUpdateInput input)` - 更新菜单

### AccountService
账号服务，处理登录、登出和权限验证。

**关键方法**：
- `LoginAsync(LoginInput input)` - 用户登录
- `LogoutAsync()` - 用户登出
- `GetAsync()` - 获取当前用户信息
- `GetUserInfoAsync()` - 获取用户权限信息

## 权限模型

### 权限层级

```
用户 (User)
  └── 角色 (Role) [多对多]
        └── 菜单 (Menu) [多对多]
              └── 权限码 (PermissionCode)
```

### 数据权限

角色支持不同的数据范围：

| 数据范围 | 说明 |
|---------|------|
| `ALL` | 全部数据权限 |
| `CUSTOM` | 自定义数据权限（通过部门关联） |
| `DEPT` | 部门数据权限 |
| `DEPT_AND_CHILD` | 部门及以下数据权限 |
| `SELF` | 仅本人数据权限 |

### 部门权限

角色可以关联特定部门，实现更细粒度的数据权限控制：

```csharp
// 角色关联部门
await _roleManager.UpdateDataScopeAsync(new UpdateDataScopeInput
{
    RoleId = roleId,
    DataScope = DataScopeEnum.CUSTOM,
    DeptIds = new List<Guid> { deptId1, deptId2 }
});
```

## 数据结构

### UserAggregateRoot
用户实体。

**主要字段**：
| 字段 | 类型 | 说明 |
|------|------|------|
| Id | Guid | 主键 |
| UserName | string | 用户名（登录名） |
| Name | string? | 真实姓名 |
| Nick | string? | 昵称 |
| EncryPassword | EncryPasswordValueObject | 加密密码（含盐值） |
| Phone | long? | 电话 |
| Email | string? | 邮箱 |
| DeptId | Guid? | 部门 ID |
| State | bool | 状态（true=启用） |
| OrderNum | int | 排序 |

**审计字段**：
- `CreationTime`, `CreatorId`
- `LastModificationTime`, `LastModifierId`
- `IsDeleted`（软删除）

**导航属性**：
- `Roles` - 用户角色列表
- `Posts` - 用户岗位列表
- `Dept` - 所属部门

### RoleAggregateRoot
角色实体。

**主要字段**：
| 字段 | 类型 | 说明 |
|------|------|------|
| Id | Guid | 主键 |
| RoleName | string | 角色名称 |
| RoleCode | string | 角色编码 |
| Remark | string? | 备注 |
| DataScope | DataScopeEnum | 数据范围 |
| State | bool | 状态 |
| OrderNum | int | 排序 |

**导航属性**：
- `Menus` - 角色菜单列表
- `Depts` - 角色部门列表（用于自定义数据权限）

### MenuAggregateRoot
菜单实体。

**主要字段**：
| 字段 | 类型 | 说明 |
|------|------|------|
| Id | Guid | 主键 |
| MenuName | string | 菜单名称 |
| MenuType | MenuTypeEnum | 菜单类型 |
| RouterName | string? | 路由名称 |
| Router | string? | 路由路径 |
| Component | string? | 组件路径 |
| PermissionCode | string? | 权限码 |
| ParentId | Guid | 父菜单 ID |
| MenuIcon | string? | 菜单图标 |
| OrderNum | int | 排序 |
| State | bool | 状态 |
| IsLink | bool | 是否外部链接 |
| IsCache | bool | 是否缓存 |
| IsShow | bool | 是否显示 |
| MenuSource | MenuSourceEnum | 菜单来源 |

**菜单类型**：
- `MenuTypeEnum.Catalogue` - 目录
- `MenuTypeEnum.Menu` - 菜单
- `MenuTypeEnum.Component` - 按钮/组件

**菜单来源**：
- `MenuSourceEnum.Ruoyi` - RuoYi 风格
- `MenuSourceEnum.Pure` - PureAdmin 风格

### 关联表

#### UserRoleEntity
用户角色关联表。

```csharp
public class UserRoleEntity
{
    public Guid UserId { get; set; }
    public Guid RoleId { get; set; }
}
```

#### RoleMenuEntity
角色菜单关联表。

```csharp
public class RoleMenuEntity
{
    public Guid RoleId { get; set; }
    public Guid MenuId { get; set; }
}
```

#### RoleDeptEntity
角色部门关联表（用于自定义数据权限）。

```csharp
public class RoleDeptEntity
{
    public Guid RoleId { get; set; }
    public Guid DeptId { get; set; }
}
```

## 用户权限验证

### 登录流程
```csharp
// 1. 用户登录
var loginResult = await _accountService.LoginAsync(new LoginInput
{
    UserName = "admin",
    Password = "123456"
});

// 2. 返回 Token
// { accessToken: "...", expiresIn: 3600 }
```

### 权限验证
```csharp
// 获取用户权限信息
var userInfo = await _accountService.GetUserInfoAsync();

// 返回结构
{
    "userId": "...",
    "userName": "admin",
    "roles": ["admin"],
    "permissions": {
        "user:add": true,
        "user:edit": true,
        // ...
    },
    "menus": [
        // 前端路由菜单
    ]
}
```

### 前端路由生成
系统根据用户角色的菜单权限动态生成前端路由：

```csharp
// RuoYi 风格路由
var ruoyiRouters = menus.Vue3RuoYiRouterBuild();

// Pure 风格路由
var pureRouters = menus.Vue3PureRouterBuild();
```

## 相关文档

- [[ABP 框架集成]] - ABP 权限系统说明
- [[JWT 认证]] - Token 管理说明
- [[菜单管理]] - 菜单详细说明

## 注意事项

1. **密码安全**：密码使用 SHA-256 加盐加密存储
2. **软删除**：用户、角色、菜单支持软删除，查询时自动过滤
3. **级联操作**：删除角色时需要清理关联关系
4. **数据权限**：数据权限过滤在业务层实现
5. **缓存策略**：用户信息和权限会被缓存，修改后需要清除
