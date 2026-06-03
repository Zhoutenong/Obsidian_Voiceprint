# 摄像机资源管理器 (CameraResourceManager)

## 概述

**CameraResourceManager** 是摄像机资源的并发管理器，解决多个巡视任务同时执行时对摄像机资源的竞争问题。通过信号量机制（SemaphoreSlim）实现摄像机资源的独占访问控制。

**核心价值**：防止多个巡视任务同时操作同一台摄像机导致流冲突或设备异常。

---

## 核心职责

### 1. 资源占用管理
- 为每台摄像机维护独立的信号量
- 同一时刻只允许一个操作占用摄像机
- 支持超时机制避免死锁

### 2. 并发控制
- 使用 `SemaphoreSlim(1, 1)` 实现互斥访问
- 每台摄像机独立信号量，互不干扰
- 支持异步等待 (`WaitAsync`)

### 3. 状态追踪
- 记录摄像机占用时间
- 关联操作ID用于追踪
- 提供状态查询接口

### 4. 资源清理
- 自动清理过期操作
- 释放未使用的信号量
- 支持强制释放（紧急情况）

---

## 数据结构

### 摄像机信号量字典
```csharp
ConcurrentDictionary<string, SemaphoreSlim> _cameraSemaphores
```
- **Key**: `DeviceId` - 摄像机设备ID
- **Value**: `SemaphoreSlim(1, 1)` - 每台摄像机独立信号量

### 操作上下文字典
```csharp
ConcurrentDictionary<string, CameraOperationContext> _cameraContexts
```
- **Key**: `OperationId` - 操作ID（用于追踪）
- **Value**: `CameraOperationContext` - 操作上下文信息

---

## 主要接口

### AcquireCameraAsync
申请摄像机使用权限

```csharp
Task<ICameraOperationScope> AcquireCameraAsync(
    string deviceId,    // 摄像机设备ID
    string operationId,  // 操作ID（用于追踪）
    int timeout = 30000 // 超时时间（毫秒）
)
```

**返回值**: `ICameraOperationScope` - 摄像机操作作用域，实现 IDisposable

**异常**:
- `ArgumentException` - 设备ID为空
- `TimeoutException` - 申请超时

**工作流程**:
1. 验证设备ID
2. 获取或创建设备专用信号量
3. 使用超时机制等待信号量
4. 创建操作上下文并记录
5. 返回可释放的作用域对象

**使用示例**:
```csharp
// 使用 using 语句确保资源释放
await using var camera = await _cameraResourceManager.AcquireCameraAsync(
    deviceId: "camera-001",
    operationId: "inspection-task-123",
    timeout: 30000
);

// 在此作用域内独占使用摄像机
await _streamingService.StartStreamAsync(camera.DeviceId);
// ... 执行巡视任务

// 离开作用域自动释放摄像机
```

---

### ReleaseCameraInternal
释放摄像机使用权限（内部方法）

```csharp
internal void ReleaseCameraInternal(CameraOperationContext context)
```

**工作流程**:
1. 从上下文字典中移除操作
2. 释放信号量
3. 记录释放日志（包含持续时间）

**注意**: 此方法由 `CameraOperationScope.Dispose()` 自动调用，不应手动调用。

---

### GetCameraStatus
获取摄像机使用状态

```csharp
CameraStatus GetCameraStatus(string deviceId)
```

**返回值**: `CameraStatus`
```csharp
public class CameraStatus
{
    public string DeviceId { get; set; }
    public bool IsInUse { get; set; }           // 是否占用中
    public string CurrentOperationId { get; set; }  // 当前操作ID
    public DateTime? AcquiredAt { get; set; }   // 占用时间
}
```

**状态判断逻辑**:
- `IsInUse = semaphore.CurrentCount == 0` - 信号量为0表示被占用

---

### ForceRelease
强制释放摄像机（紧急情况使用）

```csharp
void ForceRelease(string deviceId, string reason = "强制释放")
```

**使用场景**:
- 摄像机设备异常
- 操作卡死需要恢复
- 系统维护时清空资源

**注意**: 强制释放可能导致原操作异常，谨慎使用。

---

### CleanupExpiredOperationsAsync
清理过期的摄像机操作资源

```csharp
Task CleanupExpiredOperationsAsync(TimeSpan maxAge)
```

**工作流程**:
1. 查找超过 `maxAge` 时长的操作上下文
2. 释放对应的信号量
3. 清理不再使用的信号量并 Dispose
4. 记录清理日志

**清理内容**:
- 过期的操作上下文 (`_cameraContexts`)
- 未使用的信号量 (`_cameraSemaphores`)

**注意**: 此方法由 [[CameraResourceCleanupJob]] 定期调用。

---

### ClearAllResourcesAsync
清理所有摄像机资源（系统重启时使用）

```csharp
Task ClearAllResourcesAsync()
```

**使用场景**:
- 系统重启前的资源清理
- 系统维护时重置所有状态

**工作流程**:
1. 清空所有操作上下文
2. 释放所有信号量
3. Dispose 所有信号量对象
4. 清空信号量字典

---

## 摄像机状态

| 状态 | 说明 | 判断条件 |
|-----|------|---------|
| **Idle** (空闲) | 摄像机可用 | `semaphore.CurrentCount == 1` |
| **Occupied** (占用中) | 摄像机被占用 | `semaphore.CurrentCount == 0` |
| **Timeout** (超时) | 操作超时未释放 | 超过配置的最大操作时长 |

---

## 信号量机制 (SemaphoreSlim)

### 设计模式
每台摄像机维护独立的 `SemaphoreSlim(1, 1)`：
- **初始计数**: 1（表示可用）
- **最大计数**: 1（同时只允许一个操作）

### 生命周期
```
创建 (GetOrAdd) → 等待 (WaitAsync) → 占用 (Count=0) → 释放 (Release) → 清理 (Dispose)
```

### 并发控制流程
```
Task A                  Task B                  Task C
  |                       |                       |
  |--WaitAsync-----------|                       |
  |   (获取锁)            |                       |
  |                       |--WaitAsync-----------|
  |   (执行操作)          |   (等待中...)         |
  |                       |                       |--WaitAsync
  |--Release-------------|                       |   (等待中...)
  |   (释放锁)            |                       |
  |                       |--WaitAsync Complete--|
  |                       |   (获取锁)            |
  |                       |   (执行操作)          |
  |                       |--Release-------------|
  |                       |                       |--WaitAsync Complete
```

---

## 超时清理策略

### 自动清理机制
系统通过 [[CameraResourceCleanupJob]] 定期清理过期资源：

**清理配置** (`CameraResourceCleanupOptions`):
```json
{
  "CronExpression": "0 */5 * * * *",  // 每5分钟执行
  "ExpirationMinutes": 30             // 30分钟未释放视为过期
}
```

### 清理内容
1. **过期操作上下文** - 超过30分钟的操作
2. **未使用信号量** - 不再被引用的信号量对象

### 内存管理
通过 Dispose 未使用的信号量防止内存泄漏：
```csharp
semaphore?.Dispose();  // 释放非托管资源
```

---

## 操作作用域 (ICameraOperationScope)

### 设计模式
使用 RAII (Resource Acquisition Is Initialization) 模式确保资源释放：

```csharp
public interface ICameraOperationScope : IDisposable
{
    string DeviceId { get; }
    string OperationId { get; }
    DateTime AcquiredAt { get; }
}
```

### 自动释放
```csharp
await using var camera = await _cameraResourceManager.AcquireCameraAsync(...);
// 离开作用域时自动调用 Dispose() → ReleaseCameraInternal()
```

---

## 依赖服务

| 服务 | 用途 |
|-----|------|
| [[CameraService]] | 摄像机设备管理 |
| [[StreamingApiService]] | 流媒体服务 |
| [[CameraResourceCleanupJob]] | 定期清理过期资源 |

---

## 配置选项

### CameraResourceCleanupOptions
```csharp
public class CameraResourceCleanupOptions
{
    public string CronExpression { get; set; } = "0 */5 * * * *";
    public int ExpirationMinutes { get; set; } = 30;
}
```

---

## 日志记录

### 关键日志
| 级别 | 消息 | 说明 |
|-----|------|------|
| **LogInformation** | 成功获取摄像机使用权限 | 记录操作ID和设备ID |
| **LogInformation** | 释放摄像机 | 记录操作ID和持续时间 |
| **LogWarning** | 申请摄像机超时 | 记录超时时间和设备ID |
| **LogWarning** | 强制释放摄像机 | 记录原因和当前操作ID |
| **LogWarning** | 清理过期操作 | 记录设备ID和超时时长 |
| **LogError** | 释放摄像机异常 | 记录异常信息 |

---

## 异常处理

| 异常 | 场景 | 处理 |
|-----|------|------|
| `ArgumentException` | 设备ID为空 | 立即抛出，参数验证失败 |
| `TimeoutException` | 申请摄像机超时 | 抛出，由调用方决定重试或放弃 |
| `OperationCanceledException` | 等待信号量超时 | 转换为 `TimeoutException` 抛出 |
| `SemaphoreFullException` | 释放已满信号量 | 忽略，正常情况 |
| `Exception` | 其他未知异常 | 记录日志后继续 |

---

## 最佳实践

### ✅ 推荐做法
```csharp
// 1. 使用 await using 确保资源释放
await using var camera = await _cameraResourceManager.AcquireCameraAsync(
    deviceId: cameraId,
    operationId: inspectionTaskId,
    timeout: 30000
);

// 2. 在作用域内执行操作
await _streamingService.StartStreamAsync(camera.DeviceId);
await _inspectionService.ExecuteTaskAsync(inspectionTaskId);

// 3. 离开作用域自动释放
```

### ❌ 避免做法
```csharp
// 1. 不要忘记释放资源
var camera = await _cameraResourceManager.AcquireCameraAsync(...);
// ... 异常可能跳过释放

// 2. 不要手动调用 ReleaseCameraInternal
_cameraResourceManager.ReleaseCameraInternal(context); // ❌ 错误

// 3. 不要长期占用摄像机
await using var camera = await _cameraResourceManager.AcquireCameraAsync(...);
await Task.Delay(TimeSpan.FromHours(1)); // ❌ 长期占用
```

---

## 与后台任务的关系

### CameraResourceCleanupJob
[[CameraResourceCleanupJob]] 定期调用 `CleanupExpiredOperationsAsync` 清理过期资源：

```
Hangfire 调度 (每5分钟)
  ↓
CameraResourceCleanupJob.DoWorkAsync
  ↓
CameraResourceManager.CleanupExpiredOperationsAsync
  ↓
清理过期操作和未使用信号量
```

**清理时机**:
- 操作超过30分钟未释放
- 系统重启前的 `ClearAllResourcesAsync`
- 手动触发的 `ForceRelease`

---

## 相关文档

- [[CameraService]] - 摄像机设备管理
- [[StreamingApiService]] - 流媒体服务
- [[CameraResourceCleanupJob]] - 定期清理作业
- [[MediaService]] - 媒体服务接口
- [[NvrService]] - NVR 集成服务

---

## 文件位置

**源码**: `module/ast-intellisub/Ast.IntelliSub.Application/Services/CameraResourceManager.cs`

**依赖**:
- `Ast.IntelliSub.Application.Contracts.Dtos.Media.CameraStatus`
- `Ast.IntelliSub.Application.Services.CameraResourceCleanupJob`
- `Ast.IntelliSub.Application.Contracts.Options.CameraResourceCleanupOptions`
