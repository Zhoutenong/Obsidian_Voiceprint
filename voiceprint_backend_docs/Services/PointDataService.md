# PointDataService — 点位数据API服务

## 基本信息

- **服务名称**：`PointDataService`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Application/Services/PointDataService.cs`
- **接口实现**：`IPointDataService`
- **认证要求**：`[Authorize]` - 需要认证
- **服务定位**：API层服务 - 对 `AstPointDataService` 的简单封装

## 服务概述

PointDataService 是 ast-intellisub 模块中的点位数据API服务，作为应用层的服务，它对底层的 [[AstPointDataService]] 进行简单封装，提供统一的API接口。该服务主要负责：
- 数据查询的日志记录和调试信息
- 告警状态过滤的复杂业务逻辑
- 统一的API端点暴露

## 与 AstPointDataService 的关系

```
┌─────────────────────────────────────────┐
│       PointDataService (API层)           │
│   module/ast-intellisub/Application      │
│                                         │
│  • 统一API接口                           │
│  • 日志记录                              │
│  • 告警状态过滤业务逻辑                  │
└──────────────┬──────────────────────────┘
               │ 封装调用
               ↓
┌─────────────────────────────────────────┐
│   AstPointDataService (数据层)           │
│  module/ast-intellisubdata/Application   │
│                                         │
│  • 数据持久化                            │
│  • TDengine/SQLite操作                  │
│  • 大文本存储                            │
│  • 时序数据聚合                          │
└─────────────────────────────────────────┘
```

## 依赖注入

### 服务依赖

| 服务 | 职责 |
|------|------|
| `IAstPointDataService` | 底层点位数据服务 |
| `ILogger<PointDataService>` | 日志记录 |
| `ISqlSugarRepository<AlarmRecordItemEntity, Guid>` | 告警记录项仓储（用于告警状态过滤） |

## 核心方法

### 1. CreateAsync - 新增点位数据

**签名**：
```csharp
public async Task<PointDataCreateDto> CreateAsync(PointDataCreateDto input)
```

**功能**：
直接调用 `AstPointDataService.CreateAsync`，新增点位数据记录。

**调用链**：
```
PointDataService.CreateAsync
  └─> AstPointDataService.CreateAsync
       └─> 数据持久化到 TDengine/SQLite
```

### 2. GetListAsync - 获取点位数据列表

**签名**：
```csharp
public async Task<PagedResultDto<PointDataDto>> GetListAsync(PointDataGetListInputDto input)
```

**功能**：
查询点位数据列表，支持多维度过滤和分页。

**特点**：
- 添加详细的调试日志
- 记录查询参数和结果统计
- 直接调用底层服务

**日志记录**：
```csharp
_logger.LogDebug("查询点位数据列表: DeviceId={DeviceId}, SensorKey={SensorKey}, ...");
_logger.LogDebug("点位数据查询完成: 总数={TotalCount}, 当前页数量={Count}");
```

### 3. GetListByPointIdsAsync - 按点位ID列表查询

**签名**：
```csharp
public async Task<PagedResultDto<PointDataDto>> GetListByPointIdsAsync(
    PointDataGetListByPointIdsInputDto input)
```

**功能**：
根据点位ID列表批量查询数据，支持分页。

**日志记录**：
- 记录点位ID数量
- 记录查询结果统计

### 4. GetLatestListByPointIdsAsync - 获取最新数据

**签名**：
```csharp
public async Task<List<PointDataDto>> GetLatestListByPointIdsAsync(
    PointDataGetLatestDataByPointIdsInputDto input)
```

**功能**：
获取每个点位的最新数据，用于实时监控面板。

**使用场景**：
- 实时监控面板显示
- 设备状态概览
- 最新数据刷新

### 5. GetHistoryDataAsync - 历史数据聚合查询

**签名**：
```csharp
[HttpGet("point-data/history")]
public async Task<PagedResultDto<DataPointDto>> GetHistoryDataAsync(
    HistoryDataGetListInputDto input)
```

**功能**：
查询历史数据并进行降采样聚合，提供时序数据的统计分析。

**特性**：
- 带有路由特性 `[HttpGet("point-data/history")]`
- 详细的日志记录
- 聚合统计（平均值、最大值、最小值）

**聚合说明**：
- 数据按指定时间间隔（`intervalInMinutes`）分组
- 每个时间窗口返回统计值
- 适用于大数据量的历史趋势分析

### 6. GetFilteredByAlarmStatusAsync - 告警状态过滤 ⭐核心方法

**签名**：
```csharp
[HttpGet("point-data/filtered-by-alarm-status")]
public async Task<PagedResultDto<PointDataDto>> GetFilteredByAlarmStatusAsync(
    PointDataFilterInputDto input)
```

**功能**：
根据告警状态过滤机械特性分合闸点位数据，这是该服务最复杂的业务逻辑。

**执行流程**：

```
1. 查询原始点位数据
   │
   ├─ 调用 AstPointDataService.QueryRawPointDataAsync
   ├─ 查询参数：IsOpenPointId, IsOpen, 时间范围等
   │
2. 提取 GroupId 列表
   │
   ├─ 从原始数据中提取所有唯一的 group_id
   ├─ 用于后续告警记录关联查询
   │
3. 查询告警记录（关联 AstPoint 表）
   │
   ├─ 根据是否指定 PropertyPrefix 分两种情况：
   │
   │  a) 指定属性前缀：
   │     ├─ 关联 AstPoint 表
   │     ├─ 过滤 Property 以指定前缀开头
   │     └─ 只检查该前缀的点位告警
   │
   │  b) 未指定属性前缀：
   │     └─ 检查所有属性点位的告警
   │
   ├─ 查询结果：异常 group_id 列表
   │
4. 内存过滤（关键逻辑）
   │
   ├─ 遍历原始点位数据
   ├─ 判断每个点位的 group_id 是否在异常列表中
   ├─ 根据 IsNormal 参数过滤：
   │  • IsNormal=true: 返回正常数据（不在告警记录中）
   │  • IsNormal=false: 返回异常数据（在告警记录中）
   │  • IsNormal=null: 不过滤，返回所有数据
   │
5. 手动分页
   │
   └─ 返回分页结果
```

**优化点**：
- ✅ 先提取 group_id，只查询相关告警记录
- ✅ 支持 PropertyPrefix 过滤，减少关联查询
- ✅ 使用 HashSet 提高查找效率
- ✅ 详细的日志记录每个步骤

**日志记录示例**：
```csharp
_logger.LogDebug("查询到原始pointdata记录数: {Count}", rawPointData.Count);
_logger.LogDebug("从原始数据中提取到 {GroupIdCount} 个唯一的 group_id", groupIds.Count);
_logger.LogDebug("使用属性前缀过滤 (PropertyPrefix={PropertyPrefix}): ...");
_logger.LogDebug("过滤后数据量: {Count}", filteredData.Count);
_logger.LogDebug("根据告警状态过滤完成: 总数={TotalCount}, ...");
```

## 数据传输对象

### PointDataFilterInputDto - 告警状态过滤输入

```csharp
public class PointDataFilterInputDto
{
    public Guid IsOpenPointId { get; set; }        // IsOpen属性对应的点位ID
    public int IsOpen { get; set; }                // IsOpen值过滤
    public bool? IsNormal { get; set; }            // 是否查询正常数据
    public string? DeviceId { get; set; }          // 设备ID
    public string? SensorKey { get; set; }         // 传感器键值
    public DateTime? StartTime { get; set; }       // 开始时间
    public DateTime? EndTime { get; set; }         // 结束时间
    public string? PropertyPrefix { get; set; }     // 属性前缀过滤
    public bool IsAsc { get; set; }                // 是否按时间升序
    public int SkipCount { get; set; }             // 跳过记录数
    public int MaxResultCount { get; set; }        // 最大返回记录数
}
```

## API端点

### 1. 新增点位数据

```
POST /api/app/ast-intellisub/point-data
```

### 2. 获取点位数据列表

```
GET /api/app/ast-intellisub/point-data
```

### 3. 按点位ID列表查询

```
GET /api/app/ast-intellisub/point-data/by-point-ids
```

### 4. 获取最新数据

```
GET /api/app/ast-intellisub/point-data/latest
```

### 5. 历史数据聚合

```
GET /api/app/ast-intellisub/point-data/history
```

### 6. 告警状态过滤

```
GET /api/app/ast-intellisub/point-data/filtered-by-alarm-status
```

## 日志级别

| 级别 | 用途 |
|------|------|
| LogDebug | 详细的查询参数、中间步骤、结果统计 |
| LogError | 异常和错误信息 |

**日志示例**：
```csharp
// 查询开始
_logger.LogDebug("查询点位数据列表: DeviceId={DeviceId}, SensorKey={SensorKey}, ...");

// 查询完成
_logger.LogDebug("点位数据查询完成: 总数={TotalCount}, 当前页数量={Count}");

// 异常记录
_logger.LogError(ex, "根据告警状态过滤点位数据失败: IsOpenPointId={IsOpenPointId}");
```

## 错误处理

### 异常类型

| 异常类型 | 场景 | 处理方式 |
|---------|------|---------|
| `UserFriendlyException` | 业务逻辑失败 | 返回友好错误消息 |
| `Exception` | 未预期的错误 | 记录日志并抛出 |

### 错误响应示例

```json
{
  "error": {
    "code": "500",
    "message": "过滤点位数据失败",
    "details": "..."
  }
}
```

## 业务场景

### 1. 实时监控面板

使用 `GetLatestListByPointIdsAsync` 获取关键点位的最新数据：
```typescript
const pointIds = ['temp-001', 'vibration-002'];
const data = await pointDataService.getLatestListByPointIdsAsync({ pointIds });
```

### 2. 历史趋势分析

使用 `GetHistoryDataAsync` 获取聚合历史数据：
```typescript
const data = await pointDataService.getHistoryDataAsync({
  deviceId: 'device-001',
  sensorKey: 'temperature',
  startTime: startDate,
  endTime: endDate,
  intervalInMinutes: 60  // 每小时一个点
});
```

### 3. 机械特性分合闸分析

使用 `GetFilteredByAlarmStatusAsync` 过滤正常/异常数据：
```typescript
// 查询正常的分合闸数据
const normalData = await pointDataService.getFilteredByAlarmStatusAsync({
  isOpenPointId: 'point-guid',
  isOpen: 1,
  isNormal: true,
  startTime: startDate,
  endTime: endDate
});

// 查询异常的分合闸数据
const abnormalData = await pointDataService.getFilteredByAlarmStatusAsync({
  isOpenPointId: 'point-guid',
  isOpen: 1,
  isNormal: false,
  startTime: startDate,
  endTime: endDate
});
```

## 设计特点

### 1. 职责分离

- **PointDataService (API层)** - 统一接口、日志、业务逻辑
- **AstPointDataService (数据层)** - 数据持久化、数据库操作

### 2. 日志完整性

- 每个方法都有详细的调试日志
- 记录输入参数和输出统计
- 便于问题排查和性能分析

### 3. 性能优化

- 告警状态过滤使用 `HashSet` 提高查找效率
- 先提取 group_id，只查询相关告警记录
- 支持 PropertyPrefix 减少关联查询

### 4. 错误处理

- 捕获异常并记录日志
- 返回友好的错误消息
- 不暴露内部实现细节

## 相关服务

- [[AstPointDataService]] - 底层点位数据服务
- [[AlarmService]] - 告警管理服务
- [[PointDataCleanupJob]] - 数据清理任务

## 相关实体

- [[PointData]] - 点位数据实体
- [[AlarmRecordItemEntity]] - 告警记录项实体
- [[AstPointEntity]] - 数据点定义实体

## 相关文档

- [[传感器数据API]] - API接口文档
- [[时序数据管理]] - 时序数据管理概述
- [[数据上报流程]] - 数据上报流程说明

---

> **最后更新**：2026-06-04  
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Application/Services/PointDataService.cs`  
> **接口定义**：`module/ast-intellisub/Ast.IntelliSub.Application.Contracts/IServices/IPointDataService.cs`
