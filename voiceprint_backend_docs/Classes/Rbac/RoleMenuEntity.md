# RoleMenuEntity

**关联表：角色-菜单多对多关系**

## 概述

`RoleMenuEntity` 是角色与菜单之间的多对多关联表，用于实现基于角色的菜单访问控制。每个角色可以访问多个菜单，每个菜单也可以分配给多个角色。这是前端路由权限控制的核心表，决定了用户登录后能看到哪些菜单项。

## 表信息

| 属性 | 值 |
|------|-----|
| **表名** | `RoleMenu` |
| **主键** | `Id` (GUID) |
| **复合主键** | 无（使用单主键 Id） |

## 字段说明

| 字段名 | 类型 | 说明 | 外键 |
|--------|------|------|------|
| `Id` | `Guid` | 主键 ID | - |
| `RoleId` | `Guid` | 角色 ID | → `RoleAggregateRoot.Id` |
| `MenuId` | `Guid` | 菜单 ID | → `MenuAggregateRoot.Id` |

## 关联实体 (ER 关系)

```
RoleAggregateRoot (1) ←→ (*) MenuAggregateRoot
          │                         │
          └── RoleId                MenuId
                   │            │
                   ▼            ▼
             RoleMenuEntity
```

**关联端：**

| 导航属性 | 目标实体 | 关联类型 |
|----------|----------|----------|
| `Role` | `RoleAggregateRoot` | Many-to-One |
| `Menu` | `MenuAggregateRoot` | Many-to-One |

## 服务使用

### 写入服务

- **`RoleService`** - 角色管理时分配菜单权限
  - `AssignMenusAsync()` - 为角色分配菜单
  - `RemoveMenuAsync()` - 移除角色菜单
  - `UpdateRoleMenusAsync()` - 批量更新角色菜单

- **`MenuService`** - 菜单管理时查看关联角色
  - `GetRolesByMenuAsync()` - 查询菜单关联的角色列表

### 读取服务

- **`MenuService`** - 构建用户菜单树时查询
- **`AccountService`** - 登录时获取用户菜单权限
- **`RoleService`** - 角色详情时包含菜单信息

## 业务规则

1. **级联删除**
   - 删除角色时，自动删除对应的菜单关联
   - 删除菜单时，自动删除对应的角色关联

2. **唯一性**
   - 同一 `(RoleId, MenuId)` 组合应唯一
   - 通过业务逻辑层控制

3. **菜单树继承**
   - 分配父菜单时，默认包含子菜单
   - 删除父菜单时，自动删除子菜单关联

4. **权限计算**
   - 用户菜单 = 所有角色菜单的并集
   - 菜单权限在前端路由守卫中验证

5. **菜单类型**
   - `目录`（Directory）- 仅用于分组，无页面
   - `菜单`（Menu）- 对应具体页面
   - `按钮`（Button）- 页面内的操作权限

## 源码位置

```
module/rbac/
└── Yi.Framework.Rbac.Domain/
    └── Entities/
        └── RoleMenuEntity.cs
```

## 相关文档

- [RoleAggregateRoot](../Aggregates/RoleAggregateRoot.md) - 角色聚合根
- [MenuAggregateRoot](../Aggregates/MenuAggregateRoot.md) - 菜单聚合根
- [UserRoleEntity](UserRoleEntity.md) - 用户-角色关联表
