# AlarmRecordAggregateRoot — 告警记录聚合根

## 基本信息

- **实体名称**：`AlarmRecordAggregateRoot`
- **数据库表**：`ast_alarm_record`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Alarm/`
- **继承关系**：`AggregateRoot<Guid>` → `IAuditedObject`

## 实体说明

告警记录聚合根是智能变电站监控系统的核心业务实体，用于记录所有设备告警信息。每个告警记录关联一个监测对象和一个监测点位，包含告警内容、级别、处理状态等关键信息。

## 字段说明

### 主键与关联

| 字段名 | 数据类型 | 说明 | 约束 |
|-------|---------|------|------|
| `Id` | `Guid` | 主键 | Primary Key |
| `MonitoredObjectId` | `Guid` | 被监测对象ID | Foreign Key |
| `MonitoredPointId` | `Guid` | 监测点位ID | Foreign Key |

### 告警信息

| 字段名 | 数据类型 | 说明 | 枚举值 |
|-------|---------|------|--------|
| `AlarmContent` | `text` | 告警内容描述 | - |
| `AlarmLevel` | `AlarmLevelEnum` | 告警级别 | 见下方枚举 |
| `ProcessingStatus` | `AlarmStatusEnum` | 处理状态 | 见下方枚举 |
| `AlarmTime` | `DateTime` | 告警发生时间 | - |

### 审计信息

| 字段名 | 数据类型 | 说明 | 来源 |
|-------|---------|------|------|
| `Remarks` | `string?` | 备注 | - |
| `CreationTime` | `DateTime` | 创建时间 | IAuditedObject |
| `CreatorId` | `Guid?` | 创建者ID | IAuditedObject |
| `LastModificationTime` | `DateTime?` | 最后修改时间 | IAuditedObject |
| `LastModifierId` | `Guid?` | 最后修改者ID | IAuditedObject |

## 枚举类型

### AlarmLevelEnum — 告警级别
```csharp
public enum AlarmLevelEnum
{
    Info = 0,        // 信息
    Warning = 1,     // 警告
    Critical = 2,    // 严重
    Emergency = 3    // 紧急
}
```

### AlarmStatusEnum — 处理状态
```csharp
public enum AlarmStatusEnum
{
    Pending = 0,     // 未处理
    Processing = 1,  // 处理中
    Resolved = 2,    // 已处理
    Ignored = 3      // 忽略
}
```

## 导航属性

```csharp
[Navigate(NavigateType.OneToOne, nameof(MonitoredObjectId))]
public MonitoredObjectAggregateRoot MonitoredObject { get; set; }

[Navigate(NavigateType.OneToOne, nameof(MonitoredPointId))]
public MonitoredPointEntity MonitoredPoint { get; set; }
```

## 业务规则

### 告警生命周期
1. **创建**：当监测点位值异常时，系统自动创建告警记录
2. **处理中**：运维人员接受告警，状态变为"处理中"
3. **已处理**：问题解决后，状态更新为"已处理"
4. **忽略**：对于误报或不重要的告警，可标记为"忽略"

### 告警级别矩阵
| 级别 | 描述 | 响应时间 | 通知方式 |
|-----|------|---------|---------|
| `Info` | 信息提示 | 24小时内 | 系统内通知 |
| `Warning` | 警告 | 8小时内 | 系统内通知 |
| `Critical` | 严重 | 2小时内 | 短信+系统 |
| `Emergency` | 紧急 | 立即响应 | 短信+电话+系统 |

## 数据查询示例

### 查询未处理的高级别告警
```sql
SELECT * FROM ast_alarm_record
WHERE processing_status = 0
AND alarm_level IN (2, 3)
ORDER BY alarm_time DESC
```

### 查询特定监测对象的告警历史
```sql
SELECT * FROM ast_alarm_record
WHERE monitored_object_id = 'xxx'
AND alarm_time >= '2026-01-01'
ORDER BY alarm_time DESC
```

## 相关实体

- [[ProcessingRecordEntity]] — 告警处理记录（一对多关系）
- [[AlarmCategoryAggregateRoot]] — 告警分类
- [[MonitoredObjectAggregateRoot]] — 监测对象
- [[MonitoredPointEntity]] — 监测点位

## 相关服务

- [[AlarmRecordService]] — 告警记录服务
- [[AlarmProcessingService]] — 告警处理服务
- [[AlarmNotificationService]] — 告警通知服务

## 相关流程

- [[设备告警流程]] — 端到端告警处理流程
- [[AlarmStrategyOverview]] — 告警策略总览

## 索引建议

```sql
-- 告警时间索引（用于时间范围查询）
CREATE INDEX idx_alarm_time ON ast_alarm_record(alarm_time);

-- 处理状态索引（用于筛选未处理告警）
CREATE INDEX idx_processing_status ON ast_alarm_record(processing_status);

-- 监测对象索引（用于查询特定对象告警）
CREATE INDEX idx_monitored_object ON ast_alarm_record(monitored_object_id);

-- 复合索引（状态+级别+时间，常用查询组合）
CREATE INDEX idx_status_level_time ON ast_alarm_record(processing_status, alarm_level, alarm_time);
```

## 业务价值

**核心作用**：
1. **故障追踪**：记录设备故障历史，为维护决策提供数据支持
2. **合规性**：满足电力行业对告警记录的合规要求
3. **统计分析**：支持告警趋势分析和设备健康度评估
4. **运维协作**：提供告警处理流程跟踪，支持多人协作

---

> **最后更新**：2026-06-04
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Alarm/AlarmRecordAggregateRoot.cs`