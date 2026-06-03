# PatrolTaskAggregateRoot

巡检任务计划实体，定义变电站设备巡检的调度计划。

## 表信息

- **表名**: `ast_patrol_task`
- **主键**: `Id` (Guid)
- **索引**: 
  - `IX_SubstationId` (SubstationId)
  - `IX_PatrolType` (PatrolType)
  - `IX_ExecutionType` (ExecutionType)
  - `IX_IsEnabled` (IsEnabled)
  - `IX_NextExecutionTime` (NextExecutionTime)
  - `IX_CreationTime` (CreationTime DESC)
  - `IX_IsDeleted` (IsDeleted)

## 字段列表

| 字段名 | 类型 | 说明 | 外键 |
|--------|------|------|------|
| `Id` | Guid | 主键 | - |
| `IsDeleted` | bool | 逻辑删除标记（默认 false） | - |
| `Name` | string | 巡检任务名称 | - |
| `SubstationId` | Guid | 所属变电站ID | Substation |
| `PatrolType` | enum | 巡检类型 | - |
| `ExecutionType` | enum | 执行类型 | - |
| `CronExpression` | string? | Hangfire Cron 表达式 | - |
| `IsEnabled` | bool | 任务是否启用（默认 true） | - |
| `Description` | string? | 任务描述 | - |
| `StartDate` | DateTime? | 开始执行日期 | - |
| `IsLoopExecution` | bool | 是否循环执行 | - |
| `DaysOfWeek` | string? | 循环周期（如 "1,3,5" 表示周一三五） | - |
| `StartTime` | TimeSpan | 开始执行时间 | - |
| `IsRepeat` | bool | 是否重复 | - |
| `RepeatInterval` | int? | 执行间隔 | - |
| `IntervalUnit` | string? | 间隔单位（"Minutes", "Hours"） | - |
| `EndRepeatTime` | TimeSpan? | 结束重复时间 | - |
| `NextExecutionTime` | DateTime? | 下一次执行时间 | - |
| `CreationTime` | DateTime | 创建时间 | - |
| `CreatorId` | Guid? | 创建人ID | - |
| `LastModificationTime` | DateTime? | 最后修改时间 | - |
| `LastModifierId` | Guid? | 最后修改人ID | - |

### 枚举类型

**PatrolType** (PatrolTypeEnum):
- `Routine` — 例行巡检
- `Special` — 特殊巡检
- `Emergency` — 应急巡检

**ExecutionType** (ExecutionTypeEnum):
- `Scheduled` — 定时执行
- `Manual` — 手动执行
- `Recurring` — 循环执行

## 关联实体

### 出站关系 (Outbound)

- **Substation** (via `SubstationId`) — 所属变电站（导航属性）
- **PatrolTaskMonitoredPointRelEntity** — 任务关联的监测点位（一对多）

### 入站关系 (Inbound)

- **PatrolRecordEntity** — 根据此任务生成的巡检记录（一对多）

## 服务读写

### 读取服务

- **PatrolTaskService** — 查询任务列表、获取任务详情、按变电站查询
- **PatrolTaskScheduler** — Hangfire 调度器读取待执行任务

### 写入服务

- **PatrolTaskService** — 创建、更新、删除任务（逻辑删除）
- **PatrolTaskScheduler** — 更新 `NextExecutionTime`
- **PatrolExecutionService** — 执行任务时生成 `PatrolRecord`

## 业务规则

1. **软删除**: 使用 `IsDeleted` 标记，不物理删除
2. **调度模式**:
   - 定时任务：使用 `CronExpression`
   - 循环任务：使用 `DaysOfWeek` + `StartTime`
   - 重复任务：使用 `RepeatInterval` + `IntervalUnit`
3. **任务启用**: 只有 `IsEnabled = true` 的任务会被调度
4. **下次执行时间**: 每次执行后更新 `NextExecutionTime`
5. **审计**: 实现了 `IAuditedObject` 接口，自动记录创建和修改信息

## 调度流程

```
1. PatrolTaskScheduler (Hangfire) 定时扫描
2. 查找 IsEnabled=true && NextExecutionTime <= Now
3. 创建 PatrolRecord
4. 更新 NextExecutionTime
5. 执行巡检流程
```

## 源码位置

`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Patrol/PatrolTaskAggregateRoot.cs`
