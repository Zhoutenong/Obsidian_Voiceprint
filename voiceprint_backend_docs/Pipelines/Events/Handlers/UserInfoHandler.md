# UserInfoHandler — 用户信息查询事件处理器

## 基本信息

- **处理器名称**：`UserInfoHandler`
- **模块位置**：`module/rbac/Yi.Framework.Rbac.Domain/EventHandlers/`
- **事件类型**：`UserRoleMenuQueryEventArgs` (ILocalEventHandler)
- **生命周期**：`ITransientDependency` - 瞬态依赖
- **命名空间**：`Yi.Framework.Rbac.Domain.EventHandlers`

## 处理器概述

UserInfoHandler 负责处理用户角色菜单查询事件。当需要批量查询用户信息时，该处理器通过UserManager查询用户的详细信息和权限数据，并将结果设置到事件参数中。这是一个RBAC模块的领域事件处理器，用于用户权限查询。

## 核心职责

1. **批量用户查询** - 批量查询多个用户的信息
2. **权限数据查询** - 查询用户的角色、菜单、权限信息
3. **结果返回** - 将查询结果设置到事件参数中

## 依赖注入

| 依赖 | 职责 |
|------|------|
| `UserManager` | 用户管理器 |

## 事件数据结构

### UserRoleMenuQueryEventArgs

```csharp
public class UserRoleMenuQueryEventArgs
{
    public List<Guid> UserIds { get; set; }  // 需要查询的用户ID列表
    public object Result { get; set; }       // 查询结果（输出）
}
```

## 处理流程

```
1. 接收事件
   │
   ├─ 获取用户ID列表
   │
2. 调用UserManager查询
   │
   ├─ 调用 GetInfoListAsync 方法
   ├─ 传入用户ID列表
   │
3. 获取查询结果
   │
   ├─ 返回用户信息列表
   ├─ 包含用户基本信息
   ├─ 包含用户角色信息
   └─ 包含用户菜单权限
   │
4. 设置事件结果
   │
   └─ 将结果设置到 eventData.Result
```

## 核心方法

### HandleEventAsync - 处理用户信息查询事件

**签名**：
```csharp
public async Task HandleEventAsync(UserRoleMenuQueryEventArgs eventData)
```

**功能**：
处理用户信息查询事件，批量查询用户、角色、菜单信息。

**完整实现**：

```csharp
public async Task HandleEventAsync(UserRoleMenuQueryEventArgs eventData)
{
    // 1. 调用UserManager查询用户信息列表
    var result = await _userManager.GetInfoListAsync(eventData.UserIds);
    
    // 2. 将结果设置到事件参数中
    eventData.Result = result;
}
```

## UserManager.GetInfoListAsync

### 方法签名

```csharp
public async Task<List<UserInfoDto>> GetInfoListAsync(List<Guid> userIds)
```

### 功能说明

批量查询用户的详细信息，包括：
- 用户基本信息（用户名、邮箱、电话等）
- 用户角色列表
- 用户菜单权限列表
- 用户权限点列表

### 返回数据结构

```csharp
public class UserInfoDto
{
    public Guid Id { get; set; }                // 用户ID
    public string UserName { get; set; }       // 用户名
    public string? Email { get; set; }         // 邮箱
    public string? PhoneNumber { get; set; }    // 电话号码
    public List<RoleDto> Roles { get; set; }   // 角色列表
    public List<MenuDto> Menus { get; set; }   // 菜单列表
    // 可能还包含其他权限信息
}

public class RoleDto
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public string? Code { get; set; }
    // 可能还包含角色权限信息
}

public class MenuDto
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public string Path { get; set; }
    public string? Icon { get; set; }
    // 可能还包含子菜单、权限等
}
```

## 使用场景

### 1. 批量用户信息查询

当需要查询多个用户的详细信息时：
```csharp
var eventArgs = new UserRoleMenuQueryEventArgs
{
    UserIds = new List<Guid>
    {
        Guid.Parse("3fa85f64-5717-4562-b3fc-2c963f66afa6"),
        Guid.Parse("4fa85f64-5717-4562-b3fc-2c963f66afa7")
    }
};

// 触发事件后，eventArgs.Result 包含这两个用户的完整信息
```

### 2. 用户权限验证

在验证用户权限时，先查询用户的角色和菜单：
```csharp
var userInfo = await _userManager.GetInfoListAsync(new List<Guid> { userId });
var user = userInfo.First();

// 检查用户是否有特定角色
bool hasRole = user.Roles.Any(r => r.Code == "Admin");

// 检查用户是否有特定菜单权限
bool hasMenu = user.Menus.Any(m => m.Path == "/admin/dashboard");
```

### 3. 用户列表展示

在用户管理页面，展示用户及其角色：
```csharp
var userIds = userList.Select(u => u.Id).ToList();
var eventArgs = new UserRoleMenuQueryEventArgs { UserIds = userIds };

// 触发事件后获取用户详细信息
var userInfos = eventArgs.Result as List<UserInfoDto>;

// 在表格中展示：
// 用户名 | 邮箱 | 角色 | 菜单权限
```

### 4. 用户数据导出

导出用户及其权限数据：
```csharp
var allUsers = await _userRepository.GetListAsync();
var userIds = allUsers.Select(u => u.Id).ToList();

var eventArgs = new UserRoleMenuQueryEventArgs { UserIds = userIds };
// 触发事件后导出数据
```

## 性能考虑

1. **批量查询** - 一次查询多个用户，减少数据库往返
2. **关联查询优化** - UserManager内部应该使用Include或JOIN优化关联查询
3. **缓存策略** - 考虑对用户权限数据进行缓存
4. **分页处理** - 当用户数量很大时，考虑分批查询

## 与其他处理器的区别

### LoginEventHandler vs UserInfoHandler

| 特性 | LoginEventHandler | UserInfoHandler |
|------|-----------------|-----------------|
| **触发时机** | 用户登录时 | 需要查询用户信息时 |
| **事件类型** | LoginEventArgs | UserRoleMenuQueryEventArgs |
| **处理内容** | 记录登录日志 | 查询用户详细信息 |
| **数据写入** | 写入登录日志 | 设置事件结果 |
| **主要用途** | 审计追踪 | 权限验证、数据展示 |

## 设计特点

1. **简洁实现** - 逻辑非常简单，只做查询和结果设置
2. **委托模式** - 将查询逻辑委托给UserManager
3. **事件驱动** - 通过事件机制解耦查询调用和查询实现
4. **批量处理** - 支持批量查询，提高效率

## 相关实体

- [[UserAggregateRoot]] - 用户聚合根
- [[RoleAggregateRoot]] - 角色聚合根
- [[MenuAggregateRoot]] - 菜单聚合根

## 相关服务

- `UserManager` - 用户管理器

## 相关事件

- `UserRoleMenuQueryEventArgs` - 用户角色菜单查询事件参数

## 扩展建议

### 1. 增加查询选项

```csharp
public class UserRoleMenuQueryEventArgs
{
    public List<Guid> UserIds { get; set; }
    public bool IncludeRoles { get; set; } = true;     // 是否包含角色
    public bool IncludeMenus { get; set; } = true;     // 是否包含菜单
    public bool IncludePermissions { get; set; } = false; // 是否包含权限点
    public object Result { get; set; }
}
```

### 2. 分页支持

当用户数量很大时，支持分页查询：
```csharp
public class UserRoleMenuQueryPagedEventArgs : UserRoleMenuQueryEventArgs
{
    public int SkipCount { get; set; }
    public int MaxResultCount { get; set; }
    public int TotalCount { get; set; }
}
```

### 3. 缓存机制

在UserManager中增加缓存：
```csharp
public async Task<List<UserInfoDto>> GetInfoListAsync(List<Guid> userIds)
{
    // 先从缓存中查找
    var cached = await _cache.GetAsync<List<UserInfoDto>>(GetCacheKey(userIds));
    if (cached != null)
    {
        return cached;
    }
    
    // 查询数据库
    var result = await _repository.GetListAsync(...);
    
    // 存入缓存
    await _cache.SetAsync(GetCacheKey(userIds), result, TimeSpan.FromMinutes(30));
    
    return result;
}
```

## 典型调用链

```
应用服务
    │
    ├─ 创建事件
    │  eventArgs = new UserRoleMenuQueryEventArgs
    │  eventArgs.UserIds = userIds
    │
    ├─ 触发事件
    │  await _eventBus.PublishAsync(eventArgs)
    │
    └─ UserInfoHandler处理
       │
       └─ UserManager.GetInfoListAsync
          │
          ├─ 查询用户基本信息
          ├─ 查询用户角色关联
          ├─ 查询角色信息
          ├─ 查询用户菜单权限
          └─ 组装返回结果
```

## 错误处理

当前实现没有显式的错误处理。建议改进：

```csharp
public async Task HandleEventAsync(UserRoleMenuQueryEventArgs eventData)
{
    try
    {
        if (eventData.UserIds == null || !eventData.UserIds.Any())
        {
            eventData.Result = new List<UserInfoDto>();
            return;
        }
        
        var result = await _userManager.GetInfoListAsync(eventData.UserIds);
        eventData.Result = result ?? new List<UserInfoDto>();
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "查询用户信息失败: UserId={UserId}", 
            eventData.UserIds != null ? string.Join(",", eventData.UserIds) : "");
        eventData.Result = new List<UserInfoDto>();
    }
}
```

---

> **最后更新**：2026-06-04  
> **源码位置**：`module/rbac/Yi.Framework.Rbac.Domain/EventHandlers/UserInfoHandler.cs"
