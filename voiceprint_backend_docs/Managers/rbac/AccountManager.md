# AccountManager — 账户管理器（RBAC）

## 基本信息

- **Manager名称**：`AccountManager`
- **模块位置**：`module/rbac/Yi.Framework.Rbac.Domain/Managers/`
- **继承关系**：`DomainService` → `IAccountManager`
- **生命周期**：依赖注入（非单例）
- **命名空间**：`Yi.Framework.Rbac.Domain.Managers`

## Manager概述

AccountManager 是RBAC模块的核心账户管理器，负责用户认证、令牌生成、密码管理和用户注册。它整合了JWT认证、用户信息查询和角色权限管理功能。

## 核心职责

1. **用户认证** - 登录验证和账户存在性检查
2. **令牌生成** - 生成访问令牌和刷新令牌
3. **密码管理** - 密码更新、重置和验证
4. **用户注册** - 新用户注册和默认角色分配
5. **Claims转换** - 将用户信息转换为JWT Claims

## 依赖注入

| 依赖 | 职责 |
|------|------|
| `IUserRepository` | 用户仓储 |
| `ILocalEventBus` | 本地事件总线 |
| `JwtOptions` | JWT配置选项 |
| `RbacOptions` | RBAC配置选项 |
| `UserManager` | 用户管理器 |
| `ISqlSugarRepository<RoleAggregateRoot>` | 角色仓储 |
| `RefreshJwtOptions` | 刷新令牌配置选项 |

## 核心方法

### 1. GetTokenByUserIdAsync - 根据用户ID获取令牌

**签名**：
```csharp
public async Task<string> GetTokenByUserIdAsync(
    Guid userId,
    Action<UserRoleMenuDto>? getUserInfo = null)
```

**功能**：
根据用户ID生成访问令牌（JWT Token）。

**执行流程**：

```
1. 获取用户信息
   │
   ├─ 调用 UserManager.GetInfoAsync(userId)
   │
2. 用户状态验证
   │
   ├─ 检查用户状态（State == false → 抛出异常）
   ├─ 检查角色分配（Count == 0 → 抛出异常）
   └─ 检查权限分配（Any() == false → 抛出异常）
   │
3. 执行用户信息回调
   │
   ├─ 如果 getUserInfo 不为null
   │  └─ getUserInfo(userInfo)
   │
4. 生成访问令牌
   │
   ├─ UserInfoToClaim(userInfo) → List<KeyValuePair<string, string>>
   ├─ CreateToken(claims) → string
   │
5. 返回令牌
```

**异常抛出**：
- `UserFriendlyException` - 用户状态无效
- `UserFriendlyException` - 无角色分配
- `UserFriendlyException` - 无权限分配

### 2. LoginValidationAsync - 登录验证

**签名**：
```csharp
public async Task LoginValidationAsync(
    string userName, 
    string password, 
    Action<UserAggregateRoot> userAction = null)
```

**功能**：
验证用户登录凭据。

**执行流程**：

```
1. 检查账户存在性
   │
   ├─ await ExistAsync(userName, userAction)
   │
2. 执行用户回调
   │
   ├─ 如果 userAction 不为null
   │  └─ userAction.Invoke(user)
   │
3. 验证密码
   │
   ├─ user.EncryPassword.Password == MD5Helper.SHA2Encode(password, user.EncryPassword.Salt)
   │
4. 返回结果
   │
   ├─ 验证成功 → 正常返回
   └─ 验证失败 → 抛出 ApplicationException
```

**异常抛出**：
- `ApplicationException` - 用户不存在
- `ApplicationException` - 密码错误

### 3. ExistAsync - 判断账户合法存在

**签名**：
```csharp
public async Task<bool> ExistAsync(
    string userName, 
    Action<UserAggregateRoot> userAction = null)
```

**功能**：
检查用户名是否存在且处于有效状态。

**执行流程**：

```
1. 查询用户
   │
   ├─ await _repository.GetFirstAsync(u => u.UserName == userName && u.State == true)
   │
2. 执行用户回调
   │
   ├─ 如果 userAction 不为null
   │  └─ userAction.Invoke(user)
   │
3. 二次校验用户名（兼容数据库大小写不敏感问题）
   │
   ├─ if (user != null && user.UserName == userName)
   │  ├─ return true
   │  └─ 否则 return false
```

**设计考虑**：
- 兼容数据库开启了大小写不敏感的情况
- 进行二次精确的用户名匹配

### 4. UserInfoToClaim - 用户信息转换为Claims

**签名**：
```csharp
public List<KeyValuePair<string, string>> UserInfoToClaim(UserRoleMenuDto dto)
```

**功能**：
将用户信息转换为JWT Claims列表。

**Claims列表**：

| Claim类型 | 说明 | 示例值 |
|-----------|------|--------|
| `AbpClaimTypes.UserId` | 用户ID | `"3fa85f64-5717-4562-b3fc-2c963f66afa6"` |
| `AbpClaimTypes.UserName` | 用户名 | `"admin"` |
| `TokenTypeConst.DeptId` | 部门ID | `"dept-001"` |
| `AbpClaimTypes.Email` | 邮箱 | `"user@example.com"` |
| `AbpClaimTypes.PhoneNumber` | 电话号码 | `"13800138000"` |
| `TokenTypeConst.RoleInfo` | 角色信息（JSON） | `"[{"id":"...","dataScope":...}]"` |
| `TokenTypeConst.Permission` | 权限码（管理员） | `"*"` |
| `TokenTypeConst.Roles` | 角色码（管理员） | `"admin"` |

**管理员判断**：
```csharp
// 1. 检查是否为管理员用户名
bool isAdmin = UserConst.Admin.Equals(dto.User.UserName);

// 2. 检查是否有管理员角色
if (!isAdmin && dto.Roles.Any(r => r.RoleName.Contains("管理员") || r.RoleCode == "admin"))
{
    isAdmin = true;
}

// 3. 如果是管理员，添加管理员权限
if (isAdmin)
{
    AddToClaim(claims, TokenTypeConst.Permission, UserConst.AdminPermissionCode);
    AddToClaim(claims, TokenTypeConst.Roles, UserConst.AdminRolesCode);
}
else
{
    // 4. 普通用户，添加实际权限和角色
    dto.PermissionCodes?.ToList()?.ForEach(per => AddToClaim(claims, TokenTypeConst.Permission, per));
    dto.RoleCodes?.ToList()?.ForEach(role => AddToClaim(claims, AbpClaimTypes.Role, role));
}
```

### 5. CreateToken - 创建令牌

**签名**：
```csharp
private string CreateToken(List<KeyValuePair<string, string>> kvs)
```

**功能**：
根据Claims列表创建JWT访问令牌。

**令牌配置**：
```csharp
var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwtOptions.SecurityKey));
var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
var token = new JwtSecurityToken(
   issuer: _jwtOptions.Issuer,
   audience: _jwtOptions.Audience,
   claims: claims,
   expires: DateTime.Now.AddMinutes(_jwtOptions.ExpiresMinuteTime),
   notBefore: DateTime.Now,
   signingCredentials: creds);
```

### 6. CreateRefreshToken - 创建刷新令牌

**签名**：
```csharp
public string CreateRefreshToken(Guid userId)
```

**功能**：
为指定用户创建刷新令牌。

**刷新令牌Claims**：
```csharp
new List<Claim> {
    new Claim(AbpClaimTypes.UserId, userId.ToString()),
    new Claim(TokenTypeConst.Refresh, "true")
};
```

**配置参数**：
- 过期时间：`_refreshJwtOptions.ExpiresMinuteTime`
- 签名算法：HmacSha256

### 7. UpdatePasswordAsync - 更新密码

**签名**：
```csharp
public async Task UpdatePasswordAsync(Guid userId, string newPassword, string oldPassword)
```

**功能**：
更新用户密码。

**执行流程**：

```
1. 查询用户
   │
   ├─ await _repository.GetByIdAsync(userId)
   │
2. 验证旧密码
   │
   ├─ if (!user.JudgePassword(oldPassword))
   │  └─ 抛出 UserFriendlyException("无效更新！原密码错误！")
   │
3. 设置新密码
   │
   ├─ user.EncryPassword.Password = newPassword
   ├─ user.BuildPassword()
   │
4. 保存到数据库
   │
   └─ await _repository.UpdateAsync(user)
```

### 8. RestPasswordAsync - 重置密码

**签名**：
```csharp
public async Task<bool> RestPasswordAsync(Guid userId, string password)
```

**功能**：
重置用户密码（用于找回密码或管理员重置）。

**执行流程**：

```
1. 查询用户
   │
   ├─ await _repository.GetByIdAsync(userId)
   │
2. 设置新密码
   │
   ├─ user.EncryPassword.Password = password
   ├─ user.BuildPassword()
   │
3. 保存到数据库
   │
   └─ return await _repository.UpdateAsync(user)
```

### 9. RegisterAsync - 注册用户

**签名**：
```csharp
public async Task RegisterAsync(
    string userName, 
    string password, 
    long? phone,
    string? nick)
```

**功能**：
注册新用户并分配默认角色。

**执行流程**：

```
1. 创建用户实体
   │
   ├─ new UserAggregateRoot(userName, password, phone, nick)
   │
2. 创建用户
   │
   ├─ await _userManager.CreateAsync(user)
   │
3. 设置默认角色
   │
   └─ await _userManager.SetDefautRoleAsync(user.Id)
```

## 相关服务

- `UserManager` - 用户管理器
- `RoleManager` - 角色管理器

## 相关实体

- `UserAggregateRoot` - 用户聚合根
- `RoleAggregateRoot` - 角色聚合根

## 相关DTO

- `UserRoleMenuDto` - 用户角色菜单数据传输对象

## 配置选项

### JwtOptions

```csharp
public class JwtOptions
{
    public string SecurityKey { get; set; }        // JWT密钥
    public string Issuer { get; set; }             // 发布者
    public string Audience { get; set; }           // 受众
    public int ExpiresMinuteTime { get; set; }    // 过期时间（分钟）
}
```

### RefreshJwtOptions

```csharp
public class RefreshJwtOptions
{
    public string SecurityKey { get; set; }        // 刷新令牌密钥
    public string Issuer { get; set; }             // 发布者
    public string Audience { get; set; }           // 受众
    public int ExpiresMinuteTime { get; set; }    // 过期时间（分钟）
}
```

### RbacOptions

```csharp
public class RbacOptions
{
    // RBAC相关配置
}
```

## 常量定义

### UserConst - 用户常量

```csharp
public class UserConst
{
    public const string Admin = "admin";                          // 管理员用户名
    public const string TenantAdmin = "tenantadmin";                // 租户管理员用户名
    public const string DefaultRoleCode = "user";                  // 默认角色代码
    public const string AdminPermissionCode = "*";                 // 管理员权限码
    public const string AdminRolesCode = "admin";                 // 管理员角色码
    
    // 错误消息
    public const string State_Is_State = "账户状态异常";
    public const string No_Role = "账户未分配角色";
    public const string No_Permission = "账户无任何权限";
    public const string Login_Error = "用户名或密码错误";
    public const string Login_User_No_Exist = "用户不存在";
    public const string Exist = "用户名已存在";
    public const string Create_Passworld_Error = "密码长度不能少于6位";
    public const string Phone_Repeat = "手机号已被使用";
}
```

## 日志记录

AccountManager主要通过异常和依赖的服务进行日志记录。

## 错误处理

### 用户友好异常

所有业务逻辑错误都抛出 `UserFriendlyException`：

```csharp
// 用户状态异常
throw new UserFriendlyException(UserConst.State_Is_State);

// 无角色分配
throw new UserFriendlyException(UserConst.No_Role);

// 无权限分配
throw new UserFriendlyException(UserConst.No_Permission);

// 原密码错误
throw new UserFriendlyException("无效更新！原密码错误！");
```

### 应用异常

```csharp
// 用户不存在
throw new ApplicationException(UserConst.Login_User_No_Exist);

// 密码错误
throw new ApplicationException(UserConst.Login_Error);
```

## 使用场景

### 1. 用户登录

```csharp
await _accountManager.LoginValidationAsync(userName, password);
// 验证成功后获取令牌
var token = await _accountManager.GetTokenByUserIdAsync(userId);
```

### 2. 令牌刷新

```csharp
var refreshToken = _accountManager.CreateRefreshToken(userId);
// 使用刷新令牌获取新的访问令牌
```

### 3. 密码管理

```csharp
// 更新密码
await _accountManager.UpdatePasswordAsync(userId, newPassword, oldPassword);

// 重置密码
await _accountManager.RestPasswordAsync(userId, newPassword);
```

### 4. 用户注册

```csharp
await _accountManager.RegisterAsync(userName, password, phone, nick);
// 自动分配默认角色
```

## 设计特点

1. **Claims封装** - 将用户、角色、权限信息封装到JWT Claims
2. **管理员特权** - 自动识别管理员并授予全部权限
3. **密码安全** - 使用SHA2加密存储密码
4. **令牌双机制** - 访问令牌 + 刷新令牌
5. **容错设计** - 所有业务错误都抛出友好异常
6. **角色自动分配** - 注册时自动分配默认角色

## 安全考虑

1. **密码加密** - 使用SHA2 + Salt加密存储
2. **令牌过期** - 访问令牌和刷新令牌都有过期时间
3. **二次验证** - 用户名存在性检查进行二次校验
4. **权限验证** - 登录时验证角色和权限分配
5. **管理员识别** - 多重机制识别管理员身份

## 性能优化

1. **用户信息缓存** - 通过UserManager的缓存机制
2. ** Claims复用** - UserInfoToClaim生成一次可多次使用
3. **批量角色分配** - GiveUserSetRoleAsync支持批量操作

---

> **最后更新**：2026-06-04  
> **源码位置**：`module/rbac/Yi.Framework.Rbac.Domain/Managers/AccountManager.cs`
> **接口定义**：`module/rbac/Yi.Framework.Rbac.Domain/Managers/IAccountManager.cs`
