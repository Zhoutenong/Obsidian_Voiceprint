# UserRoleEntity

**关联表：用户-角色多对多关系**

## 概述

`UserRoleEntity` 是用户与角色之间的多对多关联表，用于实现基于角色的访问控制（RBAC）。每个用户可以拥有多个角色，每个角色也可以分配给多个用户。这是权限系统的核心关联表，通过角色继承菜单和数据权限。

## 表信息

| 属性 | 值 |
|------|-----|
| **表名** | `UserRole` |
| **主键** | `Id` (GUID) |
| **复合主键** | 无（使用单主键 Id） |

## 字段说明

| 字段名 | 类型 | 说明 | 外键 |
|--------|------|------|------|
| `Id` | `Guid` | 主键 ID | - |
| `RoleId` | `Guid` | 角色 ID | → `RoleAggregateRoot.Id` |
| `UserId` | `Guid` | 用户 ID | → `UserAggregateRoot.Id` |

## 关联实体 (ER 关系)

```
UserAggregateRoot (1) ←→ (*) RoleAggregateRoot
        │                         │
        └── UserId                RoleId
                 │            │
                 ▼            ▼
           UserRoleEntity
```

**关联端：**

| 导航属性 | 目标实体 | 关联类型 |
|----------|----------|----------|
| `User` | `UserAggregateRoot` | Many-to-One |
| `Role` | `RoleAggregateRoot` | Many-to-One |

## 服务使用

### 写入服务

- **`UserService`** - 用户管理时分配角色
  - `AssignRolesAsync()` - 为用户分配角色
  - `RemoveRoleAsync()` - 移除用户角色
  - `UpdateUserRolesAsync()` - 批量更新用户角色

- **`RoleService`** - 角色管理时查看关联用户
  - `GetUsersByRoleAsync()` - 查询角色下的用户列表

### 读取服务

- **`AccountManager`** - 登录时获取用户角色
- **`RoleManager`** - 权限验证时获取角色列表
- **`UserService`** - 用户详情时包含角色信息

## 业务规则

1. **级联删除**
   - 删除用户时，自动删除对应的角色关联
   - 删除角色时，自动删除对应的用户关联

2. **唯一性**
   - 同一 `(UserId, RoleId)` 组合应唯一
   - 通过业务逻辑层控制

3. **角色继承**
   - 用户通过角色继承菜单权限（`RoleMenuEntity`）
   - 用户通过角色继承数据权限（`RoleDeptEntity`）

4. **权限计算**
   - 用户权限 = 所有角色权限的并集
   - 优先级：用户特定权限 > 角色权限

## 源码位置

```
module/rbac/
└── Yi.Framework.Rbac.Domain/
    └── Entities/
        └── UserRoleEntity.cs
```

## 相关文档

- [UserAggregateRoot](../Aggregates/UserAggregateRoot.md) - 用户聚合根
- [RoleAggregateRoot](../Aggregates/RoleAggregateRoot.md) - 角色聚合根
- [RoleMenuEntity](RoleMenuEntity.md) - 角色-菜单关联表
- [RoleDeptEntity](RoleDeptEntity.md) - 角色-部门关联表
