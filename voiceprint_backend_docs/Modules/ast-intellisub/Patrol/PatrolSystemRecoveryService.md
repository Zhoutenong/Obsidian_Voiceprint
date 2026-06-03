# 巡视系统恢复服务 (PatrolSystemRecoveryService)

## 概述
巡视系统恢复服务是一个后台托管服务（IHostedService），在系统启动时自动执行巡视任务和摄像头资源的恢复清理操作，确保系统从异常关闭中恢复正常状态。

## 职责
- 系统启动时自动恢复异常终止的巡视任务
- 清理摄像头资源管理器的资源锁
- 检测并处理僵尸任务
- 清理长时间运行的任务
- 提供系统健康状态检查

## 主要接口

### 服务启动（自动执行）
```csharp
Task StartAsync(CancellationToken cancellationToken)
```

**执行流程**：
1. 延迟10秒等待其他服务启动完成
2. 创建服务作用域
3. 执行巡视任务异常恢复
4. 清理所有摄像头资源锁
5. 记录恢复处理日志

### 服务停止
```csharp
Task StopAsync(CancellationToken cancellationToken)
```

## 恢复机制

### 僵尸任务恢复
系统启动时会检测可能因进程异常终止而留下的僵尸任务：

**检测条件**：
- 任务状态为"进行中"
- 任务开始时间超过预期最大持续时间

**恢复操作**：
1. 将僵尸任务标记为"异常终止"
2. 记录异常原因
3. 释放相关资源

### 摄像头资源清理
清理可能在异常关闭时未释放的摄像头资源锁：

**清理内容**：
- 所有摄像头信号量锁
- 摄像头操作上下文
- 资源占用记录

### 长时间运行任务清理
定期检查和清理超过最大运行时间的任务：

**配置参数**（来自 PatrolConfigOptions）：
- `MaxExecutionDurationMinutes`: 最大执行时长（分钟）

## 数据处理流程

```
系统启动 → 延迟等待 → 创建作用域 → 恢复巡视任务 → 清理摄像头资源 → 完成恢复
```

### 恢复流程详细步骤
1. **延迟执行**：等待10秒确保依赖服务已启动
2. **任务恢复**：调用 `PatrolRecordService.RecoverFromSystemRestartAsync()`
3. **资源清理**：调用 `CameraResourceManager.ClearAllResourcesAsync()`
4. **日志记录**：记录恢复成功或失败信息

## 依赖服务

- [[PatrolRecordService]] - 巡视记录服务（提供任务恢复功能）
- [[CameraResourceManager]] - 摄像头资源管理器（提供资源清理功能）
- [[PatrolExecutionService]] - 巡视执行服务

## 相关实体

- **PatrolRecordEntity** - 巡视记录实体
- **PatrolRecordItemEntity** - 巡视记录详情实体
- **CameraOperationContext** - 摄像头操作上下文

## 配置项

### 恢复服务配置
- **启动延迟**：10秒（硬编码）
- **执行超时**：使用 CancellationToken 控制

### 相关配置（来自 PatrolConfigOptions）
```json
{
  "Patrol": {
    "MaxExecutionDurationMinutes": 120,
    "ZombieTaskCheckIntervalMinutes": 10
  }
}
```

## 注意事项

- **启动顺序**：服务延迟10秒启动，确保其他服务已完全初始化
- **作用域管理**：使用服务作用域获取依赖服务，避免生命周期问题
- **异常处理**：恢复过程中的异常会被捕获并记录，不会影响系统启动
- **幂等性**：重复执行恢复操作不会产生副作用
- **资源清理**：摄像头资源清理是强制性的，即使任务恢复失败也会执行

## 健康检查

系统提供健康检查接口，可用于监控恢复状态：

```csharp
Task<PatrolSystemHealthDto> GetSystemHealthAsync()
```

**健康指标**：
- 当前执行中的任务数量
- 僵尸任务数量
- 摄像头资源占用情况
- 系统最后恢复时间

## 相关文档链接

- [[PatrolRecordService]] - 巡视记录服务
- [[PatrolExecutionService]] - 巡视执行服务
- [[CameraResourceManager]] - 摄像头资源管理器
- [[巡视任务执行流程]] - 完整的巡视执行流程说明
