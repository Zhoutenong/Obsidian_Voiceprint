# RoleManager — 角色管理器（RBAC）

## 基本信息

- **Manager名称**：`RoleManager`
- **模块位置**：`module/rbac/Yi.Framework.Rbac.Domain/Managers/`
- **继承关系**：`DomainService`
- **生命周期**：依赖注入（非单例）
- **命名空间**：`Yi.Framework.Rbac.Domain.Managers`

## Manager概述

RoleManager 是RBAC模块的角色管理器，负责角色的菜单分配和管理。它提供批量给角色分配菜单的功能，是实现基于角色的访问控制（RBAC）的核心组件。

## 核心职责

1. **菜单分配** - 批量给角色分配菜单权限
2. **关系管理** - 管理角色-菜单关联关系
3. **批量操作** - 支持批量角色和菜单处理

## 依赖注入

| 依赖 | 职责 |
|------|------|
| `ISqlSugarRepository<RoleAggregateRoot>` | 角色仓储 |
| `ISqlSugarRepository<RoleMenuEntity>` | 角色-菜单关联仓储 |

## 核心方法

### GiveRoleSetMenuAsync - 设置角色菜单

**签名**：
```csharp
public async Task GiveRoleSetMenuAsync(List<Guid> roleIds, List<Guid> menuIds)
```

**功能**：
批量给角色分配菜单权限。

**执行流程**：

```
1. 物理删除现有菜单
   │
   ├─ await _roleMenuRepository.DeleteAsync(u => roleIds.Contains(u.RoleId))
   │
2. 遍历每个角色
   │
   ├─ foreach (var roleId in roleIds)
   │  │
   │  ├─ 创建 RoleMenuEntity 列表
   │  │  foreach (var menu in menuIds)
   │  │  {
   │  │      roleMenuEntity.Add(new RoleMenuEntity() { RoleId = roleId, MenuId = menu });
   │  │  }
   │  │
   │  └─ 批量插入
   │     └─ await _roleMenuRepository.InsertRangeAsync(roleMenuEntity)
```

**特点**：
- **物理删除** - 删除旧菜单关系，不使用软删除
- **批量操作** - 使用InsertRange提高性能
- **一次性替换** - 不是增量添加，是全部替换

## 相关实体

### RoleAggregateRoot - 角色聚合根

包含角色的基本信息、数据权限、状态等。

### RoleMenuEntity - 角色-菜单关联

```csharp
public class RoleMenuEntity
{
    public Guid RoleId { get; set; }    // 角色ID
    public Guid MenuId { get; set; }    // 菜单ID
}
```

**表结构**：
- 通常存储在 `yi_role_menu` 表中
- 联合主键：`RoleId + MenuId`
- 表示某个角色可以访问某个菜单

## RBAC权限层次

```
用户 (User)
  │
  ├─ 拥有角色 (Role)
  │
  ├─ 拥有岗位 (Post)
  │
  └─ 拥有权限 (Permission)
      │
      └─ 通过角色分配菜单 (Menu)
          │
          └─ 菜单关联权限点
```

**说明**：
- 用户通过角色或岗位获得权限
- 角色通过菜单获得前端页面访问权限
- 菜单关联具体的权限点
- RoleManager负责管理角色-菜单的关联关系

## 使用场景

### 1. 创建角色并分配菜单

```csharp
// 1. 创建角色（通过RoleService）
var role = new RoleAggregateRoot("操作员", "operator");
await _roleService.CreateAsync(role);

// 2. 分配菜单权限
var menuIds = new List<Guid> 
{ 
    menuDashboardId, 
    menuDeviceMonitorId, 
    menuAlarmManageId 
};

await _roleManager.GiveRoleSetMenuAsync(
    new List<Guid> { role.Id }, 
    menuIds
);
```

### 2. 批量角色菜单管理

```csharp
// 多个角色分配相同菜单
var roleIds = new List<Guid> { role1Id, role2Id, role3Id };
var menuIds = new List<Guid> { menu1Id, menu2Id, menu3Id };

await _roleManager.GiveRoleSetMenuAsync(roleIds, menuIds);
```

### 3. 更新角色菜单

```csharp
// 完全替换角色的菜单（不是增量添加）
await _roleManager.GiveRoleSetMenuAsync(
    new List<Guid> { roleId }, 
    newMenuIds
);
```

## 设计特点

1. **简单直接** - 逻辑简单，只做一件事：分配菜单
2. **批量操作** - 支持批量角色处理
3. **完整替换** - 分配菜单是全部替换，不是增量添加
4. **物理删除** - 删除旧关系不使用软删除
5. **高性能** - 使用InsertRange批量插入

## 注意事项

1. **事务管理** - 需要在UnitOfWork事务中执行
2. **关系替换** - 每次调用都是完全替换，不是增量操作
3. **空列表处理** - 如果menuIds为空，会清空角色的所有菜单
4. **角色删除** - 删除角色时需要同时删除角色-菜单关联

## 与其他Manager的配合

### 与UserManager配合

```csharp
// 1. 创建用户并分配角色
await _userManager.CreateAsync(user);
await _userManager.GiveUserSetRoleAsync(
    new List<Guid> { user.Id }, 
    new List<Guid> { roleId }
);

// 2. 角色已有菜单分配
// （通过RoleManager预先设置）
```

### 权限验证流程

```
1. 用户登录
   │
2. 查询用户的角色和菜单
   │
   ├─ UserManager.GetInfoAsync()
   │
3. 根据菜单生成前端路由
   │
   ├─ 前端只显示用户有权限访问的菜单
   │
4. 访问控制
   │
   └─ 后端验证用户是否有相应权限
```

## 性能优化

1. **批量插入** - 使用InsertRange而非循环插入
2. **物理删除** - 直接DELETE而非软删除，提高性能
3. **索引优化** - 在RoleId和MenuId上建立联合索引

## 日志记录

RoleManager主要通过依赖的服务进行日志记录。

## 扩展建议

### 1. 数据权限

当前版本主要管理菜单权限，可以考虑扩展数据权限：

```csharp
public async Task GiveRoleSetDataScopeAsync(
    List<Guid> roleIds, 
    List<Guid> dataScopeIds)
{
    // 分配数据权限范围
}
```

### 2. 权限继承

支持角色间的权限继承：

```csharp
public async Task InheritPermissionsFromRoleAsync(
    Guid targetRoleId, 
    Guid sourceRoleId)
{
    // 从源角色继承权限到目标角色
}
```

### 3. 临时权限

支持为角色分配临时权限（带过期时间）：

```csharp
public async Task GiveRoleTemporaryPermissionAsync(
    Guid roleId, 
    List<Guid> permissionIds,
    DateTime expireTime)
{
    // 分配临时权限
}
```

## 相关服务

- `RoleService` - 角色服务（应用层）
- `UserManager` - 用户管理器
- `MenuService` - 菜单服务
- `PermissionService` - 权限服务

## 相关实体

- `RoleAggregateRoot` - 角色聚合根
- `MenuAggregateRoot` - 菜单聚合根
- `PermissionEntity` - 权限实体

## 管理端API说明

通常在管理端（`/admin`）提供以下RBAC CRUD接口：

- 角色管理（CRUD）
- 角色菜单分配
- 角色权限设置
- 用户角色分配
- 用户岗位分配

这些接口在内部会调用RoleManager和UserManager来完成实际的数据操作。

---

> **最后更新**：2026-06-04  
> **源码位置**：`module/rbac/Yi.Framework.Rbac.Domain/Managers/RoleManager.cs`
