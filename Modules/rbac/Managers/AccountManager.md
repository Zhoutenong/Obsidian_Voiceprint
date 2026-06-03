# AccountManager

**路径**: `module/rbac/Yi.Framework.Rbac.Domain/Managers/AccountManager.cs`

**依赖**: `DomainService` (Scoped)

## 概述

账号管理器，负责用户认证、Token 生成和密码管理。是 RBAC 系统的安全核心，处理登录、注册、密码重置和 JWT 令牌颁发。

## 核心功能

### 1. 登录验证

```csharp
public async Task LoginValidationAsync(
    string userName, 
    string password, 
    Action<UserAggregateRoot>? userAction = null)
```

**职责**：
- 验证用户名和密码
- 检查用户状态（是否禁用）
- 触发用户实体回调
- 验证失败时抛出异常

**验证流程**：
```
查询用户 → 验证密码 → 检查状态 → 触发回调
```

### 2. Token 生成

```csharp
public async Task<string> GetTokenByUserIdAsync(
    Guid userId, 
    Action<UserRoleMenuDto>? getUserInfo = null)
```

**职责**：
- 获取用户完整信息（角色、菜单、权限）
- 验证用户状态和权限
- 生成 JWT 访问令牌
- 触发用户信息回调

**验证检查**：
```csharp
if (userInfo.User.State == false)
    throw new UserFriendlyException(UserConst.State_Is_State);

if (userInfo.RoleCodes.Count == 0)
    throw new UserFriendlyException(UserConst.No_Role);

if (!userInfo.PermissionCodes.Any())
    throw new UserFriendlyException(UserConst.No_Permission);
```

### 3. 用户注册

```csharp
public async Task RegisterAsync(
    string userName, 
    string password, 
    long? phone, 
    string? nick)
```

**职责**：
- 验证用户名唯一性
- 加密密码
- 创建用户实体
- 分配默认角色

**密码加密**：
```csharp
var salt = Guid.NewGuid().ToString("N");
var encryptedPassword = MD5Encrypt.Encrypt(password + salt);
```

### 4. 密码重置

```csharp
public async Task<bool> RestPasswordAsync(Guid userId, string password)
```

**职责**：
- 生成新的盐值
- 加密新密码
- 更新用户密码

### 5. 密码修改

```csharp
public async Task UpdatePasswordAsync(
    Guid userId, 
    string newPassword, 
    string oldPassword)
```

**职责**：
- 验证旧密码
- 生成新的盐值
- 加密新密码
- 更新用户密码

### 6. 刷新令牌生成

```csharp
public string CreateRefreshToken(Guid userId)
```

**职责**：
- 使用独立的 JWT 配置
- 生成长期有效的刷新令牌
- 用于访问令牌过期后刷新

## JWT 配置

### JwtOptions (访问令牌)

```json
{
  "Jwt": {
    "Issuer": "IntelliSub",
    "Audience": "IntelliSub.Client",
    "Expire": 43200,
    "Secret": "your-secret-key"
  }
}
```

**字段说明**：
- `Issuer` - 颁发者标识
- `Audience` - 受众标识
- `Expire` - 过期时间（秒，默认 12 小时）
- `Secret` - 签名密钥

### RefreshJwtOptions (刷新令牌)

```json
{
  "RefreshJwt": {
    "Issuer": "IntelliSub",
    "Audience": "IntelliSub.Client",
    "Expire": 604800,
    "Secret": "your-refresh-secret-key"
  }
}
```

**字段说明**：
- `Expire` - 过期时间（秒，默认 7 天）

## 令牌生成流程

### 1. 用户信息获取

```csharp
var userInfo = await _userManager.GetInfoAsync(userId);
```

### 2. 声明构建

```csharp
private List<Claim> UserInfoToClaim(UserRoleMenuDto userInfo)
{
    var claims = new List<Claim>
    {
        new Claim(ClaimTypes.NameIdentifier, userInfo.User.Id.ToString()),
        new Claim(ClaimTypes.Name, userInfo.User.UserName),
        new Claim(ClaimTypes Nick, userInfo.User.Nick ?? string.Empty),
        new Claim("role_codes", string.Join(",", userInfo.RoleCodes)),
        new Claim("permission_codes", string.Join(",", userInfo.PermissionCodes))
    };
    
    AddToClaim(claims, userInfo.Roles);
    return claims;
}
```

### 3. 令牌创建

```csharp
private string CreateToken(List<Claim> claims, JwtOptions options)
{
    var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(options.Secret));
    var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
    
    var token = new JwtSecurityToken(
        issuer: options.Issuer,
        audience: options.Audience,
        claims: claims,
        expires: DateTime.UtcNow.AddSeconds(options.Expire),
        signingCredentials: credentials
    );
    
    return new JwtSecurityTokenHandler().WriteToken(token);
}
```

## 密码加密

### MD5 加密方案

```csharp
public class MD5Encrypt
{
    public static string Encrypt(string input)
    {
        using var md5 = MD5.Create();
        var inputBytes = Encoding.UTF8.GetBytes(input);
        var hashBytes = md5.ComputeHash(inputBytes);
        
        return BitConverter.ToString(hashBytes).Replace("-", "").ToLower();
    }
}
```

### 密码存储格式

```csharp
public class EncryPassword
{
    public string Password { get; set; }  // MD5(password + salt)
    public string Salt { get; set; }      // 随机盐值
}
```

### 密码验证

```csharp
private bool VerifyPassword(UserAggregateRoot user, string password)
{
    var encrypted = MD5Encrypt.Encrypt(password + user.EncryPassword.Salt);
    return encrypted == user.EncryPassword.Password;
}
```

## 使用示例

### 登录流程

```csharp
public async Task<LoginOutputDto> LoginAsync(LoginInputVo input)
{
    // 1. 验证用户名密码
    UserAggregateRoot user = new();
    await _accountManager.LoginValidationAsync(input.UserName, input.Password, 
        x => user = x);
    
    // 2. 生成令牌
    var userInfo = new UserRoleMenuDto();
    var accessToken = await _accountManager.GetTokenByUserIdAsync(
        user.Id, 
        (info) => userInfo = info
    );
    
    // 3. 生成刷新令牌
    var refreshToken = _accountManager.CreateRefreshToken(user.Id);
    
    // 4. 返回登录结果
    return new LoginOutputDto 
    { 
        Token = accessToken, 
        RefreshToken = refreshToken,
        UserName = userInfo.User.UserName 
    };
}
```

### 用户注册

```csharp
public async Task RegisterAsync(RegisterDto input)
{
    await _accountManager.RegisterAsync(
        input.UserName,
        input.Password,
        input.Phone,
        input.Nick
    );
    
    // 注册后自动登录
    return await PostLoginAsync(new LoginInputVo
    {
        UserName = input.UserName,
        Password = input.Password
    });
}
```

### 密码重置

```csharp
public async Task ResetPasswordAsync(Guid userId, string newPassword)
{
    var success = await _accountManager.RestPasswordAsync(userId, newPassword);
    if (!success)
    {
        throw new UserFriendlyException("密码重置失败");
    }
}
```

### 修改密码

```csharp
public async Task ChangePasswordAsync(
    Guid userId, 
    string oldPassword, 
    string newPassword)
{
    await _accountManager.UpdatePasswordAsync(userId, newPassword, oldPassword);
}
```

## 令牌刷新

### 刷新流程

```csharp
public async Task<LoginOutputDto> RefreshTokenAsync(string refreshToken)
{
    // 1. 验证刷新令牌
    var tokenHandler = new JwtSecurityTokenHandler();
    var principal = tokenHandler.ValidateToken(refreshToken, 
        _refreshTokenValidationParameters, out _);
    
    var userId = Guid.Parse(principal.FindFirst(ClaimTypes.NameIdentifier)?.Value!);
    
    // 2. 生成新的访问令牌
    var userInfo = new UserRoleMenuDto();
    var accessToken = await _accountManager.GetTokenByUserIdAsync(
        userId,
        (info) => userInfo = info
    );
    
    // 3. 生成新的刷新令牌
    var newRefreshToken = _accountManager.CreateRefreshToken(userId);
    
    return new LoginOutputDto
    {
        Token = accessToken,
        RefreshToken = newRefreshToken,
        UserName = userInfo.User.UserName
    };
}
```

## 安全特性

### 盐值保护

```csharp
var salt = Guid.NewGuid().ToString("N");  // 32 字符随机字符串
```

### 密码复杂度

建议在应用层添加密码复杂度验证：
```csharp
if (password.Length < 8)
    throw new UserFriendlyException("密码长度至少 8 位");

if (!password.Any(char.IsDigit))
    throw new UserFriendlyException("密码必须包含数字");
```

### 令牌过期

- 访问令牌：12 小时
- 刷新令牌：7 天

### 失效令牌

```csharp
// 将令牌加入黑名单（可选）
await _tokenBlacklist.AddAsync(token, DateTime.UtcNow.AddHours(12));
```

## 错误处理

### 用户不存在

```csharp
if (user is null)
{
    throw new UserFriendlyException("用户名或密码错误");
}
```

### 密码错误

```csharp
if (!VerifyPassword(user, password))
{
    throw new UserFriendlyException("用户名或密码错误");
}
```

### 用户禁用

```csharp
if (userInfo.User.State == false)
{
    throw new UserFriendlyException(UserConst.State_Is_State);
}
```

## 依赖注入

### 构造函数

```csharp
public AccountManager(
    IUserRepository repository,
    IOptions<JwtOptions> jwtOptions,
    ILocalEventBus localEventBus,
    UserManager userManager,
    IOptions<RefreshJwtOptions> refreshJwtOptions,
    ISqlSugarRepository<RoleAggregateRoot> roleRepository,
    IOptions<RbacOptions> options)
```

## 事件发布

### 登录事件

```csharp
public async Task<LoginOutputDto> PostLoginAsync(Guid userId)
{
    // ... 生成令牌后
    
    var loginEntity = new LoginLogAggregateRoot()
        .GetInfoByHttpContext(_httpContextAccessor.HttpContext);
    
    var loginEto = loginEntity.Adapt<LoginEventArgs>();
    loginEto.UserName = userInfo.User.UserName;
    loginEto.UserId = userInfo.User.Id;
    
    await _localEventBus.PublishAsync(loginEto);
    
    return new LoginOutputDto { Token = accessToken };
}
```

## 监控指标

### 登录成功

```csharp
_logger.LogInformation("用户登录成功: UserName={UserName}, UserId={UserId}", 
    userName, userId);
```

### 登录失败

```csharp
_logger.LogWarning("登录失败: UserName={UserName}, Reason=PasswordError", userName);
```

### 令牌颁发

```csharp
_logger.LogInformation("令牌已颁发: UserId={UserId}, Expires={Expires}", 
    userId, DateTime.UtcNow.AddSeconds(_jwtOptions.Expire));
```

## 相关文档

- [UserManager](./UserManager.md) - 用户管理器
- [JWT 认证](../Security/JwtAuthentication.md)
- [权限验证](../Security/Authorization.md)
