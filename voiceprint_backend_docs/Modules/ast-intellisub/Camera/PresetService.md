# 预置位服务 (PresetService)

## 概述

预置位服务负责管理摄像机云台预置位，提供预置位的创建、更新、删除、查询等核心功能。预置位是云台摄像头的预设位置，用于快速定位到特定监控区域。

**位置**：`module/ast-intellisub/Ast.IntelliSub.Application/Services/PresetService.cs`  
**层**：Application  
**模块**：ast-intellisub  
**依赖注入**：Scoped

## 职责

- 管理预置位基本信息
- 关联预置位与监测对象
- 协调媒体服务进行预置位操作
- 自动拍摄预置位照片
- 管理预置位与巡视任务的关联
- 清理预置位相关的识别区域点位

## 主要接口

### 根据摄像头获取预置点列表

```csharp
Task<List<PresetDto>> GetByCameraIdAsync(string cameraDeviceId)
```

**功能**：获取指定摄像机下的所有预置点

**请求参数**：
- `cameraDeviceId`：摄像机设备ID

**返回数据**：
```csharp
class PresetDto
{
    Guid Id                      // 预置点ID
    string Name                  // 预置点名称
    string CameraDeviceId        // 摄像头设备ID
    string SensorKey             // 传感器键（格式：preset{编号}）
    short? PresetNo              // 预置位编号（1-255）
    string? PresetFilePath       // 预置点照片路径
    CommonStatusEnum Status      // 状态
    DateTime CreationTime        // 创建时间
    string? ExtInfo              // 扩展信息
    bool IsEnabled               // 是否启用
    Guid? MonitoredObjectId      // 关联监测对象ID
}
```

### 根据监测对象获取预置点列表

```csharp
Task<List<PresetDetailDto>> GetByMonitoredObjectIdAsync(Guid monitoredObjectId)
```

**功能**：获取指定监测对象关联的所有预置点

**请求参数**：
- `monitoredObjectId`：监测对象ID

**返回数据**：包含流媒体URL的详细预置点信息

```csharp
class PresetDetailDto : PresetDto
{
    string NvrId                 // 录像机ID
    string NvrName               // 录像机名称
    string CameraDeviceName      // 摄像头名称
    string StreamUrl             // 流媒体URL
}
```

### 创建预置点

```csharp
Task<PresetDto> CreateAsync(PresetCreateDto input)
```

**功能**：创建新的预置点并自动拍摄预置点照片

**请求参数**：
```csharp
class PresetCreateDto
{
    string CameraDeviceId        // 摄像头设备ID
    string Name                  // 预置点名称
    bool IsEnabled               // 是否启用
    Guid? MonitoredObjectId      // 关联监测对象ID（可选）
}
```

**创建流程**：
1. 验证摄像头设备是否存在
2. 验证监测对象是否存在（如果提供）
3. 生成预置位编号（同一摄像头下1-255递增）
4. 生成SensorKey（格式：preset{编号}）
5. 创建预置点实体
6. 创建监测对象关联记录（如果提供）
7. 调用媒体服务在流媒体网关中添加预置位
8. 拍摄预置点照片并更新PresetFilePath

**预置位编号生成规则**：
- 查询摄像头下现有预置位的最大编号
- 从1开始递增，跳过已占用的编号
- 编号范围：1-255
- 达到255个后抛出异常

### 更新预置点

```csharp
Task<PresetDto> UpdateAsync(Guid id, PresetUpdateDto input)
```

**功能**：更新预置点信息，可选择重新拍摄预置点照片

**请求参数**：
```csharp
class PresetUpdateDto
{
    string Name                  // 预置点名称
    bool? IsEnabled              // 是否启用
    bool? shouldUpdatePresetImage // 是否重新拍摄预置点照片
    Guid? MonitoredObjectId      // 关联监测对象ID（可选）
}
```

**更新流程**：
1. 验证预置点是否存在
2. 更新基本信息（名称、启用状态）
3. 处理监测对象关联关系（创建、更新或删除）
4. 如果需要重新拍摄照片：
   - 调用媒体服务更新预置位
   - 等待1秒确保预置位更新完成
   - 拍摄预置点照片并更新路径

### 删除预置点

```csharp
Task DeleteAsync(Guid id)
```

**功能**：删除预置点及其所有关联数据

**删除流程**：
1. 验证预置点是否存在
2. 检查预置点是否被巡视任务使用
3. 调用媒体服务删除流媒体网关中的预置位
4. 删除监测对象关联记录
5. 删除相关的识别区域点位（ast_point表）
6. 删除预置点照片文件
7. 删除预置点记录

**关联删除**：
- 监测对象关联记录（MonitoredObjectPresetRelEntity）
- 识别区域点位（AstPointEntity，Type=RecognitionArea）
- 预置点照片文件（PresetFilePath）

## 预置位编号生成

### 编号规则

```
同一摄像头设备下：
- 编号范围：1-255
- 生成格式：preset{编号}
- 分配策略：查找最小可用编号
- 上限处理：达到255个时抛出异常
```

### 生成示例

```
摄像头A下已有预置点：preset1, preset2, preset5
新创建预置点将分配：preset3

摄像头B下已有预置点：preset1, preset2, ..., preset255
新创建预置点将抛出异常：已达到上限
```

### SensorKey 生成

```csharp
var sensorKey = $"preset{presetNo}";  // preset1, preset2, ..., preset255
```

## 预置位照片管理

### 照片拍摄时机

1. **创建预置点时**：自动拍摄第一张照片
2. **更新预置点时**：可选择重新拍摄照片
3. **调用预置位时**：MediaService会自动拍摄

### 照片存储路径

照片路径存储在 `PresetFilePath` 字段，格式为Web访问路径：
```
/snapshot/2026-06-03/{nvrId}/{cameraId}/{timestamp}.jpg
```

### 照片删除

删除预置点时会尝试删除照片文件：
- 检查文件是否存在
- 删除物理文件
- 记录删除日志
- 失败不抛出异常（容忍错误）

## 监测对象关联

### 关联关系

```
MonitoredObject (监测对象)
    ↓ 1:N
MonitoredObjectPresetRel (关联表)
    ↓ N:1
Sensor (预置点)
```

### 关联操作

**创建关联**：
```csharp
var presetRel = new MonitoredObjectPresetRelEntity
{
    MonitoredObjectId = monitoredObjectId,
    AstSensorId = presetId,
    OrderNum = 1
};
```

**更新关联**：
- 如果提供新的监测对象ID，更新关联记录
- 如果未提供监测对象ID，删除现有关联

**删除关联**：
- 删除预置点时自动删除关联记录

## 巡视任务检查

### 使用检查

删除预置点前会检查是否被巡视任务使用：

```csharp
private async Task<bool> CheckIfPresetUsedInPatrolAsync(Guid presetId)
{
    // TODO: 检查巡视任务关联表
    // 暂时返回false，避免阻塞删除操作
    return false;
}
```

**未来实现**：
- 检查巡视任务预置位关联表
- 检查巡视任务状态
- 活跃任务的预置点不允许删除

## 数据流

### 创建预置点数据流

```
前端请求 → PresetService.CreateAsync()
  ↓
验证摄像头和监测对象
  ↓
生成预置位编号（1-255）
  ↓
创建 SensorEntity 预置点记录
  ↓
创建 MonitoredObjectPresetRelEntity 关联记录
  ↓
调用 MediaService.AddOrUpdatePresetAsync()
  ↓
调用 MediaService.CaptureAndSaveSnapshotAsync()
  ↓
更新 PresetFilePath
  ↓
返回 PresetDto
```

### 查询预置点数据流

```
前端请求 → PresetService.GetByCameraIdAsync()
  ↓
查询 SensorEntity（Type=PresetPoint）
  ↓
查询 MonitoredObjectPresetRelEntity
  ↓
构建 PresetDto 列表
  ↓
返回结果
```

## 依赖服务

- [[MediaService]] - 媒体服务（预置位操作、截图）
- [[StreamingUrlManager]] - 流媒体URL管理器
- [[NvrService]] - 录像机服务
- [[DeviceService]] - 设备服务
- [[MonitoredObjectService]] - 监测对象服务

## 相关实体

- [[SensorEntity]] - 传感器实体（预置点）
- [[DeviceEntity]] - 设备实体（摄像机）
- [[MonitoredObjectAggregateRoot]] - 监测对象实体
- [[MonitoredObjectPresetRelEntity]] - 预置点关联实体
- [[AstPointEntity]] - 识别区域点位
- [[NvrEntity]] - 录像机实体

## 权限控制

```csharp
[Authorize]  // 需要认证
public class PresetService : ApplicationService, IPresetService
```

所有接口都需要身份认证。

## 异常处理

| 异常类型 | 场景 |
|---------|------|
| UserFriendlyException "404" | 摄像头设备不存在<br>监测对象不存在<br>录像机设备不存在<br>预置点不存在 |
| UserFriendlyException "400" | 设备类型不是摄像机<br>摄像头未配置录像机<br>预置点数量已达上限<br>该记录不是预置点<br>预置点正在被巡视任务使用 |
| UserFriendlyException "500" | 系统内部错误 |

## API 路径

| 功能 | 路径 | 方法 |
|-----|------|------|
| 获取摄像头预置点 | `/api/app/preset/by-camera/{cameraDeviceId}` | GET |
| 获取监测对象预置点 | `/api/app/preset/by-monitored-object/{monitoredObjectId}` | GET |
| 创建预置点 | `/api/app/preset` | POST |
| 更新预置点 | `/api/app/preset/{id}` | PUT |
| 删除预置点 | `/api/app/preset/{id}` | DELETE |

## 使用场景

1. **巡视任务配置**：为监测对象配置预置点用于巡视
2. **实时监控**：快速定位到特定监控区域
3. **图像采集**：定期拍摄预置点照片用于分析
4. **设备管理**：管理摄像头的预置位配置

## 注意事项

1. **编号限制**：每个摄像头最多255个预置点
2. **自动编号**：系统自动分配预置位编号，无需手动指定
3. **照片拍摄**：创建和更新时可选择自动拍摄预置点照片
4. **关联删除**：删除预置点会删除所有关联数据
5. **巡视检查**：正在被巡视任务使用的预置点不能删除（待实现）
6. **错误容忍**：媒体服务操作失败不会中断主流程
7. **日志记录**：所有操作都有详细日志便于排查问题

## 配置依赖

该服务依赖以下配置：

```json
{
  "Patrol": {
    "PresetStabilizeDelayMs": 3000  // 预置位稳定等待时间
  },
  "StreamingApi": {
    "BaseUrl": "http://streaming-server",
    "SnapshotRootPath": "/store/snapshot"
  }
}
```

## 数据清理

### 删除预置点时的清理操作

1. **监测对象关联**：删除 MonitoredObjectPresetRelEntity 记录
2. **识别区域点位**：删除相关的 AstPointEntity 记录
3. **预置点照片**：删除物理照片文件
4. **流媒体网关**：调用媒体服务删除流媒体网关中的预置位
5. **预置点记录**：删除 SensorEntity 预置点记录

### 识别区域点位删除条件

```csharp
var relatedAstPoints = await _astPointRepository._DbQueryable
    .Where(x => x.DeviceId == preset.DeviceId && 
               x.SensorKey == preset.SensorKey && 
               x.Type == AstPointTypeEnum.RecognitionArea)
    .ToListAsync();
```

匹配条件：
- 设备ID相同
- SensorKey相同
- 类型为识别区域（RecognitionArea）
