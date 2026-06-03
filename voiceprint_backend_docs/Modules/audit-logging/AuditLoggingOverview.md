# 审计日志模块概述

## 概述

审计日志模块（audit-logging）提供完整的系统操作审计功能，记录所有用户操作、数据变更和系统事件，支持安全审计、合规检查和问题追溯。

**模块路径**：`module/audit-logging/`

## 功能说明

### 操作审计
记录所有用户的 API 调用和业务操作，包括：
- HTTP 请求详情（方法、URL、参数）
- 执行时间和耗时
- 客户端信息（IP、浏览器、设备）
- 用户信息和租户信息
- 操作结果和异常

### 数据变更审计
记录实体数据的详细变更历史：
- 变更类型（创建、更新、删除）
- 变更前后的属性值对比
- 实体类型和实体 ID
- 变更时间和操作用户

### 登录审计
记录用户认证和授权相关事件：
- 用户登录/登出
- 身份模拟操作
- 租户切换
- Token 刷新

## 主要服务

### AuditingStore
审计日志存储服务，负责保存审计日志。

**关键方法**：
- `SaveAsync(AuditLogInfo auditInfo)` - 保存审计日志
- `SaveLogAsync(AuditLogInfo auditInfo)` - 内部保存逻辑

### AuditLogInfoToAuditLogConverter
审计日志信息转换器，将 `AuditLogInfo` 转换为 `AuditLogAggregateRoot` 实体。

### IAuditLogRepository
审计日志仓储接口，提供查询功能。

**查询方法**：
- `GetListAsync(...)` - 获取审计日志列表（支持多条件筛选）
- `GetCountAsync(...)` - 获取审计日志数量
- `GetAverageExecutionDurationPerDayAsync(...)` - 获取每日平均执行时长
- `GetEntityChangeListAsync(...)` - 获取实体变更列表
- `GetEntityChangesWithUsernameAsync(...)` - 获取带用户名的实体变更

## 数据结构

### AuditLogAggregateRoot
审计日志主实体。

**主要字段**：
| 字段 | 类型 | 说明 |
|------|------|------|
| Id | Guid | 主键 |
| TenantId | Guid? | 租户 ID |
| UserId | Guid? | 用户 ID |
| UserName | string? | 用户名 |
| ApplicationName | string? | 应用名称 |
| ExecutionTime | DateTime? | 执行时间 |
| ExecutionDuration | int? | 执行耗时（毫秒） |
| ClientIpAddress | string? | 客户端 IP |
| ClientId | string? | 客户端 ID |
| HttpMethod | string? | HTTP 方法 |
| Url | string? | 请求 URL |
| HttpStatusCode | int? | HTTP 状态码 |
| Exceptions | string? | 异常信息 |
| Comments | string? | 备注 |

**导航属性**：
- `EntityChanges` - 实体变更列表
- `Actions` - 操作日志列表

### EntityChangeEntity
实体变更记录。

**主要字段**：
| 字段 | 类型 | 说明 |
|------|------|------|
| Id | Guid | 主键 |
| AuditLogId | Guid | 审计日志 ID |
| ChangeTime | DateTime | 变更时间 |
| ChangeType | EntityChangeType | 变更类型 |
| EntityId | string | 实体 ID |
| EntityTypeFullName | string | 实体类型全名 |

**导航属性**：
- `PropertyChanges` - 属性变更列表

### EntityPropertyChangeEntity
属性变更详情。

**主要字段**：
| 字段 | 类型 | 说明 |
|------|------|------|
| Id | Guid | 主键 |
| EntityChangeId | Guid | 实体变更 ID |
| PropertyName | string | 属性名称 |
| OldValue | string? | 原值 |
| NewValue | string? | 新值 |

### AuditLogActionEntity
操作日志记录。

**主要字段**：
| 字段 | 类型 | 说明 |
|------|------|------|
| Id | Guid | 主键 |
| AuditLogId | Guid | 审计日志 ID |
| ServiceName | string | 服务名称 |
| MethodName | string | 方法名称 |
| Parameters | string | 参数（JSON） |
| ExecutionTime | DateTime | 执行时间 |
| ExecutionDuration | int | 执行耗时 |

## 查询接口

### 基础查询
```csharp
// 获取审计日志列表
var logs = await _auditLogRepository.GetListAsync(
    sorting: "ExecutionTime desc",
    maxResultCount: 50,
    skipCount: 0,
    startTime: DateTime.Today,
    endTime: DateTime.Today.AddDays(1),
    userId: currentUser.Id,
    hasException: false
);

// 获取实体变更历史
var changes = await _auditLogRepository.GetEntityChangesWithUsernameAsync(
    entityId: deviceId,
    entityTypeFullName: "Yi.Framework.Ast.IntelliSub.Domain.DeviceAggregateRoot"
);
```

### 统计查询
```csharp
// 获取每日平均执行时长
var avgDuration = await _auditLogRepository.GetAverageExecutionDurationPerDayAsync(
    startDate: DateTime.Today.AddDays(-30),
    endDate: DateTime.Today
);
```

## 日志保留策略

### 数据库存储
审计日志存储在 `YiAuditLog` 表中，支持多租户隔离（通过 `TenantId`）。

### 性能优化
- 为常用查询字段建立索引：
  - `(TenantId, ExecutionTime)`
  - `(TenantId, UserId, ExecutionTime)`
- 长文本字段（如参数、异常）进行长度截断
- 导航属性按需加载

### 配置选项
在 `appsettings.json` 中配置 ABP 审计选项：

```json
{
  "AbpAuditing": {
    "Enabled": true,
    "HideErrors": false,
    "ApplicationName": "IntelliSubstation",
    "AlwaysLogSelectors": [
      {
        "Type": "Yi.Framework.Ast.IntelliSub.Domain.DeviceAggregateRoot, Yi.Framework.Ast.IntelliSub.Domain",
        "PredicateName": "IsCriticalOperation"
      }
    ]
  }
}
```

## 相关文档

- [[ABP 框架集成]] - ABP 框架集成说明
- [[多租户模块]] - 租户管理说明
- [[数据访问层]] - SqlSugar ORM 使用说明

## 注意事项

1. **性能影响**：审计日志会带来一定的性能开销，建议在高并发场景下评估启用范围
2. **敏感数据**：避免记录敏感信息（如密码、Token），可通过配置排除特定字段
3. **存储容量**：定期清理历史日志，避免数据膨胀
4. **查询优化**：大表查询时建议使用分页和索引
