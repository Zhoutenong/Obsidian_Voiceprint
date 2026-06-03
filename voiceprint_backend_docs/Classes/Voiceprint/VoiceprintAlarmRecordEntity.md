# VoiceprintAlarmRecordEntity

声纹告警记录实体，用于记录声纹分析检测到的异常告警信息。

## 表信息

- **表名**: `vp_voiceprint_alarm`
- **主键**: `Id` (Guid)
- **索引**: `IX_MonitoredObjectId` (MonitoredObjectId)

## 字段列表

| 字段名 | 类型 | 说明 | 外键 |
|--------|------|------|------|
| `Id` | Guid | 主键 | - |
| `MonitoredObjectId` | Guid | 关联监测对象 | MonitoredObject |
| `DeviceAudioRecordId` | Guid | 对应的音频记录 | VoiceprintDeviceAudioRecord |
| `AlarmMessage` | string(256) | 告警信息描述 | - |
| `AlarmTime` | DateTime | 告警时间 | - |
| `ProcessingStatus` | enum | 处理状态 | - |
| `Remark` | string(512) | 备注 | - |
| `CreatedAt` | DateTime | 创建时间 | - |
| `UpdatedAt` | DateTime? | 最近更新时间 | - |

### 枚举类型

**ProcessingStatus** (VoiceprintAlarmProcessingStatusEnum):
- `Pending` — 待处理
- `Processing` — 处理中
- `Resolved` — 已解决
- `Ignored` — 已忽略

## 关联实体

### 出站关系 (Outbound)

- **MonitoredObject** (via `MonitoredObjectId`) — 关联的监测对象
- **VoiceprintDeviceAudioRecord** (via `DeviceAudioRecordId`) — 触发告警的音频记录

### 入站关系 (Inbound)

- **告警处理服务** — 更新处理状态
- **报表生成服务** — 统计告警数据

## 服务读写

### 读取服务

- **VoiceprintAlarmRecordService** — 查询告警记录、分页列表、统计查询
- **RealTimeAlarmHub** (SignalR) — 实时推送告警到前端

### 写入服务

- **VoiceprintAnalysisService** — 检测到异常时创建告警记录
- **VoiceprintAlarmRecordService** — 更新处理状态、添加备注
- **告警后台任务** — 自动升级超时未处理的告警

## 业务规则

1. **告警触发**: 当声纹分析检测到异常（`IsNormal = false`）时自动创建
2. **处理流程**: `Pending` → `Processing` → `Resolved`/`Ignored`
3. **关联音频**: 每条告警必须关联一条音频记录用于回溯分析
4. **实时推送**: 新告警通过 SignalR 实时推送到告警中心
5. **告警升级**: 超时未处理的告警可自动升级（可配置）

## 源码位置

`module/ast-voiceprint/Ast.Voiceprint.Domain/Entities/VoiceprintAlarmRecordEntity.cs`
