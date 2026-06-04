# UserManager — 用户管理器（RBAC）

## 基本信息

- **Manager名称**：`UserManager`
- **模块位置**：`module/rbac/Yi.Framework.Rbac.Domain/Managers/`
- **继承关系**：`DomainService`
- **生命周期**：依赖注入（非单例）
- **命名空间**：`Yi.Framework.Rbac.Domain.Managers`

## Manager概述

UserManager 是RBAC模块的核心用户管理器，负责用户的创建、角色分配、岗位分配和用户信息查询。它集成了缓存、事件发布等功能，提供完整的用户生命周期管理。

## 核心职责

1. **用户创建** - 创建新用户并验证业务规则
2. **角色分配** - 批量给用户分配角色
3. **岗位分配** - 批量给用户分配岗位
4. **用户信息查询** - 查询用户的详细信息和权限数据
5. **默认角色设置** - 为新用户设置默认角色
6. **事件发布** - 发布用户创建事件

## 依赖注入

| 依赖 | 职责 |
|------|------|
| `ISqlSugarRepository<UserAggregateRoot>` | 用户仓储 |
| `ISqlSugarRepository<UserRoleEntity>` | 用户-角色关联仓储 |
| `ISqlSugarRepository<UserPostEntity>` | 用户-岗位关联仓储 |
| `ISqlSugarRepository<RoleAggregateRoot>` | 角色仓储 |
| `IGuidGenerator` | GUID生成器 |
| `IDistributedCache<UserInfoCacheItem, UserInfoCacheKey>` | 分布式缓存 |
| `IUserRepository` | 用户仓储 |
| `ILocalEventBus` | 本地事件总线 |

## 核心方法

### 1. CreateAsync - 创建用户

**签名**：
```csharp
public async Task CreateAsync(UserAggregateRoot userEntity)
```

**功能**：
创建新用户，包含完整的业务规则验证。

**验证规则**：

```
1. 校验用户名
   │
   ├─ ValidateUserName(userEntity)
   │  ├─ 不能为 "admin"
   │  ├─ 不能为 "tenantadmin"
   │  └─ 长度不能少于2位
   │
2. 校验密码长度
   │
   ├─ if (userEntity.EncryPassword?.Password.Length < 6)
   │  └─ 抛出异常
   │
3. 校验手机号重复
   │
   ├─ if (userEntity.Phone is not null)
   │  ├─ 查询手机号是否已存在
   │  └─ 已存在 → 抛出异常
   │
4. 校验用户名重复
   │
   ├─ 查询用户名是否已存在
   └─ 已存在 → 抛出异常
   │
5. 插入数据库
   │
   ├─ await _repository.InsertReturnEntityAsync(userEntity)
   │
6. 发布事件
   │
   └─ await _localEventBus.PublishAsync(new UserCreateEventArgs(entity.Id))
```

**异常抛出**：
- `UserFriendlyException` - 用户名无效（admin、tenantadmin、长度太短）
- `UserFriendlyException` - 密码长度少于6位
- `UserFriendlyException` - 手机号已存在
- `UserFriendlyException` - 用户名已存在

### 2. GiveUserSetRoleAsync - 设置用户角色

**签名**：
```csharp
public async Task GiveUserSetRoleAsync(List<Guid> userIds, List<Guid> roleIds)
```

**功能**：
批量给用户分配角色，先删除现有角色再分配新角色。

**执行流程**：

```
1. 物理删除现有角色
   │
   ├─ await _repositoryUserRole.DeleteAsync(u => userIds.Contains(u.UserId))
   │
2. 如果提供了新角色
   │
   ├─ 遍历每个用户
   │  │
   │  ├─ 创建 UserRoleEntity 列表
   │  │  foreach (var roleId in roleIds)
   │  │  {
   │  │      userRoleEntities.Add(new UserRoleEntity() { UserId = userId, RoleId = roleId });
   │  │  }
   │  │
   │  └─ 批量插入
   │     └─ await _repositoryUserRole.InsertRangeAsync(userRoleEntities)
```

**特点**：
- **物理删除** - 删除旧角色关系，不使用软删除
- **批量操作** - 使用InsertRange提高性能
- **一次性替换** - 不是增量添加，是全部替换

### 3. GiveUserSetPostAsync - 设置用户岗位

**签名**：
```csharp
public async Task GiveUserSetPostAsync(List<Guid> userIds, List<Guid> postIds)
```

**功能**：
批量给用户分配岗位，先删除现有岗位再分配新岗位。

**执行流程**：

```
1. 物理删除现有岗位
   │
   ├─ await _repositoryUserPost.DeleteAsync(u => userIds.Contains(u.UserId))
   │
2. 如果提供了新岗位
   │
   ├─ 遍历每个用户
   │  │
   │  ├─ 创建 UserPostEntity 列表
   │  │  foreach (var post in postIds)
   │  │  {
   │  │      userPostEntities.Add(new UserPostEntity() { UserId = userId, PostId = post });
   │  │  }
   │  │
   │  └─ 批量插入
   │     └─ await _repositoryUserPost.InsertRangeAsync(userPostEntities)
```

### 4. SetDefautRoleAsync - 设置默认角色

**签名**：
```csharp
public async Task SetDefautRoleAsync(Guid userId)
```

**功能**：
为新用户设置默认角色。

**执行流程**：

```
1. 查询默认角色
   │
   ├─ await _roleRepository.GetFirstAsync(x => x.RoleCode == UserConst.DefaultRoleCode)
   │
2. 分配默认角色
   │
   ├─ if (role is not null)
   │  └─ await GiveUserSetRoleAsync(new List<Guid> { userId }, new List<Guid> { role.Id })
```

**默认角色**：
- 代码：`UserConst.DefaultRoleCode` 通常为 `"user"`

### 5. ValidateUserName - 验证用户名

**签名**：
```csharp
private void ValidateUserName(UserAggregateRoot input)
```

**功能**：
验证用户名是否符合规则。

**验证规则**：

```csharp
// 不能为系统保留用户名
if (input.UserName == UserConst.Admin || input.UserName == UserConst.TenantAdmin)
{
    throw new UserFriendlyException("用户名无效注册！");
}

// 长度不能少于2位
if (input.UserName.Length < 2)
{
    throw new UserFriendlyException("用户名长度不能少于2位！");
}

// 更多验证规则...
```

## 相关实体

### UserAggregateRoot - 用户聚合根

包含用户的基本信息和加密密码。

### UserRoleEntity - 用户-角色关联

```csharp
public class UserRoleEntity
{
    public Guid UserId { get; set; }    // 用户ID
    public Guid RoleId { get; set; }    // 角色ID
}
```

### UserPostEntity - 用户-岗位关联

```csharp
public class UserPostEntity
{
    public Guid UserId { get; set; }    // 用户ID
    public Guid PostId { get; set; }    // 岗位ID
}
```

## 相关服务

- `AccountManager` - 账户管理器
- `RoleManager` - 角色管理器

## 缓存机制

### UserInfoCacheItem - 用户信息缓存

```csharp
public class UserInfoCacheItem
{
    public List<Guid> RoleIds { get; set; }        // 角色ID列表
    public List<string> RoleCodes { get; set; }   // 角色代码列表
    public List<Guid> MenuIds { get; set; }       // 菜单ID列表
    public List<string> PermissionCodes { get; set; } // 权限代码列表
}
```

**缓存键**：
```csharp
public class UserInfoCacheKey
{
    public Guid UserId { get; set; }
}
```

**使用场景**：
- 查询用户信息时先查缓存
- 用户、角色、菜单变更时清除缓存
- 减少数据库查询，提高性能

## 事件发布

### UserCreateEventArgs - 用户创建事件

```csharp
public class UserCreateEventArgs
{
    public Guid UserId { get; set; }
}
```

**发布时机**：
- 用户创建成功后自动发布

**订阅者**：
- 可能被其他模块监听用于初始化用户数据

## 常量定义

### UserConst - 用户常量

```csharp
public class UserConst
{
    public const string Admin = "admin";           // 管理员用户名
    public const string TenantAdmin = "tenantadmin"; // 租户管理员用户名
    public const string DefaultRoleCode = "user";   // 默认角色代码
    
    // 错误消息
    public const string Exist = "用户名已存在";
    public const string Create_Passworld_Error = "密码长度不能少于6位";
    public const string Phone_Repeat = "手机号已被使用";
}
```

## 日志记录

UserManager主要通过异常和依赖的服务进行日志记录。

## 错误处理

### 用户友好异常

所有业务规则错误都抛出 `UserFriendlyException`：

```csharp
// 用户名无效
throw new UserFriendlyException("用户名无效注册！");

// 密码太短
throw new UserFriendlyException(UserConst.Create_Passworld_Error);

// 用户名已存在
throw new UserFriendlyException(UserConst.Exist);

// 手机号已存在
throw new UserFriendlyException(UserConst.Phone_Repeat);
```

## 使用场景

### 1. 创建用户并分配角色

```csharp
// 创建用户
var user = new UserAggregateRoot("newuser", "password123", 13800138000, "昵称");
await _userManager.CreateAsync(user);

// 分配角色
await _userManager.GiveUserSetRoleAsync(
    new List<Guid> { user.Id }, 
    new List<Guid> { role1Id, role2Id }
);

// 分配岗位
await _userManager.GiveUserSetPostAsync(
    new List<Guid> { user.Id }, 
    new List<Guid> { post1Id, post2Id }
);
```

### 2. 批量用户角色管理

```csharp
// 批量分配角色
var userIds = new List<Guid> { user1Id, user2Id, user3Id };
var roleIds = new List<Guid> { adminRoleId, operatorRoleId };

await _userManager.GiveUserSetRoleAsync(userIds, roleIds);
```

### 3. 新用户注册流程

```csharp
// 1. 创建用户（内部会验证规则）
await _userManager.CreateAsync(userEntity);

// 2. 设置默认角色
await _userManager.SetDefautRoleAsync(userEntity.Id);

// 3. 发布用户创建事件
// （自动完成）
```

## 设计特点

1. **批量操作** - 角色和岗位分配支持批量处理
2. **完整替换** - 分配角色/岗位是全部替换，不是增量添加
3. **物理删除** - 删除旧关系不使用软删除
4. **事件驱动** - 创建用户时发布事件
5. **缓存优化** - 使用分布式缓存提高查询性能
6. **业务验证** - 创建用户时进行完整的规则验证
7. **默认角色** - 新用户自动分配默认角色

## 性能优化

1. **批量插入** - 使用InsertRange提高批量操作性能
2. **分布式缓存** - 用户信息缓存减少数据库查询
3. **批量查询** - GetInfoListAsync支持批量查询用户信息

## 注意事项

1. **系统保留用户名** - "admin" 和 "tenantadmin" 不能注册
2. **用户名长度** - 至少2个字符
3. **密码长度** - 至少6个字符
4. **手机号唯一** - 手机号不能重复
5. **角色替换** - GiveUserSetRoleAsync 是全部替换，不是增量添加
6. **事务管理** - 角色和岗位分配需要在UnitOfWork事务中执行

---

> **最后更新**：2026-06-04  
> **源码位置**：`module/rbac/Yi.Framework.Rbac.Domain/Managers/UserManager.cs`
