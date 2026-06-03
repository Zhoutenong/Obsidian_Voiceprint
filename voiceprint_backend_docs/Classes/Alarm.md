# Alarm - 告警实体

> 声纹告警记录表
>
> **位置**: `module/ast-voiceprint/Ast.Voiceprint.Domain/Entities/VoiceprintAlarmRecordEntity.cs`
>
> **表名**: `vp_voiceprint_alarm`

## 概述

`VoiceprintAlarmRecordEntity` 表示声纹分析产生的告警记录实体，记录异常声纹识别结果、告警状态和处理进度。

## 属性列表

| 属性名 | 类型 | 数据库列名 | 说明 | 默认值 |
|--------|------|-----------|------|--------|
| `Id` | `Guid` | `id` | 主键 | - |
| `MonitoredObjectId` | `Guid` | `monitored_object_id` | 关联监测对象ID | - |
| `DeviceAudioRecordId` | `Guid` | `device_audio_record_id` | 对应的音频记录ID | - |
| `AlarmMessage` | `string` | `alarm_message` | 告警信息（最大256字符） | - |
| `AlarmTime` | `DateTime` | `alarm_time` | 告警时间 | - |
| `ProcessingStatus` | `VoiceprintAlarmProcessingStatusEnum` | `processing_status` | 处理状态 | - |
| `CreatedAt` | `DateTime` | `created_at` | 创建时间 | - |
| `Remark` | `string?` | `remark` | 备注（最大512字符） | `null` |
| `UpdatedAt` | `DateTime?` | `updated_at` | 最近更新时间 | `null` |

## 方法列表

| 方法名 | 参数 | 返回值 | 说明 |
|--------|------|--------|------|
| `VoiceprintAlarmRecordEntity` | - | - | 无参构造函数 |
| `VoiceprintAlarmRecordEntity` | `Guid id` | - | 带ID构造函数 |

## 数据注释

```csharp
[SugarTable("vp_voiceprint_alarm")]
[SugarIndex($"IX_{nameof(MonitoredObjectId)}", nameof(MonitoredObjectId), OrderByType.Asc)]
public class VoiceprintAlarmRecordEntity : AggregateRoot<Guid>
```

### 主键配置

```csharp
[SugarColumn(IsPrimaryKey = true)]
public override Guid Id { get; protected set; }
```

### 索引

- 主键索引：`Id`
- 监测对象索引：`IX_MonitoredObjectId` (升序)

## 枚举类型

### VoiceprintAlarmProcessingStatusEnum - 告警处理状态

| 值 | 名称 | 描述 |
|----|------|------|
| `0` | `Pending` | 未处理 |
| `1` | `Processing` | 处理中 |
| `2` | `Completed` | 已处理 |
| `3` | `Ignored` | 已忽略 |

### AlarmLevelEnum - 告警级别（系统通用）

| 值 | 名称 | 描述 |
|----|------|------|
| `0` | `Normal` | 正常 |
| `1` | `Warning` | 预警 |
| `2` | `Alarm` | 告警 |

### VoiceprintAnomalyEnum - 声纹异常类型

| 值 | 名称 | 描述 |
|----|------|------|
| `0` | `Normal` | 正常 |
| `1` | `ClampBoltLoose` | 夹件螺栓松动 |
| `2` | `FanBearingFriction` | 风扇轴承摩擦 |
| `3` | `Excitation` | 励磁 |
| `4` | `ConnectionBoltLoose` | 连接螺栓松动 |
| `5` | `WeldingSpatter` | 焊渣异物 |
| `6` | `HalfCoreLooseExcitation` | 铁芯松动50%励磁 |
| `7` | `CoilLooseExcitation` | 线圈完全松动励磁 |

## 关系图

```
VoiceprintAlarmRecordEntity (声纹告警)
├── VoiceprintDeviceAudioRecordEntity (音频记录) [ManyToOne]
└── MonitoredObjectAggregateRoot (监测对象) [ManyToOne]

创建流程:
VoiceprintDeviceAudioRecordEntity
  ↓ (识别异常)
TryCreateAlarmAsync
  ↓
VoiceprintAlarmRecordEntity
```

## 告警创建流程

```csharp
// 声纹识别异常时自动创建告警
private async Task TryCreateAlarmAsync(VoiceprintDeviceAudioRecordEntity record)
{
    if (record.IsNormal) return;
    
    var alarm = new VoiceprintAlarmRecordEntity(GuidGenerator.Create())
    {
        MonitoredObjectId = record.MonitoredObjectId,
        DeviceAudioRecordId = record.Id,
        AlarmMessage = record.AnomalyType ?? "声纹识别异常",
        AlarmTime = record.CollectedAt,
        ProcessingStatus = VoiceprintAlarmProcessingStatusEnum.Pending,
        CreatedAt = Clock.Now,
        UpdatedAt = Clock.Now
    };
    
    await _alarmRecordRepository.InsertAsync(alarm);
}
```

## 使用示例

### 创建告警记录

```csharp
var alarm = new VoiceprintAlarmRecordEntity(GuidGenerator.Create())
{
    MonitoredObjectId = monitoredObjectId,
    DeviceAudioRecordId = audioRecordId,
    AlarmMessage = "夹件螺栓松动",
    AlarmTime = DateTime.Now,
    ProcessingStatus = VoiceprintAlarmProcessingStatusEnum.Pending,
    CreatedAt = DateTime.Now,
    Remark = "需现场检查确认"
};
```

### 更新处理状态

```csharp
alarm.ProcessingStatus = VoiceprintAlarmProcessingStatusEnum.Completed;
alarm.UpdatedAt = DateTime.Now;
alarm.Remark = "已现场维修，螺栓已紧固";
```

### 查询待处理告警

```csharp
var pendingAlarms = await _alarmRecordRepository._DbQueryable
    .Where(x => x.ProcessingStatus == VoiceprintAlarmProcessingStatusEnum.Pending)
    .OrderBy(x => x.AlarmTime)
    .ToListAsync();
```

## 相关实体

### VoiceprintDeviceAudioRecordEntity - 音频记录

| 属性名 | 说明 |
|--------|------|
| `AnomalyType` | 识别出的异常类型编码 |
| `RealAnomalyType` | 校正后的异常类型 |
| `IsNormal` | 是否判定为正常 |
| `Score` | 识别得分（0~1） |

## 相关服务

- `VoiceprintAudioAppService` - 声纹音频服务（包含告警创建逻辑）
- `AlarmInfoDto` - 告警信息DTO
- `AlarmTrendDto` - 告警趋势统计

## 告警处理策略

1. **自动创建**: 声纹识别异常时自动创建告警记录
2. **状态跟踪**: 通过 `ProcessingStatus` 跟踪处理进度
3. **关联查询**: 通过 `DeviceAudioRecordId` 关联原始音频数据
4. **监测对象**: 通过 `MonitoredObjectId` 关联设备位置

## 注意事项

1. **主键保护**: `Id` 使用 `protected set`，通过构造函数设置
2. **索引优化**: 在 `MonitoredObjectId` 上建立索引，支持按监测对象查询
3. **聚合根**: 继承 `AggregateRoot<Guid>`，支持领域事件
4. **状态机**: 处理状态遵循 `Pending → Processing → Completed/Ignored` 流程
5. **时间精度**: `AlarmTime` 使用音频采集时间，而非告警创建时间

---

**最后更新**: 2026-06-03
**模块**: `ast-voiceprint`
