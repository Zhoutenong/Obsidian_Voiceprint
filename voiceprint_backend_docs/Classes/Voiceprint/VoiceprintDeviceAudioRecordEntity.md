# VoiceprintDeviceAudioRecordEntity

设备声纹音频存储记录实体，用于存储从边缘设备采集的音频记录及其分析结果。

## 表信息

- **表名**: `vp_device_audio_record`
- **主键**: `Id` (Guid)
- **索引**: `IX_GroupId_MonitoredObjectId` (GroupId, MonitoredObjectId)

## 字段列表

| 字段名 | 类型 | 说明 | 外键 |
|--------|------|------|------|
| `Id` | Guid | 主键 | - |
| `MonitoredObjectId` | Guid | 关联监测对象 | MonitoredObject |
| `GroupId` | Guid | 对应声纹采集任务批次 | VoiceprintCaptureBatch |
| `DeviceId` | string(64) | 树莓派设备编号 | - |
| `CollectedAt` | DateTime | 树莓派采集完成时间 | - |
| `DurationSeconds` | string(64) | 录制时长文本（可包含小数秒） | - |
| `AnomalyType` | string(64) | 识别出的异常类型编码 | - |
| `RealAnomalyType` | string(64) | 校正后的异常类型编码 | - |
| `IsNormal` | bool | 是否判定为正常 | - |
| `Score` | double? | 识别得分（0~1） | - |
| `ProcessedFilePath` | string(512) | 识别后的音频或特征文件路径 | - |
| `CreatedAt` | DateTime | 创建时间 | - |
| `UpdatedAt` | DateTime? | 最近更新时间 | - |

## 关联实体

### 出站关系 (Outbound)

- **MonitoredObject** (via `MonitoredObjectId`) — 关联的监测对象
- **VoiceprintCaptureBatch** (via `GroupId`) — 声纹采集批次

### 入站关系 (Inbound)

- **VoiceprintAlarmRecordEntity** (via `DeviceAudioRecordId`) — 声纹告警记录引用此音频记录

## 服务读写

### 读取服务

- **VoiceprintDeviceAudioRecordService** — 查询音频记录、分页列表

### 写入服务

- **VoiceprintCaptureJob** — Hangfire 后台任务，定期采集音频并写入记录
- **VoiceprintDeviceAudioRecordService** — 更新异常类型、处理状态
- **VoiceprintProcessedCleanupJob** — 清理已处理的音频文件

## 业务规则

1. **采集批次**: 同一批次的音频记录共享相同的 `GroupId`
2. **异常类型**: `AnomalyType` 为 AI 识别结果，`RealAnomalyType` 为人工校正结果
3. **正常判定**: `IsNormal = true` 表示未检测到异常
4. **文件清理**: 已处理的记录会在清理任务中归档或删除

## 源码位置

`module/ast-voiceprint/Ast.Voiceprint.Domain/Entities/VoiceprintDeviceAudioRecordEntity.cs`
