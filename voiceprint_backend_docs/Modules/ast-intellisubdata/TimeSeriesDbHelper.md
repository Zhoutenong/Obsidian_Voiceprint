# 时序数据库助手 (TimeSeriesDbHelper)

## 概述

**TimeSeriesDbHelper** 是时序数据库兼容性处理的核心工具类，处理 TDengine 和 SQLite/PostgreSQL 等不同数据库类型之间的语法差异。

**源码位置**：`module/ast-intellisubdata/Ast.IntelliSubData.SqlSugarCore/Helpers/TimeSeriesDbHelper.cs`

## 核心功能

### 1. 数据库类型判断

```csharp
bool IsTDengineMode(DbType dbType)
bool IsLowResourceMode(DbType dbType)
```

### 2. 聚合查询 SQL 构建

根据数据库类型生成不同的聚合查询语法：

**TDengine 模式**（使用 INTERVAL）：
```sql
SELECT 
    _wstart as timestamp,
    AVG(val) as avg_value,
    MAX(val) as max_value,
    MIN(val) as min_value
FROM ast_pointdata 
WHERE {whereClause}
INTERVAL({intervalMinutes}m)
ORDER BY timestamp ASC
```

**SQLite 模式**（使用 GROUP BY）：
```sql
SELECT 
    datetime((strftime('%s', ts) / ({intervalMinutes} * 60)) * ({intervalMinutes} * 60), 'unixepoch') as timestamp,
    AVG(val) as avg_value,
    MAX(val) as max_value,
    MIN(val) as min_value
FROM ast_pointdata 
WHERE {whereClause}
GROUP BY (strftime('%s', ts) / ({intervalMinutes} * 60))
ORDER BY timestamp ASC
```

### 3. 表名管理

统一使用 `ast_pointdata` 作为表名，无论在 TDengine 还是 SQLite 模式下。

## 数据库兼容性

### TDengine 特性
- 使用 `INTERVAL` 进行时间窗口聚合
- `_wstart` 作为窗口起始时间
- 支持超级表和子表结构

### SQLite 特性
- 使用 `GROUP BY` 和时间函数模拟聚合
- `strftime` 函数处理时间戳
- 普通表结构，创建索引优化查询

### PostgreSQL 特性
- 与 SQLite 类似的 GROUP BY 语法
- 使用 `date_trunc` 函数处理时间
- 支持更丰富的时间函数

## 使用示例

```csharp
// 构建聚合查询 SQL
var sql = TimeSeriesDbHelper.BuildAggregationSql(
    DbType.TDengine,
    10, // 10分钟间隔
    "deviceid = 'device-001' AND ts >= NOW - 1h"
);

// 判断数据库模式
if (TimeSeriesDbHelper.IsTDengineMode(dbType))
{
    // TDengine 特定处理
}
```

## 相关文档

- [[AstPointDataService]]
- [[TDengine 集成]]
- [[数据存储策略]]

---

**最后更新**：2026-06-03
