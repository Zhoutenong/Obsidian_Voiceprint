# DeviceEntity (Camera/摄像机设备)

## 概述

`DeviceEntity` 代表连接到网关的采集设备，包括传感器采集器、摄像机、NVR 等物理设备。摄像机是视觉网关下的主要设备类型，用于视频采集和巡检。

## 表信息

- **表名**: `ast_device`
- **主键**: `Id` (string)
- **继承**: `Entity<string>`
- **命名空间**: `Ast.IntelliSub.Domain.Entities`

## 字段列表

| 字段名 | 类型 | 数据库列名 | 说明 | 约束 |
|--------|------|-----------|------|------|
| `Id` | `string` | `id` | 主键（设备唯一标识） | PK |
| `GatewayId` | `Guid` | `gateway_id` | 所属网关ID | FK |
| `Name` | `string` | `name` | 设备名称 | NOT NULL |
| `Status` | `DeviceStatusEnum` | `status` | 在线状态 | Enum |
| `Type` | `DeviceTypeEnum` | `type` | 设备类型 | Enum, Default: 0 |
| `SubType` | `DeviceSubTypeEnum?` | `sub_type` | 子类型 | Enum, Default: 0 |
| `HardwareName` | `string?` | `hardware_name` | 硬件名称 | |
| `CreationTime` | `DateTime` | `creation_time` | 创建时间 | |
| `LastHeartbeatTime` | `DateTime?` | `last_heartbeat_time` | 最后心跳时间 | Nullable |
| `NvrId` | `string?` | `nvr_id` | 所属录像机ID | FK |
| `MonitoredObjectId` | `Guid?` | `monitored_object_id` | 被监测对象ID | FK |
| `CameraDeviceType` | `CameraDeviceTypeEnum?` | `camera_device_type` | 摄像头设备类型 | Enum |
| `CameraVideoType` | `CameraVideoTypeEnum?` | `camera_video_type` | 视频类型 | Enum |
| `CameraPresetType` | `CameraPresetTypeEnum?` | `camera_preset_type` | 预置位类型 | Enum |
| `CameraHostAddress` | `string?` | `camera_host_address` | 摄像头地址 | |
| `CameraUsername` | `string?` | `camera_username` | 摄像头用户名 | 加密存储 |
| `CameraPassword` | `string?` | `camera_password` | 摄像头密码 | 加密存储 |
| `CameraManufacturer` | `string?` | `camera_manufacturer` | 摄像头厂商 | |
| `Remark` | `string?` | `remark` | 详情描述 | |
| `ExtInfo` | `string?` | `ext_info` | 扩展参数 | JSON格式 |
| `IsDisplay` | `bool` | `is_display` | 是否在界面展示 | Default: true |

## 关联实体 (ER 关系)

### 导航属性

```csharp
// 所属网关
[Navigate(NavigateType.OneToOne, nameof(GatewayId))]
public GatewayAggregateRoot Gateway { get; set; }

// 被监测对象
[Navigate(NavigateType.OneToOne, nameof(MonitoredObjectId))]
public MonitoredObjectAggregateRoot MonitoredObject { get; set; }
```

### 关系图

```
GatewayAggregateRoot (1) ←──→ (N) DeviceEntity
                                      │
                                      ├── (1) NvrEntity (通过 NvrId)
                                      └── (1) MonitoredObjectAggregateRoot
```

## 被哪些服务读写

### 写入服务

- `DeviceService` — 设备 CRUD、状态更新
- `CameraService` — 摄像机设备管理
- `StreamingGatewayService` — 流媒体网关设备管理

### 读取服务

- `RealtimeMonitoringPointService` — 实时监控设备状态
- `PatrolExecutionService` — 巡检任务执行时访问摄像机
- `MediaService` — 视频流、录像回放
- `ReportService` — 报告生成时获取设备信息

## 业务规则约束

### 1. 枚举约束

#### DeviceTypeEnum

```csharp
public enum DeviceTypeEnum
{
    Sensor = 0,        // 传感器采集器
    Camera = 1,        // 摄像机
    NVR = 2            // 录像机
}
```

#### DeviceSubTypeEnum

```csharp
public enum DeviceSubTypeEnum
{
    PTZ = 0,          // 云台摄像机
    Fixed = 1,        // 固定摄像机
    Dome = 2,         // 半球摄像机
    Bullet = 3        // 枪机摄像机
}
```

#### CameraDeviceTypeEnum

```csharp
public enum CameraDeviceTypeEnum
{
    IPC = 0,          // 网络摄像机
    Analog = 1,       // 模拟摄像机
    SDI = 2,          // SDI摄像机
    USB = 3           // USB摄像机
}
```

#### CameraVideoTypeEnum

```csharp
public enum CameraVideoTypeEnum
{
    H264 = 0,         // H.264编码
    H265 = 1,         // H.265编码
    MJPEG = 2         // MJPEG编码
}
```

#### CameraPresetTypeEnum

```csharp
public enum CameraPresetTypeEnum
{
    None = 0,         // 无预置位
    Standard = 1,     // 标准预置位（1-255）
    Cruise = 2,      // 巡航路径
    Pattern = 3      // 轨迹扫描
}
```

#### DeviceStatusEnum

```csharp
public enum DeviceStatusEnum
{
    Offline = 0,      // 离线
    Online = 1,       // 在线
    Fault = 2         // 故障
}
```

### 2. 心跳机制

- `LastHeartbeatTime` 通过 Hangfire 后台任务定期更新
- 设备离线判断：`LastHeartbeatTime` 超过配置的超时时间（默认 5 分钟）
- 离线设备在监控界面标记为灰色

### 3. 摄像机字段约束

当 `Type = DeviceTypeEnum.Camera` 时：
- `CameraDeviceType` 必须有效
- `CameraVideoType` 必须有效
- `CameraHostAddress` 必须提供（IP 地址或域名）
- `CameraUsername` 和 `CameraPassword` 用于 RTSP/ONVIF 认证

### 4. 显示状态

- `IsDisplay = false` 的设备不在监控界面显示
- 巡检任务会跳过隐藏的设备

### 5. 安全约束

- `CameraUsername` 和 `CameraPassword` 应加密存储
- 不得在日志或错误消息中暴露密码

## 使用场景

### 1. 创建 PTZ 摄像机

```csharp
var ptzCamera = new DeviceEntity
{
    Id = "CAMERA_PTZ_001",
    GatewayId = gatewayId,
    Name = "1号机柜云台摄像机",
    Status = DeviceStatusEnum.Online,
    Type = DeviceTypeEnum.Camera,
    SubType = DeviceSubTypeEnum.PTZ,
    HardwareName = "HIKVISION-DS-2DC3304IW",
    MonitoredObjectId = objectId,
    CameraDeviceType = CameraDeviceTypeEnum.IPC,
    CameraVideoType = CameraVideoTypeEnum.H265,
    CameraPresetType = CameraPresetTypeEnum.Standard,
    CameraHostAddress = "192.168.1.101",
    CameraUsername = "admin",
    CameraPassword = "encrypted_password",
    CameraManufacturer = "HIKVISION",
    IsDisplay = true
};
```

### 2. 创建固定摄像机

```csharp
var fixedCamera = new DeviceEntity
{
    Id = "CAMERA_FIXED_001",
    GatewayId = gatewayId,
    Name = "大门固定摄像机",
    Status = DeviceStatusEnum.Online,
    Type = DeviceTypeEnum.Camera,
    SubType = DeviceSubTypeEnum.Fixed,
    MonitoredObjectId = objectId,
    CameraDeviceType = CameraDeviceTypeEnum.IPC,
    CameraVideoType = CameraVideoTypeEnum.H264,
    CameraPresetType = CameraPresetTypeEnum.None,
    CameraHostAddress = "192.168.1.102",
    CameraUsername = "admin",
    CameraPassword = "encrypted_password",
    CameraManufacturer = "DAHUA",
    IsDisplay = true
};
```

### 3. 创建传感器采集器

```csharp
var sensorCollector = new DeviceEntity
{
    Id = "SENSOR_COLLECTOR_001",
    GatewayId = gatewayId,
    Name = "温湿度采集器",
    Status = DeviceStatusEnum.Online,
    Type = DeviceTypeEnum.Sensor,
    HardwareName = "SENSOR-HUB-01",
    MonitoredObjectId = objectId,
    ExtInfo = "{\"sensorCount\":8,\"protocol\":\"Modbus\"}",
    IsDisplay = true
};
```

## 扩展信息 (ExtInfo)

`ExtInfo` 存储设备特定配置：

### 摄像机扩展信息

```json
{
  "rtspPort": 554,
  "onvifPort": 80,
  "resolution": "1920x1080",
  "framerate": 25,
  "bitrate": 4096,
  "audioEnabled": true,
  "nightVision": true,
  "wdr": true,
  "regionOfInterest": [
    { "name": "main_transformer", "polygon": [[0,0], [100,0], [100,100], [0,100]] }
  ]
}
```

### 传感器采集器扩展信息

```json
{
  "sensorCount": 8,
  "protocol": "Modbus",
  "baudRate": 9600,
  "dataBits": 8,
  "stopBits": 1,
  "parity": "None",
  "sensors": [
    { "id": "TEMP_001", "type": "Temperature", "address": 40001 },
    { "id": "HUM_001", "type": "Humidity", "address": 40002 }
  ]
}
```

## 索引建议

```sql
-- 网关索引（用于查询网关下的所有设备）
CREATE INDEX IX_DEVICE_GATEWAY_ID ON ast_device(gateway_id);

-- 设备类型索引
CREATE INDEX IX_DEVICE_TYPE ON ast_device(type);

-- 在线状态索引
CREATE INDEX IX_DEVICE_STATUS ON ast_device(status);

-- NVR 索引（用于查询 NVR 下的摄像机）
CREATE INDEX IX_DEVICE_NVR_ID ON ast_device(nvr_id);

-- 被监测对象索引
CREATE INDEX IX_DEVICE_MONITORED_OBJECT_ID ON ast_device(monitored_object_id);

-- 复合索引（网关 + 类型 + 状态，用于查询在线摄像机）
CREATE INDEX IX_DEVICE_GATEWAY_TYPE_STATUS ON ast_device(gateway_id, type, status);
```

## 设备状态管理

### 心跳检测

```csharp
// Hangfire 后台任务
public class DeviceHeartbeatJob
{
    public async Task ExecuteAsync()
    {
        var timeout = DateTime.Now.AddMinutes(-5);

        // 标记超时设备为离线
        var offlineDevices = await _deviceRepo.GetListAsync(
            d => d.LastHeartbeatTime < timeout && d.Status == DeviceStatusEnum.Online
        );

        foreach (var device in offlineDevices)
        {
            device.Status = DeviceStatusEnum.Offline;
            await _deviceRepo.UpdateAsync(device);

            // 发送离线告警
            await _alarmService.SendDeviceOfflineAlarmAsync(device);
        }
    }
}
```

### 设备统计

```csharp
public async Task<DeviceStatisticsDto> GetStatisticsAsync(Guid gatewayId)
{
    var devices = await _deviceRepo.GetListAsync(d => d.GatewayId == gatewayId);

    return new DeviceStatisticsDto
    {
        Total = devices.Count,
        Online = devices.Count(d => d.Status == DeviceStatusEnum.Online),
        Offline = devices.Count(d => d.Status == DeviceStatusEnum.Offline),
        Fault = devices.Count(d => d.Status == DeviceStatusEnum.Fault),
        Cameras = devices.Count(d => d.Type == DeviceTypeEnum.Camera),
        Sensors = devices.Count(d => d.Type == DeviceTypeEnum.Sensor)
    };
}
```

## 相关文件

- **源码**: `module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Gateway/DeviceEntity.cs`
- **服务**: `module/ast-intellisub/Ast.IntelliSub.Application/Services/CameraService.cs`
- **媒体服务**: `module/ast-intellisub/Ast.IntelliSub.Application/Services/MediaService.cs`
- **枚举**: `module/ast-intellisub/Ast.IntelliSub.Domain.Shared/Enums/Camera*.cs`
- **网关实体**: `Classes/Gateway/GatewayAggregateRoot.md`
- **NVR实体**: `Classes/Gateway/NvrEntity.md`
