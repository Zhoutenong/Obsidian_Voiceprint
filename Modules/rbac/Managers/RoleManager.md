# RoleManager

**路径**: `module/rbac/Yi.Framework.Rbac.Domain/Managers/RoleManager.cs`

**依赖**: `DomainService` (Scoped)

## 概述

角色管理器，负责角色与菜单的关联管理。是 RBAC 系统中权限分配的核心组件，通过角色-菜单关系实现权限的细粒度控制。

## 核心功能

### 1. 角色菜单分配

```csharp
public async Task GiveRoleSetMenuAsync(List<Guid> roleIds, List<Guid> menuIds)
```

**职责**：
- 清除角色所有现有菜单关系（物理删除）
- 批量建立新的角色-菜单关联
- 支持多角色同时分配相同菜单

**处理流程**：
```
删除旧关系 → 遍历角色 → 创建 RoleMenuEntity → 批量插入
```

**事务要求**：需要在应用层使用工作单元（Unit of Work）

### 2. 权限传播机制

**RBAC 权限链**：
```
用户 → 角色 → 菜单 → 权限码
```

**示例**：
```
用户 Alice → 角色 "运维工程师" → 菜单 "设备管理" → 权限码 "device:manage"
                                   → 菜单 "告警查看" → 权限码 "alarm:view"
```

## 实体关系

### RoleMenuEntity

```csharp
public class RoleMenuEntity
{
    public Guid RoleId { get; set; }
    public Guid MenuId { get; set; }
}
```

### RoleAggregateRoot

```csharp
public class RoleAggregateRoot : AggregateRoot<Guid>
{
    public string RoleName { get; set; }
    public string RoleCode { get; set; }
    public List<MenuAggregateRoot> Menus { get; set; }
    public int OrderNum { get; set; }
    public bool State { get; set; }
}
```

### MenuAggregateRoot

```csharp
public class MenuAggregateRoot : AggregateRoot<Guid>
{
    public string MenuName { get; set; }
    public string? RouterName { get; set; }
    public MenuTypeEnum MenuType { get; set; }
    public string? PermissionCode { get; set; }
    public Guid ParentId { get; set; }
    public int OrderNum { get; set; }
    public bool State { get; set; }
}
```

## 菜单类型

### MenuTypeEnum

```csharp
public enum MenuTypeEnum
{
    Menu = 0,        // 目录菜单
    Menu_Item = 1,   // 菜单项
    Button = 2       // 按钮/权限
}
```

**类型说明**：
- `Menu` - 导航目录，不直接对应权限
- `MenuItem` - 功能菜单，通常对应权限码
- `Button` - 按钮权限，对应具体操作权限

## 使用示例

### 分配菜单权限

```csharp
public class RoleService
{
    private readonly RoleManager _roleManager;
    
    [UnitOfWork]
    public async Task AssignMenusToRoles(List<Guid> roleIds, List<Guid> menuIds)
    {
        await _roleManager.GiveRoleSetMenuAsync(roleIds, menuIds);
    }
}
```

### 批量分配

```csharp
// 为多个角色分配相同的菜单
var roleIds = new List<Guid> { role1Id, role2Id, role3Id };
var menuIds = new List<Guid> { menu1Id, menu2Id, menu3Id };

await _roleManager.GiveRoleSetMenuAsync(roleIds, menuIds);
```

### 细粒度权限分配

```csharp
// 只分配按钮权限
var buttonMenus = await _menuRepository
    .Where(m => m.MenuType == MenuTypeEnum.Button)
    .Select(m => m.Id)
    .ToListAsync();

await _roleManager.GiveRoleSetMenuAsync(roleIds, buttonMenus);
```

## 依赖注入

### 构造函数

```csharp
public RoleManager(
    ISqlSugarRepository<RoleAggregateRoot> repository,
    ISqlSugarRepository<RoleMenuEntity> roleMenuRepository)
{
    _repository = repository;
    _roleMenuRepository = roleMenuRepository;
}
```

**依赖说明**：
- `repository` - 角色主表仓储
- `roleMenuRepository` - 角色菜单关系仓储

## 数据查询

### 查询角色菜单

```csharp
var role = await _roleRepository
    .Includes(r => r.Menus)
    .FirstAsync(r => r.Id == roleId);

foreach (var menu in role.Menus)
{
    Console.WriteLine($"菜单: {menu.MenuName}, 权限码: {menu.PermissionCode}");
}
```

### 查询菜单权限码

```csharp
var permissionCodes = await _roleRepository
    .Where(r => r.Id == roleId)
    .SelectMany(r => r.Menus)
    .Where(m => !string.IsNullOrEmpty(m.PermissionCode))
    .Select(m => m.PermissionCode!)
    .ToListAsync();
```

## 性能优化

### 批量操作

```csharp
// 一次性批量添加
List<RoleMenuEntity> roleMenuEntity = new();
foreach (var menu in menuIds)
{
    roleMenuEntity.Add(new RoleMenuEntity() { RoleId = roleId, MenuId = menu });
}
await _roleMenuRepository.InsertRangeAsync(roleMenuEntity);
```

### 索引建议

```sql
CREATE INDEX idx_role_menu_role_id ON role_menu(RoleId);
CREATE INDEX idx_role_menu_menu_id ON role_menu(MenuId);
```

## 权限传播流程

### 1. 角色分配菜单

```csharp
await _roleManager.GiveRoleSetMenuAsync(roleIds, menuIds);
```

### 2. 用户分配角色

```csharp
await _userManager.GiveUserSetRoleAsync(userIds, roleIds);
```

### 3. 获取用户权限

```csharp
var userInfo = await _userManager.GetInfoAsync(userId);

// userInfo.PermissionCodes 包含所有权限码
foreach (var permissionCode in userInfo.PermissionCodes)
{
    Console.WriteLine($"权限码: {permissionCode}");
}
```

## 菜单树构建

### 递归构建

```csharp
public List<MenuDto> BuildMenuTree(List<MenuAggregateRoot> menus)
{
    var menuDtos = menus.Adapt<List<MenuDto>>();
    var rootMenus = menuDtos.Where(m => m.ParentId == Guid.Empty).ToList();
    
    foreach (var root in rootMenus)
    {
        root.Children = BuildChildren(root.Id, menuDtos);
    }
    
    return rootMenus;
}

private List<MenuDto> BuildChildren(Guid parentId, List<MenuDto> allMenus)
{
    var children = allMenus.Where(m => m.ParentId == parentId).ToList();
    
    foreach (var child in children)
    {
        child.Children = BuildChildren(child.Id, allMenus);
    }
    
    return children;
}
```

## 权限验证

### 前端路由守卫

```typescript
// 检查权限码
function hasPermission(permissionCode: string): boolean {
  return userInfo.permissionCodes.includes(permissionCode);
}

// 路由守卫
router.beforeEach((to, from, next) => {
  if (to.meta.permission && !hasPermission(to.meta.permission)) {
    next('/403');
  } else {
    next();
  }
});
```

### 后端 API 验证

```csharp
[Authorize]
public class DeviceService : ApplicationService
{
    [RequiresPermission("device:manage")]
    public async Task<DeviceDto> CreateAsync(CreateDeviceDto input)
    {
        // 需要权限码 "device:manage"
    }
}
```

## 数据种子

### 初始化角色菜单

```csharp
public class RoleMenuDataSeeder
{
    public async Task SeedAsync()
    {
        var adminRole = await _roleRepository.GetAsync(r => r.RoleCode == "admin");
        var allMenus = await _menuRepository.GetListAsync();
        
        await _roleManager.GiveRoleSetMenuAsync(
            new List<Guid> { adminRole.Id },
            allMenus.Select(m => m.Id).ToList()
        );
    }
}
```

## 监控指标

### 角色菜单数量

```csharp
var count = await _roleMenuRepository.GetCountAsync(rm => rm.RoleId == roleId);
_logger.LogInformation("角色 {RoleId} 菜单数量: {Count}", roleId, count);
```

### 权限码覆盖率

```csharp
var roleMenus = await _roleMenuRepository
    .Where(rm => rm.RoleId == roleId)
    .ToList();

var withPermissions = roleMenus.Count(rm => rm.Menu?.PermissionCode != null);
var coverage = (double)withPermissions / roleMenus.Count * 100;
```

## 相关文档

- [UserManager](./UserManager.md) - 用户管理器
- [菜单管理](../Services/MenuService.md)
- [权限验证](../Security/Authorization.md)
