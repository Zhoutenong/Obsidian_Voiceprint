# Voiceprint - 声纹音频记录实体

> 设备声纹音频存储记录表
>
> **位置**: `module/ast-voiceprint/Ast.Voiceprint.Domain/Entities/VoiceprintDeviceAudioRecordEntity.cs`
>
> **表名**: `vp_device_audio_record`

## 概述

`VoiceprintDeviceAudioRecordEntity` 表示从边缘采集设备收集的声纹音频记录实体，包含音频文件路径、识别结果、异常类型等关键信息。

## 属性列表

| 属性名 | 类型 | 数据库列名 | 说明 | 默认值 |
|--------|------|-----------|------|--------|
| `Id` | `Guid` | `id` | 主键 | - |
| `MonitoredObjectId` | `Guid` | `monitored_object_id` | 关联监测对象ID | - |
| `GroupId` | `Guid` | `group_id` | 对应声纹采集任务批次 | - |
| `DeviceId` | `string?` | `device_id` | 树莓派设备编号（最大64字符） | `null` |
| `CollectedAt` | `DateTime` | `collected_at` | 树莓派采集完成时间 | - |
| `DurationSeconds` | `string` | `duration_seconds` | 录制时长文本（可包含小数秒） | - |
| `AnomalyType` | `string?` | `anomaly_type` | 识别出的异常类型编码（最大64字符） | `null` |
| `RealAnomalyType` | `string?` | `real_anomaly_type` | 校正后的异常类型（最大64字符） | `null` |
| `IsNormal` | `bool` | `is_normal` | 是否判定为正常 | - |
| `Score` | `double?` | `score` | 识别得分（0~1） | `null` |
| `ProcessedFilePath` | `string?` | `processed_file_path` | 识别后的音频或特征文件路径（最大512字符） | `null` |
| `CreatedAt` | `DateTime` | `created_at` | 创建时间 | - |
| `UpdatedAt` | `DateTime?` | `updated_at` | 最近更新时间 | `null` |

## 方法列表

| 方法名 | 参数 | 返回值 | 说明 |
|--------|------|--------|------|
| `VoiceprintDeviceAudioRecordEntity` | - | - | 无参构造函数 |
| `VoiceprintDeviceAudioRecordEntity` | `Guid id` | - | 带ID构造函数 |

## 数据注释

```csharp
[SugarTable("vp_device_audio_record")]
[SugarIndex($"IX_{nameof(GroupId)}_{nameof(MonitoredObjectId)}", nameof(GroupId), OrderByType.Asc, nameof(MonitoredObjectId), OrderByType.Asc)]
public class VoiceprintDeviceAudioRecordEntity : AggregateRoot<Guid>
```

### 主键配置

```csharp
[SugarColumn(IsPrimaryKey = true)]
public override Guid Id { get; protected set; }
```

### 索引

- 主键索引：`Id`
- 组合索引：`IX_GroupId_MonitoredObjectId` (GroupId升序, MonitoredObjectId升序)

## 枚举类型

### VoiceprintAnomalyEnum - 声纹异常类型

| 值 | 名称 | 描述 | 编码值 |
|----|------|------|--------|
| `0` | `Normal` | 正常 | `"Normal"` |
| `1` | `ClampBoltLoose` | 夹件螺栓松动 | `"ClampBoltLoose"` |
| `2` | `FanBearingFriction` | 风扇轴承摩擦 | `"FanBearingFriction"` |
| `3` | `Excitation` | 励磁 | `"Excitation"` |
| `4` | `ConnectionBoltLoose` | 连接螺栓松动 | `"ConnectionBoltLoose"` |
| `5` | `WeldingSpatter` | 焊渣异物 | `"WeldingSpatter"` |
| `6` | `HalfCoreLooseExcitation` | 铁芯松动50%励磁 | `"HalfCoreLooseExcitation"` |
| `7` | `CoilLooseExcitation` | 线圈完全松动励磁 | `"CoilLooseExcitation"` |

## 关系图

```
VoiceprintDeviceAudioRecordEntity (声纹音频记录)
├── MonitoredObjectAggregateRoot (监测对象) [ManyToOne]
├── VoiceprintAlarmRecordEntity (告警记录) [OneToMany]
└── 关联到
    └── VoiceprintCaptureJob (采集任务)

数据流向:
边缘设备 → 音频采集 → 声纹识别 → 记录存储 → 告警触发
```

## 声纹识别流程

```csharp
// 1. 音频采集
var audioRecord = new VoiceprintDeviceAudioRecordEntity(GuidGenerator.Create())
{
    MonitoredObjectId = monitoredObjectId,
    GroupId = batchId,
    DeviceId = "raspberrypi-001",
    CollectedAt = DateTime.Now,
    DurationSeconds = "60.5",
    CreatedAt = DateTime.Now
};

// 2. 声纹识别
audioRecord.AnomalyType = "ClampBoltLoose";
audioRecord.Score = 0.89;
audioRecord.IsNormal = false;
audioRecord.ProcessedFilePath = "/data/processed/audio_001.wav";

// 3. 更新记录
audioRecord.UpdatedAt = DateTime.Now;

// 4. 触发告警（如果异常）
await TryCreateAlarmAsync(audioRecord);
```

## 使用示例

### 创建音频记录

```csharp
var record = new VoiceprintDeviceAudioRecordEntity(GuidGenerator.Create())
{
    MonitoredObjectId = monitoredObjectId,
    GroupId = Guid.Parse("..."),
    DeviceId = "edge-device-001",
    CollectedAt = DateTime.UtcNow,
    DurationSeconds = "60.0",
    CreatedAt = DateTime.Now
};
```

### 标记识别结果

```csharp
record.AnomalyType = "FanBearingFriction";  // 风扇轴承摩擦
record.RealAnomalyType = null;  // 人工校正前为空
record.IsNormal = false;  // 识别为异常
record.Score = 0.92;  // 置信度92%
record.ProcessedFilePath = "/storage/processed/abc123.wav";
record.UpdatedAt = DateTime.Now;
```

### 人工校正异常类型

```csharp
record.RealAnomalyType = "WeldingSpatter";  // 专家校正为焊渣异物
record.UpdatedAt = DateTime.Now;
```

### 查询异常记录

```csharp
var abnormalRecords = await _repository._DbQueryable
    .Where(x => !x.IsNormal)
    .Where(x => x.MonitoredObjectId == monitoredObjectId)
    .OrderByDescending(x => x.CollectedAt)
    .ToListAsync();
```

## 批次管理

### GroupId 作用

`GroupId` 标识同一次采集批次的音频记录，用于：

1. **批次追溯**: 查询某次采集任务的所有音频
2. **统计分析**: 按批次统计识别成功率
3. **数据清理**: 定期清理过期的批次数据

```csharp
// 查询某批次的所有记录
var batchRecords = await _repository._DbQueryable
    .Where(x => x.GroupId == batchId)
    .ToListAsync();
```

## 告警关联

### 自动告警条件

```csharp
// 当满足以下条件时自动创建告警：
if (!audioRecord.IsNormal && audioRecord.Score > 0.8)
{
    // 创建 VoiceprintAlarmRecordEntity
}
```

## 识别得分说明

| 得分范围 | 说明 | 处理建议 |
|---------|------|----------|
| `0.9 ~ 1.0` | 高置信度 | 直接触发告警 |
| `0.7 ~ 0.9` | 中等置信度 | 触发告警，建议人工复核 |
| `0.5 ~ 0.7` | 低置信度 | 标记可疑，不直接告警 |
| `< 0.5` | 极低置信度 | 忽略，记录为正常 |

## 文件路径管理

### ProcessedFilePath 格式

```
/storage/processed/{monitored_object_id}/{date}/{audio_id}.wav
```

示例：
```
/storage/processed/a1b2c3d4-.../2026-06-03/abc123-def456.wav
```

## 相关实体

### VoiceprintAlarmRecordEntity - 告警记录

通过 `DeviceAudioRecordId` 关联：

```csharp
// 查询某音频记录的告警
var alarms = await _alarmRepository._DbQueryable
    .Where(x => x.DeviceAudioRecordId == audioRecordId)
    .ToListAsync();
```

## 相关服务

- `VoiceprintAudioAppService` - 声纹音频上传服务
- `VoiceprintCaptureJob` - 声纹采集后台任务
- `VoiceprintProcessedCleanupJob` - 处理后文件清理任务

## 存储配置

```json
{
  "VoiceprintStorageOptions": {
    "BasePath": "/data/voiceprint",
    "ProcessedPath": "/data/voiceprint/processed",
    "RawPath": "/data/voiceprint/raw",
    "MaxRetentionDays": 90
  }
}
```

## 注意事项

1. **主键保护**: `Id` 使用 `protected set`，通过构造函数设置
2. **时长格式**: `DurationSeconds` 为字符串类型，支持小数秒（如 `"60.5"`）
3. **异常类型**: `AnomalyType` 和 `RealAnomalyType` 存储枚举名称，非数值
4. **批次索引**: 组合索引优化按监测对象查询批次的性能
5. **时间戳**: `CollectedAt` 使用采集时间，非服务器接收时间
6. **文件路径**: `ProcessedFilePath` 可能为空（识别失败时）

---

**最后更新**: 2026-06-03
**模块**: `ast-voiceprint`
