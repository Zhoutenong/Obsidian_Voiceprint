# SensorEntity

## 概述

`SensorEntity` 代表连接到网关的传感器设备或预置点位。传感器用于采集环境数据（温度、湿度、声音等），预置点位则是摄像机可移动的预设位置。

## 表信息

- **表名**: `ast_sensor`
- **主键**: `Id` (Guid)
- **继承**: `Entity<Guid>`
- **命名空间**: `Ast.IntelliSub.Domain.Entities`

## 字段列表

| 字段名 | 类型 | 数据库列名 | 说明 | 约束 |
|--------|------|-----------|------|------|
| `Id` | `Guid` | `id` | 主键 | PK |
| `SensorKey` | `string` | `sensor_key` | 传感器唯一标识 | Unique |
| `DeviceId` | `string` | `device_id` | 设备ID | 关联设备 |
| `Name` | `string` | `name` | 显示名称 | NOT NULL |
| `Type` | `SensorTypeEnum` | `type` | 类型 | Enum (0:传感器,1:预置点位) |
| `Status` | `CommonStatusEnum` | `status` | 状态 | Enum |
| `Frequency` | `int?` | `frequency` | 采集频率(毫秒) | Nullable |
| `CreationTime` | `DateTime` | `creation_time` | 创建时间 | |
| `PresetNo` | `short?` | `preset_no` | 预置位号 | Nullable (仅预置点位) |
| `PresetFilePath` | `string?` | `preset_file_path` | 预置位文件路径 | Nullable |
| `PresetPtz` | `string?` | `preset_ptz` | 预置位PTZ参数 | JSON格式 |
| `Remark` | `string?` | `remark` | 详情描述 | |
| `ExtInfo` | `string?` | `ext_info` | 扩展参数 | JSON格式 |
| `IsEnabled` | `bool?` | `is_enabled` | 是否展示 | Default: false |

## 关联实体 (ER 关系)

### 关系说明

`SensorEntity` 通过 `DeviceId` 字段关联到 `DeviceEntity`，表示传感器所属的采集设备。

### 关系图

```
DeviceEntity (1) ←──→ (N) SensorEntity
     │
     └── GatewayAggregateRoot
```

## 被哪些服务读写

### 写入服务

- `PresetService` — 预置点位 CRUD、PTZ参数更新
- `DeviceService` — 传感器设备管理

### 读取服务

- `RealtimeMonitoringPointService` — 实时监控数据查询
- `PatrolExecutionService` — 巡检任务执行时读取预置点位信息
- `ReportService` — 报告生成时获取传感器配置

## 业务规则约束

### 1. 唯一性约束

- `SensorKey` + `DeviceId` 组合应唯一（索引注释显示）

### 2. 类型约束

#### SensorTypeEnum

```csharp
public enum SensorTypeEnum
{
    Sensor = 0,      // 物理传感器（温度、湿度等）
    Preset = 1       // 摄像机预置点位
}
```

#### CommonStatusEnum

```csharp
public enum CommonStatusEnum
{
    Disabled = 0,    // 禁用
    Enabled = 1,     // 启用
    Error = 2        // 故障
}
```

### 3. 预置点位规则

当 `Type = SensorTypeEnum.Preset` 时：
- `PresetNo` 必须有效（1-255）
- `PresetPtz` 存储 PTZ (Pan-Tilt-Zoom) 参数，格式示例：
  ```json
  {
    "pan": 120.5,
    "tilt": -15.0,
    "zoom": 10.0
  }
```

### 4. 采集频率

- `Frequency` 单位为毫秒
- 推荐值：1000ms（1秒）至 60000ms（60秒）
- 值为 `null` 表示使用默认频率

### 5. 启用状态

- `IsEnabled = false` 的传感器在监控界面不展示
- 巡检任务会跳过禁用的预置点位

## 扩展信息 (ExtInfo)

`ExtInfo` 字段存储 JSON 格式的扩展参数：

```json
{
  "unit": "℃",           // 测量单位
  "minValue": -40,       // 最小值
  "maxValue": 120,       // 最大小值
  "accuracy": 0.1,       // 精度
  "model": "DHT22",      // 传感器型号
  "manufacturer": "XXX"   // 制造商
}
```

## 索引定义

```csharp
// 注释中的唯一索引定义（当前被注释）
[SugarIndex("IX_SENSOR_UNIQUE", nameof(DeviceId) + "," + nameof(SensorKey), OrderByType.Asc, true)]
```

建议启用此索引以确保数据完整性：

```sql
CREATE UNIQUE INDEX IX_SENSOR_UNIQUE ON ast_sensor(device_id, sensor_key);
```

## 使用场景

### 1. 环境监测

```csharp
// 创建温度传感器
var temperatureSensor = new SensorEntity
{
    SensorKey = "TEMP_001",
    DeviceId = "DEVICE_001",
    Name = "1号机柜温度",
    Type = SensorTypeEnum.Sensor,
    Status = CommonStatusEnum.Enabled,
    Frequency = 5000,  // 5秒采集一次
    ExtInfo = "{\"unit\":\"℃\",\"minValue\":-40,\"maxValue\":120}"
};
```

### 2. 摄像机预置位

```csharp
// 创建预置点位
var preset = new SensorEntity
{
    SensorKey = "PRESET_001",
    DeviceId = "CAMERA_001",
    Name = "主变压器位置",
    Type = SensorTypeEnum.Preset,
    PresetNo = 1,
    PresetPtz = "{\"pan\":120.5,\"tilt\":-15.0,\"zoom\":10.0}",
    IsEnabled = true
};
```

## 相关文件

- **源码**: `module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Gateway/SensorEntity.cs`
- **服务**: `module/ast-intellisub/Ast.IntelliSub.Application/Services/PresetService.cs`
- **枚举**: `module/ast-intellisub/Ast.IntelliSub.Domain.Shared/Enums/`
