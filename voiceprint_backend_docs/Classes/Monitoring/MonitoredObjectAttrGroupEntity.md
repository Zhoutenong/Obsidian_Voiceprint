# MonitoredObjectAttrGroupEntity — 监测对象属性分组实体

## 基本信息

- **实体名称**：`MonitoredObjectAttrGroupEntity`
- **数据库表**：`ast_monitored_object_attr_group`
- **模块位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Monitoring/`
- **继承关系**：`Entity<Guid>`

## 实体说明

监测对象属性分组实体用于对监测对象的属性进行分组管理，支持在前端以分组形式展示属性信息。例如：基础信息、技术参数、维护信息等。

## 字段说明

### 主键与关联

| 字段名 | 数据类型 | 说明 | 约束 |
|-------|---------|------|------|
| `Id` | `Guid` | 主键 | Primary Key |
| `MonitoredObjectId` | `Guid` | 监测对象ID | Foreign Key |

### 分组信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `Name` | `string` | 分组名称 | 如："基础信息"、"技术参数" |
| `OrderNum` | `int?` | 排序号 | 控制分组显示顺序 |
| `Icon` | `string?` | 图标 | 前端显示的图标 |

### 审计信息

| 字段名 | 数据类型 | 说明 | 备注 |
|-------|---------|------|------|
| `CreationTime` | `DateTime?` | 创建时间 | - |

## 导航属性

```csharp
[Navigate(NavigateType.OneToOne, nameof(MonitoredObjectId))]
public MonitoredObjectAggregateRoot MonitoredObject { get; set; }
```

## 业务规则

### 分组管理规则
1. **自定义分组**：每个监测对象可以有独立的属性分组
2. **排序控制**：通过 `OrderNum` 控制分组显示顺序
3. **图标支持**：支持为每个分组配置图标，提升用户体验
4. **可选配置**：属性分组是可选的，可以不配置而使用默认分组

### 预置分组建议
| 分组名称 | 排序号 | 图标 | 适用对象 |
|---------|--------|------|---------|
| 基础信息 | 10 | info | 所有对象 |
| 技术参数 | 20 | settings | 设备对象 |
| 维护信息 | 30 | tool | 设备对象 |
| 运行数据 | 40 | chart | 设备对象 |
| 安全信息 | 50 | shield | 安全设备 |

## 使用场景

### 场景1：变压器属性分组
```
基础信息（图标: info）
- 设备名称
- 设备型号
- 制造厂家
- 投运日期

技术参数（图标: settings）
- 额定容量
- 额定电压
- 额定电流
- 冷却方式

维护信息（图标: tool）
- 上次检修日期
- 检修周期
- 下次检修日期
```

### 场景2：开关柜属性分组
```
基础信息
- 设备名称
- 设备型号
- 额定电压
- 额定电流

运行数据
- 负载电流
- 母线温度
- 触头温度
```

## 相关实体

- [[MonitoredObjectAggregateRoot]] — 监测对象
- [[MonitoredObjectAttrEntity]] — 监测对象属性
- [[MonitoredObjectAttrService]] — 监测对象属性服务

## 相关服务

- [[MonitoredObjectAttrGroupService]] — 监测对象属性分组服务
- [[MonitoredObjectService]] — 监测对象服务

## 数据查询示例

### 查询某监测对象的所有属性分组
```sql
SELECT * FROM ast_monitored_object_attr_group
WHERE monitored_object_id = 'xxx'
ORDER BY order_num;
```

### 查询分组及其属性
```sql
SELECT
    grp.id as group_id,
    grp.name as group_name,
    grp.icon as group_icon,
    grp.order_num as group_order,
    attr.id as attr_id,
    attr.name as attr_name,
    attr.value as attr_value
FROM ast_monitored_object_attr_group grp
LEFT JOIN ast_monitored_object_attr attr ON grp.id = attr.group_id
WHERE grp.monitored_object_id = 'xxx'
ORDER BY grp.order_num, attr.order_num;
```

## 索引建议

```sql
-- 监测对象索引（用于查询某对象的分组）
CREATE INDEX idx_monitored_object ON ast_monitored_object_attr_group(monitored_object_id);

-- 排序索引（用于显示排序）
CREATE INDEX idx_order_num ON ast_monitored_object_attr_group(order_num);
```

## 业务价值

**核心作用**：
1. **信息组织**：将大量属性信息按分组组织，便于查看
2. **用户体验**：支持图标和排序，提升前端展示效果
3. **灵活配置**：支持不同对象类型有不同的属性分组
4. **扩展性**：支持运行时动态添加属性分组

## 前端展示建议

### Accordion 折叠面板
```tsx
<Accordion>
  {groups.map(group => (
    <AccordionItem key={group.id} value={group.id}>
      <AccordionTrigger>
        <Icon name={group.icon} />
        {group.name}
      </AccordionTrigger>
      <AccordionContent>
        {attributes.filter(a => a.groupId === group.id).map(attr => (
          <div key={attr.id}>
            <span>{attr.name}:</span>
            <span>{attr.value}</span>
          </div>
        ))}
      </AccordionContent>
    </AccordionItem>
  ))}
</Accordion>
```

### Tabs 标签页
```tsx
<Tabs>
  {groups.map(group => (
    <TabsItem key={group.id} value={group.id}>
      <Icon name={group.icon} />
      {group.name}
    </TabsItem>
  ))}
</Tabs>

{groups.map(group => (
  <TabsPanel key={group.id} value={group.id}>
    {attributes.filter(a => a.groupId === group.id).map(attr => (
      <AttributeItem key={attr.id} attribute={attr} />
    ))}
  </TabsPanel>
))}
```

---

> **最后更新**：2026-06-04
> **源码位置**：`module/ast-intellisub/Ast.IntelliSub.Domain/Entities/Monitoring/MonitoredObjectAttrGroupEntity.cs`