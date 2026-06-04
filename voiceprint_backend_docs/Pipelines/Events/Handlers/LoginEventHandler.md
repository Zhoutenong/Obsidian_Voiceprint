# LoginEventHandler — 登录事件处理器

## 基本信息

- **处理器名称**：`LoginEventHandler`
- **模块位置**：`module/rbac/Yi.Framework.Rbac.Domain/EventHandlers/`
- **事件类型**：`LoginEventArgs` (ILocalEventHandler)
- **生命周期**：`ITransientDependency` - 瞬态依赖
- **命名空间**：`Yi.Framework.Rbac.Domain.EventHandlers`

## 处理器概述

LoginEventHandler 负责处理用户登录事件，在用户成功登录系统后，将登录信息记录到登录日志表中。这是一个RBAC模块的领域事件处理器，用于审计用户的登录行为。

## 核心职责

1. **登录日志记录** - 记录用户登录信息到数据库
2. **审计追踪** - 保存用户登录的详细记录
3. **异步处理** - 异步插入登录日志，不影响登录流程

## 依赖注入

| 依赖 | 职责 |
|------|------|
| `IRepository<LoginLogAggregateRoot>` | 登录日志仓储 |
| `ILogger<LoginEventHandler>` | 日志记录器 |

## 事件数据结构

### LoginEventArgs

```csharp
public class LoginEventArgs
{
    public Guid UserId { get; set; }        // 用户ID
    public string UserName { get; set; }    // 用户名
    // 可能还包含其他登录相关信息
}
```

## 处理流程

```
1. 记录事件日志
   │
   ├─ 输出：用户ID、用户名
   │
2. 转换事件数据
   │
   ├─ 使用Mapster将EventArgs转换为Entity
   │
3. 补充实体信息
   │
   ├─ 设置LogMsg: "{UserName}登录系统"
   ├─ 设置LoginUser: UserName
   └─ 设置CreatorId: UserId
   │
4. 异步插入
   │
   └─ 插入到登录日志表
```

## 核心方法

### HandleEventAsync - 处理登录事件

**签名**：
```csharp
public async Task HandleEventAsync(LoginEventArgs eventData)
```

**功能**：
处理用户登录事件，记录登录日志。

**完整实现**：

```csharp
public async Task HandleEventAsync(LoginEventArgs eventData)
{
    // 1. 记录事件日志
    _logger.LogInformation($"用户【{eventData.UserId}:{eventData.UserName}】登入系统");
    
    // 2. 转换事件数据为实体
    var loginLogEntity = eventData.Adapt<LoginLogAggregateRoot>();
    
    // 3. 补充实体信息
    loginLogEntity.LogMsg = eventData.UserName + "登录系统";
    loginLogEntity.LoginUser = eventData.UserName;
    loginLogEntity.CreatorId = eventData.UserId;
    
    // 4. 异步插入
    await _loginLogRepository.InsertAsync(loginLogEntity);
}
```

## 实体转换

### 使用Mapster进行对象映射

```csharp
var loginLogEntity = eventData.Adapt<LoginLogAggregateRoot>();
```

**自动映射的字段**（假设）：
- `UserId` → 登录用户ID
- `UserName` → 登录用户名
- 可能还包含登录时间、IP地址等

**手动设置的字段**：
```csharp
loginLogEntity.LogMsg = eventData.UserName + "登录系统";
loginLogEntity.LoginUser = eventData.UserName;
loginLogEntity.CreatorId = eventData.UserId;
```

## 日志记录

### 信息日志 (LogInformation)

```csharp
// 用户登录事件
$"用户【{UserId}:{UserName}】登入系统"

// 示例输出
"用户【3fa85f64-5717-4562-b3fc-2c963f66afa6:admin】登入系统"
```

## 相关实体

- [[LoginLogAggregateRoot]] - 登录日志聚合根

## 实体字段说明

### LoginLogAggregateRoot

根据处理器实现，登录日志实体包含以下字段：

| 字段名 | 类型 | 说明 | 来源 |
|--------|------|------|------|
| `UserId` | Guid | 登录用户ID | eventData.UserId |
| `LoginUser` | string | 登录用户名 | eventData.UserName |
| `LogMsg` | string | 日志消息 | "{UserName}登录系统" |
| `CreatorId` | Guid | 创建者ID | eventData.UserId |
| `CreationTime` | DateTime | 创建时间 | 系统自动设置 |

## 使用场景

### 1. 用户名密码登录

用户通过用户名和密码登录：
```csharp
var eventArgs = new LoginEventArgs
{
    UserId = Guid.Parse("3fa85f64-5717-4562-b3fc-2c963f66afa6"),
    UserName = "admin"
};

// 触发事件后，处理器会记录：
// LogMsg = "admin登录系统"
// LoginUser = "admin"
// CreatorId = 3fa85f64-5717-4562-b3fc-2c963f66afa6
```

### 2. OAuth登录

用户通过OAuth（QQ、Gitee）登录：
```csharp
var eventArgs = new LoginEventArgs
{
    UserId = userId,
    UserName = oauthUserNickname
};

// 同样记录登录日志
```

### 3. Token刷新登录

用户使用Refresh Token获取新Token：
```csharp
// 通常不触发登录事件，或者单独标识
```

## 审计功能

### 登录历史查询

通过登录日志表，可以查询：

1. **用户登录历史**
   - 查询某用户的所有登录记录
   - 分析用户登录频率

2. **安全审计**
   - 检测异常登录行为
   - 追踪可疑账户活动

3. **系统使用统计**
   - 统计活跃用户数量
   - 分析系统使用高峰期

### 典型查询示例

```sql
-- 查询某用户的登录历史
SELECT * FROM YiLoginLog 
WHERE login_user = 'admin' 
ORDER BY creation_time DESC 
LIMIT 10;

-- 查询最近7天的登录统计
SELECT DATE(creation_time) as login_date, 
       COUNT(DISTINCT login_user) as unique_users,
       COUNT(*) as total_logins
FROM YiLoginLog
WHERE creation_time >= DATE('now', '-7 days')
GROUP BY DATE(creation_time);
```

## 性能考虑

1. **异步处理** - 使用Async方法，不阻塞登录流程
2. **数据库插入** - 单条插入，性能影响小
3. **索引建议** - 在`LoginUser`和`CreationTime`字段上建立索引

## 设计特点

1. **简单直接** - 逻辑简单，只做数据转换和插入
2. **异步非阻塞** - 异步处理，不影响登录性能
3. **审计完整** - 记录完整的登录信息
4. **自动映射** - 使用Mapster自动映射对象

## 错误处理

当前实现没有显式的错误处理。建议改进：

```csharp
public async Task HandleEventAsync(LoginEventArgs eventData)
{
    try
    {
        _logger.LogInformation($"用户【{eventData.UserId}:{eventData.UserName}】登入系统");
        
        var loginLogEntity = eventData.Adapt<LoginLogAggregateRoot>();
        loginLogEntity.LogMsg = eventData.UserName + "登录系统";
        loginLogEntity.LoginUser = eventData.UserName;
        loginLogEntity.CreatorId = eventData.UserId;
        
        await _loginLogRepository.InsertAsync(loginLogEntity);
    }
    catch (Exception ex)
    {
        // 登录日志记录失败不应影响登录流程
        _logger.LogError(ex, "记录登录日志失败: UserId={UserId}, UserName={UserName}", 
            eventData.UserId, eventData.UserName);
    }
}
```

## 扩展建议

### 1. 记录更多登录信息

```csharp
public class LoginEventArgs
{
    public Guid UserId { get; set; }
    public string UserName { get; set; }
    public string IpAddress { get; set; }        // 新增：IP地址
    public string UserAgent { get; set; }        // 新增：浏览器信息
    public LoginTypeEnum LoginType { get; set; } // 新增：登录类型
}

public enum LoginTypeEnum
{
    Password = 1,      // 密码登录
    OAuth = 2,         // OAuth登录
    Token = 3          // Token登录
}
```

### 2. 登录失败记录

建议增加登录失败事件处理器：
```csharp
public class LoginFailedEventHandler : ILocalEventHandler<LoginFailedEventArgs>
{
    // 记录登录失败信息，用于安全审计
}
```

### 3. 登出记录

建议增加登出事件处理器：
```csharp
public class LogoutEventHandler : ILocalEventHandler<LogoutEventArgs>
{
    // 记录用户登出时间，计算在线时长
}
```

## 相关事件

- `LoginEventArgs` - 登录事件参数
- `LoginFailedEventArgs` - 登录失败事件（建议新增）
- `LogoutEventArgs` - 登出事件（建议新增）

## 相关服务

- `AccountService` - 账户服务（可能触发登录事件）
- `AuthService` - 认证服务（可能触发登录事件）

---

> **最后更新**：2026-06-04  
> **源码位置**：`module/rbac/Yi.Framework.Rbac.Domain/EventHandlers/LoginEventHandler.cs`
