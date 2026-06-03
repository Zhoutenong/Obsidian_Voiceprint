# 巡检记录服务 (PatrolRecordService)

## 概述
巡检记录服务负责管理和查询巡检执行产生的记录数据。

## 职责
- 创建巡检记录
- 查询巡检记录
- 关联采集的图像和传感器数据
- 提供记录详情查询
- 统计巡检数据

## 主要接口

### 获取巡检记录列表
```csharp
Task<PagedResultDto<PatrolRecordDto>> GetListAsync(GetPatrolRecordListDto input)
```

**查询参数**：
- `TaskId`: 巡检任务ID
- `DeviceId`: 设备ID
- `StartTime`: 开始时间
- `EndTime`: 结束时间
- `Status`: 执行状态
- `PageIndex`: 页码
- `PageSize`: 每页大小

### 获取巡检记录详情
```csharp
Task<PatrolRecordDetailDto> GetDetailAsync(Guid recordId)
```

### 按分组ID获取记录
```csharp
Task<List<PatrolRecordDto>> GetByGroupIdAsync(Guid groupId)
```

### 获取巡检统计
```csharp
Task<PatrolStatisticsDto> GetStatisticsAsync(GetPatrolStatisticsDto input)
```

**统计维度**：
- 总巡检次数
- 成功次数
- 失败次数
- 异常点位统计
- 时间分布统计

## 数据结构

### 巡检记录 (PatrolRecord)

| 字段 | 类型 | 说明 |
|-----|------|------|
| Id | Guid | 记录ID |
| TaskId | Guid | 巡检任务ID |
| DeviceId | Guid | 设备ID |
| ExecutionTime | DateTime | 执行时间 |
| Status | ExecutionStatus | 执行状态 |
| ImagePath | string | 图像路径 |
| InfraredPath | string | 红外图像路径 |
| SensorData | string | 传感器数据（JSON） |
| Result | string | 执行结果 |
| ErrorMsg | string | 错误信息 |
| CreatedAt | DateTime | 创建时间 |

### 巡检记录详情 (PatrolRecordDetail)

包含基础记录信息外，还包含：
- 巡检任务信息
- 设备信息
- 点位列表
- 传感器数据详情
- 图像元数据
- 告警信息

## 查询场景

### 按设备查询
```csharp
var records = await _service.GetListAsync(new GetPatrolRecordListDto
{
    DeviceId = deviceId,
    StartTime = DateTime.Today.AddDays(-7),
    EndTime = DateTime.Today
});
```

### 按任务查询
```csharp
var records = await _service.GetListAsync(new GetPatrolRecordListDto
{
    TaskId = taskId
});
```

### 按日期范围查询
```csharp
var records = await _service.GetListAsync(new GetPatrolRecordListDto
{
    StartTime = startDate,
    EndTime = endDate
});
```

## 数据处理

### 记录创建
1. 接收巡检执行结果
2. 整理图像和传感器数据
3. 创建巡检记录
4. 关联点位数据
5. 保存到数据库

### 数据关联
- 关联巡检任务
- 关联设备信息
- 关联点位信息
- 关联图像文件
- 关联传感器数据
- 关联告警记录

### 数据清理
- 定期清理过期记录
- 清理关联的图像文件
- 清理临时数据

## 统计分析

### 执行统计
```csharp
var stats = await _service.GetStatisticsAsync(new GetPatrolStatisticsDto
{
    StartTime = DateTime.Today.AddMonths(-1),
    EndTime = DateTime.Today,
    GroupBy = "device"  // 按设备分组
});
```

**返回数据**：
- 各设备巡检次数
- 各设备成功率
- 异常点位分布
- 时间分布趋势

### 趋势分析
- 巡检频率趋势
- 异常发生率趋势
- 设备健康度趋势

## 数据导出

### 导出巡检报告
```csharp
Task<FileDto> ExportReportAsync(ExportPatrolReportDto input)
```

**导出格式**：
- Excel 格式
- PDF 格式
- 包含图像和图表

## 依赖服务

- [[PatrolTaskService]] - 巡检任务服务
- [[PatrolExecutionService]] - 巡检执行服务
- [[CameraService]] - 摄像机服务
- [[MonitoredPointService]] - 监测点位服务
- [[FileService]] - 文件服务

## 相关实体

- [[PatrolRecord]] - 巡检记录实体
- [[PatrolTask]] - 巡检任务
- [[Device]] - 设备
- [[PatrolPoint]] - 巡检点位
- [[PatrolPointRecord]] - 点位记录

## 性能优化

### 查询优化
- 添加索引：TaskId, DeviceId, ExecutionTime
- 分页查询
- 条件过滤

### 缓存策略
- 缓存统计数据（5分钟）
- 缓存热点数据
- 使用 Redis 分布式缓存

### 存储优化
- 图像文件独立存储
- 传感器数据压缩存储
- 历史数据归档

## 相关文档

- [[PatrolTaskService]] - 巡检任务服务文档
- [[PatrolExecutionService]] - 巡检执行服务文档
- [[PatrolJobManager]] - 巡检任务调度器文档

## API 路径

- `GET /api/app/patrol-records` - 获取巡检记录列表
- `GET /api/app/patrol-records/{id}` - 获取巡检记录详情
- `GET /api/app/patrol-records/by-group/{groupId}` - 按分组获取记录
- `GET /api/app/patrol-records/statistics` - 获取巡检统计
- `POST /api/app/patrol-records/export` - 导出巡检报告
