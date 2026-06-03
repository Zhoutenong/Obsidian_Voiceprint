# PointData - 传感器数据实体

> 点位数据超级表（TDengine）/ 普通表（SQLite）
>
> **位置**: `module/ast-intellisubdata/Ast.IntelliSubData.Domain/Entities/PointData.cs`
>
> **表名**: `ast_pointdata`

## 概述

`PointData` 是时序传感器数据实体，采用多数据库架构：
- **TDengine 模式**: 超级表（STable），使用三级 Tag 结构
- **SQLite/PostgreSQL 模式**: 普通表，所有字段作为普通列

支持高频数据采集、可配置保留期、灵活的数据类型（数值/字符串/JSON/枚举）。

## 属性列表

| 属性名 | 类型 | 数据库列名 | 说明 | TDengine角色 |
|--------|------|-----------|------|--------------|
| `Id` | `int` | `id` | 自增主键（仅SQLite/PG） | IGNORED |
| `Ts` | `DateTime` | `ts` | 事件时间戳 | 主键 |
| `Val` | `double?` | `val` | 数值型属性值 | 数据列 |
| `StrVal` | `string?` | `str_val` | 字符串型属性值（NCHAR 4000） | 数据列 |
| `PointId` | `string` | `point_id` | 关联业务点位ID（NCHAR 36） | 数据列 |
| `GroupId` | `string?` | `group_id` | 分组ID（NCHAR 36） | 数据列 |
| `ValueType` | `int` | `value_type` | 值类型：0=int,1=float,2=string,3=json,4=enum | 数据列 |
| `ExtInfo` | `string?` | `ext_info` | 扩展信息（NCHAR 4000） | 数据列 |
| `AlarmLevel` | `short` | `alarm_level` | 告警级别：0=正常,1=预警,2=告警 | 数据列 |
| `DeviceId` | `string` | `deviceid` | 设备ID（Tag 1） | **Tag** |
| `SensorKey` | `string` | `sensorkey` | 传感器键值（Tag 2） | **Tag** |
| `Property` | `string` | `property` | 属性名称（Tag 3） | **Tag** |

## 方法列表

| 方法名 | 参数 | 返回值 | 说明 |
|--------|------|--------|------|
| `GetKeys` | - | `object[]` | 返回主键（SQLite: Id, TDengine: 复合主键） |

## 数据注释

```csharp
[SugarTable("ast_pointdata")]
[IgnoreCodeFirst]  // 不在主数据库CodeFirst扫描
[STableAttribute(
    STableName = "ast_pointdata",
    Tag1 = nameof(DeviceId), 
    Tag2 = nameof(SensorKey),
    Tag3 = nameof(Property)
)]
public class PointData : STable, IEntity
```

### TDengine 超级表定义

```sql
CREATE STABLE ast_pointdata (
    ts TIMESTAMP,                  -- 主键
    val DOUBLE,                    -- 数值
    str_val NCHAR(4000),          -- 字符串
    point_id NCHAR(36),           -- 业务点位ID
    group_id NCHAR(36),           -- 分组ID
    value_type INT,               -- 值类型
    ext_info NCHAR(4000),         -- 扩展信息
    alarm_level SMALLINT          -- 告警级别
) TAGS (
    deviceid NCHAR(100),          -- Tag 1: 设备ID
    sensorkey NCHAR(100),         -- Tag 2: 传感器键值
    property NCHAR(100)            -- Tag 3: 属性名称
);
```

### SQLite 表定义

```sql
CREATE TABLE ast_pointdata (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    ts TIMESTAMP NOT NULL,
    val DOUBLE,
    str_val TEXT,
    point_id TEXT NOT NULL,
    group_id TEXT,
    value_type INTEGER NOT NULL,
    ext_info TEXT,
    alarm_level SMALLINT DEFAULT 0,
    deviceid TEXT NOT NULL,
    sensorkey TEXT NOT NULL,
    property TEXT NOT NULL
);

CREATE INDEX ix_pointdata_ts ON ast_pointdata(ts);
CREATE INDEX ix_pointdata_deviceid ON ast_pointdata(deviceid);
```

## 枚举类型

### DataValueTypeEnum - 值类型

| 值 | 名称 | 存储字段 | 示例 |
|----|------|----------|------|
| `0` | `Int` | `Val` | `42` |
| `1` | `Float` | `Val` | `3.14` |
| `2` | `String` | `StrVal` | `"open"` |
| `3` | `Json` | `StrVal` | `"{\"temp\": 25}"` |
| `4` | `Enum` | `StrVal` | `"0:关,1:开"` |

### AlarmLevelEnum - 告警级别

| 值 | 名称 | 描述 |
|----|------|------|
| `0` | `Normal` | 正常 |
| `1` | `Warning` | 预警 |
| `2` | `Alarm` | 告警 |

## 架构设计

### TDengine 三级 Tag 结构

```
ast_pointdata (超级表)
├── Tags (时序标识)
│   ├── DeviceId     # 设备ID（Tag 1）
│   ├── SensorKey    # 传感器键值（Tag 2）
│   └── Property     # 属性名称（Tag 3）
└── Columns (数据值)
    ├── Ts           # 时间戳（主键）
    ├── Val          # 数值
    ├── StrVal       # 字符串
    └── ...          # 其他字段
```

### 子表命名规则

TDengine 自动为每个 (DeviceId, SensorKey, Property) 组合创建子表：

```
ast_pointdata_{deviceid}_{sensorkey}_{property}
```

示例：
```
ast_pointdata_device001_temp_Temperature
ast_pointdata_device001_pressure_Pressure
```

## 主键策略

### GetKeys() 实现

```csharp
public object[] GetKeys()
{
    // SQLite/PostgreSQL: 返回自增 Id
    // TDengine: 返回复合主键 (DeviceId + SensorKey + Property + Ts)
    return Id > 0 
        ? new object[] { Id } 
        : new object[] { DeviceId + SensorKey + Property + Ts };
}
```

### 主键对比

| 数据库 | 主键类型 | 唯一标识 |
|--------|----------|-----------|
| SQLite | `Id` (自增int) | `Id` |
| PostgreSQL | `Id` (SERIAL) | `Id` |
| TDengine | `Ts` (TIMESTAMP) | `(DeviceId, SensorKey, Property, Ts)` |

## 时序数据特性

### 高频写入支持

```csharp
// 批量插入（优化性能）
var dataList = new List<PointData>
{
    new PointData { Ts = now, DeviceId = "d1", SensorKey = "temp", Property = "T", Val = 25.5 },
    new PointData { Ts = now, DeviceId = "d1", SensorKey = "temp", Property = "T", Val = 26.1 },
    // ...
};

await _repository.InsertManyAsync(dataList);
```

### 批次聚合

`GroupId` 标识同一批次上传的数据，天然支持按时间聚合：

```sql
-- 查询某批次的所有数据
SELECT * FROM ast_pointdata WHERE group_id = 'batch-001';
```

### 时间范围查询

```csharp
// 查询最近1小时数据
var recentData = await _repository._DbQueryable
    .Where(x => x.DeviceId == deviceId)
    .Where(x => x.SensorKey == sensorKey)
    .Where(x => x.Property == property)
    .Where(x => x.Ts >= DateTime.Now.AddHours(-1))
    .OrderBy(x => x.Ts)
    .ToListAsync();
```

## 数据清理策略

### 保留期配置

```json
{
  "PointDataCleanupJob": {
    "Enabled": true,
    "CronExpression": "0 0 2 * * *",
    "RetentionDays": 90
  }
}
```

### 自动清理逻辑

```csharp
// PointDataCleanupJob 实现
var cutoffDate = DateTime.Now.AddDays(-_options.RetentionDays);
var oldData = await _repository._DbQueryable
    .Where(x => x.Ts < cutoffDate)
    .ToListAsync();

if (oldData.Any())
{
    await _repository.DeleteAsync(oldData);
    _logger.LogInformation("清理了 {Count} 条过期数据", oldData.Count);
}
```

## 使用示例

### 插入数值型数据

```csharp
var pointData = new PointData
{
    Ts = DateTime.UtcNow,
    DeviceId = "transformer-001",
    SensorKey = "temp",
    Property = "Temperature",
    Val = 45.6,
    ValueType = (int)DataValueTypeEnum.Float,
    AlarmLevel = (short)AlarmLevelEnum.Normal
};

await _repository.InsertAsync(pointData);
```

### 插入字符串型数据

```csharp
var pointData = new PointData
{
    Ts = DateTime.UtcNow,
    DeviceId = "switch-001",
    SensorKey = "status",
    Property = "IsOpen",
    StrVal = "1",  // 枚举值
    ValueType = (int)DataValueTypeEnum.Enum,
    AlarmLevel = (short)AlarmLevelEnum.Normal
};
```

### 插入JSON数据

```csharp
var jsonData = JsonSerializer.Serialize(new { temp = 25.5, humidity = 60 });

var pointData = new PointData
{
    Ts = DateTime.UtcNow,
    DeviceId = "sensor-001",
    SensorKey = "env",
    Property = "Environment",
    StrVal = jsonData,
    ValueType = (int)DataValueTypeEnum.Json,
    AlarmLevel = (short)AlarmLevelEnum.Normal
};
```

### 带告警级别的数据

```csharp
var pointData = new PointData
{
    Ts = DateTime.UtcNow,
    DeviceId = "transformer-001",
    SensorKey = "temp",
    Property = "Temperature",
    Val = 95.0,  // 超温
    ValueType = (int)DataValueTypeEnum.Float,
    AlarmLevel = (short)AlarmLevelEnum.Alarm,  // 告警级别
    ExtInfo = "{\"threshold\": 90, \"actual\": 95}"  // 扩展信息
};
```

## TDengine 特性

### 超级表查询

```sql
-- 查询所有设备的温度数据
SELECT * FROM ast_pointdata WHERE property = 'Temperature';

-- 按设备分组统计
SELECT deviceid, COUNT(*) FROM ast_pointdata 
WHERE ts > NOW - 1h 
GROUP BY deviceid;

-- 最新数据（每个时序）
SELECT LAST_ROW(*) FROM ast_pointdata 
GROUP BY deviceid, sensorkey, property;
```

### 子表自动管理

```csharp
// 插入时TDengine自动创建子表（如果不存在）
await _repository.InsertAsync(new PointData
{
    Ts = DateTime.UtcNow,
    DeviceId = "new-device",  // 新设备
    SensorKey = "new-sensor", // 新传感器
    Property = "NewProperty", // 新属性
    Val = 100.0
});
// TDengine自动创建：ast_pointdata_new-device_new-sensor_NewProperty
```

## 性能优化

### SQLite 索引

```sql
CREATE INDEX ix_pointdata_ts ON ast_pointdata(ts);
CREATE INDEX ix_pointdata_deviceid ON ast_pointdata(deviceid);
CREATE INDEX ix_pointdata_composite ON ast_pointdata(deviceid, sensorkey, property);
```

### TDengine 分区

```sql
-- 按时间分区（自动优化）
-- 查询时自动利用分区剪枝
SELECT * FROM ast_pointdata WHERE ts > NOW - 1d;
```

### 批量写入

```csharp
// 使用 SqlSugar 批量插入优化
await _repository.Fastest<PointData>().BulkCopy(dataList);
```

## 相关实体

### AstPointEntity - 业务点位定义

```csharp
// 关联业务点位元数据
public class AstPointEntity
{
    public Guid Id { get; set; }
    public string DeviceId { get; set; }
    public string SensorKey { get; set; }
    public string Property { get; set; }
    public string Name { get; set; }  // 显示名称
    public string Unit { get; set; } // 单位
    public DataValueTypeEnum ValueType { get; set; }
}
```

## 相关服务

- `PointDataService` - 点位数据服务
- `PointDataCleanupJob` - 数据清理后台任务
- `PointDataRepository` - 数据仓储

## 数据库模式对比

| 特性 | SQLite | PostgreSQL | TDengine |
|------|--------|------------|----------|
| 表结构 | 普通表 | 普通表 | 超级表 |
| 主键 | `Id` (自增) | `Id` (SERIAL) | `Ts` + Tags |
| 时序优化 | ❌ | ❌ | ✅ |
| 压缩 | ❌ | ❌ | ✅ |
| 分区 | ❌ | ❌ | ✅ |
| 适用场景 | 低资源边缘设备 | 中小规模部署 | 大规模数据中心 |

## 注意事项

1. **双模式支持**: `Id` 字段仅在 SQLite/PG 模式有效，TDengine 模式忽略
2. **主键保护**: `Id` 使用 `IsIdentity = true`，自动生成
3. **时区处理**: `Ts` 使用 UTC 时间
4. **Tag 不可变**: TDengine 的 Tags (DeviceId/SensorKey/Property) 创建后不可修改
5. **数据类型**: 数值用 `Val`，字符串用 `StrVal`，根据 `ValueType` 判断
6. **告警级别**: `AlarmLevel` 与策略引擎配合使用
7. **CodeFirst 跳过**: 使用 `[IgnoreCodeFirst]` 避免主数据库扫描

---

**最后更新**: 2026-06-03
**模块**: `ast-intellisubdata`
