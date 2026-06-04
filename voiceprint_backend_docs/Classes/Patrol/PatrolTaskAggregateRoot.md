# PatrolTaskAggregateRoot — 巡检任务聚合根

## 基本信息

- **实体名称**：`PatrolTaskAggregateRoot`
- **数据库表**：`ast_patrol_task`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Patrol/`
- **继承关系**：`AggregateRoot<Guid>` → `IAuditedObject` → `ISoftDelete`

## 实体说明

巡检任务聚合根定义了自动巡检的计划和调度配置。支持基于Cron表达式的灵活调度，以及基于时间间隔的循环执行。每个巡检任务关联一个变电站，可配置巡检类型、执行方式、启用状态等。

## 字段说明

### 主键与状态

| 字段名 | 数据类型 | 说明 | 约束 |
|-------|---------|------|------|
| `Id` | `Guid` | 主键 | Primary Key |
| `IsDeleted` | `bool` | 逻辑删除标记 | ISoftDelete |
| `IsEnabled` | `bool` | 任务是否启用 | 默认true |

### 任务信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `Name` | `string` | 巡检任务名称 | 如："每日巡检"、"周巡检" |
| `SubstationId` | `Guid` | 所属变电站ID | Foreign Key |
| `PatrolType` | `PatrolTypeEnum` | 巡检类型 | 见下方枚举 |
| `ExecutionType` | `ExecutionTypeEnum` | 执行类型 | 见下方枚举 |
| `Description` | `string?` | 任务描述 | - |

### Cron调度配置

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `CronExpression` | `string?` | Cron表达式 | 如："0 9 * * *"（每天9点） |

### 时间调度配置

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `StartDate` | `DateTime?` | 开始执行日期 | - |
| `IsLoopExecution` | `bool` | 是否循环执行 | - |
| `DaysOfWeek` | `string?` | 循环周期 | 如："1,3,5"（周一三五） |
| `StartTime` | `TimeSpan` | 开始执行时间 | - |
| `IsRepeat` | `bool` | 是否重复 | - |
| `RepeatInterval` | `int?` | 执行间隔 | 数值 |
| `IntervalUnit` | `string?` | 间隔单位 | 如："Minute"、"Hour" |
| `EndRepeatTime` | `TimeSpan?` | 结束重复时间 | - |

### 系统字段

| 字段名 | 数据类型 | 说明 | 来源 |
|-------|---------|------|------|
| `NextExecutionTime` | `DateTime?` | 下一次执行时间 | 系统计算 |
| `CreationTime` | `DateTime` | 创建时间 | IAuditedObject |
| `CreatorId` | `Guid?` | 创建者ID | IAuditedObject |
| `LastModificationTime` | `DateTime?` | 最后修改时间 | IAuditedObject |
| `LastModifierId` | `Guid?` | 最后修改者ID | IAuditedObject |

## 枚举类型

### PatrolTypeEnum — 巡检类型
```csharp
public enum PatrolTypeEnum
{
    Video = 0,       // 视频巡检
    Robot = 1,       // 机器人巡检
    Manual = 2,      // 人工巡检
    Drone = 3        // 无人机巡检
}
```

### ExecutionTypeEnum — 执行类型
```csharp
public enum ExecutionTypeEnum
{
    Cron = 0,         // Cron表达式调度
    TimeInterval = 1 // 时间间隔调度
}
```

## 导航属性

```csharp
[Navigate(NavigateType.OneToOne, nameof(SubstationId))]
public SubstationAggregateRoot Substation { get; set; }
```

## 业务规则

### 任务调度规则
1. **启用控制**：只有 IsEnabled=true 的任务才会被调度
2. **执行类型**：Cron调度或时间间隔调度二选一
3. **逻辑删除**：删除任务仅标记 IsDeleted，不物理删除
4. **下次执行**：系统自动计算并更新 NextExecutionTime

### Cron表达式示例
| Cron表达式 | 说明 |
|-----------|------|
| `0 9 * * *` | 每天9点执行 |
| `0 9 * * 1` | 每周一9点执行 |
| `0 */6 * * *` | 每6小时执行一次 |
| `0 0 9 * * 1-5` | 周一到周五早上9点执行 |

### 时间间隔示例
| 配置 | 说明 |
|------|------|
| IsRepeat=true, RepeatInterval=30, IntervalUnit="Minute" | 每30分钟执行一次 |
| IsRepeat=true, RepeatInterval=2, IntervalUnit="Hour" | 每2小时执行一次 |
| StartTime=09:00, EndRepeatTime=17:00 | 9点到17点之间按间隔执行 |

## 相关实体

- [[SubstationAggregateRoot]] — 变电站聚合根
- [[PatrolRecordEntity]] — 巡检记录（一对多）
- [[PatrolTaskMonitoredPointRelEntity]] — 任务点位关联

## 相关服务

- [[PatrolTaskService]] — 巡检任务服务
- [[PatrolExecutionService]] — 巡检执行服务
- [[PatrolJobManager]] — 巡检任务调度器（Hangfire）

## 数据查询示例

### 查询所有启用的巡检任务
```sql
SELECT * FROM ast_patrol_task
WHERE is_enabled = 1
AND is_deleted = 0
ORDER BY next_execution_time;
```

### 查询某变电站的所有巡检任务
```sql
SELECT
    pt.id,
    pt.name,
    pt.patrol_type,
    pt.execution_type,
    pt.cron_expression,
    pt.is_enabled,
    pt.next_execution_time
FROM ast_patrol_task pt
WHERE pt.substation_id = 'xxx'
AND pt.is_deleted = 0
ORDER BY pt.creation_time DESC;
```

### 查询即将执行的任务（未来1小时内）
```sql
SELECT * FROM ast_patrol_task
WHERE is_enabled = 1
AND is_deleted = 0
AND next_execution_time >= GETDATE()
AND next_execution_time <= DATEADD(HOUR, 1, GETDATE())
ORDER BY next_execution_time;
```

### 统计巡检任务执行情况
```sql
SELECT
    pt.name,
    COUNT(pr.id) as execution_count,
    SUM(CASE WHEN pr.status = 1 THEN 1 ELSE 0 END) as success_count,
    SUM(CASE WHEN pr.status = 0 THEN 1 ELSE 0 END) as failed_count
FROM ast_patrol_task pt
LEFT JOIN ast_patrol_record pr ON pt.id = pr.patrol_task_id
WHERE pt.is_deleted = 0
GROUP BY pt.id, pt.name
ORDER BY execution_count DESC;
```

## 索引建议

```sql
-- 变电站ID索引
CREATE INDEX IX_SubstationId ON ast_patrol_task(substation_id);

-- 巡检类型索引
CREATE INDEX IX_PatrolType ON ast_patrol_task(patrol_type);

-- 执行类型索引
CREATE INDEX IX_ExecutionType ON ast_patrol_task(execution_type);

-- 启用状态索引
CREATE INDEX IX_IsEnabled ON ast_patrol_task(is_enabled);

-- 下次执行时间索引
CREATE INDEX IX_NextExecutionTime ON ast_patrol_task(next_execution_time);

-- 删除标记索引
CREATE INDEX IX_IsDeleted ON ast_patrol_task(is_deleted);
```

## 业务价值

**核心作用**：
1. **自动化巡检**：支持配置自动巡检任务，减少人工操作
2. **灵活调度**：支持Cron表达式和时间间隔两种调度方式
3. **状态管理**：支持启用/禁用和逻辑删除
4. **执行追踪**：通过NextExecutionTime追踪下次执行时间

## 使用场景

### 场景1：每日定时巡检
```
任务名称：每日巡检
巡检类型：Video
执行类型：Cron
Cron表达式：0 9 * * * （每天9点）
启用状态：true
```

### 场景2：工作日定时巡检
```
任务名称：工作日巡检
巡检类型：Video
执行类型：Cron
Cron表达式：0 9 * * 1-5 （周一到周五9点）
启用状态：true
```

### 场景3：周期性巡检
```
任务名称：定期巡检
巡检类型：Robot
执行类型：TimeInterval
StartTime：09:00
IsRepeat：true
RepeatInterval：2
IntervalUnit：Hour
启用状态：true
```

## Hangfire集成

### 任务调度
```csharp
// 在Hangfire中注册巡检任务
public class PatrolJobManager
{
    public async Task RegisterPatrolTasksAsync()
    {
        var tasks = await _repository.GetListAsync(t => 
            t.IsEnabled && !t.IsDeleted
        );

        foreach (var task in tasks)
        {
            if (task.ExecutionType == ExecutionTypeEnum.Cron)
            {
                // 使用Cron表达式
                RecurringJob.AddOrUpdate(
                    $"patrol_{task.Id}",
                    () => _patrolExecutionService.ExecuteAsync(task.Id),
                    task.CronExpression,
                    TimeZoneInfo.Local
                );
            }
            else if (task.ExecutionType == ExecutionTypeEnum.TimeInterval)
            {
                // 使用时间间隔
                RecurringJob.AddOrUpdate(
                    $"patrol_{task.Id}",
                    () => _patrolExecutionService.ExecuteAsync(task.Id),
                    () => Cron.Minutely(task.RepeatInterval.Value),
                    TimeZoneInfo.Local
                );
            }
        }
    }
}
```

## 设计模式

### 聚合根模式
- `PatrolTaskAggregateRoot` 是聚合根
- 巡检记录和点位关联是其聚合的一部分
- 确保巡检任务相关的数据一致性

### 软删除模式
```csharp
// 删除任务时不物理删除
public async Task DeleteAsync(Guid taskId)
{
    var task = await _repository.GetAsync(taskId);
    task.IsDeleted = true;
    task.IsEnabled = false;
    await _repository.UpdateAsync(task);
    
    // 从Hangfire移除
    RecurringJob.RemoveIfExists($"patrol_{taskId}");
}
```

### 策略模式
- 根据执行类型选择不同的调度策略
- Cron调度：灵活支持复杂时间规则
- 时间间隔调度：简单的周期性执行

---

> **最后更新**：2026-06-04
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Patrol/PatrolTaskAggregateRoot.cs`