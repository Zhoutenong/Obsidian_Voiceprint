# OperationLogEntity — 操作日志实体

## 基本信息

- **实体名称**：`OperationLogEntity`
- **数据库表**：`OperationLog`
- **模块位置**：`module/rbac/Yi.Framework.Rbac.Domain/Operlog/`
- **继承关系**：`Entity<Guid>` → `ICreationAuditedObject`

## 实体说明

操作日志实体用于记录用户在系统中的所有重要操作，如删除、修改、新增等。与审计日志不同，操作日志更关注业务层面的操作行为，提供完整的操作追溯链路。

## 字段说明

### 主键与审计

| 字段名 | 数据类型 | 说明 | 约束 |
|-------|---------|------|------|
| `Id` | `Guid` | 主键 | Primary Key |
| `CreationTime` | `DateTime` | 创建时间（操作时间） | ICreationAuditedObject |
| `CreatorId` | `Guid?` | 创建者ID（操作用户ID） | ICreationAuditedObject |

### 操作信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `Title` | `string?` | 操作模块 | 如："设备管理"、"用户管理" |
| `OperType` | `OperEnum` | 操作类型 | 见下方枚举 |
| `OperUser` | `string?` | 操作人员 | - |
| `Method` | `string?` | 操作方法 | 服务方法名 |

### 请求信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `RequestMethod` | `string?` | 请求方法 | HTTP方法：GET、POST等 |
| `RequestParam` | `string?` | 请求参数 | BigString类型 |
| `RequestResult` | `string?` | 请求结果 | BigString类型 |

### 客户端信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `OperIp` | `string?` | 操作IP | - |
| `OperLocation` | `string?` | 操作地点 | - |

## 枚举类型

### OperEnum — 操作类型
```csharp
public enum OperEnum
{
    Other = 0,       // 其他
    Delete = 1,      // 删除
    Update = 2,      // 更新
    Add = 3,         // 新增
    Import = 4,      // 导入
    Export = 5,      // 导出
    Query = 6        // 查询
}
```

## 业务规则

### 日志记录规则
1. **重要操作记录**：删除、修改、新增等操作自动记录
2. **参数完整**：保存请求参数和结果，支持完整追溯
3. **用户追踪**：记录操作用户、IP、地点等信息
4. **模块分类**：通过Title字段区分不同业务模块

### 操作级别
- **高危操作**：Delete（删除）、权限变更等
- **重要操作**：Update（修改）、Import（导入）
- **普通操作**：Add（新增）、Query（查询）
- **数据操作**：Export（导出）

## 相关服务

- [[OperationLogService]] — 操作日志服务
- [[AccountService]] — 账号服务

## 数据查询示例

### 查询某用户的所有操作
```sql
SELECT * FROM OperationLog
WHERE creator_id = 'xxx'
ORDER BY creation_time DESC;
```

### 查询某模块的所有删除操作
```sql
SELECT * FROM OperationLog
WHERE title = '设备管理'
AND oper_type = 1  -- Delete
ORDER BY creation_time DESC;
```

### 查询异常操作（失败的操作）
```sql
SELECT * FROM OperationLog
WHERE request_result LIKE '%error%'
OR request_result LIKE '%失败%'
ORDER BY creation_time DESC;
```

### 统计操作频率
```sql
SELECT
    title,
    oper_type,
    COUNT(*) as count,
    oper_user
FROM OperationLog
WHERE creation_time >= DATEADD(DAY, -7, GETDATE())
GROUP BY title, oper_type, oper_user
ORDER BY count DESC;
```

## 索引建议

```sql
-- 操作用户索引
CREATE INDEX IX_OperUser ON OperationLog(oper_user);

-- 操作时间索引
CREATE INDEX IX_CreationTime ON OperationLog(creation_time);
```

## 业务价值

**核心作用**：
1. **操作追溯**：完整记录用户的业务操作行为
2. **安全审计**：支持安全事件的调查和责任认定
3. **行为分析**：分析用户操作习惯和频率
4. **故障排查**：通过参数和结果排查操作问题

## 与审计日志的区别

| 对比项 | 操作日志 | 审计日志 |
|-------|---------|---------|
| **记录粒度** | 业务操作层面 | HTTP请求层面 |
| **记录内容** | 操作类型、模块 | 完整HTTP信息 |
| **适用场景** | 业务行为审计 | 技术层面审计 |
| **参数详情** | 业务参数 | 请求/响应参数 |

## 使用场景

### 场景1：删除操作记录
```
操作模块：设备管理
操作类型：Delete
操作用户：张三
操作方法：DeleteDeviceAsync
请求参数：{"deviceId":"xxx"}
请求结果：{"success":true,"message":"设备删除成功"}
操作IP：192.168.1.100
操作地点：北京-北京市
```

### 场景2：数据修改记录
```
操作模块：用户管理
操作类型：Update
操作用户：李四
操作方法：UpdateUserAsync
请求参数：{"userId":"xxx","name":"新名称","status":1}
请求结果：{"success":true}
```

### 场景3：批量导入记录
```
操作模块：设备管理
操作类型：Import
操作用户：王五
操作方法：ImportDevicesAsync
请求参数：{"file":"devices.xlsx","count":100}
请求结果：{"success":true,"imported":98,"failed":2}
```

---

> **最后更新**：2026-06-04
> **源码位置**：`module/rbac/Yi.Framework.Rbac.Domain/Operlog/OperationLogEntity.cs`
