# Device - 设备实体

> 采集器设备表
>
> **位置**: `module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Gateway/DeviceEntity.cs`
>
> **表名**: `ast_device`

## 概述

`DeviceEntity` 表示智能变电站监控系统中的采集器设备实体，支持网关设备管理、在线状态监控、摄像头集成等功能。

## 属性列表

| 属性名 | 类型 | 数据库列名 | 说明 | 默认值 |
|--------|------|-----------|------|--------|
| `Id` | `string` | `id` | 主键 | - |
| `GatewayId` | `Guid` | `gateway_id` | 网关ID | - |
| `Name` | `string` | `name` | 显示名称 | - |
| `Status` | `DeviceStatusEnum` | `status` | 在线状态 | - |
| `Type` | `DeviceTypeEnum` | `type` | 设备类型 | `0` |
| `SubType` | `DeviceSubTypeEnum?` | `sub_type` | 子类型 | `0` |
| `HardwareName` | `string?` | `hardware_name` | 采集器硬件名称 | `null` |
| `CreationTime` | `DateTime` | `creation_time` | 创建时间 | - |
| `LastHeartbeatTime` | `DateTime?` | `last_heartbeat_time` | 最后心跳时间 | `null` |
| `NvrId` | `string?` | `nvr_id` | 所属录像机ID | `null` |
| `MonitoredObjectId` | `Guid?` | `monitored_object_id` | 被监测对象ID | `null` |
| `CameraDeviceType` | `CameraDeviceTypeEnum?` | `camera_device_type` | 摄像头设备类型 | `null` |
| `CameraVideoType` | `CameraVideoTypeEnum?` | `camera_video_type` | 视频类型 | `null` |
| `CameraPresetType` | `CameraPresetTypeEnum?` | `camera_preset_type` | 预置位类型 | `null` |
| `CameraHostAddress` | `string?` | `camera_host_address` | 摄像头主机地址 | `null` |
| `CameraUsername` | `string?` | `camera_username` | 摄像头用户名 | `null` |
| `CameraPassword` | `string?` | `camera_password` | 摄像头密码 | `null` |
| `CameraManufacturer` | `string?` | `camera_manufacturer` | 摄像头厂商 | `null` |
| `Remark` | `string?` | `remark` | 详情描述 | `null` |
| `ExtInfo` | `string?` | `ext_info` | 扩展参数 | `null` |
| `IsDisplay` | `bool` | `is_display` | 是否在界面上展示 | `true` |

## 方法列表

| 方法名 | 参数 | 返回值 | 说明 |
|--------|------|--------|------|
| `GetKeys` | - | `object[]` | 返回主键数组 |
| `SetId` | `string id` | `void` | 设置设备ID（内部方法） |

## 导航属性

| 属性名 | 类型 | 关联 | 说明 |
|--------|------|------|------|
| `Gateway` | `GatewayAggregateRoot` | OneToOne | 关联的网关实体 |
| `MonitoredObject` | `MonitoredObjectAggregateRoot` | OneToOne | 关联的监测对象实体 |

## 数据注释

```csharp
[SugarTable("ast_device")]
public class DeviceEntity : Entity<string>
```

### 主键配置

```csharp
[SugarColumn(IsPrimaryKey = true)]
public override string Id { get; protected set; }
```

### 索引

- 主键索引：`Id`

## 枚举类型

### DeviceStatusEnum - 设备状态

| 值 | 名称 | 描述 |
|----|------|------|
| `0` | `Normal` | 正常 |
| `1` | `Offline` | 离线 |
| `2` | `Fault` | 故障 |

### DeviceTypeEnum - 设备类型

参见 `DeviceTypeEnum.cs`

### DeviceSubTypeEnum - 设备子类型

参见相关枚举文件

## 关系图

```
DeviceEntity (设备)
├── GatewayAggregateRoot (网关) [OneToOne]
├── MonitoredObjectAggregateRoot (监测对象) [OneToOne]
└── 关联到
    ├── AstPointEntity (点位数据)
    ├── VoiceprintDeviceAudioRecordEntity (声纹音频记录)
    └── 告警相关实体
```

## 使用示例

### 创建新设备

```csharp
var device = new DeviceEntity();
device.SetId(Guid.NewGuid().ToString());
device.Name = "变压器温度传感器";
device.GatewayId = gatewayId;
device.Status = DeviceStatusEnum.Normal;
device.Type = DeviceTypeEnum.Sensor;
device.CreationTime = DateTime.Now;
```

### 更新心跳时间

```csharp
device.LastHeartbeatTime = DateTime.Now;
```

### 关联监测对象

```csharp
device.MonitoredObjectId = monitoredObjectId;
device.IsDisplay = true;
```

## 相关服务

- `DeviceService` - 设备管理服务
- `DeviceStatusDto` - 设备状态DTO
- `DeviceListDto` - 设备列表DTO

## 注意事项

1. **主键类型**: 使用 `string` 类型作为主键，需通过 `SetId` 方法设置
2. **软删除**: 实现了 `ISoftDelete` 接口
3. **审计字段**: 继承自 `Entity` 基类，包含审计字段
4. **导航属性**: 使用 `[Navigate]` 特性配置关联关系
5. **默认值**: `IsDisplay` 默认为 `true`

---

**最后更新**: 2026-06-03
**模块**: `ast-intellisub`
