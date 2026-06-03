# ApplicationStartupService (应用启动服务)

## 概述
ApplicationStartupService 是应用程序的核心启动协调服务，负责在应用启动时初始化各种后台作业和关键服务。该服务作为托管服务运行，确保系统组件按正确顺序启动。

## 职责
- 协调应用启动时的初始化任务
- 启动视觉网关健康检查作业
- 启动网关数据同步作业
- 启动巡视系统相关作业
- 管理应用关闭时的作业清理

## 主要接口

### StartAsync
服务启动时执行初始化

**参数：**
- `cancellationToken` (CancellationToken) - 取消令牌

**执行流程：**
1. 延迟 5 秒启动，确保其他服务已就绪
2. 启动视觉网关健康检查作业
3. 启动网关数据同步作业
4. 启动巡视系统相关作业
5. 记录初始化完成日志

### StopAsync
服务停止时执行清理

**参数：**
- `cancellationToken` (CancellationToken) - 取消令牌

**清理操作：**
- 停止视觉网关健康检查作业
- 停止网关数据同步作业

### StartVisualGatewayHealthCheckJobAsync
启动视觉网关健康检查作业（私有方法）

**操作：**
- 调用 [[VisualGatewayHealthCheckJobManager.StartHealthCheckJob]]
- 立即触发一次健康检查
- 每 2 分钟定期执行健康检查

### StartGatewayDataSyncJobAsync
启动网关数据同步作业（私有方法）

**操作：**
- 调用 [[GatewaySyncJobManager.StartSyncJob]]
- 延迟 10 秒后触发一次数据同步
- 避免与健康检查作业冲突

### StartPatrolSystemJobsAsync
启动巡视系统相关作业（私有方法）

**说明：**
- 预留用于巡视系统作业初始化
- 当前为空实现，便于未来扩展

## 技术实现

### 生命周期管理
- **实现接口** - `IHostedService`, `ITransientDependency`
- **启动时机** - 应用启动后自动执行
- **启动延迟** - 5 秒，确保依赖服务就绪

### 启动策略

#### 延迟启动
```csharp
await Task.Delay(TimeSpan.FromSeconds(5), cancellationToken);
```
确保以下组件已完全初始化：
- 数据库连接
- 依赖注入容器
- 其他托管服务
- 配置系统

#### 作业启动顺序
1. **视觉网关健康检查** - 最优先，确保网关可用性
2. **网关数据同步** - 延迟 10 秒，避免启动冲突
3. **巡视系统作业** - 最后启动，依赖前两项

### 取消支持
全程支持取消令牌，确保优雅关闭：
```csharp
if (cancellationToken.IsCancellationRequested) return;
```

## 管理的作业

### 视觉网关健康检查作业
- **管理器** - [[VisualGatewayHealthCheckJobManager]]
- **执行频率** - 每 2 分钟
- **功能** - 检查视觉网关在线状态
- **立即执行** - 启动时触发一次

### 网关数据同步作业
- **管理器** - [[GatewaySyncJobManager]]
- **执行频率** - 可配置（通过 [[GatewaySyncOptions]]）
- **功能** - 同步网关设备数据和通道信息
- **延迟执行** - 启动 10 秒后触发

### 巡视系统作业
- **状态** - 预留扩展
- **计划功能** - 巡视任务调度、数据采集等

## 启动日志示例

### 成功启动
```
应用程序启动服务开始初始化...
启动视觉网关健康检查作业...
视觉网关健康检查作业启动成功
启动网关数据同步作业...
网关数据同步作业启动成功
启动巡视系统相关作业...
巡视系统相关作业启动成功
应用程序启动服务初始化完成
```

### 停止日志
```
应用程序启动服务正在停止...
应用程序启动服务已停止
```

## 错误处理

### 启动失败
如果启动过程中发生异常：
```csharp
catch (Exception ex)
{
    _logger.LogError(ex, "应用程序启动服务初始化失败");
    throw;  // 重新抛出异常，阻止应用启动
}
```

### 停止失败
停止过程中的错误不会影响应用关闭：
```csharp
catch (Exception ex)
{
    _logger.LogError(ex, "应用程序启动服务停止失败");
    // 不重新抛出，允许应用正常关闭
}
```

## 依赖服务
- [[ILogger]] - 日志记录
- [[IServiceProvider]] - 服务定位器，用于创建作用域
- [[VisualGatewayHealthCheckJobManager]] - 视觉网关健康检查作业管理器
- [[GatewaySyncJobManager]] - 网关数据同步作业管理器

## 相关服务
- [[IHostedService]] - 后台托管服务接口
- [[ITransientDependency]] - 依赖注入生命周期标记
- [[VisualGatewayHealthCheckJob]] - 视觉网关健康检查作业
- [[GatewaySyncJob]] - 网关数据同步作业

## 相关文档
- [[Hangfire 后台任务]]
- [[视觉网关集成]]
- [[网关数据同步]]
- [[巡视系统]]

## 扩展指南

### 添加新的启动作业
在 `StartAsync` 方法中添加：
```csharp
// 启动新的后台作业
await StartNewJobAsync();

private async Task StartNewJobAsync()
{
    try
    {
        _logger.LogInformation("启动新作业...");

        // 启动作业逻辑
        NewJobManager.StartJob();

        _logger.LogInformation("新作业启动成功");
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "启动新作业失败");
        throw;
    }

    await Task.CompletedTask;
}
```

### 添加停止清理
在 `StopAsync` 方法中添加：
```csharp
// 停止作业
NewJobManager.StopJob();
```

## 注意事项
- 该服务对系统启动时间有 5 秒影响
- 作业启动顺序很重要，避免依赖冲突
- 新增作业需考虑资源占用和并发控制
- 建议为每个作业提供独立的启动方法，便于维护
