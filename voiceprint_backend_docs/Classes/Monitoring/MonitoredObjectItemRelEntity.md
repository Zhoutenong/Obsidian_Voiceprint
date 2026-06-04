# MonitoredObjectItemRelEntity — 监测对象-监测项关联实体

## 基本信息

- **实体名称**：`MonitoredObjectItemRelEntity`
- **数据库表**：`ast_monitored_object_item_rel`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Monitoring/`
- **继承关系**：`Entity<Guid>` → `IAuditedObject`

## 实体说明

监测对象-监测项关联实体用于定义"哪些监测对象需要监测哪些参数"。这是监测系统的核心配置实体，实现了灵活的监测方案配置。

例如：
- 变压器A 监测 温度、振动、声音
- 变压器B 监测 温度、油位
- 开关柜C 监测 局放、温度

## 字段说明

### 主键与关联

| 字段名 | 数据类型 | 说明 | 约束 |
|-------|---------|------|------|
| `Id` | `Guid` | 主键 | Primary Key |
| `MonitoredObjectId` | `Guid` | 监测对象ID | Foreign Key |
| `MonitoredItemId` | `Guid` | 监测项ID | Foreign Key |

### 关联信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `Name` | `string` | 关联名称 | 自定义名称，便于识别 |
| `Status` | `CommonStatusEnum` | 状态 | 启用/禁用 |

### 审计信息

| 字段名 | 数据类型 | 说明 | 来源 |
|-------|---------|------|------|
| `CreationTime` | `DateTime` | 创建时间 | IAuditedObject |
| `CreatorId` | `Guid?` | 创建者ID | IAuditedObject |
| `LastModificationTime` | `DateTime?` | 最后修改时间 | IAuditedObject |
| `LastModifierId` | `Guid?` | 最后修改者ID | IAuditedObject |

## 枚举类型

### CommonStatusEnum — 通用状态
```csharp
public enum CommonStatusEnum
{
    Disabled = 0,    // 禁用
    Enabled = 1      // 启用
}
```

## 导航属性

```csharp
[Navigate(NavigateType.OneToOne, nameof(MonitoredObjectId))]
public MonitoredObjectAggregateRoot MonitoredObject { get; set; }

[Navigate(NavigateType.OneToOne, nameof(MonitoredItemId))]
public MonitoredItemEntity MonitoredItem { get; set; }
```

## 业务规则

### 关联配置规则
1. **唯一性**：同一监测对象 + 同一监测项只能有一条关联记录
2. **必需性**：只有启用的关联才会实际采集数据
3. **继承性**：监测对象可以从对象类型继承监测项配置
4. **独立性**：每个对象可以有独立的监测项配置

### 配置场景示例

#### 场景1：同类设备，不同监测方案
```
# 变压器类型 - 预置监测项：温度、振动、声音

变压器A → 监测项：温度、振动
变压器B → 监测项：温度、声音、油位
变压器C → 监测项：温度、振动、局放
```

#### 场景2：监测项启用/禁用
```
# 设备维修时禁用某项监测
变压器A → 温度（启用）、振动（禁用）、声音（启用）
```

## 相关实体

- [[MonitoredObjectAggregateRoot]] — 监测对象
- [[MonitoredItemEntity]] — 监测项
- [[MonitoredObjectTypeEntity]] — 监测对象类型

## 相关服务

- [[MonitoredObjectItemRelService]] — 监测对象监测项关联服务
- [[MonitoredObjectService]] — 监测对象服务
- [[MonitoredItemService]] — 监测项服务

## 数据查询示例

### 查询某监测对象的所有监测项
```sql
SELECT mi.*, rel.name as rel_name, rel.status
FROM ast_monitored_object_item_rel rel
JOIN ast_monitored_item mi ON rel.monitored_item_id = mi.id
WHERE rel.monitored_object_id = 'xxx'
AND rel.status = 1  -- 仅启用的
ORDER BY mi.order_num;
```

### 查询某监测项被哪些对象使用
```sql
SELECT mo.name as object_name, rel.name as rel_name
FROM ast_monitored_object_item_rel rel
JOIN ast_monitored_object mo ON rel.monitored_object_id = mo.id
WHERE rel.monitored_item_id = 'xxx'
AND rel.status = 1;
```

### 批量查询监测对象的监测项列表
```sql
-- 查询多个监测对象及其监测项
SELECT
    mo.id as object_id,
    mo.name as object_name,
    mi.id as item_id,
    mi.name as item_name,
    rel.status as item_status
FROM ast_monitored_object mo
LEFT JOIN ast_monitored_object_item_rel rel ON mo.id = rel.monitored_object_id AND rel.status = 1
LEFT JOIN ast_monitored_item mi ON rel.monitored_item_id = mi.id
WHERE mo.id IN ('obj1', 'obj2', 'obj3')
ORDER BY mo.name, mi.order_num;
```

## 索引建议

```sql
-- 唯一约束（对象ID + 监测项ID唯一）
CREATE UNIQUE INDEX ux_object_item ON ast_monitored_object_item_rel(monitored_object_id, monitored_item_id);

-- 监测对象索引（用于查询某对象的监测项）
CREATE INDEX idx_monitored_object ON ast_monitored_object_item_rel(monitored_object_id);

-- 监测项索引（用于查询某项被哪些对象使用）
CREATE INDEX idx_monitored_item ON ast_monitored_object_item_rel(monitored_item_id);

-- 状态索引（用于筛选启用的关联）
CREATE INDEX idx_status ON ast_monitored_object_item_rel(status);
```

## 业务价值

**核心作用**：
1. **灵活配置**：支持每个对象独立配置监测方案
2. **按需启用**：可以临时禁用某项监测而不删除配置
3. **批量管理**：支持批量查询和管理监测配置
4. **继承扩展**：支持从类型继承 + 独立扩展的混合模式

## 设计模式

### 配置模式
- **类型级配置**：在监测对象类型上定义默认监测项
- **对象级覆盖**：在对象级别可以覆盖类型级配置
- **运行时调整**：支持运行时启用/禁用监测项

### 性能考虑
- 监测配置缓存在内存中，避免频繁查询数据库
- 配置变更时刷新缓存
- 数据采集时直接使用缓存配置

---

> **最后更新**：2026-06-04
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Monitoring/MonitoredObjectItemRelEntity.cs`