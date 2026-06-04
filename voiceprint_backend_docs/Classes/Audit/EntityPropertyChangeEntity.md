# EntityPropertyChangeEntity — 属性变更记录实体

## 基本信息

- **实体名称**：`EntityPropertyChangeEntity`
- **数据库表**：`YiEntityPropertyChange`
- **模块位置**：`module/audit-logging/Yi.Framework.AuditLogging.Domain/Entities/`
- **继承关系**：`Entity<Guid>` → `IMultiTenant`

## 实体说明

属性变更记录实体用于记录实体属性级别的变更，包括变更前后的值。这是审计日志系统最细粒度的记录，能够精确追踪每个属性的变化。

## 字段说明

### 主键与关联

| 字段名 | 数据类型 | 说明 | 约束 |
|-------|---------|------|------|
| `Id` | `Guid` | 主键 | Primary Key |
| `EntityChangeId` | `Guid` | 关联实体变更ID | Foreign Key |
| `TenantId` | `Guid?` | 租户ID | 多租户支持 |

### 属性变更信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `PropertyName` | `string?` | 属性名称 | 如："Name"、"Status" |
| `PropertyTypeFullName` | `string?` | 属性类型全名 | 如："System.String"、"System.Int32" |
| `OriginalValue` | `string?` | 原始值（变更前） | 属性变更前的值 |
| `NewValue` | `string?` | 新值（变更后） | 属性变更后的值 |

## 业务规则

### 属性变更记录规则
1. **自动记录**：ABP 框架自动记录每个属性的变更
2. **字符串存储**：所有值都转换为字符串存储
3. **长度限制**：通过 Truncate 方法限制字段长度
4. **关联变更**：通过 EntityChangeId 关联到实体变更记录

### 值转换规则
- **简单类型**：直接转换为字符串（如：123 → "123"）
- **复杂类型**：序列化为 JSON 字符串
- **枚举类型**：存储枚举名称或数值
- **日期类型**：ISO 8601 格式字符串
- **null 值**：存储为 null 或空字符串

## 相关实体

- [[EntityChangeEntity]] — 实体变更记录（多对一）
- [[AuditLogAggregateRoot]] — 审计日志聚合根（间接关联）

## 相关服务

- [[AuditLogService]] — 审计日志服务

## 数据查询示例

### 查询某实体变更的所有属性变更
```sql
SELECT
    property_name,
    property_type_full_name,
    original_value,
    new_value
FROM YiEntityPropertyChange
WHERE entity_change_id = 'xxx'
ORDER BY property_name;
```

### 查询某属性的变更历史
```sql
SELECT
    ec.entity_id,
    ec.entity_type_full_name,
    ep.property_name,
    ep.original_value,
    ep.new_value,
    ec.change_time,
    al.user_name
FROM YiEntityPropertyChange ep
JOIN YiEntityChange ec ON ep.entity_change_id = ec.id
JOIN YiAuditLog al ON ec.audit_log_id = al.id
WHERE ep.property_name = 'Name'
AND ec.entity_type_full_name = 'Ast.IntelliSub.Domain.Entities.Device'
ORDER BY ec.change_time DESC;
```

### 查询某时间段的大额属性变更（假设）
```sql
-- 查询设备状态的所有变更
SELECT
    ep.property_name,
    ep.original_value,
    ep.new_value,
    ec.change_time,
    al.user_name
FROM YiEntityPropertyChange ep
JOIN YiEntityChange ec ON ep.entity_change_id = ec.id
JOIN YiAuditLog al ON ec.audit_log_id = al.id
WHERE ep.property_name = 'Status'
AND ec.change_time >= '2026-06-04 00:00:00'
AND ec.change_time <= '2026-06-04 23:59:59'
ORDER BY ec.change_time DESC;
```

### 对比属性的新旧值
```sql
-- 查询特定实体变更的属性对比
SELECT
    ep.property_name,
    ep.original_value as old_value,
    ep.new_value as new_value,
    CASE 
        WHEN ep.original_value = ep.new_value THEN '无变化'
        WHEN ep.original_value IS NULL AND ep.new_value IS NOT NULL THEN '新增'
        WHEN ep.original_value IS NOT NULL AND ep.new_value IS NULL THEN '删除'
        ELSE '修改'
    END as change_type
FROM YiEntityPropertyChange ep
WHERE ep.entity_change_id = 'xxx'
ORDER BY ep.property_name;
```

## 索引建议

```sql
-- 实体变更ID索引（用于查询某实体变更的属性变更）
CREATE INDEX index_EntityChangeId ON YiEntityPropertyChange(entity_change_id);
```

## 变更示例

### 示例1：文本属性变更
```
属性名称：Name
属性类型：System.String
原始值：变压器A
新值：1号主变压器
变更时间：2026-06-04 14:30:45
```

### 示例2：数值属性变更
```
属性名称：Status
属性类型：System.Int32
原始值：1（运行中）
新值：2（维护中）
变更时间：2026-06-04 14:30:45
```

### 示例3：日期属性变更
```
属性名称：InstallationDate
属性类型：System.DateTime
原始值：2020-01-15T00:00:00
新值：2020-03-20T00:00:00
变更时间：2026-06-04 14:30:45
```

### 示例4：枚举属性变更
```
属性名称：DeviceType
属性类型：Ast.IntelliSub.Domain.Shared.Enums.DeviceTypeEnum
原始值：0（Transformer）
新值：2（Switchgear）
变更时间：2026-06-04 14:30:45
```

### 示例5：复杂对象属性变更
```
属性名称：Metadata
属性类型：System.Collections.Generic.Dictionary`2[System.String,System.String]
原始值：{"Key1":"Value1","Key2":"Value2"}
新值：{"Key1":"Value1","Key2":"Value2","Key3":"Value3"}
变更时间：2026-06-04 14:30:45
```

## 业务价值

**核心作用**：
1. **细粒度审计**：精确追踪每个属性的变化
2. **数据恢复**：支持从历史值恢复数据
3. **变化对比**：直观展示属性的新旧值对比
4. **根因分析**：通过属性变更历史分析问题根因
5. **合规要求**：满足对重要字段变更的审计要求

## 使用场景

### 场景1：设备属性变更历史
```
查询设备 "变压器A" 的所有属性变更：

2026-06-04 14:30:45 - 用户：李四
  Name: "变压器A" → "1号主变压器"
  Status: 1 → 2
  Remarks: null → "设备升级后名称变更"

2026-06-03 10:15:23 - 用户：张三
  Name: "" → "变压器A"
  Type: null → 1
  Status: null → 1
```

### 场景2：用户权限变更审计
```
查询用户 "张三" 的角色变更历史：

2026-06-04 09:00:00 - 用户：管理员
  RoleId: "role-admin" → "role-manager"
  UpdatedBy: "admin"

2026-06-01 14:20:30 - 用户：管理员
  RoleId: null → "role-admin"
  CreatedBy: "admin"
```

### 场景3：配置参数变更追踪
```
查询系统配置 "告警阈值" 的变更历史：

2026-06-04 16:45:10 - 用户：系统管理员
  AlarmThreshold: "80" → "90"
  ModifiedBy: "admin"
  Reason: "根据实际运行情况调整"

2026-06-01 10:00:00 - 用户：系统管理员
  AlarmThreshold: "75" → "80"
  ModifiedBy: "admin"
  Reason: "初始配置"
```

## 设计模式

### 聚合关系
- `EntityPropertyChangeEntity` 是 `EntityChangeEntity` 的子聚合
- 三层聚合：`AuditLog` → `EntityChange` → `EntityPropertyChange`
- 确保数据一致性和完整性

### 值对象模式
```csharp
// 使用 EntityPropertyChangeInfo 值对象创建
public EntityPropertyChangeEntity(
    IGuidGenerator guidGenerator,
    Guid entityChangeId,
    EntityPropertyChangeInfo entityChangeInfo,
    Guid? tenantId = null)
{
    // 构造函数确保数据完整性
    Id = guidGenerator.Create();
    TenantId = tenantId;
    EntityChangeId = entityChangeId;
    NewValue = entityChangeInfo.NewValue.Truncate(...);
    OriginalValue = entityChangeInfo.OriginalValue.Truncate(...);
    PropertyName = entityChangeInfo.PropertyName.TruncateFromBeginning(...);
    PropertyTypeFullName = entityChangeInfo.PropertyTypeFullName.TruncateFromBeginning(...);
}
```

## 性能考虑

### 字段长度限制
- `NewValue` 和 `OriginalValue` 都有最大长度限制
- 通过 `Truncate` 方法自动截断过长的值
- 大文本内容建议存储到外部，这里仅存储引用

### 查询优化
- 避免查询所有属性变更，仅查询需要的属性
- 使用 EntityChangeId 索引加速查询
- 分页返回属性变更列表

### 存储优化
- 相同的属性变更可以合并存储
- 删除不需要审计的属性变更记录
- 定期归档旧的属性变更记录

## 敏感数据处理

### 敏感属性过滤
```csharp
// 某些敏感属性不应该记录变更
[DisableAuditing]
public class UserEntity
{
    public string Password { get; set; }  // 不审计密码变更
    public string SecretKey { get; set; } // 不审计密钥变更
}
```

### 数据脱敏
```csharp
// 某些属性的值需要脱敏存储
public string PhoneNumber 
{
    get => _phoneNumber;
    set
    {
        // 审计时只存储部分信息
        _phoneNumber = value;
        // 存储脱敏值：138****5678
    }
}
```

---

> **最后更新**：2026-06-04
> **源码位置**：`module/audit-logging/Yi.Framework.AuditLogging.Domain/Entities/EntityPropertyChangeEntity.cs`