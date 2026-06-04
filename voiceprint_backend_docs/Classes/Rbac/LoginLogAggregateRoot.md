# LoginLogAggregateRoot — 登录日志聚合根

## 基本信息

- **实体名称**：`LoginLogAggregateRoot`
- **数据库表**：`LoginLog`
- **模块位置**：`module/rbac/Yi.Framework.Rbac.Domain/Entities/`
- **继承关系**：`AggregateRoot<Guid>` → `ICreationAuditedObject`

## 实体说明

登录日志聚合根用于记录用户登录系统的详细信息，包括登录时间、登录IP、地理位置、浏览器信息等。支持安全审计和异常登录检测。

## 字段说明

### 主键与审计

| 字段名 | 数据类型 | 说明 | 约束 |
|-------|---------|------|------|
| `Id` | `Guid` | 主键 | Primary Key |
| `CreationTime` | `DateTime` | 创建时间（登录时间） | ICreationAuditedObject |
| `CreatorId` | `Guid?` | 创建者ID（用户ID） | ICreationAuditedObject |

### 登录用户信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `LoginUser` | `string?` | 登录用户名 | - |

### 地理位置信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `LoginLocation` | `string?` | 登录地点 | 格式："省份-城市" |
| `LoginIp` | `string?` | 登录IP地址 | - |

### 客户端信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `Browser` | `string?` | 浏览器 | 如："Chrome"、"Firefox" |
| `Os` | `string?` | 操作系统 | 如："Windows 10"、"Android" |
| `LogMsg` | `string?` | 登录信息 | 额外的登录相关信息 |

## 业务规则

### 日志记录规则
1. **自动记录**：用户登录时自动记录登录日志
2. **IP解析**：自动解析IP地址的地理位置
3. **浏览器识别**：通过User-Agent解析浏览器和操作系统信息
4. **失败记录**：登录失败也记录日志（通过LogMsg区分）

### 地理位置解析
- 本地登录（127.0.0.1）显示为"本地-本机"
- 其他IP通过IP工具解析省份和城市
- 解析失败的显示为"IP地址-未知地区"

## 相关服务

- [[LoginLogService]] — 登录日志服务
- [[AccountService]] — 账号服务

## 数据查询示例

### 查询某用户的所有登录记录
```sql
SELECT * FROM LoginLog
WHERE creator_id = 'xxx'
ORDER BY creation_time DESC;
```

### 查询最近的登录记录
```sql
SELECT TOP 10
    login_user,
    login_location,
    login_ip,
    browser,
    os,
    creation_time
FROM LoginLog
ORDER BY creation_time DESC;
```

### 查询异地登录（安全审计）
```sql
-- 查询同一用户在不同地点的登录
SELECT
    creator_id,
    login_user,
    login_location,
    login_ip,
    creation_time
FROM LoginLog
WHERE creator_id = 'xxx'
AND creation_time >= DATEADD(DAY, -7, GETDATE())
GROUP BY creator_id, login_user, login_location, login_ip
HAVING COUNT(DISTINCT login_location) > 1;
```

### 统计登录次数
```sql
SELECT
    login_user,
    COUNT(*) as login_count,
    MAX(creation_time) as last_login
FROM LoginLog
WHERE creation_time >= DATEADD(DAY, -30, GETDATE())
GROUP BY login_user
ORDER BY login_count DESC;
```

## 索引建议

```sql
-- 登录用户索引
CREATE INDEX index_LoginUser ON LoginLog(login_user);
```

## 使用场景

### 场景1：正常登录记录
```
登录用户：张三
登录时间：2026-06-04 09:15:30
登录地点：北京-北京市
登录IP：192.168.1.100
浏览器：Chrome
操作系统：Windows 10
```

### 场景2：异地登录检测
```
用户：李四
最近登录：
- 2026-06-04 08:30:00 - 上海-上海市 - 200.100.x.x
- 2026-06-03 18:45:00 - 北京-北京市 - 192.168.1.x

⚠️ 异地登录警告！
```

### 场景3：失败登录记录
```
登录用户：王五
登录时间：2026-06-04 10:20:15
登录地点：未知地区
登录IP：203.0.113.45
登录信息：密码错误，登录失败
```

## 业务价值

**核心作用**：
1. **安全审计**：记录所有登录行为，支持安全审计
2. **异常检测**：检测异地登录、异常IP等安全问题
3. **用户行为分析**：分析用户登录习惯和活跃度
4. **故障排查**：通过登录日志排查登录问题

## 安全功能

### 异地登录告警
```csharp
public async Task CheckAbnormalLoginAsync(LoginLogAggregateRoot loginLog)
{
    // 查询该用户最近7天的登录记录
    var recentLogs = await _repository.GetListAsync(l => 
        l.CreatorId == loginLog.CreatorId &&
        l.CreationTime >= DateTime.Now.AddDays(-7)
    );

    // 检查是否有异地登录
    var locations = recentLogs
        .Select(l => l.LoginLocation)
        .Distinct()
        .ToList();

    if (locations.Count > 1)
    {
        // 发送异地登录告警
        await _notificationService.SendAsync(
            loginLog.CreatorId,
            "异地登录警告",
            $"检测到您的账号在 {loginLog.LoginLocation} 登录，如非本人操作请及时修改密码。"
        );
    }
}
```

### 登录失败统计
```csharp
public async Task CheckFailedLoginAsync(string username, string ip)
{
    // 查询最近1小时的失败登录次数
    var failedCount = await _repository.CountAsync(l =>
        l.LoginUser == username &&
        l.LogMsg.Contains("失败") &&
        l.CreationTime >= DateTime.Now.AddHours(-1)
    );

    if (failedCount >= 5)
    {
        // 锁定账号或IP
        await _securityService.LockAccountAsync(username, ip);
    }
}
```

## 设计模式

### 工厂方法模式
```csharp
// 从HttpContext创建登录日志
public LoginLogAggregateRoot GetInfoByHttpContext(HttpContext context)
{
    var ipAddr = context.GetClientIp();
    var location = GetLocationByIp(ipAddr);
    var userAgent = ParseUserAgent(context);

    return new LoginLogAggregateRoot
    {
        LoginIp = ipAddr,
        LoginLocation = location.Province + "-" + location.City,
        Browser = userAgent.Device.Family,
        Os = userAgent.OS.ToString()
    };
}
```

### 单一职责
- 负责记录登录日志
- IP解析、浏览器解析等职责委托给专用工具
- 保持实体简单，专注于数据存储

## 性能考虑

### IP解析优化
```csharp
// 缓存IP解析结果
private readonly IMemoryCache _cache;

public IpInfo GetLocationByIp(string ip)
{
    var cacheKey = $"ip_location_{ip}";
    
    return _cache.GetOrCreate(cacheKey, () =>
    {
        return IpTool.Search(ip);
    }, TimeSpan.FromHours(24));
}
```

### 批量查询优化
```csharp
// 使用分页查询历史登录记录
public async Task<PagedResultDto<LoginLogAggregateRoot>> GetPagedAsync(
    Guid userId,
    int pageIndex,
    int pageSize)
{
    var query = await _repository.GetQueryableAsync();
    
    query = query.Where(l => l.CreatorId == userId)
               .OrderByDescending(l => l.CreationTime);
    
    return await query.ToPagedListAsync(pageIndex, pageSize);
}
```

## 隐私考虑

### 数据脱敏
- 登录IP地址可以根据需要进行脱敏处理
- 敏感用户的登录日志可以设置更短的保留期
- 导出日志时需要注意用户隐私保护

### 数据保留策略
- **重要账号**：永久保留登录日志
- **普通账号**：保留6-12个月
- **测试账号**：保留1个月

---

> **最后更新**：2026-06-04
> **源码位置**：`module/rbac/Yi.Framework.Rbac.Domain/Entities/LoginLogAggregateRoot.cs`
