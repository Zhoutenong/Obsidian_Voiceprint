# PatrolRecordEntity

巡检记录实体，记录单次巡检任务的执行过程和结果。

## 表信息

- **表名**: `ast_patrol_record`
- **主键**: `Id` (Guid)
- **索引**:
  - `IX_PatrolTaskId` (PatrolTaskId)
  - `IX_ScheduledTime` (ScheduledTime DESC)
  - `IX_StartTime` (StartTime DESC)
  - `IX_Status` (Status)
  - `IX_CreationTime` (CreationTime DESC)

## 字段列表

| 字段名 | 类型 | 说明 | 外键 |
|--------|------|------|------|
| `Id` | Guid | 主键 | - |
| `PatrolTaskId` | Guid | 关联的巡检任务ID | PatrolTask |
| `PatrolTaskName` | string | 任务名称（冗余） | - |
| `ScheduledTime` | DateTime | 计划执行时间 | - |
| `StartTime` | DateTime? | 实际开始执行时间 | - |
| `EndTime` | DateTime? | 执行完成时间 | - |
| `CurrentPatrolTaskMonitoredPointRelId` | Guid | 当前正在识别的点位ID | PatrolTaskMonitoredPointRel |
| `Status` | enum | 任务状态 | - |
| `IsManualStopped` | bool | 是否被人工中止（默认 true） | - |
| `OverallResult` | enum? | 总体结果 | - |
| `IsStopped` | bool | 是否中止 | - |
| `CancelReason` | string? | 取消/中止说明 | - |
| `TotalPointCount` | int | 巡视点位总数 | - |
| `CompletedPointCount` | int | 已巡视点位数 | - |
| `CreationTime` | DateTime | 创建时间 | - |

### 枚举类型

**Status** (PatrolStatusEnum):
- `Pending` — 待执行
- `Running` — 执行中
- `Completed` — 已完成
- `Failed` — 失败
- `Cancelled` — 已取消

**OverallResult** (PatrolResultEnum):
- `Normal` — 正常
- `Abnormal` — 异常
- `PartiallyAbnormal` — 部分异常

## 关联实体

### 出站关系 (Outbound)

- **PatrolTask** (via `PatrolTaskId`) — 关联的巡检任务
- **PatrolTaskMonitoredPointRel** (via `CurrentPatrolTaskMonitoredPointRelId`) — 当前执行的点位
- **PatrolRecordItemEntity** — 巡检记录项（一对多，每个点位的检查结果）

### 入站关系 (Inbound)

- **报表服务** — 统计巡检完成率、异常率
- **历史查询服务** — 查询巡检历史记录

## 服务读写

### 读取服务

- **PatrolRecordService** — 查询巡检记录、分页列表、统计查询
- **实时监控服务** — 获取正在执行的巡检状态
- **SignalR Hub** — 实时推送巡检进度

### 写入服务

- **PatrolExecutionService** — 创建记录、更新状态、更新进度
- **PatrolTaskScheduler** — 定时创建新的巡检记录
- **PatrolControlService** — 中止/取消巡检

## 业务规则

1. **生命周期**: `Pending` → `Running` → `Completed`/`Failed`/`Cancelled`
2. **进度跟踪**: `CompletedPointCount / TotalPointCount` 反映执行进度
3. **当前点位**: `CurrentPatrolTaskMonitoredPointRelId` 指向正在检查的点位
4. **人工中止**: `IsManualStopped` 标记是否由用户手动中止
5. **总体结果**: 根据所有点位的检查结果汇总得出
6. **时间记录**: `StartTime` 和 `EndTime` 记录实际执行时长

## 执行流程

```
1. PatrolTaskScheduler 创建记录 (Status=Pending, ScheduledTime)
2. PatrolExecutionService 启动巡检 (Status=Running, StartTime)
3. 逐个执行点位，更新 CurrentPatrolTaskMonitoredPointRelId
4. 每完成一个点位，CompletedPointCount++
5. 所有点位完成或中途中止 (Status=Completed/Cancelled, EndTime)
6. 汇总 OverallResult
```

## 源码位置

`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Patrol/PatrolRecordEntity.cs`
