# 点位数据服务 (AstPointDataService)

## 概述

**AstPointDataService** 是时序数据入库的核心服务，负责点位数据的持久化存储和查询。作为 `ast-intellisubdata` 模块的数据访问层，该服务不直接暴露 API，而是为上层应用服务提供基础的数据操作能力。

**源码位置**：`module/ast-intellisubdata/Ast.IntelliSubData.Application/Services/AstPointDataService.cs`

## 核心职责

### 1. 数据持久化
- 点位数据的 CRUD 操作
- 支持 TDengine 和 SQLite/PostgreSQL 等多种数据库
- 自动处理大文本数据的文件存储

### 2. 时序数据管理
- 高频数据批量处理
- 历史数据聚合查询（降采样）
- 数据保留期管理和过期数据清理

### 3. 查询优化
- 基于时间序列的高效查询
- 支持多维度过滤（设备、传感器、属性、时间范围）
- 查询负载保护机制

## 主要接口

### CreateAsync
新增点位数据，支持多种数据类型和大文本存储。

```csharp
Task<PointDataCreateDto> CreateAsync(PointDataCreateDto input)
```

**功能特点**：
- 自动处理数值类型（Int、Float、Enum）
- 字符串类型支持大文本文件存储
- JSON 类型自动序列化
- TDengine 模式下自动生成子表名
- 详细的日志记录（成功/失败）

**支持的数据类型**：
- `DataValueTypeEnum.Int` - 整数
- `DataValueTypeEnum.Float` - 浮点数
- `DataValueTypeEnum.String` - 字符串
- `DataValueTypeEnum.Json` - JSON 对象
- `DataValueTypeEnum.Enum` - 枚举值

### GetListAsync
分页查询点位数据，支持多维度过滤。

```csharp
Task<PagedResultDto<PointDataDto>> GetListAsync(PointDataGetListInputDto input)
```

**支持的过滤条件**：
- 时间范围（StartTime/EndTime）
- 设备 ID（DeviceId）
- 传感器键值（SensorKey）
- 点位 ID（PointId）
- 分组 ID（GroupId）
- 属性名称（Property）
- 数值/字符串值

### GetListByPointIdsAsync
按点位 ID 列表查询数据，支持灵活查询。

```csharp
Task<PagedResultDto<PointDataDto>> GetListByPointIdsAsync(PointDataGetListByPointIdsInputDto input)
```

**特点**：
- 支持点位 ID 列表精确查询
- 当点位 ID 列表为空时，可通过设备 ID 或传感器键值查询
- 数据库层面分页，性能优化

### GetLatestListByPointIdsAsync
获取每个点位的最新数据。

```csharp
Task<List<PointDataDto>> GetLatestListByPointIdsAsync(PointDataGetLatestDataByPointIdsInputDto input)
```

**使用场景**：
- 实时监控面板显示
- 设备状态概览
- 最新数据刷新

### GetHistoryDataAsync
历史数据聚合查询，提供降采样功能。

```csharp
Task<PagedResultDto<DataPointDto>> GetHistoryDataAsync(HistoryDataGetListInputDto input)
```

**聚合统计**：
- `avg_value` - 平均值
- `max_value` - 最大值
- `min_value` - 最小值
- 支持自定义时间间隔（1-10080 分钟）

**查询保护**：
- 最大聚合点数限制：5000
- 防止过大时间范围查询导致性能问题

### QueryRawPointDataAsync
查询原始点位数据（不分页），供上层服务二次处理。

```csharp
Task<List<PointDataDto>> QueryRawPointDataAsync(
    Guid pointId, 
    int isOpen,
    bool isAsc,
    DateTime? startTime = null, 
    DateTime? endTime = null, 
    string? deviceId = null, 
    string? sensorKey = null)
```

**使用场景**：
- 机械特性分合闸分析
- 原始数据导出
- 自定义数据处理

### DeleteExpiredDataAsync
删除过期的点位数据。

```csharp
Task<int> DeleteExpiredDataAsync(int retentionDays, List<string> excludedSensorKeys = null)
```

**功能特点**：
- 基于保留天数自动清理
- 支持排除特定传感器数据
- 返回删除的记录数

## 数据存储策略

### TDengine 超级表设计

**超级表名**：`ast_pointdata`

**Tag 结构**（三级标签）：
1. `DeviceId` - 设备 ID
2. `SensorKey` - 传感器键值
3. `Property` - 属性名称

**子表命名规则**：
```
ast_pointdata_{clean_deviceId}_{clean_sensorKey}_{clean_property}
```

**字段设计**：

| 字段名 | 类型 | 说明 | TDengine 角色 |
|--------|------|------|---------------|
| `Ts` | TIMESTAMP | 事件时间戳 | 主键/时间戳 |
| `Val` | DOUBLE | 数值类型值 | 数据列 |
| `StrVal` | NCHAR(4000) | 字符串类型值 | 数据列 |
| `PointId` | NCHAR(36) | 业务点位 ID | 普通列 |
| `GroupId` | NCHAR(36) | 分组 ID | 普通列 |
| `ValueType` | INT | 值类型 | 普通列 |
| `ExtInfo` | NCHAR(4000) | 扩展信息 | 普通列 |
| `AlarmLevel` | SMALLINT | 告警级别 | 普通列 |

### 分表逻辑

**子表生成策略**：
- 基于 `(DeviceId, SensorKey, Property)` 三元组自动创建子表
- 使用 [[ChildTableNameManager]] 生成符合 TDengine 命名规范的子表名
- 清理非法字符（替换为下划线）
- 超长表名采用"截断 + 短哈希"策略

**文件存储路径**（大文本）：
```
{StorageDirectory}/{DeviceId}/{SensorKey}/{Ts}_{Property}_{PointId}.data
```

### 查询优化

**TDengine 模式**：
```sql
-- 聚合查询使用 INTERVAL 语法
SELECT 
    _wstart as timestamp,
    AVG(val) as avg_value,
    MAX(val) as max_value,
    MIN(val) as min_value
FROM ast_pointdata 
WHERE deviceid = @deviceId AND sensorkey = @sensorKey
INTERVAL(10m)
ORDER BY timestamp ASC
```

**SQLite 模式**：
```sql
-- 使用标准 GROUP BY 语法
SELECT 
    datetime((strftime('%s', ts) / (600)) * (600), 'unixepoch') as timestamp,
    AVG(val) as avg_value,
    MAX(val) as max_value,
    MIN(val) as min_value
FROM ast_pointdata 
WHERE deviceid = @deviceId AND sensorkey = @sensorKey
GROUP BY (strftime('%s', ts) / (600))
ORDER BY timestamp ASC
```

## 依赖服务

### 核心依赖

| 服务 | 职责 |
|------|------|
| [[ITDengineDbContext]] | TDengine 数据库上下文 |
| [[ILargeTextStorageManager]] | 大文本存储管理 |
| [[IPointDataRepository]] | 点位数据仓储 |
| [[ChildTableNameManager]] | TDengine 子表名生成 |
| [[TimeSeriesDbHelper]] | 数据库兼容性处理 |

### 相关实体

- [[PointData]] - 点位数据实体
- [[PointDataDto]] - 点位数据传输对象
- [[DataPointDto]] - 聚合数据点对象

### 相关枚举

- [[DataValueTypeEnum]] - 数据值类型（Int、Float、String、Json、Enum）
- [[AlarmLevelEnum]] - 告警级别（Normal、Warning、Alarm）

## 性能优化建议

### 1. 查询优化

**推荐做法**：
- 始终提供时间范围过滤（`StartTime`/`EndTime`）
- 优先使用 `(DeviceId, SensorKey, Property)` 组合查询
- 避免过大的时间范围查询

**避免做法**：
- ❌ 查询全表数据而不指定时间范围
- ❌ 使用 `LIKE` 模糊查询 Tag 字段
- ❌ 频繁查询小间隔的长时间范围数据

### 2. 数据写入优化

**推荐做法**：
- 使用批量插入而非单条插入
- 合理设置 `GroupId` 聚合同批次数据
- 启用大文本存储避免 TDengine 字段长度限制

**避免做法**：
- ❌ 高频写入时使用同步等待
- ❌ 存储超大 JSON 对象到 TDengine

### 3. 存储优化

**TDengine 模式**：
- 利用超级表的 Tag 分区特性
- 合理设置数据保留期
- 定期清理过期数据（[[DeleteExpiredDataAsync]]）

**SQLite 模式**：
- 为 `(DeviceId, SensorKey, Property, Ts)` 创建复合索引
- 定期 VACUUM 优化数据库文件

### 4. 查询负载保护

**聚合查询限制**：
- 最大聚合点数：5000
- 单页最大记录数：1000
- 时间间隔范围：1-10080 分钟

**计算公式**：
```
预期聚合点数 = (EndTime - StartTime).TotalMinutes / IntervalInMinutes
```

## 错误处理

### 常见异常

| 异常类型 | 场景 | 处理方式 |
|---------|------|---------|
| `UserFriendlyException` | 业务规则验证失败 | 返回友好错误消息 |
| `JsonException` | JSON 解析失败 | 记录详细诊断信息 |
| `SqlSugarException` | 数据库操作失败 | 记录日志并重试 |

### 日志级别

- **LogDebug** - 详细调试信息（SQL 语句、参数值）
- **LogInformation** - 关键操作（数据创建、删除）
- **LogWarning** - 查询性能问题、空结果
- **LogError** - 异常和错误

## 相关文档

### 模块文档
- [[ast-intellisubdata 模块概览]]
- [[时序数据管理]]
- [[TDengine 集成]]

### 服务文档
- [[传感器数据管理服务]]
- [[数据上报流程]]
- [[大文本存储管理]]

### 配置文档
- [[LargeTextStorageOptions]]
- [[TDengine 配置]]

## 使用示例

### 创建点位数据
```csharp
var input = new PointDataCreateDto
{
    PointId = "guid-123",
    DeviceId = "device-001",
    SensorKey = "temperature",
    Property = "value",
    Value = 25.5,
    ValueType = DataValueTypeEnum.Float,
    Ts = DateTime.Now,
    AlarmLevel = AlarmLevelEnum.Normal
};

await _pointDataService.CreateAsync(input);
```

### 查询历史数据聚合
```csharp
var input = new HistoryDataGetListInputDto
{
    DeviceId = "device-001",
    SensorKey = "temperature",
    Property = "value",
    StartTime = DateTime.Now.AddDays(-7),
    EndTime = DateTime.Now,
    IntervalInMinutes = 60, // 1小时间隔
    SkipCount = 1,
    MaxResultCount = 100
};

var result = await _pointDataService.GetHistoryDataAsync(input);
```

### 删除过期数据
```csharp
// 删除 30 天前的数据，排除关键传感器
var excludedSensors = new List<string> { "critical-sensor-1", "critical-sensor-2" };
var deletedCount = await _pointDataService.DeleteExpiredDataAsync(30, excludedSensors);
```

---

**最后更新**：2026-06-03
**文档版本**：v1.0.0
