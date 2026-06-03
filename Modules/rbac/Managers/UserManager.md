# UserManager

**路径**: `module/rbac/Yi.Framework.Rbac.Domain/Managers/UserManager.cs`

**依赖**: `DomainService` (Scoped)

## 概述

用户管理器，负责用户与角色、岗位的关联管理以及用户信息查询。是 RBAC（基于角色的访问控制）系统的核心领域服务，处理用户授权关系和数据转换。

## 核心功能

### 1. 用户角色分配

```csharp
public async Task GiveUserSetRoleAsync(List<Guid> userIds, List<Guid> roleIds)
```

**职责**：
- 清除用户所有现有角色关系（物理删除）
- 批量建立新的用户-角色关联
- 支持多用户同时分配相同角色

**处理流程**：
```
删除旧关系 → 遍历用户 → 创建 UserRoleEntity → 批量插入
```

**事务要求**：需要在应用层使用工作单元（Unit of Work）

### 2. 用户岗位分配

```csharp
public async Task GiveUserSetPostAsync(List<Guid> userIds, List<Guid> postIds)
```

**职责**：
- 清除用户所有现有岗位关系
- 批量建立新的用户-岗位关联
- 支持多用户同时分配相同岗位

**使用场景**：组织架构调整、人员岗位变动

### 3. 用户信息查询

```csharp
public async Task<UserRoleMenuDto> GetInfoAsync(Guid userId)
```

**职责**：
- 查询用户完整信息（包含导航属性）
- 转换为 DTO 并清除敏感信息
- 处理超级管理员特殊逻辑
- 构建角色码和权限码集合

**返回结构**：
```csharp
public class UserRoleMenuDto
{
    public UserDto User { get; set; }
    public HashSet<RoleDto> Roles { get; set; }
    public HashSet<MenuDto> Menus { get; set; }
    public HashSet<string> RoleCodes { get; set; }
    public HashSet<string> PermissionCodes { get; set; }
}
```

### 4. 用户完整信息获取

```csharp
public async Task<UserAggregateRoot> GetUserAllInfoAsync(Guid userId)
```

**职责**：
- 查询用户及其角色、菜单、岗位等关联数据
- 使用 `Includes()` 预加载导航属性
- 用于登录认证和权限验证

## 实体关系

### UserRoleEntity

```csharp
public class UserRoleEntity
{
    public Guid UserId { get; set; }
    public Guid RoleId { get; set; }
}
```

### UserPostEntity

```csharp
public class UserPostEntity
{
    public Guid UserId { get; set; }
    public Guid PostId { get; set; }
}
```

## 超级管理员处理

### 特殊逻辑

```csharp
if (UserConst.Admin.Equals(user.UserName))
{
    userRoleMenu.RoleCodes.Add(UserConst.AdminRolesCode);
    userRoleMenu.PermissionCodes.Add(UserConst.AdminPermissionCode);
    return userRoleMenu;
}
```

**特权**：
- 无需角色即可访问
- 拥有所有权限码
- 绕过常规权限检查

## 数据转换

### EntityMapToDto

```csharp
private UserRoleMenuDto EntityMapToDto(UserAggregateRoot user)
{
    // 清除密码和盐值
    user.EncryPassword.Password = string.Empty;
    user.EncryPassword.Salt = string.Empty;
    
    // 转换角色
    foreach (var role in user.Roles)
    {
        userRoleMenu.RoleCodes.Add(role.RoleCode);
        
        // 转换菜单和权限
        foreach (var menu in role.Menus)
        {
            if (!string.IsNullOrEmpty(menu.PermissionCode))
            {
                userRoleMenu.PermissionCodes.Add(menu.PermissionCode);
            }
            userRoleMenu.Menus.Add(menu.Adapt<MenuDto>());
        }
    }
}
```

## 使用示例

### 分配角色

```csharp
public class UserService
{
    private readonly UserManager _userManager;
    
    public async Task AssignRolesToUsers(List<Guid> userIds, List<Guid> roleIds)
    {
        await _userManager.GiveUserSetRoleAsync(userIds, roleIds);
    }
}
```

### 分配岗位

```csharp
public async Task AssignPostsToUsers(List<Guid> userIds, List<Guid> postIds)
{
    await _userManager.GiveUserSetPostAsync(userIds, postIds);
}
```

### 获取用户信息

```csharp
public async Task<UserRoleMenuDto> GetUserInfo(Guid userId)
{
    return await _userManager.GetInfoAsync(userId);
}
```

### 登录时获取权限

```csharp
public async Task<LoginOutputDto> LoginAsync(string userName, string password)
{
    var user = await _userManager.GetUserAllInfoAsync(userId);
    
    // 验证密码
    if (!ValidatePassword(user, password))
    {
        throw new UserFriendlyException("密码错误");
    }
    
    // 获取权限信息
    var userInfo = await _userManager.GetInfoAsync(userId);
    
    // 生成 Token
    var token = await _accountManager.GetTokenByUserIdAsync(userId);
    
    return new LoginOutputDto { Token = token };
}
```

## 依赖注入

### 构造函数

```csharp
public UserManager(
    ISqlSugarRepository<UserAggregateRoot> repository,
    ISqlSugarRepository<UserRoleEntity> repositoryUserRole,
    ISqlSugarRepository<UserPostEntity> repositoryUserPost,
    IGuidGenerator guidGenerator,
    IDistributedCache<UserInfoCacheItem, UserInfoCacheKey> userCache,
    IUserRepository userRepository,
    ILocalEventBus localEventBus,
    ISqlSugarRepository<RoleAggregateRoot> roleRepository)
```

**依赖说明**：
- `repository` - 用户主表仓储
- `repositoryUserRole` - 用户角色关系仓储
- `repositoryUserPost` - 用户岗位关系仓储
- `userCache` - 用户信息分布式缓存
- `userRepository` - 用户自定义仓储
- `roleRepository` - 角色仓储（用于查询）

## 缓存策略

### UserInfoCacheItem

```csharp
public class UserInfoCacheItem
{
    public Guid UserId { get; set; }
    public string UserName { get; set; }
    public List<string> RoleCodes { get; set; }
    public List<string> PermissionCodes { get; set; }
}
```

### 缓存键

```csharp
public class UserInfoCacheKey
{
    public Guid UserId { get; set; }
}
```

## 性能优化

### 批量操作

```csharp
// 一次性批量添加
await _repositoryUserRole.InsertRangeAsync(userRoleEntities);
```

### 导航属性预加载

```csharp
var user = await _repository
    .Includes(u => u.Roles, r => r.Menus)
    .Includes(u => u.Posts)
    .FirstAsync(u => u.Id == userId);
```

## 错误处理

### 用户不存在

```csharp
if (user is null)
{
    throw new UserFriendlyException($"数据错误，查询用户不存在，请重新登录");
}
```

### 权限不足

```csharp
if (userInfo.RoleCodes.Count == 0)
{
    throw new UserFriendlyException(UserConst.No_Role);
}

if (!userInfo.PermissionCodes.Any())
{
    throw new UserFriendlyException(UserConst.No_Permission);
}
```

## 事件发布

### 用户状态变更

```csharp
await _localEventBus.PublishAsync(userEventData);
```

## 常量定义

### UserConst

```csharp
public static class UserConst
{
    public const string Admin = "admin";
    public const string AdminRolesCode = "admin";
    public const string AdminPermissionCode = "*";
    public const string No_Role = "该用户未分配角色";
    public const string No_Permission = "该用户无任何权限";
    public const string State_Is_State = "用户已被禁用";
}
```

## 相关文档

- [RoleManager](./RoleManager.md) - 角色管理器
- [AccountManager](./AccountManager.md) - 账号管理器
- [RBAC 架构](../Architecture.md)
