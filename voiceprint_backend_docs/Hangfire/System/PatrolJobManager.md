# 巡检任务调度器 (PatrolJobManager)

## 概述
巡检任务调度器是 Hangfire 后台任务，负责按计划触发巡检任务的执行。

## 职责
- 查询待执行的巡检任务
- 触发巡检执行服务
- 监控执行状态
- 记录执行日志
- 处理执行异常

## 配置

### appsettings.json

```json
{
  "PatrolJob": {
    "Enabled": true,
    "CronExpression": "0 0 */2 * * *",
    "MaxConcurrentJobs": 3,
    "TimeoutMinutes": 60
  }
}
```

### Cron 表达式说明

```
0 0 */2 * * *    // 每2小时执行一次
0 30 8 * * *     // 每天8:30执行
0 0 8,20 * * *   // 每天8:00和20:00执行
0 0 8-18/2 * * * // 每天8:00-18:00，每2小时执行
```

## 执行流程

```
触发执行 → 查询待执行任务 → 过滤任务 → 并发执行 → 监控状态 → 记录日志
```

### 详细步骤

1. **任务查询**
   - 查询状态为 `Pending` 的任务
   - 过滤到达执行时间的任务
   - 检查任务依赖关系

2. **任务过滤**
   - 检查设备状态
   - 检查任务有效期
   - 检查资源可用性

3. **并发执行**
   - 最多并发执行 N 个任务
   - 每个任务独立执行
   - 超时任务自动终止

4. **状态监控**
   - 更新任务状态为 `Running`
   - 监控执行进度
   - 检测超时

5. **结果记录**
   - 记录执行成功/失败
   - 记录执行时间
   - 记录异常信息

## 执行逻辑

### 任务调度

```csharp
public async Task ExecuteAsync()
{
    // 1. 查询待执行任务
    var tasks = await _patrolTaskService.GetPendingTasksAsync();

    // 2. 过滤可执行任务
    var executableTasks = tasks
        .Where(t => ShouldExecute(t))
        .Take(_maxConcurrentJobs)
        .ToList();

    // 3. 并发执行
    var executions = executableTasks.Select(task =>
        _patrolExecutionService.ExecuteAsync(task.Id)
    );

    await Task.WhenAll(executions);
}

private bool ShouldExecute(PatrolTask task)
{
    // 检查设备状态
    if (!IsDeviceAvailable(task.DeviceId))
        return false;

    // 检查有效期
    if (task.EndDate.HasValue && task.EndDate < DateTime.Now)
        return false;

    // 检查是否正在执行
    if (task.Status == PatrolTaskStatus.Running)
        return false;

    return true;
}
```

### 超时处理

```csharp
public async Task ExecuteWithTimeoutAsync(Guid taskId, int timeoutMinutes)
{
    using var cts = new CancellationTokenSource(TimeSpan.FromMinutes(timeoutMinutes));

    try
    {
        await _patrolExecutionService.ExecuteAsync(taskId, cts.Token);
    }
    catch (OperationCanceledException)
    {
        // 超时处理
        await _patrolTaskService.UpdateStatusAsync(taskId, PatrolTaskStatus.Failed, "执行超时");
        _logger.LogWarning($"巡检任务 {taskId} 执行超时");
    }
}
```

## 依赖服务

- [[PatrolTaskService]] - 巡检任务服务
- [[PatrolExecutionService]] - 巡检执行服务
- [[DeviceService]] - 设备服务

## 相关任务

- [[PatrolSystemCleanupJob]] - 巡检系统清理任务
- [[PatrolExecutionService]] - 巡检执行服务
- [[CameraService]] - 摄像机服务

## 任务状态

### 执行前状态

| 状态 | 说明 |
|-----|------|
| Pending | 待执行 |

### 执行中状态

| 状态 | 说明 |
|-----|------|
| Running | 执行中 |

### 执行后状态

| 状态 | 说明 |
|-----|------|
| Completed | 已完成 |
| Failed | 失败 |
| Cancelled | 已取消 |
| Timeout | 超时 |

## 异常处理

### 任务执行异常
- 记录异常日志
- 更新任务状态为 `Failed`
- 保存错误信息
- 可选：发送告警

### 系统异常
- 系统崩溃 → Hangfire 自动重试
- 网络中断 → 重试连接
- 服务重启 → 恢复未完成任务

### 资源不足
- 并发数限制 → 排队等待
- 设备占用 → 延后执行
- 存储不足 → 记录日志，跳过执行

## 日志记录

### 执行日志

```
[2026-06-03 08:00:00] INFO  开始执行巡检任务调度
[2026-06-03 08:00:01] INFO  查询到 3 个待执行任务
[2026-06-03 08:00:01] INFO  开始执行任务: Task-001
[2026-06-03 08:00:01] INFO  开始执行任务: Task-002
[2026-06-03 08:00:01] INFO  开始执行任务: Task-003
[2026-06-03 08:15:23] INFO  任务 Task-001 执行完成
[2026-06-03 08:15:45] INFO  任务 Task-002 执行完成
[2026-06-03 08:16:02] ERROR 任务 Task-003 执行失败: 摄像机连接超时
[2026-06-03 08:16:02] INFO  巡检任务调度完成，成功: 2, 失败: 1
```

## 监控指标

### 执行统计
- 总执行次数
- 成功次数
- 失败次数
- 超时次数
- 平均执行时间

### 资源使用
- 并发执行数
- 内存使用
- CPU 使用
- 网络流量

## 配置项

```json
{
  "PatrolJob": {
    "Enabled": true,
    "CronExpression": "0 0 */2 * * *",
    "MaxConcurrentJobs": 3,
    "TimeoutMinutes": 60,
    "RetryCount": 3,
    "RetryDelayMinutes": 5
  }
}
```

## Hangfire Dashboard

在 Hangfire Dashboard 中可以看到：
- 任务执行历史
- 任务执行时间
- 任务执行结果
- 任务异常信息

访问地址：`http://localhost:19001/hangfire`

## 相关文档

- [[PatrolTaskService]] - 巡检任务服务文档
- [[PatrolExecutionService]] - 巡检执行服务文档
- [[PatrolRecordService]] - 巡检记录服务文档
- [[PatrolSystemCleanupJob]] - 巡检系统清理任务文档

## 注意事项

1. **并发控制**：同一设备的巡检任务不能并发执行
2. **资源占用**：巡检执行会占用摄像机和网络资源
3. **超时处理**：长时间执行的任务需要设置合理超时
4. **错误恢复**：失败任务支持手动重试
5. **日志管理**：执行日志需要定期清理

## 最佳实践

1. **调度策略**
   - 避免高峰时段调度
   - 合理设置并发数
   - 考虑天气影响

2. **异常处理**
   - 记录详细错误信息
   - 支持断点续传
   - 失败任务重试

3. **性能优化**
   - 任务预加载
   - 资源预热
   - 并发控制

4. **监控告警**
   - 执行超时告警
   - 失败率告警
   - 资源占用告警
