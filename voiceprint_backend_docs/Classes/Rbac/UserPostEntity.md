# UserPostEntity

**关联表：用户-岗位多对多关系**

## 概述

`UserPostEntity` 是用户与岗位之间的多对多关联表，用于管理用户的岗位归属。岗位通常表示组织结构中的职位（如"主任"、"技术员"），可以用于数据权限控制和组织管理。每个用户可以拥有多个岗位，每个岗位也可以分配给多个用户。

## 表信息

| 属性 | 值 |
|------|-----|
| **表名** | `UserPost` |
| **主键** | `Id` (GUID) |
| **复合主键** | 无（使用单主键 Id） |

## 字段说明

| 字段名 | 类型 | 说明 | 外键 |
|--------|------|------|------|
| `Id` | `Guid` | 主键 ID | - |
| `UserId` | `Guid` | 用户 ID | → `UserAggregateRoot.Id` |
| `PostId` | `Guid` | 岗位 ID | → `PostAggregateRoot.Id` |

## 关联实体 (ER 关系)

```
UserAggregateRoot (1) ←→ (*) PostAggregateRoot
        │                         │
        └── UserId                PostId
                 │            │
                 ▼            ▼
           UserPostEntity
```

**关联端：**

| 导航属性 | 目标实体 | 关联类型 |
|----------|----------|----------|
| `User` | `UserAggregateRoot` | Many-to-One |
| `Post` | `PostAggregateRoot` | Many-to-One |

## 服务使用

### 写入服务

- **`UserService`** - 用户管理时分配岗位
  - `AssignPostsAsync()` - 为用户分配岗位
  - `RemovePostAsync()` - 移除用户岗位
  - `UpdateUserPostsAsync()` - 批量更新用户岗位

- **`PostService`** - 岗位管理时查看关联用户
  - `GetUsersByPostAsync()` - 查询岗位下的用户列表

### 读取服务

- **`UserService`** - 用户详情时包含岗位信息
- **`AccountService`** - 登录时获取用户岗位
- **`PostService`** - 岗位详情时包含用户列表

## 业务规则

1. **级联删除**
   - 删除用户时，自动删除对应的岗位关联
   - 删除岗位时，自动删除对应的用户关联

2. **唯一性**
   - 同一 `(UserId, PostId)` 组合应唯一
   - 通过业务逻辑层控制

3. **岗位用途**
   - 用于组织管理和人员编制
   - 可作为数据权限控制的维度
   - 支持按岗位进行工作流审批

4. **与角色的区别**
   - **岗位**：组织结构中的职位，相对固定
   - **角色**：权限集合，可灵活分配
   - 用户可同时拥有多个岗位和多个角色

## 源码位置

```
module/rbac/
└── Yi.Framework.Rbac.Domain/
    └── Entities/
        └── UserPostEntity.cs
```

## 相关文档

- [UserAggregateRoot](../Aggregates/UserAggregateRoot.md) - 用户聚合根
- [PostAggregateRoot](../Aggregates/PostAggregateRoot.md) - 岗位聚合根
- [UserRoleEntity](UserRoleEntity.md) - 用户-角色关联表
