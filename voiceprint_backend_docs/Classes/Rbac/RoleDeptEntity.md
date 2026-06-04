# RoleDeptEntity

**关联表：角色-部门数据权限关联表**

## 概述

`RoleDeptEntity` 是角色与部门之间的关联表，用于实现基于角色的数据权限控制。通过此表，可以定义角色可以访问哪些部门的数据，支持自定义数据权限范围。这是数据权限系统的核心表，决定了用户能看到哪些部门的数据。

## 表信息

| 属性 | 值 |
|------|-----|
| **表名** | `RoleDept` |
| **主键** | `Id` (GUID) |
| **复合主键** | 无（使用单主键 Id） |

## 字段说明

| 字段名 | 类型 | 说明 | 外键 |
|--------|------|------|------|
| `Id` | `Guid` | 主键 ID | - |
| `RoleId` | `Guid` | 角色 ID | → `RoleAggregateRoot.Id` |
| `DeptId` | `Guid` | 部门 ID | → `DeptAggregateRoot.Id` |

## 关联实体 (ER 关系)

```
RoleAggregateRoot (1) ←→ (*) DeptAggregateRoot
          │                         │
          └── RoleId                DeptId
                   │            │
                   ▼            ▼
             RoleDeptEntity
```

**关联端：**

| 导航属性 | 目标实体 | 关联类型 |
|----------|----------|----------|
| `Role` | `RoleAggregateRoot` | Many-to-One |
| `Dept` | `DeptAggregateRoot` | Many-to-One |

## 服务使用

### 写入服务

- **`RoleService`** - 角色管理时配置数据权限
  - `SetDataScopeAsync()` - 设置角色数据权限范围
  - `AssignDeptsAsync()` - 为角色分配部门权限
  - `RemoveDeptAsync()` - 移除角色部门权限

- **`DeptService`** - 部门管理时查看关联角色
  - `GetRolesByDeptAsync()` - 查询部门关联的角色列表

### 读取服务

- **`RoleService`** - 角色详情时包含数据权限配置
- **`UserService`** - 用户列表数据权限过滤
- **`DataPermissionExtensions` - 数据权限查询时应用过滤

## 业务规则

1. **级联删除**
   - 删除角色时，自动删除对应的部门关联
   - 删除部门时，自动删除对应的角色关联

2. **数据权限范围**
   角色的 `DataScope` 决定如何使用此表：

   | 数据权限类型 | 说明 | RoleDept 用法 |
   |-------------|------|---------------|
   | `全部` | 可查看所有数据 | 不使用 RoleDept |
   | `自定义` | 仅查看指定部门数据 | RoleDept 定义的部门 |
   | `本部门` | 仅查看所属部门 | 根据用户所属部门过滤 |
   | `本部门及以下` | 查看本部门及子部门 | 根据部门树向下过滤 |
   | `仅本人` | 仅查看自己的数据 | 按 UserId 过滤 |

3. **权限继承**
   - 用户数据权限 = 所有角色数据权限的并集
   - 优先级：自定义 > 本部门 > 本部门及以下 > 仅本人

4. **部门树**
   - 部门支持层级结构（父子关系）
   - "本部门及以下"自动包含所有子部门

## 源码位置

```
module/rbac/
└── Yi.Framework.Rbac.Domain/
    └── Entities/
        └── RoleDeptEntity.cs
```

## 相关文档

- [RoleAggregateRoot](../Aggregates/RoleAggregateRoot.md) - 角色聚合根
- [DeptAggregateRoot](../Aggregates/DeptAggregateRoot.md) - 部门聚合根
- [UserRoleEntity](UserRoleEntity.md) - 用户-角色关联表
- [DataPermissionExtensions](Authorization/DataPermissionExtensions.md) - 数据权限扩展
