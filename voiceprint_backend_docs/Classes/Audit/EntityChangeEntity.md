# EntityChangeEntity — 实体变更记录实体

## 基本信息

- **实体名称**：`EntityChangeEntity`
- **数据库表**：`YiEntityChange`
- **模块位置**：`module/audit-logging/Yi.Framework.AuditLogging.Domain/Entities/`
- **继承关系**：`Entity<Guid>` → `IMultiTenant`

## 实体说明

实体变更记录实体用于记录领域实体的变更操作，包括创建、修改、删除。每次实体状态变更时，ABP 框架会自动创建变更记录，并保存变更前后的属性值。

## 字段说明

### 主键与关联

| 字段名 | 数据类型 | 说明 | 约束 |
|-------|---------|------|------|
| `Id` | `Guid` | 主键 | Primary Key |
| `AuditLogId` | `Guid` | 关联审计日志ID | Foreign Key |
| `TenantId` | `Guid?` | 租户ID | 多租户支持 |

### 变更信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `ChangeTime` | `DateTime?` | 变更时间 | 实体变更发生的时间 |
| `ChangeType` | `EntityChangeType?` | 变更类型 | Created/Modified/Deleted |
| `EntityId` | `string?` | 实体ID | 被变更实体的ID |
| `EntityTypeFullName` | `string?` | 实体类型全名 | 如："Ast.IntelliSub.Domain.Entities.Device" |
| `EntityTenantId` | `Guid?` | 实体租户ID | 多租户实体的租户ID |

## 枚举类型

### EntityChangeType — 实体变更类型
```csharp
public enum EntityChangeType
{
    Created = 0,     // 创建
    Updated = 1,     // 更新
    Deleted = 2,     // 删除
}
```

## 导航属性

```csharp
[Navigate(NavigateType.OneToMany, nameof(EntityPropertyChangeEntity.EntityChangeId))]
public virtual List<EntityPropertyChangeEntity> PropertyChanges { get; protected set; }
```

## 业务规则

### 实体变更记录规则
1. **自动记录**：ABP 框架自动记录实体的 CRUD 操作
2. **属性级变更**：详细记录每个属性的新值和旧值
3. **聚合根关联**：通过 AuditLogId 关联到审计日志
4. **变更类型**：区分创建、更新、删除三种类型

### 记录触发场景
- **Created**：实体首次创建时
- **Updated**：实体属性修改时
- **Deleted**：实体删除时

## 相关实体

- [[AuditLogAggregateRoot]] — 审计日志聚合根（多对一）
- [[EntityPropertyChangeEntity]] — 属性变更记录（一对多）

## 相关服务

- [[AuditLogService]] — 审计日志服务
- [[AuditLogActionService]] — 审计日志操作服务

## 数据查询示例

### 查询某实体的变更历史
```sql
SELECT
    ec.change_time,
    ec.change_type,
    al.user_name,
    al.url
FROM YiEntityChange ec
JOIN YiAuditLog al ON ec.audit_log_id = al.id
WHERE ec.entity_id = 'xxx'
AND ec.entity_type_full_name = 'Ast.IntelliSub.Domain.Entities.Device'
ORDER BY ec.change_time DESC;
```

### 查询某用户的实体变更操作
```sql
SELECT
    ec.entity_type_full_name,
    ec.entity_id,
    ec.change_type,
    ec.change_time,
    al.user_name
FROM YiEntityChange ec
JOIN YiAuditLog al ON ec.audit_log_id = al.id
WHERE al.user_id = 'xxx'
AND ec.change_time >= '2026-06-01'
ORDER BY ec.change_time DESC;
```

### 查询某时间段的所有删除操作
```sql
SELECT
    ec.entity_type_full_name,
    ec.entity_id,
    ec.change_time,
    al.user_name,
    al.client_ip_address
FROM YiEntityChange ec
JOIN YiAuditLog al ON ec.audit_log_id = al.id
WHERE ec.change_type = 2  -- Deleted
AND ec.change_time >= '2026-06-04 00:00:00'
AND ec.change_time <= '2026-06-04 23:59:59'
ORDER BY ec.change_time DESC;
```

### 查询实体变更及其属性变更
```sql
SELECT
    ec.id as change_id,
    ec.entity_type_full_name,
    ec.entity_id,
    ec.change_type,
    ec.change_time,
    ep.property_name,
    ep.original_value,
    ep.new_value
FROM YiEntityChange ec
LEFT JOIN YiEntityPropertyChange ep ON ec.id = ep.entity_change_id
WHERE ec.audit_log_id = 'xxx'
ORDER BY ec.change_time, ep.property_name;
```

## 索引建议

```sql
-- 审计日志ID索引（用于查询某审计日志的实体变更）
CREATE INDEX index_AuditLogId ON YiEntityChange(audit_log_id);

-- 租户ID + 实体类型 + 实体ID复合索引（用于查询特定实体的变更历史）
CREATE INDEX index_TenantId_EntityId ON YiEntityChange(tenant_id, entity_type_full_name, entity_id);
```

## 变更示例

### 示例1：设备创建变更
```
变更时间：2026-06-04 10:15:23
变更类型：Created
实体类型：Ast.IntelliSub.Domain.Entities.Device
实体ID：device-guid-123
操作用户：张三
属性变更：
  - Name: "" → "变压器A"
  - Type: null → 1
  - Status: null → 1
```

### 示例2：设备更新变更
```
变更时间：2026-06-04 14:30:45
变更类型：Updated
实体类型：Ast.IntelliSub.Domain.Entities.Device
实体ID：device-guid-123
操作用户：李四
属性变更：
  - Name: "变压器A" → "1号主变压器"
  - Status: 1 → 2
  - Remarks: null → "设备升级后名称变更"
```

### 示例3：设备删除变更
```
变更时间：2026-06-04 16:45:10
变更类型：Deleted
实体类型：Ast.IntelliSub.Domain.Entities.Device
实体ID：device-guid-456
操作用户：王五
属性变更：
  - Name: "旧开关柜" → (删除)
  - Type: 2 → (删除)
  - Status: 0 → (删除)
```

## 业务价值

**核心作用**：
1. **变更追溯**：完整记录实体的生命周期变更
2. **数据恢复**：支持从历史记录中恢复数据
3. **合规审计**：满足对重要数据变更的审计要求
4. **问题排查**：通过变更历史排查数据异常问题
5. **责任认定**：明确谁在何时修改了什么数据

## 设计模式

### 值对象模式
- `EntityChangeInfo` 是创建实体变更的值对象
- 包含变更时间、类型、实体信息等
- 通过构造函数确保数据完整性

### 聚合关系
- `EntityChangeEntity` 属于 `AuditLogAggregateRoot` 聚合
- `EntityPropertyChangeEntity` 属于 `EntityChangeEntity` 聚合
- 三层聚合关系：审计日志 → 实体变更 → 属性变更

### 审计最佳实践
```csharp
// ABP 框架自动审计
public class DeviceAppService : IApplicationService
{
    public async Task CreateAsync(CreateDeviceDto input)
    {
        // 框架自动记录：
        // 1. AuditLog（HTTP请求）
        // 2. EntityChange（实体创建）
        // 3. EntityPropertyChange（属性变更）
        
        var device = await _repository.InsertAsync(new Device {
            Name = input.Name,
            Type = input.Type
        });
        
        return ObjectMapper.Map<DeviceDto>(device);
    }
}
```

## 性能考虑

### 索引策略
- 审计日志ID索引：快速查询某操作的实体变更
- 实体ID索引：快速查询某实体的变更历史
- 复合索引：租户 + 实体类型 + 实体ID

### 数据清理
- 定期归档旧的实体变更记录
- 保留重要实体的完整变更历史
- 删除不重要的临时实体变更记录

### 查询优化
- 避免全表扫描，使用索引字段查询
- 分页查询变更历史
- 仅在需要时加载属性变更详情

---

> **最后更新**：2026-06-04
> **源码位置**：`module/audit-logging/Yi.Framework.AuditLogging.Domain/Entities/EntityChangeEntity.cs`