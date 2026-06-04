# DeviceMetadataEventHandler — 设备元数据事件处理器

## 基本信息

- **处理器名称**：`DeviceMetadataEventHandler`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Application/EventHandlers/`
- **事件类型**：`DeviceMetadataEventArgs` (ILocalEventHandler)
- **生命周期**：`ITransientDependency` - 瞬态依赖

## 处理器概述

DeviceMetadataEventHandler 负责处理设备元数据变更事件。当网关上报设备、传感器、点位的元数据时，该处理器会同步更新数据库中的三层实体结构（Device → Sensor → Point），实现增量更新和删除清理。

## 核心职责

1. **三层实体同步** - 同步Device、Sensor、AstPoint三层结构
2. **增量更新** - 只更新真正变化的实体
3. **删除清理** - 删除不再存在的实体
4. **详细统计** - 记录每层实体的变更统计

## 依赖注入

| 依赖 | 职责 |
|------|------|
| `ISqlSugarRepository<DeviceEntity, string>` | 设备实体仓储 |
| `ISqlSugarRepository<SensorEntity, Guid>` | 传感器实体仓储 |
| `ISqlSugarRepository<AstPointEntity, Guid>` | 点位实体仓储 |
| `ILogger<DeviceMetadataEventHandler>` | 日志记录器 |

## 事件数据结构

### DeviceMetadataEventArgs

```csharp
public class DeviceMetadataEventArgs
{
    public Guid GatewayId { get; set; }                    // 网关ID
    public MqDeviceMetadataDto Data { get; set; }         // 设备元数据
}

public class MqDeviceMetadataDto
{
    public List<MqDeviceDto> Devices { get; set; }        // 设备列表
}

public class MqDeviceDto
{
    public string DeviceId { get; set; }                   // 设备ID
    public string Name { get; set; }                      // 设备名称
    public string Type { get; set; }                      // 设备类型
    public string SubType { get; set; }                   // 设备子类型
    public string HardwareName { get; set; }              // 硬件名称
    public List<MqSensorDto> Sensors { get; set; }        // 传感器列表
}

public class MqSensorDto
{
    public string SensorKey { get; set; }                 // 传感器键值
    public string Name { get; set; }                      // 传感器名称
    public string Type { get; set; }                      // 传感器类型
    public int Frequency { get; set; }                    // 采集频率
    public List<MqPointDto> Points { get; set; }          // 点位列表
}

public class MqPointDto
{
    public string Property { get; set; }                  // 属性名称
    public string Name { get; set; }                      // 点位名称
    public string Unit { get; set; }                      // 单位
    public int ValueType { get; set; }                   // 值类型
    public string Type { get; set; }                      // 点位类型
    public string SrcPointId { get; set; }                // 源点位ID
}
```

## 处理流程

```
1. 获取现有数据
   │
   ├─ 查询网关下所有现有设备
   │
2. 遍历设备列表
   │
   ├─ 对每个设备：
   │  │
   │  ├─ 2.1 更新或插入设备
   │  │  ├─ 判断是否有变化
   │  │  └─ InsertOrUpdateAsync
   │  │
   │  ├─ 2.2 查询设备下现有传感器
   │  │
   │  ├─ 2.3 遍历传感器列表
   │  │  │
   │  │  ├─ 对每个传感器：
   │  │  │  │
   │  │  │  ├─ 更新或插入传感器
   │  │  │  │  ├─ 判断是否有变化
   │  │  │  │  └─ InsertOrUpdateAsync
   │  │  │  │
   │  │  │  ├─ 查询传感器下现有点位
   │  │  │  │
   │  │  │  ├─ 遍历点位列表
   │  │  │  │  │
   │  │  │  │  └─ 对每个点位：
   │  │  │  │     ├─ 更新或插入点位
   │  │  │  │     ├─ 判断是否有变化
   │  │  │  │     └─ InsertOrUpdateAsync
   │  │  │  │
   │  │  │  ├─ 删除不再存在的点位
   │  │  │  │  └─ 删除点位 → 删除关联数据
   │  │  │  │
   │  │  │  └─ 返回传感器处理结果
   │  │  │
   │  │  ├─ 删除不再存在的传感器
   │  │  │  └─ 删除传感器 → 删除关联点位
   │  │  │
   │  │  └─ 返回设备处理结果
   │  │
   │  └─ 返回所有设备处理结果
   │
3. 删除不再存在的设备
   │
   └─ 删除设备 → 删除关联传感器和点位
   │
4. 输出统计信息
   │
   └─ 输出三层变更统计
```

## 核心方法

### HandleEventAsync - 处理元数据事件

**签名**：
```csharp
public async Task HandleEventAsync(DeviceMetadataEventArgs eventData)
```

**功能**：
处理设备元数据同步的主方法。

### 变更判断逻辑

**设备变更条件**：
```csharp
bool deviceChanged = existingDevice == null ||
    existingDevice.Name != device.Name ||
    existingDevice.Type != device.Type ||
    existingDevice.HardwareName != device.HardwareName ||
    existingDevice.SubType != device.SubType;
```

**传感器变更条件**：
```csharp
bool sensorChanged = existingSensor == null ||
    existingSensor.Name != sensor.Name ||
    existingSensor.Type != sensor.Type ||
    existingSensor.Frequency != sensor.Frequency;
```

**点位变更条件**：
```csharp
bool pointChanged = existingPoint == null ||
    existingPoint.Name != point.Name ||
    existingPoint.Unit != point.Unit ||
    existingPoint.ValueType != point.ValueType ||
    existingPoint.Type != point.Type ||
    existingPoint.SrcPointId != point.SrcPointId;
```

## 统计信息

### 三层统计结构

```csharp
// 设备层统计
int deviceInsertCount = 0,     // 新增设备数
    deviceUpdateCount = 0,     // 更新设备数
    deviceNoChangeCount = 0,   // 无变化设备数
int deviceDeleteCount = 0;     // 删除设备数

// 传感器层统计
int sensorInsertCount = 0,     // 新增传感器数
    sensorUpdateCount = 0,     // 更新传感器数
    sensorNoChangeCount = 0;   // 无变化传感器数

// 点位层统计
int pointInsertCount = 0,      // 新增点位数
    pointUpdateCount = 0,      // 更新点位数
    pointNoChangeCount = 0;    // 无变化点位数
```

## 删除清理逻辑

### 1. 点位删除

```csharp
// 获取当前上报的点位属性列表
var currentPoints = sensorDto.Points.Select(p => p.Property).ToList();

// 查找不再存在的点位
var pointsToDelete = existingPoints.Where(p =>
    !currentPoints.Contains(p.Property)).ToList();

if (pointsToDelete.Any())
{
    foreach (var pointToDelete in pointsToDelete)
    {
        _logger.LogDebug($"[点位-删除] PointId={pointToDelete.Id}, ...");
    }
    await _pointRepository.DeleteAsync(pointsToDelete);
}
```

### 2. 传感器删除

```csharp
// 获取当前上报的传感器键值列表
var currentSensors = deviceDto.Sensors.Select(s => s.SensorKey).ToList();

// 查找不再存在的传感器
var sensorsToDelete = existingSensors.Where(s =>
    !currentSensors.Contains(s.SensorKey)).ToList();

if (sensorsToDelete.Any())
{
    // 先删除传感器下的所有点位
    await _pointRepository.DeleteAsync(p =>
        p.DeviceId == device.Id &&
        sensorsToDelete.Select(s => s.SensorKey).Contains(p.SensorKey));
    
    // 再删除传感器
    await _sensorRepository.DeleteAsync(sensorsToDelete);
}
```

### 3. 设备删除

```csharp
// 获取当前上报的设备ID列表
var currentDevices = eventData.Data.Devices.Select(d => d.DeviceId).ToList();

// 查找不再存在的设备
var devicesToDelete = existingDevices.Where(d =>
    !currentDevices.Contains(d.Id)).ToList();

if (devicesToDelete.Any())
{
    // 先删除设备下的所有点位和传感器
    foreach (var deviceToDelete in devicesToDelete)
    {
        await _pointRepository.DeleteAsync(p => p.DeviceId == deviceToDelete.Id);
        await _sensorRepository.DeleteAsync(s => s.DeviceId == deviceToDelete.Id);
    }
    
    // 再删除设备
    await _deviceRepository.DeleteAsync(devicesToDelete);
}
```

## 日志记录

### 信息日志 (LogInformation)

```csharp
// 处理开始
$"开始处理网关 {gatewayId} 的设备元数据"

// 处理完成（统计信息）
$"网关 {gatewayId} 的设备元数据处理完成: " +
$"设备[新增:{deviceInsertCount}, 更新:{deviceUpdateCount}, 无变化:{deviceNoChangeCount}, 删除:{deviceDeleteCount}] " +
$"传感器[新增:{sensorInsertCount}, 更新:{sensorUpdateCount}, 无变化:{sensorNoChangeCount}] " +
$"点位[新增:{pointInsertCount}, 更新:{pointUpdateCount}, 无变化:{pointNoChangeCount}]"

// 删除统计
$"删除了 {count} 个不再存在的点位"
$"删除了 {count} 个不再存在的传感器"
$"删除了 {count} 个不再存在的设备"
```

### 调试日志 (LogDebug)

```csharp
// 设备变更
$"[设备-新增] ID={id}, Name={name}, Type={type}"
$"[设备-更新] ID={id}, Name={oldName}->{newName}, Type={oldType}->{newType}"
$"[设备-无变化] ID={id}, Name={name}, Type={type}"

// 传感器变更
$"  [传感器-新增] DeviceId={deviceId}, SensorKey={sensorKey}, Name={name}, Frequency={frequency}"
$"  [传感器-更新] DeviceId={deviceId}, SensorKey={sensorKey}, Name={oldName}->{newName}, Frequency={oldFreq}->{newFreq}"
$"  [传感器-无变化] DeviceId={deviceId}, SensorKey={sensorKey}, Name={name}"

// 点位变更
$"    [点位-新增] PointId={pointId}, DeviceId={deviceId}, SensorKey={sensorKey}, Property={property}, Name={name}, ValueType={valueType}, Unit={unit}"
$"    [点位-更新] PointId={pointId}, Property={property}, Name={oldName}->{newName}, ValueType={oldValueType}->{newValueType}, Unit={oldUnit}->{newUnit}"
$"    [点位-无变化] PointId={pointId}, Property={property}, Name={name}, ValueType={valueType}, Unit={unit}"

// 删除操作
$"    [点位-删除] PointId={pointId}, DeviceId={deviceId}, SensorKey={sensorKey}, Property={property}, Name={name}, ValueType={valueType}, Unit={unit}"
$"  [传感器-删除] SensorKey={sensorKey}, Name={name}, Type={type}"
$"[设备-删除] ID={id}, Name={name}, Type={type}"
```

### 错误日志 (LogError)

```csharp
// 处理失败
$"处理网关 {gatewayId} 的设备元数据时发生错误"
```

## 三层实体结构

```
Device (设备)
  │
  ├─ DeviceId: 设备唯一标识
  ├─ Name: 设备名称
  ├─ Type: 设备类型
  ├─ SubType: 设备子类型
  ├─ HardwareName: 硬件名称
  ├─ Status: 设备状态
  └─ Sensors (传感器集合)
      │
      ├─ SensorKey: 传感器键值（设备内唯一）
      ├─ Name: 传感器名称
      ├─ Type: 传感器类型
      ├─ Frequency: 采集频率
      ├─ Status: 传感器状态
      └─ Points (点位集合)
          │
          ├─ Property: 属性名称（传感器内唯一）
          ├─ Name: 点位名称
          ├─ Unit: 单位
          ├─ ValueType: 值类型
          ├─ Type: 点位类型
          └─ SrcPointId: 源点位ID
```

## 性能优化

### 1. 批量查询

```csharp
// ❌ 错误：每个设备查询一次传感器
foreach (var device in devices)
{
    var sensors = await _sensorRepository.GetListAsync(s => s.DeviceId == device.Id);
}

// ✅ 正确：只查询一次所有设备
var existingDevices = await _deviceRepository.GetListAsync(d =>
    d.GatewayId == gatewayId);
```

### 2. 增量更新

```csharp
// 判断是否真的有变化
bool changed = existing == null ||
    existing.Name != new.Name ||
    existing.Type != new.Type;

if (changed)
{
    await repository.UpdateAsync(new);
}
```

### 3. 级联删除

- 先删除子实体（Point → Sensor → Device）
- 减少数据库约束检查
- 提高删除性能

## 错误处理

### 异常处理策略

```csharp
try
{
    // 处理逻辑
}
catch (Exception ex)
{
    _logger.LogError(ex, $"处理网关 {gatewayId} 的设备元数据时发生错误");
    throw;
}
```

## 相关实体

- [[DeviceEntity]] - 设备实体
- [[SensorEntity]] - 传感器实体
- [[AstPointEntity]] - 点位实体

## 相关事件

- `DeviceMetadataEventArgs` - 设备元数据事件参数

## 使用场景

### 1. 网关首次上报

网关首次连接时上报完整的设备元数据：
```csharp
// 所有设备、传感器、点位都是新增
deviceInsertCount = 10
sensorInsertCount = 50
pointInsertCount = 200
```

### 2. 设备配置变更

设备添加或删除传感器时：
```csharp
// 传感器变更
sensorInsertCount = 2      // 新增传感器
sensorDeleteCount = 1      // 删除传感器
pointInsertCount = 10      // 新增点位
pointDeleteCount = 5       // 删除点位
```

### 3. 点位属性变更

点位属性（名称、单位、值类型等）发生变化：
```csharp
// 点位更新
pointUpdateCount = 3       // 更新点位属性
```

### 4. 定期全量同步

网关定期上报完整元数据以保持数据一致性：
```csharp
// 全量同步，大部分无变化
deviceNoChangeCount = 8    // 设备无变化
sensorNoChangeCount = 45   // 传感器无变化
pointNoChangeCount = 180   // 点位无变化
deviceInsertCount = 2      // 新增设备
sensorUpdateCount = 5      // 更新传感器
pointDeleteCount = 10      // 删除点位
```

## 设计特点

1. **三层结构** - Device → Sensor → Point 层级关系
2. **增量更新** - 只更新真正变化的实体
3. **级联删除** - 自动清理子实体
4. **详细统计** - 记录每层的新增、更新、删除数量
5. **数据一致性** - 使用InsertOrUpdate确保幂等性
6. **详细日志** - 记录每个实体的变更情况

## 注意事项

1. **ID保持不变** - 设备、传感器、点位的ID在更新时保持不变
2. **创建时间保留** - 实体的CreationTime在更新时保持原值
3. **级联删除顺序** - Point → Sensor → Device，避免外键约束问题
4. **Property作为唯一键** - 点位通过Property字段判断是否同一实体
5. **SensorKey作为唯一键** - 传感器通过SensorKey字段判断是否同一实体

---

> **最后更新**：2026-06-04  
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Application/EventHandlers/DeviceMetadataEventHandler.cs`
