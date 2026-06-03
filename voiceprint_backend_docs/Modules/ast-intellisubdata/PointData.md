# 点位数据实体 (PointData)

## 概述

**PointData** 是时序点位数据的核心实体，支持 TDengine 超级表和普通关系型数据库两种存储模式。

**源码位置**：`module/ast-intellisubdata/Ast.IntelliSubData.Domain/Entities/PointData.cs`

## 实体定义

### TDengine 模式
- **超级表名**：`ast_pointdata`
- **三级 Tag 结构**：`DeviceId` + `SensorKey` + `Property`
- **自动创建子表**：基于 Tag 组合

### SQLite 模式
- **表名**：`ast_pointdata`（与 TDengine 统一）
- **索引优化**：为 Tag 字段创建索引

## 字段结构

### 主键字段

| 字段名 | 类型 | 说明 | TDengine | SQLite |
|--------|------|------|----------|--------|
| `Id` | int | 自增主键 | ❌ 忽略 | ✅ 主键 |
| `Ts` | TIMESTAMP | 事件时间戳 | ✅ 主键 | ✅ 索引 |

### Tag 字段（TDengine）

| 字段名 | 类型 | 说明 | 作用 |
|--------|------|------|------|
| `DeviceId` | NCHAR(36) | 设备 ID | Tag 1 |
| `SensorKey` | NCHAR | 传感器键值 | Tag 2 |
| `Property` | NCHAR | 属性名称 | Tag 3 |

### 数据字段

| 字段名 | 类型 | 说明 | 用途 |
|--------|------|------|------|
| `Val` | DOUBLE? | 数值类型值 | 存储 Int、Float、Enum |
| `StrVal` | NCHAR(4000)? | 字符串类型值 | 存储 String、Json |
| `ValueType` | INT | 值类型 | 0:int,1:float,2:string,3:json,4:enum |

### 普通列字段

| 字段名 | 类型 | 说明 |
|--------|------|------|
| `PointId` | NCHAR(36) | 业务点位 ID |
| `GroupId` | NCHAR(36)? | 分组 ID |
| `ExtInfo` | NCHAR(4000)? | 扩展信息 |
| `AlarmLevel` | SMALLINT | 告警级别 |

## 数据类型映射

### ValueType 枚举

| 值 | 枚举名 | 存储字段 | 示例 |
|----|--------|---------|------|
| 0 | Int | Val | 25 |
| 1 | Float | Val | 25.5 |
| 2 | String | StrVal | "正常" |
| 3 | Json | StrVal | {"temp": 25.5} |
| 4 | Enum | Val | 1 |

### 值选择逻辑
```csharp
switch (valueType)
{
    case DataValueTypeEnum.Int:
    case DataValueTypeEnum.Float:
    case DataValueTypeEnum.Enum:
        // 使用 Val 字段
        entity.Val = Convert.ToDouble(input.Value);
        break;
    
    case DataValueTypeEnum.String:
    case DataValueTypeEnum.Json:
        // 使用 StrVal 字段
        entity.StrVal = await _largeTextStorageManager.ProcessLargeTextAsync(jsonString);
        break;
}
```

## 主键策略

### TDengine 模式
- 使用 `Ts`（时间戳）作为主键
- `Id` 字段被忽略（`TDengineIgnore`）

### SQLite/PostgreSQL 模式
- 使用 `Id`（自增）作为主键
- `Ts` 字段有索引优化查询

### GetKeys() 实现
```csharp
public object[] GetKeys()
{
    // SQLite/PostgreSQL: 返回 Id
    // TDengine: 返回组合键
    return Id > 0 
        ? new object[] { Id } 
        : new object[] { DeviceId + SensorKey + Property + Ts };
}
```

## 特性标记

### SugarTable
```csharp
[SugarTable("ast_pointdata")]  // 统一表名
[IgnoreCodeFirst]              // 忽略主数据库 CodeFirst
```

### STableAttribute
```csharp
[STableAttribute(
    STableName = "ast_pointdata",
    Tag1 = nameof(DeviceId),   // Tag 1
    Tag2 = nameof(SensorKey),  // Tag 2
    Tag3 = nameof(Property)     // Tag 3
)]
```

### TDengineIgnore
```csharp
[TDengineIgnore("TDengine 使用 TIMESTAMP(Ts) 作为主键，不需要自增 Id")]
public int Id { get; set; }
```

## 使用示例

### 创建点位数据
```csharp
var pointData = new PointData
{
    Ts = DateTime.Now,
    PointId = "guid-123",
    DeviceId = "device-001",
    SensorKey = "temperature",
    Property = "value",
    Val = 25.5,
    ValueType = (int)DataValueTypeEnum.Float,
    AlarmLevel = 0
};
```

### TDengine 子表创建
```csharp
// 自动生成子表名
var childTableName = _childTableNameManager.GenerateChildTableName(
    "ast_pointdata",
    pointData.DeviceId,
    pointData.SensorKey,
    pointData.Property
);

// 插入时自动创建子表
await _tdDb.Db.Insertable(pointData)
    .SetTDengineChildTableName((stable, row) => 
        _childTableNameManager.GenerateChildTableName(
            stable, row.DeviceId, row.SensorKey, row.Property))
    .ExecuteCommandAsync();
```

## 索引策略

### TDengine 模式
- Tag 字段天然分区
- `Ts` 字段作为主键
- 基于 Tag 的查询自动优化

### SQLite 模式
```sql
CREATE INDEX idx_pointdata_ts ON ast_pointdata(ts);
CREATE INDEX idx_pointdata_deviceid ON ast_pointdata(deviceid);
CREATE INDEX idx_pointdata_sensorkey ON ast_pointdata(sensorkey);
CREATE INDEX idx_pointdata_property ON ast_pointdata(property);
```

## 相关文档

- [[AstPointDataService]]
- [[时序数据管理]]
- [[TDengine 集成]]
- [[数据类型映射]]

---

**最后更新**：2026-06-03
