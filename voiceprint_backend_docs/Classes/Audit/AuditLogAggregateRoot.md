# AuditLogAggregateRoot — 审计日志聚合根

## 基本信息

- **实体名称**：`AuditLogAggregateRoot`
- **数据库表**：`YiAuditLog`
- **模块位置**：`module/audit-logging/Yi.Framework.AuditLogging.Domain/Entities/`
- **继承关系**：`AggregateRoot<Guid>` → `IMultiTenant`

## 实体说明

审计日志聚合根是 ABP 框架审计日志系统的核心实体，用于记录系统中所有重要的操作日志。每次 HTTP 请求、实体变更或重要操作都会生成审计日志，支持完整的操作追溯和合规审计。

## 字段说明

### 主键与租户

| 字段名 | 数据类型 | 说明 | 约束 |
|-------|---------|------|------|
| `Id` | `Guid` | 主键 | Primary Key |
| `TenantId` | `Guid?` | 租户ID | 多租户支持 |

### 用户信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `UserId` | `Guid?` | 操作用户ID | - |
| `UserName` | `string?` | 操作用户名 | 最大长度限制 |
| `ImpersonatorUserId` | `Guid?` | 模拟用户ID | 用户模拟场景 |
| `ImpersonatorUserName` | `string?` | 模拟用户名 | - |
| `ImpersonatorTenantId` | `Guid?` | 模拟租户ID | - |
| `ImpersonatorTenantName` | `string?` | 模拟租户名 | - |
| `TenantName` | `string?` | 租户名称 | - |

### 执行信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `ApplicationName` | `string?` | 应用名称 | 如："Yi.Abp.Web" |
| `ExecutionTime` | `DateTime?` | 执行时间 | 操作发生时间 |
| `ExecutionDuration` | `int?` | 执行时长（毫秒） | 用于性能分析 |
| `HttpStatusCode` | `int?` | HTTP状态码 | 如：200, 404, 500 |

### 客户端信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `ClientIpAddress` | `string?` | 客户端IP地址 | 最大长度限制 |
| `ClientName` | `string?` | 客户端名称 | - |
| `ClientId` | `string?` | 客户端ID | - |
| `BrowserInfo` | `string(2000)` | 浏览器信息 | User-Agent等 |
| `CorrelationId` | `string?` | 关联ID | 用于关联多个操作 |

### 请求信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `HttpMethod` | `string?` | HTTP方法 | GET、POST等 |
| `Url` | `string?` | 请求URL | - |

### 错误与备注

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `Exceptions` | `string?` | 异常信息 | BigString类型 |
| `Comments` | `string?` | 备注 | 最大长度限制 |

### 扩展属性

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `ExtraProperties` | `ExtraPropertyDictionary` | 扩展属性字典 | IsIgnore，不映射到数据库 |

## 导航属性

```csharp
[Navigate(NavigateType.OneToMany, nameof(EntityChangeEntity.AuditLogId))]
public virtual List<EntityChangeEntity> EntityChanges { get; protected set; }

[Navigate(NavigateType.OneToMany, nameof(AuditLogActionEntity.AuditLogId))]
public virtual List<AuditLogActionEntity> Actions { get; protected set; }
```

## 业务规则

### 审计日志记录规则
1. **自动记录**：ABP 框架自动记录所有 HTTP 请求
2. **实体变更**：自动记录实体的创建、修改、删除操作
3. **重要操作**：手动记录重要业务操作
4. **多租户隔离**：通过 TenantId 隔离不同租户的审计日志

### 性能考虑
- 使用 `[DisableAuditing]` 特性禁用对审计日志本身的审计
- 索引优化：TenantId + ExecutionTime 复合索引
- 定期清理：根据合规要求定期归档或清理旧日志

## 相关实体

- [[EntityChangeEntity]] — 实体变更记录（一对多）
- [[EntityPropertyChangeEntity]] — 属性变更记录
- [[AuditLogActionEntity]] — 审计日志操作记录

## 相关服务

- [[AuditLogService]] — 审计日志服务
- [[AuditLogActionService]] — 审计日志操作服务

## 数据查询示例

### 查询某用户的所有操作
```sql
SELECT * FROM YiAuditLog
WHERE user_id = 'xxx'
ORDER BY execution_time DESC
LIMIT 100;
```

### 查询某时间段的错误日志
```sql
SELECT * FROM YiAuditLog
WHERE execution_time >= '2026-06-04 00:00:00'
AND execution_time <= '2026-06-04 23:59:59'
AND http_status_code >= 400
ORDER BY execution_time DESC;
```

### 查询某 IP 的所有请求
```sql
SELECT
    user_name,
    http_method,
    url,
    http_status_code,
    execution_duration,
    execution_time
FROM YiAuditLog
WHERE client_ip_address = '192.168.1.100'
ORDER BY execution_time DESC;
```

### 统计用户操作频率
```sql
SELECT
    user_name,
    COUNT(*) as operation_count,
    AVG(execution_duration) as avg_duration
FROM YiAuditLog
WHERE execution_time >= DATEADD(DAY, -7, GETDATE())
GROUP BY user_name
ORDER BY operation_count DESC;
```

## 索引建议

```sql
-- 租户ID + 执行时间复合索引
CREATE INDEX index_ExecutionTime ON YiAuditLog(tenant_id, execution_time);

-- 租户ID + 用户ID + 执行时间复合索引
CREATE INDEX index_ExecutionTime_UserId ON YiAuditLog(tenant_id, user_id, execution_time);
```

## 业务价值

**核心作用**：
1. **合规审计**：满足电力行业对操作审计的合规要求
2. **安全追溯**：追踪所有用户操作，支持安全事件调查
3. **性能分析**：通过 ExecutionDuration 分析系统性能
4. **故障排查**：记录异常信息，辅助故障诊断
5. **操作统计**：支持用户行为分析和系统使用统计

## 使用场景

### 场景1：用户操作审计
```
查询用户 "张三" 在 2026-06-04 的所有操作：
- 10:15:23 - POST /api/app/device/create - 200 (45ms)
- 10:18:45 - POST /api/app/device/update - 200 (32ms)
- 10:22:10 - DELETE /api/app/device/delete - 200 (28ms)
```

### 场景2：实体变更追溯
```
查询设备 ID 为 xxx 的变更历史：
- 2026-06-04 10:18:45 - Modified - 张三修改设备名称
- 2026-06-03 14:20:30 - Created - 李四创建设备
```

### 场景3：异常操作监控
```
查询所有失败的操作（HTTP 状态码 >= 400）：
- 2026-06-04 11:05:22 - POST /api/app/device/create - 403 (权限不足)
- 2026-06-04 11:10:15 - PUT /api/app/device/update - 404 (设备不存在)
```

## 设计模式

### 聚合根模式
- `AuditLogAggregateRoot` 是聚合根
- `EntityChangeEntity` 和 `AuditLogActionEntity` 是聚合的一部分
- 只能通过聚合根访问和修改聚合内部对象

### 多租户支持
- 通过 `IMultiTenant` 接口支持多租户
- TenantId 用于隔离不同租户的审计日志
- 查询时自动过滤当前租户的日志

### 审计豁免
```csharp
[DisableAuditing]
public class AuditLogAggregateRoot
{
    // 避免无限递归：审计日志本身不需要审计
}
```

## 数据保留策略

根据行业合规要求，建议：
1. **在线存储**：最近 3-6 个月的审计日志
2. **归档存储**：6 个月至 3 年的审计日志（压缩存储）
3. **永久保存**：重要操作的审计日志（如用户权限变更）
4. **清理策略**：超过 3 年的普通审计日志可安全删除

---

> **最后更新**：2026-06-04
> **源码位置**：`module/audit-logging/Yi.Framework.AuditLogging.Domain/Entities/AuditLogAggregateRoot.cs`