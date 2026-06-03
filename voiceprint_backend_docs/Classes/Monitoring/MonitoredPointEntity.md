# MonitoredPointEntity

监测点位实体，定义变电站设备上需要监测的具体点位。

## 表信息

- **表名**: `ast_monitored_point`
- **主键**: `Id` (Guid)

## 字段列表

| 字段名 | 类型 | 说明 | 外键 |
|--------|------|------|------|
| `Id` | Guid | 主键 | - |
| `Name` | string | 显示名称 | - |
| `Description` | string? | 描述信息 | - |
| `MonitoredItemId` | Guid | 被监测项ID | MonitoredItem |
| `DataBindingTypeId` | Guid? | 数据绑定类型ID | DataType |
| `OrderNum` | int? | 排序号 | - |
| `Icon` | string? | 图标 | - |
| `DataSrcType` | enum? | 数据来源类型 | - |
| `IsDisplay` | bool | 是否在界面上展示（默认 true） | - |

### 枚举类型

**DataSrcType** (DataSrcTypeEnum):
- `Manual` — 人工录入
- `IEC61850` — IEC61850 协议
- `IEC104` — IEC104 协议
- `Modbus` — Modbus 协议
- `Sensor` — 传感器直连

## 关联实体

### 出站关系 (Outbound)

- **MonitoredItem** (via `MonitoredItemId`) — 关联的监测项（导航属性）
- **DataType** (via `DataBindingTypeId`) — 数据绑定类型

### 入站关系 (Inbound)

- **PatrolTaskMonitoredPointRelEntity** — 巡检任务关联此点位
- **VoiceprintDeviceAudioRecordEntity** — 声纹音频记录关联此监测对象
- **PointDataEntity** — 时序数据点位（如果启用传感器数据采集）

## 服务读写

### 读取服务

- **MonitoredPointService** — 查询点位列表、按监测项分组、获取数据绑定项
- **PatrolTaskService** — 获取巡检任务关联的点位
- **VoiceprintCaptureService** — 获取需要采集声纹的点位

### 写入服务

- **MonitoredPointService** — 创建、更新、删除点位
- **PatrolTaskService** — 配置巡检任务时关联点位

## 业务规则

1. **监测项绑定**: 每个点位必须绑定到一个 `MonitoredItem`（如 "温度"、"声音"）
2. **数据源配置**: `DataSrcType` 决定数据采集方式
3. **显示控制**: `IsDisplay = false` 可隐藏不需要展示的点位
4. **排序**: `OrderNum` 用于控制点位在界面上的显示顺序
5. **图标**: `Icon` 字段存储图标标识（如 FontAwesome 类名）

## 数据流程

```
配置阶段: MonitoredItem → MonitoredPoint → PatrolTaskMonitoredPointRel
巡检阶段: PatrolTask → PatrolRecord → PatrolRecordItem (引用 MonitoredPoint)
声纹监测: MonitoredPoint → VoiceprintDeviceAudioRecord → VoiceprintAlarmRecord
```

## 源码位置

`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Monitoring/MonitoredPointEntity.cs`
