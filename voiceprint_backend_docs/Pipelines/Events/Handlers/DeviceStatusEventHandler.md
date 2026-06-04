# DeviceStatusEventHandler — 设备状态事件处理器

## 基本信息

- **处理器名称**：`DeviceStatusEventHandler`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Application/EventHandlers/`
- **事件类型**：`DeviceStatusEventArgs` (ILocalEventHandler)
- **生命周期**：`ITransientDependency` - 瞬态依赖

## 处理器概述

DeviceStatusEventHandler 负责处理批量设备状态更新事件。当网关上报设备状态时，该处理器会批量更新数据库中对应设备（Device）和传感器（Sensor）的状态信息。

## 核心职责

1. **批量设备状态更新** - 批量更新多个设备的状态
2. **批量传感器状态更新** - 批量更新多个传感器的状态
3. **性能优化** - 使用批量查询和更新减少数据库往返
4. **统计记录** - 记录处理的统计信息

## 依赖注入

| 依赖 | 职责 |
|------|------|
| `ISqlSugarRepository<DeviceEntity, string>` | 设备实体仓储 |
| `ISqlSugarRepository<SensorEntity, Guid>` | 传感器实体仓储 |
| `ILogger<DeviceStatusEventHandler>` | 日志记录器 |

## 事件数据结构

### DeviceStatusEventArgs

```csharp
public class DeviceStatusEventArgs
{
    public Guid GatewayId { get; set; }                    // 网关ID
    public MqDeviceStatusDataDto Data { get; set; }       // 设备状态数据
}

public class MqDeviceStatusDataDto
{
    public List<MqDeviceStatusDto> Devices { get; set; }  // 设备状态列表
}

public class MqDeviceStatusDto
{
    public string DeviceId { get; set; }                  // 设备ID
    public int Status { get; set; }                       // 状态
    public List<MqSensorStatusDto> Sensors { get; set; }  // 传感器状态列表
}

public class MqSensorStatusDto
{
    public string SensorKey { get; set; }                 // 传感器键值
    public int Status { get; set; }                       // 状态
}
```

## 处理流程

```
1. 事件接收与验证
   │
   ├─ 记录日志：网关ID、设备数量
   │
   ├─ 如果设备列表为空
   │  ├─ 记录警告日志
   │  └─ 直接返回
   │
2. 批量处理（UnitOfWork事务）
   │
   ├─ 2.1 批量查询设备
   │  ├─ 2.2 批量查询传感器
   │  ├─ 2.3 准备更新数据
   │  └─ 2.4 执行批量更新
   │
3. 记录完成日志
   │
   └─ 输出：处理设备数、处理传感器数、总耗时
```

## 核心方法

### 1. HandleEventAsync - 处理设备状态事件

**签名**：
```csharp
[UnitOfWork(isTransactional: true)]
public async Task HandleEventAsync(DeviceStatusEventArgs eventData)
```

**功能**：
处理批量设备状态更新事件的主入口。

### 2. ProcessBatchDeviceStatusOptimizedAsync - 优化的批量处理

**签名**：
```csharp
[UnitOfWork(isTransactional: true)]
private async Task ProcessBatchDeviceStatusOptimizedAsync(
    Guid gatewayId, 
    List<MqDeviceStatusDto> deviceStatusList)
```

**功能**：
优化的批量设备状态处理方法，包含完整的批量查询、准备和更新流程。

**执行步骤**：

```csharp
// 1. 批量查询所有相关设备
var deviceIds = deviceStatusList.Select(d => d.DeviceId).Distinct().ToList();
var devices = await _deviceRepository.GetListAsync(d =>
    deviceIds.Contains(d.Id) && d.GatewayId == gatewayId);
var deviceDict = devices.ToDictionary(d => d.Id, d => d);

// 2. 批量查询所有相关传感器
var sensorDict = await BatchQuerySensorsAsync(deviceStatusList, deviceIds, startTime);

// 3. 准备批量更新数据
var (devicesToUpdate, sensorsToUpdate, stats) = 
    PrepareUpdateData(deviceStatusList, deviceDict, sensorDict, gatewayId);

// 4. 执行批量更新
await ExecuteBatchUpdatesAsync(devicesToUpdate, sensorsToUpdate, startTime);

// 5. 记录统计信息
_logger.LogInformation($"批量处理完成 - 设备: {stats.ProcessedDevices}/{deviceStatusList.Count}, " +
                     $"传感器: {stats.ProcessedSensors}, 总耗时: {(DateTime.Now - startTime).TotalMilliseconds}ms");
```

### 3. BatchQuerySensorsAsync - 批量查询传感器

**签名**：
```csharp
private async Task<Dictionary<string, Dictionary<string, SensorEntity>>> BatchQuerySensorsAsync(
    List<MqDeviceStatusDto> deviceStatusList, 
    List<string> deviceIds, 
    DateTime startTime)
```

**功能**：
批量查询所有相关传感器，并按设备ID和传感器键值分组。

**优化点**：
- 提取所有传感器键，避免重复查询
- 使用 `GroupBy` 优化分组逻辑
- 返回嵌套字典结构：`Dictionary<DeviceId, Dictionary<SensorKey, SensorEntity>>`

### 4. PrepareUpdateData - 准备更新数据

**签名**：
```csharp
private (List<DeviceEntity> DevicesToUpdate, List<SensorEntity> SensorsToUpdate, ProcessingStats Stats)
    PrepareUpdateData(List<MqDeviceStatusDto> deviceStatusList,
                    Dictionary<string, DeviceEntity> deviceDict,
                    Dictionary<string, Dictionary<string, SensorEntity>> sensorDict,
                    Guid gatewayId)
```

**功能**：
准备需要更新的设备和传感器数据，实现变更追踪和状态判断。

**状态更新条件**：

**设备状态更新条件**：
```csharp
// 满足以下任一条件即更新
if (device.Status != newStatus ||                          // 状态改变
    device.LastHeartbeatTime == null ||                     // 无心跳记录
    (currentTime - device.LastHeartbeatTime.Value).TotalMinutes > 1) // 心跳间隔超过1分钟
{
    device.Status = newStatus;
    device.LastHeartbeatTime = currentTime;
    devicesToUpdate.Add(device);
}
```

**传感器状态更新条件**：
```csharp
// 只有状态真正改变时才更新
if (sensor.Status != newSensorStatus)
{
    sensor.Status = newSensorStatus;
    sensorsToUpdate.Add(sensor);
}
```

### 5. ExecuteBatchUpdatesAsync - 执行批量更新

**签名**：
```csharp
private async Task ExecuteBatchUpdatesAsync(
    List<DeviceEntity> devicesToUpdate,
    List<SensorEntity> sensorsToUpdate,
    DateTime startTime)
```

**功能**：
执行批量更新操作。

**⚠️ 重要设计决策**：
```csharp
// ✅ 顺序执行更新，保持在同一个 UnitOfWork 上下文中
// 虽然失去了并行性能，但避免了 UOW 泄漏和线程安全问题
if (devicesToUpdate.Any())
{
    await _deviceRepository.UpdateRangeAsync(devicesToUpdate);
}

if (sensorsToUpdate.Any())
{
    await _sensorRepository.UpdateRangeAsync(sensorsToUpdate);
}
```

**不使用并行**的原因：
- 保持UnitOfWork事务上下文
- 避免线程安全问题
- 防止数据库连接泄漏

## 统计信息

### ProcessingStats - 处理统计

```csharp
private class ProcessingStats
{
    public int ProcessedDevices { get; set; }   // 已处理设备数
    public int ProcessedSensors { get; set; }   // 已处理传感器数
    public int MissingDevices { get; set; }    // 缺失设备数
    public int MissingSensors { get; set; }     // 缺失传感器数
}
```

## 日志记录

### 信息日志 (LogInformation)

```csharp
// 处理开始
$"开始处理网关 {gatewayId} 的批量设备状态更新，设备数量: {deviceCount}"

// 处理完成
$"网关 {gatewayId} 的批量设备状态更新完成，处理设备数量: {deviceCount}"

// 批量更新统计
$"批量处理完成 - 设备: {processed}/{total}, 传感器: {sensorCount}, 总耗时: {time}ms"

// 批量更新完成
$"批量更新 {devicesToUpdate.Count} 个设备状态完成"
$"批量更新 {sensorsToUpdate.Count} 个传感器状态完成"
```

### 调试日志 (LogDebug)

```csharp
// 批量查询完成
$"批量查询设备完成，找到 {found}/{total} 个设备，耗时: {time}ms"
$"批量查询传感器完成，找到 {count} 个传感器，耗时: {time}ms"

// 批量更新执行
$"批量更新执行完成，耗时: {time}ms"
```

### 警告日志 (LogWarning)

```csharp
// 数据为空
$"网关 {gatewayId} 的设备状态更新数据为空"

// 设备或传感器未找到
$"未找到传感器: {sensorKey}，设备: {deviceId}"
$"未找到设备: {deviceId}，网关: {gatewayId}"
```

### 错误日志 (LogError)

```csharp
// 处理失败
$"处理网关 {gatewayId} 的批量设备状态更新时发生错误"
$"批量处理设备状态更新时发生错误，网关: {gatewayId}"
```

## 性能优化

### 1. 批量查询

```csharp
// ❌ 错误：逐个查询
foreach (var deviceId in deviceIds)
{
    var device = await _deviceRepository.GetAsync(deviceId);
}

// ✅ 正确：批量查询
var devices = await _deviceRepository.GetListAsync(d =>
    deviceIds.Contains(d.Id));
```

### 2. 批量更新

```csharp
// ❌ 错误：逐个更新
foreach (var device in devicesToUpdate)
{
    await _deviceRepository.UpdateAsync(device);
}

// ✅ 正确：批量更新
await _deviceRepository.UpdateRangeAsync(devicesToUpdate);
```

### 3. 内存优化

- 使用字典（Dictionary）快速查找，避免线性搜索
- 只收集真正需要更新的实体，减少数据库操作
- 提前提取所有ID，一次性查询

### 4. 事务控制

```csharp
[UnitOfWork(isTransactional: true)]
```

- 整个处理过程在一个事务中完成
- 保证数据一致性
- 失败时自动回滚

## 错误处理

### 异常处理策略

```csharp
try
{
    // 处理逻辑
}
catch (Exception ex)
{
    _logger.LogError(ex, $"处理网关 {gatewayId} 的批量设备状态更新时发生错误");
    throw;  // 重新抛出，触发事务回滚
}
```

## 相关实体

- [[DeviceEntity]] - 设备实体
- [[SensorEntity]] - 传感器实体

## 相关枚举

- [[DeviceStatusEnum]] - 设备状态枚举
- [[CommonStatusEnum]] - 通用状态枚举

## 相关事件

- `DeviceStatusEventArgs` - 设备状态事件参数

## 使用场景

### 1. 网关心跳上报

网关定期上报所有设备的状态：
```csharp
var eventData = new DeviceStatusEventArgs
{
    GatewayId = gatewayId,
    Data = new MqDeviceStatusDataDto
    {
        Devices = new List<MqDeviceStatusDto>
        {
            new MqDeviceStatusDto { DeviceId = "device-001", Status = 1, Sensors = [...] }
        }
    }
};
```

### 2. 设备状态变化

当设备状态发生变化时，网关实时上报：
```csharp
eventData.Data.Devices[0].Status = (int)DeviceStatusEnum.Fault;
```

### 3. 传感器状态变化

传感器状态单独上报：
```csharp
eventData.Data.Devices[0].Sensors[0].Status = (int)CommonStatusEnum.Fault;
```

## 设计特点

1. **批量处理** - 一次性处理多个设备和传感器
2. **性能优化** - 批量查询、批量更新、字典查找
3. **事务保证** - UnitOfWork确保数据一致性
4. **详细日志** - 记录每个步骤的统计信息
5. **变更追踪** - 只更新真正变化的实体
6. **心跳优化** - 心跳间隔超过1分钟才更新

## 性能指标

根据日志记录，典型的处理耗时：
- 批量查询设备：10-50ms
- 批量查询传感器：20-100ms
- 批量更新操作：50-200ms
- **总耗时**：100-500ms（取决于设备和传感器数量）

---

> **最后更新**：2026-06-04  
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Application/EventHandlers/DeviceStatusEventHandler.cs`
