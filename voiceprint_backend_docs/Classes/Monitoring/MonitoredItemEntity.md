# MonitoredItemEntity — 监测项实体

## 基本信息

- **实体名称**：`MonitoredItemEntity`
- **数据库表**：`ast_monitored_item`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Monitoring/`
- **继承关系**：`Entity<Guid>`

## 实体说明

监测项实体定义了系统可以监测的具体参数类型，如温度、振动、压力等。监测项是监测参数的模板定义，可以关联到多个监测对象，实现参数复用。

## 字段说明

### 主键

| 字段名 | 数据类型 | 说明 | 约束 |
|-------|---------|------|------|
| `Id` | `Guid` | 主键 | Primary Key |

### 基本信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `Name` | `string` | 显示名称 | 如："温度"、"振动" |
| `Description` | `string?` | 描述 | 详细说明 |

### 显示配置

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `OrderNum` | `int?` | 排序号 | 控制显示顺序 |
| `Icon` | `string?` | 图标 | 前端显示图标 |
| `IsDisplay` | `bool` | 是否显示 | 默认true，控制是否在界面展示 |

## 业务规则

### 监测项管理规则
1. **全局复用**：监测项是全局定义的参数类型，可被多个对象使用
2. **显示控制**：通过 IsDisplay 控制是否在界面展示
3. **排序支持**：通过 OrderNum 控制显示顺序
4. **图标配置**：支持为每个监测项配置图标，提升用户体验

### 预置监测项建议
| 监测项名称 | 描述 | 排序号 | 图标 |
|-----------|------|--------|------|
| 温度 | 设备或环境温度 | 10 | thermometer |
| 振动 | 机械振动参数 | 20 | activity |
| 声音 | 声音强度或频谱 | 30 | volume-up |
| 湿度 | 环境湿度 | 40 | water |
| 压力 | 液体或气体压力 | 50 | gauge |

## 相关实体

- [[MonitoredObjectItemRelEntity]] — 监测对象-监测项关联
- [[MonitoredPointEntity]] — 监测点位

## 相关服务

- [[MonitoredItemService]] — 监测项服务
- [[MonitoredObjectItemRelService]] — 对象监测项关联服务

## 数据查询示例

### 查询所有可显示的监测项
```sql
SELECT * FROM ast_monitored_item
WHERE is_display = 1
ORDER BY order_num;
```

### 查询某对象的所有监测项
```sql
SELECT
    mi.id,
    mi.name,
    mi.description,
    mi.icon,
    rel.status
FROM ast_monitored_item mi
JOIN ast_monitored_object_item_rel rel ON mi.id = rel.monitored_item_id
WHERE rel.monitored_object_id = 'xxx'
AND rel.status = 1  -- 仅启用的
AND mi.is_display = 1
ORDER BY mi.order_num;
```

### 查询监测项及其使用统计
```sql
SELECT
    mi.id,
    mi.name,
    mi.description,
    COUNT(rel.monitored_object_id) as used_count
FROM ast_monitored_item mi
LEFT JOIN ast_monitored_object_item_rel rel ON mi.id = rel.monitored_item_id
GROUP BY mi.id, mi.name, mi.description
ORDER BY used_count DESC, mi.order_num;
```

## 索引建议

```sql
-- 排序索引（用于按顺序显示）
CREATE INDEX IX_order_num ON ast_monitored_item(order_num);

-- 显示状态索引（用于筛选可显示项）
CREATE INDEX IX_is_display ON ast_monitored_item(is_display);
```

## 业务价值

**核心作用**：
1. **参数复用**：一次定义，多处使用，避免重复配置
2. **标准化管理**：统一管理监测参数类型和命名
3. **灵活配置**：支持显示控制和排序，优化用户体验
4. **扩展性**：便于添加新的监测参数类型

## 使用场景

### 场景1：创建监测项
```
创建"温度"监测项：
- Name: "温度"
- Description: "设备或环境温度监测"
- OrderNum: 10
- Icon: "thermometer"
- IsDisplay: true
```

### 场景2：关联到监测对象
```
变压器A 关联监测项：
- 温度（启用）
- 振动（启用）
- 声音（启用）

开关柜B 关联监测项：
- 温度（启用）
- 湿度（启用）
```

### 场景3：前端展示
```tsx
<MonitorItemList>
  {items.map(item => (
    <MonitorItem
      key={item.id}
      icon={item.icon}
      name={item.name}
      description={item.description}
    />
  ))}
</MonitorItemList>
```

## 设计模式

### 模板模式
- `MonitoredItemEntity` 是监测参数的模板定义
- 通过 `MonitoredObjectItemRelEntity` 将模板实例化到具体对象
- 实现了参数定义和参数实例的分离

### 全局配置
```csharp
// 全局预置监测项
var preItems = new[]
{
    new MonitoredItemEntity { Name = "温度", OrderNum = 10 },
    new MonitoredItemEntity { Name = "振动", OrderNum = 20 },
    new MonitoredItemEntity { Name = "声音", OrderNum = 30 }
};

// 对象复用监测项
var objectItem = new MonitoredObjectItemRelEntity
{
    MonitoredObjectId = objectId,
    MonitoredItemId = preItems[0].Id,  // 复用"温度"监测项
    Status = CommonStatusEnum.Enabled
};
```

---

> **最后更新**：2026-06-04
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Monitoring/MonitoredItemEntity.cs`
