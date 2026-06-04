# AuditLogActionEntity — 审计日志操作实体

## 基本信息

- **实体名称**：`AuditLogActionEntity`
- **数据库表**：`YiAuditLogAction`
- **模块位置**：`module/audit-logging/Yi.Framework.AuditLogging.Domain/Entities/`
- **继承关系**：`Entity<Guid>` → `IMultiTenant`

## 实体说明

审计日志操作实体用于记录审计日志中执行的具体操作方法，包括服务名称、方法名、参数、执行时间等详细信息。每个审计日志可以包含多个操作记录。

## 字段说明

### 主键与关联

| 字段名 | 数据类型 | 说明 | 约束 |
|-------|---------|------|------|
| `Id` | `Guid` | 主键 | Primary Key |
| `AuditLogId` | `Guid` | 关联审计日志ID | Foreign Key |
| `TenantId` | `Guid?` | 租户ID | 多租户支持 |

### 操作信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `ServiceName` | `string?` | 服务名称 | 如："DeviceAppService" |
| `MethodName` | `string?` | 方法名称 | 如："CreateAsync" |
| `Parameters` | `string?` | 方法参数 | JSON格式的参数信息 |

### 执行信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `ExecutionTime` | `DateTime?` | 执行时间 | 方法执行的时间戳 |
| `ExecutionDuration` | `int?` | 执行时长（毫秒） | 方法执行耗时 |

## 业务规则

### 操作记录规则
1. **自动记录**：ABP框架自动记录Application Service的方法调用
2. **参数序列化**：方法参数序列化为JSON字符串存储
3. **性能分析**：通过ExecutionDuration分析方法性能
4. **租户隔离**：通过TenantId隔离不同租户的操作记录

### 记录触发场景
- Application Service中的公开方法调用
- Repository的数据库操作
- Domain Service的业务逻辑执行

## 相关实体

- [[AuditLogAggregateRoot]] — 审计日志聚合根（多对一）

## 相关服务

- [[AuditLogActionService]] — 审计日志操作服务
- [[AuditLogService]] — 审计日志服务

## 数据查询示例

### 查询某审计日志的所有操作
```sql
SELECT 
    service_name,
    method_name,
    parameters,
    execution_time,
    execution_duration
FROM YiAuditLogAction
WHERE audit_log_id = 'xxx'
ORDER BY execution_time;
```

### 查询执行缓慢的操作
```sql
SELECT 
    service_name,
    method_name,
    AVG(execution_duration) as avg_duration,
    MAX(execution_duration) as max_duration,
    COUNT(*) as call_count
FROM YiAuditLogAction
WHERE execution_time >= DATEADD(HOUR, -1, GETDATE())
GROUP BY service_name, method_name
HAVING AVG(execution_duration) > 1000  -- 超过1秒
ORDER BY avg_duration DESC;
```

### 查询某服务的操作统计
```sql
SELECT 
    method_name,
    COUNT(*) as call_count,
    AVG(execution_duration) as avg_duration
FROM YiAuditLogAction
WHERE service_name = 'DeviceAppService'
AND execution_time >= DATEADD(DAY, -7, GETDATE())
GROUP BY method_name
ORDER BY call_count DESC;
```

## 索引建议

```sql
-- 审计日志ID索引（用于查询某审计日志的操作）
CREATE INDEX index_AuditLogId ON YiAuditLogAction(audit_log_id);

-- 租户ID + 服务名 + 方法名 + 执行时间复合索引
CREATE INDEX index_TenantId_ExecutionTime ON YiAuditLogAction(tenant_id, service_name, method_name, execution_time);
```

## 操作示例

### 示例1：设备创建操作
```
审计日志ID：audit-log-guid-123
服务名称：DeviceAppService
方法名称：CreateAsync
参数：{"name":"变压器A","type":1,"status":1}
执行时间：2026-06-04 10:15:30
执行时长：145ms
```

### 示例2：用户更新操作
```
审计日志ID：audit-log-guid-456
服务名称：UserAppService
方法名称：UpdateAsync
参数：{"userId":"xxx","name":"新名称","email":"new@example.com"}
执行时间：2026-06-04 11:20:45
执行时长：89ms
```

### 示例3：复杂业务操作
```
审计日志ID：audit-log-guid-789
服务名称：PatrolExecutionService
方法名称：ExecutePatrolAsync
参数：{"taskId":"xxx","startTime":"2026-06-04 09:00:00","pointCount":15}
执行时间：2026-06-04 09:00:05
执行时长：15230ms（15.23秒）
```

## 业务价值

**核心作用**：
1. **详细追踪**：记录每个方法调用的详细信息
2. **性能分析**：通过执行时长分析系统性能
3. **参数追溯**：保存方法参数，支持问题复现
4. **调用链分析**：通过服务名和方法名分析调用链

## 性能优化建议

### 参数长度限制
- Parameters字段通过`Truncate`方法限制长度
- 过长的参数会被截断或记录为空字符串
- 避免存储过大的参数（如文件内容）

### 查询优化
- 避免查询所有Parameters字段（仅在需要时查询）
- 使用ServiceName和MethodName进行过滤
- 分页查询操作记录

### 数据清理
- 定期清理旧的操作记录（如3个月前）
- 保留重要操作的历史记录
- 归档关键业务的操作日志

## 设计模式

### 值对象模式
```csharp
// 使用AuditLogActionInfo值对象创建
public AuditLogActionEntity(
    Guid id, 
    Guid auditLogId, 
    AuditLogActionInfo actionInfo, 
    Guid? tenantId = null)
{
    Id = id;
    TenantId = tenantId;
    AuditLogId = auditLogId;
    ExecutionTime = actionInfo.ExecutionTime;
    ExecutionDuration = actionInfo.ExecutionDuration;
    
    // 字符串截断防止超长
    ServiceName = actionInfo.ServiceName.TruncateFromBeginning(512);
    MethodName = actionInfo.MethodName.TruncateFromBeginning(512);
    Parameters = actionInfo.Parameters.Length > 2000 ? "" : actionInfo.Parameters;
}
```

---

> **最后更新**：2026-06-04
> **源码位置**：`module/audit-logging/Yi.Framework.AuditLogging.Domain/Entities/AuditLogActionEntity.cs`
