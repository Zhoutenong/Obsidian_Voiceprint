# 巡检任务服务 (PatrolTaskService)

## 概述
巡检任务服务负责管理智能巡检系统的巡检任务配置和调度。

## 职责
- 创建、更新、删除巡检任务
- 管理巡检任务状态（待执行、执行中、已完成、失败）
- 关联巡检路线和设备
- 配置巡检计划（定时任务）

## 主要接口

### 创建巡检任务
```csharp
Task<PatrolTaskDto> CreateAsync(CreatePatrolTaskDto input)
```

**请求参数**：
- `Name`: 任务名称
- `RouteId`: 巡检路线ID
- `DeviceIds`: 设备ID列表
- `ScheduleType`: 调度类型（定时/手动）
- `CronExpression`: Cron表达式（定时任务）
- `StartDate`: 开始日期
- `EndDate`: 结束日期

**响应**：
- 巡检任务详细信息

### 更新巡检任务
```csharp
Task<PatrolTaskDto> UpdateAsync(Guid id, UpdatePatrolTaskDto input)
```

### 删除巡检任务
```csharp
Task DeleteAsync(Guid id)
```

### 获取巡检任务列表
```csharp
Task<PagedResultDto<PatrolTaskDto>> GetListAsync(GetPatrolTaskListDto input)
```

**查询参数**：
- `Name`: 任务名称（模糊查询）
- `Status`: 任务状态
- `StartDate`: 开始日期
- `EndDate`: 结束日期
- `PageIndex`: 页码
- `PageSize`: 每页大小

### 获取巡检任务详情
```csharp
Task<PatrolTaskDto> GetDetailAsync(Guid id)
```

## 巡检任务状态

| 状态 | 说明 |
|-----|------|
| Pending | 待执行 |
| Running | 执行中 |
| Completed | 已完成 |
| Failed | 失败 |
| Cancelled | 已取消 |

## 调度类型

| 类型 | 说明 |
|-----|------|
| Manual | 手动触发 |
| Scheduled | 定时调度 |
| Recurring | 周期性任务 |

## 数据处理流程

1. **创建任务**
   - 验证输入参数
   - 检查巡检路线是否存在
   - 检查设备是否有效
   - 生成任务ID
   - 保存到数据库

2. **调度执行**
   - 根据调度类型配置
   - 注册到 Hangfire
   - 等待触发执行

3. **更新任务**
   - 验证任务状态
   - 更新任务信息
   - 重新调度（如需要）

4. **删除任务**
   - 检查任务状态
   - 取消待执行任务
   - 删除历史记录

## 异常处理

- ` BusinessException`: 巡检路线不存在
- ` BusinessException`: 设备不存在
- ` BusinessException`: 任务状态不允许修改
- ` BusinessException`: Cron 表达式无效

## 相关服务

- [[PatrolExecutionService]] - 巡检执行服务
- [[PatrolRecordService]] - 巡检记录服务
- [[PatrolJobManager]] - 巡检任务调度器
- [[PatrolSystemCleanupJob]] - 巡检系统清理任务

## 相关实体

- [[PatrolTask]] - 巡检任务实体
- [[PatrolRoute]] - 巡检路线
- [[Device]] - 设备
- [[PatrolRecord]] - 巡检记录

## 相关文档

- [[PatrolExecutionService]] - 巡检执行服务文档
- [[PatrolRecordService]] - 巡检记录服务文档
- [[PatrolJobManager]] - 巡检任务调度器文档

## API 路径

- `POST /api/app/patrol-tasks` - 创建巡检任务
- `PUT /api/app/patrol-tasks/{id}` - 更新巡检任务
- `DELETE /api/app/patrol-tasks/{id}` - 删除巡检任务
- `GET /api/app/patrol-tasks` - 获取巡检任务列表
- `GET /api/app/patrol-tasks/{id}` - 获取巡检任务详情
