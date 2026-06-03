# Yi.Framework.BackgroundWorkers.Hangfire

BackgroundWorkers.Hangfire 模块提供基于 Hangfire 的后台任务调度和管理，支持 Cron 表达式配置、内存/Redis 存储和 ABP 工作单元集成。

## 核心组件

### 1. Worker 基类

#### HangfireBackgroundWorker

后台任务的抽象基类，所有定时任务继承此类：

**核心属性：**
```csharp
public abstract class HangfireBackgroundWorker : IHangfireBackgroundWorker
{
    // 任务唯一标识（用于 Hangfire 作业管理）
    public string RecurringJobId { get; set; }

    // Cron 表达式（定义执行频率）
    public string CronExpression { get; set; }

    // 时区（默认服务器本地时区）
    public TimeZoneInfo TimeZone { get; set; }

    // 执行任务的核心方法
    public abstract Task DoWorkAsync(CancellationToken cancellationToken = default);
}
```

**使用示例：**
```csharp
public class VoiceprintCaptureJob : HangfireBackgroundWorker
{
    public VoiceprintCaptureJob()
    {
        RecurringJobId = "voiceprint-capture";
        CronExpression = "0 */10 * * * *"; // 每 10 分钟
    }

    public override async Task DoWorkAsync(CancellationToken cancellationToken)
    {
        // 任务执行逻辑
        await CaptureAudioFromEdgeDevicesAsync();
    }
}
```

### 2. 工作单元过滤器

#### UnitOfWorkHangfireFilter

确保 Hangfire 任务在 ABP 工作单元中执行，提供事务支持：

**位置：** `framework/Yi.Framework.BackgroundWorkers.Hangfire/UnitOfWorkHangfireFilter.cs`

```csharp
public class UnitOfWorkHangfireFilter : IServerFilter
{
    public void OnPerforming(PerformingContext context)
    {
        // 创建新的服务作用域
        var scope = _serviceProvider.CreateScope();
        context.Items.Add(CurrentJobScope, scope);

        // 开始工作单元
        var unitOfWorkManager = scope.ServiceProvider
            .GetRequiredService<IUnitOfWorkManager>();
        var uow = unitOfWorkManager.Begin();
        context.Items.Add(CurrentJobUow, uow);
    }

    public void OnPerformed(PerformedContext context)
    {
        // 完成或回滚工作单元
        if (context.Exception == null && !uow.IsCompleted)
        {
            await uow.CompleteAsync();
        }
        else
        {
            await uow.RollbackAsync();
        }

        // 释放资源
        uow?.Dispose();
        scope?.Dispose();
    }
}
```

**关键特性：**
- 独立服务作用域（避免依赖注入冲突）
- 自动事务管理（成功提交、失败回滚）
- 资源清理（防止内存泄漏）

### 3. 自动注册器

#### YiHangfireConventionalRegistrar

自动发现并注册 `IHangfireBackgroundWorker` 实现：

```csharp
public class YiHangfireConventionalRegistrar : DefaultConventionalRegistrar
{
    protected override bool IsConventionalRegistrationDisabled(Type type)
    {
        return !typeof(IHangfireBackgroundWorker).IsAssignableFrom(type) ||
               base.IsConventionalRegistrationDisabled(type);
    }

    protected override List<Type> GetExposedServiceTypes(Type type)
    {
        return new List<Type>() { typeof(IHangfireBackgroundWorker) };
    }
}
```

### 4. Token 认证过滤器

#### YiTokenAuthorizationFilter

为 Hangfire Dashboard 提供基于 JWT Token 的认证：

```csharp
public class YiTokenAuthorizationFilter : IDashboardAuthorizationFilter
{
    public bool Authorize(DashboardContext context)
    {
        var httpContext = context.GetHttpContext();
        var _currentUser = _serviceProvider.GetRequiredService<ICurrentUser>();

        if (_currentUser.IsAuthenticated)
        {
            // 设置 Token Cookie
            var authorization = httpContext.Request.Headers["Authorization"].ToString();
            if (!string.IsNullOrWhiteSpace(authorization))
            {
                var token = authorization.Substring(Bearer.Length - 1);
                httpContext.Response.Cookies.Append("Token", token, cookieOptions);
            }

            // 验证用户权限
            if (_currentUser.UserName == RequireUser)
            {
                return true;
            }
        }

        SetChallengeResponse(httpContext);
        return false;
    }
}
```

**配置方法：**
```csharp
services.AddHangfireServer(options =>
{
    options.Authorization = new[] { new YiTokenAuthorizationFilter(serviceProvider)
        .SetRequireUser("admin")
        .SetExpiresTime(TimeSpan.FromMinutes(10))
    };
});
```

## 任务注册流程

### YiFrameworkBackgroundWorkersHangfireModule

**位置：** `framework/Yi.Framework.BackgroundWorkers.Hangfire/YiFrameworkBackgroundWorkersHangfireModule.cs`

#### 1. PreConfigureServices - 注册约定

```csharp
public override void PreConfigureServices(ServiceConfigurationContext context)
{
    context.Services.AddConventionalRegistrar(new YiHangfireConventionalRegistrar());
}
```

#### 2. OnApplicationInitialization - 自动注册任务

```csharp
public override async Task OnApplicationInitializationAsync(ApplicationInitializationContext context)
{
    var backgroundWorkerManager = context.ServiceProvider
        .GetRequiredService<IBackgroundWorkerManager>();
    var works = context.ServiceProvider.GetServices<IHangfireBackgroundWorker>();

    // 检查是否启用 Redis（决定使用队列还是内存存储）
    bool.TryParse(configuration["Redis:IsEnabled"], out var redisEnabled);

    foreach (var work in works)
    {
        // 检查任务是否应该被注册（通过配置控制）
        if (!ShouldRegisterJob(work, configuration, logger))
        {
            logger.LogInformation("跳过注册后台任务: {JobId} (已通过配置禁用)", work.RecurringJobId);
            RecurringJob.RemoveIfExists(work.RecurringJobId);
            continue;
        }

        // 设置时区
        work.TimeZone = TimeZoneInfo.Local;

        if (redisEnabled)
        {
            // Redis 模式：使用队列和后台任务管理器
            await backgroundWorkerManager.AddAsync(work);
        }
        else
        {
            // 内存模式：直接使用 RecurringJob
            var unProxyWorker = ProxyHelper.UnProxy((object)work);
            RecurringJob.AddOrUpdate(
                work.RecurringJobId,
                (Expression<Func<Task>>)(() =>
                    ((IHangfireBackgroundWorker)unProxyWorker).DoWorkAsync(default(CancellationToken))
                ),
                work.CronExpression,
                new RecurringJobOptions() { TimeZone = work.TimeZone }
            );
        }

        logger.LogInformation("成功注册后台任务: {JobId}, Cron: {Cron}", work.RecurringJobId, work.CronExpression);
    }
}
```

### 任务启用/禁用配置

```csharp
private bool ShouldRegisterJob(IHangfireBackgroundWorker worker, IConfiguration configuration, ILogger logger)
{
    var jobId = worker.RecurringJobId;
    var enabledKey = $"BackgroundJobs:{jobId}:Enabled";

    var enabledConfig = configuration[enabledKey];
    if (bool.TryParse(enabledConfig, out var isEnabled))
    {
        return isEnabled;
    }

    // 默认启用
    return true;
}
```

**配置示例：**
```json
{
  "BackgroundJobs": {
    "voiceprint-capture": {
      "Enabled": true
    },
    "voiceprint-processed-cleanup": {
      "Enabled": false
    }
  }
}
```

## Cron 表达式

### 基本格式

```
秒 分 时 日 月 周
*  *  *  *  *  *
```

### 常用表达式

| 表达式 | 说明 | 示例时间 |
|--------|------|----------|
| `0 * * * * *` | 每分钟 | 10:00:00, 10:01:00, ... |
| `0 */5 * * * *` | 每 5 分钟 | 10:00:00, 10:05:00, ... |
| `0 0 * * * *` | 每小时 | 10:00:00, 11:00:00, ... |
| `0 0 */2 * * *` | 每 2 小时 | 10:00:00, 12:00:00, ... |
| `0 0 0 * * *` | 每天 00:00 | 每天 00:00:00 |
| `0 0 0 * * 1` | 每周一 00:00 | 每周一 00:00:00 |
| `0 0 0 1 * *` | 每月 1 日 00:00 | 每月 1 日 00:00:00 |
| `0 30 9 * * 1-5` | 工作日 09:30 | 工作日 09:30:00 |

### 项目内置任务

#### VoiceprintCaptureJob（声纹采集）

```csharp
public class VoiceprintCaptureJob : HangfireBackgroundWorker
{
    public VoiceprintCaptureJob()
    {
        RecurringJobId = "voiceprint-capture";
        CronExpression = "0 */10 * * * *"; // 每 10 分钟
    }

    public override async Task DoWorkAsync(CancellationToken cancellationToken)
    {
        // 从边缘设备采集音频
        await CaptureAudioAsync();
    }
}
```

**配置：**
```json
{
  "VoiceprintCaptureJob": {
    "Enabled": true,
    "CronExpression": "0 */10 * * * *"
  }
}
```

#### VoiceprintProcessedCleanupJob（音频清理）

```csharp
public class VoiceprintProcessedCleanupJob : HangfireBackgroundWorker
{
    public VoiceprintProcessedCleanupJob()
    {
        RecurringJobId = "voiceprint-processed-cleanup";
        CronExpression = "0 0 2 * * *"; // 每天凌晨 2 点
    }

    public override async Task DoWorkAsync(CancellationToken cancellationToken)
    {
        // 清理已处理的音频文件
        await CleanupProcessedFilesAsync();
    }
}
```

#### PointDataCleanupJob（传感器数据清理）

```csharp
public class PointDataCleanupJob : HangfireBackgroundWorker
{
    public PointDataCleanupJob()
    {
        RecurringJobId = "point-data-cleanup";
        CronExpression = "0 0 3 * * *"; // 每天凌晨 3 点
    }

    public override async Task DoWorkAsync(CancellationToken cancellationToken)
    {
        // 清除旧传感器数据（根据保留期配置）
        await CleanupOldPointDataAsync();
    }
}
```

**配置：**
```json
{
  "PointDataCleanupJob": {
    "Enabled": true,
    "RetainDays": 30
  }
}
```

## 存储模式

### 内存存储（默认）

**适用场景：** 单实例部署、开发环境

```json
{
  "Hangfire": {
    "StorageMode": "Memory"
  }
}
```

**配置：**
```csharp
services.AddHangfire(config =>
{
    config.UseSimpleAssemblyNameTypeSerializer()
        .UseRecommendedSerializerSettings()
        .UseMemoryStorage(); // 内存存储
});
```

### SQLite 存储

**适用场景：** 单实例部署、任务持久化需求

```json
{
  "Hangfire": {
    "StorageMode": "SQLite",
    "ConnectionString": "Data Source=db/hangfire.db"
  }
}
```

### Redis 存储

**适用场景：** 多实例部署、高并发环境

```json
{
  "Hangfire": {
    "StorageMode": "Redis",
    "ConnectionString": "localhost:6379,defaultDatabase=1"
  },
  "Redis": {
    "IsEnabled": true
  }
}
```

**配置：**
```csharp
services.AddHangfire(config =>
{
    config.UseSimpleAssemblyNameTypeSerializer()
        .UseRecommendedSerializerSettings()
        .UseRedisStorage(connectionString, new RedisStorageOptions()
        {
            Db = 1,
            Prefix = "hangfire:"
        });
});
```

## Hangfire Dashboard

### 配置

```csharp
app.UseHangfireDashboard("/hangfire", new DashboardOptions
{
    Authorization = new[] { new YiTokenAuthorizationFilter(serviceProvider) }
});
```

### 访问方式

1. **Token 认证访问**
   - 请求头：`Authorization: Bearer <your-jwt-token>`
   - URL：`http://localhost:5000/hangfire`

2. **Cookie 认证访问**
   - 登录后自动设置 Token Cookie
   - 直接访问：`http://localhost:5000/hangfire`

### 功能

- **监控任务**：查看所有定期任务的状态
- **执行日志**：查看任务执行历史和结果
- **手动触发**：立即执行任意任务
- **失败重试**：查看失败任务并手动重试
- **任务统计**：查看任务执行频率和耗时

## 任务依赖注入

### 构造函数注入

```csharp
public class VoiceprintCaptureJob : HangfireBackgroundWorker
{
    private readonly IVoiceprintCollectionService _collectionService;
    private readonly ILogger<VoiceprintCaptureJob> _logger;

    public VoiceprintCaptureJob(
        IVoiceprintCollectionService collectionService,
        ILogger<VoiceprintCaptureJob> logger)
    {
        _collectionService = collectionService;
        _logger = logger;
    }

    public override async Task DoWorkAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("开始声纹采集任务");
        await _collectionService.CollectFromEdgeDevicesAsync(cancellationToken);
    }
}
```

### 服务解析

Hangfire 任务在独立的服务作用域中执行，通过 `UnitOfWorkHangfireFilter` 创建：

```csharp
var scope = _serviceProvider.CreateScope();
context.Items.Add(CurrentJobScope, scope);

var unitOfWorkManager = scope.ServiceProvider
    .GetRequiredService<IUnitOfWorkManager>();
```

## 并发控制

### 禁用并发（默认）

Hangfire 默认禁用任务并发，同一任务不会重叠执行：

```csharp
[DisableConcurrentExecution(timeoutInSeconds: 60)]
public class VoiceprintCaptureJob : HangfireBackgroundWorker
{
    // 任务执行逻辑
}
```

### 允许并发

如果需要并发执行，使用 `AllowConcurrency`：

```csharp
[AllowConcurrency]
public class DataProcessingJob : HangfireBackgroundWorker
{
    // 任务执行逻辑
}
```

## 错误处理

### 自动重试

Hangfire 默认自动重试失败的任务（10 次，指数退避）：

```csharp
[AutomaticRetry(Attempts = 10, DelaysInSeconds = new[] { 30, 60, 90 })]
public class VoiceprintCaptureJob : HangfireBackgroundWorker
{
    // 任务执行逻辑
}
```

### 禁用重试

```csharp
[AutomaticRetry(Attempts = 0)]
public class OneTimeJob : HangfireBackgroundWorker
{
    // 任务执行逻辑
}
```

### 异常捕获

```csharp
public override async Task DoWorkAsync(CancellationToken cancellationToken)
{
    try
    {
        await ExecuteTaskAsync();
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "任务执行失败: {Message}", ex.Message);
        throw; // 重新抛出以触发 Hangfire 重试
    }
}
```

## 最佳实践

1. **任务设计**
   - 任务应该幂等（多次执行结果一致）
   - 避免长时间运行（超过 30 分钟考虑分片）
   - 使用 `CancellationToken` 支持取消

2. **依赖注入**
   - 仅依赖瞬态或作用域服务
   - 避免依赖单例服务（可能导致内存泄漏）

3. **异常处理**
   - 记录详细日志
   - 抛出异常以触发 Hangfire 重试
   - 使用 `AutomaticRetry` 控制重试策略

4. **性能优化**
   - 批量操作减少数据库调用
   - 使用分页处理大数据集
   - 考虑使用 Hangfire 队列处理高并发任务

5. **配置管理**
   - 使用配置文件控制任务启用/禁用
   - 为不同环境配置不同的 Cron 表达式
   - 监控任务执行频率和耗时

## 相关文档

- [Framework 概览](./README.md)
- [SqlSugarCore](./SqlSugarCore.md)
- [AspNetCore](./AspNetCore.md)
