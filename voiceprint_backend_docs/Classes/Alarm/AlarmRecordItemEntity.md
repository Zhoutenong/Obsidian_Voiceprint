# AlarmRecordItemEntity

## 概述

`AlarmRecordItemEntity` 代表告警记录中的具体告警项。每条告警记录（`AlarmRecordAggregateRoot`）可以包含多个告警项，每个告警项对应一个点位的数据异常。

## 表信息

- **表名**: `ast_alarm_record_item`
- **主键**: `Id` (Guid)
- **继承**: `Entity<Guid>`
- **命名空间**: `Ast.IntelliSub.Domain.Entities`

## 字段列表

| 字段名 | 类型 | 数据库列名 | 说明 | 约束 |
|--------|------|-----------|------|------|
| `Id` | `Guid` | `id` | 主键 | PK |
| `AlarmRecordId` | `Guid` | `alarm_record_id` | 告警记录ID | FK |
| `PointBindingRelId` | `Guid` | `point_binding_rel_id` | 点位绑定关系ID | FK |
| `AstPointId` | `Guid` | `ast_point_id` | 点位ID | FK |
| `BindingItemStrategyRelId` | `Guid` | `binding_item_strategy_rel_id` | 策略关系ID | FK |
| `Ts` | `DateTime` | `ts` | 数据采集时间戳 | |
| `GroupId` | `string?` | `group_id` | 数据组ID | 来自 pointdata |
| `AlarmContent` | `string` | `alarm_content` | 告警内容 | TEXT |
| `AlarmLevel` | `AlarmLevelEnum` | `alarm_level` | 告警级别 | Enum |
| `AlarmCategoryName` | `string?` | `alarm_category_name` | 告警类别名称 | |
| `AlarmTime` | `DateTime` | `alarm_time` | 告警时间 | |
| `Remarks` | `string?` | `remarks` | 备注 | |
| `CreationTime` | `DateTime` | `creation_time` | 创建时间 | |

## 关联实体 (ER 关系)

### 导航属性

```csharp
// 告警记录
[Navigate(NavigateType.OneToOne, nameof(AlarmRecordId))]
public AlarmRecordAggregateRoot AlarmRecord { get; set; }

// 点位绑定关系
[Navigate(NavigateType.OneToOne, nameof(PointBindingRelId))]
public PointBindingRelEntity PointBindingRel { get; set; }

// 点位
[Navigate(NavigateType.OneToOne, nameof(AstPointId))]
public AstPointEntity AstPoint { get; set; }
```

### 关系图

```
AlarmRecordAggregateRoot (1) ←──→ (N) AlarmRecordItemEntity                                              │
                                              ├── (N) PointBindingRelEntity
                                              ├── (N) AstPointEntity
                                              └── (N) BindingItemStrategyRelEntity
```

## 被哪些服务读写

### 写入服务

- `AlarmRecordService` — 告警记录项创建、更新
- `AlarmAnalysisService` — 告警分析和告警项生成

### 读取服务

- `AlarmRecordService` — 告警查询、告警详情
- `ReportService` — 告警统计报告
- `RealtimeMonitoringPointService` — 实时告警展示

## 业务规则约束

### 1. 枚举约束

#### AlarmLevelEnum

```csharp
public enum AlarmLevelEnum
{
    Info = 0,         // 信息
    Warning = 1,       // 警告
    Critical = 2,     // 严重
    Emergency = 3     // 紧急
}
```

### 2. 告警级别规则

- **Info (0)**: 一般信息，不需要处理
- **Warning (1)**: 需要关注，建议检查
- **Critical (2)**: 严重告警，需要立即处理
- **Emergency (3)**: 紧急告警，需要立即采取行动

### 3. 时间字段说明

- `Ts`: 数据采集时间戳（传感器数据产生时间）
- `AlarmTime`: 告警判断时间（系统判断为告警的时间）
- `CreationTime`: 告警记录创建时间（入库时间）

通常：`Ts` ≤ `AlarmTime` ≤ `CreationTime`

### 4. 告警内容格式

`AlarmContent` 存储详细的告警描述：

```
温度异常告警：当前值 85.5℃，超过上限阈值 80.0℃，
持续时长 5 分钟，点位：1号机柜温度传感器，
位置：朝阳变电站 > 1号机房 > 1号机柜
```

### 5. 删除约束

- 删除告警记录（`AlarmRecord`）时级联删除所有告警项
- 建议使用软删除，保留历史告警数据用于统计分析

## 使用场景

### 1. 创建告警项

```csharp
var alarmItem = new AlarmRecordItemEntity
{
    Id = Guid.NewGuid(),
    AlarmRecordId = alarmRecordId,
    PointBindingRelId = pointBindingRelId,
    AstPointId = astPointId,
    BindingItemStrategyRelId = strategyRelId,
    Ts = DateTime.Now.AddMinutes(-5),
    GroupId = "GROUP_20240604_120000",
    AlarmContent = "温度异常：85.5℃，超过上限80.0℃",
    AlarmLevel = AlarmLevelEnum.Critical,
    AlarmCategoryName = "温度告警",
    AlarmTime = DateTime.Now,
    Remarks = "持续5分钟",
    CreationTime = DateTime.Now
};

await _alarmItemRepo.InsertAsync(alarmItem);
```

### 2. 查询告警记录的所有告警项

```csharp
public async Task<List<AlarmRecordItemEntity>> GetItemsByRecordIdAsync(Guid recordId)
{
    return await _alarmItemRepo.GetListAsync(
        item => item.AlarmRecordId == recordId,
        orderBy: item => item.AlarmTime,
        orderByOrderByType: OrderByType.Desc
    );
}
```

### 3. 查询严重告警

```csharp
public async Task<List<AlarmRecordItemEntity>> GetCriticalAlarmsAsync(DateTime startTime)
{
    return await _alarmItemRepo.GetListAsync(
        item => item.AlarmLevel >= AlarmLevelEnum.Critical
                 && item.AlarmTime >= startTime,
        orderBy: item => item.AlarmTime,
        orderByOrderByType: OrderByType.Desc
    );
}
```

### 4. 按点位统计告警

```csharp
public async Task<List<PointAlarmStatisticsDto>> GetStatisticsByPointAsync(DateTime startTime)
{
    var items = await _alarmItemRepo.GetListAsync(
        item => item.AlarmTime >= startTime
    );

    return items.GroupBy(item => item.AstPointId)
               .Select(g => new PointAlarmStatisticsDto
               {
                   PointId = g.Key,
                   AlarmCount = g.Count(),
                   CriticalCount = g.Count(i => i.AlarmLevel == AlarmLevelEnum.Critical),
                   LastAlarmTime = g.Max(i => i.AlarmTime)
               })
               .ToList();
}
```

## 索引建议

```sql
-- 告警记录索引（用于查询某条记录的所有告警项）
CREATE INDEX IX_ALARM_ITEM_RECORD_ID ON ast_alarm_record_item(alarm_record_id);

-- 点位索引（用于查询某点位的历史告警）
CREATE INDEX IX_ALARM_ITEM_POINT_ID ON ast_alarm_record_item(ast_point_id);

-- 告警时间索引（用于时间范围查询）
CREATE INDEX IX_ALARM_ITEM_TIME ON ast_alarm_record_item(alarm_time);

-- 告警级别索引（用于查询严重告警）
CREATE INDEX IX_ALARM_ITEM_LEVEL ON ast_alarm_record_item(alarm_level);

-- 复合索引（时间 + 级别，用于查询特定时间段的严重告警）
CREATE INDEX IX_ALARM_ITEM_TIME_LEVEL ON ast_alarm_record_item(alarm_time, alarm_level);
```

## 告警统计示例

### 按级别统计告警数量

```sql
SELECT
    alarm_level,
    COUNT(*) AS count
FROM ast_alarm_record_item
WHERE alarm_time >= @StartTime
GROUP BY alarm_level
ORDER BY alarm_level;
```

### 按类别统计告警趋势

```sql
SELECT
    DATE(alarm_time) AS alarm_date,
    alarm_category_name,
    COUNT(*) AS count
FROM ast_alarm_record_item
WHERE alarm_time >= DATEADD(day, -30, GETDATE())
GROUP BY DATE(alarm_time), alarm_category_name
ORDER BY alarm_date DESC, alarm_category_name;
```

## 相关文件

- **源码**: `module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Alarm/AlarmRecordItemEntity.cs`
- **服务**: `module/ast-intellisub/Ast.IntelliSub.Application/Services/AlarmRecordService.cs`
- **告警记录**: `Classes/Alarm/AlarmRecordAggregateRoot.md`
- **告警类别**: `Classes/Alarm/AlarmCategoryAggregateRoot.md`
- **点位绑定**: `Classes/DataBinding/PointBindingRelEntity.md`
