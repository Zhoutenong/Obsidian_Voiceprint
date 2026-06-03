# 声纹采集运行时状态服务 (VoiceprintCaptureRuntimeStateService)

## 概述

**VoiceprintCaptureRuntimeStateService** 是声纹手动采集运行时状态管理的核心服务，负责管理手动采集和上传流程的完整状态机。该服务采用单例模式，使用内存存储状态，通过线程安全的方式管理多设备采集批次的完成状态。

**核心职责**：
- 管理手动采集批次的运行时状态
- 跟踪多设备采集的完成进度
- 检测采集超时并自动恢复
- 提供手动取消机制
- 防止定时任务与手动采集冲突

## 状态机设计

### 状态枚举

| 状态 | 说明 | 触发条件 |
|-----|------|----------|
| **Idle** | 空闲状态 | 初始状态、采集完成后 |
| **ManualCapturing** | 手动采集中 | 调用 `StartManualCapture` |
| **WaitingUpload** | 等待设备上传 | 采集指令已下发，等待设备上传音频 |
| **UploadCompleted** | 上传完成 | 所有预期设备完成上传 |
| **Timeout** | 超时状态 | 超过配置的超时时间 |

### 状态转换流程

```
┌─────────────┐
│    Idle     │
└──────┬──────┘
       │ StartManualCapture
       ▼
┌─────────────────────┐
│ ManualCapturing    │
│ (等待设备上传)      │
└──────┬──────────────┘
       │ TryMarkUploadCompleted
       ▼
┌─────────────────────┐
│ UploadCompleted     │
│ (批次完成)          │
└──────┬──────────────┘
       │ 超时/手动取消
       ▼
┌─────────────┐
│    Idle     │
└─────────────┘
```

## 核心属性

| 属性 | 类型 | 说明 |
|-----|------|------|
| `IsManualCaptureRunning` | `bool` | 当前是否有手动采集批次运行中 |
| `CurrentGroupId` | `Guid?` | 当前采集批次 ID，未运行时为 `null` |
| `_startedAtUtc` | `DateTime` | 采集批次启动时间（UTC） |
| `_timeout` | `TimeSpan` | 超时时长，默认 10 分钟 |
| `_expectedDevices` | `HashSet<string>` | 预期需要上传的设备 ID 集合 |
| `_completedDevices` | `HashSet<string>` | 已完成上传的设备 ID 集合 |

## 主要接口

### StartManualCapture

启动手动采集批次。

```csharp
void StartManualCapture(Guid groupId, IEnumerable<string> expectedDevices, TimeSpan timeout)
```

**参数**：
- `groupId` - 采集批次唯一标识
- `expectedDevices` - 预期参与采集的设备 ID 列表
- `timeout` - 超时时长，≤0 时使用默认值 10 分钟

**行为**：
1. 重置内部状态
2. 设置采集批次 ID 和启动时间
3. 初始化预期设备集合（去除空白项，忽略大小写）
4. 清空已完成设备集合

**线程安全**：使用 `lock (_sync)` 保护

### TryMarkUploadCompleted

标记指定设备上传完成，并检查是否所有预期设备都已完成。

```csharp
bool TryMarkUploadCompleted(Guid groupId, string deviceId, 
    out bool allCompleted, out string message)
```

**参数**：
- `groupId` - 采集批次 ID
- `deviceId` - 设备 ID

**返回值**：
- `allCompleted` - 是否所有预期设备都已完成
- `message` - 完成状态消息
- `bool` - 是否成功标记（失败表示批次不匹配或不在运行中）

**验证逻辑**：
1. `groupId` 和 `deviceId` 不能为空
2. 必须处于手动采集运行状态
3. `groupId` 必须与当前批次匹配
4. 设备 ID 去除空白后添加到已完成集合
5. 检查已完成集合是否为预期设备的超集

### TryRecoverTimeout

检测是否超时，如果超时则自动恢复到空闲状态。

```csharp
bool TryRecoverTimeout(out Guid? recoveredGroupId, out string reason)
```

**返回值**：
- `recoveredGroupId` - 超时的批次 ID
- `reason` - 超时原因（包含已用时间）
- `bool` - 是否发生了超时恢复

**超时判定**：
```csharp
var elapsed = DateTime.UtcNow - _startedAtUtc;
if (elapsed < _timeout) return false;
```

**行为**：
- 超时后自动调用 `ResetUnsafe()` 清理状态
- 返回超时时长的描述信息

### CancelManualCapture

手动取消当前采集批次。

```csharp
bool CancelManualCapture(out Guid? cancelledGroupId)
```

**参数**：
- `cancelledGroupId` - 被取消的批次 ID

**返回值**：
- 是否成功取消（`false` 表示当前无运行中的批次）

**行为**：
- 调用 `ResetUnsafe()` 清理所有状态
- 用于用户主动取消或切换算法时

## 线程安全保证

所有公共方法都使用 `lock (_sync)` 保护关键区：

```csharp
private readonly object _sync = new();

public bool IsManualCaptureRunning
{
    get
    {
        lock (_sync)
        {
            return _isManualCaptureRunning;
        }
    }
}
```

**设计要点**：
- 使用私有 `_sync` 对象作为锁
- 所有状态读写都在锁内完成
- `HashSet` 使用 `StringComparer.OrdinalIgnoreCase` 忽略大小写
- `ResetUnsafe()` 仅在已获取锁的上下文中调用

## 与其他服务的协作

### 与 VoiceprintAudioAppService.UploadAsync 的关系

**调用位置**：`VoiceprintAudioAppService.UploadAsync` (第 174 行)

```csharp
if (_runtimeState.TryMarkUploadCompleted(input.GroupId, input.DeviceId, 
    out var allCompleted, out var completionMessage) && allCompleted)
{
    _logger.LogInformation(
        "Voiceprint 手动采集批次上传已完成，保持手动模式等待超时或手动恢复: " +
        "groupId={GroupId}, message={Message}",
        input.GroupId, completionMessage);
}
```

**协作流程**：
1. 设备通过 `/api/app/voiceprint/upload` 上传音频
2. `VoiceprintAudioAppService` 处理上传后调用 `TryMarkUploadCompleted`
3. 如果所有预期设备都完成上传，记录日志但保持手动模式
4. 等待超时自动恢复或用户手动切换回原模式

### 与 VoiceprintCaptureJob 的关系

**依赖注入**：`VoiceprintCaptureJob` 构造函数注入

```csharp
public VoiceprintCaptureJob(
    IMqttService mqttService,
    IOptions<VoiceprintCaptureJobOptions> options,
    IOptions<VoiceprintRecognitionOptions> recognitionOptions,
    IGuidGenerator guidGenerator,
    VoiceprintCaptureRuntimeStateService runtimeState,  // 注入
    IHttpClientFactory httpClientFactory)
```

**调用位置**：`DoWorkAsync` 方法 (第 54 行)

```csharp
public override async Task DoWorkAsync(CancellationToken cancellationToken = default)
{
    // 1. 先检查是否需要超时恢复
    if (_runtimeState.TryRecoverTimeout(out var recoveredGroupId, out var timeoutReason))
    {
        Logger.LogWarning("Voiceprint 手动采集超时，自动恢复定时任务: " +
            "groupId={GroupId}, reason={Reason}", recoveredGroupId, timeoutReason);
        await TrySwitchAnalysisToOriginalAsync($"job-timeout-recover:{recoveredGroupId}");
    }

    // 2. 检查是否有手动采集运行中，如果有则跳过本轮
    if (_runtimeState.IsManualCaptureRunning)
    {
        Logger.LogInformation("VoiceprintCaptureJob 跳过本轮执行：手动采集批次运行中，" +
            "groupId={GroupId}", _runtimeState.CurrentGroupId);
        return;
    }

    // 3. 正常执行定时采集任务
    // ...
}
```

**协作流程**：
1. 每次定时任务执行前，先检查是否需要超时恢复
2. 如果超时，自动切回原算法模式
3. 如果有手动采集运行中，跳过本轮定时任务
4. 确保手动采集和定时采集不会冲突

## 配置项说明

### VoiceprintCaptureJobOptions

**配置节**：`VoiceprintCaptureJob`

| 配置项 | 类型 | 默认值 | 说明 |
|-------|------|--------|------|
| `Enabled` | `bool` | `true` | 是否启用定时采集任务 |
| `CronExpression` | `string` | `"0 */5 * * * *"` | Cron 表达式，默认每 5 分钟 |
| `DurationSeconds` | `int` | `10` | 定时采集录音时长（秒） |
| `ManualCaptureTimeoutMinutes` | `int` | `10` | 手动采集超时时间（分钟） |
| `Agents` | `List<VoiceprintCaptureAgentOption>` | - | 采集终端配置 |

**示例配置**：
```json
{
  "VoiceprintCaptureJob": {
    "Enabled": true,
    "CronExpression": "0 */5 * * * *",
    "DurationSeconds": 10,
    "ManualCaptureTimeoutMinutes": 10,
    "Agents": [
      {
        "GatewayId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
        "DeviceIds": ["device-001", "device-002"],
        "RetryLimit": 3
      }
    ]
  }
}
```

### 超时配置的影响

- **默认超时**：10 分钟（从 `ManualCaptureTimeoutMinutes` 配置）
- **超时判定**：`DateTime.UtcNow - _startedAtUtc >= _timeout`
- **超时恢复**：自动调用 `ResetUnsafe()` 清理状态

## 使用场景

### 场景 1：算法切换触发的手动采集

1. 用户调用 `/api/app/voiceprint/capture/algorithm-switch` 切换到增强模式
2. `VoiceprintAudioAppService.SwitchToEnhancedModeAsync` 调用 `StartManualCapture`
3. 系统下发 60 秒采集指令到所有配置的设备
4. 设备上传音频到 `/api/app/voiceprint/upload`
5. 每次上传调用 `TryMarkUploadCompleted` 追踪进度
6. 所有设备完成后等待超时或用户手动切回

### 场景 2：超时自动恢复

1. 手动采集启动后，如果在 `ManualCaptureTimeoutMinutes` 时间内未完成
2. 下次 `VoiceprintCaptureJob.DoWorkAsync` 执行时检测到超时
3. 自动调用 `TryRecoverTimeout` 清理状态
4. 自动切回原算法模式
5. 定时任务恢复正常执行

### 场景 3：手动取消

1. 用户调用 `/api/app/voiceprint/capture/manual-cancel`
2. `VoiceprintAudioAppService.CancelManualCaptureAsync` 调用 `CancelManualCapture`
3. 立即清理状态，允许定时任务下次执行

## 重要注意事项

### 线程安全
- 所有状态修改都在 `lock (_sync)` 保护下进行
- 使用 `HashSet<string>` 并配置 `StringComparer.OrdinalIgnoreCase` 忽略大小写

### 状态不一致风险
- 服务是内存态，重启后状态丢失
- 超时恢复依赖定时任务触发，如果定时任务被禁用可能导致状态无法自动恢复

### 设备 ID 规范
- 设备 ID 会自动 `Trim()` 去除首尾空白
- 空白或 `null` 的设备 ID 会被忽略
- 比较时忽略大小写

### 依赖服务生命周期
- 服务注册为 `ISingletonDependency`，单例模式
- 与 `VoiceprintCaptureJob`（每次执行创建新实例）协作时需要注意状态访问

## 相关文档

- [[VoiceprintAudioAppService]] - 声纹音频上传服务
- [[VoiceprintCaptureJob]] - 声纹采集定时任务
- [[VoiceprintCaptureJobOptions]] - 采集任务配置选项
- [[声纹采集服务]] - 声纹采集架构概述
- [[音频采集流程]] - 完整的采集流程说明

## 源码位置

- **服务实现**：`module/ast-voiceprint/Ast.Voiceprint.Application/Services/VoiceprintCaptureRuntimeStateService.cs`
- **依赖注入**：作为 `ISingletonDependency` 自动注册
- **使用位置**：
  - `VoiceprintAudioAppService` (上传处理、算法切换)
  - `VoiceprintCaptureJob` (定时任务、超时恢复)
